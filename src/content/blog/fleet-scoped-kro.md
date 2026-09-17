---
title: "kro knows your app is one thing. The cluster boundary makes it forget."
description: "Composition stops at the edge of a cluster. What it would take for a kro instance to describe resources that span a fleet, why that isn't a scheduling problem, and the questions I'm bringing to SIG Multicluster, SIG Cloud Provider and the kro community."
pubDate: 2026-09-17
---

If you've used [kro](https://github.com/kubernetes-sigs/kro), you know the pitch: a platform team writes a `ResourceGraphDefinition`, kro turns it into a real CRD, and application teams get a five-field API instead of a four-hundred-line pile of cloud resources. I've written about [why that's valuable](/blog/kro-headlamp-plugin-alpha/) and [built tooling to make it visible](/blog/kro-headlamp-plugin-deep-dive/).

This post is about the thing kro can't do yet, and about the problem-framing document I've been working through with SIG Multicluster, SIG Cloud Provider and the kro community to figure out whether it *should*. It's deliberately not a proposal. It ends in questions, and the point of writing it up here is to get more people asking them.

## What kro actually knows

The templating is the visible part of kro, but it isn't the important part. The important part is that after an RGD expands, **kro knows those N objects are one unit.** Ordering between them is derived, not declared — kro reads the CEL references between templates, computes the DAG, and publishes it in status. One owner, one readiness condition, one delete, one place where environment branching lives.

Here's the shape of that in practice. An infrastructure team publishes an RGD for a managed Postgres on Google Cloud — tier sizing, HA, backups, point-in-time recovery, private networking, SSL, IAM auth, query insights, slow-query logging, a deletion policy that refuses to nuke prod. Roughly a hundred lines of Config Connector resources wired together with CEL. And here is the *entire* surface an application team touches:

```yaml
apiVersion: kro.run/v1alpha1
kind: PostgresDatabase
metadata:
  name: orders
  namespace: team-orders
spec:
  name: orders
  environment: prod
  size: medium
  iamServiceAccount: orders-api@my-project.iam.gserviceaccount.com
```

Self-service infrastructure that's fast for developers, safe by default, and cheap to govern. That's the value.

## Where it stops

That knowledge stops at the cluster boundary. A kro instance is a single-cluster object by design, and every resource it expands into lands in the cluster the instance lives in.

There are two distinct layers here, and only one of them exists across clusters today:

- **Composition** knows that a set of resources is one unit, and how they relate.
- **Placement** decides where things go.

SIG Multicluster has built substantial placement and delivery machinery. [ClusterProfile](https://multicluster.sigs.k8s.io/concepts/cluster-profile-api/) gives you inventory. The About API gives you cluster attributes to select on. The Work API delivers manifests. And the [PlacementDecision API](https://github.com/kubernetes/enhancements/blob/master/keps/sig-multicluster/5313-placement-decision-api/README.md) — more on that below — standardises the *output* of a scheduler so consumers don't have to learn every scheduler's private format.

**Nothing carries composition knowledge across the boundary.**

To be precise, because this is the most likely misreading: I am *not* claiming multicluster workload delivery is unsolved. Karmada, Open Cluster Management, Fleet and Argo all propagate workloads to many clusters, and do it well. But they propagate manifests that were already written. They don't compose a dependency graph from references, and they don't produce a single object whose status means "this application is ready."

## Three use cases that put a face on it

**Capacity.** A model server needs eight GPU replicas and no single cluster has eight free. Today the workload is split by hand into per-cluster deployments that then drift apart. Same shape applies to the efficiency framings — bin-packing for utilisation, spot capacity, cheaper clusters.

**Compliance.** Regulated data may only run in a FedRAMP region. Note that this *inverts* the capacity case: spilling over to a cluster with room isn't a graceful fallback, it's a violation. It's a hard constraint that must fail closed. And it's a constraint the infrastructure team has to be able to author *once*, in the RGD, so application teams inherit it and can't override it. Only kro sees both the constraint and the composition, so kro has to carry it into the placement request — the scheduler evaluates it, and kro enforces the answer.

**Failover.** A cluster goes unhealthy, is drained for maintenance, or is consolidated away for cost. Everything on it has to end up somewhere still valid — together, and in dependency order.

## Why today's answers don't close it

- **Propagation tools** (Karmada, OCM, Fleet, Argo) place and deliver, but the unit they move is a manifest set someone else assembled.
- **Per-resource CRD models** (Crossplane, Config Connector, ACK) reconcile each cloud resource independently and continuously. That's correct, but there's no notion of a group. Ordering is left to retry-until-ready, which works for compute and does not work where a missing dependency loses data rather than degrading.
- **Flux `dependsOn` + health checks** genuinely provides ordering today, at Kustomization granularity. But the DAG is hand-authored and kept in sync by people, grouping lives in folder layout rather than in the API, nothing answers "is my app ready" as one object, and it says nothing about teardown order.

## What any solution has to do

Whether or not kro is the vehicle, the requirements are the same. Condensed:

1. Express "these resources are one unit" such that it survives crossing a cluster boundary — ordering, readiness and teardown included.
2. Consume SIG Multicluster primitives rather than inventing parallel ones. ClusterProfile for inventory, About API properties for selection, PlacementDecision for the decision, probably Work API for delivery.
3. Leave the developer-facing object unchanged. The `PostgresDatabase` above should look identical whether it lands in one cluster or five. Cluster names must not leak into the developer's spec.
4. Treat placement as an **input** the composition layer consumes, never something it computes.
5. Support hard, fail-closed constraints as a first-class case. "No compliant cluster has capacity" must be a terminal, auditable condition — never a fallback to a non-compliant cluster.
6. Keep delivery pluggable. In some accredited environments a hub holding credentials into member clusters is simply not permitted, so push cannot be assumed.
7. Degrade honestly under partial failure. A half-placed unit must be visible as such, with per-cluster detail, rather than reporting healthy.

Explicitly out of scope: building a scheduler (that's Karmada, OCM and Kueue territory), data movement, cross-cluster networking (that's MCS-API's job), and replacing GitOps.

## Five shapes, and why no single one is enough

The document sketches five approaches. Three are **semantics** — what a composed unit means once it can span clusters. Two are **transports** — how a resource actually gets created in another cluster. A real design is one of the first three sitting on one of the last two.

**Whole-instance placement.** The instance is atomic; it lands in exactly one cluster (or is replicated wholesale to N), and placement chooses which. Simplest possible semantics, no new composition concepts, the graph never spans a boundary, and the compliance case is largely served. Does nothing for the capacity case, because replicating a graph to N clusters is not the same as dividing it across them. This is the one I've prototyped end to end — see below.

**Placement groups.** Resources inside one instance are grouped; groups are placed; edges *within* a group are guaranteed co-located, edges *across* groups must be connectivity-only. This is deliberately the Pod analogy: a Pod is the atomic scheduling unit because its containers must share a node, and a resource graph has the same property — a Deployment cannot mount a PVC in another cluster. Groups are what the GPU case actually needs. They're also the largest new API surface, and they make cross-group references distributed dataflow and partial failure a first-class state.

**Cross-cluster `externalRef`.** kro already has a read-only reference to an object it doesn't own. Let it name another cluster. Small, reuses an existing concept, and probably covers a lot of real "the thing is over there" cases — a platform config on a hub, a model registry, an existing managed database. It creates nothing remotely, so it serves neither capacity nor drain, and every such reference is potential data movement across a jurisdiction.

**Work API delivery.** kro resolves the graph but doesn't apply remotely; it emits Work objects that existing fleet agents deliver, and reads status back. Best ecosystem fit, least new machinery, and a pull model comes almost for free. The open question is whether the summarised, asynchronous status coming back is enough to drive readiness aggregation and CEL references.

**API projection.** This one exists today, in-family: [manifold](https://github.com/a-hilaly/k8s-multi-cluster), by kro maintainer a-hilaly, registers an aggregated API server on a hub so remote resources are addressable as native objects — `deployments.apps.cluster-1.k8s.io`, rewriting URLs, watch streams and discovery bidirectionally. Because OpenAPI is projected too, kro can type-check CEL against remote schemas, and an RGD on the hub referencing a remote resource "just works." It needs no changes to kro at all. It also has the most unanswered semantics: the hub sits on the request path and holds credentials for every member, cluster identity is defined inline rather than via ClusterProfile, and — critically — ownership doesn't cross the boundary. manifold strips `ownerReferences` from projected responses by default, because otherwise the hub's garbage collector would act on objects whose owners it can't see. So you get addressability without getting teardown.

The honest summary: **no single row is sufficient.** The transports solve reachability and say nothing about policy or lifecycle. The semantics rows assume a transport exists. That split is itself the argument that the design questions are the right place to start, rather than the YAML.

## The questions I'm bringing to the SIGs

**Is a single instance spanning multiple clusters even the right model?** Or should a spanning application be several coordinated instances under a parent? Concretely: is one object whose resources live in several clusters consistent with how SIG Multicluster thinks about clustersets and namespace sameness, or does it violate an assumption I haven't spotted? This is the biggest fork in the road, so it goes first.

**Where does the placement decision come from?** When I first wrote the framing, my ask here was "would SIG Multicluster standardise the output side of scheduling the way ClusterProfile standardised inventory?" The answer turns out to be that they already have. [KEP-5313](https://github.com/kubernetes/enhancements/blob/master/keps/sig-multicluster/5313-placement-decision-api/README.md) defines `PlacementDecision` in [cluster-inventory-api](https://github.com/kubernetes-sigs/cluster-inventory-api): a namespaced list of `clusterProfileRef`s, each with an optional reason, sliced EndpointSlice-style at a hundred clusters per object and correlated by a decision-key label. Any scheduler can produce one; any consumer can read one. That's most of what a composition layer needs — an ordered set of member identities that resolve to ClusterProfiles. What it doesn't carry today is a **per-member parameter map** (this cluster gets three replicas, that one gets five), which is exactly what the capacity case needs to divide rather than replicate. Whether that belongs in `PlacementDecision`, in a sidecar object, or in the consumer's own config is a question I'd rather settle with the SIG than assume.

**What is the atomic unit of placement — instance, group, or resource?** Per-resource is most flexible and, in my view, wrong: some edges can't be cut. Whole-instance is simplest and doesn't serve the GPU case. Groups are the middle. The closer in-tree analogy may be PodGroup ([KEP-4671](https://www.kubernetes.dev/resources/keps/4671/)): a group with an all-or-nothing policy and a single scheduled condition. Is that the right model at fleet scope?

**Should cross-cluster CEL references be allowed at all?** In one cluster, `${platformConfig.data.endpoint}` is a template reference. Across clusters it's an export of data from one jurisdiction into another — a Secret referenced from a US cluster into an EU cluster has moved that Secret. My first-iteration proposal: allow hub-to-member and within-cluster references, and require an explicit export declaration for anything crossing a boundary. Too restrictive to be useful?

**What does "Ready" mean for a spanning instance?** Three of five clusters converged — is the instance Ready? The PoC's answer is per-cluster entries in `status.clusters[]` plus an explicit `minReadyClusters` tolerance folding into the rolled-up condition. Whether that should generalise to a policy enum (All, Any, Quorum, percentage), and whether it matches how the SIG thinks about fleet-level status elsewhere, is open. What's *not* open, in my view: an **empty** placement — no cluster qualifies, or a policy engine refused — must be terminal and explicit. Never "not ready yet." Never a fallback to a broader set. That distinction is what makes the compliance requirement real.

**Should the transport be projection or Work-style delivery?** Or the third option the PoC actually uses: [multicluster-runtime](https://github.com/kubernetes-sigs/multicluster-runtime) with its ClusterProfile provider, which starts and stops reconciliation against clusters as they're discovered and applies server-side. It's push, like projection, but without putting the hub on the request path for every client, and it reuses an existing SIG Multicluster component. Is there a reason the SIG would prefer one of these as *the* convention, or should the composition layer treat them as pluggable back ends?

**Who is trusted to assert cluster properties?** If placement respects `compliance/fedramp: high` from a ClusterProperty, a mislabelled cluster is a compliance breach rather than a scheduling mistake. Is there a trust or provenance model for About API properties, or is that an open gap?

## The hard parts, listed so nobody has to find them in review

**Data gravity is the hardest constraint, and it's not theoretical.** A Deployment cannot mount a PVC in another cluster. That makes certain edges non-cuttable and is the main argument for groups over per-resource placement. The single-cluster version of this bit me in the demo: two replicas sharing one `ReadWriteOnce` PVC work fine on single-node kind and deadlock with a Multi-Attach error the moment they land on different nodes of a real cluster. The fleet version of that mistake is worse and quieter. Consequence for failover: stateless groups can genuinely be evacuated; stateful ones need a data plane and a human decision, and an RGD probably has to declare which is which.

**Ownership does not cross clusters.** Kubernetes garbage collection is per-cluster. An owner in cluster A cannot own a dependent in cluster B, whatever the transport. So "one delete reclaims the whole unit" is not free. The PoC's answer is a hub-side applied-manifest inventory per (instance, member) — conceptually OCM's `AppliedManifestWork` — with a finalizer on the hub instance driving deletion: on delete, or when a member leaves the placement, exactly the tracked objects are removed from that member. The narrower question for the SIG is whether hub-side tracking plus a finalizer is the pattern they want standardised, given OCM already does essentially this.

**Push vs. pull may be dictated by compliance, not preference.** In FedRAMP High and IL5-style environments, a hub outside the accreditation boundary holding credentials into clusters inside it is frequently not permissible. That rules out hub-centric remote apply for exactly the users who most want the compliance case.

**Version skew breaks static analysis.** kro's graph acceptance is computed against one cluster's API surface. Member clusters differ, so `GraphAccepted` may need to become per-target-cluster.

**Scale.** N clusters times M instances of watches and status is the obvious wall. Summarised per-cluster status is probably required, and it trades away exactly the detail the composition layer exists to provide.

## What runs today

I didn't want to bring the SIGs a document with nothing behind it, so whole-instance placement has been prototyped end to end in [kro-fleet](https://github.com/danbruno101/kro-fleet). You author one placement-enabled object on a hub; it's placed onto matching members via ClusterProfile and multicluster-runtime; per-member status aggregates back into `status.clusters[]`; a finalizer tears down exactly what was applied, with no orphans; there's an e2e suite on kind and a Headlamp fleet view. It runs against stock kro on the members — no fork.

It's a discussion-stage PoC and says so on the tin. Alongside it is a [draft KEP](https://github.com/danbruno101/kro-fleet/blob/main/docs/proposals/KEP-kro-multicluster.md) sketching the mechanism, including a `decisionRef` so placement can be supplied by an external producer rather than an inline selector, and a [gap ledger](https://github.com/danbruno101/kro-fleet/blob/main/docs/KEP-GAP.md) that's honest about what's proposed versus what runs: per-member parameters, terminal refusal on empty placement, decision provenance in status, and a `Degraded` state for unreachable members are all designed and not yet built. The KEP is deliberately provisional. It gets revised after the SIGs weigh in, not before.

## Where to weigh in

The full problem framing is in [this document](https://docs.google.com/document/d/14wPdfwK4yHFHRCtaLrSlLvn0_Yl8Wv5p6j2dYfk3MO0/edit), and it's open for comments. If you run kro today and have hit the cluster boundary, if you maintain a multicluster scheduler and have opinions about what a consumer should be allowed to expect from you, or if you think the whole framing is wrong — I want to hear it, on the doc, in the [kro-fleet](https://github.com/danbruno101/kro-fleet) issues, or in `#sig-multicluster` and `#kro` on Kubernetes Slack.

Nothing here should block MCS-API, ClusterProfile or PlacementDecision work. If anything, it consumes all three.
