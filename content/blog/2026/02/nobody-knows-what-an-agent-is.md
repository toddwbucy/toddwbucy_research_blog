---
title: "Nobody Knows What an Agent Is. And That Is the Problem."
date: 2026-02-19T09:00:00-06:00
draft: false
type: "post"
status: "published"
tags:
  - agents
  - statefulness
  - weavertools
  - ai-infrastructure
  - machine-learning
  - definition
  - harness
abstract: |
  A diagnostic framework for practitioners building production agent
  systems. Warren Weaver's 1949 three-level communication framework
  maps onto the three engineering requirements an agent must satisfy
  to project intent past the first point of contact with the world: a
  Semantic Processing Unit, a Memory System, and a Harness. Most of what
  the industry currently calls "agents" are actors — sophisticated
  stateless ones whose agency terminates at delivery. Until we own that
  distinction, we keep building Level B solutions and calling them Level
  C achievements.
medium_url: ""
---

![](/images/blog/nobody-knows-what-an-agent-is/image2.jpg)

## Who This Is For

**This is not a research paper.** It is a diagnostic framework for practitioners, and it is deliberately written as one. Researchers are busy building the next model. This is for the people trying to make agents work in production. The engineers, architects, and technical leaders who get up every morning and face the real problem: how do you build agent systems that are reliable, predictable, and economically viable?

That is our job as practitioners. We are not here to discover new science. We are here to take the science that exists and make the technology economically viable at scale. We invented SaaS. We invented IaaS. We invented PaaS. We did not invent the underlying technology of these service models. We operationalized virtualization technology so that it could be deployed at scale in an economically sustainable manner. That is what needs to happen with agents.

For the past three years, we have been building agents with the tools available to us — stateless architectures that worked brilliantly for one set of problems. We have reached the limits of what statelessness can deliver. To go further, we need to take state seriously. And that requires a fundamentally different mental model.

The foundation problem is not technical. It is theoretical. We do not have a clear definition of what an agent actually is. And without that, everything we build is guesswork. What I propose in this paper is one candidate definition of what an agent is and the downstream engineering implications if such a definition is adopted. I think it is a strong one and I hope to convince you that it is as well. To do that, we must go all the way back to the beginning of the whole project.

## The Thesis

In 1949, Warren Weaver wrote an introduction to Claude Shannon's *The Mathematical Theory of Communication* that identified three distinct levels of the communication problem:

> Relative to the broad subject of communication, there seem to be problems at three levels.
>
> Thus it seems reasonable to ask, serially:
>
> **Level A.** How accurately can the symbols of communication be transmitted? *(The technical problem.)*
>
> **Level B.** How precisely do the transmitted symbols convey the desired meaning? *(The semantic problem.)*
>
> **Level C.** How effectively does the received meaning affect conduct in the desired way? *(The effectiveness problem.)*
>
> — Warren Weaver, *The Mathematical Theory of Communication*, 1949

Shannon's work made Level A tractable for engineers. The machine learning community, across eighty years of cumulative work, made Level B tractable. Level C — making meaning produce intended, reliable, measurable behavioral change — is where everyone is stuck right now.

This paper argues that Weaver's three levels of communication are not just a framework for understanding communication. They are, inadvertently and from first principles, the right starting point for a definition of what an agent must be — one that makes the engineering requirements explicit rather than leaving them as implementation choices. We will come back to that.

## Level A — The Technical Problem

Shannon's *A Mathematical Theory of Communication* gave us information theory: entropy, channel capacity, error-correcting codes, the mathematical foundation for digital communication. Every network protocol, every storage system, every codec — all of it is downstream engineering based on Claude Shannon's 1948 framework.

Shannon did not solve telecommunications. He gave engineers the theoretical framework they needed to make the problem tractable. Theory does not solve a problem. Theory states the problem precisely enough so that engineers can make measurable, incremental progress toward a defined ceiling. This was Shannon's achievement.

But Weaver saw something else in Shannon's work. The theory of the channel — its capacity, its noise, its fundamental limits — does not stay neatly inside Level A. Channel constraints propagate upward. You cannot convey more meaning than the channel permits. You cannot produce more effective action than the meaning conveyed will support.

This is not abstract. It has direct engineering consequences that we will return to when we look at how current agent infrastructure constrains agent effectiveness by artificially narrowing the channels through which agents access their own memory and context.

## Level B — The Semantic Problem

**Level B is the computation of meaning. The model is the Level B problem.** Not the architecture, not the training method, not the scale — the language model. Any system that can take arbitrary input and produce semantically coherent, contextually relevant output has made Level B tractable.

The fact that this works at all is remarkable. The destination, from the moment the first stored-program computers came online in the late 1940s, was always the thinking machine. The electronic brain. Warren Weaver understood the scale of this project earlier than most. His 1949 framework was not academic commentary. It was a map drawn by someone actively laying the institutional and theoretical groundwork for the technology we are building on today.

It took eighty years, two AI winters, and the fortuitous collision of transformer architecture with GPU compute to get here (Vaswani et al., 2017). The winters were not failures. They were the cost of a multi-generational project running ahead of its hardware.

## Level C — The Effectiveness Problem

Level C is making Level B usable and productive in the real world. The model can compute meaning. Level C is everything required to turn that computation into reliable, consistent, predictable action.

Every agent framework launched in the last three years is an attempt at Level C, and they represent real progress. Each one has pushed further than what came before. But most share the same ceiling. Not because of poor engineering, but because the stateless infrastructure they run on was never designed for what agents actually require. Single-turn tasks they handle remarkably well. Anything that requires monitoring the consequences of their own actions, adjusting based on what actually happened, and maintaining coherent intent across time — that is where the architecture runs out of road.

The structural reason is straightforward. Our agents are split across network boundaries. The Memory System is across a network connection. The harness is calling remote APIs. Every internal reasoning step over your data pays a latency tax before the system can think again. That cost is not incidental. It changes what the system can effectively do. A system that cannot accumulate knowledge of your domain over time — your architecture, your conventions, your operational history, your twenty-year-old Windows 2003 box still humming in the back room running custom software no one has touched since 2018 (you know who you are) — cannot be an effective agent for your project, your company, or your organization. Not because the components are missing, but because the architecture was built for a different problem.

We have seen this pattern before. In 2023, Prime Video engineering published a case study on their video quality monitoring pipeline. What should have been a single coherent process had been deployed as distributed cloud components, paying network and orchestration costs at every internal step. When they moved that tightly coupled workflow into a more co-located design, infrastructure costs dropped by 90 percent (Kolny, 2023).

The agent harness is the same class of problem. The SPU, the Memory System, the policy layer, and the tool interface are parts of one tightly coupled reasoning process — but we have been running them as distributed services, because that is the infrastructure we know how to build at scale. The lesson from Prime Video applies directly: co-locate the process. Keep state close to inference. Keep data transfer in memory. Stop paying network penalties on work that should never have crossed a network boundary in the first place.

## Why Level C Belongs to Practitioners

Levels A and B both have theoretical frameworks that make engineering tractable. Shannon gave engineers the ceiling for Level A. Transformers and embedding theory gave engineers the ceiling for Level B. In both cases, researchers defined the problem precisely enough that practitioners could build toward a known target.

Level C is different — and not just because it lacks a framework. It is different because the problem is not primarily a research problem. Level A was about signal fidelity. Level B was about semantic computation. Both could be solved in a lab and then handed to practitioners to implement.

Level C is about making agents work reliably in the real world, at scale, in an economically viable way. That is not a lab problem. That is our problem. The infrastructure, the economics, the deployment reality that we live with every day.

Level C will not be won by the next paper alone. It will be won by the companies and organizations that turn persistent agent state into a deployable, governable, economically viable service model. They will be the ones who figure out how to provision stateful agent infrastructure sustainably, how to make the economics work, how to build harnesses that keep agents coherent over time in production environments. That is engineering work. That is our work.

## What Weaver Actually Gave Us

This paper proposes that Weaver's three levels of communication are, from first principles, the right structure for defining what an agent must be for production engineering. Not because Weaver intended this — he did not — but because the three levels map onto the three requirements an agent must satisfy to project intent past the first point of contact with the world.

The channel constraint is bidirectional, and both directions are load-bearing. On the input side, a bandwidth-starved SPU cannot process what it cannot receive. The model's capability is irrelevant if the retrieval channel cannot deliver sufficient context in a timely manner. On the output side, if the agent's projected output does not successfully traverse the channel and produce effects in the world, the agent has computed but not acted. By the definition this paper is about to propose, that is not agency. It is arithmetic. Level A failure on the output side does not degrade Level C. It collapses it entirely.

## Actors and Agents Are Not the Same Thing

*What follows is a proposed definition, not an appeal to consensus. But if the definition holds, its implications are concrete: a system that cannot sustain intent, track downstream effects, and update across time is not an agent in the sense this paper is concerned with.*

In any network, there are actors. In its simplest form an actor is any entity that modifies the state of other entities in a network. A pneumatic door closer is an actor. It participates in a network. Its agency is real. But its agency ends at the state of the door. One predetermined singular state variable against one predetermined threshold. Is the door open? Close it. That is the limit of its function in the network. Its output terminates at the first point of contact with the world (Latour, 1992).

These conditions can be stacked. You can build quite sophisticated if/then decision trees, and researchers did exactly that. BDI architectures gave agents explicit belief states, desire hierarchies, and intention structures. SOAR added production rules and chunking. These were not naive systems; they crossed over into robotics, air traffic control, and game AI. Decades of serious engineering work went into them and it produced real results.

But they shared the same ceiling as the door closer, just much higher up. The beliefs were explicit and pre-authored. The desires were specified in advance. When the world diverged from the symbolic model, as the real world reliably does, the system had no mechanism to learn from that divergence — because the symbolic representation itself was the boundary of what the system could perceive. It could only react to state changes it had been built to recognize. Everything outside the authored model was invisible.

This is the definitional limit of an actor, however sophisticated: its agency is bounded by what it was designed to anticipate. Its output terminates at the first point of contact with the world it was modeled to expect.

An agent is a specific and more demanding kind of actor.

> **An agent creates persistent effects in the world, then observes, learns from, and adapts to the resulting feedback.**

That distinction is everything.

Whether an agent is messaging, requesting, using a tool, or modifying a record, its job is not finished at the moment of execution. Its purpose is to trigger shifts throughout a network and gather intelligence on the results. By driving downstream actions and receiving feedback, it can observe, learn, and iterate based on how the network's state has changed. Agency does not end with delivery; it persists through the entire lifecycle of the network response.

This is precisely why a stateless service, no matter how advanced its model, cannot function effectively as an agent. Such services produce a result that simply ends once contact is made. Once the response is sent and the session closes, the service becomes blind to any following events. Because it cannot monitor or react to downstream impact, it lacks true intent; its agency is stopped short at the very start.

This is also why memory is not optional. To produce output designed to create follow-on effects, the agent must be able to:

1. Remember what it projected into the network.
2. Observe what came back — what effects its output produced.
3. Update its understanding of the network based on those effects.
4. Project again, informed by what it learned.

Without memory, steps 2 through 4 are impossible. The agent cannot sustain projected intent across time. It is an actor, not an agent.

## What Weaver's Three Levels Define

Weaver's three levels of communication map onto the three requirements for an agent to function as an agent, not just as an actor. But they are not three separate isolated things. They are three optimization requirements that drive how the agent must be built.

**Level A — The substrate.** Your agent does not float in abstraction. It lives on hardware. It breathes through network connections. Every component relationship — SPU to memory, memory to tool interface, observation to inference — is a physical transaction on a physical substrate. What the substrate permits is the ceiling for everything built above it.

**Level B — Semantic computation.** The agent must be able to compute meaning from input and produce meaningful output. This is the Semantic Processing Unit, the model. The thing the field has spent eighty years building. Necessary but not sufficient. And critically, swappable: the SPU is infrastructure, not identity.

**Level C — Effectiveness over time.** The agent must be able to act in ways that produce intended follow-on effects and must be able to track, learn from, and respond to those effects. This requires persistent state. Memory of what was projected, what came back, what worked, what the network looks like now versus before. This is the Memory System. The state is where agent identity and value actually live.

The three levels define what an agent needs. They do not define what makes those needs work together.

That is a different problem entirely. And it is the hardest one.

**That is the Harness.**

The Harness is the autonomic system of the agent. Just as the autonomic nervous system coordinates breathing, circulation, and digestion without interrupting conscious thought, the Harness coordinates the SPU and Memory System into unified action. It manages the query loop. When to call the SPU, with what context from memory, how to route the output. It provides the Tool Interface: the mechanisms by which the agent acts on the world, dispatches tools, enforces permissions, and classifies operations before execution. It decides what gets written to memory and when. It maintains identity across sessions. It audits everything.

The Tool Interface is not a separate component. It is what the Harness does at the Level A boundary. The mechanism by which the agent projects output past the first point of contact and receives feedback from the world.

When the autonomic nervous system fails, the organism does not operate suboptimally. It dies. When the Harness is absent or broken, the agent does not perform poorly. It ceases to be an agent — because there is nothing making the SPU and Memory System function as a unified entity with persistent intent.

The field does not currently have a consensus definition of "agent" that has made production engineering tractable. This paper proposes one. What follows is conditional: if this definition is right, then persistent memory, statefulness, and infrastructure designed around long-lived entity continuity are not optional implementation details. They are engineering requirements implied by the definition itself.

Put these together and you get the proposed definition:

> **An agent is SPU + Memory System + Harness.** The Harness is not another component alongside the others. It is what makes the other two components into an agent rather than two disconnected systems. The agent is greater than the sum of its parts and the Harness is why.

![](/images/blog/nobody-knows-what-an-agent-is/image1.jpg)

Remove any component and you no longer have an agent:

- **SPU without Memory System:** A stateless calculator. Output terminates at delivery. Cannot track follow-on effects. Cannot learn. Cannot accumulate value. *This is what most current "agents" actually are.*
- **SPU without Harness:** A model with no coordination layer. No tool dispatch, no memory management, no identity continuity. Semantic capability with no agency.
- **Memory System without Harness:** A database. Storage with no reasoning, no coordination, no mechanism to turn accumulated state into action. This is the core failure of naive RAG. The data exists, but the agent cannot reason over it effectively because there is nothing coordinating the retrieval into coherent action.
- **SPU + Memory without Harness:** Two components that do not know how to work together. Parts on a bench. The Harness is what makes them an agent.

**By this definition, most of what the industry currently calls "agents" are not agents.** They are actors — sophisticated ones, capable of remarkable semantic computation. However their agency terminates at the first point of contact. They are door closers with language models inside.

To be precise: this paper is not describing how people currently use the word "agent." It is establishing what an agent must be to function as one. A single-turn system that forgets everything between sessions is a very capable tool and there is nothing wrong with that. But calling it an agent is aspirational marketing, not engineering. Confusing the two is exactly the diagnostic error that is costing the field years of misdirected effort.

## The Statefulness Insight

**You do not need a stateful model to have a stateful agent.**

A common objection is that modern models do learn within context — they adapt, reason, and update their responses based on everything in the context window. That is true. But in-context learning is ephemeral. It vanishes the moment the context window resets. That is not statefulness, it is a working memory illusion. The model is not remembering. It is re-reading. When the session ends, everything learned in that session is gone. The next session starts from zero.

Durable statefulness requires persistent storage that survives session boundaries. Storage the agent can write to, read from, and build on over time. That is the Memory System. And critically, it does not need to live inside the model.

The SPU — the model — can be stateless. That is fine. Stateless models are cheaper, easier to scale, easier to replicate. Keep them stateless.

What you need is a stateful *system*. The Memory System provides statefulness independently of the SPU. The agent's identity, history, and accumulated understanding live in memory — not in the model weights. The model reads from memory, reasons over it, writes to it, and produces output. The model itself remains stateless. The agent as a whole is stateful.

This reframes the entire engineering problem. The question is not "how do we make models stateful?" The question is: how do we engineer a stateful system around a stateless SPU, at scale, in an economically viable way?

That is a different problem. And it is one practitioners know how to approach because it is the same class of problem we solved with SaaS, IaaS, and every other infrastructure abstraction we have built. Except we are currently using the wrong abstraction.

## The Economic Problem

This is where the theoretical gap becomes the crisis practitioners live in every day.

Recall Weaver's constraint: you cannot convey more meaning than the channel permits. Channel constraints at Level A propagate upward through B into C. This is what happens when you put the SPU in a cloud data center and the Memory System in a remote database — or worse, on the user's local machine.

Every time the SPU needs to reason over its own history, it must traverse a network connection. Network latency. Bandwidth limits. Round-trip overhead. All of it constraining what the model can access, which constrains the meaning it can compute, which constrains the effectiveness of the action it can take.

By Weaver's own framework, the SaaS model — with its stateless distributed service substrate — is suboptimal for tightly coupled SPU-memory reasoning loops and economically misaligned with agents. It artificially narrows the channel between the SPU and its own memory, importing Level A constraints into Level B processes and degrading Level C effectiveness before the agent has even begun to operate.

The field has normalized this as "that is just how AI works." It is not. It is what AI looks like when you build stateful entities on stateless infrastructure because you do not have an economic model for anything else.

## Individually Unique, Fungible at Snapshot

SaaS works because stateless services are permanently fungible. Instance 47 is identical to instance 1. Always. That permanent identity between instances is what makes replication and amortization possible — and what makes the economics work.

Agents are fungible at provisioning. Two agents started from the same model weights and the same initial configuration are identical at the moment of instantiation. The SaaS model can handle that.

**What it cannot handle is what happens next.**

The moment an agent begins accumulating state it begins diverging from its siblings. After a week of operation it is distinct. After a month its accumulated knowledge represents real value that a fresh instance does not have.

This does not mean agents are irreplaceable. A memory system that is a first-class component can be cloned, snapshotted, and restored. Fungibility takes a new form: rather than spinning up identical instances from a common template, you provision new agents from a memory snapshot inheriting the accumulated domain knowledge of a predecessor. The memory system is coupled to the harness, but the data it holds is what carries fungible value. Thus you can pair that accumulated knowledge with a more capable SPU as better models become available. Upgrade the model, and the agent gets more capability without losing memory state. The agent gets more valuable over time to the organization, not less. The state is the fungible asset, not the specific model running over it.

This is what a properly architected agent makes possible. Right now every model upgrade is effectively a hard reset. The agent loses everything it learned because the learning lived in the context window or fine-tuning, both tightly coupled to the model. With a decoupled memory system, model upgrades become infrastructure decisions, not identity destruction.

You cannot kill a six-month-old agent's state to save infrastructure costs without destroying the asset. SaaS treats instances as disposable by design. Agent infrastructure must treat accumulated state as the asset because it is.

## The Accumulated State Is Not Just a Sunk Cost. It Is the Value Proposition.

**The agent gets more valuable the longer it runs.**

In SaaS, value delivered per request is roughly flat. Infrastructure costs decrease over time through optimization and amortization, but the value ceiling stays constant. A web server that has been running for two years is not meaningfully better at serving requests than one that has been running for two days.

Agents are different in kind, not just in degree. The relationship between time and value is not flat. It compounds.

A day-one agent knows nothing about you, your systems, your codebase, your conventions, your past decisions, your domain. It is capable but generic. A six-month-old agent that has worked alongside your team, accumulated your architecture's patterns, learned which approaches have worked and which have failed in your specific context — that agent is worth dramatically more. Not slightly more. *Categorically* more.

**The right economic analogy is not a server. It is an employee.**

A new employee and a senior employee cost roughly the same in salary terms. But they do not deliver the same value. The senior employee's accumulated knowledge, judgment, and institutional memory are the assets. You do not replace a ten-year employee with a cheaper new hire without losing something that cannot be quickly rebuilt.

The implications of this are uncomfortable, and the industry has been avoiding them. If agent value compounds with tenure:

- Provisioning an agent is an investment, not a subscription.
- Destroying an agent's state is not a deployment decision — it is a decision to destroy accumulated value.
- The "just call the API" model is the equivalent of hiring a temp who forgets everything between shifts.
- An agent that has learned your systems deeply is a competitive asset, not a commodity service you swap out for a cheaper provider next quarter.

We do not yet have the economic framework for provisioning stateful agent infrastructure sustainably. One that accounts for the appreciating value of accumulated state, the cost of provisioning at instantiation, and the compounding return on maintaining continuity over time.

Until we do, every practitioner building agent systems is working without a blueprint and paying the cost in degraded performance, architectural compromises, and unsustainable spend. And every time they kill an agent's state to save infrastructure costs, they are destroying the very thing that makes agents worth building in the first place.

## The Call

The field is not failing because engineers are not smart enough. It is not failing because the models are not powerful enough. It is failing because we have not agreed on what we are building.

## Agents Defined

> *An agent is an entity that produces output designed and intended to create follow-on effects past the first point of contact with the world and that observes, learns from, and adapts to the results of those effects over time.*

The goal is not a single action. It is the accumulation of network effects through continuous, adaptive engagement with the world. It requires three things to do this: a Semantic Processing Unit to compute meaning, a Memory System to sustain projected intent and accumulate the knowledge required to adapt, and a Harness to coordinate both as a single coherent entity within a larger network of actors and agents. Remove any one of those components and you do not have an agent. You have parts.

The model is not the agent. The stateless service is not the agent. The context window stuffed with tokens is not memory. We have been treating the Semantic Processing Unit as if it were the whole system, and building the economics, the infrastructure, and the organizational expectations around that mistake.

The infrastructure gap is real. The economic model does not yet exist. We do not have a **Statefulness as a Service** model yet. The substrate is not ready. None of that changes the diagnosis — and the diagnosis has to come first. You cannot build the right infrastructure for something you have not defined correctly. You cannot scale a solution to a problem you have not named.

That is the gap this paper is asking you to close. Not by yourself. Together.

The engineering community invented SaaS to make stateless services economically viable at scale. It invented IaaS to make compute infrastructure accessible. It invented containerization to make deployment reproducible. Each of those was a paradigm shift that came from practitioners facing a real problem and refusing to accept that the current model was the only model.

**We are at that moment again.**

**The model is not the agent. The model is the Semantic Processing Unit of the agent. Until the field owns that distinction, we will keep building Level B solutions and calling them Level C achievements.**

So here is the question this paper puts to the community:

*Now that we have named it, what are we going to do about it?*

**The discussion starts here.**

## References

- Shannon, C. E. (1948). "A Mathematical Theory of Communication." *Bell System Technical Journal,* 27(3), 379–423.
- Weaver, W. (1949). "Recent Contributions to the Mathematical Theory of Communication." In Shannon, C. E. & Weaver, W., *The Mathematical Theory of Communication.* University of Illinois Press.
- Vaswani, A., et al. (2017). "Attention Is All You Need." arXiv:1706.03762.
- Latour, B. (1992). "Where Are the Missing Masses? The Sociology of a Few Mundane Artifacts." In Bijker, W. E. & Law, J. (Eds.), *Shaping Technology / Building Society.* MIT Press.
- Kolny, M. (2023). "Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%." Amazon Prime Video Tech Blog.
