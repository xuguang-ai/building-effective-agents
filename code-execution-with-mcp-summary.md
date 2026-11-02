# Code execution with MCP: Building more efficient agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Code execution with MCP: Building more efficient agents" (the page's HTML `<title>` and og:title now read "Code execution with MCP for efficient agents"; the page's `<h1>` element and the JSON-LD headline keep the original wording) |
| URL | https://www.anthropic.com/research/code-execution-with-mcp (first published under /engineering/code-execution-with-mcp) |
| Authors | Adam Jones and Conor Kelly, with thanks to Jeremy Fox, Jerome Swannack, Stuart Ritchie, Molly Vorwerck, Matt Samuels and Maggie Vo for feedback on drafts |
| Published | 2025-11-04 (datePublished in the page's JSON-LD; dateModified 2026-10-18) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-11-04 (original URL under /engineering/); differences listed at the end |
| Companion | No cookbook or sample repository. The post links the MCP site (modelcontextprotocol.io), the community servers repository (github.com/modelcontextprotocol/servers), the MCP SDK docs, Cloudflare's "Code Mode" blog post, the Claude Agent Skills docs, Anthropic's Claude Code sandboxing post and the MCP community page |

## One-sentence takeaway

When an agent is connected to dozens of MCP servers, loading every tool definition upfront and passing every intermediate result through the model wastes tokens and time; presenting the MCP servers as a **code API on a filesystem** and letting the agent write code against them cuts the post's example from **150,000 to 2,000 tokens (a 98.7% saving)**, keeps large data and PII out of the context window, and gives the agent loops, state and reusable skills, at the price of running a secure sandbox.

## 1. Introduction: MCP and the scaling problem

- **MCP is an open standard** for connecting AI agents to external systems. Traditionally every agent-to-tool pairing needed a custom integration, which fragments effort and makes connected systems hard to scale; with MCP a developer implements the protocol once and unlocks a whole ecosystem of integrations.
- **Adoption since November 2024**: the community has built thousands of MCP servers, SDKs exist for all major programming languages, and the post calls MCP "the de-facto standard for connecting agents to tools and data".
- **The new problem**: developers now routinely build agents with access to hundreds or thousands of tools across dozens of MCP servers. As the number of tools grows, loading all tool definitions upfront and passing intermediate results through the context window slows agents down and raises costs.
- **The post's thesis**: code execution lets agents interact with MCP servers more efficiently, "handling more tools while using fewer tokens".

## 2. Excessive token consumption from tools makes agents less efficient

Two common patterns raise cost and latency as MCP usage scales: tool definitions overload the context window, and intermediate tool results consume additional tokens.

### 2.1 Tool definitions overload the context window

- **Most MCP clients load all tool definitions upfront** directly into context and expose them through a direct tool-calling syntax.
- **Two example definitions** are shown in a schema-like layout: `gdrive.getDocument` (description "Retrieves a document from Google Drive"; parameters `documentId`, required string, and `fields`, optional string; returns a Document object with title, body content, metadata, permissions, etc.) and `salesforce.updateRecord` (description "Updates a record in Salesforce"; parameters `objectType`, required string such as Lead, Contact or Account, `recordId`, required string, and `data`, required object of fields to update; returns the updated record with confirmation).
- **The cost**: tool descriptions occupy context, which increases response time and cost. With thousands of tools connected, an agent may "need to process hundreds of thousands of tokens before reading a request".

### 2.2 Intermediate tool results consume additional tokens

- **Direct calls route every result through the model.** The example request is: "Download my meeting transcript from Google Drive and attach it to the Salesforce lead."
- **The resulting call sequence**: `gdrive.getDocument(documentId: "abc123")` returns "Discussed Q4 goals...\n[full transcript text]" into the model's context; then `salesforce.updateRecord(objectType: "SalesMeeting", recordId: "00Q5f000001abcXYZ", data: { "Notes": "Discussed Q4 goals...\n[full transcript text written out]" })`, for which the model has to write the whole transcript into context a second time.
- **The numbers**: the full transcript flows through the model twice. For a 2-hour sales meeting "that could mean processing an additional 50,000 tokens". Even larger documents may exceed the context window entirely and break the workflow.
- **Error risk**: with large documents or complex data structures, models may be more likely to make mistakes when copying data between tool calls.
- **The diagram** (caption: "The MCP client loads tool definitions into the model's context window and orchestrates a message loop where each tool call and result passes through the model between operations") shows three columns: Model (a chip icon), MCP client and MCP server. Inside the client sits the context window as a stack of blocks: System prompt, Tool 1 def, Tool 2 def, User msg 1, Assistant msg 1, Call tool 1, Call tool 1 result, Assistant msg 2, User msg 2, then empty space. A `tools/list` request to the server and its `tools/list result` fill the two tool-definition blocks; `tools/call (tool 1)` goes to the server and `tools/call result` comes back into the context. Arrows between the client and the model show the growing context being sent to the model before each assistant message and the model's output being appended.

## 3. Code execution with MCP improves context efficiency

- **The idea**: as code execution environments become common for agents, present MCP servers as code APIs rather than direct tool calls, and let the agent write code to interact with them. This addresses both problems: the agent loads only the tools it needs and processes data in the execution environment before anything returns to the model.
- **One implementation (TypeScript)**: generate a file tree of all available tools from the connected servers:

```text
servers
├── google-drive
│   ├── getDocument.ts
│   ├── ... (other tools)
│   └── index.ts
├── salesforce
│   ├── updateRecord.ts
│   ├── ... (other tools)
│   └── index.ts
└── ... (other servers)
```

- **Each tool is a file.** `./servers/google-drive/getDocument.ts` imports `callMCPTool` from `../../../client.js`, declares a `GetDocumentInput` interface (`documentId: string`) and a `GetDocumentResponse` interface (`content: string`), and exports an async `getDocument(input)` function that returns `callMCPTool<GetDocumentResponse>('google_drive__get_document', input)`, documented with the comment "Read a document from Google Drive".
- **The Drive-to-Salesforce task becomes a short script**: import `* as gdrive` from `./servers/google-drive` and `* as salesforce` from `./servers/salesforce`; `const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;` then `await salesforce.updateRecord({ objectType: 'SalesMeeting', recordId: '00Q5f000001abcXYZ', data: { Notes: transcript } })`. The transcript never enters the model's context.
- **Tool discovery by exploring the filesystem**: the agent lists `./servers/` to find servers such as google-drive and salesforce, then reads only the tool files it needs (such as getDocument.ts and updateRecord.ts) to learn each interface, so it loads only the definitions relevant to the current task.
- **The headline number**: "This reduces the token usage from 150,000 tokens to 2,000 tokens—a time and cost saving of 98.7%." The post does not break down how the 150,000-token baseline was measured.
- **Prior art**: Cloudflare published similar findings and calls the approach "Code Mode". The shared insight is that LLMs are adept at writing code, and developers should exploit this strength to build agents that interact with MCP servers more efficiently.

## 4. Benefits of code execution with MCP

The post lists the benefits as loading tools on demand, filtering data before it reaches the model, executing complex logic in a single step, plus security and state-management advantages.

### Progressive disclosure

- **Models are good at navigating filesystems**, so tools presented as code on a filesystem can be read on demand instead of all upfront.
- **Alternative: a `search_tools` tool** on the server. For the hypothetical Salesforce server, the agent searches for "salesforce" and loads only the tools it needs. A **detail level parameter** (name only; name and description; or the full definition with schemas) helps the agent conserve context and find tools efficiently.

### Context-efficient tool results

- **Filter and transform in code** before anything returns to the model. The example fetches a 10,000-row spreadsheet: without code execution, `gdrive.getSheet(sheetId: 'abc123')` returns all 10,000 rows into context to be filtered by the model; with code execution the script calls `gdrive.getSheet`, filters `row["Status"] === 'pending'`, logs `Found ${pendingOrders.length} pending orders` and prints only `pendingOrders.slice(0, 5)` for review.
- **Result**: "The agent sees five rows instead of 10,000." The same pattern covers aggregations, joins across multiple data sources and extraction of specific fields without bloating the context window.

### More powerful and context-efficient control flow

- **Loops, conditionals and error handling** use familiar code patterns rather than chains of individual tool calls. The example waits for a deployment notification in Slack: a `while (!found)` loop calls `slack.getChannelHistory({ channel: 'C123456' })`, checks whether any message text includes 'deployment complete', and otherwise sleeps 5000 ms via `setTimeout` before polling again, then logs "Deployment notification received".
- **Why it is cheaper**: this beats alternating MCP tool calls and sleep commands through the agent loop. Writing out a conditional tree that the execution environment evaluates also saves "time to first token" latency, because the model does not have to be invoked to evaluate each if-statement.

### Privacy-preserving operations

- **Intermediate results stay in the sandbox by default.** The agent sees only what is explicitly logged or returned, so data you do not wish to share with the model can flow through the workflow without entering its context.
- **Automatic tokenization for sensitive workloads**: the agent harness can tokenize sensitive data. In the example the agent imports customer contact details from a spreadsheet into Salesforce: `gdrive.getSheet({ sheetId: 'abc123' })`, then for each row `salesforce.updateRecord({ objectType: 'Lead', recordId: row.salesforceId, data: { Email: row.email, Phone: row.phone, Name: row.name } })`, finally logging `Updated ${sheet.rows.length} leads`.
- **What the model would see** if it logged `sheet.rows`: objects such as `{ salesforceId: '00Q...', email: '[EMAIL_1]', phone: '[PHONE_1]', name: '[NAME_1]' }`, `[EMAIL_2]`, `[PHONE_2]`, `[NAME_2]` and so on. The MCP client intercepts the data and tokenizes PII before it reaches the model; when the data is passed to another MCP tool call it is untokenized through a lookup in the MCP client.
- **Effect**: real email addresses, phone numbers and names flow from Google Sheets to Salesforce "but never through the model", preventing accidental logging or processing of sensitive data. The same mechanism can enforce deterministic security rules about where data may flow to and from.

### State persistence and skills

- **Files as working memory**: with filesystem access an agent can write intermediate results to disk and resume later. Example: `salesforce.query({ query: 'SELECT Id, Email FROM Lead LIMIT 1000' })`, map the leads to `Id,Email` CSV lines, `fs.writeFile('./workspace/leads.csv', csvData)`; a later execution reads `./workspace/leads.csv` back with `fs.readFile` and picks up where it left off.
- **Persisting code as reusable functions**: once working code exists, the agent saves it. Example: `./skills/save-sheet-as-csv.ts` exports `saveSheetAsCsv(sheetId)`, which fetches the sheet through `gdrive.getSheet`, joins rows into CSV, writes `./workspace/sheet-${sheetId}.csv` and returns the path; any later execution can `import { saveSheetAsCsv } from './skills/save-sheet-as-csv'` and call `saveSheetAsCsv('abc123')`.
- **Link to Skills**: this ties into Anthropic's concept of Skills, "folders of reusable instructions, scripts, and resources for models to improve performance on specialized tasks". Adding a `SKILL.md` file to the saved functions turns them into a structured skill the model can reference. Over time the agent builds "a toolbox of higher-level capabilities, evolving the scaffolding that it needs to work most effectively".

### The trade-off: code execution adds its own complexity

- **Infrastructure cost**: running agent-generated code needs a secure execution environment with appropriate sandboxing (the post links Anthropic's Claude Code sandboxing article), resource limits and monitoring. These add operational overhead and security considerations that direct tool calls avoid.
- **Weigh it**: the benefits, which the post names as reduced token costs, lower latency and improved tool composition, should be weighed against these implementation costs.

## 5. Summary (the post's own)

- **MCP is the foundation** for connecting agents to many tools and systems, but once too many servers are connected, tool definitions and results consume excessive tokens and reduce efficiency.
- **Old solutions for new-feeling problems**: context management, tool composition and state persistence have known answers in software engineering; code execution applies those established patterns so agents use familiar programming constructs to work with MCP servers.
- **Call to action**: developers who implement the approach are encouraged to share findings with the MCP community.

## Quick reference

| Technique | What it fixes | Example and numbers in the post |
| --- | --- | --- |
| Tools as files in `./servers/<server>/<tool>.ts`, each wrapping `callMCPTool` | Loading every tool definition upfront | Agent lists `./servers/`, reads only the files it needs; 150,000 tokens down to 2,000, a 98.7% saving |
| `search_tools` with a detail-level parameter | Finding the right tool without loading all schemas | Search "salesforce"; choose name only, name and description, or full schema |
| Filter, aggregate or join inside the sandbox | Large tool results flowing through the model | 10,000-row sheet filtered to pending orders; agent sees 5 rows instead of 10,000 |
| Loops and conditionals in code | Many round trips through the agent loop; time-to-first-token latency | `while (!found)` polling Slack for "deployment complete" with a 5000 ms sleep |
| Intermediate results stay in the sandbox; MCP client tokenizes PII | Sensitive data entering the model's context | `[EMAIL_1]`, `[PHONE_1]`, `[NAME_1]` placeholders, untokenized on the way to Salesforce |
| Write state to files in `./workspace/` | Losing progress between executions | `leads.csv` from a 1,000-lead query, read back later |
| Save working code under `./skills/` with a `SKILL.md` | Re-deriving the same code every run | `saveSheetAsCsv(sheetId)` reused from any later execution |
| Cost to accept | Sandboxing, resource limits, monitoring | Operational and security overhead that direct tool calls avoid |

## Changes between the November 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot taken on 2025-11-04 (original URL /engineering/code-execution-with-mcp) shows the body text, code samples, headings, diagram and acknowledgements identical word for word. The differences are page chrome and metadata: the article moved from /engineering/ to /research/; the HTML `<title>` changed from "Code execution with MCP: building more efficient AI agents" to "Code execution with MCP for efficient agents" (the page's `<h1>` element and the JSON-LD headline still read "Code execution with MCP: Building more efficient agents"; that `<h1>` sits in a header the HTML marks visibility:hidden, and the page was not rendered to confirm how it displays); the standfirst ("Direct tool calls consume context for each definition and result. Agents scale better by writing code to call tools instead. Here's how it works with MCP.") no longer appears; the "Engineering at Anthropic" label and the newsletter sign-up text are gone; and the current page adds a "Chapter 01" table-of-contents entry and a related-article link to "Introducing advanced tool use on the Claude Developer Platform".
