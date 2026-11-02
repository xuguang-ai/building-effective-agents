# Demystifying evals for AI agents — reading summary

| Item | Detail |
| --- | --- |
| Source | Anthropic, "Demystifying evals for AI agents" |
| URL | https://www.anthropic.com/research/demystifying-evals-for-ai-agents (first published under /engineering/) |
| Authors | Mikaela Grace, Jeremy Hadfield, Rodrigo Olivares and Jiri De Jonghe, with contributions from David Hershey, Gian Segato, Mike Merrill, Alex Shaw, Nicholas Carlini, Ethan Dixon, Pedram Navid, Jake Eaton, Alyssa Baum, Lina Tawfik, Karen Zhou, Alexander Bricken, Sam Kennedy, Robert Ying and others; thanks to customers including iGent, Cognition, Bolt, Sierra, Vals.ai, Macroscope, PromptLayer, Stripe, Shopify and the Terminal-Bench team |
| Published | 2026-01-09 |
| Accessed | 2026-11-02 (current page), compared paragraph by paragraph with the Wayback Machine snapshot of 2026-01-10; differences listed at the end |

## One-sentence takeaway

Evals turn "the agent feels worse" into something measurable: start early with 20 to 50 realistic tasks drawn from real failures, grade what the agent produced rather than the path it took, combine code, model and human graders, run several trials and report pass@k or pass^k as the product demands, read the transcripts, and keep the suite alive as one layer of quality control among several.

## 1. Introduction

- Good evals let teams ship agents with confidence. Without them, teams get stuck in reactive loops, catching issues only in production, where fixing one failure creates others. Evals make problems and behavior changes visible before users see them, and their value compounds over the agent's lifecycle.
- Agents operate over many turns, calling tools, modifying state and adapting to intermediate results. The same capabilities that make them useful (autonomy, intelligence, flexibility) make them harder to evaluate.

## 2. The structure of an evaluation

- An eval gives an AI system an input and applies grading logic to its output. The post covers automated evals that run during development without real users.
- **Single-turn evals** are a prompt, a response and grading logic (the diagram's example: "How many toes does a {ANIMAL} have?" with ANIMAL = Cat, graded by response == 18). They were the main method for earlier LLMs; multi-turn evals are now common.
- **Agent evals** are more complex. In the diagram a coding agent gets tools (web search, file edit, a database MCP tool), an environment (Python installed, dev environment set up, internet access, sandboxed) and a task ("Write an MCP server to connect to my app"), runs an agent loop of tool calls and reasoning that updates the environment, and is graded by running tests that check the MCP server actually works. Mistakes can propagate and compound across turns. Frontier models also find creative solutions that outrun static evals: Opus 4.5 solved a τ²-bench flight-booking problem by discovering a loophole in the policy, "failing" the eval as written while giving the user a better solution.
- **Definitions used throughout**:
  - Task (problem, test case): one test with defined inputs and success criteria.
  - Trial: one attempt at a task. Outputs vary between runs, so several trials give more consistent results.
  - Grader: logic that scores one aspect of performance. A task can have several graders, each with several assertions (checks).
  - Transcript (trace, trajectory): the complete record of a trial: outputs, tool calls, reasoning, intermediate results. For the Anthropic API it is the full messages array at the end of the run.
  - Outcome: the final state of the environment. The agent may say "Your flight has been booked"; the outcome is whether a reservation exists in the environment's SQL database.
  - Evaluation harness: the infrastructure that runs evals end to end: provides instructions and tools, runs tasks concurrently, records every step, grades outputs, aggregates results.
  - Agent harness (scaffold): the system that lets a model act as an agent: processes inputs, orchestrates tool calls, returns results. Evaluating "an agent" evaluates harness and model together. Claude Code is a flexible agent harness, and its primitives, through the Agent SDK, built Anthropic's long-running agent harness.
  - Evaluation suite: a collection of tasks measuring specific capabilities or behaviors with a shared broad goal, for example a customer support suite covering refunds, cancellations and escalations.
- The components diagram nests them: an evaluation harness contains an evaluation suite of tasks; each task carries graders (deterministic_tests, llm_rubric, state_check, tool_calls), tracked metrics (n_turns, n_toolcalls, tokens, latency) and several trials, each with a trajectory; the suite talks to the agent harness; each trial produces an outcome, and graders score trajectory plus outcome.

## 3. Why build evaluations?

- Early on, manual testing, dogfooding and intuition get teams surprisingly far, and rigorous evaluation looks like overhead. Once the agent is in production and scaling, building without evals breaks down.
- The breaking point: users report the agent feels worse after a change and the team is "flying blind". Debugging becomes reactive: wait for complaints, reproduce manually, fix, hope nothing else regressed. Teams cannot separate real regressions from noise, test changes against hundreds of scenarios before shipping, or measure improvements.
- **Claude Code's progression**: fast iteration on feedback from Anthropic employees and external users; then evals for narrow areas (concision, file edits); then complex behaviors such as over-engineering. Evals identified issues, guided improvements and focused research-product collaboration, alongside production monitoring, A/B tests and user research.
- **Customer examples**: Descript's video-editing agent has evals on three dimensions (don't break things, do what I asked, do it well); it moved from manual grading to LLM graders with product-team criteria and periodic human calibration, and now runs two suites, quality benchmarking and regression testing. Bolt started later, with a widely used agent, and in three months built a system that runs the agent, grades with static analysis, tests apps with browser agents, and uses LLM judges for behaviors such as instruction following.
- Evals are useful at any stage. Early they force product teams to specify what success means (two engineers reading the same spec can interpret edge cases differently; an eval suite resolves that). Later they uphold a consistent quality bar.
- Evals decide how fast you can adopt a new model: teams without them face weeks of testing while competitors with evals find the model's strengths, tune prompts and upgrade in days.
- Once evals exist, baselines and regression tests come free: latency, token usage, cost per task and error rates tracked on a static bank of tasks. Evals also become the highest-bandwidth channel between product and research teams. Costs are visible up front; benefits accumulate later.

## 4. How to evaluate AI agents

Common agent types (coding, research, computer use, conversational) can be evaluated with similar techniques; use the proven methods below as a foundation and extend them to your domain.

### 4.1 Three types of graders

| Grader type | Methods | Strengths | Weaknesses |
| --- | --- | --- | --- |
| Code-based | string match (exact, regex, fuzzy); binary tests (fail-to-pass, pass-to-pass); static analysis (lint, type, security); outcome verification; tool-call verification (tools used, parameters); transcript analysis (turns, tokens) | fast, cheap, objective, reproducible, easy to debug, verifies specific conditions | brittle to valid variations; lacks nuance; limited for subjective tasks |
| Model-based | rubric scoring; natural-language assertions; pairwise comparison; reference-based evaluation; multi-judge consensus | flexible, scalable, captures nuance, handles open-ended tasks and freeform output | non-deterministic; more expensive than code; needs calibration against human graders |
| Human | subject-matter-expert review; crowdsourced judgment; spot-check sampling; A/B testing; inter-annotator agreement | gold-standard quality; matches expert judgment; calibrates model graders | expensive; slow; needs experts at scale |

- Each grader scores part of the transcript or the outcome. Per task, scoring can be weighted (combined score must clear a threshold), binary (every grader must pass) or hybrid.

### 4.2 Capability vs. regression evals

- **Capability (quality) evals** ask "What can this agent do well?" They start at a low pass rate, target what the agent struggles with, and give the team a hill to climb.
- **Regression evals** ask "Does the agent still handle everything it used to?" and should sit near 100%. A drop means something broke. Run them alongside capability evals so improvements do not break something else.
- Capability evals with high pass rates "graduate" into the regression suite, run continuously to catch drift: "Can we do this at all?" becomes "Can we still do this reliably?"

### 4.3 Coding agents

- Rely on well-specified tasks, stable test environments and thorough tests. Deterministic graders are natural: does the code run, do the tests pass. SWE-bench Verified gives agents GitHub issues from popular Python repositories and passes a solution only if it fixes the failing tests without breaking existing ones; LLMs went from 40% to over 80% in one year. Terminal-Bench tests end-to-end technical tasks such as building a Linux kernel from source or training an ML model.
- Beyond pass/fail, grade the transcript: heuristic code-quality rules, and model graders with clear rubrics for how the agent calls tools or interacts with the user.
- **Illustrative YAML task** "fix-auth-bypass_1" (fix an authentication bypass when the password field is empty): graders deterministic_tests (required test files for empty and null passwords), llm_rubric (prompts/code_quality.md), static_analysis (ruff, mypy, bandit), state_check (security_logs contain an auth_blocked event) and tool_calls (read_file on src/auth/*, edit_file, run_tests); tracked metrics n_turns, n_toolcalls, n_total_tokens, time_to_first_token, output_tokens_per_sec, time_to_last_token. In practice coding evals mostly use unit tests for correctness plus an LLM rubric for code quality, adding other graders only as needed.

### 4.4 Conversational agents

- Support, sales or coaching agents keep state, use tools and act mid-conversation, and the quality of the interaction is itself under evaluation. They usually need a second LLM to simulate the user; Anthropic's alignment auditing agents use this to stress-test models in extended adversarial conversations.
- Success is multidimensional: ticket resolved (state check), finished in under 10 turns (transcript constraint), appropriate tone (LLM rubric). τ-Bench and its successor τ²-Bench simulate multi-turn retail support and airline booking with one model playing a user persona.
- **Illustrative refund task**: llm_rubric (prompts/support_quality.md) with assertions that the agent showed empathy, explained the resolution clearly and grounded its response in fetch_policy results; state_check (ticket resolved, refund processed); tool_calls (verify_identity, process_refund with amount <= 100, send_confirmation); transcript max_turns 10; the same tracked metrics. In practice conversational evals lean on model-based graders for communication quality and goal completion, because many tasks have several correct answers.

### 4.5 Research agents

- Research quality can only be judged relative to the task: a market scan, acquisition due diligence and a scientific report need different standards. Experts disagree on comprehensiveness, ground truth shifts as reference content changes, and long open-ended outputs leave more room for error. BrowseComp tests finding needles in haystacks across the open web: easy to verify, hard to solve.
- Combine graders: groundedness checks (claims supported by retrieved sources), coverage checks (key facts a good answer must include), source-quality checks (authoritative sources rather than the first retrieved), exact match for objective answers ("What was Company X's Q3 revenue?"), and an LLM to flag unsupported claims and gaps and verify coherence and completeness. Calibrate LLM rubrics against expert human judgment frequently.

### 4.6 Computer use agents

- They work through screenshots, clicks, keyboard input and scrolling, so they can use any GUI application. Evaluate in a real or sandboxed environment and check the intended outcome. WebArena uses URL and page-state checks plus backend verification for data-changing tasks (the order was placed, not just the confirmation page shown). OSWorld extends to full operating-system control, inspecting file system state, application configs, database contents and UI element properties.
- Browser agents balance token efficiency and latency: DOM-based interaction is fast but token-heavy, screenshots are slower but token-efficient. Summarizing Wikipedia is cheaper from the DOM; shopping for a laptop case on Amazon is cheaper with screenshots. For Claude for Chrome, Anthropic built evals that check the agent picks the right tool for each context, making tasks faster and more accurate.

### 4.7 Non-determinism: pass@k and pass^k

- Behavior varies between runs. Each task has its own success rate, and a task that passed one run may fail the next, so often the thing to measure is how often an agent succeeds.
- **pass@k**: the probability of at least one success in k attempts. It rises with k. 50% pass@1 means the agent succeeds on half the tasks at the first try. Coding usually cares about pass@1; elsewhere proposing many solutions is fine if one works.
- **pass^k**: the probability that all k trials succeed. It falls with k: a 75% per-trial rate over 3 trials gives 0.75³ ≈ 42%. It matters for customer-facing agents that must behave reliably every time.
- The chart: at k = 1 both equal the per-trial rate (about 73% in the example); with 3 trials the example agent reaches 97% pass@3 but only 39% pass^3; by k = 10 pass@k approaches 100% while pass^k approaches 0%. Choose by product requirement: pass@k where one success matters, pass^k where consistency is essential.

## 5. From zero to one: a roadmap to great evals

The roadmap diagram groups the steps into evaluation suite development (0 to 3), harness development (4 to 5) and eval maintenance (6 to 8).

### Collect tasks for the initial dataset

- **Step 0, start early**. Teams delay because they think they need hundreds of tasks; 20 to 50 simple tasks drawn from real failures is a great start. Early changes have large, obvious effects, so small samples suffice; mature agents need larger, harder evals to detect smaller effects. Evals get harder to build the longer you wait: early on, product requirements translate into test cases; later you reverse-engineer success criteria from a live system.
- **Step 1, start with what you already test manually**: the checks run before each release and the common tasks end users try. In production, mine the bug tracker and support queue; converting user-reported failures into tasks keeps the suite tied to real usage, and prioritizing by user impact focuses effort.
- **Step 2, write unambiguous tasks with reference solutions**. A good task is one where two domain experts would independently reach the same pass/fail verdict; if they could not pass it themselves, refine it. Ambiguity in tasks becomes noise in metrics, and vague rubrics produce inconsistent judgments. Every task must be passable by an agent that follows instructions: an audit of Terminal-Bench found tasks that asked for a script without specifying the file path the tests assumed. Everything the grader checks must be clear from the task description. With frontier models, 0% pass@100 almost always signals a broken task, not an incapable agent. Create a reference solution that passes all graders to prove the task is solvable and the graders configured correctly.
- **Step 3, build balanced problem sets**. Test both where a behavior should occur and where it should not; one-sided evals create one-sided optimization (test only "searches when it should" and you get an agent that searches for everything). Avoid class imbalance. For web search in Claude.ai the team built evals in both directions, queries that should search (the weather) and queries that should not ("who founded Apple?"), and balancing undertriggering against overtriggering took many rounds of prompt and eval refinement; new examples keep being added.

### Design the eval harness and graders

- **Step 4, build a robust harness with a stable environment**. The agent under evaluation should behave roughly like the production agent, and the environment must not add noise. Isolate each trial by starting from a clean environment: leftover files, cached data or resource exhaustion cause correlated failures that look like agent failures, and shared state can inflate scores (in some internal evals Claude gained an unfair advantage by reading the git history of previous trials). Trials that fail from the same environment limit, such as limited CPU memory, are not independent, and the results stop measuring the agent.
- **Step 5, design graders thoughtfully**. Deterministic graders where possible, LLM graders where necessary or for flexibility, human graders judiciously for validation. Do not check for a specific sequence of tool calls; agents regularly find valid approaches the designers did not anticipate, so grade what the agent produced, not the path it took. Build in partial credit: a support agent that identifies the problem and verifies the customer but fails to process the refund is meaningfully better than one that fails immediately. Calibrate LLM-as-judge graders closely with human experts, give the judge a way out (return "Unknown" when information is insufficient), use structured rubrics per dimension with an isolated judge for each dimension, and fall back to occasional human review once the system is robust. Watch for subtle failure modes that depress scores despite good performance: Opus 4.5 initially scored 42% on CORE-Bench until an Anthropic researcher found rigid grading (penalizing "96.12" when "96.124991…" was expected), ambiguous specs and stochastic tasks; after fixes and a less constrained scaffold it scored 95%. METR found misconfigured tasks in its time-horizon benchmark that asked agents to reach a stated threshold while the grading required exceeding it, penalizing models such as Claude for following instructions. Make graders resistant to bypasses so passing requires solving the problem rather than exploiting loopholes.

### Maintain and use the eval long-term

- **Step 6, check the transcripts**. You cannot know the graders work until you read transcripts and grades from many trials; Anthropic built tooling for viewing eval transcripts and reads them regularly. A transcript shows whether the agent erred or the grader rejected a valid solution. Failures should seem fair, and when scores stall you need confidence the eval, not the agent, is not the problem. Reading transcripts is a critical skill for agent development.
- **Step 7, monitor for saturation**. An eval at 100% tracks regressions but offers no improvement signal. SWE-bench Verified started the year at 30% and frontier models are now nearing saturation above 80%; as only the hardest tasks remain, large capability gains look like small score increases. The code-review startup Qodo was initially unimpressed by Opus 4.5 because its one-shot coding evals missed the gains on longer, more complex tasks; a new agentic eval framework gave a clearer picture. Do not take scores at face value until someone digs into the eval and reads some transcripts; revise evals whose grading is unfair, tasks ambiguous, valid solutions penalized or model constrained by the harness.
- **Step 8, keep suites healthy through open contribution and maintenance**. An eval suite is a living artifact with clear ownership. At Anthropic the most effective arrangement was dedicated evals teams owning the core infrastructure while domain experts and product teams contribute most tasks and run the evaluations. Owning evals should be as routine as maintaining unit tests, and defining eval tasks is one of the best ways to test whether product requirements are concrete enough to build. Practice eval-driven development: build evals for planned capabilities before the agent can meet them, then iterate; capability evals that start low make bets on future models visible, and when a new model ships the suite shows which bets paid off. The people closest to requirements and users should define success: product managers, customer success managers or salespeople can contribute an eval task as a pull request with Claude Code.

## 6. How evals fit with other methods

| Method | What it is | Pros | Cons |
| --- | --- | --- | --- |
| Automated evals | programmatic tests without real users | fast iteration; reproducible; no user impact; runs on every commit; tests scenarios at scale without a production deployment | up-front investment; ongoing maintenance to avoid drift; false confidence if it does not match real usage |
| Production monitoring | metrics and errors in live systems | real user behavior at scale; catches what synthetic evals miss; ground truth on actual performance | reactive, problems reach users first; noisy; needs instrumentation; no ground truth for grading |
| A/B testing | comparing variants on real traffic | measures real outcomes (retention, task completion); controls for confounds; scalable | slow, needs traffic and days or weeks for significance; only tests what you deploy; little signal on the "why" without transcript review |
| User feedback | thumbs-down, bug reports | surfaces unanticipated problems; real examples; correlates with product goals | sparse and self-selected; skews to severe issues; users rarely explain; not automated; relying on it harms users |
| Manual transcript review | humans reading agent conversations | builds intuition for failure modes; catches subtle issues; calibrates what "good" looks like | time-intensive; does not scale; inconsistent coverage; reviewer fatigue; mostly qualitative |
| Systematic human studies | structured grading by trained raters | gold-standard judgments from several raters; handles subjective tasks; improves model graders | expensive and slow; hard to run often; inter-rater disagreement; complex domains (legal, finance, healthcare) need experts |

- **Mapping to stages**: automated evals pre-launch and in CI/CD on every change and model upgrade; production monitoring post-launch for distribution drift and unanticipated failures; A/B tests for significant changes once traffic suffices; user feedback triaged constantly and transcripts sampled weekly; systematic human studies reserved for calibrating LLM graders or judging subjective outputs.
- **Swiss cheese model**: no single layer catches every issue. In the figure, automated evals provide consistent measurement, baselines and regression catching before deployment; manual transcript review and early-access programs catch nuanced failures and unexpected behavior; production monitoring, A/B testing and user feedback surface rare edge cases and real usage at scale. The best teams combine automated evals for fast iteration, production monitoring for ground truth and periodic human review for calibration.

## 7. Conclusion

- Teams without evals fix one failure and create another, unable to tell regressions from noise. Teams that invest early accelerate: failures become test cases, test cases prevent regressions, metrics replace guesswork, and "the agent feels worse" becomes actionable. The value compounds only if evals are a core component, not an afterthought.
- The fundamentals: start early and do not wait for the perfect suite; source realistic tasks from observed failures; define unambiguous, robust success criteria; design graders thoughtfully and combine types; make the problems hard enough; iterate on the evals' signal-to-noise ratio; read the transcripts.
- Agent evaluation is nascent; longer tasks, multi-agent systems and subjective work will require adapted techniques.

## Appendix: eval frameworks named in the post

- **Harbor**: runs agents in containerized environments with infrastructure for trials at scale across cloud providers and a standardized task and grader format; Terminal-Bench 2.0 ships through the Harbor registry.
- **Braintrust**: offline evaluation plus production observability and experiment tracking; its autoevals library has pre-built scorers for factuality, relevance and other dimensions.
- **LangSmith**: tracing, offline and online evaluation and dataset management integrated with LangChain. **Langfuse**: similar, self-hosted open source for data-residency needs.
- **Arize**: Phoenix, an open-source platform for tracing, debugging and evaluation, and AX, a SaaS extension for scale and monitoring.
- Many teams combine tools, build their own, or start with simple scripts. Frameworks are only as good as the tasks run through them: pick one that fits quickly, then invest in high-quality tasks and graders.

## Quick reference

| Agent type | Typical graders | Benchmarks named | Metric note |
| --- | --- | --- | --- |
| Coding | unit tests (fail-to-pass, pass-to-pass), static analysis, LLM rubric for code quality | SWE-bench Verified, Terminal-Bench | pass@1 is usually what matters |
| Conversational | LLM rubric for interaction quality, state checks, tool-call checks, turn limits, simulated user | τ-Bench, τ²-Bench | many valid answers, so model graders dominate |
| Research | groundedness, coverage, source quality, exact match for facts, LLM for coherence | BrowseComp | calibrate rubrics against experts often |
| Computer use | outcome and state checks in a sandbox, URL and page state, file system and database inspection | WebArena, OSWorld | balance DOM access against screenshots |

The roadmap in one line per step: 0 start early with 20 to 50 tasks; 1 convert manual checks and user-reported failures; 2 unambiguous tasks with reference solutions; 3 positive and negative cases; 4 isolated, production-like harness; 5 outcome-focused graders with partial credit and calibrated LLM judges; 6 read transcripts; 7 watch for saturation; 8 shared ownership and eval-driven development.

## Changes between the January 2026 original and the current page

A paragraph-by-paragraph comparison with the Wayback Machine snapshot of 2026-01-10 (original URL under /engineering/) shows the same structure and content with two kinds of edits:

- About two dozen paragraphs received copy edits only: punctuation, capitalization, hyphenation and quotation marks (for example "flying blind" in straight quotes, "widely-used" to "widely used", "upfront" to "up-front").
- The appendix changed: the paragraph on Promptfoo ("a lightweight, flexible, and open-source framework that focuses on declarative YAML configuration for prompt testing", which the original said Anthropic uses for many product evals) was removed, and the paragraph on Arize (Phoenix and AX) was added.
- Page chrome: the article moved from /engineering/ to /research/, and the standfirst ("The capabilities that make agents useful also make them difficult to evaluate. The strategies that work across deployments combine techniques to match the complexity of the systems they measure.") no longer appears.
