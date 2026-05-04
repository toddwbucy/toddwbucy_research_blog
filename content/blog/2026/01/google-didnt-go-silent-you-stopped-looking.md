---
title: "Google Didn't Go Silent. You Stopped Looking."
date: 2026-01-29T09:00:00-06:00
draft: false
type: "post"
status: "published"
tags:
  - nested-learning
  - titans
  - transformers
  - google
  - machine-learning
  - ml-infrastructure
  - pytorch
  - jax
abstract: |
  A response to Delanoe Pirard's "Transformers Are Dead" and "A 170M Model
  Beats GPT-4." Titans wasn't a standalone architecture. It was the first
  chapter of a seven-paper research program that redefines neural
  computation as nested optimization. The code was never released because
  you cannot ship a paradigm that requires infrastructure that doesn't
  exist yet.
medium_url: ""
---

![](/images/blog/google-didnt-go-silent/image1.jpg)

*A response to Delanoe Pirard's "Transformers Are Dead" and "A 170M Model Beats GPT-4"*

In December 2025, Delanoe Pirard published two articles that crystallized a question the ML community had been asking for a year: If Titans is so good, why has Google gone silent?

The answer is this: Titans is one component of a seven-paper research program, not a standalone architecture. Google didn't go silent. The community stopped looking for the rest of the papers.

Pirard's first article, "[Transformers Are Dead. Google Killed Them, Then Went Silent](https://blog.delanoe-pirard.com/transformers-are-dead-google-killed-them-then-went-silent-a379ed35409b)," walked through the Titans architecture with impressive rigor and ended with a direct challenge: release the code or watch the narrative shift from breakthrough to vaporware. His second, "[A 170M Model Just Beat GPT-4](https://ai.gopubby.com/google-titans-neuroscience-ai-memory-explained-69b319f1f516)," went deeper into neuroscience, math, and open questions. Both articles crystallized the real frustration many of us felt. The community looked for the single-paper discovery of a Transformer killer, but we were instead given a seven-paper dissertation that seeks to redefine how we think about neural computation itself.

This article is about what that dissertation actually says, why it matters, and why the tools we have today cannot build what it describes.

## The Discovery

I read the Titans paper when it came out in early 2025, like everyone else. I waited for the promised repository. It never came.

A few months later, another paper crossed my radar: *Atlas: Learning to Optimally Memorize the Context at Test Time.* The phrase "memorize at test time" caught me; it was the exact language Titans used. I opened the paper and saw a familiar name on the author list: Ali Behrouz. He was on the Titans paper. So was Vahab Mirrokni. So was Peilin Zhong.

I looked up Ali's LinkedIn. PhD student at Cornell. Student Researcher at Google. I thought: Am I looking at dissertation chapters? I set up a cron job to poll arXiv daily for publications from these authors.

![](/images/blog/google-didnt-go-silent/image5.jpg)

Within months, the alerts started. Seven papers. One team. One paradigm. Published incrementally across twelve months, each assuming you'd read the others:

| Paper           | Published     | Role in the Program                         |
|-----------------|---------------|---------------------------------------------|
| Titans          | January 2025  | WHERE memory goes in the architecture       |
| Miras           | April 2025    | WHAT memory module designs are possible     |
| Lattice         | April 2025    | HOW TO COMPRESS memory efficiently          |
| Atlas           | May 2025      | HOW MUCH capacity, and HOW WELL to optimize |
| TNT             | November 2025 | HOW TO TRAIN deep memory at scale           |
| Trellis         | December 2025 | Lattice evolution with two-pass compression |
| Nested Learning | December 2025 | HOW MANY levels, AT WHAT FREQUENCIES        |

This taxonomy doesn't appear in any single paper. I reconstructed it by tracing conceptual dependencies across all seven. The "silence" after Titans was the team publishing the rest of the program. The code never appeared because a single Titans implementation was never the point.

Here is what Pirard's five critical problems look like when you have all seven papers instead of one:

| Pirard's Problem                 | Paper That Addresses It    | How                                                                                     |
|----------------------------------|----------------------------|-----------------------------------------------------------------------------------------|
| No official code                 | Nested Learning (capstone) | The implementation IS the unified program. You can't release chapter one.               |
| Chunking degrades performance    | TNT (November 2025)        | Hierarchical memory: 1 global + N local modules. 17x faster, better perplexity.         |
| Test-time training overhead      | TNT + Lattice              | Chunkwise parallelism + O(m*d) per token instead of O(d³). Quarter-memory beats full.   |
| Scale unknown (only 760M tested) | Atlas (May 2025)           | Scales context to 10M tokens. Proves DeepTransformers strictly generalize Transformers. |
| Synthetic benchmarks only        | Miras (April 2025)         | Systematically tests dozens of variants. Attention-free models match or beat hybrids.   |

That question mark in Pirard's limitation column has an answer. The limitation of Titans-alone is that it is one-seventh of a design.

## What Nested Learning Actually Is

This is where the community's understanding stops. People know Titans have a memory module that learns during inference. What they don't know, because the arXiv preprint only dropped in December and it takes all seven papers to see that, it's not a better Transformer. It's not attention plus memory. It's a claim that the concepts we use to think about neural networks are wrong, and that the apparent diversity of deep learning architectures is an illusion.

Here is that claim, as precisely as I can state it.

### Everything Is Nested Optimization

In conventional ML, architecture, optimizer, and memory are three separate concerns. You pick an architecture (Transformer, RNN, MLP). You pick an optimizer (Adam, SGD). Memory is either the KV cache, or an external module you bolt on. These are independent choices from independent menus.

Nested Learning says they're the same thing observed at different timescales.

An MLP is an optimization loop that runs once per input; one step of gradient descent on its internal parameters, frozen between inputs. Add a second frequency level to that loop (let the parameters update across inputs) and the MLP becomes an RNN. Add a third level (let it track gradient statistics across updates) and you've reinvented momentum. Add a fourth (let it learn *how* to update itself) and you have a self-modifying learning module.

![](/images/blog/google-didnt-go-silent/image6.jpg)

This isn't a metaphor. It's the paper's central mathematical result. Definition 1 of the capstone paper defines associative memory as an operator M(.) that maps keys to values by minimizing an objective over the data. That's it. An MLP qualifies; it maps inputs to outputs by minimizing a loss. An attention layer qualifies; it maps queries to values by minimizing a retrieval objective. Adam's momentum qualifies; it maps gradient history to update directions. They're all the same primitive at different frequencies. The diversity of deep learning architectures (Transformers, RNNs, state-space models, memory networks) comes from observing the *solutions* of nested optimization problems at different timescales, not from fundamentally different computational mechanisms.

### There Is No Training. There Is No Inference.

This is the claim that breaks people's mental models, so let me be precise.

In a conventional neural network, there are two distinct modes of operation. During training, parameters update via backpropagation. During inference, parameters are frozen and the model produces outputs. You train the model, you deploy the model, and the deployed model never changes. model.train() and model.eval() are not just API calls; they encode a worldview: that learning and using are fundamentally different activities.

Nested Learning says there are only two states: receiving input, or isolated. When the model receives input, *all* frequency levels process it. The fast levels update on every token. The slow levels update on every chunk, or every sequence, or every thousand sequences. The "training" phase is just the slowest frequency levels processing their context; the entire dataset. The "inference" phase is the same computation, just without the slowest levels actively updating (because they've already processed their context window). The forward pass is identical. The model doesn't know what phase it's in.

This means: when you deploy a Nested Learning model and feed it input, the fast-frequency levels are learning from that input. The model adapts to the context it's reading. Not through fine-tuning or not through retrieval, but through the same mechanism it used during "training," running at a faster timescale. The paper calls this where "in-context learning naturally emerges". It's not a special capability bolted on; it's what happens when you have multiple frequency levels and the fast ones keep running. This adaptation is bounded by the model's capacity and the compression tradeoffs of its memory update rules, but within those bounds, it is genuine learning from context.

### There Is No Separate Memory System

In every community Titans implementation I've examined, memory is a separate class (MemoryModule, NeuralMemory, LongTermMemory) bolted onto the side of a Transformer. The memory has its own lifecycle, its own forward method, its own optimizer. Information flows between the model and its memory through an explicit interface. The model is one thing. The memory is another thing next to it.

The research program rejects this architecture entirely. In Nested Learning, every parameter in the model that changes in response to input is already storing memory. The fast-frequency parameters (the ones that update every token) remember what just happened. The slow-frequency parameters (the ones that update every thousand tokens) remember what's been true across the entire context. There is no "memory module" sitting next to "the model." The model's weights *are* the memory; distributed across every layer, organized by how frequently they update.

Think about what this means in practice. In the bolt-on approach, you have three separate engineering problems: what goes into memory (encoding), what comes back out (retrieval), and when to forget (decay). You solve each one independently. In the NL approach, all three collapse into a single problem: define the update rule for each frequency level. Encoding happens during the forward pass. Retrieval happens during the forward pass. Forgetting is built into the update rule's retention mechanism. The papers spend hundreds of pages cataloging different update rules (delta rule, MONETA, YAAD, Lattice OSR) because the update rule is the memory system. Choose a different rule and you get a different kind of memory, with different capacity, different compression properties, different forgetting behavior. But it's never a separate module. It's always the model's own parameters doing the remembering.

![](/images/blog/google-didnt-go-silent/image2.jpg)

### The Frequency Continuum

Pirard came closest to understanding this through his Complementary Learning Systems (CLS) framework. CLS theory from neuroscience says real learning requires two coordinated systems at different timescales: fast (hippocampus) and slow (neocortex). Pirard maps this onto Titans: attention is fast, memory is slow. He's correct, but he stops one step short.

Nested Learning goes further: even "two timescales" is a simplification. The full picture is a continuum. One level updates every chunk; it reacts to everything. The next updates every 4th chunk; paragraph-scale patterns. The next every 16th; section-scale. The slowest updates once across the entire context window. They're not layers stacked on top of each other. They're frequency levels operating in parallel, like radio bands tuned to different timescales of the same signal.

The paper calls this a Continuum Memory System (CMS). The k=4 implementation samples four points from the frequency spectrum. You could sample eight, or sixteen, or a hundred. The framework doesn't care. There's no boundary between "fast memory" and "slow memory." There's a dial. And the system operates at every position on that dial simultaneously.

### Optimizers Are Not Tools. They Are Knowledge.

In conventional ML, an optimizer is a utility. You pick it from a menu. Its internal state (momentum terms, second moments) is bookkeeping. Reset the optimizer and you lose nothing of value.

Nested Learning says the optimizer IS a learning module. Its momentum terms store knowledge about the gradient distribution and loss landscape. That knowledge helps it better update the weights it serves. Resetting the optimizer is not clearing bookkeeping, it's erasing learned knowledge about data distribution. The optimizer's frequency structure must match the architecture: a single-frequency AdamW applied uniformly to a multi-frequency CMS is a category error. Parameters updating every token need aggressive momentum. Parameters updating every thousand tokens need momentum that accumulates over a completely different timescale. Treating them identically is like watering a cactus and a fern on the same schedule, one of them is going to die.

This reframing cost me four months of development time. The paper was telling me the optimizer is not a plug-and-play choice. I couldn't hear it.

## Why This Matters

Three consequences fall directly out of the paradigm.

**Models that learn while you use them.** A deployed NL model processes your input through all its frequency levels. The fast levels learn from your prompt in real time. The model gets smarter within a given context, but the Nested Learning paper is explicit about the limits. Catastrophic forgetting is not solved. The authors write that it is "a natural consequence of compression, where the limited capacity of the network forces the model to forget so that it retains capacity for new information." The CMS design manages this tradeoff through its multi-frequency structure: fast levels forget quickly to stay responsive, slow levels retain longer but eventually compress too. It is a sophisticated management system for entropy, not a magic pill. The boundary between "pre-training" and "inference" dissolves, but the boundary between "remembers" and "has forgotten" does not.

**The economics of precision over scale.** Atlas proves that polynomial feature mappings give memory capacity that scales as O(d_k^p), the p-th power of key dimension. This means a smaller model at full numerical precision can outperform a much larger model at reduced precision, because the polynomial terms that encode memory vanish below fp16's representable range. The economic argument is stark: a 10B parameter model at fp32 with full polynomial capacity may outperform a 100B parameter model at fp8 that has lost it. Smaller, more precise, and cheaper to serve.

**A composable design space, not a single architecture.** Miras maps the entire space of memory update rules. Lattice and Trellis provide compression variants. TNT provides training infrastructure. Atlas provides capacity theory. These are not competing architectures, they are interchangeable components in a modular design space. Pick an update rule (Titans delta rule, MONETA, YAAD, Lattice OSR). Pick a composition pattern (MAC, MAG, MAL). Pick a training strategy (TNT chunkwise, two-stage). Pick a frequency schedule (CMS with k levels). The research program didn't publish one architecture. It published a toolkit. The number of valid configurations is combinatorial.

![](/images/blog/google-didnt-go-silent/image3.jpg)

## Why Our Tools Cannot Build It

Here is where the problem becomes concrete. The math works. The Lattice paper includes a complete JAX implementation in its appendix. TNT benchmarks its throughput against Flash Attention. The researchers built these models in JAX, which doesn't even have a model.train()/model.eval() distinction. So the claim is not that the frameworks can't express the math. They can. We proved it. The papers' own authors proved it.

The problem is twofold. First, the dominant framework in the community, PyTorch, pushes you toward the wrong paradigm through its defaults, its tutorials, and its API design. Every assumption you absorb by writing import torch is an assumption Nested Learning rejects. Second, and more critically, the entire serving and deployment ecosystem assumes a kind of model that Nested Learning is not. You can build it. You cannot deploy it. And the gap between those two is where the real engineering crisis lives.

### model.train() and model.eval() Encode the Wrong Ontology

PyTorch's most basic workflow encodes a worldview:

```python
model.train()    # Learning mode: gradients flow, dropout active
# ... training loop ...
model.eval()     # Inference mode: gradients blocked, dropout disabled
```

This is not an API preference. It is an ontological claim: that learning and inference are different activities requiring different computational behavior. NL says there are no such modes. The forward pass processes context; always and unconditionally. The inner loop, the mechanism the papers are named for, runs during "inference" because that's when the model is supposed to learn from context. Gate it behind "if self.training" and you've disabled the defining feature of the architecture.

There is a 43-million parameter model sitting on our workstation right now that demonstrates this in miniature. It is an Atlas-MAG model, a hybrid architecture combining sliding window attention with deep polynomial memory, trained on educational text for 8,800 steps. It scores 85.9% on needle-in-a-haystack retrieval at 2,048 tokens. It works.

But only if you lie to PyTorch.

The model's defining capability is test-time learning: a gradient-based inner loop that updates memory parameters as each new token arrives during inference. This is the mechanism that lets a 43-million parameter model retrieve information from contexts far longer than its 512-token attention window can see. Without it, the model is blind beyond that window. With it, the model can reach back across the full sequence.

The gate that controls this inner loop is a single conditional:

```python
if self.ttl_enabled and self.training:
```

self.training is a PyTorch flag. When you call model.eval(), the standard, correct, universally-recommended practice for inference, PyTorch sets self.training = False. The TTL inner loop never fires. The model's long-range memory goes silent. Context beyond the attention window vanishes.

We tested this directly. Same model, same weights, same prompts. In eval mode: standard inference, no memory updates. In train mode: the TTL inner loop fires, memory updates at every position, 1.7 to 5 times slower than the outer, and the outputs change. On "The capital of France is," the model generated different continuations depending on whether the inner loop was active. Seven tokens of context. Already different behavior.

To activate test-time learning during inference, you call model.train(). This tells PyTorch the model is training when it is not. It disables inference optimizations. It keeps dropout active (if any exists). It breaks every assumption that serving infrastructure makes about frozen weights. This is not a bug in the implementation. This is the correct behavior for a model whose parameters are supposed to change during inference. The assumption baked into PyTorch's API, that "training" and "inference" are different things and Nested Learning says they are not.

### torch.optim Assumes a Single Global Optimizer

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=3e-4)
```

One optimizer. All parameters. One learning rate. One momentum schedule. This is how every PyTorch tutorial starts. It's also a category error for NL.

A CMS with k=4 frequency levels needs frequency-aware optimization: different parameter groups receiving updates at different cadences, with momentum that respects the frequency structure. The slow-frequency parameters accumulate gradient context over thousands of steps. The fast-frequency parameters update every step. Applying the same momentum to both treats them as interchangeable when their timescales differ by orders of magnitude.

PyTorch supports parameter groups, so you *can* work around this. But the default is wrong, every tutorial teaches the default, and the effort required to set up frequency-aware parameter groups from scratch is non-trivial. The framework's design assumes a uniform optimizer is the normal case.

![](/images/blog/google-didnt-go-silent/image9.jpg)

### DataLoader and Epochs Assume Batched, Finite Data

```python
for epoch in range(num_epochs):
    for batch in dataloader:
        loss = model(batch)
```

Three concepts in three lines, none of which exist in NL.

**Epochs.** The idea that you loop over the dataset multiple times. NL processes a continuous context stream. The "dataset" is the context window for the slowest frequency level. You don't loop over it; you process it once, at the slowest frequency.

**Batches.** The idea that data comes in discrete chunks from a shuffled dataset. NL processes a stream where ordering matters; the context builds up. Shuffling destroys the temporal dependencies that frequency levels rely on.

**DataLoader.** The idea that data loading is separate from computation. NL's context stream is the input to all frequency levels simultaneously. The context processor isn't loading data for the model, it IS the model's input at every timescale.

### Autograd Assumes Gradients OR Inference

PyTorch's torch.no_grad() context manager exists because gradients are expensive and you don't want them during inference. This encodes the assumption that inference and gradient computation are mutually exclusive.

In NL, the inner loop runs gradient descent inside the forward pass. Every token triggers: forward through memory, compute loss, compute gradient with respect to memory parameters, update memory. This happens during what you'd call "inference." The model needs gradients to process input as part of its primary computation. Disabling gradients during "inference" doesn't just lose learning capability; it prevents the model from functioning at all.

### These Are PyTorch Problems. The Real Problem Is Bigger.

Everything above is PyTorch-specific. JAX, the framework the research team actually uses, doesn't have most of these issues. There is no model.train()/model.eval() distinction. Gradient computation composes functionally inside forward passes. lax.scan maps directly to chunkwise parallel patterns. The Lattice paper includes a complete JAX implementation in its appendix. The TNT paper benchmarks every model, Titans, TNT, Transformer baselines, in JAX on TPUs. The math works. The researchers proved it. We proved it.

So why doesn't the code exist as a pip-installable package?

Because the real problem isn't the training framework. It's what comes after.

### Training and Inference Are the Same Computation. Our Infrastructure Says They're Not.

This is the point everything else has been building toward.

If there is no distinction between training and inference, if the model does the same computation whether you're feeding it a training corpus or a user's prompt, then there is no reason for separate training infrastructure and serving infrastructure. The thing that processes context IS the thing. You don't train it, export it, freeze it, quantize it, and deploy it to a serving stack. You run it. The frequency levels that are active might differ based on how much context the model has seen, but the mechanism is identical. "Serving" is just "processing context with the slowest frequencies already converged."

Now look at what the industry has built. vLLM, TGI, TensorRT-LLM, Triton Inference Server, these are billion-dollar investments in infrastructure optimized for one assumption: that model weights are constants after training. Every optimization they provide (continuous batching, PagedAttention, speculative decoding, model sharding across GPUs) depends on parameters being static. The TNT paper itself lists this as Challenge #1: current implementations lead to poor hardware utilization because the infrastructure was never designed for models that learn during inference.

![](/images/blog/google-didnt-go-silent/image7.jpg)

When the weights change on every forward pass, you can't shard the model the same way (replicas diverge), you can't cache the same way (the computation graph changes per request), and you can't batch the same way (each request modifies shared state). Model quantization breaks (parameters are moving targets). Graph compilation breaks (the computation changes per input). Horizontal scaling breaks (each replica diverges as it processes different inputs). A/B testing breaks (the model is different after every request).

It doesn't matter whether you built the model in PyTorch or JAX. The entire ML deployment pipeline (model export, quantization, ONNX conversion, TensorRT optimization, serving infrastructure) assumes stationarity. NL is non-stationary by design. The separation of training and inference is not a property of these models. It is a property of our infrastructure. Remove that separation, as Nested Learning does, and the infrastructure has no answer.

This is why the code was never released. Not secrecy. Not vaporware. You cannot pip install a paradigm that requires infrastructure that doesn't exist yet.

### The Community Implementations Prove This

I examined eight publicly available Titans implementations on GitHub. Every single one converges on the same pattern: take a Transformer, bolt on a memory module, combine outputs, benchmark. I don't blame the repo creators for this. Titans describes exactly such a method as a demonstration, and given the infrastructure we have, there is no other way to test these architectures without building new infrastructure from scratch.

The convergence was structural, not accidental. The hybrid pattern (Memory as Context, or MAC) works because it doesn't violate any infrastructure assumption. You keep the train/eval split. You keep the global optimizer. You keep the DataLoader. You keep the serving stack. You just add a module that runs an inner loop during training and freezes during inference, restoring the separation that the papers explicitly reject. It's the only configuration that current infrastructure can accept.

Tests pass. Benchmarks look good. The memory module genuinely improves long-context performance. But it's the smallest slice of what the papers describe, the one slice that fits through the infrastructure we have.

## The Evidence: What Happened When I Tried

Over the latter half of 2025, I collected the papers and periodically took a swing at building a Titans or Atlas model. I always failed. Unlike most of the community I focused on the MAG variant and while my intention was to build a "pure" Atlas model; NOT a hybrid. I had a hard time wrapping my mind around what the authors were describing, so I filled the gaps with my own assumptions about "how AI works." And I eventually was forced into the hybrid model because of an infrastructure that assumed that training and inference were two separate modes.

In January 2026, I devoted two solid weeks to the problem and built a 43-million-parameter Atlas-MAG model. 109 tests, all passing. 85.9% needle-in-a-haystack accuracy. Every equation faithfully implemented. Almost four days of continuous training on dual A6000 GPUs; a model smaller than most hobby projects. The code and the trained checkpoint are publicly available (*Atlas-MAG_OmegaRule*).

![](/images/blog/google-didnt-go-silent/image8.jpg)

It works. But only if you lie to PyTorch.

The model's defining capability, test-time learning (TTL), is gated behind a single conditional: if self.ttl_enabled and self.training. Call model.eval(), the standard practice for inference, and the inner loop goes silent. The model's long-range memory shuts off. Context beyond the 512-token attention window vanishes. To activate test-time learning during inference, you call model.train(), telling PyTorch the model is training when it is not.

We tested this directly. Same model, same prompts, same weights. In eval mode: fast, no memory updates. In train mode: the inner loop fires, memory updates at every position, 1.7 to 5 times slower, and the outputs change. The memory was learning from seven tokens of context and already producing different behavior. The model works. The infrastructure says it shouldn't.

As I noted before, my intention was to build a pure Atlas model but the tools at my disposal prevented me from doing that. My code, and all other titan and Atlas hybrid model variants, when audited against the framework of the Nested Learning paper fail on the following points:

- A model.train()/model.eval() switch that silenced the inner loop during evaluation. The mechanism the papers call "test-time memorization," is disabled at test time
- A separate MemoryModule class when the papers say memory IS the parameters
- Four methods beyond forward() when the papers say the forward pass is the only external API
- A single global AdamW where the architecture demands frequency-aware optimization
- An epoch-based training loop processing shuffled batches where the paradigm requires a continuous context stream

Every one of these choices came from the framework I was using. PyTorch taught me model.train(). Every tutorial taught me a global optimizer. The DataLoader API taught me batches and epochs. I didn't consciously import these assumptions. They are the default behavior of the tools.

So I dissected it. I rebuilt the model component by component. I added one piece at a time, polynomial features, then the Lattice inner loop, then CMS frequency nesting, and measured where the compute cost came from at each step. The polynomial features that give the model its memory capacity square values that go subnormal at FP16. Quantize the model and the memory module produces infinity. Not degraded performance. *Infinity.* The model works, benchmarks well at FP32, and you can never quant it down. This is a deployment constraint that falls directly out of the precision-over-scale argument above. Scale up to 120 million parameters and you hit sawtooth divergence. Every serving platform assumes you can quantize, shard, and batch. This model says no to all three. The full component-by-component analysis, training logs, and technical report are in the companion repository (*Atlas*).

109 tests passed. Every equation is correct. Zero structural compliance with the paradigm the papers described. And a cost profile that makes it clear: yes, you *can* build this in with our conventional toolset. However you can also hammer a nail with a pipewrench, but you wouldn't want to build a house that way.

## What Comes Next

The infrastructure gap I've described is not theoretical. I lived it. At least eight community implementations have lived it. Everyone who has tried to build from these papers has converged on the same hybrid pattern; not because we lack skill, but because the only models our infrastructure can serve are the ones that maintain the training/inference split.

The math works. The training frameworks can express it. The question was never "can we build it?" The question is "where does it run?" And right now, the answer is: nowhere that the industry has built.

But the nesting itself is real. We tested this with a minimal CMS implementation; same model, same data, same learning rate, same inner-loop parameters. The only variable was k, the number of frequency levels. At a learning rate where the inner-loop step size exceeds the error signal itself (softplus of 1.49), a single-level model (k=1) diverges to NaN at step 9,000. Add one level of nesting (k=2, where the slow level fires every 8th step) and the model converges; a 98.7% loss reduction where the single-level model produces infinity. The slow level acts as a temporal momentum buffer, averaging over 8 fast-level updates before applying its own, effectively dividing gradient variance by 8. Nested Learning is not philosophy. It is variance reduction on the temporal axis, and it is measurably stabilizing.

![](/images/blog/google-didnt-go-silent/image4.jpg)

So what would it actually take to serve a model like this?

We need a specification; a new contract for our primitives. The seven papers define concrete requirements that any serving infrastructure must satisfy, and those requirements eliminate most of what currently exists:

- The forward pass must support interior gradient computation. The memory update runs gradient descent inside the forward pass. This is not optional; silence it and the model loses its memory architecture entirely. Any serving runtime that assumes the forward pass is a pure function of frozen weights is structurally incompatible.
- Replicas must maintain an independent state. Each request modifies the model's memory parameters. Two replicas processing different inputs will diverge. Load balancers cannot treat replicas as interchangeable. Session affinity becomes a hard requirement, not a performance optimization.
- Quantization must respect capacity theory. Atlas proves polynomial feature mappings give O(d_k^p) memory capacity. Those polynomial terms involve squaring values that go subnormal at FP16. The paper's own results show 10B parameters at full precision outperforming 100B at reduced precision. Quantization is not a deployment decision. It is a capacity decision, and the capacity math is known.
- The computation graph changes per input. The inner-loop optimization is data-dependent. torch.compile, TensorRT, and XLA graph caching all assume the graph is fixed after tracing. A model that optimizes its own parameters during inference has a different graph for every input.
- Differentiation must work through mutation. Nested optimization requires gradients through code that modifies state in place, without materializing the full computation graph for every nesting level. PyTorch unrolls this and blows VRAM. JAX handles it better with functional transforms but still materializes intermediate states. What you actually need is differentiation at the compiler level through arbitrary control flow, mutation, and nested function calls. The language and toolchain follow from this requirement, not the other way around.

These are not aspirational goals. They are constraints extracted directly from the papers' mathematics. Any implementation that violates them is building a different model than what the papers describe which is exactly what every community implementation we examined has done.

In Part 2, I'll describe how we encoded these constraints into a specification graph that enforces them automatically rejecting code that introduces concepts the papers never defined and what a serving stack built from those specifications might look like.

*This is a response to Delanoe Pirard's "Transformers Are Dead. Google Killed Them, Then Went Silent" (December 21, 2025) and "A 170M Model Just Beat GPT-4" (December 31, 2025). Both are excellent and worth reading.*

*The Nested Learning research program is by Behrouz, Razaviyayn, Zhong, Karami, Li, Kacham, Daliri, Deng, Pascanu, and Mirrokni at Google Research.*

*I am an independent researcher with no affiliation with Google or any institution mentioned in this article. The implementation described in Part 2 was built with Claude Code on two A6000 GPUs and late nights.*
