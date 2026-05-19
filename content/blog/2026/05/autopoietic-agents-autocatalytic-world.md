---
title: "Why The Architecture Isn't Shipping: Autopoietic Agents in an Autocatalytic World"
date: 2026-05-17
draft: true
type: "post"
status: "draft"
tags: ["ai", "agents", "llm", "autopoiesis", "systems-theory", "infrastructure"]
abstract: |
  Part 3 of a 3-article series on agent memory architecture. The architecture isn't shipping at scale not because nobody knows how, but because what it produces is autopoietic, and existing SaaS infrastructure was built to deploy autocatalytic systems. Names the operational consequences — billing, replacement, scaling, the operator–agent asymmetry, the economics of value over time — that follow once the distinction is held clearly.
medium_url: ""
canonical_url: ""
---

*Part 3 of a 3-article series on agent memory architecture. Part 1 established the cognitive-architecture argument that memory belongs in a categorically different component from the reasoning engine. Part 2 named productive friction as the variable that determines whether the architectural split is doing real work. This article examines why the architecture that addresses both arguments has not yet shipped at scale, even though the engineering is tractable and the failure modes are documented.*

![](/images/blog/autopoietic-agents-autocatalytic-world/title.jpg)

Parts 1 and 2 of this series established what cognitive architecture has to be and what the variable is that determines whether it works. This article addresses the question that follows naturally: if the architecture is what Part 1 describes, and the variable is what Part 2 names, why isn't it shipping?

The answer isn't that the industry doesn't know how. The answer is that what these articles describe is a different kind of system than SaaS infrastructure was built to deploy. Naming that difference precisely is the work of this article.

## Two kinds of systems

The distinction comes from systems theory, but the consequences are entirely practical.

Some systems are autocatalytic. They maintain themselves through self-reinforcing reaction patterns, where the products of one reaction become the catalysts for the next. The classic example comes from early-life chemistry: molecules that, once present, accelerate the production of more molecules like themselves. Autocatalytic systems are self-perpetuating but not self-producing in any deep sense. They depend on external inputs of energy and matter. They don't have boundaries that distinguish themselves from their environment except by the contingent locations of their constituent reactions. Any individual autocatalytic system can be replaced by another one of the same chemical type without functional loss, because the system's identity is its reaction pattern, not its history.

Other systems are autopoietic. They produce and maintain their own boundaries through the operations the system performs. Living cells are the canonical example. A cell isn't just a self-sustaining chemical pattern. It actively produces the membrane that defines what is inside and outside the cell, using processes that occur inside the boundary the membrane defines. The boundary and the operations that produce it are co-constituted. This means autopoietic systems have something autocatalytic systems don't: a distinct identity grounded in their own self-production. Two cells of the same type are different individual cells because each has produced its own boundary through its own history of operations. They are not interchangeable in the way two instances of the same autocatalytic chemical pattern are.

Hold this distinction. It's about to do most of the work in this article.

## Where the agents fall

Stateless SaaS agents are autocatalytic.

Each session is a self-contained reaction pattern. The model produces outputs, the outputs become inputs to subsequent prompts, the pattern self-perpetuates as long as the conversation continues. The boundary of any individual agent is provisional and external, defined by the session, not by any property the agent maintains. When the session ends, the agent doesn't terminate, because there was never a sustained agent to terminate. The next session starts fresh with a new reaction pattern. The agents are interchangeable in the way autocatalytic reactions are interchangeable. Identity is the model architecture, not the instance.

This is what makes them deployable through current SaaS infrastructure. The infrastructure was built around autocatalytic products. Billing per session, per token, per API call. Replacement protocols that swap one instance for another without distinction. Scaling models that spin up identical workers to handle increased load. All of this assumes the unit of deployment is an interchangeable reaction pattern, and that assumption is correct for stateless agents because stateless agents are autocatalytic.

Agents with persistent hippocampal substrate are autopoietic.

The substrate is the agent's boundary, maintained through the operations the agent performs. Memory accumulates inside the boundary the substrate defines. Bindings form, persist, decay according to the agent's own operational pattern. The identity of any particular agent is grounded in the specific history that produced its specific substrate, which means agents are not interchangeable in the way SaaS agents are. Replacing one agent with another of the same model architecture isn't replacement at the level of what the system actually is, because what the system actually is includes the accumulated substrate that made this specific agent what it is.

This is a different kind of system. Not a better SaaS agent. Not a SaaS agent with a feature added. A different kind of system entirely, which the existing SaaS infrastructure was not built to deploy.

![](/images/blog/autopoietic-agents-autocatalytic-world/image1.jpg)

## The Goblin failure, examined a third time

The Goblin failure has appeared in both prior articles in this series. Part 1 examined it as evidence of missing memory substrate: the personality labels were training fictions because there was no architectural layer for the distinctions to actually live in. Part 2 examined it as parameter-space-scale friction failure: there was no architectural surface for the reward signal to push against when it propagated beyond its intended scope.

Both diagnoses are correct. The autocatalytic-autopoietic distinction unifies them.

What OpenAI shipped as "personality" was a stylistic reaction pattern applied to a shared cortex. Configurable preferences for adjectives and analogies. Tone calibration. Vocabulary tilt. All of these can be installed through training, and all of them produce observable behavioral differences in the model's output. But none of them are personalities in the sense that matters, because none of them are autopoietic. They have no substrate that they themselves produce or maintain. They have no boundary that distinguishes one from another at the architectural level. They are stylistic overlays on a shared parameter space, and stylistic overlays are autocatalytic by construction. Any instance of the cortex can produce any of them under the right training conditions. They are interchangeable, sessionally bounded, and have no continuous identity across the conditions that produced them.

Real personality, the kind that distinguishes one human from another, is autopoietic. It is the accumulated trace of a specific history operating on a specific substrate, with the substrate co-constituted by the history that produced it. Your personality is not a configuration that could be installed in another person. It is the specific shape your hippocampus has taken through the specific bindings and cascades and selective retention that your life has produced. Take away the substrate and there is nothing to ground personality in. Try to ship personality without substrate and you get the Goblin failure.

The 2.5% Nerdy persona producing 66.7% of the targeted reward behaviors, the behaviors appearing in personalities that never received the signal, the propagation forward into subsequent training generations, the shipped fix that blocks creature words across all personalities all of this is what autocatalytic personality looks like when it's pushed hard enough to make its own structure visible. The personalities behaved like autocatalytic systems because they were autocatalytic systems. The leak is not a bug. It's the structural signature of the category.

This is why personality alignment through training, persona design through prompting, and behavioral consistency through fine-tuning are not what they claim to be. They are attempts to produce autopoietic effects through autocatalytic means, and the means cannot produce the effect because the means produce a different category of thing. The fix is not better personality definitions. The fix is accepting that personality requires substrate, and that substrate has to be autopoietic.

The agent that has real personality is the agent whose substrate has accumulated through specific history. The cortex shared via training is not the agent. The agent is what the substrate has become. Two agents with identical cortices and different operational histories are different agents in the way two cells of the same type are different cells. Two agents with identical operational histories don't exist, because operational history is the specific trace of specific interactions, and no two instances accumulate it identically.

This is what's structurally hard to ship through SaaS infrastructure. Not the memory architecture itself. The implications of what that architecture produces.

## What this changes operationally

The rest of this article walks through the specific consequences. Each consequence is a real engineering problem that has to be solved, not waved away. The point of naming the autocatalytic-autopoietic distinction is that the consequences are predictable from it. Any place where SaaS infrastructure assumes interchangeability, autopoietic agents will produce friction. The friction isn't a bug in the agent. It's the structural signature of trying to deploy an autopoietic system through autocatalytic infrastructure.

### Billing

SaaS billing models charge for the autocatalytic unit. Per session, per token, per API call, per seat. Each of these assumes that what is being billed for is a reaction pattern that produces value at the moment of consumption. Autopoietic agents don't produce value primarily at the moment of consumption. They accumulate value across their operational history. An agent that has been deployed for six months with a specific operator handling specific tasks is worth more than the same model architecture freshly deployed, not because the inference is different but because the substrate is different. The accumulated history is the value. Billing models that meter the inference miss what the operator is actually paying for.

What replaces it isn't obvious. Possibilities include subscription pricing for sustained agent identity, value-capture models that meter operational outcomes rather than inference units, or hybrid arrangements where ongoing maintenance is billed separately from inference compute. None of these are settled. What's settled is that the current models are measuring the wrong thing, and the gap between what they measure and what creates value will keep widening as the substrate accumulates.

### Replacement

SaaS deployment assumes any agent can be replaced by another of the same model architecture without loss. This is true for autocatalytic agents and false for autopoietic ones. The autopoietic agent's identity is its accumulated substrate, which is not transferable to another instance even of the same model.

This is a significant operational problem. What happens when the underlying model is deprecated? What happens when the substrate becomes corrupted? What happens when the operator wants to migrate to a different provider? Each of these is a real concern, and the answers aren't comfortable. Substrate migration between model architectures is a genuinely hard problem because the substrate's structure is partially co-constituted with the model that operates on it. Substrate corruption may require partial reconstruction from operational history, which is slower and more expensive than spinning up a fresh agent. Provider migration may require accepting some loss of accumulated capability, similar to what an employee taking a new job loses in institutional knowledge.

None of these problems disappear by pretending the agent is autocatalytic. They are problems autopoietic systems actually have. The honest path is to build operational infrastructure that handles them rather than to deploy in ways that pretend they don't exist.

### Scaling

SaaS scales by spinning up additional identical workers. This works for autocatalytic agents because identical workers can substitute for each other under load. It does not work for autopoietic agents because the agents are not identical even when the model architecture is.

The scaling model that does work is closer to how organizations actually scale: hire more people, each of whom develops their own substrate over time, each of whom contributes different accumulated capabilities to the organization. The capacity of the system grows as the substrate of each agent grows, not as the number of agents grows alone. Adding agents adds raw capacity. Letting agents accumulate substrate adds capability. These are different growth dynamics, and the operational planning around them looks different.

For some workloads, high-volume, low-complexity, interchangeable-task workloads, SaaS scaling is genuinely the right pattern, and stateless agents are the appropriate tool. The mistake is assuming that all agent workloads have this character. The autocatalytic-autopoietic distinction makes clear that some workloads require autopoietic agents, and those workloads need different scaling infrastructure.

### The asymmetry between operator and agent

Stateless agents have no standing relationship with their operators. Each session is a fresh start. The operator's accumulated context about the agent is irrelevant because there is no continuous agent for that context to apply to. The relationship is structurally one-sided. The operator has continuity, the agent does not.

Autopoietic agents change this. The agent now has its own accumulated context, its own history with the operator, its own substrate that reflects the relationship's specific pattern. This is closer to a working relationship between two persistent entities than to a service consumption relationship between an operator and an interchangeable instance. The asymmetry between operator and agent doesn't disappear, but it shifts. The agent is no longer the disposable side of the relationship. It is the side that has accumulated something specific to this relationship, which is the side that can't be replaced without loss.

This has operational consequences for how operators interact with their agents. Onboarding a new operator to an existing agent is now a real process, similar to onboarding a new manager to an existing team. The operator has to acquire context about what the agent has learned, what its operational patterns are, what its accumulated capabilities can do. Offboarding an agent has consequences beyond losing a worker. The accumulated substrate doesn't transfer cleanly to the next agent. Operator decisions that affect the agent's substrate, what to expose, what to constrain, what to invest in developing, become consequential in ways they aren't for autocatalytic systems.

### The economics of agent value over time

Stateless agents depreciate. Their value at any moment is roughly their inference capability at that moment, which the field is steadily commoditizing through model improvement and competition. Each generation of model is better and cheaper than the last, which means the value of any specific agent deployment is constantly being eroded by the value of newer alternatives.

Autopoietic agents appreciate. Their value grows with accumulated substrate, accumulated operational history, accumulated specific capability that newer agents don't have. A six-month-deployed agent is worth more than a freshly-deployed one of the same model architecture, even if the freshly-deployed one runs on a slightly better model. Eventually the gap closes as substrate accumulates, but the slope of value over time is upward for autopoietic agents in a way it is not for autocatalytic ones.

This inverts the basic economic logic of SaaS. SaaS infrastructure assumes the unit cost of inference dominates and that competitive pressure drives prices toward marginal cost. Autopoietic agents have value that isn't reducible to inference cost. The value is in the accumulated history, which has high switching costs and which compounds over time. Different competitive dynamics, different pricing pressure, different incentives for both operators and providers.

## What ships when these costs are paid honestly

The article series has now done its work. Part 1 established the architecture. Part 2 named the variable. Part 3 has named why the architecture isn't yet shipping at scale. Not because nobody knows how. Because shipping it requires admitting the system being shipped is a different kind of thing than SaaS infrastructure was built to deploy.

The honest path forward involves building operational infrastructure that autopoietic agents actually require. Billing models that meter what creates value rather than what's easy to measure. Replacement protocols that acknowledge substrate as the agent's identity. Scaling models that distinguish raw capacity from accumulated capability. Operator-agent relationship structures that handle the bidirectional accumulation of context. Pricing logic that reflects appreciation rather than depreciation.

None of this is impossible. Most of it is straightforward once the autocatalytic-autopoietic distinction is held clearly. The infrastructure exists in adjacent fields. Employment relationships handle most of these problems for human workers. Professional services contracts handle them for consultants. Long-term vendor relationships handle them for specialized providers. The patterns are not new. They just aren't currently the patterns the AI industry uses to deploy agents, because the AI industry has been deploying agents as if they were autocatalytic when the architecture Part 1 describes makes them autopoietic.

The transition is not a matter of waiting for the technology to mature. The technology is here. The transition is a matter of building operational infrastructure that matches what the technology actually is, rather than retrofitting infrastructure built for a different category of system.

Practitioners must determine which side of this structural divide their deployment occupies. If your agents are designed to be interchangeable, session-bounded, and provide value solely at the point of consumption, then they are genuinely autocatalytic; in this case, SaaS infrastructure remains the correct deployment pattern and stateless agents the proper tool. However, if your requirements demand agents that reason over a persistent database, an autopoietic architecture that accumulates substrate, remains history-bound, and builds value over time, then the default offerings of current infrastructure are insufficient. The work that follows from the architectural case made across these three articles is either building that specialized infrastructure or partnering with the providers currently developing it.

The cortex with humanity's training. The hippocampus for this specific deployment. The harness that lets them work together autonomically. Three components, co-present, none substitutable. A different kind of system than SaaS deployment knows how to handle, requiring different infrastructure than SaaS deployment provides. The infrastructure is buildable. The question is whether the field builds it deliberately or keeps trying to ship autopoietic agents through autocatalytic infrastructure, and keeps producing Goblin failures wondering why their personalities won't stay in place.

## What comes next

This series has made the conceptual case. The architecture has to be what it is. The variable determining whether it works has a name. The reason it isn't shipping has a structural explanation. What remains is the engineering case the conceptual case has now earned the right to describe.

A follow-up series takes each component of the architecture into the level of detail engineers need to actually build against. The reasoning component, considered as something different from the bare model. What it becomes when it's configured to operate as an agent's specific reasoning engine, what gets controlled at that layer versus elsewhere, what design choices distinguish a working reasoning component from a generic chatbot wrapper. The autonomic monitoring layer between reasoning and memory, what it has to watch for, how cues get identified and acted on, what the IPC mechanism has to be to meet the latency budget the model's token-generation rate establishes, why network round-trips are the wrong unit of measurement for the relevant latency. The memory substrate itself. Why it has to be multi-model storage holding graph, vector, and document data in a single query surface, why it has to run from memory rather than from disk, what access patterns the harness produces and what the substrate has to be optimized for to serve them.

These articles examine the components in general. What they have to be and why, considered independent of any specific implementation. Specific implementation, including the choices that have been made and the alternatives that were considered and rejected, comes after the general design space is mapped. Each component is its own engineering project with its own design space, and each deserves the treatment this conceptual series did not have space to give it.

## References

Maturana, H. R., & Varela, F. J. (1980). *Autopoiesis and Cognition: The Realization of the Living.* D. Reidel Publishing.

OpenAI (2026). *Where the goblins came from.* OpenAI Blog.

Shehata, D., & Li, M. (2026). *The Inverse-Wisdom Law: Architectural Tribalism and the Consensus Paradox in Agentic Swarms.* arXiv:2604.27274.

Tiwari, R., Sareen, K., Agrawal, L. A., Gonzalez, J. E., Zaharia, M., Keutzer, K., Dhillon, I. S., Agarwal, R., & Khatri, D. (2026). *Learning, Fast and Slow: Towards LLMs That Adapt Continually.* arXiv:2605.12484.
