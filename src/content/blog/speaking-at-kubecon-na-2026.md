---
title: "I'm Speaking at KubeCon + CloudNativeCon North America 2026"
description: "I'll be in Salt Lake City this November to talk about achieving cross-cloud portability with the Kubernetes Resource Orchestrator (kro)."
pubDate: 2026-08-13
---

I'm speaking at **#KubeCon + #CloudNativeCon North America**, November 9–12 in Salt Lake City, Utah!

![KubeCon + CloudNativeCon North America 2026, November 9-12, Salt Lake City, Utah — "I'm Speaking" announcement graphic. Register now + join me.](/images/kubecon-na-2026-speaking.png)

My session is titled **"Achieving Cross-Cloud Portability with the Kubernetes Resource Orchestrator"**, about using kro to collapse sprawling, cloud-specific infrastructure into standard, portable Kubernetes APIs — no bespoke Go operators required.

- ⛅ **Register now + join me:** [https://bit.ly/4wAxjwr](https://bit.ly/4wAxjwr)
- 🗓️ **Explore the schedule:** [https://bit.ly/4z9rOa8](https://bit.ly/4z9rOa8)

## What the talk is about

The rise of highly complex workloads — most notably GenAI — has driven incredible innovation in managed services and specialized infrastructure-as-code templates. Those tools genuinely reduce developer friction. But organizations operating across multi-cloud or hybrid environments hit a different problem: maintaining a unified, highly portable deployment model that standardizes on native Kubernetes APIs.

When teams try to pull these workloads back into pure, cloud-agnostic Kubernetes, they run into a wall of complexity. Orchestrating the necessary components — highly specialized Deployments, Argo Workflows, involved Prometheus configurations — becomes an unsustainable burden unless someone resorts to writing and maintaining bespoke Go operators.

This talk shows how maintainers and platform builders can use the [Kubernetes Resource Orchestrator (kro)](https://kro.run) to encapsulate that complexity behind highly portable, standard Kubernetes APIs. I'll cover three things:

- **Architecting for portability.** How to encapsulate sprawling, multi-resource infrastructure into a single, cloud-agnostic kro template that behaves consistently across any conformant cluster.
- **The operator-less platform.** How platform teams can build and extend self-service infrastructure catalogs natively in Kubernetes without writing or maintaining custom Go controllers.
- **The API experience.** Delivering a unified, 10-line YAML interface to end users that abstracts away the underlying complexity — keeping the developer experience pristine while eliminating cloud lock-in.

You should leave with concrete strategies for using kro to improve cloud-agnostic patterns in your workloads, reduce resource orchestration complexity, and cut down on the bespoke software controllers you have to keep alive.

## Why I think this matters

As platform teams scale increasingly complex workloads across diverse multi-cloud and hybrid environments, a consistent deployment model is paramount. kro works as a universal abstraction layer that *complements* the cloud-specific innovation happening everywhere else rather than fighting it.

By encapsulating sprawling infrastructure into standardized, portable Kubernetes APIs, organizations can build deeply integrated, self-service platforms — and the Kubernetes community can deliver a frictionless, unified developer experience across any environment. That means drastically fewer custom Go operators to maintain, and a lot more flexibility in where and how your architecture runs.

## Come say hi

If you're going to be in Salt Lake City, I'd love to talk kro, platform engineering, or portability war stories. And if you've been following my [kro Plugin for Headlamp](/blog/kro-headlamp-plugin-alpha/) work, this session is very much the other half of that story — the plugin makes kro's portability visible, and this talk is about designing for it in the first place.

See you in November! ⛅
