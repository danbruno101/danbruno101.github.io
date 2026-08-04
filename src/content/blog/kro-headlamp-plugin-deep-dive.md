---
title: 'Inside the kro Plugin for Headlamp: Dynamic CRDs, Topological Graphs, and Label-Driven Discovery'
description: 'Technical internals of the kro Plugin for Headlamp: resolving runtime-generated CRDs, reconstructing a topological resource graph, and discovering label-owned sub-resources with no owner references to lean on.'
pubDate: 2026-08-04
---

This is the technical follow-up to my [announcement post](/blog/kro-headlamp-plugin-alpha/) on the kro Plugin for Headlamp's first alpha release. That post covered what the plugin does and why; this one covers how — three problems that made this plugin harder to build than a typical "list some resources" Headlamp plugin, and how the actual implementation solves each one.

If you haven't used kro before: a `ResourceGraphDefinition` (RGD) describes a graph of Kubernetes resources plus a simplified schema for the API you want developers to use. kro turns that RGD into a real CRD, and reconciles instances of it into the underlying resources. The plugin's job is to make that entire pipeline — definition → generated API → live instances → composed resources — inspectable from the UI.

## Problem 1: the CRDs don't exist until runtime

A normal Headlamp resource plugin targets a known `apiVersion`/`kind` at build time. This plugin can't, because the whole point of kro is that platform teams define *new* APIs at runtime — every cluster can have a completely different set of Active RGDs, each minting its own CRD.

The plugin resolves this in two steps. First, `useGeneratedCrd` lists the cluster's CRDs (via Headlamp's `CustomResourceDefinition.useList()`) and finds the one matching an RGD's `spec.schema.group` + `spec.schema.kind`. Second — and this is the detail that matters — the *CRD*, not the RGD, is treated as the source of truth for the plural name, scope, and storage version, because the RGD's schema block never publishes those. `instanceApiInfoFromCrdSpec` reads them straight off the CRD spec.

From that API info, `makeInstanceClass` builds a Headlamp `KubeObject` subclass on the fly — but it caches these classes in a `Map` keyed by `group/version/plural`. That cache isn't an optimization; it's a correctness requirement. Headlamp's list/watch hooks are keyed on class identity, so if you rebuilt a fresh class on every render, you'd remount every consumer and silently break the watch. Same generated API always resolves to the same class instance.

## Problem 2: the dependency graph and the node list live in different places

An RGD's `status.resources` looks like it should be the list of composed resources, but it isn't — kro only publishes an entry there for a resource if it has at least one dependency (verified against kro 0.9.2). A resource with no dependencies is simply absent from `status.resources`, which means you can't use it as your node list without silently dropping standalone resources.

`getComposedResources` in `rgdGraph.ts` gets this right by pulling from three different places and merging them:
- **Node identity and kind** come from `spec.resources[]` — the actual, complete list.
- **Dependency edges** come from `status.resources[].dependencies` — kro's own static analysis, never re-derived locally.
- **Render order** follows `status.topologicalOrder` when present, falling back to spec declaration order for any id the status doesn't cover (which also makes `Inactive` RGDs without status render sensibly instead of blank).

The sort is a genuine two-key sort — topological index first, spec index as a deterministic tie-breaker — rather than relying on `Array.sort`'s stability, which isn't guaranteed the same way across engines.

## Problem 3: sub-resources are custom resources with no fixed type, discovered by label

This was the hardest part, because "show me what kro created for this instance" can't be a single typed watch — the resources are Deployments, PVCs, Services, or anything else a template happens to produce, and none of them carry an `ownerReference` back to the instance. **kro 0.9.x uses applysets instead of owner references**, so ownership is only recoverable from labels: every resource kro creates gets stamped with `kro.run/owned=true` and `kro.run/instance-id=<uid>`.

Given that, the plugin can't run one generic watch — Kubernetes list/watch APIs are per-GVK. So `getKindsToWatch` first scans the RGD's *templates* (excluding external references, which kro reads but never creates) to get the distinct `apiVersion:kind` pairs actually in play for this RGD. Then, for each kind, `resolveResourceClass` picks a watchable class in priority order: an exact built-in match, then the cluster's CRD for that group+kind (for kro-generated or other custom kinds), then a built-in class with the same kind as a last-resort fallback rather than dropping the watch entirely.

Each kind gets its own `KindCollector` component running its own `useList({ labelSelector: 'kro.run/owned=true,kro.run/instance-id=<uid>' })` — deliberately structured as N small components rather than a hook-in-a-loop, so the number of hooks mounted stays constant even as the set of kinds changes between RGDs (avoiding React's rules-of-hooks footgun). Each collector reports its items up through a parent reducer that dedupes by a `uid:resourceVersion` signature string, since watch updates can mutate objects in place and defeat a naive identity check.

Health and "resolved values" — the actual `storageClassName` a PVC ended up with, a Deployment's live ready-replica count, a Service's assigned type and `clusterIP` — are computed by small pure, per-kind functions (`getSubResourceHealth`, `getResolvedValues`) operating on the live `jsonData` these watches deliver. Nothing is read from the instance's declared spec; it's all the reconciled state, which is what actually answers "did this resolve correctly on this cluster."

RBAC gaps are handled at the same per-kind granularity: if listing one kind fails, that `KindCollector`'s error is reported and rendered as "Unable to list `<Kind>`: `<server message>`" without blocking the other kinds or the page. The kro-install check (`KroInstallGuard`) works similarly but coarser — it does a single `useGet` on the RGD CRD itself, treats a 404 as "not installed" and shows an install pointer, but stays neutral on any other error (including RBAC) so pages still attempt to render rather than assuming absence.

## The Map integration doesn't fetch instances at all

I expected the Map integration to reuse the same instance/sub-resource watching machinery. It doesn't, and the reason is a real constraint: Map sources need a fixed, static list of hooks registered up front, but instances are custom resources of dynamically discovered CRDs — you can't know what to watch without already having watched something else first.

The actual approach sidesteps the problem instead of solving it head-on. The kro Map source (`kroMapSource`) has one sub-source per *built-in* kind kro commonly creates (`Deployment`, `StatefulSet`, `Service`, `PersistentVolumeClaim`, `ConfigMap`, `Secret`, `Job`, `ServiceAccount`, `Role`, `RoleBinding`) — a fixed list, so the hook count is fixed too — each watching with the `kro.run/owned=true` selector. The instance *node itself* is never separately fetched; it's synthesized purely from labels already present on each owned resource (`kro.run/instance-id`, `instance-name`, `instance-kind`, `instance-namespace`, `resource-graph-definition-name/id`). Every owned resource independently reconstructs the same instance node and the same RGD→instance edge, and Map's node deduplication (first-write-wins on id) collapses the duplicates back into one. It's a clever way to get an "instance" node into the graph without ever running a watch against the instance's actual (unknown-until-runtime) type.

"View in Map" links are just `?node=<uid>` query params into Headlamp's existing Map route — since RGDs, synthesized instances, and owned resources are all registered by `uid`, any of the plugin's detail pages can deep-link into Map without Map needing to know anything kro-specific beyond the source registration.

## What's next

The current Map integration is a deliberate workaround, not the end state — embedding a kro-specific graph directly on RGD/instance detail pages is still blocked on Headlamp exposing its Map renderer to plugins ([headlamp#6556](https://github.com/kubernetes-sigs/headlamp/issues/6556)). There's also a known upstream issue where Map can get stuck loading on certain clusters ([headlamp#6555](https://github.com/kubernetes-sigs/headlamp/issues/6555)), independent of this plugin. Compatibility is currently verified against kro 0.9.2 and the `v1alpha1` API; since sub-resource and instance discovery are both driven off live cluster state (CRDs, labels, RGD status) rather than anything hardcoded, the hope is that most of this keeps working as kro's API surface evolves — but that's a hypothesis I want more alpha usage to stress-test, not a guarantee.

If you're building on either kro or Headlamp's plugin API, I'd like to compare notes — particularly on the applyset-over-ownerReferences implication for label-based discovery, since I suspect that pattern shows up anywhere you're layering a simplified API on top of raw Kubernetes objects without owner references to lean on.

- [Plugin PR #912](https://github.com/headlamp-k8s/plugins/pull/912) / [proposal, #911](https://github.com/headlamp-k8s/plugins/issues/911)
- [Source: `headlamp-k8s/plugins/kro`](https://github.com/headlamp-k8s/plugins/tree/main/kro)
- [Release notes](https://github.com/headlamp-k8s/plugins/releases/tag/kro-0.1.0-alpha)
- [kro.run](https://kro.run) · [Headlamp plugin docs](https://headlamp.dev)
