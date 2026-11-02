# building-effective-agents

Reading summaries of Anthropic's engineering articles on building AI agents, one English Markdown file per article.

Each summary has the same shape: a metadata table (source, authors, publication date, access date), a one-sentence takeaway, section-by-section notes that keep the article's numbers, examples and named tools, a quick-reference table, and a final section listing how the live page differs from its earliest Wayback Machine snapshot.

| Article | Published | Summary |
| --- | --- | --- |
| [Building effective AI agents](https://www.anthropic.com/research/building-effective-agents) | 2024-12-19 | [building-effective-agents-summary.md](building-effective-agents-summary.md) |
| [How we built our multi-agent research system](https://www.anthropic.com/research/multi-agent-research-system) | 2025-06-13 | [multi-agent-research-system-summary.md](multi-agent-research-system-summary.md) |
| [Writing effective tools for AI agents](https://www.anthropic.com/research/writing-tools-for-agents) | 2025-09-11 | [writing-tools-for-agents-summary.md](writing-tools-for-agents-summary.md) |
| [Effective context engineering for AI agents](https://www.anthropic.com/research/effective-context-engineering-for-ai-agents) | 2025-09-29 | [effective-context-engineering-for-ai-agents-summary.md](effective-context-engineering-for-ai-agents-summary.md) |
| [Building agents with the Claude Agent SDK](https://claude.com/resources/articles/building-agents-with-the-claude-agent-sdk) | 2025-09-29 | [building-agents-with-the-claude-agent-sdk-summary.md](building-agents-with-the-claude-agent-sdk-summary.md) |
| [Equipping agents for the real world with Agent Skills](https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills) | 2025-10-16 | [equipping-agents-for-the-real-world-with-agent-skills-summary.md](equipping-agents-for-the-real-world-with-agent-skills-summary.md) |
| [Beyond permission prompts: making Claude Code more secure and autonomous](https://www.anthropic.com/research/claude-code-sandboxing) | 2025-10-20 | [claude-code-sandboxing-summary.md](claude-code-sandboxing-summary.md) |
| [Code execution with MCP: Building more efficient agents](https://www.anthropic.com/research/code-execution-with-mcp) | 2025-11-04 | [code-execution-with-mcp-summary.md](code-execution-with-mcp-summary.md) |
| [Introducing advanced tool use for Claude](https://www.anthropic.com/research/advanced-tool-use) | 2025-11-24 | [advanced-tool-use-summary.md](advanced-tool-use-summary.md) |
| [Effective harnesses for long-running agents](https://www.anthropic.com/research/effective-harnesses-for-long-running-agents) | 2025-11-26 | [effective-harnesses-for-long-running-agents-summary.md](effective-harnesses-for-long-running-agents-summary.md) |
| [Demystifying evals for AI agents](https://www.anthropic.com/research/demystifying-evals-for-ai-agents) | 2026-01-09 | [demystifying-evals-for-ai-agents-summary.md](demystifying-evals-for-ai-agents-summary.md) |
| [Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/research/managed-agents) | 2026-04-08 | [managed-agents-summary.md](managed-agents-summary.md) |

## How the summaries were made

The summaries were written with Claude Code on 2026-11-02 from the live pages. The raw HTML of each article was downloaded and converted to text, inline figures were viewed so that chart values and code screenshots appear in the notes, and each page was compared paragraph by paragraph with the earliest Wayback Machine snapshot of the article. Most of these posts moved from anthropic.com/engineering/ to anthropic.com/research/ in October 2026 with their standfirsts removed and their bodies left almost unchanged; the exceptions are recorded in each file's last section.
