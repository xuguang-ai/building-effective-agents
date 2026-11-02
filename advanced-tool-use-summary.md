# Introducing advanced tool use for Claude — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Introducing advanced tool use for Claude" |
| URL | https://www.anthropic.com/research/advanced-tool-use (first published under /engineering/ with the title "Introducing advanced tool use on the Claude Developer Platform") |
| Authors | Written by Bin Wu, with contributions from Adam Jones, Artur Renault, Henry Tay, Jake Noble, Noah Picard, Sam Jiang and the Claude Developer Platform team; builds on foundational research by Chris Gorgolewski, Daniel Jiang, Jeremy Fox and Mike Lambert; inspiration credited to Joel Pobar's LLMVM, Cloudflare's Code Mode and Anthropic's "Code execution with MCP" post; thanks to Andy Schumeister, Hamish Kerr, Keir Bradwell, Matt Bleifer and Molly Vorwerck |
| Published | 2025-11-24 (datePublished in the page's JSON-LD; the page shows "Published Nov 24, 2025"; dateModified 2026-10-18) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-11-24; differences listed at the end |
| Companion | Docs and cookbooks: Tool Search Tool documentation plus the cookbook notebook tool_use/tool_search_with_embeddings.ipynb in anthropics/claude-cookbooks; Programmatic Tool Calling documentation plus tool_use/programmatic_tool_calling_ptc.ipynb; Tool Use Examples section of the "implement tool use" documentation (platform.claude.com) |
| Video | An embedded YouTube video (id 2MJDdzSXL74) sits after the introduction; its content is not in the page text and is not summarized here |
| Related posts | "Building effective agents" and "Code execution with MCP" (both cited in the introduction); Claude for Excel named as an internal user of Programmatic Tool Calling |

## One-sentence takeaway

Agents that must work across hundreds or thousands of tools hit three bottlenecks, and the post ships one beta feature for each: the **Tool Search Tool** loads tool definitions on demand instead of upfront (85% fewer tokens, better MCP-eval accuracy), **Programmatic Tool Calling** lets Claude write Python that calls tools inside the Code Execution sandbox so only the final output enters its context (37% fewer tokens on complex research tasks), and **Tool Use Examples** put sample calls into tool definitions so Claude learns conventions a JSON schema cannot express (72% to 90% on complex parameter handling).

## 1. Introduction: what agents with large tool libraries need

- **The target picture**: models working seamlessly across hundreds or thousands of tools. Two examples: an IDE assistant integrating git operations, file manipulation, package managers, testing frameworks and deployment pipelines; an operations coordinator connecting Slack, GitHub, Google Drive, Jira, company databases and dozens of MCP servers simultaneously.
- **Need 1, unlimited tool libraries** without stuffing every definition into context upfront. The earlier post on code execution with MCP found that tool results and definitions can consume 50,000+ tokens before an agent reads a request; agents should discover and load tools on demand and keep only what is relevant to the current task.
- **Need 2, calling tools from code.** With natural-language tool calling each invocation costs a full inference pass, and intermediate results pile up in context whether useful or not. Code is the natural fit for orchestration logic (loops, conditionals, data transformations), and agents need the flexibility to choose between code execution and inference per task.
- **Need 3, learning usage from examples**, not just schema definitions. JSON schemas say what is structurally valid but cannot express when to include optional parameters, which combinations make sense, or what conventions an API expects.
- **Three features released** (all beta): the Tool Search Tool ("allows Claude to use search tools to access thousands of tools without consuming its context window"), Programmatic Tool Calling ("invoke tools in a code execution environment reducing the impact on the model's context window") and Tool Use Examples ("a universal standard for demonstrating how to effectively use a given tool").
- **Internal proof point**: the features let Anthropic build things not possible with conventional tool-use patterns; Claude for Excel uses Programmatic Tool Calling to read and modify spreadsheets with thousands of rows without overloading the context window.

## 2. Tool Search Tool

### The challenge

- **Definitions add up** as MCP servers connect. The post's five-server setup: GitHub 35 tools (~26K tokens), Slack 11 tools (~21K), Sentry 5 tools (~3K), Grafana 5 tools (~3K), Splunk 2 tools (~2K): 58 tools consuming approximately 55K tokens before the conversation starts. Adding Jira (~17K tokens alone) approaches 100K+ tokens of overhead. At Anthropic, tool definitions consumed 134K tokens before optimization.
- **Token cost is not the only issue**: the most common failures are wrong tool selection and incorrect parameters, especially with similar names such as notification-send-user vs. notification-send-channel.

### Our solution

- **Discover tools on demand** instead of loading every definition upfront; Claude only sees the tools it actually needs for the current task.
- **The post's chart** ("Context Usage: Traditional vs. Tool Search Tool") shows two 200k-token bars. Traditional approach: 77.2k/200k tokens used, 61.4% free space; legend System prompt 3,000, Custom tools 2,000, MCP tools 72,000, Messages 200, Free space 122,800. Tool search tool: 8.7k/200k tokens used, 95.65% free; legend System prompt 3,000, System tools 2,000, Tool Search Tool 500, Discovered MCP tools 3,000, Messages 200, Free space 191,300. Caption: "Tool Search Tool preserves 191,300 tokens of context compared to 122,800 with Claude's traditional approach."
- **Traditional approach in words**: all tool definitions loaded upfront (~72K tokens for 50+ MCP tools); conversation history and system prompt compete for the remaining space; ~77K tokens consumed before any work begins.
- **With the Tool Search Tool**: only the search tool loaded upfront (~500 tokens); tools discovered as needed (3-5 relevant tools, ~3K tokens); total ~8.7K tokens, preserving 95% of the context window. The post calls this an 85% reduction in token usage while keeping access to the full tool library.
- **Accuracy on MCP evaluations** with large tool libraries, internal testing: Opus 4 improved from 49% to 74%, Opus 4.5 from 79.5% to 88.1% with the Tool Search Tool enabled.

### How the Tool Search Tool works

- **Mark tools deferred**: you still send every tool definition to the API, but tools with `defer_loading: true` are not loaded into context initially. Claude sees only the Tool Search Tool itself plus tools with `defer_loading: false` (your most critical, frequently used tools).
- **Search, then expand**: when Claude needs a capability it searches; the tool returns references to matching tools, which get expanded into full definitions in context. Example: searching "github" loads only github.createPullRequest and github.listIssues, not the other 50+ tools from Slack, Jira and Google Drive. Claude keeps access to the whole library while paying tokens only for what it uses.
- **Prompt caching note** (a paragraph added after first publication): deferred tools are excluded from the initial prompt entirely and only added after a search, so the system prompt and core tool definitions remain cacheable.
- **Implementation gist**: the `tools` array contains a search tool, for example `{"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"}` (regex, BM25 or custom), followed by ordinary tool definitions such as github.createPullRequest carrying `"defer_loading": true`, "hundreds more" of them.
- **Whole MCP servers can be deferred** while keeping specific high-use tools loaded:

```json
{"type": "mcp_toolset", "mcp_server_name": "google-drive",
 "default_config": {"defer_loading": true},
 "configs": {"search_files": {"defer_loading": false}}}
```

- **Search backends**: the Claude Developer Platform ships regex-based and BM25-based search tools out of the box; custom search tools using embeddings or other strategies are possible.

### When to use the Tool Search Tool

- **Trade-off**: it adds a search step before tool invocation, so it pays off when context savings and accuracy gains outweigh the added latency.
- **Use it when**: tool definitions consume >10K tokens; you see tool-selection accuracy issues; you build MCP-powered systems with multiple servers; 10+ tools are available.
- **Less beneficial when**: the library is small (<10 tools); all tools are used frequently in every session; definitions are compact.

## 3. Programmatic Tool Calling

### The challenge

- **Context pollution from intermediate results**: analyzing a 10MB log file for error patterns puts the whole file in context although Claude only needs a summary of error frequencies; fetching customer data across multiple tables accumulates every record regardless of relevance; this can push important information out of the window entirely.
- **Inference overhead and manual synthesis**: each tool call is a full model inference pass, after which Claude must "eyeball" the data, reason about how pieces fit and decide what to do next in natural language. A five-tool workflow means five inference passes plus parsing, comparing and synthesizing: slow and error-prone.

### Our solution

- **Orchestrate through code**: instead of requesting tools one at a time with each result returned to its context, Claude writes code that calls multiple tools, processes their outputs and controls what enters its context window.
- **Why code**: Claude excels at writing code, and orchestration logic in Python gives more reliable, precise control flow; loops, conditionals, data transformations and error handling become explicit rather than implicit in Claude's reasoning.

### Example: budget compliance check

- **Task**: "Which team members exceeded their Q3 travel budget?" with three tools: get_team_members(department) returns team members with IDs and levels; get_expenses(user_id, quarter) returns expense line items for a user; get_budget_by_level(level) returns budget limits for an employee level.
- **Traditional approach**: fetch team members (20 people); fetch each person's Q3 expenses (20 tool calls, each returning 50-100 line items: flights, hotels, meals, receipts); fetch budget limits by level; all of it enters context, 2,000+ expense line items (50 KB+); Claude manually sums each person's expenses, looks up the budget and compares; more round-trips and significant context consumption.
- **With Programmatic Tool Calling**: Claude writes one Python script that orchestrates the whole workflow. It runs in the Code Execution tool (a sandboxed environment) and pauses when it needs a result from your tools; results you return via the API are processed by the script rather than consumed by the model; the script continues and Claude sees only the final output.
- **The post's sequence diagram** ("Programmatic Tool Calling Flow") has lanes User, API and Claude plus a Code Execution Tool box: User sends Request to API; API samples Claude; Claude generates a Python script with custom tool calls; API runs the script in a container; the script pauses at a custom tool call; API returns a Response with the custom tool call to the User; User sends a Request with the custom tool result; API resumes the script with it; the script result comes back; API samples Claude with the script result; Claude interprets it; API sends the Response to the User. Caption: "Programmatic Tool Calling enables Claude to orchestrate tools through code rather than through individual API round-trips, allowing for parallel tool execution."
- **The orchestration code in words**: await get_team_members("engineering"); collect the unique levels and fetch their budgets in parallel with asyncio.gather; build a dict of level to budget; fetch all members' Q3 expenses in parallel with asyncio.gather; loop over members, sum e["amount"], compare with budget["travel_limit"] and collect {"name", "spent", "limit"} for those over; print(json.dumps(exceeded)).
- **What Claude sees**: only the two to three people who exceeded their budget. The 2,000+ line items, intermediate sums and budget lookups never touch its context, cutting consumption from 200KB of raw expense data to 1KB of results.
- **Reported gains**:
  - Token savings: average usage dropped from 43,588 to 27,297 tokens, a 37% reduction on complex research tasks.
  - Reduced latency: each API round-trip costs an inference pass (hundreds of milliseconds to seconds); orchestrating 20+ tool calls in a single code block eliminates 19+ inference passes, because the API handles tool execution without returning to the model each time.
  - Improved accuracy: internal knowledge retrieval improved from 25.6% to 28.5%; "GIA benchmarks" (the post's link points to arXiv 2311.12983) from 46.5% to 51.2%.
- **Production framing**: messy data, conditional logic and scale are handled programmatically while Claude stays focused on actionable results rather than raw data processing.

### How Programmatic Tool Calling works

1. **Mark tools as callable from code**: add the `{"type": "code_execution_20250825", "name": "code_execution"}` tool and set `"allowed_callers": ["code_execution_20250825"]` on each tool that opts in (get_team_members, get_expenses, get_budget_by_level in the example). The API converts these tool definitions into Python functions Claude can call.
2. **Claude writes orchestration code**: the response contains a `server_tool_use` block (id srvtoolu_abc, name code_execution) whose input.code is the script above.
3. **Tools execute without hitting Claude's context**: when the code calls get_expenses(), you receive a `tool_use` request (id toolu_xyz, input {"user_id": "emp_123", "quarter": "Q3"}) with a `caller` field {"type": "code_execution_20250825", "tool_id": "srvtoolu_abc"}. You return the result, which is processed in the Code Execution environment; the cycle repeats for each call in the code.
4. **Only the final output enters context**: a `code_execution_tool_result` block (tool_use_id srvtoolu_abc) whose content.stdout holds the printed JSON, for example [{"name": "Alice", "spent": 12500, "limit": 10000}...]. That is all Claude sees, not the 2000+ expense line items processed along the way.

### When to use Programmatic Tool Calling

- **Overhead**: it adds a code execution step, worthwhile when token, latency and accuracy gains are substantial.
- **Most beneficial when**: processing large datasets where only aggregates or summaries are needed; multi-step workflows with three or more dependent tool calls; filtering, sorting or transforming results before Claude sees them; tasks where intermediate data should not influence Claude's reasoning; parallel operations across many items (checking 50 endpoints, for example).
- **Less beneficial when**: simple single-tool invocations; tasks where Claude should see and reason about all intermediate results; quick lookups with small responses.

## 4. Tool Use Examples

### The challenge

- **Schemas define structure, not usage**: JSON Schema handles types, required fields and allowed enums but cannot express when to include optional parameters, which combinations make sense or what conventions the API expects.
- **The create_ticket example schema**: title (string, the only required field); priority enum low/medium/high/critical; labels (array of strings); reporter object with id, name and a nested contact object (email, phone); due_date (string); escalation object with level (integer), notify_manager (boolean) and sla_hours (integer).
- **Questions the schema leaves open**: format ambiguity (should due_date be "2024-11-06", "Nov 6, 2024" or "2024-11-06T00:00:00Z"?); ID conventions (is reporter.id a UUID, "USR-12345" or just "12345"?); nested structure usage (when should Claude populate reporter.contact?); parameter correlations (how do escalation.level and escalation.sla_hours relate to priority?). These lead to malformed tool calls and inconsistent parameter usage.

### Our solution

- **Sample calls in the definition**: an `input_examples` array next to `input_schema`. The post gives three examples for create_ticket:
  - Full: "Login page returns 500 error", priority critical, labels ["bug", "authentication", "production"], reporter USR-12345 Jane Smith with contact jane@acme.com and +1-555-0123, due_date 2024-11-06, escalation level 2, notify_manager true, sla_hours 4.
  - Partial: "Add dark mode support", labels ["feature-request", "ui"], reporter USR-67890 Alex Chen with no contact and no escalation.
  - Minimal: "Update API documentation" only.
- **What Claude learns**: format conventions (dates YYYY-MM-DD, user IDs USR-XXXXX, labels in kebab-case); nested structure patterns (how to build reporter with its nested contact); optional-parameter correlations (critical bugs carry full contact info plus escalation with tight SLAs; feature requests carry a reporter but no contact or escalation; internal tasks carry a title only).
- **Measured effect**: in internal testing, tool use examples improved accuracy from 72% to 90% on complex parameter handling.

### When to use Tool Use Examples

- **Cost**: examples add tokens to tool definitions, so they are worth it when accuracy gains outweigh the cost.
- **Most beneficial when**: complex nested structures where valid JSON does not imply correct usage; tools with many optional parameters where inclusion patterns matter; APIs with domain-specific conventions not captured in schemas; similar tools where examples clarify which to use (create_ticket vs create_incident).
- **Less beneficial when**: simple single-parameter tools with obvious usage; standard formats like URLs or emails that Claude already understands; validation concerns better handled by JSON Schema constraints.

## 5. Best practices

### Layer features strategically

- **Start with the biggest bottleneck** rather than enabling everything: context bloat from tool definitions points to the Tool Search Tool; large intermediate results polluting context point to Programmatic Tool Calling; parameter errors and malformed calls point to Tool Use Examples.
- **Then layer** as needed; the features are complementary: the Tool Search Tool ensures the right tools are found, Programmatic Tool Calling ensures efficient execution, Tool Use Examples ensure correct invocation.

### Set up Tool Search Tool for better discovery

- **Search matches names and descriptions**, so clear, descriptive definitions improve discovery. Good: name search_customer_orders, description "Search for customer orders by date range, status, or total amount. Returns order details including items, shipping, and payment info." Bad: name query_db_orders, description "Execute order query".
- **Tell Claude what exists** in the system prompt, for example: "You have access to tools for Slack messaging, Google Drive file management, Jira ticket tracking, and GitHub repository operations. Use the tool search to find specific capabilities."
- **Keep three to five most-used tools always loaded** and defer the rest, balancing immediate access for common operations with on-demand discovery for everything else.

### Set up Programmatic Tool Calling for correct execution

- **Document return formats** clearly, because Claude writes code that parses tool outputs. The post's get_orders description spells out the returned list of order objects: id (str) order identifier; total (float) in USD; status (str) one of 'pending', 'shipped', 'delivered'; items (list) of {sku, quantity, price}; created_at (str) ISO 8601 timestamp.
- **Opt in the right tools**: tools that can run in parallel (independent operations) and operations safe to retry (idempotent).

### Set up Tool Use Examples for parameter accuracy

- **Use realistic data** (real city names, plausible prices, not "string" or "value").
- **Show variety** with minimal, partial and full specification patterns.
- **Keep it concise**: 1-5 examples per tool.
- **Focus on ambiguity**: add examples only where correct usage is not obvious from the schema.

## 6. Getting started

- **Beta header**: `betas=["advanced-tool-use-2025-11-20"]` on `client.beta.messages.create`. The sample uses model claude-sonnet-4-5-20250929, max_tokens 4096, and a tools list containing `{"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"}`, `{"type": "code_execution_20250825", "name": "code_execution"}` and your own tools carrying defer_loading, allowed_callers and input_examples.
- **Further reading**: documentation and cookbook for the Tool Search Tool (tool_search_with_embeddings.ipynb), documentation and cookbook for Programmatic Tool Calling (programmatic_tool_calling_ptc.ipynb), and the "providing tool use examples" section of the implement-tool-use documentation.
- **Closing claim**: these features move tool use "from simple function calling toward intelligent orchestration"; as agents span dozens of tools and large datasets, dynamic discovery, efficient execution and reliable invocation become foundational.

## Quick reference

| Feature | Bottleneck it targets | How to enable (all under the beta header advanced-tool-use-2025-11-20) | Numbers reported in the post | Use it when / skip it when |
| --- | --- | --- | --- | --- |
| Tool Search Tool | Tool definitions bloating context; wrong tool selection among similar names | Add a search tool (`tool_search_tool_regex_20251119`, a BM25 variant, or custom); set `defer_loading: true` on tools, or `mcp_toolset` with `default_config` to defer a whole server | ~77K to ~8.7K tokens before work begins (85% reduction, 95% of window free); MCP evals Opus 4 49% to 74%, Opus 4.5 79.5% to 88.1% | Use: definitions >10K tokens, 10+ tools, multiple MCP servers, selection errors. Skip: <10 tools, every tool used every session, compact definitions |
| Programmatic Tool Calling | Intermediate results polluting context; one inference pass per tool call | Add `code_execution_20250825`; set `allowed_callers: ["code_execution_20250825"]` on opt-in tools; answer `tool_use` requests that carry a `caller` field | 43,588 to 27,297 tokens (37% less) on complex research tasks; 20+ calls in one block removes 19+ inference passes; knowledge retrieval 25.6% to 28.5%, GIA 46.5% to 51.2%; budget example 200KB to 1KB | Use: aggregates over large data, 3+ dependent calls, filter or transform before Claude sees results, parallel work over many items. Skip: single calls, cases where Claude must see every intermediate result, quick small lookups |
| Tool Use Examples | Malformed calls, ambiguous formats, unclear optional-parameter patterns | `input_examples` array in the tool definition, 1-5 realistic examples covering minimal, partial and full calls | 72% to 90% accuracy on complex parameter handling | Use: nested structures, many optional parameters, domain conventions, similar tools. Skip: simple single-parameter tools, standard formats, constraints JSON Schema already enforces |
| Combining them | Whatever limits the agent most | Start with the single biggest bottleneck, then layer the others | Not quantified | Search finds the tools, code executes them efficiently, examples make calls correct |

## Changes between the November 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-11-24 (two captures that day, 19:06 and 20:06 UTC, with identical text, both under /engineering/advanced-tool-use) shows a body that is word for word identical except for two edits:

- **Added paragraph**: the "Prompt caching note" in "How the Tool Search Tool works" (deferred tools are excluded from the initial prompt, so the system prompt and core tool definitions stay cacheable) does not exist in the snapshot.
- **Acknowledgements edited**: the snapshot lists "Nathan McCandlish" among the contributors between Jake Noble and Noah Picard; the current page omits him. Everything else in the acknowledgements is unchanged.

The remaining differences are title and page chrome:

- **Title changed** from "Introducing advanced tool use on the Claude Developer Platform" to "Introducing advanced tool use for Claude" (the page's JSON-LD headline still carries the old wording; the HTML title and og:title carry the new one).
- **Path moved** from /engineering/ to /research/; the "Engineering at Anthropic" label is gone and a chapter navigation header ("Chapter 01: Tool Search Tool") was added.
- **Standfirst dropped**: "We've added three new beta features that let Claude discover, learn, and execute tools dynamically. Here's how they work." no longer appears. The page's meta description now reads "Claude can now discover, learn, and execute tools dynamically to enable agents that take action in the real world. Here's how.", but that sentence is not shown in the body text.
- **Hero image replaced by a video**: the snapshot opened with a static illustration; the current page embeds a YouTube video after the introduction. Both diagrams, their captions, all code samples and all numbers are unchanged.
- **Newsletter block removed**: the "Get the developer newsletter" sign-up text at the end of the snapshot is gone. The page's dateModified is 2026-10-18.
