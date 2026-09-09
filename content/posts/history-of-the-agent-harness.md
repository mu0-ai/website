---
title: "The History of the AI Agent Harness"
date: 2026-09-10T08:00:00+10:00
draft: false
summary: "Most of the recent gains in AI agents came from the software wrapped around the model, not the weights. A history of the harness in three stages — the token loop, the static harness, and the harness that improves itself — alongside YC's own path from a shared bot to a centralised multi-user agent system."
tags: ["AI", "Agents", "Research"]
---

*Note: this post is a write-up of a talk on the history of the agent harness, covering the research lineage from GPT-2-era generation loops to systems such as Prime Agent, OpenJarvis and YC's internal QM harness. The ideas and reported numbers belong to the speakers. I have reorganised them into an essay and added my own framing where it seemed useful, but nothing here has been independently verified.*

---

A raw language model is a sequential processor. It receives tokens, predicts more tokens, and has no durable state and no independent way to affect the world. Everything that makes it an *agent* comes from the software wrapped around it: the context it is shown, the tools it can call, the memory it can read and write, the environments it can execute in, the feedback loops it is placed inside, the subagents it can spawn, the schedule it runs on and the permissions it is granted.

That wrapper is the **harness**, and the central claim of this history is simple: much of the recent improvement in AI agents has not come from changing model weights. It has come from changing the harness. The same model can perform dramatically differently depending on the harness it runs in.

The story has three broad stages.

1. **The minimal generation loop.** The model generated text until it hit a stop token.
2. **The static harness.** Designers progressively added examples, reasoning traces, tools, memory, skills, reflection, subagents and orchestration, but the harness itself remained fixed.
3. **The self-improving harness.** The agent can now revise its own prompts, memories, skills, agent structure and, in some systems, the harness code or even the weights.

Running alongside the research history is a practical one: YC's internal agents moved from a single shared bot, to coding agents in virtual machines, to a fleet of personal agents, and finally to a centralised multi-user harness in which persistent state is separated from disposable execution sandboxes. The two histories rhyme, and I think the second one is the more instructive.

## Why the harness stopped being "just scaffolding"

Harness work was initially dismissed as scaffolding, or prompt engineering, rather than serious technical work. That view became untenable once harness changes started producing large performance differences with no change to the underlying model.

ARC-AGI is the clearest example. A strong base model achieved only around 30 percent in an earlier setup, while newer general-purpose harnesses reportedly pushed the same class of models into the 90 percent range. Prime Agent reports 95.5 percent with Claude Opus in its setup, and an Nvidia system reached 100 percent. The exact comparisons depend on sandboxing, prompts, budgets and evaluation procedure. But that qualification is the point: the harness is now part of the system being evaluated, not an incidental wrapper around it.

A second example comes from automated research. What began as an interface for Karpathy's AutoResearcher gradually became a multi-agent research system. A user supplies a research purpose, seed experiments and a validation metric. A scoping agent searches papers and repositories, a principal-investigator agent directs the work, research agents run experiments, a council provides criticism, and an author agent freezes the idea, runs ablations and writes the paper. The system runs on remote GPU nodes, can be monitored through a cockpit, sends progress updates and continues largely unattended. The important change was not a new weight file. It was the surrounding organisation, persistence and control structure.

## Stage one: the minimal generation loop

The GPT-2 era, from February 2019, is harness version zero. Its structure was little more than:

- provide a prompt;
- ask the model for the next token;
- sample from the output distribution, for example with top-p sampling;
- append the token;
- stop when an end-of-sequence token appears.

For a benchmark such as GSM8K, the prompt might assign a simple persona ("You are a math teacher"), present the question and require a final answer after a delimiter. There were no tools, no editable memory, no reusable skills, no subagents and no independent execution environment. The model had to perform the entire task inside its fixed weights and its current token sequence.

This is the baseline. Nearly every subsequent change expanded one of four things: what information the model could **see**, how much computation it could **perform**, what actions it could **take**, or what state could **survive** beyond the immediate generation. It is worth keeping those four axes in mind, because the whole history fits on them.

## Stage two: the static harness expands

### In-context learning

The first major expansion was to place solved examples in the prompt. The few-shot work of 2020 showed that a model could infer a task from demonstrations supplied at inference time. This did not alter the weights, but it gave the harness a new way to condition behaviour for a particular task or domain.

It also established a pattern that runs through everything that follows: **useful capability can be stored outside the model and compiled into context when needed.**

### Chain of thought and test-time computation

Chain of thought changed the output space. Instead of forcing the model to jump directly from a problem to a short answer, the harness allowed it to spread intermediate computation across many tokens. The model could externalise steps, preserve intermediate results and spend more inference-time compute before committing.

This was an early form of test-time scaling. Capability was no longer determined only by the weights; it also depended on how the harness let the model spend its token budget.

### Tool use

WebGPT and Toolformer expanded the action space. The model could emit a structured tool call, usually JSON, and the harness would execute it and return the result to the context. Arithmetic could be delegated to Python, information retrieved from the web, external systems queried through APIs.

This changed the model from a closed text generator into a controller. It no longer needed to contain every fact or perform every operation internally. The harness became responsible for advertising tools, validating calls, executing them and appending their results.

### Editable memory

Systems such as MemGPT introduced explicit read and write access to context and longer-term memory. Previously, interaction history was effectively append-only: the agent could add tokens but could not deliberately reorganise or remove old information.

Memory tools introduced create, read, update and delete operations over selected state. The context became an actively managed resource, and immediate conversational context was separated from information intended to survive beyond the current window.

### Skills

Voyager showed how successful sequences of actions could be distilled into reusable skills. In its Minecraft environment, the agent chained tools to accomplish a task and then preserved the resulting procedure for future use.

A modern skill is typically a named instruction set or program stored outside the active prompt. The harness retrieves it when relevant and compiles it into the agent's context. Skills sit in a middle layer between raw tool calls and retraining: learned procedures persist without changing the weights.

### Code as a general action language

Work such as InterCode expanded the action space again by letting the model write and execute code. Instead of requiring the harness developer to anticipate every useful API call, the agent could compose operations programmatically and create task-specific procedures on demand.

This matters because code compresses long sequences of actions. An agent can manipulate a large object, run a search or test many hypotheses inside a runtime without placing every intermediate value into the token context.

### Acting, reflecting and revising

ReAct, Self-Refine and Reflexion added feedback loops. A model could act, inspect an outcome, critique the result and try again. Feedback might come from an internal evaluator, another agent or the external environment.

The key change was that a first answer ceased to be final. The harness allocated multiple turns and used errors as additional context. This improved reliability, but the improvement procedure was still prescribed by the system designer.

### Multi-agent systems and recursive delegation

The next expansion made subagent creation itself a tool. A root agent could delegate focused tasks to separate model sessions, gather their reports and continue coordinating. Persistent subagents could retain the context they had developed and be resumed later rather than recreated from scratch.

Recursive Language Models pushed this further. An agent could programmatically call another language-model session over a subset of a larger problem, and that session could recursively delegate again. The root session became an orchestrator over a tree or swarm of specialised contexts.

By this point the static harness included an agent specification, a system prompt, limits on turns and tool calls, a tool catalogue, a skill catalogue, subagent definitions, session management, context compilation and an execution loop. It could start in response to a message or wake on a schedule. It was far more capable than the original token loop, but its overall architecture and instructions were still designed in advance.

## Context becomes a memory hierarchy

Prime Agent frames modern agent state as a hierarchy, much like the memory hierarchy of a computer.

| Layer | What it contains | Why it exists |
| --- | --- | --- |
| Model weights | Fast, implicit knowledge and behaviour | Immediately available but expensive to update |
| Active context | Current instructions, messages, observations and examples | Directly visible to the model but limited in size and costly in tokens |
| Live runtime state | Variables, programs and active subagents in a persistent Python or IPython environment | Lets large or structured information be processed without serialising everything into tokens |
| Durable storage | Files, memories, skills, prompts and archived sessions | Persists information beyond a context window or process lifetime |

Compaction was an early bridge between these layers: the harness summarises older context so that work can continue past the nominal window. Newer systems go further. They clean up runtime variables, offload inactive subagents, refine stored memories and revise skills or prompts. Context management starts to look like garbage collection and cache management rather than transcript truncation.

This also changes the conceptual model of an agent. A raw LLM resembles a Turing machine reading and writing a token tape. A harness resembles a von Neumann computer, with working memory, storage, programs and input-output devices. The analogy is not that one is formally more computable than the other. It is that the harness makes useful state and operations directly addressable.

## Stage three: the harness begins to improve itself

The current phase moves from a fixed harness to a system that can revise the machinery around the model.

### Optimising prompts

DSPy treats prompts and demonstrations as optimisable program components. Given training examples and an evaluation measure, it searches for prompt configurations that perform better. One way to put it: an optimisation process is given controlled write access to the system prompt.

This replaces manual prompt tweaking with a measured search loop. The harness remains structurally fixed, but its instructions evolve in response to evidence.

### Optimising the harness code

Darwin-style systems go further by allowing candidate agents to modify the harness itself. A meta-harness generates or mutates combinations of prompts and harness code, evaluates them against a fitness function and retains stronger candidates in an archive.

The object being optimised is no longer the answer or the prompt. It is the agent design: how contexts are compiled, how agents are arranged, which loops they use and which capabilities are exposed. The meta-harness is a harness for producing better harnesses.

### Continual refinement and test-time learning

Continual Harness adds a richer history of trajectories and outcomes. The agent can use prior work to revise its system prompt, memories, skills and reusable subagent specifications. The next step is online updates to the weight file using newly collected examples, bringing test-time experience back into training.

This addresses a real limitation of current systems. Agents generate large amounts of valuable experience while operating, but most do not efficiently incorporate it. In-context learning saturates, while LoRA, supervised fine-tuning and other weight updates remain separate workflows. The research direction is to make those levels of adaptation part of one continuous system.

## Prime Agent as a long-horizon research harness

Prime Agent combines many of these ideas into a first-principles harness for long-running work.

The user enters through an agents view showing parallel sessions. Each project has a root orchestrator that can create subagents automatically. Tools, memories and subagents are exposed programmatically through a persistent IPython environment. A background daemon keeps sessions alive after the visible terminal or laptop session closes.

Several design choices stand out:

- **Persistent subagents.** Completed subagents become idle sessions rather than being discarded. The parent can message and reactivate them with their earlier context intact.
- **Agent-to-agent messaging.** Parents, children and siblings exchange information directly, reducing the need for a human to relay context.
- **Programmatic context processing.** The agent operates on data inside the runtime rather than repeatedly inserting it into prompts, which reduces token cost.
- **Refinement across runs.** Prompts, memories, skills and agent specifications are updated from earlier trajectories.
- **Long-horizon evaluation.** Performance is measured by the practical plateau reached as more time and tokens are supplied, not by a short fixed run.

The reported results illustrate why these properties matter, with appropriate caveats. After correcting an initially invalid ARC-AGI run that had leaked information, Prime Agent reportedly achieved 78 percent with one model and 95.5 percent with Claude Opus, using a general ARC prompt plus its standard runtime capabilities. On other coding and kernel benchmarks, results were closer to parity with established harnesses, which suggests the benefit is task-dependent rather than universal.

The more revealing tests were long-running ones. In a week-long research experiment on an eight-H200 system, agents ran cheaper out-of-loop CPU experiments to choose better expensive GPU experiments. In a seven-day Factorio run, 633 agents generated 23 million output tokens and kept progressing through the technology tree rather than becoming permanently stuck. These behaviours support the case for persistent memory, delegation and refinement even where benchmark score differences are noisy.

## OpenJarvis and the move to personal local agents

OpenJarvis represents a different direction at the frontier: moving the personal AI stack from cloud services onto user-owned devices.

Its designers reduce a personal agent stack to five composable areas:

1. user interfaces;
2. agent logic;
3. intelligence and its inference engine;
4. tools and memory;
5. learning and optimisation.

The goal is to make model, inference, agent execution, memory and learning independently replaceable. The same harness can select among local models, engines and hardware, while tools are exposed through standard interfaces such as MCP.

The interesting move is that the project uses frontier cloud models during development to optimise a cheaper local configuration. A cloud model diagnoses weaknesses and proposes changes to prompts, model selection, tools or agent logic. The optimised system is then deployed locally, avoiding repeated cloud inference costs and keeping personal data on the device. The reported figure is up to an 800-fold cost reduction for selected workloads, along with latency improvements, with the acknowledgement that small local models still cannot handle every task.

This extends the history in two ways. The harness becomes a portable specification rather than an inseparable product around one model. And optimisation can cross system boundaries: a powerful cloud agent can improve a smaller local agent that later runs independently.

## YC's practical path from a shared bot to QM

The research history is instructive, but the product history is where the constraints become concrete.

### January 2025: a shared general agent

YC's first internal system was a straightforward system prompt, a collection of tools and an execution loop. Everyone interacted with essentially the same agent. Despite its simplicity, it was useful for questions about internal data, and its scope improved as the underlying models improved.

YC gradually connected it to Slack, added scheduled jobs and exposed more tools. The architecture did not change radically. Better models and more integrations made the same basic loop useful across more domains.

### June 2025: coding agents in virtual machines

As engineers adopted Claude Code and Codex, YC realised that coding agents could run inside virtual machines and be controlled through Slack. A person could describe a bug or a desired change, and the agent could edit code, run CI and start a development environment for testing.

This made software changes accessible to employees who had never contributed code. YC also established a manual improvement loop: staff examined failures and updated the repository's agent instructions so that future runs handled similar situations better. That is stage three of the research history, done by hand.

### Early 2026: OpenClaw as a personal computer-using agent

YC partners then began using OpenClaw. The important difference was that each agent effectively had its own computer. That made it far more customisable and let it behave like a personal assistant rather than a shared question-answering service.

The fit was good because partners faced heavy email, application-review and scheduling workloads. Giving the agent its own environment let users install tools and shape workflows around their individual work.

### April 2026: a fleet of Hermes agents

YC then tried to give the same experience to every employee without buying a separate physical computer for each person. It provisioned more than 50 Hermes agents in virtual machines.

The fleet preserved personalisation, but it exposed serious operational problems. Each instance needed configuration and maintenance. Fixes often required an administrator to connect to individual machines, a recurring game of whack-a-mole. Sessions, state and the agent's identity were trapped inside each VM.

This stage revealed that "one agent equals one permanent computer" is powerful for the user but a poor systems boundary for a centrally administered product.

### May 2026 onward: QM separates the brain from the sandbox

QM was built to retain the benefits of personal agents while removing the fleet-management problem. Its decisive architectural move was to pull the agent's persistent brain out of the sandbox.

Conversations and state are centralised in PostgreSQL. Sandboxes no longer define where an agent lives; they are execution resources the agent uses when needed. A user still has a personal context with files, memories and scheduled work, and people can invoke the system together in a Slack channel. But the durable identity of the agent is no longer tied to a particular virtual machine.

That separation buys a lot:

- sessions can be administered and inspected centrally;
- an agent can see the context it is permitted to use across the wider system;
- execution environments can be disposable;
- the agent can choose a lightweight sandbox for simple work or a more powerful machine for demanding development tasks;
- multiple agents or users can converge on shared environments when collaboration is useful;
- conversation traces form an evaluation set for improving the system.

QM also lets the agent select its model provider at runtime, routing around model-specific weaknesses or refusals and using a model appropriate to the task. Likewise it can choose among sandbox providers rather than having those choices hard-coded.

The team deliberately keeps the core harness thin. It describes three foundational capabilities: execute in a remote sandbox, read and write object storage, and publish internal applications. Memory and scheduling tools remain important today, but the designers expect stronger models to eventually absorb more of the explicit orchestration logic.

### The grind tool and explicit effort budgets

One practical weakness remained: agents often stopped too early. QM introduced a "grind" mechanism that assigns a minimum wall-clock or token budget to a goal. The agent is not permitted to abandon the task before spending that budget.

This is test-time scaling expressed as product infrastructure. Instead of prescribing every reasoning step, the harness creates the conditions for sustained work. Multi-hour budgets reportedly improve research reports and other ordinary office tasks.

### Human review and permissions remain essential

QM keeps most direct database access read-only. For writes, the agent can propose bulk updates that a human reviews before execution. The team notes that reviewers begin to rubber-stamp these approvals as trust grows, so approval design has to account for human behaviour rather than merely placing a confirmation dialog in the loop.

The hardest unresolved issue is social context. A human usually understands implicitly where information may be repeated and where it is confidential. An agent does not reliably share that judgement. Once state is centralised and available across personal and group conversations, the amount of knowledge the agent can safely use is bounded by the quality of the permission system.

YC can build on an existing fine-grained authorisation system, but the general problem remains open. The modern harness is therefore not only an orchestration and memory system. It is also an identity, security and information-governance system.

## What changed at each transition

| Transition | Previous limitation | Change introduced | Result |
| --- | --- | --- | --- |
| Token loop to few-shot prompting | Behaviour was largely fixed by weights and one instruction | Demonstrations compiled into context | Fast task adaptation without retraining |
| Direct answers to chain of thought | Too little inference-time computation | Intermediate reasoning used additional tokens | More complex problems became tractable |
| Text generation to tool use | Model could only emit text | Harness executed structured actions | Access to computation, retrieval and external systems |
| Append-only context to memory CRUD | State could only accumulate until the window filled | Agent could create, retrieve, revise and delete memories | Deliberate long-term state management |
| Tool calls to skills and code | Useful procedures had to be rediscovered | Procedures and programs became reusable | Faster, more consistent execution |
| Single pass to reflection | First response was final | Evaluation and retry loops added | Better error correction |
| One session to multi-agent orchestration | One context had to contain the entire problem | Work delegated to specialised persistent sessions | Parallelism and context isolation |
| Fixed harness to self-improving harness | Prompts and orchestration stayed static | System could optimise prompts, skills, topology or code | Learning moved into the surrounding system |
| Permanent agent computer to portable sandboxes | Personalisation created an unmanageable VM fleet | Durable state moved to a central store; compute became selectable | Central administration with retained personalisation |
| Short task execution to long-horizon operation | Agents stopped early or lost context | Daemons, schedules, budgets, compaction and persistent subagents | Multi-hour and multi-day work |

## Where harnesses are now

The modern harness is becoming an agent operating system. It gives a model persistent identity, a hierarchy of memory, programmatic computation, tools, reusable procedures, parallel workers, scheduling, evaluation, model and compute selection, security boundaries and mechanisms for self-improvement.

None of this argues for endlessly complicated, hard-coded workflows. The direction is toward **thin but expressive** harnesses. Earlier systems forced agents through explicit plan-act-critique sequences because the models needed that structure. Stronger models can increasingly choose their own procedure. The harness should therefore expose the capabilities a model cannot create by thought alone: persistence, execution, communication, storage, feedback, permissions and a budget within which to work.

The frontier is no longer a better chatbot loop. It is a persistent system that can operate over days, coordinate many contexts, learn from its own traces, choose where and how to run, and gradually improve the instructions and infrastructure surrounding its fixed model. The remaining bottlenecks are less about generating fluent text and more about managing experience, cost, security, social boundaries and reliable improvement over long periods.

## A concise chronology

- **February 2019:** GPT-2-style generation loop: prompt, sample tokens, stop at end of sequence.
- **2020:** few-shot prompting makes examples part of the inference-time program.
- **Following years:** chain of thought expands test-time reasoning; WebGPT and Toolformer add external actions.
- **Memory era:** MemGPT gives the agent editable memory rather than append-only context.
- **Skills and code:** Voyager preserves successful procedures; code execution makes action composition open-ended.
- **Reflection era:** ReAct, Self-Refine and Reflexion introduce observation, criticism and retries.
- **Multi-agent era:** persistent subagents and recursive delegation turn one context into an orchestrated system.
- **January 2025:** YC deploys a shared general internal agent with tools and a loop.
- **June 2025:** YC runs coding agents in VMs through Slack and improves repository instructions from observed failures.
- **Early 2026:** YC partners adopt OpenClaw-style personal agents with their own computers.
- **April 2026:** YC provisions more than 50 Hermes VM agents and hits configuration and fleet-management problems.
- **From May 2026:** QM centralises state in PostgreSQL, treats sandboxes as selectable resources and supports personal and multiplayer work.
- **Current frontier:** DSPy, Darwin machines, Continual Harness and Prime Agent optimise prompts, memories, skills, agent structures and sometimes harness code or weights; OpenJarvis applies the same principles to optimised local-first personal AI.

The overall trajectory is from **generation**, to **action**, to **persistence**, to **coordination**, and finally to **adaptation of the agent system itself**.
