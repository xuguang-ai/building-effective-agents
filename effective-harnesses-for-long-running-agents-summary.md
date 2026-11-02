# Effective harnesses for long-running agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Effective harnesses for long-running agents" |
| URL | https://www.anthropic.com/research/effective-harnesses-for-long-running-agents (first published under /engineering/) |
| Author | Justin Young, with thanks to David Hershey, Prithvi Rajasakeran, Jeremy Hadfield, Naia Bouscal, Michael Tingley, Jesse Mu, Jake Eaton, Marius Buleandra, Maggie Vo, Pedram Navid, Nadine Yasser and Alex Notov |
| Published | 2025-11-26 |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2025-11-27; differences listed at the end |
| Code | Autonomous coding quickstart in the anthropics/claude-quickstarts repository |

## One-sentence takeaway

Long-running agents fail because every session starts with no memory of the last one. The fix is a harness with an **initializer agent** that scaffolds the environment (feature list, progress file, init script, git repo) and a **coding agent** that makes one feature of progress per session, tests it end to end, and leaves a clean, documented state for the next session.

## 1. The long-running agent problem

- Developers increasingly want agents on tasks that span hours or days, but consistent progress across multiple context windows is still an open problem.
- Agents work in discrete sessions, and each new session begins with no memory of what came before: like a software project staffed by engineers in shifts, where each arriving engineer knows nothing about the previous shift.
- The Claude Agent SDK is a general-purpose harness with context management such as compaction, so in theory an agent could keep doing useful work indefinitely. In practice **compaction is not sufficient**: even Opus 4.5 running in a loop across many windows falls short of a production-quality web app when given only a high-level prompt such as "build a clone of claude.ai".
- Two failure patterns:
  1. **Doing too much at once**, trying to one-shot the app. The agent runs out of context mid-implementation, leaving the next session a half-built, undocumented feature. The next agent has to guess what happened and spends much of its time getting the basic app working again. This happens even with compaction, which does not always hand clear instructions to the next session.
  2. **Declaring victory too early**, later in the project: an agent looks around, sees that progress has been made, and decides the job is done.
- The decomposition: first, set up an initial environment that lays the foundation for every feature the prompt requires, so the agent can work step by step and feature by feature; second, prompt each agent to make incremental progress and leave the environment in a **clean state**, meaning code fit to merge to main: no major bugs, orderly and well documented, so a developer could start a new feature without cleaning up an unrelated mess.
- The two-part solution:
  - **Initializer agent**: the very first session uses a specialized prompt that sets up the environment: an init.sh script, a claude-progress.txt file that logs what agents have done, and an initial git commit showing which files were added.
  - **Coding agent**: every subsequent session makes incremental progress and leaves structured updates. (Footnote: they are called separate agents only because their initial user prompts differ; system prompt, tools and harness are identical.)
- The key insight: an agent with a fresh context must be able to understand the state of work quickly, which claude-progress.txt plus the git history provides. The practices are borrowed from what effective software engineers do every day.

## 2. Environment management

Anthropic's Claude 4 prompting guide already recommends, for multi-context-window workflows, "a different prompt for the very first context window". That prompt asks the initializer agent to set up everything future coding agents need. The key components:

### Feature list

- To stop the agent from one-shotting the app or declaring it complete, the initializer writes a comprehensive file of feature requirements that expands the user's prompt. For the claude.ai clone this meant **over 200 features**, such as "a user can open a new chat, type in a query, press enter, and see an AI response". All start marked as failing, so later agents have a clear picture of full functionality.
- Each entry is a JSON object with a category, a description, verification steps and a passes flag. The post's example: category "functional", description "New chat button creates a fresh conversation", steps from navigating to the main interface through verifying the conversation appears in the sidebar, passes: false.
- Coding agents may edit the file only by changing the passes field, backed by strongly worded instructions: "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality." **JSON was chosen over Markdown** because the model is less likely to inappropriately change or overwrite JSON files.

### Incremental progress

- With the scaffolding in place, the coding agent is asked to work on **one feature at a time**. This was critical to curbing the tendency to do too much at once.
- It must still leave the environment clean after each change. The best way found was to have the model **commit to git with descriptive messages** and write summaries to the progress file. Git lets the model revert bad changes and recover working states.
- These practices also improved efficiency: no more guessing what happened and rebuilding the basic app.

### Testing

- Another major failure mode: marking features complete without proper testing. Left to itself, Claude would change code and even run unit tests or curl against a development server, yet fail to notice that the feature did not work end to end.
- Once explicitly prompted to use browser automation tools and test as a human user would, Claude mostly verified features end to end well (the post shows screenshots Claude took through the Puppeteer MCP server while testing the clone). Giving the agent these testing tools **dramatically improved performance**, because it could find and fix bugs not visible from the code alone.
- Limits remain: Claude's vision and the browser automation tools cannot catch every bug. For example, Claude cannot see browser-native alert modals through the Puppeteer MCP, so features depending on them tended to be buggier.

## 3. Getting up to speed: the start of every session

Every coding agent is prompted to run through a set of steps, some basic but still helpful:

- Run pwd to see the working directory; only files there can be edited.
- Read the git log and the progress file to learn what was recently worked on.
- Read the feature list and choose the highest-priority feature not yet done.
- The initializer also writes **init.sh** to start the development server, and the coding agent runs a basic end-to-end test **before** implementing anything new. For the clone this meant starting the server and using the Puppeteer MCP to open a new chat, send a message and receive a response. A broken app is caught and fixed immediately; starting a new feature on top of it would only make things worse. This also saves tokens every session, because the agent does not have to work out how to test the code.
- A typical session opening (quoted in the post): get bearings with pwd, read claude-progress.txt and feature_list.json; check `git log --oneline -20`; check for init.sh and start the servers; navigate to the app and verify fundamental features still work (core chat, theme switching, conversation loading, error handling); review the feature file; start on a new feature.

## 4. Failure modes and solutions (the post's table)

| Problem | Initializer agent behavior | Coding agent behavior |
| --- | --- | --- |
| Claude declares victory on the whole project too early | Set up a feature list file: a structured JSON list of end-to-end feature descriptions based on the input spec | Read the feature list at the start of a session; choose a single feature to work on |
| Claude leaves the environment with bugs or undocumented progress | Create the initial git repo and progress notes file | Start by reading the progress notes and git log and run a basic test on the dev server to catch undocumented bugs; end with a git commit and a progress update |
| Claude marks features as done prematurely | Set up the feature list file | Self-verify every feature; mark it passing only after careful testing |
| Claude spends time figuring out how to run the app | Write an init.sh script that runs the development server | Start the session by reading init.sh |

## 5. Future work

- Open question: does a single general-purpose coding agent perform best, or would a multi-agent architecture with specialized testing, quality-assurance or code-cleanup agents do better on sub-tasks of the development lifecycle?
- The demo is tuned for full-stack web app development. Generalizing the lessons to other long-running agentic work, such as scientific research or financial modeling, is a future direction.

## Quick reference: the harness in one list

- **First session (initializer)**: expand the prompt into a JSON feature list with every feature marked failing; write init.sh; create claude-progress.txt; make the initial git commit.
- **Every later session (coding agent)**: pwd, read the progress file and git log, run init.sh and a basic end-to-end check, fix anything broken, pick one failing feature, implement it, test it end to end through browser automation, flip its passes flag, commit with a descriptive message, update the progress file.
- **Rules**: never remove or edit feature entries except the passes field; leave the code in a mergeable state at the end of every session.

## Changes between the November 2025 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2025-11-27 (original URL under /engineering/) shows the body identical word for word. The differences are page chrome: the article moved from /engineering/ to /research/, the standfirst ("Agents still face challenges working across many context windows. We looked to human engineers for inspiration in creating a more effective harness for long-running agents.") no longer appears, and the footnote is formatted differently.
