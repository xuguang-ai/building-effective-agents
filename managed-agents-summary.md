# Scaling Managed Agents: Decoupling the brain from the hands — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Scaling Managed Agents: Decoupling the brain from the hands" (the on-page headline and the JSON-LD headline; the HTML `<title>` and og:title of the current page instead read "Scaling Managed Agents with stable interfaces") |
| URL | https://www.anthropic.com/research/managed-agents (first published under /engineering/managed-agents with the "Engineering at Anthropic" masthead; other anthropic.com pages still link to the /engineering/ path) |
| Authors | Lance Martin, Gabe Cemaj and Michael Cohen; thanks to N.T.A. and Jeremy Fox for conversations, and to the Agents API team and Jake Eaton for their contributions (the JSON-LD author field names only the organization "Anthropic") |
| Published | 2026-04-08 (JSON-LD datePublished; the page shows "Published Apr 8, 2026"; dateModified 2026-10-30) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshots of 2026-04-08 and 2026-04-09 (both under /engineering/); differences listed at the end |
| Docs | "Get started with Claude Managed Agents by following our docs": https://platform.claude.com/docs/en/managed-agents/overview |
| Cited Anthropic posts | Building effective agents; Effective harnesses for long-running agents; Harness design for long-running apps (/engineering/harness-design-long-running-apps); Effective context engineering for AI agents |
| Other references | "The Bitter Lesson" (incompleteideas.net/IncIdeas/BitterLesson.html); "programs as yet unthought of" (catb.org/esr/writings/taoup, chapter 3); the pets-vs-cattle history (cloudscaling.com); arXiv 2512.24601 on context stored outside the context window |

## One-sentence takeaway

Harnesses encode assumptions about what Claude cannot yet do, and those assumptions go stale as models improve (the context resets added for Sonnet 4.5's "context anxiety" were dead weight on Opus 4.5), so Managed Agents is built as a **meta-harness**: like an operating system virtualizing hardware into process and file, it virtualizes the agent into a **session** (append-only event log), a **harness** (the loop that calls Claude and routes its tool calls) and a **sandbox** (where code runs), each behind a small stable interface, so that the "brain" (Claude plus harness) is decoupled from the "hands" (sandboxes and tools) and any part can fail or be replaced on its own.

## 1. Introduction: harness assumptions go stale, interfaces should not

- **A running topic** on the Engineering Blog is how to build effective agents and design harnesses for long-running work. The common thread: harnesses encode assumptions about what Claude cannot do on its own, and those assumptions must be questioned often because they go stale as models improve (the post links The Bitter Lesson here).
- **The worked example**: prior work on harness design for long-running apps found that Claude Sonnet 4.5 wrapped up tasks prematurely as it sensed its context limit approaching, a behavior "sometimes called 'context anxiety'". The fix was adding context resets to the harness. Run on Claude Opus 4.5, the same harness showed no such behavior: "The resets had become dead weight."
- **Managed Agents** is Anthropic's answer to harnesses that keep evolving: a hosted service in the Claude Platform that runs long-horizon agents on your behalf through a small set of interfaces "meant to outlast any particular implementation—including the ones we run today".
- **The operating-system analogy**: the old problem of designing a system for "programs as yet unthought of". Operating systems virtualized hardware into abstractions (process, file) general enough for programs that did not yet exist; `read()` does not care whether it is reading a 1970s disk pack or a modern SSD. The abstractions stayed stable while implementations changed freely.
- **The three virtualized components**: a session (the append-only log of everything that happened), a harness (the loop that calls Claude and routes Claude's tool calls to the relevant infrastructure) and a sandbox (an execution environment where Claude can run code and edit files). Each implementation can be swapped without disturbing the others: "We're opinionated about the shape of these interfaces, not about what runs behind them."
- **Figure 1 (uncaptioned architecture diagram)**: a Harness box with the Claude spark in the center; around it Session (checklist icon, left), Sandbox (terminal prompt icon, right), Tools "+ Resources / MCP" (toolbox icon, top) and Orchestration (flow icon, bottom). Every connection is drawn as a solid arrow one way and a dashed arrow back, that is, a request-and-response pair between the harness and each neighbor.

## 2. Don't adopt a pet

- **The first design** put all agent components into a single container, so session, harness and sandbox shared one environment. Benefits: file edits were direct syscalls and there were no service boundaries to design.
- **The cost**: coupling everything into one container meant "we'd adopted a pet". In the pets-vs-cattle analogy a pet is a named, hand-tended individual you cannot afford to lose; cattle are interchangeable. The server became the pet: if a container failed, the session was lost; if it was unresponsive, it had to be nursed back to health.
- **Nursing meant debugging stuck sessions** with the WebSocket event stream as the only window in. That stream could not say where a failure arose: a bug in the harness, a packet drop in the event stream and a container going offline all looked the same. Finding out required an engineer to open a shell inside the container, but the container often also held user data, so in effect "we lacked the ability to debug".
- **A second issue**: the harness assumed that whatever Claude worked on lived in the same container. Customers who wanted Claude connected to their virtual private cloud had to either peer their network with Anthropic's or run Anthropic's harness in their own environment. An assumption baked into the harness became a problem as soon as it had to connect to different infrastructure.

## 3. Decouple the brain from the hands

- **The solution**: decouple the "brain" (Claude and its harness) from both the "hands" (sandboxes and tools that perform actions) and the "session" (the log of session events). Each became an interface that makes few assumptions about the others, and each can fail or be replaced independently.
- **The harness leaves the container.** The harness now calls the container the way it calls any other tool: `execute(name, input) → string`. The container became cattle: if it died, the harness caught the failure as a tool-call error and passed it back to Claude; if Claude decided to retry, a new container could be reinitialized with a standard recipe, `provision({resources})`. No more nursing failed containers.
- **Recovering from harness failure.** The harness also became cattle. Because the session log sits outside the harness, nothing in the harness needs to survive a crash: a new one is rebooted with `wake(sessionId)`, calls `getSession(id)` to get the event log back, and resumes from the last event. During the agent loop the harness writes to the session with `emitEvent(id, event)` to keep a durable record.
- **Figure 2 (interface table)**, reproduced from the image:

| Component | Interface (pseudocode) | Satisfied by |
| --- | --- | --- |
| Session | `getSession(session_id) -> (Session, Event[])`; `getEvents(session_id) -> PendingEvent[]` (comment: not yet processed); `emitEvent(id, event)` | Any append-only log that can be consumed in order from any event point and accepts idempotent appends: Postgres, SQLite, in-memory array, etc. |
| Orchestration | `wake(session_id) -> void` | Any scheduler that can call a function with an ID and retry on failure: a cron job, a queue consumer, a while-loop, etc. |
| Harness | `yield Effect<T> -> EffectResult<T>` | Any loop that yields effects and appends progress to the Session. |
| Sandbox | `provision({resources})`; `execute(name, input) -> String` | Any executor that can be configured once and called many times as a tool: a local process, a remote container, etc. |
| Resources | `[{source_ref, mount_path}]` | Any durable store the container can fetch from by reference: Filestore, GCS, a git remote, S3. |
| Tools | `{name, description, input_schema}` | Any capability describable as a name and an input shape: MCP server, custom tool, etc. |

- **The security boundary.** In the coupled design, untrusted code generated by Claude ran in the same container as credentials, so a prompt injection only had to convince Claude to read its own environment; with those tokens an attacker can spawn fresh, unrestricted sessions and delegate work to them. Narrow scoping is the obvious mitigation, but it encodes an assumption about what Claude cannot do with a limited token, "and Claude is getting increasingly smart". The structural fix: tokens are never reachable from the sandbox where Claude's generated code runs.
- **Two patterns enforce this**: auth is either bundled with a resource or held in a vault outside the sandbox. For Git, each repository's access token is used to clone the repo during sandbox initialization and is wired into the local git remote, so git push and pull work from inside the sandbox without the agent ever handling the token. For custom tools, MCP is supported and OAuth tokens live in a secure vault; Claude calls MCP tools through a dedicated proxy that takes a token associated with the session, fetches the matching credentials from the vault and makes the call to the external service. "The harness is never made aware of any credentials."

## 4. The session is not Claude's context window

- **The problem**: long-horizon tasks often exceed Claude's context window, and the standard remedies all involve irreversible decisions about what to keep. Prior work on context engineering covered them: compaction lets Claude save a summary of its context window; the memory tool lets Claude write context to files, enabling learning across sessions; both can be paired with context trimming, which selectively removes tokens such as old tool results or thinking blocks.
- **Why irreversibility hurts**: it is difficult to know which tokens future turns will need. After a compaction step the harness removes the compacted messages from Claude's context window, and they are recoverable only if they were stored somewhere. Prior work (arXiv 2512.24601) explored storing context as an object outside the context window, for example an object in a REPL that the LLM accesses programmatically by writing code to filter or slice it.
- **Figure 3 (small diagram)**: Session (checklist) on the left, Harness (Claude spark) on the right; an arrow labeled "Events" runs from Session to Harness and an arrow labeled "getEvents" runs back from Harness to Session, with a dashed arrow inside the harness box feeding the fetched events to Claude.
- **The Managed Agents answer**: the session plays the role of that context object, but context is durably stored in the session log rather than in the sandbox or a REPL. The `getEvents()` interface lets the brain interrogate context by selecting positional slices of the event stream: pick up where it last stopped reading, rewind a few events before a specific moment to see the lead-up, or reread context before a specific action.
- **Transformation happens in the harness**: fetched events can be transformed before they reach Claude's context window, with whatever the harness encodes, including context organization for a high prompt cache hit rate and context engineering. Recoverable context storage (session) and arbitrary context management (harness) are separated because "we can't predict what specific context engineering will be required in future models"; the interfaces only guarantee that the session is durable and available for interrogation.

## 5. Many brains, many hands

- **Many brains: the VPC complaint.** One of the earliest customer complaints was that teams wanting Claude to work against resources in their own VPC had no path except peering their network with Anthropic's, because the container holding the harness assumed every resource sat next to it. Once the harness left the container, that assumption went away.
- **The performance payoff**: with the brain inside a container, many brains needed as many containers, and no inference could happen for a brain until its container was provisioned. Every session paid the full container setup cost up front, even sessions that would never touch the sandbox: clone the repo, boot the process, fetch pending events from Anthropic's servers.
- **Time-to-first-token (TTFT)** measures how long a session waits between accepting work and producing its first response token, "the latency the user most acutely feels".
- **After decoupling**, the brain provisions containers through a tool call (`execute(name, input) → string`) only when needed, so a session that did not need a container right away did not wait for one, and inference could start as soon as the orchestration layer pulled pending events from the session log. Result: **p50 TTFT dropped roughly 60% and p95 dropped over 90%**. Scaling to many brains means starting many stateless harnesses and connecting them to hands only if needed.
- **Many hands: the cognitive step.** Connecting each brain to many hands means Claude must reason about many execution environments and decide where to send work, a harder cognitive task than operating in a single shell. The brain started in a single container because earlier models were not capable of this; as intelligence scaled, the single container became the limitation instead, because when it failed, state was lost for every hand the brain was reaching into.
- **Each hand is a tool**, `execute(name, input) → string`: a name and input go in, a string comes back. The interface supports any custom tool, any MCP server and Anthropic's own tools. "The harness doesn't know whether the sandbox is a container, a phone, or a Pokémon emulator." Because no hand is coupled to any brain, brains can pass hands to one another.
- **Figure 4 (small diagram)**: a stack of three Harness cards (Claude spark) on the left and a stack of three Sandbox cards (terminal prompt) on the right, joined by a dashed arrow toward the sandboxes and a solid arrow back: many harnesses, many sandboxes, any-to-any.

## 6. Conclusion

- **Same old problem**: designing for "programs as yet unthought of". Operating systems have lasted decades by virtualizing hardware into abstractions general enough for programs that did not exist yet; Managed Agents aims to accommodate future harnesses, sandboxes or other components around Claude in the same way.
- **A meta-harness**: Managed Agents is unopinionated about the specific harness Claude will need in the future; it is a system of general interfaces that allows many different harnesses. Claude Code is cited as "an excellent harness that we use widely across tasks", and task-specific agent harnesses have been shown to excel in narrow domains; Managed Agents can accommodate any of these, "matching Claude's intelligence over time".
- **What the design is opinionated about**: the interfaces around Claude. Claude will need to manipulate state (the session) and perform computation (the sandbox), and to scale to many brains and many hands; the interfaces are designed so these run reliably and securely over long time horizons, with no assumptions about the number or location of brains or hands.

## Quick reference

| Term in the post | What it means |
| --- | --- |
| Brain | Claude plus its harness |
| Hands | Sandboxes and tools that perform actions; every hand is a tool with the signature `execute(name, input) → string` |
| Session | The durable, append-only log of everything that happened; lives outside both harness and sandbox; read by positional slices with `getEvents()`; satisfied by Postgres, SQLite or an in-memory array |
| Harness | The stateless loop that calls Claude and routes tool calls; rebooted with `wake(sessionId)` and `getSession(id)`, writes with `emitEvent(id, event)`; Claude Code is one example of a harness |
| Sandbox | Execution environment provisioned on demand with `provision({resources})`; a local process or a remote container |
| Orchestration | Any scheduler that can wake a harness for a session ID and retry on failure (cron job, queue consumer, while-loop) |
| Resources | Durable stores mounted by reference, `[{source_ref, mount_path}]`: Filestore, GCS, a git remote, S3 |
| Tools | `{name, description, input_schema}`: MCP servers or custom tools, called through a proxy that holds the credentials |
| Pet vs cattle | The single container was a pet (a lost container meant a lost session); decoupling made both containers and harnesses interchangeable cattle |
| Credential rule | Tokens are never reachable from the sandbox: Git tokens are wired into the local remote at initialization, OAuth tokens stay in a vault behind an MCP proxy, and the harness never sees credentials |
| Context anxiety | Sonnet 4.5 wrapping up early near its context limit; the context resets added for it were dead weight on Opus 4.5, the post's example of a stale harness assumption |
| TTFT result | After decoupling, p50 time-to-first-token fell roughly 60% and p95 fell over 90% |
| Meta-harness | A system opinionated about the interfaces around Claude (session, sandbox, many brains, many hands) but not about which harness runs inside them |

## Changes between the April 2026 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshots of 2026-04-08 (20:01 UTC) and 2026-04-09 (01:47 UTC), both captured at /engineering/managed-agents and identical to each other in text, shows the body, the section headings and the four figures identical word for word with the current page. The differences are at the edges:

- **Path and framing**: the article moved from /engineering/ to /research/ (the current canonical link is /research/managed-agents; JSON-LD dateModified is 2026-10-30). The "Engineering at Anthropic" masthead is gone; the current header shows "Team: Engineering" and the topics Agents, Claude Developer Platform, Context engineering, Tool use and Safeguards.
- **HTML title**: changed from "Scaling Managed Agents: Decoupling the brain from the hands" to "Scaling Managed Agents with stable interfaces", while the on-page H1 kept the original headline.
- **Standfirst dropped**: "Harnesses encode assumptions that go stale as models improve. Managed Agents—our hosted service for long-horizon agent work—is built around interfaces that stay stable as harnesses change." no longer appears on the page; a reworded version survives only as the meta description.
- **Acknowledgements edited**: the April text "Written by Lance Martin, Gabe Cemaj and Michael Cohen. Special thanks to the Agents API team and Jake Eaton for their contributions." now reads "Written by Lance Martin, Gabe Cemaj, and Michael Cohen. Thanks to N.T.A. and Jeremy Fox for helpful conversations on these topics. Special thanks to the Agents API team and Jake Eaton for their contributions.", under a new "Authors & acknowledgements" heading.
- **One link fixed**: in April the words "design harnesses" linked to a staging host (www.staging.ant.dev/engineering/effective-harnesses-for-long-running-agents); the current page links to www.anthropic.com/engineering/effective-harnesses-for-long-running-agents.
- **Page chrome**: the developer-newsletter sign-up block of the April page is gone. Wayback holds no snapshot of /news/managed-agents, and its snapshots of /research/managed-agents begin on 2026-10-20.
