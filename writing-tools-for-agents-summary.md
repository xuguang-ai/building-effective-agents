# Writing effective tools for AI agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Writing effective tools for AI agents" (originally titled "Writing effective tools for agents — with agents") |
| URL | https://www.anthropic.com/research/writing-tools-for-agents (first published under /engineering/) |
| Author | Ken Aizawa, with contributions from colleagues in Research, MCP, Product Engineering, Marketing, Design and Applied AI |
| Published | 2025-09-11 |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-09-15; differences listed at the end |
| Companion | Tool evaluation cookbook on platform.claude.com |

## One-sentence takeaway

A tool is a contract between deterministic software and a non-deterministic agent, so it has to be designed for the agent: prototype it, build a realistic evaluation, let Claude Code optimize the tool against that evaluation, and follow five principles (few consolidated tools, namespacing, meaningful context, token efficiency, prompt-engineered descriptions).

## 1. What is a tool?

- Deterministic systems give the same output for the same input. Agents are non-deterministic: the same starting conditions can produce different responses.
- Traditional software is a contract between deterministic systems: getWeather("NYC") always fetches New York weather the same way. Tools are a new kind of software, a contract between deterministic systems and non-deterministic agents. Asked "Should I bring an umbrella today?", an agent might call the weather tool, answer from general knowledge, ask which location first, hallucinate, or fail to understand how to use the tool at all.
- So tools and MCP servers should not be written the way we write functions and APIs for other developers or systems. They have to be designed for agents.
- The goal is to **increase the surface area over which agents can be effective**, letting them pursue a variety of successful strategies. In Anthropic's experience the tools most "ergonomic" for agents are also surprisingly intuitive for humans.

## 2. How to write tools: prototype, evaluate, collaborate

### 2.1 Build a prototype

- It is hard to predict which tools agents will find ergonomic without hands-on testing, so stand up a quick prototype first.
- If Claude Code writes the tools (possibly in one shot), give it documentation for the libraries, APIs and SDKs involved, including the MCP SDK. LLM-friendly documentation often lives in flat llms.txt files on official doc sites.
- Wrap the tools in a local MCP server or a Desktop extension (DXT) to test them in Claude Code (`claude mcp add <name> <command> [args...]`) or in the Claude Desktop app (Settings > Developer, or Settings > Extensions). Tools can also be passed directly into API calls for programmatic testing.
- Test the tools yourself to find rough edges, and collect user feedback to build intuition about the use cases and prompts the tools must support.

### 2.2 Run an evaluation

- Generate many evaluation tasks grounded in real-world use, then collaborate with an agent to analyze the results and decide how to improve the tools.
- **Generating tasks**: Claude Code can explore the prototype and produce dozens of prompt and response pairs. Prompts should come from real uses and realistic data sources (internal knowledge bases, microservices). Avoid simplistic "sandbox" environments that do not stress-test the tools. Strong tasks may need many tool calls, potentially dozens.
  - Strong tasks (from the post): "Schedule a meeting with Jane next week to discuss our latest Acme Corp project. Attach the notes from our last project planning meeting and reserve a conference room." "Customer ID 9182 reported that they were charged three times for a single purchase attempt. Find all relevant log entries and determine if any other customers were affected by the same issue." "Customer Sarah Chen just submitted a cancellation request. Prepare a retention offer. Determine: (1) why they're leaving, (2) what retention offer would be most compelling, and (3) any risk factors we should be aware of before making an offer."
  - Weak tasks: "Schedule a meeting with jane@acme.corp next week." "Search the payment logs for purchase_complete and customer_id=9182." "Find the cancellation request by Customer ID 45892."
- Pair every prompt with a **verifiable response or outcome**. The verifier can be as simple as an exact string match or as advanced as Claude judging the response. Avoid overly strict verifiers that reject correct answers over formatting, punctuation or valid alternative phrasing.
- Optionally list the tools you expect the agent to call, to measure whether it grasps each tool's purpose, but do not overspecify: there are often several valid paths.
- **Running it**: run programmatically with direct LLM API calls, using simple agentic loops (a while loop alternating LLM calls and tool calls), one loop per task, each agent given a single task prompt plus the tools.
- Instruct evaluation agents to output reasoning and feedback blocks as well as the structured response block, and to output them before tool calls and responses. This can raise effective intelligence by triggering chain-of-thought behavior. With Claude, interleaved thinking gives similar functionality off the shelf and helps probe why agents do or do not call certain tools.
- Beyond accuracy, collect runtime of individual tool calls and tasks, number of tool calls, total token consumption and tool errors. Tool-call tracking reveals the workflows agents actually follow and where tools could be consolidated.
- Results on Anthropic's internal held-out test sets (from the two charts in the post), human-written vs Claude-optimized MCP server:

| Internal tool set | Human-written | Claude-optimized |
| --- | --- | --- |
| Slack tools | 67.4% | 80.1% |
| Asana tools | 79.6% | 85.7% |

### 2.3 Analyze the results

- Agents are helpful partners in spotting contradictory descriptions, inefficient implementations and confusing schemas. But what agents **omit** from their feedback is often more important than what they include; LLMs do not always say what they mean.
- Observe where agents get stumped. Read their reasoning and feedback (chain of thought). Review the raw transcripts, including tool calls and tool responses, to catch behavior the agent never described. Read between the lines; evaluation agents do not necessarily know the correct answers or strategies.
- Analyze tool-calling metrics. Many redundant calls suggest rightsizing pagination or token limits. Many invalid-parameter errors suggest clearer descriptions or better examples. Example: at the launch of Claude's web search tool, Claude was needlessly appending "2025" to the query parameter, biasing results; a better tool description fixed it.

### 2.4 Collaborate with agents

- Concatenate the evaluation transcripts and paste them into Claude Code. Claude is good at analyzing transcripts and refactoring many tools at once, for example keeping implementations and descriptions self-consistent after changes.
- Most of the advice in the post came from repeatedly optimizing Anthropic's internal tools with Claude Code. The evaluations were built on the internal workspace with real projects, documents and messages.
- Held-out test sets guarded against overfitting to the "training" evaluations, and showed gains beyond "expert" implementations, whether hand-written by researchers or generated by Claude.

## 3. Principles for writing effective tools

### 3.1 Choose the right tools (and not too many)

- More tools do not mean better outcomes. A common error is wrapping existing API endpoints one for one, whether or not they suit agents. Agents have different "affordances" than traditional software.
- Agent context is limited while computer memory is cheap. A tool that returns ALL contacts forces the agent to read each one token by token, like finding a contact by reading the address book from the first page. Better to skip to the relevant page: a search_contacts or message_contact tool instead of list_contacts.
- Build a few thoughtful tools for specific high-impact workflows that match the evaluation tasks, then scale up.
- **Consolidate**: a tool can run several operations under the hood, enrich its response with related metadata, or handle a frequently chained multi-step task in one call. Examples from the post:
  - schedule_event (finds availability and schedules) instead of list_users, list_events and create_event.
  - search_logs (relevant lines plus some context) instead of read_logs.
  - get_customer_context (all recent, relevant customer information at once) instead of get_customer_by_id, list_transactions and list_notes.
- Each tool needs a clear, distinct purpose. Tools should let agents subdivide tasks the way a human with the same resources would, while cutting the context that intermediate outputs would otherwise consume. Too many or overlapping tools distract agents from efficient strategies.

### 3.2 Namespace the tools

- Agents may get dozens of MCP servers and hundreds of tools, including other developers' tools. Overlapping or vaguely purposed tools confuse them.
- Group related tools under common prefixes (some MCP clients do this by default): by service (asana_search, jira_search) and by resource (asana_projects_search, asana_users_search).
- Prefix- vs suffix-based namespacing had non-trivial effects on Anthropic's tool-use evaluations. The effect varies by model, so choose by your own evaluations.
- Agents can call the wrong tool, the right tool with wrong parameters, too few tools, or mishandle responses. Tools whose names reflect natural task subdivisions reduce the number of tools and descriptions loaded into context and move computation out of the agent's context into the tool calls themselves, lowering the risk of mistakes.

### 3.3 Return meaningful context

- Return only high-signal information. Prioritize contextual relevance over flexibility. Drop low-level technical identifiers (uuid, 256px_image_url, mime_type) in favor of fields like name, image_url and file_type that inform the agent's next action.
- Agents handle natural-language names far better than cryptic identifiers. Merely resolving arbitrary alphanumeric UUIDs into meaningful language, or even a 0-indexed ID scheme, significantly improved Claude's precision on retrieval tasks by reducing hallucinations.
- When the agent needs both natural language and technical IDs (search_user(name='jane') then send_message(id=12345)), expose a **response_format** enum parameter so the agent can choose "concise" or "detailed", GraphQL-style. The post's Slack example: the detailed response (206 tokens) includes channel IDs, user IDs, timestamps and the thread_ts needed for follow-up calls; the concise response (72 tokens) returns only channel, author, text and date, about one third of the tokens.
- Even the response structure (XML, JSON or Markdown) affects evaluation performance, and there is no one-size-fits-all answer: LLMs do better with formats that resemble their training data. Pick by your own evaluation.

### 3.4 Optimize tool responses for token efficiency

- Quantity of context matters as much as quality. Implement some combination of pagination, range selection, filtering and truncation with sensible defaults for any tool that could return a lot. Claude Code limits tool responses to 25,000 tokens by default. Effective context lengths will grow, but the need for context-efficient tools will remain.
- When truncating, steer the agent with helpful instructions, for example toward many small targeted searches instead of one broad search. The post's Stripe example: a transactions_search response says 2,847 transactions matched, shows the first three, gives a summary (total revenue, date range), and lists how to refine: search by payee, filter by amount range, fetch the next page.
- Prompt-engineer error responses to state specific, actionable fixes instead of opaque codes or tracebacks. The post's Asana example, a user_info call with userId "john.doe@acme.corp": the unhelpful version is a JSON blob with code RESOURCE_NOT_FOUND, status 422 and message "Invalid value"; the helpful version explains that the userId does not exist or is in the wrong format, shows examples of valid user IDs, and says to call user_search() to resolve one.

### 3.5 Prompt-engineer the tool descriptions

- One of the most effective improvements. Descriptions and specs are loaded into the agent's context and collectively steer tool-calling behavior.
- Describe the tool as you would to a new hire. Make implicit context explicit: specialized query formats, niche terminology, relationships between resources. Remove ambiguity by describing, and enforcing with strict data models, the expected inputs and outputs. Name parameters unambiguously: user_id, not user.
- With an evaluation you can measure the effect of such changes with confidence. Small refinements can yield dramatic gains: Claude Sonnet 3.5 reached state-of-the-art on SWE-bench Verified after precise refinements to tool descriptions, with far fewer errors and better task completion.
- Further reading named in the post: the Developer Guide's best practices for tool definitions, how tools are dynamically loaded into Claude's system prompt, and MCP tool annotations that disclose which tools need open-world access or make destructive changes.

## 4. Looking ahead

- Building for agents means re-orienting software practice from deterministic to non-deterministic patterns.
- Effective tools are intentionally and clearly defined, use agent context judiciously, combine into diverse workflows, and let agents solve real-world tasks intuitively.
- The mechanisms will change (MCP updates, stronger models), but a systematic, evaluation-driven approach keeps tools improving alongside agents.

## Quick reference

| Principle | Do | Avoid | Example in the post |
| --- | --- | --- | --- |
| Choose tools | a few consolidated, workflow-targeted tools | wrapping every API endpoint; list_* tools that dump everything | schedule_event replaces list_users + list_events + create_event |
| Namespace | prefix by service and resource; test prefix vs suffix | overlapping, vaguely named tools | asana_projects_search, asana_users_search |
| Return context | high-signal fields, natural-language names, response_format enum | raw UUIDs, MIME types, low-level IDs | Slack search: 206-token detailed vs 72-token concise |
| Token efficiency | pagination, filtering, truncation with guidance; actionable errors | unbounded dumps; opaque error codes | Stripe search truncated with refine hints; Asana error with valid-ID examples |
| Descriptions | explain as to a new hire; unambiguous parameter names; measure with evals | implicit assumptions; ambiguous names such as user | user_id; SWE-bench Verified gains from description refinements |

## Changes between the September 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-09-15 (original URL under /engineering/) shows the body identical word for word. The only changes:

- The article moved from /engineering/ to /research/.
- The title changed from "Writing effective tools for agents — with agents" (page title "Writing effective tools for AI agents—using AI agents") to "Writing effective tools for AI agents".
- The standfirst ("Agents are only as effective as the tools we give them. We share how to write high-quality tools and evaluations, and how you can boost performance by using Claude to optimize its tools for itself.") no longer appears.
