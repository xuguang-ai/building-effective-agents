# Building effective AI agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic Research, "Building effective AI agents" (originally titled "Building effective agents") |
| URL | https://www.anthropic.com/research/building-effective-agents |
| Authors | Erik Schluntz (now bylined as Erik S.) and Barry Zhang |
| Published | 2024-12-19 |
| Accessed | 2026-11-02 (current page), compared section by section against the Wayback Machine snapshot of 2025-01-05; differences listed at the end |

The page now carries a note at the top: much of the tooling landscape described in the post has changed since December 2024, and Anthropic's current approach is described in the Claude Managed Agents engineering post and documentation. The body of the article itself is unchanged.

## One-sentence takeaway

The most successful agent implementations do not rely on complex frameworks or specialized libraries. They are built from **simple, composable patterns**. Start with the simplest solution and add complexity only when measurement shows that it pays off.

## 1. What is an agent: workflows vs. agents

Anthropic groups all the variants under the umbrella term **agentic systems**, but draws a firm architectural distinction:

- **Workflows**: LLMs and tools are orchestrated through **predefined code paths**. The code owns the control flow.
- **Agents**: the LLM **dynamically directs its own process and tool usage**, keeping control over how it accomplishes the task. The model owns the control flow.

## 2. When (and when not) to use agents

- Find the simplest solution possible and add complexity only when needed. That may mean not building an agentic system at all.
- Agentic systems trade **latency and cost** for **better task performance**. Decide first whether that trade is worth it.
- When more complexity is warranted: well-defined tasks that need predictability and consistency call for a **workflow**; flexibility and model-driven decision-making at scale call for an **agent**.
- For many applications, **a single LLM call with retrieval and in-context examples**, well optimized, is enough.

## 3. Frameworks: whether and how to use them

- Frameworks named in the current version: Claude Agent SDK, Strands Agents SDK (AWS), Rivet (a drag-and-drop GUI workflow builder) and Vellum (a GUI tool for building and testing complex workflows).
- **Benefit**: they simplify low-level chores such as calling LLMs, defining and parsing tools, and chaining calls, so getting started is easy.
- **Cost**: extra layers of abstraction hide the underlying prompts and responses and make debugging harder. They also tempt you to add complexity when a simpler setup would do.
- **Advice**: start by using LLM APIs directly; many patterns take only a few lines of code. If you do use a framework, understand the code underneath it. Wrong assumptions about what is under the hood are a common source of customer error.
- Sample implementations live in the Anthropic cookbook (patterns/agents/basic_workflows).

## 4. The building block and five workflow patterns

### 4.0 Building block: the augmented LLM

- An LLM enhanced with **retrieval, tools and memory**. Current models use these actively: they generate their own search queries, select the right tools and decide what information to retain.
- Two implementation priorities: **tailor** these capabilities to your use case, and give the LLM an **easy, well-documented interface**.
- One way to implement them is the Model Context Protocol (MCP), which lets a simple client integrate with a growing ecosystem of third-party tools.
- The rest of the article assumes every LLM call has these augmented capabilities.

### 4.1 Workflow: prompt chaining

- **How**: decompose the task into a fixed sequence of steps, each LLM call processing the previous call's output. Programmatic checks ("gates" in the diagram) on intermediate steps confirm the process is still on track.
- **When**: the task decomposes cleanly into fixed subtasks. The goal is to **trade latency for accuracy** by making each call an easier task.
- **Examples**: generate marketing copy, then translate it into another language; write a document outline, check it against criteria, then write the document from the outline.

### 4.2 Workflow: routing

- **How**: classify the input and send it to a specialized follow-up task. This separates concerns and allows more specialized prompts. Without it, optimizing for one kind of input can hurt performance on others.
- **When**: the task has distinct categories that are better handled separately, and classification can be done accurately, by an LLM or by a traditional classifier.
- **Examples**: send customer-service queries (general questions, refund requests, technical support) to different downstream processes, prompts and tools; send easy or common questions to a smaller, cost-efficient model (Claude Haiku 4.5) and hard or unusual ones to a more capable model (Claude Sonnet 4.5).

### 4.3 Workflow: parallelization

- **How**: several LLM calls work simultaneously and their outputs are aggregated programmatically. Two variants:
  - **Sectioning**: split the task into independent subtasks run in parallel.
  - **Voting**: run the same task several times to get diverse outputs.
- **When**: subtasks can run in parallel for speed, or multiple perspectives or attempts are needed for higher confidence. For complex tasks with several considerations, LLMs generally do better when each consideration gets its own call with focused attention.
- **Examples**:
  - Sectioning: guardrails, where one model instance handles the user query while another screens it for inappropriate content (this beats one call doing both); automated evals, where each call scores a different aspect of performance.
  - Voting: code vulnerability review, where several prompts review the code and flag problems; content moderation, where several prompts evaluate different aspects or use different vote thresholds to balance false positives and negatives.

### 4.4 Workflow: orchestrator-workers

- **How**: a central LLM dynamically breaks the task down, delegates subtasks to worker LLMs and synthesizes their results.
- **When**: complex tasks where **the subtasks cannot be predicted in advance**. In coding, for example, how many files need changing and how depends on the task. Topologically similar to parallelization, but the key difference is **flexibility**: subtasks are not predefined, the orchestrator decides them from the specific input.
- **Examples**: coding products that make complex changes across multiple files; search tasks that gather and analyze information from multiple sources.
- Note: the article classifies this as a workflow, but because an LLM decides the subtasks it already sits between workflow and agent.

### 4.5 Workflow: evaluator-optimizer

- **How**: one LLM call generates a response while another evaluates it and gives feedback, in a loop.
- **When**: there are **clear evaluation criteria** and iterative refinement provides measurable value. Two signs of fit: LLM responses demonstrably improve when a human articulates feedback, and the LLM can provide that feedback itself. Analogous to a human writer polishing a document.
- **Examples**: literary translation, where the translator LLM misses nuances that an evaluator LLM can critique; complex search tasks that need several rounds of searching and analysis, where the evaluator decides whether to keep searching.

## 5. Agents

- **Why now**: LLMs have matured in the key capabilities: understanding complex inputs, reasoning and planning, using tools reliably, and recovering from errors.
- **How they run**:
  1. Start from a command or an interactive discussion with the user. Once the task is clear, plan and operate independently, returning to the human for information or judgement when needed.
  2. At every step, obtain **ground truth** from the environment (tool results, code execution) to assess progress.
  3. Pause for human feedback at checkpoints or when blocked.
  4. Terminate on completion, usually with **stopping conditions** such as a maximum number of iterations to keep control.
- **Implementation is often straightforward**: an LLM using tools based on environmental feedback, in a loop. That makes clear, thoughtful design of the toolset and its documentation critical (see Appendix 2).
- **When**: open-ended problems where the number of steps is hard or impossible to predict and no fixed path can be hard-coded. The LLM may run for many turns, so you need some level of trust in its decisions. Autonomy makes agents ideal for scaling tasks in **trusted environments**.
- **Costs and risks**: higher cost and the potential for compounding errors. Test extensively in **sandboxed environments** with appropriate guardrails.
- **Examples** (Anthropic's own implementations): a coding agent that resolves SWE-bench tasks by editing many files from a task description; the "computer use" reference implementation, where Claude operates a computer to accomplish tasks.

## 6. Combining and customizing the patterns

- The patterns are not prescriptive. They are common shapes that developers can adapt and combine to fit their use case.
- The key to success, as with any LLM feature, is to **measure performance and iterate**. Add complexity only when it demonstrably improves outcomes.

## 7. Three core principles

Success is not about building the most sophisticated system but the **right system for your needs**. Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short.

1. **Simplicity**: keep the agent's design simple.
2. **Transparency**: explicitly show the agent's planning steps.
3. **A well-crafted agent-computer interface (ACI)**: thorough tool documentation and testing.

Frameworks help you start quickly, but do not hesitate to reduce abstraction layers and build with basic components as you move to production.

## Appendix 1: agents in practice

What the two domains share: tasks that need both **conversation and action**, **clear success criteria**, **feedback loops**, and **meaningful human oversight**.

### A. Customer support

- Support interactions follow a natural conversation flow while needing external information and actions. Tools can pull customer data, order history and knowledge-base articles. Actions such as refunds or ticket updates can be handled programmatically. Success is clearly measurable through user-defined resolutions.
- Several companies charge **only for successful resolutions** under usage-based pricing, a sign of confidence in their agents.

### B. Coding agents

- Code solutions are verifiable through automated tests, agents can iterate using test results as feedback, the problem space is well defined and structured, and output quality can be measured objectively.
- Anthropic's own agents solve real GitHub issues in SWE-bench Verified from the pull-request description alone. Automated tests verify functionality, but **human review** remains crucial for making sure solutions fit broader system requirements.

## Appendix 2: prompt engineering your tools

- Tool definitions and specifications deserve **as much prompt-engineering attention as the overall prompt**.
- The same action can often be specified in several formats: a file edit as a diff or as a full rewrite, structured output as code inside markdown or inside JSON. For humans these differences are cosmetic and losslessly convertible, but for an LLM some formats are much harder: a diff requires knowing the number of changed lines for the chunk header before the new code is written, and code inside JSON needs extra escaping of newlines and quotes.
- **Three suggestions for choosing tool formats**:
  - Give the model enough tokens to "think" before it writes itself into a corner.
  - Keep the format close to what the model has seen naturally in text on the internet.
  - Avoid formatting "overhead" such as keeping an accurate count of thousands of lines of code or string-escaping the code it writes.
- **Rule of thumb**: invest as much effort in the agent-computer interface (ACI) as goes into human-computer interfaces (HCI). Concretely:
  - **Put yourself in the model's shoes**: is it obvious how to use the tool from its description and parameters? If a human would have to think carefully, so will the model. Good tool definitions include example usage, edge cases, input-format requirements and clear boundaries from other tools.
  - **Rename parameters and rewrite descriptions** to make things obvious, as if writing a great docstring for a junior developer. This matters most when many tools are similar.
  - **Test how the model uses the tools**: run many example inputs in the workbench, see what mistakes the model makes, and iterate.
  - **Poka-yoke (mistake-proof) the tools**: change the arguments so mistakes are harder to make.
- **Case**: while building the SWE-bench agent, Anthropic spent more time optimizing tools than the overall prompt. The model made mistakes with relative file paths once the agent had moved out of the root directory. Changing the tool to require **absolute file paths** fixed it: the model then used the tool flawlessly.

## Quick reference

| Pattern | Type | Core mechanism | Fit signals | Examples in the article |
| --- | --- | --- | --- | --- |
| Augmented LLM | building block | LLM + retrieval / tools / memory | foundation for every pattern | tools via MCP |
| Prompt chaining | workflow | fixed sequence of calls with gates | task splits cleanly into fixed steps; latency traded for accuracy | copy then translate; outline, check, write |
| Routing | workflow | classify, then dispatch | distinct input categories; accurate classification | support triage; small model for easy questions, large for hard ones |
| Parallelization | workflow | sectioning / voting, programmatic aggregation | independent subtasks; or multiple perspectives for confidence | guardrails; automated evals; vulnerability-review voting |
| Orchestrator-workers | workflow (agent-like) | LLM decomposes, delegates, synthesizes | subtasks cannot be enumerated in advance | multi-file code changes; multi-source search |
| Evaluator-optimizer | workflow | generate, evaluate, feed back, repeat | clear evaluation criteria; measurable gains from iteration | literary translation; multi-round search |
| Agent | agent | tools + environmental feedback + loop + stop conditions | open-ended, unpredictable step count, trusted environment | SWE-bench coding agent; computer use |

## Changes between the 2024 original and the current page

A section-by-section diff of the Wayback Machine snapshot of 2025-01-05 against the page as read on 2026-11-02 shows the patterns, principles and both appendices unchanged word for word. The only changes:

- Title changed from "Building effective agents" to "Building effective AI agents".
- A note was added at the top pointing to the Claude Managed Agents post and the Managed Agents documentation.
- Framework list: originally LangGraph from LangChain, Amazon Bedrock's AI Agent framework, Rivet and Vellum; now Claude Agent SDK, Strands Agents SDK by AWS, Rivet and Vellum.
- Models in the routing example: originally Claude 3.5 Haiku / Claude 3.5 Sonnet "to optimize cost and speed"; now Claude Haiku 4.5 / Claude Sonnet 4.5 "to optimize for best performance".
- Byline changed from Erik Schluntz to Erik S.
