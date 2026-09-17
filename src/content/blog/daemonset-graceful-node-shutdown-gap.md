---
title: "The DaemonSet controller doesn't know your node is shutting down"
description: "What happens when Graceful Node Shutdown and the DaemonSet controller disagree, four years of production reports that say it matters, and the fix being proposed upstream."
pubDate: 2026-09-17
---

Every Kubernetes node eventually shuts down. Spot instances get reclaimed, autoscalers scale in, operating systems get patched, hardware gets retired. Kubernetes has a well-designed mechanism for handling that moment — Graceful Node Shutdown — and it works. But there's a gap in how the rest of the cluster finds out about it, and DaemonSets fall straight into that gap.

People have been falling into it, and filing issues about it, since 2022. This post is about that gap: what actually happens, what operators have reported when it happens to them, and what a fix looks like.

## What Graceful Node Shutdown does

When a node's operating system begins shutting down, kubelet finds out through a systemd inhibitor lock. It holds the shutdown open for a configured grace period and uses that time to terminate pods in order — ordinary workloads first, then critical system pods, with an optional priority-based ordering for finer control. Pods get their `preStop` hooks and termination grace periods. Nothing is yanked.

During that window, kubelet also flips into a defensive mode: any new pod that lands on the node is rejected at admission with a `NodeShutdown` reason. This is correct. There's no point starting a container on a machine that will be powered off in sixty seconds.

So kubelet knows the node is shutting down, and kubelet behaves accordingly. The problem is that kubelet is the *only* component that knows.

## What the DaemonSet controller sees

The DaemonSet controller has a simple job: for every node that matches, make sure exactly one pod is running. It doesn't wait for the scheduler to find room the way a Deployment does — it targets nodes directly, and it's built to be stubborn about it, because DaemonSets are how you run the things that must exist on every node: CNI agents, kube-proxy, log shippers, node exporters, storage drivers, security agents.

That stubbornness is why the usual "don't schedule here" signals don't reach it:

- **Cordoning** doesn't stop DaemonSets. They're supposed to run on cordoned nodes.
- **`NotReady`** doesn't stop them either. Kubelet does mark the node NotReady during shutdown, but DaemonSet pods tolerate `node.kubernetes.io/not-ready` by default — they have to, or your CNI agent would never get scheduled onto a node that's NotReady *because* it has no CNI yet. And `NotReady` doesn't say *why*: a planned shutdown and a network partition look identical from the API.
- **Taints** are the wrong tool. A taint applied by kubelet would affect every workload on the node, not just DaemonSets, and the controller would still need to be taught which taint means what.

So the controller does exactly what it's designed to do. It observes that the node should have a pod, observes that kubelet has terminated it, and creates a replacement. Kubelet rejects it. The controller sees a failed pod, deletes it, and creates another. Repeat until the machine loses power.

This isn't theoretical. The earliest report I can find is from April 2022, on Kubernetes 1.23, in [kubernetes/kubernetes#109450](https://github.com/kubernetes/kubernetes/issues/109450). The reporter triggered a graceful shutdown with a 30-second grace period and watched the node's system DaemonSets — `calico-node`, `kube-proxy`, `node-local-dns` — get replaced onto the very node that was going away:

```
kube-system   calico-node-qlbbl   0/1   Pending   0   54s   worker-pool1-w9dpshqq-stack1
```

That issue rotted without a fix. In January 2024, [kubernetes/kubernetes#122912](https://github.com/kubernetes/kubernetes/issues/122912) described the same loop precisely — the DaemonSet controller "sees that its previously managed workloads were removed, assumes that this is an invalid state and attempts to reconcile it," while kubelet "sees a new Pod being scheduled on the given node that is currently shutting down, and rejects it." It was reproduced on upstream 1.29 and on OpenShift 4.14, and it's been sitting at `priority/important-longterm` since.

The most recent report is the one that finally puts numbers on it. In March 2026, an EKS user upgrading to 1.34 filed [kubernetes/kubernetes#137895](https://github.com/kubernetes/kubernetes/issues/137895) after watching every DaemonSet on a departing node — `ztunnel`, `istio-cni-node`, `aws-node`, `kube-proxy`, the CSI drivers — churn through the loop. Their words: "a single ztunnel pod goes through the 'Create-Reject-Delete' cycle hundreds of times during a single node's removal," and roughly 300 DaemonSet pods were created for one node deletion. That's the cost of one node leaving a cluster.

## Why this matters in production

At the scale of one node, this is noise. At the scale of a real cluster, it's a set of compounding costs — and each of them has its own paper trail.

**Failed pods pile up.** Each rejected pod is a real object with a `Failed` phase and a `NodeShutdown` reason. On a cluster where nodes cycle regularly — spot-heavy or aggressively autoscaled — these accumulate until garbage collection catches up. The pod garbage collector only kicks in once the cluster holds 12,500 terminated pods by default, so in practice they don't go away on their own. [kubernetes/kubernetes#113278](https://github.com/kubernetes/kubernetes/issues/113278) asked in 2022 for a way to make graceful shutdown evict pods without leaving `Failed` objects behind, noting that cleaning them up by hand doesn't scale and that the obvious workarounds — lowering the GC threshold or bulk-deleting failed pods — mask real failures. It was closed as not planned. The reporter's observation that some operators had disabled the feature entirely rather than deal with the fallout is worth sitting with: the workaround for a graceful shutdown feature became *not shutting down gracefully*.

The confusion was bad enough that the pod status itself got renamed. Pods used to show a reason of `Shutdown`, which — as [kubernetes/kubernetes#102820](https://github.com/kubernetes/kubernetes/issues/102820) put it — "may appear as a process of shutting down." It's now `Terminated`, with the message "Pod was terminated in response to imminent node shutdown." Better wording, same pile of objects. Anyone looking at `kubectl get pods -A` still sees a wall of red for a problem that isn't one.

**Alerts fire on non-events.** Most monitoring stacks alert on pod failures, restart counts, or DaemonSet "desired vs. ready" mismatches — kube-prometheus ships `KubePodNotReady` and `KubeDaemonSetRolloutStuck` out of the box. Every graceful shutdown trips them. The community response has been to build tooling that deletes the evidence: [i-see-dead-pods](https://github.com/tyriis/i-see-dead-pods) is a CronJob whose README opens with "pods can remain in a broken state for a long time if graceful shutdown is enabled. This state results in an alert getting fired by kube-prometheus-stack," and it specifically targets pods carrying the "imminent node shutdown" message so it doesn't sweep up genuine failures. [failed-pod-controller](https://github.com/PlayerData/failed-pod-controller) is a controller doing the same job. [MediaMarktSaturn's engineering blog](https://medium.com/mediamarktsaturn-tech-blog/dealing-with-terminated-pods-in-gke-clusters-using-non-standard-provisioning-models-29950281a75) describes building the same thing for GKE spot and preemptible node pools. These are reasonable tools. They are also three separate teams independently writing a garbage collector for a symptom, because the cause lives somewhere they can't reach. And every team that instead learns to ignore the alerts is one step closer to missing the DaemonSet failure that's real.

**Control plane write amplification.** Every iteration of the loop is a pod create, a status update from kubelet, and a pod delete, plus the events attached to each. Multiply by the number of DaemonSets on the node — the EKS report above named four system DaemonSets plus the CSI drivers before counting anything the platform team added — then by the number of nodes shutting down concurrently during an autoscaler scale-in or an OS upgrade rollout. Three hundred pod creations for one node is manageable. Three hundred per node across a fifty-node scale-in, during the window when the control plane is already busy reacting to a topology change, is not something anyone budgeted for. A related 2022 report, [kubernetes/kubernetes#114729](https://github.com/kubernetes/kubernetes/issues/114729), hit the same loop through a Deployment with a match-all toleration and framed it bluntly as a potential denial-of-service vector, since the loop can create pods faster than they're cleaned up. That one was closed as not planned too; the reporter themselves conceded that a match-all toleration on a Deployment is poor practice. DaemonSets don't have that excuse — the not-ready toleration is there by design.

**Rolling updates stall or misreport.** A DaemonSet mid-rollout counts a shutting-down node as one it still needs to update. Depending on `maxUnavailable`, that node can hold up progress on the rest of the fleet, or make a rollout appear stuck when it isn't — which is, again, exactly what `KubeDaemonSetRolloutStuck` is watching for.

**Debugging gets harder.** When something *does* go wrong on a draining node — a CSI driver that didn't unmount cleanly, a log shipper that lost its last batch — the actual failure is buried under a dozen synthetic ones from the recreation loop. And the loop itself has produced its own genuine bugs. [kubernetes/kubernetes#117018](https://github.com/kubernetes/kubernetes/issues/117018) reported `calico-node` pods left in `Completed` after a reboot with graceful shutdown enabled, with the DaemonSet controller never restarting them — the node came back with no CNI until someone deleted the pods by hand. That one was a regression and got fixed, but it only existed because of how kubelet and the controller hand off terminated pods during shutdown. [kubernetes/kubernetes#130490](https://github.com/kubernetes/kubernetes/issues/130490), still open and triaged as a bug, shows a single-replica workload that tolerates not-ready producing seven `ContainerStatusUnknown` pods and one `Error` across a single reboot. When the controller and kubelet are fighting over the same node, the edge cases in that fight become your incident.

None of these is dramatic. That's part of why the problem has persisted for four years: it's a persistent low-grade tax on operators rather than an outage, and each team pays it in their own way, in alert rules and runbooks and cleanup CronJobs and shrugs.

## What's actually missing

The root cause is simple to state. Kubelet has a piece of state — *this node is in a graceful shutdown* — and there is no first-class way to publish it. Workload controllers that would happily do the right thing with that information can't get it.

The clearest evidence of that is the fix that got stuck for lack of it. In May 2026, [kubernetes/kubernetes#139246](https://github.com/kubernetes/kubernetes/issues/139246) reported the scheduler side of the same problem — the scheduler will happily bind not-ready-tolerant pods to a node in shutdown, only for kubelet to reject them — and the reporter sent [a PR](https://github.com/kubernetes/kubernetes/pull/139249) to make the scheduler stop. The only thing kubelet publishes during shutdown is a `Ready=False` condition whose *message* says "node is shutting down" while the *reason* stays `KubeletNotReady`. So the patch had to parse the message string. A reviewer asked, reasonably, for a stable signal — a dedicated condition or reason — before letting the scheduler depend on free text. The PR has been waiting since.

That's the whole problem in one review thread. The state exists. It's in a human-readable string. Nothing machine-readable says it.

What's needed is a Node condition that means exactly that, set by the component that knows (kubelet), cleaned up by the component responsible for the node's lifecycle when kubelet is gone (the Node Lifecycle Controller), and read by the controllers that care. The DaemonSet controller is the first and most obvious reader, but the scheduler PR above shows it won't be the last.

Kubernetes 1.37 laid the groundwork for this with [Node Lifecycle Conditions](https://kubernetes.io/blog/2026/09/09/kubernetes-v1-37-node-lifecycle-conditions/) — a defined vocabulary for describing what's happening to a node, including conditions for drain and graceful shutdown in progress. In 1.37 those conditions are admin-managed; nothing on the node sets them yet. Turning that vocabulary into a live signal is the next step.

## The fix

That next step is what I'm working on with the Node Lifecycle working group and SIG Node. [KEP-6249](https://github.com/kubernetes/enhancements/issues/6249) proposes that kubelet publish `GracefulNodeShutdownInProgress` (alongside `DrainInProgress`) on the Node object at the moment shutdown begins — the same moment it starts rejecting admission — and that the DaemonSet controller stop creating pods for a node in that state, leaving the existing pods for kubelet to terminate in order. The Node Lifecycle Controller clears the conditions if the node never returns.

Alpha is intentionally narrow: one signal, one reader, no new API types, its own feature gate. Priority-tier-aware behavior (letting system-critical DaemonSets stay scheduled while user workloads drain), rollout-budget accounting, and multi-writer ownership of the conditions are all real questions, and they're explicitly deferred to follow-ups so this piece can land. So is the scheduler: once the condition exists, the stuck PR above has the stable signal its reviewer asked for.

The proposal is in review as [kubernetes/enhancements#6351](https://github.com/kubernetes/enhancements/pull/6351), targeting v1.38. If you've hit this in production — especially in ways I haven't described above — I'd like to hear about it, on the PR or in `#wg-node-lifecycle` on Kubernetes Slack.
