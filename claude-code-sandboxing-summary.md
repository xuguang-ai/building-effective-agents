# Beyond permission prompts: making Claude Code more secure and autonomous — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Beyond permission prompts: making Claude Code more secure and autonomous" (on-page headline; the browser-tab title is "Making Claude Code safer with sandboxing") |
| URL | https://www.anthropic.com/research/claude-code-sandboxing (first published under /engineering/) |
| Authors | David Dworken and Oliver Weller-Davies, with contributions from Meaghan Choi, Catherine Wu, Molly Vorwerck, Alex Isken, Kier Bradwell and Kevin Garcia |
| Published | 2025-10-20 ("datePublished" in the page's JSON-LD; the page shows "Published Oct 20, 2025"; "dateModified" is 2026-10-18) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-10-20; differences listed at the end |
| Code | sandbox-runtime, an open-source research preview, in the anthropic-experimental/sandbox-runtime repository on GitHub |
| Docs | Claude Code sandboxing docs (docs.claude.com/en/docs/claude-code/sandboxing) and Claude Code on the web docs (docs.claude.com/en/docs/claude-code/claude-code-on-the-web); the article also points to the Claude Code on the web launch post (anthropic.com/news/claude-code-on-the-web) |

## One-sentence takeaway

Claude Code's ask-before-every-command permission model causes "approval fatigue", so Anthropic replaced per-action prompts with **sandboxing**: OS-enforced filesystem and network boundaries (Linux bubblewrap, MacOS seatbelt, a proxy outside the sandbox) inside which Claude runs commands freely, which in Anthropic's internal usage cut permission prompts by 84% while keeping even a successfully prompt-injected Claude from stealing SSH keys or phoning home; a second feature, Claude Code on the web, runs each session in an isolated cloud sandbox with credentials kept outside and git traffic routed through a validating proxy.

## 1. Keeping users secure on Claude Code

- **What Claude Code does**: Claude writes, tests and debugs code alongside you, navigating the codebase, editing multiple files and running commands to verify its work. Giving Claude this much access to the codebase and files introduces risks, "especially in the case of prompt injection".
- **Two new features built on sandboxing** are introduced. Both are designed to give developers a more secure place to work while letting Claude run more autonomously with fewer permission prompts. In Anthropic's internal usage, sandboxing "safely reduces permission prompts by 84%". The stated principle: "By defining set boundaries within which Claude can work freely, they increase security and agency."
- **The default permission model**: Claude Code is read-only by default and asks for permission before making modifications or running any commands. Exceptions exist: safe commands like echo or cat are auto-allowed, but most operations still need explicit approval.
- **Approval fatigue**: constantly clicking "approve" slows development cycles and can lead to users not paying close attention to what they approve, which in turn makes development less safe. Sandboxing for Claude Code was launched to address this.

## 2. Sandboxing: a safer and more autonomous approach

- **Pre-defined boundaries instead of per-action permission**: with sandboxing enabled, Claude works more freely within set limits, giving "drastically fewer permission prompts and increased safety".
- **Built on operating-system-level features** to enable two boundaries:
  - **Filesystem isolation**: Claude can only access or modify specific directories. This matters most for stopping a prompt-injected Claude from modifying sensitive system files.
  - **Network isolation**: Claude can only connect to approved servers. This stops a prompt-injected Claude from leaking sensitive information or downloading malware.
- **Both boundaries are required**: without network isolation a compromised agent could exfiltrate sensitive files such as SSH keys; without filesystem isolation a compromised agent could easily escape the sandbox and gain network access. Only the combination provides "a safer and faster agentic experience".

### 2a. Sandboxed bash tool: safe bash execution without permission prompts

- **A new sandbox runtime**, in beta as a research preview, lets you define exactly which directories and network hosts an agent can access, "without the overhead of spinning up and managing a container". It can sandbox arbitrary processes, agents and MCP servers, and is also released as an open-source research preview (anthropic-experimental/sandbox-runtime).
- **In Claude Code the runtime sandboxes the bash tool**: Claude runs commands within the limits you set and, inside the sandbox, executes them autonomously without permission prompts. If Claude tries to access something outside the sandbox, you are notified immediately and can choose whether or not to allow it.
- **OS-level primitives**: the restrictions are enforced with Linux bubblewrap and MacOS seatbelt, so they cover not only Claude Code's direct interactions but also any scripts, programs or subprocesses spawned by the command.
- **How each boundary is enforced**:
  - Filesystem: read and write access to the current working directory; modification of any file outside it is blocked.
  - Network: internet access is allowed only through a unix domain socket connected to a proxy server running outside the sandbox. The proxy restricts the domains a process can connect to and handles user confirmation for newly requested domains. For further-increased security the proxy can be customized to enforce arbitrary rules on outgoing traffic.
- **Both components are configurable**: specific file paths or domains can be allowed or disallowed.
- **Figure 1, "Claude Code Sandboxing"** (caption: the architecture "isolates code execution with filesystem and network controls, automatically allowing safe operations, blocking malicious ones, and asking permission only when needed"). The diagram has three columns, User, Claude code and Sandbox. The user's request becomes the Bash tool call `python foo.py`, which runs inside the sandbox as `bash -c 'python foo.py'`. Its network calls leave through socat and a domain socket to a second socat outside the sandbox, which splits traffic into UDP/SSH/etc to a SOCKS proxy and HTTP(s) to an HTTP proxy, both inside Claude Code. Errors are streamed to a log listener in Claude Code and shown to the user as failures. The decisions shown:

| Attempt in the diagram | Decision |
| --- | --- |
| Write `./README.md` (inside the working directory) | Allow |
| Write `~/.bad_file` (outside the working directory) | Deny |
| HTTPS to a denied host on port 443 | Deny |
| SOCKS proxy, allowed host on port 443 | Allow |
| SOCKS proxy, unknown host on port 80 | Ask user for permission |
| HTTP proxy, denied host on port 443 | Deny |
| HTTP proxy, unknown host on port 443 | Ask user for permission |

- **Result**: even a successful prompt injection is fully isolated and cannot impact overall user security; a compromised Claude Code "can't steal your SSH keys, or phone home to an attacker's server".
- **Getting started**: run `/sandbox` in Claude Code; the docs describe the security model.
- **Open sourced** so other teams can build safer agents; Anthropic says others "should consider adopting this technology for their own agents in order to enhance the security posture of their agents".

### 2b. Claude Code on the web: running Claude Code securely in the cloud

- **Released the same day** ("Today, we're also releasing Claude Code on the web"): each Claude Code session executes in an isolated sandbox in the cloud, where it has full access to its server "in a safe and secure way".
- **Credentials stay outside the sandbox**: the design ensures sensitive credentials such as git credentials or signing keys are never inside the sandbox with Claude Code, so even if the code running in the sandbox is compromised, the user is kept safe from further harm.
- **A custom proxy handles all git interactions** transparently. Inside the sandbox the git client authenticates to this service with a custom-built scoped credential. The proxy verifies the credential and the contents of the git interaction (for example, that it is only pushing to the configured branch), then attaches the right authentication token before sending the request to GitHub.
- **Figure 2, "Claude.com/code"** (caption: the Git integration "routes commands through a secure proxy that validates authentication tokens, branch names, and repository destinations—allowing safe version control workflows while preventing unauthorized pushes"). Three panels: a "GVisor Sandbox" holding a Claude Code window, a "Git proxy", and "Github.com". Inside the sandbox, Claude Code edits or creates files; `git add .` is auto-allowed; `git commit -m "..."` becomes "Sign this commit (auth=JWT)" and `git push` becomes "Push to github.com/foo/bar (auth=JWT)", both sent to the proxy. The proxy performs three checks, 1. validate JWT auth, 2. validate pushed branch name, 3. validate pushed repo destination, before forwarding to GitHub. The labels "GVisor" and "JWT" appear only in the diagram; the body text says "isolated sandbox" and "custom-built scoped credential".

## 3. Getting started

- **Claimed benefit**: the sandboxed bash tool and Claude Code on the web "offer substantial improvements in both security and productivity" for developers using Claude for engineering work.
- **Steps**: run `/sandbox` in Claude and read the docs on configuring the sandbox; go to claude.com/code (the link target is claude.ai/code) to try Claude Code on the web.
- **For agent builders**: check out the open-sourced sandboxing code and consider integrating it; "We look forward to seeing what you build."
- **More on Claude Code on the web**: the launch blog post at anthropic.com/news/claude-code-on-the-web.

## Quick reference

| Topic | What the article says |
| --- | --- |
| Default permission model | Read-only by default; asks before modifications or commands; auto-allows safe commands like echo or cat |
| Problem addressed | Approval fatigue from constant "approve" clicks, which slows development and makes it less safe |
| Measured effect | Permission prompts reduced by 84% in Anthropic's internal usage |
| Filesystem boundary | Read and write only in the current working directory; writes outside it blocked |
| Network boundary | Only via a unix domain socket to a proxy outside the sandbox; domain allow/deny, confirmation for new domains, proxy customizable for arbitrary outgoing-traffic rules |
| Enforcement | OS primitives Linux bubblewrap and MacOS seatbelt; covers spawned scripts, programs and subprocesses |
| Why both boundaries | Network-only gap: exfiltration of files such as SSH keys; filesystem-only gap: escape and network access |
| Runtime | sandbox runtime, beta research preview, no container needed; sandboxes arbitrary processes, agents and MCP servers; open source at anthropic-experimental/sandbox-runtime |
| How to enable | `/sandbox` in Claude Code; details in the sandboxing docs |
| Claude Code on the web | One isolated cloud sandbox per session; git credentials and signing keys never inside; git proxy verifies a scoped credential, the branch and the destination, then attaches the real token for GitHub |
| Threats named | Prompt injection; modifying sensitive system files; leaking information or downloading malware; stealing SSH keys; phoning home to an attacker's server; unauthorized pushes |

## Changes between the October 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-10-20 (18:34 UTC, original URL under /engineering/; the index lists five snapshots of that path from 2025-10-20 to 2025-11-07, one snapshot of the /research/ path dated 2026-10-23 and none of a /news/ path) shows every body paragraph and both figures (same image files, alt text and captions) identical word for word. The differences are in title, framing and page chrome:

- **Moved path**: /engineering/claude-code-sandboxing became /research/claude-code-sandboxing.
- **Browser-tab title changed** from "Making Claude Code more secure and autonomous with sandboxing" to "Making Claude Code safer with sandboxing". The on-page headline "Beyond permission prompts: making Claude Code more secure and autonomous" and the "Published Oct 20, 2025" line are unchanged.
- **Standfirst dropped**: the original's lead-in, "Claude Code's new sandboxing features, a bash tool and Claude Code on the web, reduce permission prompts and increase user safety by enabling two boundaries: filesystem and network isolation.", no longer appears.
- **Heading level**: "Keeping users secure on Claude Code" was an H3 and is now an H2, listed as chapter 01 in a new three-chapter table of contents (01 Keeping users secure on Claude Code, 02 Sandboxing: a safer and more autonomous approach, 03 Getting started).
- **Author credit edited**: Meaghan Choi was added to the contributors (the original credited Catherine Wu, Molly Vorwerck, Alex Isken, Kier Bradwell and Kevin Garcia), and the credit line now appears twice, under "Contributors" and under "Acknowledgements".
- **Chrome**: the "Engineering at Anthropic" label and the developer-newsletter sign-up are gone; a "Related articles" block linking "How we contain Claude across products" was added. The meta description was rewritten from "Learn how Claude Code's new sandboxing feature protects developers with filesystem and network isolation, reducing permission prompts and increasing user safety." to "Learn how sandboxing in Claude Code uses filesystem and network isolation to protect developers, reduce permission prompts, and increase autonomy." The JSON-LD "dateModified" is 2026-10-18.
