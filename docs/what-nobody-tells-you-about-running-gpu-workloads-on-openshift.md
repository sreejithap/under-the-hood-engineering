# What nobody tells you about running GPU workloads on OpenShift

_By Sreejitha Karthik_

Running GPU workloads on OpenShift looks straightforward on paper. The NVIDIA GPU Operator installs clean. The docs are reasonably good. Then you hit production — a regulated enterprise environment, real inference traffic, models that cannot go down — and a different set of problems emerges entirely.

I've run A100 and H100 GPU clusters on OpenShift Container Platform in production at a major financial institution, serving speech and language AI models for a live, customer-facing system. Here's what I wish someone had written down before I started.

## 1. gRPC routing will break silently and the error won't tell you why

This one cost us the most time.

NVIDIA Triton Inference Server uses gRPC as its primary inference protocol. gRPC requires HTTP/2. OpenShift Routes, backed by HAProxy, default to HTTP/1.1 — and they don't fail loudly when a gRPC client connects. They just degrade. You get intermittent failures, dropped streams under load, and error messages that point you in completely the wrong direction.

The fix is not obvious from the OpenShift documentation. You need to configure your Route to use passthrough TLS termination and let Triton handle TLS itself, or configure the Route with the correct annotations to enable HTTP/2 end-to-end. The specific annotation that mattered for us was setting the route to edge termination with HTTP/2 explicitly — without it, HAProxy was silently mangling the protocol negotiation.

What makes this particularly frustrating: it works fine at low traffic. The failure mode only shows up under concurrent inference load, which means you can get through a lot of testing before you see it. If your Triton client is throwing `StatusCode.UNAVAILABLE` or `StatusCode.INTERNAL` under load and everything looks healthy on the server side — this is probably your problem.

## 2. MIG partitioning is operationally harder than the benchmarks suggest

Multi-Instance GPU (MIG) on A100 and H100 is genuinely powerful. The ability to carve a single GPU into isolated slices with dedicated memory and compute — each appearing as an independent GPU to Kubernetes — is exactly what multi-tenant inference needs.

The part that doesn't make it into the benchmarks: MIG profiles are static configuration. Changing them requires draining the node, resetting the GPU, and reconfiguring the MIG manager. In a production cluster where inference traffic is live, that's a real operational cost.

On OpenShift specifically, there are two additional layers of friction. First, OpenShift's Security Context Constraints (SCCs) — more on those in a moment — add complexity to getting the MIG device plugin and manager running with the permissions they need. Second, when a MIG instance gets into a bad state (and eventually one will), the recovery path is not well documented. We built our own runbook for this because we couldn't find one.

The practical advice: spend more time upfront on your MIG profile strategy than you think you need to. Model memory footprint at peak batch size, not average — because the slice needs to handle the worst case without spilling. We ran Whisper and BART on separate MIG profiles sized to their actual peak consumption. Resizing later in production is painful.

## 3. DCGM metrics tell you less than you think — until you combine them correctly

The DCGM Exporter integrates with Prometheus and gives you GPU utilization, memory usage, temperature, SM clock frequency. It looks like complete observability. It isn't.

The first problem on OpenShift: the DCGM Exporter pod needs specific privileged SCC permissions that are not clearly documented for OCP environments. We had silent deployment failures — the pod would start but metrics wouldn't appear — before we traced it back to SCC. In a regulated environment, getting those SCC exemptions also means writing justification documentation for your security team. Budget time for that.

The bigger issue is interpretive. GPU utilization at 10% does not mean your inference platform has headroom. Triton's internal request queue can be saturated — requests waiting, latency climbing — while the GPU itself looks nearly idle. This happens when your bottleneck is preprocessing, tokenization, or the CPU-side request handling, not the GPU compute itself.

The dashboard that actually told us something useful combined three sources: DCGM GPU utilization and memory, Triton's own metrics endpoint (queue depth, inference latency p99, request throughput), and pod-level CPU and network metrics from OCP's built-in monitoring. Only when all three were on the same Grafana dashboard did we have a real picture of platform health.

The metric that became our primary health signal: Triton's `nv_inference_queue_duration_us` (queue wait time in microseconds). When that number climbed, something was wrong — regardless of what GPU utilization was showing.

## 4. OpenShift SCCs are the hidden tax on everything

I've saved this for last because it applies to every point above, and to every GPU component you deploy on OCP.

OpenShift Security Context Constraints are stricter than vanilla Kubernetes Pod Security Standards. The NVIDIA GPU Operator, the MIG Manager, the Device Plugin, and the DCGM Exporter all need elevated privileges to function. On vanilla K8s in a lab, this is a footnote. On OpenShift in a regulated financial environment, each SCC exception requires review, documentation, and sign-off.

This isn't a criticism of OCP's security model — in a regulated environment that strictness is exactly what you want, and it's a core reason we ran on OpenShift rather than vanilla Kubernetes. But it means you cannot treat GPU Operator deployment as a straightforward Helm install. You need to understand what each component needs, why it needs it, and how to articulate that to a security team that is (rightly) skeptical of privileged workloads.

The practical advice: read the SCC requirements for each NVIDIA component before you start the deployment conversation, not after. Having answers ready speeds up the approval process significantly.

## What the other side looks like

When all of this is working correctly — gRPC routing solid, MIG profiles stable, DCGM and Triton metrics aligned, SCCs documented and approved — you have an inference platform that is genuinely robust. Ours has run zero inference failures and zero platform downtime since production launch.

That outcome is achievable. It just requires knowing which walls you're going to hit before you hit them.

---

Happy to go deeper on any of these — especially the gRPC routing solution, which took longer than I'd like to admit. If you're building GPU inference on OpenShift or hitting similar problems on vanilla Kubernetes, I'd like to hear what you've run into.

`#GPUComputing #OpenShift #Kubernetes #MLOps #AIInfrastructure #NVIDIATriton #PlatformEngineering #MLPlatform`
