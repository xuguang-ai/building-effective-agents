# Building agents with the Claude Agent SDK — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Building agents with the Claude Agent SDK". Standfirst: "The Claude Agent SDK is a collection of tools that helps developers build powerful agents on top of Claude Code. In this article, we walk through how to get started and share our best practices." |
| URL | https://claude.com/resources/articles/building-agents-with-the-claude-agent-sdk (first published under https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk; by January 2026 it was served from https://claude.com/blog/building-agents-with-the-claude-agent-sdk) |
| Authors | Thariq Shihipar, with notes and editing from Molly Vorwerck, Suzanne Wang, Alex Isken, Cat Wu, Keir Bradwell, Alexander Bricken and Ashwin Bhat |
| Published | 2025-09-29 ("datePublished" in the page's JSON-LD; the page shows "September 29, 2025"; the earliest Wayback snapshot is from the same day) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-09-29 of the original anthropic.com/engineering/ page; differences listed at the end |
| Companion | Claude Agent SDK documentation at docs.claude.com (overview and getting started, subagents, custom tools, the /compact slash command, the migration guide to the latest SDK version); earlier Anthropic posts the article leans on: "Building effective agents", "Writing effective tools for agents – with agents", contextual retrieval, context management, "How Anthropic teams use Claude Code"; the Model Context Protocol site and the modelcontextprotocol/servers repository on GitHub |
| Code | No repository or cookbook of its own; the code is three screenshots (a folder tree, a one-line shell pipeline, a JavaScript email-rule handler), transcribed in sections 5 and 6 |

## One-sentence takeaway

The harness behind Claude Code, which gives Claude a computer (terminal, file system, bash), turned out to power agents far beyond coding, so Anthropic renamed the Claude Code SDK the **Claude Agent SDK**; build an agent around the loop **gather context, take action, verify work, repeat**, using agentic file search, subagents and compaction for context, tools, bash, generated code and MCP for action, and rules, visual feedback and an LLM judge for verification.

## 1. Introduction: from Claude Code to the Claude Agent SDK

- **Where the post starts**: last year Anthropic shared "Building effective agents" with its customers; since then it released Claude Code, an agentic coding solution originally built to support developer productivity at Anthropic.
- **Claude Code outgrew coding**: over the past several months Anthropic has used it for deep research, video creation and note-taking "among countless other non-coding applications", and it "has begun to power almost all of our major agent loops".
- **The rename**: the agent harness that powers Claude Code (the Claude Code SDK) can power many other types of agents, so to reflect this broader vision the Claude Code SDK becomes the Claude Agent SDK.
- **What the post covers**: why the SDK was built, how to build your own agents with it, and the best practices that emerged from the team's own deployments.

## 2. Giving Claude a computer

- **The design principle behind Claude Code**: Claude needs the same tools programmers use every day: find the right files in a codebase, write and edit files, lint the code, run it, debug, edit, and sometimes repeat these actions iteratively until the code succeeds.
- **The terminal was enough**: giving Claude access to the user's computer via the terminal gave it what it needed to write code the way programmers do.
- **The same tools generalize**: with tools to run bash commands and to edit, create and search files, Claude can read CSV files, search the web, build visualizations, interpret metrics and do all sorts of other digital work; in short, you get "general-purpose agents with a computer".
- **The SDK's principle**: give your agents a computer, allowing them to work like humans do.

## 3. Creating new types of agents

The post argues that a computer lets developers build agents that are more effective than before, and lists four kinds:

- **Finance agents** that understand your portfolio and goals and help evaluate investments by accessing external APIs, storing data and running code to make calculations.
- **Personal assistant agents** that book travel, manage your calendar, schedule appointments and put together briefs by connecting to internal data sources and tracking context across applications.
- **Customer support agents** that handle high-ambiguity requests such as customer service tickets by collecting and reviewing user data, connecting to external APIs, messaging users back and escalating to humans when needed.
- **Deep research agents** that research large document collections by searching file systems, analyzing and synthesizing information from multiple sources, cross-referencing data across files and generating detailed reports.
- **"And much more"**: at its core the SDK provides the primitives to build agents for whatever workflow you want to automate.

## 4. Building your agent loop

- **The loop**: in Claude Code, Claude operates in a specific feedback loop: gather context, take action, verify work, repeat.
- **Figure (diagram titled "Claude Agent SDK Loop")**: three boxes left to right, "Gather context", "Take action" and "Verify work", joined by arrows, with a return arrow from "Verify work" back to "Gather context". Caption: "Agents often operate in a specific feedback loop: gather context -> take action -> verify work -> repeat."
- **How the post uses it**: the loop is a way to think about other agents and the capabilities they need; the rest of the article walks through how one might build an **email agent** with the Claude Agent SDK, one loop stage at a time.

## 5. Gather context

An agent needs more than a prompt: it must be able to fetch and update its own context. Four SDK features help.

### Agentic search and the file system

- **The file system is latent context**: it represents information that could be pulled into the model's context. Faced with large files such as logs or user uploads, Claude decides how to load them using bash scripts like grep and tail. "In essence, the folder and file structure of an agent becomes a form of context engineering."
- **Email agent**: store previous conversations in a "Conversations" folder so the agent can search them when asked.
- **Figure (folder tree)**: `/email-agent/` containing `conversations/` with three Markdown files: `2025-09-project-alpha-discussion.md`, `2025-09-customer-support-thread.md` and `2025-08-vendor-negotiations.md`.

### Semantic search

- **Trade-off**: semantic search is usually faster than agentic search but less accurate, more difficult to maintain and less transparent. It means chunking the relevant context, embedding the chunks as vectors, and searching for concepts by querying those vectors.
- **Recommendation**: start with agentic search; add semantic search only if you need faster results or more variations.

### Subagents

- **Supported by default** in the Claude Agent SDK, for two reasons: parallelization (several subagents work on different tasks simultaneously) and context management (each subagent has its own isolated context window and sends back only the relevant information, not its full context). This makes them ideal for sifting through large amounts of information where most of it will not be useful.
- **Email agent**: a "search subagent" capability; the agent spins off multiple search subagents in parallel, each running a different query against the email history, and they return only relevant excerpts rather than full email threads.

### Compaction

- **Long runs need context maintenance**: the SDK's compact feature automatically summarizes previous messages when the context limit approaches, so the agent does not run out of context. It is built on Claude Code's /compact slash command.

## 6. Take action

### Tools

- **Primary building blocks of execution**: tools are prominent in Claude's context window, which makes them the primary actions Claude considers when deciding how to complete a task, so design them to maximize context efficiency (the post points to "Writing effective tools for agents – with agents" and to the custom-tools documentation).
- **Rule of thumb**: your tools should be the primary actions you want the agent to take.
- **Email agent**: tools such as `fetchInbox` or `searchEmails` as the agent's primary, most frequent actions.

### Bash and scripts

- **General-purpose tool**: bash lets the agent do flexible work using a computer.
- **Email agent**: important information may live in attachments; Claude can write code to download a PDF, convert it to text and search it. The post's screenshot (labelled Shell):

```sh
pdftotext document.pdf - | grep -n "invoice" | tail -10
```

### Code generation

- **Why code**: the SDK excels at code generation, and code is precise, composable and "infinitely reusable", an ideal output for agents that must perform complex operations reliably. When building an agent, ask which tasks would benefit from being expressed as code; the answer often unlocks significant capabilities.
- **Production example**: file creation in Claude.ai relies entirely on code generation; Claude writes Python scripts that create Excel spreadsheets, PowerPoint presentations and Word documents, giving consistent formatting and complex functionality that would be hard to achieve any other way.
- **Email agent**: let users create rules for inbound emails as code that runs on that event. The screenshot (labelled JavaScript) defines `async function onEmailReceived(email)`: the sender counts as a customer if the address includes `@customer.com` or matches an entry in `contacts.clients`; otherwise the handler returns; it then asks an LLM "Does this email seem urgent: ${renderEmail(email)}, return TRUE OR FALSE"; if the answer is "TRUE" and the sender is a customer, it calls `forwardEmail` to `teamlead@company.com` with the note "Auto-forwarded: Urgent customer email requiring immediate attention" and `preserveAttachments: true`, and `createTask` with the title "Urgent: Respond to ${email.from}", priority "high" and a due date two hours out. (The screenshot is illustrative: it stores the LLM answer in `isEmailUrgnet` but tests `isUrgent`.)

### MCPs

- **Standardized integrations**: the Model Context Protocol (MCP) connects to external services and handles authentication and API calls automatically, so you can attach Slack, GitHub, Google Drive or Asana without writing custom integration code or managing OAuth flows yourself.
- **Email agent**: search Slack messages to understand team context, or check Asana tasks to see whether someone is already assigned to a customer request, by calling tools such as `search_slack_messages` or `get_asana_tasks`; the MCP server handles the rest.
- **Ecosystem**: the growing MCP ecosystem (the modelcontextprotocol/servers repository) lets you add capabilities as pre-built integrations appear and focus on agent behavior.

## 7. Verify your work

- **Why verification matters**: "The Claude Code SDK finishes the agentic loop by evaluating its work." Agents that can check and improve their own output are fundamentally more reliable: they catch mistakes before they compound, self-correct when they drift, and get better as they iterate. The key is giving Claude concrete ways to evaluate its work; the post offers three.

### Defining rules

- **Best form of feedback**: clearly defined rules for an output, plus an explanation of which rules failed and why.
- **Linting as the model case**: code linting is an excellent form of rules-based feedback, and the more in-depth the feedback the better; it is usually better to generate TypeScript and lint it than to generate pure JavaScript, because TypeScript adds multiple additional layers of feedback.
- **Email agent**: check that the email address is valid (if not, throw an error) and "that the user has sent an email to them before (if so, throw a warning)" (quoted as written).

### Visual feedback

- **For visual tasks** such as UI generation or testing, screenshots or renders help. For an HTML-formatted email, screenshot the generated email and hand it back to the model for visual verification and iterative refinement, checking whether the output matches what was requested:
  - Layout: are elements positioned correctly, is spacing appropriate?
  - Styling: do colors, fonts and formatting appear as intended?
  - Content hierarchy: is information in the right order with proper emphasis?
  - Responsiveness: does it look broken or cramped? (A single screenshot has limited viewport information.)
- **Automation**: an MCP server such as Playwright can take screenshots of rendered HTML, capture different viewport sizes and even test interactive elements inside the agent's workflow.
- **Figure (annotated email screenshot)**: a "TechCorp Analytics / Your Monthly Performance Report" email ("Hello Sarah") with every block labelled and measured as the model would see it: `header_section` (1040x110, "z-index: 1"), `greeting_text` (1040x65), `main_message` (1040x120, reporting a 47% increase in efficiency), three stat tiles each labelled `stat_box_1` (315x215) showing "47% Efficiency Gain", "128 Tasks Completed" and "4.8/5 Team Rating", a `cta_button` (335x75) "View Full Report" tagged "clickable", `closing_message` (1040x80, "top 5% of all teams this quarter") and `footer_section` (1040x80, copyright 2025 TechCorp Analytics with Unsubscribe, Privacy Policy and Contact Us links). Alt text: "Claude provides visual feedback on the body of an email generated by an agent." Caption: "Visual feedback from a large-language model (LLM) can provide helpful guidance to your agent."

### LLM as a judge

- **Fuzzy rules**: another language model can "judge" the agent's output against fuzzy rules. This is generally not very robust and can carry heavy latency trade-offs, but it helps where any boost in performance is worth the cost.
- **Email agent**: a separate subagent judges the tone of drafts against the user's previous messages.

## 8. Testing and improving your agent

- **Look at the failures**: after a few passes through the loop, test the agent. The best way to improve it is to look carefully at its output, especially the failing cases, and to put yourself in its shoes: does it have the right tools for the job?
- **Diagnostic questions from the post**:
  - If the agent misunderstands the task, it may be missing key information: can you alter the structure of your search APIs so it finds what it needs more easily?
  - If it fails at a task repeatedly, can you add a formal rule in your tool calls to identify and fix the failure?
  - If it cannot fix its errors, can you give it more useful or creative tools to approach the problem differently?
  - If performance varies as you add features, build a representative test set for programmatic evaluations (evals) based on customer usage.

## 9. Getting started

- **Summary claim**: the Claude Agent SDK makes autonomous agents easier to build by giving Claude a computer where it can write files, run commands and iterate on its work; with the loop in mind (gathering context, taking action, verifying work) you can build reliable agents that are easy to deploy and iterate on.
- **Links**: get started with the SDK overview at docs.claude.com; developers already building on the SDK are asked to migrate to the latest version by following the migration guide.

## Quick reference

| Loop stage | Capability | Email-agent example | Guidance in the post |
| --- | --- | --- | --- |
| Gather context | Agentic search over the file system (grep, tail via bash) | A conversations folder of past threads as dated Markdown files | Folder and file structure become a form of context engineering |
| Gather context | Semantic search (chunk, embed, query vectors) | Not shown | Faster but less accurate, harder to maintain, less transparent; add only for speed or more variations |
| Gather context | Subagents (parallel, isolated context windows) | Several search subagents, each with a different query, return only relevant excerpts | Ideal for sifting large volumes where most content is irrelevant |
| Gather context | Compaction | Not shown | Automatically summarizes earlier messages near the context limit; built on /compact |
| Take action | Tools | fetchInbox, searchEmails | Tools are prominent in context, so make them the primary, most frequent actions and design for context efficiency |
| Take action | Bash and scripts | pdftotext on an attachment, grep for "invoice", tail the last 10 hits | General-purpose, flexible computer work |
| Take action | Code generation | onEmailReceived rule: forward urgent customer mail to the team lead and open a high-priority task due in 2 hours | Code is precise, composable and reusable; Claude.ai file creation relies on it entirely |
| Take action | MCP | search_slack_messages, get_asana_tasks | Standard integrations handle auth and API calls; no custom OAuth |
| Verify work | Defining rules | Error on an invalid address; warning tied to prior correspondence | Clear rules plus which failed and why; TypeScript plus lint beats plain JavaScript |
| Verify work | Visual feedback | Screenshot the HTML email; check layout, styling, hierarchy, responsiveness; automate with the Playwright MCP | One screenshot carries limited viewport information |
| Verify work | LLM as a judge | A subagent judges draft tone against the user's past messages | Not very robust, heavy latency; use when any gain is worth the cost |
| Improve | Inspect failures and ask what the agent lacks | Restructure search APIs, add formal rules, add tools, build an eval set from customer usage | "Put yourself in its shoes: does it have the right tools for the job?" |

## Changes between the September 2025 original and the current page

- **Path history**: the article first appeared on 2025-09-29 at anthropic.com/engineering/ (earliest Wayback snapshot 2025-09-29 17:24 UTC; page title "Building agents with the Claude Agent SDK \ Anthropic", section label "Engineering at Anthropic"). A snapshot of 2026-01-29 shows it at claude.com/blog/, and snapshots from 2026-10-10 onward show the current claude.com/resources/articles/ address. No snapshot exists under anthropic.com/news/.
- **Structure unchanged**: the title, the standfirst, all section headings (nine H2 sections with the same eleven H3 subsections), the five figures (same alt texts and captions) and the acknowledgements are present in both versions. The figures were re-hosted: the original served SVG files from www-cdn.anthropic.com, the current page serves PNG/JPG files from assets.claude.com; the image contents themselves were not compared.
- **Four wording edits in the body** (all already present in the 2026-01-29 blog snapshot, so made between 2025-09-29 and 2026-01-29):
  1. The second paragraph gained the sentence "In fact, it has begun to power almost all of our major agent loops."
  2. The last paragraph of "Giving Claude a computer" gained the sentence "The key design principle behind the Claude Agent SDK is to give your agents a computer, allowing them to work like humans do."
  3. In "Creating new types of agents", "build agents that were not as effective as before" became "build agents that are more effective than before".
  4. In the acknowledgements, "Molly Vorweck" became "Molly Vorwerck".
- **Link changes**: the Claude Code link now points to claude.com/product/claude-code instead of anthropic.com/claude-code; the phrase "the agent harness that powers Claude Code" gained a link to claude.com/resources/articles/harnessing-claudes-intelligence; the claude.ai redirect identifier on the file-creation link changed.
- **Page chrome** (ignored): the current page adds four related-article cards (dated 2026-10-16 to 2026-10-27), a "Transform how your organization operates with Claude" block with pricing and sales links, and a restyled newsletter sign-up.
