---
title: "Latency Is the Enemy of Agency"
date: 2026-05-18
draft: false
type: "post"
status: "published"
tags: ["ai", "agents", "llm", "architecture", "hardware", "latency", "kv-cache", "weavertools"]
abstract: |
  Personal note in a series that has been making architectural arguments. The architectural papers describe what the framework is. This piece describes why someone would build it. Specifically: why I built it, what about current AI deployment offends me as someone who actually understands the hardware, and what the architectural commitments come from.
medium_url: ""
canonical_url: ""
---

**Why an Engineer with the Hardware in Front of Him Wants AI to Do Better**

![](/images/blog/latency-is-the-enemy-of-agency/title.jpg)

*Personal note in a series that has been making architectural arguments. The architectural papers describe what the framework is. This piece describes why someone would build it. Specifically: why I built it, what about current AI deployment offends me as someone who actually understands the hardware, and what the architectural commitments come from.*

## What sits on the bench

There's a Threadripper 7960X on my bench. 256 gigabytes of ECC RDIMM. Dual RTX A6000 GPUs, 48 gigabytes of VRAM each, connected by NVLink. An RTX 2000 Ada handling display. Fifteen terabytes of NVMe storage. This is a reasonable engineering workstation by current standards. Not exotic. Plenty of organizations have hardware in this range or substantially more. People run gaming setups that approach this. Small companies have rack-mounted systems that exceed it. Universities have departmental compute that dwarfs it. The hardware I'm describing is unremarkable in the broader landscape of what people actually have available.

What's remarkable is what this hardware is capable of doing that current AI deployment patterns don't let it do.

When I look at the bus speeds, I see hardware operating in microseconds and nanoseconds. PCIe Gen4 transfers complete in single-digit microseconds for the data sizes we're talking about. NVLink between the A6000s operates at nanosecond latencies. Memory access happens in tens of nanoseconds. The components on my bench can communicate with each other at speeds that are essentially instantaneous from the perspective of any cognitive task they might be coordinating to perform.

When I look at how AI gets deployed, I see operations running in milliseconds. Every retrieval round-trips through API boundaries. Every tool call breaks the model's generation stream. Every context switch cools the KV cache. The architecture defaults to network-mediated communication between components even when the components are sitting on the same physical machine, because the deployment patterns were designed for cloud infrastructure where network mediation is unavoidable. The patterns get carried over to local deployment by default, and the local deployment ends up operating at cloud speeds even though the hardware can do six orders of magnitude better.

This bothers me as an engineer. The hardware can do better. The architectural patterns are not letting it.

## What latency does to reasoning

The model on GPU 0 is generating tokens. Maybe 30 to 60 per second for a 70B model at 4-bit quantization on my hardware. That's one token every 16 to 33 milliseconds. Within each of those milliseconds, the model is doing computation against its KV cache, the live fused tensor that represents its current working state. The KV cache is the model's cognition in motion. Every token the model produces extends and updates the cache. Every token requires the cache to be coherent and current.

What happens when the model needs context it doesn't have? In current deployment patterns, this is when the model makes a tool call. The model emits tokens that constitute a function invocation. The framework around the model parses those tokens, executes the call, gets a result, formats it into the model's context window, and resumes generation. The whole operation takes hundreds of milliseconds at minimum, often more. Every one of those milliseconds is a millisecond the model is not generating, not reasoning, not maintaining its cognitive continuity. The KV cache sits frozen waiting for the round-trip to complete. When the model resumes, it has to re-acquire its previous reasoning state and integrate the new context, which is computational work the model has to do because the architecture broke the cognitive continuity.

This is the structural problem. The model's reasoning is the KV cache evolving over time. Anything that interrupts the KV cache breaks the reasoning. Tool calls interrupt the KV cache. Retrieval round-trips interrupt the KV cache. API boundaries interrupt the KV cache. Every interruption is a break in the model's cognitive continuity that has to be repaired through additional computation when the model resumes.

The model doesn't experience this as catastrophic failure. It experiences it as the operating environment, because the operating environment is what the model has been trained against. But the operating environment is enormously wasteful. The model spends substantial fractions of its operational time doing repair work to recover from architectural decisions that broke its reasoning unnecessarily. The wasted computation is invisible because it's normal. It's only visible when you start asking what the alternative would look like.

The alternative is keeping the model fed. The model needs context. The architecture's job is to make sure the context arrives in the model's working state before the model needs it, without breaking the generation stream, without forcing the model to make tool calls to retrieve what it needs. If the context just shows up already integrated into the prompt the model is generating against, then the context the model needs in the moment becomes part of the reasoning loop with no interruption.

This is what autonomic operation means architecturally. The model doesn't decide to retrieve. The model doesn't make tool calls. The model just finds the relevant context present in its working state, the way your cortex finds memories you're not consciously retrieving. The cognitive continuity is maintained because the architecture handles the retrieval beneath the model's conscious decision-making, in components that operate without asking the model anything.

## What the hardware can actually do

Run the math on what's available. The model is generating at one token per millisecond. The PCIe bus between GPU 0 and CPU memory operates in single-digit microseconds for transfers of the data sizes we care about. The NVLink between GPU 0 and GPU 1 operates in hundreds of nanoseconds. CPU memory access happens in tens of nanoseconds. Unix domain sockets between processes on the same machine complete in low microseconds. Everything that needs to happen between the model and its supporting infrastructure can happen at speeds that are essentially instantaneous compared to the model's per-token budget.

A retrieval against an in-memory substrate can complete in microseconds. A graph traversal through a multi-model database can complete in microseconds for typical query shapes. A GNN propagation for a query node entering an existing substrate can complete in hundreds of microseconds even for substantial graphs. Each of these operations is at least three orders of magnitude faster than the model's per-token generation rate. There is enormous margin between what the hardware can do and what the cognitive task needs.

The margin is what enables the architectural pattern that keeps the model fed. A small support model, say a BERT classifier in the Harness running on GPU1, can monitor the model's output continuously, classifying token windows for patterns that indicate retrieval would be valuable. The classifier is small enough that its inference happens in microseconds. When it detects a cue, the substrate query happens in microseconds. The retrieved context arrives in the model's working state before the model would have asked for it, because the model was never going to ask. The model just finds the context present. The cognitive continuity is maintained because the architecture handled the retrieval in background time that the model never had to wait for.

This requires clever coding. The support models need to be well-placed and well-trained. The substrate has to be designed for the access pattern. The IPC has to use the right mechanisms. The component placement has to respect the bus topology. None of this is exotic engineering. It's just engineering that takes the hardware seriously instead of accepting the network-mediated defaults that came over from cloud deployment.

The frustrating part is how recoverable the latency is once you decide to recover it. The hardware is right there. The capabilities are right there. The architectural patterns just need to be designed around the hardware instead of around the network. And the result is a system that operates the way the hardware is capable of operating, with a model that stays in continuous reasoning state instead of being interrupted constantly by architectural decisions that throw away most of what the hardware provides.

A note that matters more now than it would have when I bought this system. The [DGX Spark](https://www.nvidia.com/en-us/products/workstations/dgx-spark/) came out about eighteen months after I built this bench, and I'll admit some regret about that timing. It would have saved me thousands of dollars and given me hardware better suited to what WeaverTools actually needs. The DGX Spark and the clones that have followed, [ASUS](https://shop.asus.com/us/ascent-gx10.html?gad_source=1&gad_campaignid=23841937130&gbraid=0AAAAACzDM82U6H8vxpo96oBxqsK5HY_aS&gclid=CjwKCAjw8arQBhB9EiwAfIKdQiSU4MFktZGjD5BvXecijpQNl-kQBSTTL6H4oycCFAtZtLJ-uugQ5RoCZhsQAvD_BwE) has one, [MSI](https://ipc.msi.com/product_detail/Industrial-Computer-Box-PC/AI-Supercomputer/EdgeXpert-MS-C931) has one, and more are coming, are doing something architecturally interesting that my discrete-GPU setup can only approximate. They put a Blackwell GPU and substantial CPU in the same package with 128 gigabytes of unified memory shared between them. The whole thing sits on your desk and consumes laptop-class power.

What unified memory does for the WeaverTools architecture matters substantively. On my bench, the model lives in GPU VRAM, the substrate lives in CPU RAM, and the data crosses PCIe between them. PCIe transfers happen in single-digit microseconds, which is fast, but it's still microseconds. On unified memory hardware, the model, the substrate, the embedder, the GNN, and the Harness all access the same memory pool. There is no PCIe boundary to cross. The transfers that take microseconds on my hardware take nanoseconds on unified memory hardware. The latency margin the article has been developing gets even larger because the bus boundary that was already fast becomes architecturally invisible.

For a typical agent workload, this matters more than it might initially appear. A ten gigabyte knowledge graph is a substantial substrate that holds a meaningful agent's accumulated state. On the DGX Spark's 128 gigabytes of unified memory, that graph sits alongside the model weights, the inference state, the embedder, the Harness operations, all in the same memory pool with no bus crossings between them. The components communicate at memory speeds rather than at bus speeds. The whole architecture operates in a topology that my discrete-GPU bench can approximate but can't quite match. For engineers reading this article who have one on hand and want to get more out of their hardware, let's talk. I would love to see if WeaverTools would work on an edge device like the DGX Spark. I suspect that it could work quite well for those who want to take their personal assistant to the next level.

## What this implies for agency

Latency is the enemy of agency. Every millisecond the model waits is a millisecond it's not reasoning. Every interruption is a break in cognitive continuity. Every tool call invokes a context switch that has to be repaired through additional computation when the model resumes.

Agency in any meaningful sense requires sustained cognition. The cortex doesn't make a tool call every time it needs a memory. Memories show up in working consciousness because the autonomic substrate is continuously delivering them. The cortex stays in continuous cognitive operation because the cognitive support work happens beneath consciousness. The same principle applies to AI systems that aspire to anything more than transactional response generation. If the model has to interrupt itself to retrieve context, the model is not exercising agency; it is conducting a sequence of transactions.

The architecture that produces agency in AI systems has to keep the model fed. The model has to stay in continuous reasoning state. The cognitive support work has to happen autonomically, in components that operate without the model asking them anything. The substrate has to participate in the model's cognition rather than waiting to be queried. The communication between components has to happen at hardware speeds rather than at network speeds. All of these are commitments that follow from the basic recognition that latency breaks agency.

Most current AI deployment isn't trying to produce agency. It's trying to produce capable transactional response generation, which the current patterns serve adequately for most workloads. The framework I'm building isn't an argument that everyone should abandon what works for what they're doing. The framework is for the workloads where transactional response generation isn't enough, where sustained agency is actually what the workload needs. For those workloads, current patterns structurally fail, and the failure shows up as the model being interrupted constantly by architectural decisions that throw away the hardware capabilities that would have let it sustain agency in the first place.

The workloads that need sustained agency are real. Research assistants that maintain coherent understanding across long projects. Legal AI working with case files that has to accumulate sustained understanding of the case. Medical AI that maintains compliance through architectural commitment. Logistics AI that integrates pattern recognition with strategic decision-making across operational periods. Personal assistants that genuinely know the person they're assisting. Scientific research support that maintains coherent understanding of a research program. All of these workloads need sustained agency. All of them fail when the architecture keeps interrupting the cognitive flow.

## Why I'm doing this

I have the hardware. I can see what it's capable of. I've watched AI being deployed in ways that throw away most of what the hardware provides, because the deployment patterns inherited assumptions from cloud infrastructure that don't apply to local hardware. The patterns work fine for the workloads they were designed for, but they fail for the workloads that need sustained agency, because sustained agency requires the latency characteristics that local hardware provides and that the patterns systematically throw away.

The fix is architectural. Keep the KV cache hot. Keep the model fed. Have support models doing the autonomic work that frees the main model from having to invoke tools. Let the substrate participate in the reasoning rather than waiting to be queried. Co-locate the components so the bus speeds work in the architecture's favor. Use Unix domain sockets and shared memory rather than network protocols for inter-component communication. Design around what the hardware can do rather than around what cloud infrastructure forces.

This isn't a complicated insight once you start looking at the hardware. It's just an insight that the field hasn't been emphasizing because most AI work has happened in cloud environments where the latency characteristics make these architectural choices impossible. For the work I'm doing, which is fundamentally about agents that need sustained agency for organizational and personal workloads, the architectural choices become available the moment the deployment moves to local hardware.

### **WeaverTools, and where the work actually sits**

The framework's name, WeaverTools, is a deliberate position statement. It honors Warren Weaver, whose 1949 introduction to Claude Shannon's *A Mathematical Theory of Communication* introduced the three-level diagnostic foundation, Level A the technical, Level B the semantic, and Level C effectiveness, that anchors this series. Weaver recognized over seven decades ago that significant advancement required tools capable of operating at the levels of meaning and impact (Levels B and C). Yet today's landscape remains largely preoccupied with applying Level B solutions to Level C challenges while ignoring the technical root at Level A. WeaverTools serves as an architectural commitment to Weaver's original call, translating his 1949 insights into a modern engineering response.

WeaverTools has several components in active development. The Memory System component provides the substrate operations the Memory System paper describes, an ArangoDB-backed knowledge graph with inductive GNN integration that makes the substrate live. The Harness component is in active refactor, with the architectural pattern this series describes being implemented incrementally. The reasoning component sits on local hardware running quantized models at full-precision KV cache, with FlashAttention and the inference optimizations the framework's commitments require. The substrate-level components have been through substantial implementation work; the cellular individuation through OS-level user identity and SO_PEERCRED authentication is operational; the basic component interactions are functional.

What's not yet in place is full end-to-end agent operation with all components integrated and validated. The cue classifier as a production-deployed component with measured accuracy is forthcoming. The dream cycle with surprise-gated detail preservation is partially implemented. The integrated bootstrap workflow that takes a fresh agent from foundational document to operational deployment is being developed. The empirical validation results, HeroBench, Tower of Hanoi divergence, the latency benchmarks the calibration article promises, are in progress but not yet ready for publication.

The honest framing of the project's current state is that this is research engineering in active development, not a finished framework being marketed. Substantial architectural work has been done. Substantial implementation work has been done. WeaverTools is being built in public through both the writing in this series and the implementation work happening alongside it. Engineers who recognize the architectural commitments as serving workloads they care about are welcome to follow the development. Code, demonstrations, and empirical validation results get published as they become available rather than waiting for a polished release that delays getting the work in front of the people who would engage with it most substantively.

This article is part of how WeaverTools gets developed. The architectural arguments have to be articulated correctly before the implementation makes sense to anyone but me. The implementation has to substantiate the architectural arguments before the framework can be evaluated against actual operation. Both halves of the work proceed in parallel, with the writing handling the architectural communication and the implementation handling the engineering substantiation. Neither half is sufficient alone; both are necessary for WeaverTools to eventually be something other engineers can adopt for their own workloads.

## What this series is for

There are thousands of organizations with the hardware to deploy autopoietic agents for the workloads where sustained agency matters. There are millions of personal workstations capable of running smaller versions. There are research labs, professional offices, small companies, mid-sized enterprises, all sitting on infrastructure that could run autopoietic agents but lacks the architectural pattern that lets the hardware do that work.

The frontier-AI companies are doing important work at consumer scale that requires their massive infrastructure. The work I'm doing is for everyone else. For organizations and individuals who have hardware that could serve sustained-agency workloads but who don't have the architectural pattern that lets the hardware do that work. The framework is the result of taking that observation seriously and building what comes out of it.

If you have the hardware and you're tired of watching AI throw away what your hardware can do, this series is for you. The architectural papers describe what comes after the recognition this piece tries to land. The hardware is fast. The architecture has to be worthy of it. Latency is the enemy of agency, and recovering the latency the hardware actually provides is what produces agents capable of sustained reasoning rather than capable only of transactional response generation.

## References

Bucy, T. W. (2026). *Nobody Knows What an Agent Is and That Is the Problem.* Stackademic.

Bucy, T. W. (2026). *Your Model Has Humanity's Cortex. It Needs Its Own Hippocampus.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *Productive Friction: The Variable You're Already Trying to Control.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *Why The Architecture Isn't Shipping: Autopoietic Agents in an Autocatalytic World.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *The Agent is a Cell: Cellular Individuation as Architectural Commitment.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *Your Latency Intuitions Are Calibrated for the Cloud.* Todd W. Bucy Research Blog.

Weaver, W. (1949). *Recent Contributions to the Mathematical Theory of Communication.* In Shannon, C.E. & Weaver, W., The Mathematical Theory of Communication. University of Illinois Press.
