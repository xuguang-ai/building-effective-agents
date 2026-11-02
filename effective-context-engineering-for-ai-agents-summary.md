# Effective context engineering for AI agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Effective context engineering for AI agents" |
| URL | https://www.anthropic.com/research/effective-context-engineering-for-ai-agents (first published under /engineering/) |
| Authors | Anthropic's Applied AI team: Prithvi Rajasekaran, Ethan Dixon, Carly Ryan and Jeremy Hadfield, with contributions from Rafi Ayub, Hannah Moran, Cal Rueb and Connor Jennings |
| Published | 2025-09-29 |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-09-30; differences listed at the end |
| Companion | Memory and context management cookbook on platform.claude.com |

## One-sentence takeaway

Context is a finite resource with diminishing returns, so the job at every step of an agent loop is to find **the smallest set of high-signal tokens that maximizes the likelihood of the desired outcome**.

## 1. Context engineering vs. prompt engineering

- **Context** is the set of tokens included when sampling from an LLM. The engineering problem is to optimize the utility of those tokens against the model's inherent constraints so that it consistently produces the desired outcome. This means "thinking in context": considering the whole state available to the model at a given moment and the behaviors that state might yield.
- **Prompt engineering** covers writing and organizing instructions for optimal outcomes, above all system prompts. It dominated when most use cases were one-shot classification or generation.
- **Context engineering** is its natural progression: the strategies for curating and maintaining the optimal set of tokens during inference, including everything that lands in context beyond the prompt: system instructions, tools, Model Context Protocol (MCP) servers, external data, message history.
- An agent in a loop generates ever more data that could matter for the next turn, and that information must be refined cyclically. Unlike the discrete act of writing a prompt, context engineering is iterative: curation happens every time we decide what to pass to the model.
- The post's diagram: single-turn prompt engineering is system prompt plus user message into the model, out comes an assistant message. Context engineering for agents starts from a large pool of possible context (documents, tools, memory files, comprehensive instructions, domain knowledge, message history), curates a subset into the window, and feeds tool results back into the pool for the next turn.

## 2. Why it matters: context rot and the attention budget

- LLMs, like humans, lose focus past a point. Needle-in-a-haystack benchmarks revealed **context rot**: as the number of tokens in the window grows, the model's ability to recall information from it accurately declines. Some models degrade more gently than others, but it shows up in all of them.
- Context must therefore be treated as a **finite resource with diminishing marginal returns**. Like human working memory, an LLM has an "attention budget", and every new token depletes it a little.
- Architectural reasons: transformers let every token attend to every other token, so n tokens create n² pairwise relationships, and that capacity is stretched thin as context grows. Models also learn their attention patterns from training data in which short sequences are far more common than long ones, so they have less experience with, and fewer specialized parameters for, context-wide dependencies. Position encoding interpolation lets models handle longer sequences than they were trained on, with some loss in understanding token positions.
- The result is a **performance gradient, not a cliff**: models remain highly capable at long context but lose some precision in information retrieval and long-range reasoning compared with shorter contexts.

## 3. The anatomy of effective context

Guiding principle: the smallest possible set of high-signal tokens that maximizes the likelihood of the desired outcome. Applied per component:

### System prompts

- Extremely clear, simple, direct language at the **right altitude**, the Goldilocks zone between two failure modes. Too specific: complex, brittle logic hardcoded into the prompt to force exact behavior, which is fragile and costly to maintain. Too vague: high-level guidance that gives no concrete signals or falsely assumes shared context. Just right: specific enough to guide behavior, flexible enough to hand the model strong heuristics.
- The post's illustration uses a bakery support agent. The too-specific prompt is a numbered procedure of intent classification rules with nested if-this-then-that cases. The just-right prompt states the role, the systems available (order management, product catalog, store policies), a four-step response framework (identify the core issue, gather context with tools, give a clear resolution, confirm satisfaction) and a short list of guidelines such as when to call the human_assistance tool. The too-vague prompt is two sentences about solving issues "consistent with the principles and essence of the company brand".
- Organize prompts into distinct sections (for example background information, instructions, tool guidance, output description) using XML tags or Markdown headers, though exact formatting matters less as models improve.
- Aim for the **minimal set of information that fully outlines the expected behavior**. Minimal does not mean short: the agent still needs enough up front to behave as intended. Start by testing a minimal prompt with the best model available, then add clear instructions and examples based on the failure modes seen in testing.

### Tools

- Tools let the agent act on its environment and pull in new context. Because they define the contract between the agent and its information and action space, they must promote efficiency: token-efficient returns and encouragement of efficient agent behavior.
- As in the companion post on writing tools: tools should be self-contained, robust to error and extremely clear about their intended use, like functions in a well-designed codebase; input parameters descriptive, unambiguous and playing to the model's strengths.
- The most common failure mode is a **bloated tool set** that covers too much or creates ambiguous decision points. If a human engineer cannot say definitively which tool applies in a situation, an agent cannot be expected to do better. A minimal viable set of tools also makes maintenance and context pruning easier over long interactions.

### Examples

- Few-shot examples remain a strongly advised best practice. But do not stuff a laundry list of edge cases into the prompt to articulate every rule. Curate a set of **diverse, canonical examples** that portray the expected behavior. For an LLM, examples are the "pictures" worth a thousand words.

Across all components (system prompt, tools, examples, message history), keep the context informative yet tight.

## 4. Context retrieval and agentic search

- Since "Building effective agents", Anthropic has gravitated toward a simple definition of agents: **LLMs autonomously using tools in a loop**. The field is converging on this paradigm, and as models improve, agent autonomy scales with them.
- The shift: many AI-native applications use embedding-based retrieval before inference to surface context. Agentic approaches increasingly add **"just in time" context**: the agent keeps lightweight identifiers (file paths, stored queries, web links) and loads data at runtime through tools. Claude Code analyzes large databases this way, writing targeted queries, storing results, and using Bash commands such as head and tail to inspect large volumes without loading whole data objects into context. It mirrors human cognition: we do not memorize corpora, we build file systems, inboxes and bookmarks and retrieve on demand.
- The **metadata of references** refines behavior. To an agent in a file system, test_utils.py inside a tests folder implies a different purpose than the same name under src/core_logic/. Folder hierarchies, naming conventions and timestamps all signal how and when to use information.
- Autonomous navigation enables **progressive disclosure**: the agent discovers relevant context incrementally. File sizes suggest complexity, names hint at purpose, timestamps proxy relevance. It builds understanding layer by layer, keeps only what is necessary in working memory, and uses note-taking for persistence, staying focused on relevant subsets instead of drowning in exhaustive but irrelevant information.
- The trade-off: runtime exploration is slower than retrieving pre-computed data, and it takes opinionated engineering to give the model the right tools and heuristics. Without guidance an agent wastes context misusing tools, chasing dead ends, or missing key information.
- **Hybrid strategy**: retrieve some data up front for speed, then explore autonomously at the agent's discretion. Claude Code does this: CLAUDE.md files are dropped into context up front, while primitives such as glob and grep retrieve files just in time, sidestepping stale indexes and complex syntax trees. The hybrid approach suits less dynamic content such as legal or finance work.
- As models improve, agent design will let intelligent models act intelligently with progressively less human curation. "Do the simplest thing that works" remains the best advice.

## 5. Context engineering for long-horizon tasks

Tasks that run from tens of minutes to many hours, such as large codebase migrations or comprehensive research, exceed any context window. Waiting for larger windows will not remove context pollution and relevance problems where the strongest performance is required. Three techniques:

### Compaction

- Summarize a conversation nearing the window limit and start a new window with the summary. Usually the first lever for long-term coherence; done well, it distills the window with high fidelity so the agent continues with minimal degradation.
- In Claude Code the model summarizes the message history, preserving architectural decisions, unresolved bugs and implementation details while discarding redundant tool outputs and messages. The agent continues with the compressed context plus the five most recently accessed files.
- The art is choosing what to keep and what to drop; aggressive compaction loses subtle context whose importance only becomes clear later. Tune the compaction prompt on complex agent traces: **maximize recall first**, then iterate to improve precision by removing superfluous content.
- The lowest-hanging fruit is clearing old tool calls and results. Once a tool has been called deep in the history, the agent rarely needs the raw result again. Tool result clearing is one of the safest, lightest-touch forms of compaction, and ships as a feature on the Claude Developer Platform.

### Structured note-taking (agentic memory)

- The agent regularly writes notes to memory outside the context window and pulls them back in later. Persistent memory with minimal overhead: Claude Code's to-do list, or a custom agent maintaining a NOTES.md file, tracks progress and dependencies that would otherwise be lost across dozens of tool calls.
- Claude playing Pokémon shows the effect outside coding: precise tallies across thousands of steps ("for the last 1,234 steps I've been training my Pokémon in Route 1, Pikachu has gained 8 levels toward the target of 10"), maps of explored regions, unlocked achievements and combat notes, all without being prompted about memory structure. After context resets it reads its own notes and continues multi-hour training or dungeon runs.
- With the Sonnet 4.5 launch, Anthropic released a memory tool in public beta on the Claude Developer Platform: a file-based system for storing and consulting information outside the window, so agents can build knowledge bases, keep project state across sessions and reference previous work.

### Sub-agent architectures

- Specialized sub-agents handle focused tasks with clean context windows while the main agent coordinates from a high-level plan. Each sub-agent may spend tens of thousands of tokens exploring but returns a condensed summary, often 1,000 to 2,000 tokens.
- Separation of concerns: detailed search context stays inside the sub-agents, the lead agent synthesizes and analyzes. The pattern, described in "How we built our multi-agent research system", showed a substantial improvement over single-agent systems on complex research tasks.

### Choosing among the three

- Compaction keeps conversational flow for tasks with extensive back-and-forth.
- Note-taking excels at iterative development with clear milestones.
- Multi-agent architectures suit complex research and analysis where parallel exploration pays off.

Maintaining coherence across extended interactions will stay central even as models improve.

## 6. Conclusion

- Context engineering is a fundamental shift: not just crafting the perfect prompt but curating what enters the model's limited attention budget at each step.
- Whether implementing compaction, designing token-efficient tools or enabling just-in-time exploration, the principle is the same: the smallest set of high-signal tokens that maximizes the likelihood of the desired outcome.
- Smarter models need less prescriptive engineering and can run with more autonomy, but treating context as a precious, finite resource remains central to reliable agents.

## Quick reference

| Lever | What to do | What to avoid |
| --- | --- | --- |
| System prompt | right altitude; sectioned; minimal but complete; start minimal with the best model, add from observed failures | brittle hardcoded logic; vague guidance that assumes shared context |
| Tools | minimal viable set; self-contained, clear, token-efficient | bloated sets with ambiguous choices between tools |
| Examples | a few diverse, canonical examples | a laundry list of edge cases |
| Retrieval | just-in-time loading through lightweight references; hybrid with up-front loading (CLAUDE.md plus glob and grep) | slow unguided exploration; agents chasing dead ends |
| Compaction | summarize near the limit; recall first, then precision; clear old tool results | aggressive compaction that drops subtle context |
| Notes and memory | persist progress outside the window (to-do list, NOTES.md, memory tool) | relying on the window alone for multi-hour tasks |
| Sub-agents | isolate deep exploration; return 1,000 to 2,000-token summaries | one agent carrying every detail of a large project |

## Changes between the September 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-09-30 (original URL under /engineering/) shows the body unchanged except for one path example: "src/core_logic.py." became "src/core_logic/" (the sentence-ending period was dropped with it). Other differences are page chrome: the article moved from /engineering/ to /research/, and the standfirst ("Context is a critical but finite resource for AI agents. In this post, we explore strategies for effectively curating and managing the context that powers them.") no longer appears.
