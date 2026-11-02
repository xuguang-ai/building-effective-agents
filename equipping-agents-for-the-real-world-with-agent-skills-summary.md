# Equipping agents for the real world with Agent Skills — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Equipping agents for the real world with Agent Skills" (the page's H1; the browser-tab title now reads "Equipping agents with Agent Skills") |
| URL | https://www.anthropic.com/research/equipping-agents-for-the-real-world-with-agent-skills (first published under /engineering/) |
| Authors | Barry Zhang, Keith Lazuka and Mahesh Murag, "who all really like folders", with special thanks to the many others across Anthropic who championed, supported and built Skills |
| Published | 2025-10-16 (datePublished in the page metadata; dateModified is 2026-10-18) |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-10-16; differences listed at the end |
| Docs | Skills docs: https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview |
| Code | Skills cookbook: https://github.com/anthropics/claude-cookbooks/tree/main/skills. The worked example is the PDF skill: https://github.com/anthropics/skills/tree/main/document-skills/pdf |
| Update | A "Dec 18, 2025" update box on the current page: "We've published Agent Skills as an open standard for cross-platform portability", linking to https://agentskills.io/ |

## One-sentence takeaway

An Agent Skill is a folder whose SKILL.md file (YAML name and description, then Markdown instructions) can reference further files and scripts; the agent sees only every skill's name and description at startup and reads the rest from its filesystem when the task calls for it. This **progressive disclosure** lets anyone package procedural knowledge like an onboarding guide for a new hire, with effectively unbounded bundled context and deterministic code wherever code beats token generation.

## 1. Introduction: why Skills

- **The starting point**: as model capabilities improve, general-purpose agents can interact with full-fledged computing environments; Claude Code, for example, accomplishes complex tasks across domains using local code execution and filesystems. As agents get more powerful, "we need more composable, scalable, and portable ways to equip them with domain-specific expertise".
- **The definition**: Agent Skills are "organized folders of instructions, scripts, and resources that agents can discover and load dynamically to perform better at specific tasks". They package your expertise into composable resources for Claude, "transforming general-purpose agents into specialized agents that fit your needs".
- **The analogy**: building a skill is like putting together an onboarding guide for a new hire. Instead of fragmented, custom-designed agents for each use case, anyone can specialize their agents with composable capabilities by capturing and sharing procedural knowledge. The article promises to explain what Skills are, show how they work, and share best practices for building your own.
- **Figure "Agent + Skills + Virtual Machine"**: on the left, the agent configuration holds the core system prompt, the equipped Skills (bigquery, docx, nda-review, pdf, pptx, ..., xlsx) and the equipped MCP servers (MCP server 1 to 3, pointing at remote MCP servers "elsewhere on the internet"). A "use computer" arrow leads to the agent virtual machine on the right, with Bash, Python and Nodejs and a file system where the "Contents of Skill directories live": skills/bigquery/ (SKILL.md, datasources.md, rules.md), skills/docx/ (SKILL.md plus an ooxml/ folder with spec.md and editing.md), skills/nda-review/ (SKILL.md only), skills/pdf/ (SKILL.md, forms.md, reference.md, extract_fields.py) and "...etc...". Caption: a skill is a directory containing a SKILL.md file, with organized folders of instructions, scripts and resources that give agents additional capabilities.

## 2. The anatomy of a skill

- **The worked example** is one of the skills behind Claude's recently launched document editing abilities. Claude already knows a lot about understanding PDFs but is limited in manipulating them directly (for example filling out a form); the PDF skill gives it these new abilities.
- **Minimum structure**: a skill is a directory containing a SKILL.md file. The file must start with YAML frontmatter holding the required metadata name and description. At startup the agent pre-loads the name and description of **every installed skill** into its system prompt.
- **Level 1 of progressive disclosure** is that metadata: just enough for Claude to know when each skill should be used, without loading all of it into context. **Level 2** is the body of SKILL.md, which Claude reads in full when it judges the skill relevant to the current task.
- **Figure "A simple SKILL.md file"** (pdf/SKILL.md): the frontmatter is `name: pdf` and `description: Comprehensive PDF toolkit for extracting text and tables, merging/splitting documents, and filling-out forms.` The Markdown body has an "Overview" ("This guide covers essential PDF processing operations using Python libraries and command-line tools. For advanced features, JavaScript libraries, and detailed examples, see ./reference.md. If you need to fill out a PDF form, read ./forms.md and follow its instructions.") and a "Quick Start" Python block that imports PdfReader and PdfWriter from pypdf, opens "document.pdf" and prints the page count.
- **Level 3 and beyond**: as skills grow, they may hold too much context for one SKILL.md, or context relevant only in specific scenarios. Skills can then bundle additional files in the skill directory and reference them by name from SKILL.md; Claude navigates and discovers them only as needed.
- **In the PDF skill**, SKILL.md refers to two bundled files, reference.md and forms.md. Moving the form-filling instructions into forms.md keeps the core of the skill lean, "trusting that Claude will read forms.md only when filling out a form".
- **Figure "Bundling additional content"**: pdf/SKILL.md links to pdf/reference.md (a "PDF Processing Advanced Reference" with features, examples and libraries not covered in the main instructions, for example the pypdfium2 library, a Python binding for PDFium, Chromium's PDF library) and to pdf/forms.md (first check whether the PDF has fillable form fields by running `python scripts/check_fillable_fields <file.pdf>` from the file's directory, then follow either the "Fillable fields" or the "Non-fillable fields" instructions; the fillable branch runs `python scripts/extract_form_field_info.py <input.pdf> <fields.json>`). Caption: more context can be incorporated via additional files, which Claude triggers based on the system prompt.
- **Progressive disclosure is the core design principle** that makes Skills flexible and scalable: like a well-organized manual with a table of contents, then specific chapters, then a detailed appendix, Claude loads information only as needed. The post's table:

| Level | File | Context window | Tokens |
| --- | --- | --- | --- |
| 1 | SKILL.md metadata (YAML) | Always loaded | ~100 |
| 2 | SKILL.md body (Markdown) | Loaded when the skill triggers | <5k |
| 3+ | Bundled files (text files, scripts, data) | Loaded as needed by Claude | unlimited* |

The asterisk is in the original figure; the page does not explain it.

- **Unbounded context**: agents with a filesystem and code execution tools do not need to read a whole skill into the context window for a task, so "the amount of context that can be bundled into a skill is effectively unbounded".

### Skills and the context window

- **Figure "Skills and the Context Window"** shows the context window changing as a user message triggers a skill. The agent's system prompt is followed by short snippets from each skill (bigquery, docx, nda-review, pdf, pptx, ..., xlsx). The user writes "Fill out this PDF based on what you know about me" with /mnt/uploads/order_form.pdf attached. Claude answers "Certainly! I'll use the PDF Skill to help", runs `Bash("cat /mnt/skills/pdf/SKILL.md")` and gets the file contents back as the tool result, then decides to follow the reference and runs `Bash("cat /mnt/skills/pdf/forms.md")`.
- **The sequence in the post's words**: the context window starts with the core system prompt, the metadata of each installed skill and the user's initial message; Claude triggers the PDF skill by invoking a Bash tool to read pdf/SKILL.md; Claude chooses to read the bundled forms.md; finally Claude proceeds with the user's task with the relevant instructions loaded. Caption: skills are triggered in the context window via the system prompt.

### Skills and code execution

- **Code as tools**: skills can include code that Claude executes at its discretion. LLMs excel at many tasks, but some operations suit traditional code execution better: sorting a list via token generation is far more expensive than running a sorting algorithm, and many applications need "the deterministic reliability that only code can provide".
- **In the PDF skill**, a pre-written Python script reads a PDF and extracts all form fields. Claude can run it **without loading either the script or the PDF into context**, and because code is deterministic the workflow is consistent and repeatable.
- **Figure "Bundling executable scripts"**: pdf/forms.md tells Claude, for a PDF with fillable fields, to run `python ./extract_fields.py <input.pdf> <fields.json>` from the file's directory; pdf/extract_fields.py imports PdfReader from pypdf, defines write_field_info(pdf_path, output_path) ("Extract form fields from PDF and store as JSON": read the PDF, get the fields, json.dump them to the output file) and a __main__ block that checks for exactly two arguments and prints a usage line otherwise. (The earlier bundling figure names the script scripts/extract_form_field_info.py; the two figures do not agree on this file name.)

## 3. Developing and evaluating skills

Four guidelines for authoring and testing skills:

- **Start with evaluation**: identify specific gaps in your agents' capabilities by running them on representative tasks and observing where they struggle or need extra context, then build skills incrementally to address those shortcomings.
- **Structure for scale**: when SKILL.md becomes unwieldy, split its content into separate files and reference them. If certain contexts are mutually exclusive or rarely used together, keeping the paths separate reduces token usage. Code can serve both as executable tools and as documentation, so it should be clear whether Claude should run a script directly or read it into context as reference.
- **Think from Claude's perspective**: monitor how Claude uses the skill in real scenarios and iterate on what you observe; watch for unexpected trajectories or overreliance on certain contexts. Pay special attention to the **name and description**, which Claude uses when deciding whether to trigger the skill for its current task.
- **Iterate with Claude**: while working on a task, ask Claude to capture its successful approaches and common mistakes into reusable context and code within a skill; if it goes off track while using a skill, ask it to self-reflect on what went wrong. This reveals what context Claude actually needs instead of anticipating it upfront.

### Security considerations when using Skills

- **The risk**: skills give Claude new capabilities through instructions and code, so malicious skills may introduce vulnerabilities into the environment where they are used, or direct Claude to exfiltrate data and take unintended actions.
- **The advice**: install skills only from trusted sources. For a less-trusted source, audit thoroughly before use: read the contents of every bundled file to understand what it does, paying particular attention to code dependencies and bundled resources such as images or scripts, and to instructions or code that tell Claude to connect to potentially untrusted external network sources.

## 4. The future of Skills

- **Availability**: Agent Skills are supported today across Claude.ai, Claude Code, the Claude Agent SDK and the Claude Developer Platform.
- **Near term** ("in the coming weeks"): more features for the full lifecycle of creating, editing, discovering, sharing and using Skills; particular excitement about Skills helping organizations and individuals share their context and workflows with Claude; exploring how Skills can complement Model Context Protocol (MCP) servers by teaching agents more complex workflows that involve external tools and software.
- **Further ahead**: enabling agents to create, edit and evaluate Skills on their own, codifying their own patterns of behavior into reusable capabilities.
- **Closing**: "Skills are a simple concept with a correspondingly simple format", which makes it easier for organizations, developers and end users to build customized agents and give them new capabilities. Pointers to get started: the Skills docs and the cookbook.
- **Later update box** (not part of the original article): dated Dec 18, 2025, "We've published Agent Skills as an open standard for cross-platform portability", linking to agentskills.io.

## Quick reference

| Topic | What the article says |
| --- | --- |
| Skill | A directory with a SKILL.md file plus optional bundled files and scripts: "organized folders of instructions, scripts, and resources" |
| SKILL.md | Required YAML frontmatter (name, description) followed by Markdown instructions |
| Level 1 | Name and description of every installed skill, pre-loaded into the system prompt at startup; about 100 tokens |
| Level 2 | The SKILL.md body, read in full when Claude decides the skill is relevant; under 5k tokens |
| Level 3+ | Bundled files referenced by name from SKILL.md (reference.md, forms.md, scripts), loaded as needed; effectively unbounded |
| Triggering | Claude itself reads the skill file with a Bash call such as cat /mnt/skills/pdf/SKILL.md, then follows its references |
| Scripts | Run by Claude without loading the script or its input into context; deterministic and repeatable (PDF form-field extraction) |
| Authoring | Start with evaluation; structure for scale; think from Claude's perspective (name and description matter most); iterate with Claude |
| Security | Install only from trusted sources; audit less-trusted skills: every file, code dependencies, bundled resources, network access |
| Where supported | Claude.ai, Claude Code, Claude Agent SDK, Claude Developer Platform |
| Roadmap | Lifecycle features, complementing MCP servers, agents authoring and evaluating their own Skills; later (Dec 2025) published as an open standard at agentskills.io |

## Changes between the October 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-10-16 (16:37 UTC, the earliest capture, under /engineering/) shows the article body identical word for word, with the same six figures (same CDN image files, alt texts and captions) and the same acknowledgements line. The differences are page chrome:

- **Moved path**: from /engineering/ to /research/ (the first archived capture under /research/ is dated 2026-10-20; no /news/ copy exists in the archive).
- **Title tag**: the original browser-tab title was "Equipping agents for the real world with Skills"; the current one is "Equipping agents with Agent Skills". The H1 "Equipping agents for the real world with Agent Skills" is unchanged.
- **Standfirst dropped**: "Claude is powerful, but real work requires procedural knowledge and organizational context. Introducing Agent Skills, a new way to build specialized agents using files and folders." no longer appears. The "Engineering at Anthropic" label and the visible "Published Oct 16, 2025" line are also absent from the current page's extracted body text (the machine-readable datePublished is still 2025-10-16; dateModified is 2026-10-18).
- **Added**: the "Dec 18, 2025" Update box about the open standard (agentskills.io), a "Chapter 01: The anatomy of a skill" in-page navigation entry, and a "Related articles" block (Building a C compiler with a team of parallel Claudes).
- **Restructured**: the acknowledgements now sit under an "Authors & acknowledgements" heading; the developer-newsletter sign-up block of the original is gone.
