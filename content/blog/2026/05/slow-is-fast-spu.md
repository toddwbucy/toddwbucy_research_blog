---
title: "Slow Is Fast: What the SPU Actually Is"
date: 2026-05-18
draft: true
type: "post"
status: "draft"
tags: ["ai", "agents", "llm", "architecture", "inference", "hardware", "spu"]
abstract: |
  Article 1 of a follow-up series examining each component of the agent memory architecture. The SPU is not the bare model — it is what the model becomes when integrated into a specific agent's architecture. This article examines what the SPU actually contains, why peak decoder throughput is the wrong optimization target, and what the computational-topology design space looks like.
medium_url: ""
canonical_url: ""
---

*Article 1 of a follow-up series examining each component of the agent memory architecture. The first series established that an agent is SPU + Memory System + Harness, with the SPU performing the Level B semantic computation that turns input into meaning. This article examines what the SPU actually is at the engineering level, why the conventional optimization target is wrong for agent architectures, and what placement and integration decisions engineers need to make consciously.*

![](/images/blog/slow-is-fast-spu/title.jpg)

## A car you wouldn't want to be in

Putting a Porsche 911 engine into a Volkswagen Beetle creates the ultimate sleeper. A 1973 Super Beetle from the outside, with a flat-six producing more horsepower than the original Bug's transmission was designed to imagine. Done right, the car will have 400HP on a chassis weighing 1,500 pounds and will beat most things at a light and hold its own on a road course. Done wrong, the car becomes a list of failures waiting for stress to expose them in order.

I knew a soldier in Germany in the 1990s who did this conversion well enough to ship the car back to the US. The story matters because of what he didn't do. He didn't just swap engines. He rebuilt the transaxle to handle the torque the original transmission was never designed to receive. He upgraded the brakes proportional to the new top speed, because stopping a 1,500-pound car going 120 miles per hour is a different problem than stopping the same car going 80. He adjusted the suspension geometry to match the new weight distribution and the new power band. By the time he was done, he had essentially built a new car using the Bug's body and the 911's drivetrain, with everything in between rebuilt to match.

The guys who do this conversion wrong drop the engine in and head for the track. The transmission grenades on the first hard run. Or the brakes fade halfway through the second lap. Or the suspension can't hold a line through a hard corner because the geometry assumes a different weight distribution and a different torque curve. The car isn't fast. The car is broken in a specific pattern. The breakage is invisible until the system gets pushed, and then it's all that's visible.

The model is the engine. The agent is the car. Most of the industry is currently dropping 911 engines into Bugs and wondering why the agents keep blowing transmissions.

## The SPU is not the model

The agent architecture defines three components: the SPU performs Level B semantic computation, the Memory System maintains the substrate of accumulated state, the Harness coordinates them autonomically into unified action. These are real architectural elements with real engineering surfaces, and the SPU specifically is where the most consequential category mistake gets made.

The SPU is not the model. The SPU is what the model becomes when it's integrated into a specific agent's architecture.

A bare model can do remarkable things. It produces fluent text. It reasons over complex input. It integrates structured context into coherent output. These are real capabilities and the field has gotten very good at producing them. None of this makes the model an SPU, because the SPU isn't a capability; it's an architectural position. The SPU is the role the model plays within the agent, and playing that role requires more than the model's raw capabilities. It requires interfaces to the Harness. It requires conventions for how retrieved context arrives and gets integrated. It requires a foundational configuration that grounds the agent's identity across sessions. It requires pacing that matches what the rest of the architecture can sustain.

A model with none of these things is not an SPU. It's an engine sitting in a crate. Powerful, capable, ready to do work, but not integrated into anything that can use what it does. The work of building the SPU is the work of integration, and integration is where the architectural decisions actually live.

This distinction matters because the conventional commercial framing of AI products conflates the model with the agent. Every chatbot is sold as if its capabilities were the model's capabilities. Every benchmark measures the model in isolation. Every model release is positioned as if a better model would automatically produce a better agent. None of this is true if the agent's architecture treats the model as one component embedded in a larger system. The model's capabilities are necessary. They are not sufficient. The architecture has to do work the model alone cannot do, and the SPU is where that work first becomes visible.

## The integration depth spectrum

The SPU exists on a spectrum from lightly integrated to heavily integrated with the rest of the agent's architecture. Both ends are defensible engineering positions. The choice between them is a real architectural decision with real consequences.

Light integration means the model is essentially off-the-shelf, configured through system prompts and interface conventions but not modified at the weights level. The agent's identity lives in the foundational configuration document and in the Memory System's accumulated state. The model serves as a swappable cortex. When a better model becomes available, you replace the weights without disturbing the rest of the architecture. The integration is shallow because the deployment doesn't justify deeper investment. For many agent deployments, especially shorter-lived ones or those with narrower scope, this is the right choice.

Heavy integration means the model has been fine-tuned to operate against the specific Harness and Memory System the agent uses. The model has learned the conventions for how retrieved context arrives. It has internalized the patterns that the Harness uses for cue detection. It has been trained against the specific Memory System's representational structure. When you upgrade the underlying model architecture, you have to retrain the new model within the framework to recover the operational fit. The integration is deep because the deployment justifies it. For long-lived agents accumulating significant operational value over months and years, this is what the natural trajectory looks like.

Most real deployments will sit somewhere on the spectrum rather than at the extremes. A medium-integration agent might use a base model plus foundational configuration plus custom retrieval integration, without the full weight-level coupling but with more than just system-prompt configuration. The choice depends on deployment lifetime, scope, value-of-accumulated-state, and the cost of maintaining the integration as model architectures evolve.

What stays constant across the spectrum is that the SPU is not the bare model. Even the lightest integration involves interface design, configuration patterns, and operational pacing that no off-the-shelf model provides. The model is the substrate the SPU is built on. The SPU is the architectural element the agent actually contains.

This also clarifies what "swappable" means. The February foundation established that the SPU is infrastructure rather than identity, the agent's identity lives in the Memory System and the foundational configuration, so the agent doesn't die when you upgrade the model. This is true, but it doesn't mean the upgrade is free. For lightly-integrated SPUs, the upgrade is mostly a configuration adjustment. For heavily-integrated SPUs, the upgrade requires retraining within the framework. The Memory System preserves the agent's accumulated state across the transition. The integration work has to be redone for the new model. The agent survives, but the SPU has to be rebuilt around the new substrate.

## What the SPU actually contains

The SPU isn't a single computational component. It's a composite of multiple elements that together perform the Level B work the architecture requires.

The main inference model is the centerpiece. The decoder that takes prompts and produces tokens. The transformer that does the semantic computation the rest of the architecture coordinates around.

The embedder is the second component most deployments will need. The model that converts text to vector representations. The embedder runs whenever new content is written to the Memory System, converting raw text into the vectors that get stored alongside the graph structure. It runs again whenever the Harness queries the Memory System, converting cue text into vectors that match against stored representations. Without the embedder, the Memory System has no semantic surface to retrieve against.

Some deployments include additional Level B components. Cross-encoders for query reranking. Specialized models for particular semantic operations. The architecture supports these without prescribing them. The core SPU composition is decoder plus embedder. Other components, the Harness's cue detection classifier, the Memory System's representation-maintaining GNN, also do work that looks computationally like Level B operations, but they belong architecturally to the components they serve. Subsequent articles in this series address each of those components in turn.

The point worth holding clearly is that the SPU's computational requirements are not one model running inference. The SPU is the decoder and the embedder cooperating to do the semantic work the agent's main reasoning requires. Engineers thinking about deployment have to think about both of them, not just the headline decoder.

## Slow is fast

Here is where the conventional inference-optimization habit produces the wrong answer.

The conventional engineering instinct is to maximize decoder throughput. Faster tokens per second is better. Every benchmark measures this. Every optimization guide treats peak throughput as the target. Quantization, speculative decoding, attention optimization, batch management. All of these get tuned to make the decoder run as fast as the hardware allows.

For an autonomic agent architecture, this optimization target is wrong. The architecture doesn't care about peak throughput. It cares about continuity.

The decoder needs to generate continuously at a rate the rest of the system can sustain. Not as fast as possible. As fast as the Harness, embedder, and Memory System can keep up with. A decoder running at 200 tokens per second is worse than a decoder running at 50 tokens per second if the support systems can only keep up with 50. The fast decoder will outrun cue detection, miss retrieval windows, and force the architecture into the synchronous tool-call pattern the framework exists to avoid. The slow decoder, paced to what the system can sustain, preserves the continuous generation property the architecture requires.

![](/images/blog/slow-is-fast-spu/image1.jpg)

This is the Porsche-Bug principle. The 911 engine running at full output destroys the transaxle it's bolted to if the engine is not tuned properly. The same engine de-tuned to match a properly-rebuilt transaxle produces a faster car overall, because the car keeps running. Peak engine output isn't the same as peak car performance. Peak decoder throughput isn't the same as peak agent performance.

The KV cache continuity argument makes this concrete. The decoder's working state lives in the KV cache, which accumulates as generation proceeds. If the decoder stops to wait for a synchronous retrieval, the cache freezes but the cognitive process pauses. When generation resumes, the decoder is operating on stale working state and has to rebuild context momentum. The pause has cost beyond its measured latency, it has disrupted the continuous cognitive flow the architecture depends on.

The autonomic pattern preserves the cache and the flow specifically because the decoder never stops. Retrieval happens in parallel with generation. Context arrives at natural integration points without the decoder pausing to wait for it. The KV cache stays hot. The cognitive process doesn't gap.

This means engineers tuning the SPU should not be tuning the decoder for maximum throughput. They should be tuning the decoder to operate at the rate the rest of the architecture can sustain without falling behind. Quantization choices, parallelism patterns, batch sizing, attention optimization, all of these affect the decoder's rate. The right setting isn't whatever produces the fastest tokens per second. It's whatever produces continuous generation at a pace the system can pace alongside.

Slow is fast. The phrase compresses the insight enough to be useful in conversation. The principle behind it is that agent performance is system performance, not component performance, and the system's sustainable rate is set by the slowest component that has to keep up with the decoder. Operating above that rate doesn't make the agent faster. It breaks the architecture.

## Computational Topology

The physical placement of SPU components is a critical architectural decision. These choices directly influence both the economic cost of deployment and the achievable performance ceiling. Because the architecture accommodates various topologies, selecting the optimal configuration requires balancing specific workload demands against available hardware resources.

To illustrate this, consider a midrange workstation built specifically for this architectural use case:

- AMD Threadripper 7960X (24 cores / 48 threads)

- ASRock TRX50 Motherboard

- 256GB ECC RAM

- Dual RTX A6000 GPUs (48GB VRAM each) linked via NVLink

- RTX 2000 Ada (16GB VRAM) dedicated to display

- High-speed storage: 2TB Gen5 RAID0, 2TB Gen5 RAID1, and 4TB Gen4 RAID10

- 16TB SATA HDD and 1600W PSU

This isn't a standard consumer rig or a gaming machine; it is a professional-grade system with significant but finite resources. While this build is substantial, the underlying architectural principles should remain applicable to consumer-grade or edge hardware if properly optimized.

The placement decisions on this hardware shape what the agent can do.

GPU 0 hosts the main decoder model. Whatever size and quantization fits within 48GB VRAM (I can confirm that Qwen 30b Coder Q6 can load a 129k seq_len on this card quite comfortably) The decoder gets this GPU's full compute time. No contention with other models. The KV cache, the activation memory, and the model weights together fill the VRAM budget. This is where the headline inference work happens.

The embedder is hosted on GPU 1. Given that a capable sentence transformer typically requires only a few hundred megabytes, it fits easily within the 48GB VRAM limit, leaving significant overhead. This setup fulfills the SPU's architectural requirement: the embedder must operate on a separate GPU from the decoder to avoid compute contention during primary inference tasks. The surplus VRAM on this unit is reserved for Harness and Memory System components, whose specific placement and computational functions are detailed in later articles. While this dedicated allocation is ideal, a constrained hardware environment might allow running a small, performant embedding model on the CPU or GPU 0, provided the system can manage the resulting performance trade-offs.

The Harness itself runs on CPU. So does the Memory System's database storing the knowledge graph. Both have access to the full 256GB of system RAM, which is enough to hold a substantial agent's accumulated state. The CPU's 24 cores handle the orchestration logic and the substrate operations that don't need GPU acceleration.

Communication between the two GPUs happens over NVLink at 600GB per second of bidirectional bandwidth. At that speed, cross-GPU communication is effectively free for the data volumes involved, embedded query vectors flowing toward the components that consume them, retrieved context flowing back to the decoder. Communication between either GPU and the CPU happens over PCIe, which is slower but adequate for the operations that need to cross that boundary.

This is one configuration. It works because every component has access to compute that matches its requirements, the data flows naturally between components without forced bus transfers on the critical path, and the decoder gets its full GPU without contention from other models.

Other configurations are defensible. A single-GPU setup would put the decoder on the GPU and everything else on the CPU, accepting that the smaller models run slower on CPU but eliminating the need for a second GPU. A tensor-parallel setup would spread the decoder across both GPUs, accepting per-token inter-GPU latency in exchange for being able to run larger models than fit on one GPU. A multi-GPU setup with three or more GPUs could give each support model its own dedicated hardware. Larger configurations could distribute across machines, paying network latency for the components separated across hosts.

Each configuration optimizes for different things. Each has costs. The framework doesn't dictate one answer. What it does dictate is that the topology is itself a design space, and engineers building deployments should be making the placement decisions consciously rather than accepting whatever defaults their inference framework happens to provide.

The "slow is fast" principle interacts with topology in a useful way. A configuration that runs the decoder at maximum possible throughput but starves the support systems is the wrong choice for this architecture even if it benchmarks well on token-per-second measures. A configuration that runs the decoder at moderate throughput while ensuring the support systems can keep up is the right choice even if it benchmarks lower on the decoder in isolation. The right benchmark is whether the agent works. Whether bindings form properly. Whether retrieved context arrives in the window the decoder allows. Whether the autonomic property is preserved across extended operation. These are system-level properties that component benchmarks don't measure.

![](/images/blog/slow-is-fast-spu/image2.jpg)

## The work that remains

The SPU article in this series has done what it can at the conceptual and architectural level. The SPU is not the bare model. The SPU includes the decoder plus the supporting Level B components; embedder, classifier, possibly others. The integration with the rest of the agent's architecture happens on a spectrum from light to heavy coupling. The optimization target is continuity, not peak throughput. The topology is a design space, not a fixed property. Slow is fast.

What this article has not done is specify a particular reference implementation. The midrange workstation example is illustrative, not prescriptive. The principles map onto other hardware configurations. The integration choices map onto different deployment requirements. The SPU's specific composition will depend on what the agent has to do and how long it has to do it.

The next article the series examines is the Memory System, the substrate that holds the agent's accumulated state, structured as a knowledge graph with vector and document layers unified in a single query surface, maintained dynamically by an inductive graph neural network. Memory is not a database. The article develops why, and what the substrate has to be to function as memory rather than as storage.

The final article in this series examines the Harness, the autonomic coordination layer that makes the SPU and the Memory System work together as a unified agent rather than as two disconnected systems. The Harness is the component the field most consistently leaves out, partly because the field doesn't have vocabulary for what the Harness actually is. The conventional framing treats whatever sits around the model as configuration.

The series taken together gives engineers a complete picture of what each component has to be and what the design decisions across the architecture look like. None of the articles prescribe a specific implementation. All of them establish the principles that valid implementations have to satisfy.

My friend's Porche-Bug taught me that the engine isn't the car. The same lesson applies to agent architecture. The model isn't the agent. The SPU isn't just the model. Performance is a system property. Continuous operation matters more than peak throughput. Integration is where the engineering lives. The work of building an agent that actually performs is the work of building every component to match every other component, with the integration as carefully designed as the components themselves.

Most of the field is still dropping 911 engines into Bugs. The opportunity for engineers willing to do the integration work is correspondingly large.

## References

Bucy, T. W. (2026). *Nobody Knows What an Agent Is and That Is the Problem.* Stackademic.

Bucy, T. W. (2026). *Your Model Has Humanity's Cortex. It Needs Its Own Hippocampus.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *Productive Friction: The Variable You're Already Trying to Control.* Todd W. Bucy Research Blog.

Bucy, T. W. (2026). *Why the Architecture Isn't Shipping: Autopoietic Agents in an Autocatalytic World.* Todd W. Bucy Research Blog.
