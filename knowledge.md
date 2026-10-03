# CCAR-F Knowledge Base

Source of truth for my daily quiz. Generate questions ONLY from facts in this file.
If a topic has no facts here yet, skip it — do not invent details.

Last updated: 2026-10-01
Source: Official "Claude Certified Architect – Foundations Exam Guide" (v1.0, effective July 2026), full PDF obtained directly from the user on 2026-10-01 — includes Section 9 "Sample Questions" (12 official Q&A with explanations) and full Task Statement detail per domain, cross-checked against this file's existing content. Content matched closely; additions below are concrete examples/numbers from the official guide that weren't previously captured (e.g. specific tool-rename examples, the 85%/15% verify_fact split, batch SLA math, tool_choice:any for unknown document types).
Cross-checked 2026-09-29 against a friend's copy of an earlier draft (v0.2, June 2026) — content (domains, task statements, scenarios, sample questions) matched almost verbatim. One conflict found: v0.2 states strictly single-answer multiple-choice (4 options, 1 correct); v1.0 states multiple-choice AND multiple-response. Decision: trust v1.0 (fetched live from the current official page) — quiz keeps occasional multi-response questions per Question Generation Rules.

---

## Exam Overview

- Exam: Claude Certified Architect – Foundations (CCAR-F)
- Format: proctored, 60 questions, 120 minutes
- Item format: multiple-choice AND multiple-response (some questions require selecting more than one answer — the item states how many to select)
- Exam structure: 4 scenarios presented per exam, drawn at random from a bank of 6 scenarios (see "Exam Scenarios" below)
- Passing score: scaled 720 on a 100–1,000 scale
- Domains and weights:

| # | Domain | Weight |
|---|--------|--------|
| D1 | Agentic Architecture & Orchestration | 27% |
| D2 | Tool Design & MCP Integration | 18% |
| D3 | Claude Code Configuration & Workflows | 20% |
| D4 | Prompt Engineering & Structured Output | 20% |
| D5 | Context Management & Reliability | 15% |

Note: official domain numbering is D1, D2 (Tool/MCP), D3 (Claude Code), D4 (Prompt), D5 (Context) — D2 and D3 order differs from an earlier draft of this file; weights unchanged.

---

## Exam Scenarios (official — 4 of these 6 appear per exam)

Real exam questions are framed inside one of these production contexts. Quiz scenarios should draw from or resemble these rather than generic setups.

1. **Customer Support Resolution Agent** — Agent SDK agent handling returns/billing/account issues via custom MCP tools: `get_customer`, `lookup_order`, `process_refund`, `escalate_to_human`. Target: 80%+ first-contact resolution, correct escalation. Primary domains: D1, D2, D5.
2. **Code Generation with Claude Code** — team uses Claude Code for codegen/refactor/debug/docs; needs custom slash commands, CLAUDE.md config, plan mode vs direct execution judgment. Primary domains: D3, D5.
3. **Multi-Agent Research System** — coordinator agent delegates to specialized subagents (web search, document analysis, synthesis, report generation); produces cited reports. Primary domains: D1, D2, D5.
4. **Developer Productivity with Claude** — agent helps explore unfamiliar codebases using built-in tools (Read, Write, Bash, Grep, Glob) + MCP servers. Primary domains: D2, D3, D1.
5. **Claude Code for CI/CD** — Claude Code runs automated code review, test generation, PR feedback in a pipeline; needs actionable feedback with minimal false positives. Primary domains: D3, D4.
6. **Structured Data Extraction** — extracts info from unstructured documents, validates via JSON schemas, handles edge cases, feeds downstream systems. Primary domains: D4, D5.

---

## Cross-Domain Theme: Advisory vs Deterministic Enforcement

This is my #1 weak area. Include at least one question on it every few days.

- Text in context is ALWAYS advisory, no matter how strongly worded:
  - CLAUDE.md instructions
  - System prompts
  - Code comments
  - "NEVER do X" / "MUST do Y" in prompts
  - Few-shot examples showing desired behavior (still advisory — they raise the odds, not a guarantee)
- The model can still deviate from advisory text.
- Only code-level mechanisms are deterministic, because they act outside the model's discretion:
  - Hooks (e.g. PreToolUse with exit-code logic that blocks an action; PostToolUse for logging/normalization)
  - Programmatic prerequisite gates (e.g. blocking `process_refund` until `get_customer` has returned a verified ID)
  - Server-side / schema validation
  - Permission settings enforced by the harness (`allowedTools`)
- Exam pattern: "A team wants to GUARANTEE the agent never does X — which approach?"
  → The answer is a deterministic mechanism, not a stronger prompt, more few-shot examples, or a routing classifier.
- Official exam example (Customer Support scenario): agent skips `get_customer` 12% of the time before calling `lookup_order`, causing misidentified accounts. Correct fix = a programmatic prerequisite blocking `lookup_order`/`process_refund` until `get_customer` returns a verified customer ID — NOT a stronger system prompt or few-shot examples.

---

## D1 — Agentic Architecture & Orchestration (27%)

### The agentic loop
- Lifecycle: send request to Claude → inspect `stop_reason` → if `"tool_use"`, execute requested tool(s) and return results for the next iteration; if `"end_turn"`, the loop terminates and the response is final.
- Tool results are appended to conversation history so the model can reason about the next action using that new information.
- Model-driven decision-making (Claude decides which tool to call next based on context) is distinct from pre-configured decision trees or fixed tool sequences.
- Anti-patterns to avoid: parsing natural-language signals to decide when to stop, setting an arbitrary iteration cap as the *primary* stopping mechanism, or checking for assistant text content as a completion signal — the correct signal is `stop_reason`.
- It is NOT a multi-agent review pattern (I got this wrong before) — a single agent looping is still just the agentic loop.

### Multi-agent orchestration (coordinator-subagent)
- Hub-and-spoke: a coordinator agent manages ALL inter-subagent communication, error handling, and information routing — subagents do not talk to each other directly.
- Subagents have **isolated context** — they do NOT automatically inherit the coordinator's conversation history. Context must be explicitly included in the subagent's prompt (e.g. passing prior findings directly in the prompt text).
- Coordinator responsibilities: task decomposition, delegation, result aggregation, and dynamically deciding which subagents to invoke based on query complexity (not always routing through the full pipeline).
- Risk: overly narrow task decomposition by the coordinator → incomplete coverage of a broad topic (e.g. decomposing "AI in creative industries" into only visual-arts subtasks, missing music/writing/film — the root cause is the coordinator's decomposition, not the subagents, which executed correctly).
- Iterative refinement: coordinator evaluates synthesis output for gaps, re-delegates to search/analysis subagents with targeted queries, re-invokes synthesis until coverage is sufficient.

### Subagent invocation, context passing, spawning
- The `Task` tool is the mechanism for spawning subagents; the coordinator's `allowedTools` configuration must include `"Task"` for it to invoke subagents at all.
- `AgentDefinition` configures each subagent type: description, system prompt, tool restrictions.
- Parallel subagents: emit multiple `Task` tool calls in a **single coordinator response** (not across separate turns) to run them in parallel and reduce latency.
- Pass complete findings directly in the subagent's prompt (not relying on inherited memory); use structured data formats separating content from metadata (source URLs, doc names, page numbers) to preserve attribution across handoffs.
- Coordinator prompts should specify research goals and quality criteria, not step-by-step procedural instructions — this lets subagents adapt.
- `fork_session` creates an independent branch from a shared analysis baseline, for exploring divergent approaches without recomputing the shared context.

### Multi-step workflows: enforcement and handoff
- Programmatic enforcement (hooks, prerequisite gates) vs prompt-based guidance for workflow ordering — prompt instructions alone have a non-zero failure rate, unacceptable when deterministic compliance is required (e.g. identity verification before a financial operation).
- Programmatic prerequisites block downstream tool calls until prerequisite steps complete (e.g. blocking `process_refund` until `get_customer` returns a verified ID).
- Multi-concern requests: decompose into distinct items, investigate each in parallel using shared context, then synthesize one unified resolution.
- Structured handoff to a human: include customer details, root cause analysis, and recommended action — the human escalatee lacks the full conversation transcript.

### Agent SDK hooks (D1 angle)
- `PostToolUse` hooks can intercept tool RESULTS to normalize heterogeneous data (e.g. Unix timestamps vs ISO 8601 vs numeric status codes from different MCP tools) before the model processes them.
- Hooks can also intercept OUTGOING tool calls to enforce compliance rules (e.g. blocking a refund tool call above a $ threshold) and redirect to an alternative workflow (e.g. human escalation).
- Choose hooks over prompt-based enforcement whenever business rules require guaranteed (deterministic) compliance — connects to the cross-domain advisory-vs-deterministic theme.

### Task decomposition strategies
- Fixed sequential pipelines ("prompt chaining": e.g. analyze each file individually, then a separate cross-file integration pass) vs dynamic adaptive decomposition that generates subtasks based on what's discovered along the way.
- Use prompt chaining for predictable, multi-aspect reviews; use dynamic/adaptive decomposition for open-ended investigation tasks.
- Large code reviews: split into per-file local analysis passes + one separate cross-file integration pass — avoids "attention dilution" (inconsistent depth, contradictory findings across files in one big pass).
- Open-ended tasks (e.g. "add comprehensive tests to a legacy codebase"): first map structure, identify high-impact areas, then build a prioritized plan that adapts as dependencies are discovered.

### Session state, resumption, forking
- `--resume <session-name>` continues a specific named prior conversation.
- `fork_session` creates independent branches from a shared baseline to explore divergent approaches (e.g. comparing two refactoring strategies).
- When resuming after code changes, explicitly inform the agent which files changed — don't assume it re-discovers this.
- Starting a NEW session with a structured summary is often more reliable than resuming a session whose tool results have gone stale (e.g. after significant file changes).

### Claude Managed Agents (background, still relevant)
- Core concepts: agents, environments, sessions, events.
- Rubrics and graders evaluate outcomes — connects to advisory vs deterministic: automated outcome evaluation, not "the agent self-checks," is the more reliable pattern.
- Parallel sessions can run at the same time.

### Design decisions
- Where state is stored and when it is read/written are deliberate design decisions, not defaults.

---

## D2 — Tool Design & MCP Integration (18%)

### Effective tool interfaces
- The tool **description** is the PRIMARY mechanism an LLM uses for tool selection — minimal descriptions ("Retrieves customer information") cause unreliable selection among similar tools.
- Descriptions should include: input formats, example queries, edge cases, and explicit boundaries ("use this vs. X when...").
- Ambiguous/overlapping descriptions cause misrouting (e.g. `analyze_content` vs `analyze_document` with near-identical wording).
- System prompt wording can also bias tool selection via keyword sensitivity — unintended associations can override even well-written tool descriptions.
- Fixes, in order of effort: (1) improve/expand the tool description first — lowest effort, addresses root cause; only then consider (2) renaming tools to remove functional overlap, or (3) splitting one generic tool into purpose-specific tools with defined input/output contracts. Few-shot examples and routing/classifier layers are NOT the first-line fix for weak descriptions.
- Official exam concrete examples of fixes 2–3: renaming `analyze_content` to `extract_web_results` with a web-specific description; splitting a generic `analyze_document` into `extract_data_points`, `summarize_content`, and `verify_claim_against_source`; replacing a generic `fetch_url` with a constrained `load_document` that validates document URLs.
- Official exam "proportionate first step" pattern: when a problem can be solved by a low-effort fix (expanding descriptions), a higher-effort but still-valid fix (consolidating/splitting tools, adding a routing layer) is the WRONG answer for "what's the most effective FIRST step" — correctness of an option doesn't matter if it's disproportionate effort for an untried simpler fix.

### Structured error responses (MCP)
- MCP's `isError` flag communicates a tool failure back to the agent.
- Distinguish error categories: **transient** (timeouts, service unavailable), **validation** (bad input), **business** (policy violation), **permission**.
- Uniform/generic errors ("Operation failed") prevent the agent from choosing an appropriate recovery action.
- Return structured metadata: `errorCategory`, `isRetryable` boolean, human-readable description — prevents wasted retries on non-retryable errors and lets the agent explain business-rule failures to the user appropriately.
- Distinguish **access failures** (e.g. timeout — needs a retry decision) from **valid empty results** (a successful query that legitimately found nothing) — conflating them is an anti-pattern.
- Subagents should attempt local error recovery for transient failures themselves, propagating to the coordinator only errors they can't resolve, along with partial results and what was attempted.

### Distributing tools across agents / tool_choice
- Giving one agent too many tools (e.g. 18 instead of 4–5) degrades tool-selection reliability by increasing decision complexity.
- Agents with tools outside their specialization tend to misuse them (e.g. a synthesis agent attempting web searches).
- Scope tool access per agent role; provide limited, scoped cross-role tools only for specific high-frequency needs (e.g. giving a synthesis agent a narrow `verify_fact` tool instead of full web-search access, routing complex verification through the coordinator).
- `tool_choice` options: `"auto"` (model may return text instead of calling a tool), `"any"` (must call *some* tool but picks which), forced (`{"type": "tool", "name": "..."}`, must call that specific tool).
- Use forced tool_choice to guarantee a specific tool runs first (e.g. `extract_metadata` before enrichment steps); use `"any"` when you need to guarantee a tool call happens at all.
- Official exam concrete example (principle of least privilege): a synthesis agent needing frequent simple fact-checks (85% of cases) but occasional deep investigation (15%) should get a scoped `verify_fact` tool for the common case, while complex verification still routes through the coordinator to the web-search agent — avoids both the latency of round-tripping everything through the coordinator AND the over-provisioning risk of giving the synthesis agent full web-search access.

### MCP server integration
- Scoping: project-level `.mcp.json` (shared, version-controlled, team tooling) vs user-level `~/.claude.json` (personal/experimental).
- `.mcp.json` supports environment-variable expansion (e.g. `${GITHUB_TOKEN}`) so credentials aren't committed to source control.
- Tools from ALL configured MCP servers are discovered at connection time and available simultaneously.
- MCP **resources** expose content catalogs (issue summaries, doc hierarchies, DB schemas) to reduce exploratory tool calls — resources are for browsing/context, tools are for actions.
- Prefer existing community MCP servers for standard integrations (e.g. Jira); reserve custom servers for team-specific workflows.
- Enhance MCP tool descriptions in detail — otherwise the agent may prefer a built-in tool (e.g. Grep) over a more capable MCP tool simply because the MCP tool's description under-sells it.

### Built-in tools (Read, Write, Edit, Bash, Grep, Glob)
- `Grep`: search file CONTENTS for patterns (function names, error messages, import statements).
- `Glob`: find files by PATH/NAME pattern (e.g. `**/*.test.tsx`).
- `Read`/`Write`: full-file operations. `Edit`: targeted modification via unique text matching.
- When `Edit` fails because the anchor text isn't unique in the file, fall back to `Read` + `Write` for a reliable full-file rewrite.
- Codebase exploration pattern: start with `Grep` to find entry points → `Read` to follow imports/trace flow — don't read every file upfront. To trace a function's usage across re-export/wrapper modules, first identify all exported names, then `Grep` each name across the codebase.
- The `Explore` subagent isolates verbose discovery output during codebase investigation, returning only a summary to the main/coordinating agent — preserves context during multi-phase developer-productivity tasks (e.g. understanding a legacy system before making changes).
- Scratchpad files: have agents write and re-reference persistent findings files to counteract context degradation during long codebase-exploration sessions, where the model may start giving inconsistent answers or referencing "typical patterns" instead of the specific classes/files it found earlier.
- Crash recovery pattern for long-running exploration: each agent/subagent exports its state to a known location; on resume, the coordinator loads a manifest from that location and re-injects it into agent prompts, rather than re-exploring from scratch.

---

## D3 — Claude Code Configuration & Workflows (20%)

### CLAUDE.md hierarchy and modular organization
- Three levels: user-level (`~/.claude/CLAUDE.md`), project-level (`.claude/CLAUDE.md` or root `CLAUDE.md`), directory-level (subdirectory `CLAUDE.md` files).
- User-level settings apply ONLY to that individual user — they are NOT shared with teammates via version control. (Common failure: a new team member doesn't get expected instructions because they were placed at user-level, not project-level.)
- `@import` syntax references external files to keep CLAUDE.md modular (e.g. importing only the standards files relevant to a given package).
- `.claude/rules/` directory: an alternative to one monolithic CLAUDE.md — organizes topic-specific rule files.
- `/memory` command shows which memory files are actually loaded — use it to diagnose inconsistent behavior across sessions.

### Custom slash commands and skills
- Commands: project-scoped in `.claude/commands/` (version-controlled, shared with the team) vs user-scoped in `~/.claude/commands/` (personal, not shared).
- Skills live in `.claude/skills/` as `SKILL.md` files with frontmatter options: `context: fork`, `allowed-tools`, `argument-hint`.
- `context: fork` runs a skill in an ISOLATED sub-agent context — prevents verbose skill output (e.g. full codebase analysis) from polluting the main conversation.
- `allowed-tools` in skill frontmatter restricts which tools the skill can use during execution (e.g. limit to file-write only, to prevent destructive actions).
- `argument-hint` prompts the developer for required parameters when the skill is invoked without arguments.
- Personal skill variants can be created in `~/.claude/skills/` under a different name without affecting teammates.
- Choose skills (on-demand, task-specific) vs CLAUDE.md (always-loaded, universal standards) based on whether the guidance applies to every session or only specific tasks.
- Official exam example (project-wide `/review` command): to make a custom slash command available to EVERY developer automatically when they clone/pull the repo, it must live in `.claude/commands/` in the repository (version-controlled) — NOT in each developer's personal `~/.claude/commands/`, NOT described inside CLAUDE.md (that's for context/instructions, not command definitions), and there is no `.claude/config.json` commands-array mechanism (distractor, doesn't exist).

### Path-specific rules
- `.claude/rules/` files use YAML frontmatter with a `paths` field containing glob patterns for CONDITIONAL rule activation (e.g. `paths: ["terraform/**/*"]`).
- These rules load ONLY when editing matching files — reduces irrelevant context/token usage.
- Advantage over directory-level CLAUDE.md: glob patterns apply to files BY TYPE regardless of which directory they live in (e.g. `**/*.test.tsx` catches test files scattered across many folders — a directory-bound CLAUDE.md can't do this).

### Plan mode vs direct execution
- Plan mode: for complex tasks — large-scale changes, multiple valid approaches, architectural decisions, multi-file modifications. Enables safe exploration/design BEFORE committing to changes, preventing costly rework.
- Direct execution: for simple, well-scoped changes (e.g. adding one validation check to one function, a single-file bug fix with a clear stack trace).
- The `Explore` subagent isolates verbose discovery output, returning only a summary — preserves main conversation context during multi-phase tasks.
- Can combine both: plan mode to investigate/design, then direct execution to implement the planned approach.
- Official exam logic: match the mode to task complexity signaled by scope. "Enter plan mode to explore, understand dependencies, and design before changing" is correct for large architectural changes (e.g. monolith → microservices); starting in direct execution and only switching if things get complicated is WRONG because the complexity was already evident upfront.

### Iterative refinement
- Concrete input/output examples (2–3) communicate expected transformations more reliably than prose descriptions when prose alone produces inconsistent results.
- Test-driven iteration: write the test suite FIRST, then iterate by sharing test failures to guide progressive improvement.
- "Interview pattern": have Claude ask clarifying questions first to surface considerations you hadn't thought of, before it implements a solution in an unfamiliar domain.
- Interacting issues (fixes that affect each other) → address together in one detailed message; independent issues → fix sequentially.

### Claude Code in CI/CD
- `-p` (or `--print`) flag runs Claude Code in NON-INTERACTIVE mode — required in pipelines, or the job hangs waiting for interactive input.
- `--output-format json` + `--json-schema` flags enforce machine-parseable structured output for CI (e.g. posting inline PR comments programmatically).
- CLAUDE.md supplies project context (testing standards, fixture conventions, review criteria) to CI-invoked Claude Code runs.
- Session context isolation: the SAME session that generated code is a worse reviewer of that code than an independent review instance without the generator's reasoning context — self-review is less likely to question its own decisions.
- When re-running reviews after new commits, include prior findings in context and instruct Claude to report only new/still-unaddressed issues (avoids duplicate PR comments).
- There is no `CLAUDE_HEADLESS` env var and no `--batch` flag for Claude Code CLI — these are exam distractors, not real flags.
- Exit codes for scripting: Claude Code exits with code `0` on success and a non-zero code when the run fails, so a CI pipeline script can branch on the exit status (e.g. fail the build step) without needing to parse the output text. (Source: official Claude Code docs, "Run Claude Code programmatically.")
- `--allowedTools` flag auto-approves specific tools (e.g. `"Read,Edit,Bash"`) so a non-interactive `-p` run doesn't stall waiting for a permission prompt that nobody is there to answer — distinct from `--output-format`/`--json-schema`, which control output shape, not permissions.
- `--bare` flag skips auto-discovery of hooks, skills, custom commands, subagents, plugins, MCP servers, auto memory, and CLAUDE.md at startup — used in CI/scripts for faster, more deterministic runs that don't depend on whatever happens to be configured on the runner machine (a teammate's personal hooks, a project's `.mcp.json`, etc. won't silently run).
- `--continue` (or `-c`) continues the most recent conversation in a script/CI context (e.g. a follow-up review pass after an initial `-p` call), distinct from `--resume`/`-r` which continues a conversation by a specific session ID/name.

---

## D4 — Prompt Engineering & Structured Output (20%)

### Explicit criteria to reduce false positives
- Explicit, specific criteria (e.g. "flag comments only when claimed behavior contradicts actual code behavior") outperform vague instructions (e.g. "check that comments are accurate").
- Vague confidence-based instructions like "be conservative" or "only report high-confidence findings" do NOT reliably improve precision — they need to be replaced with specific categorical criteria, not just repeated more strongly.
- High false-positive rates in one issue category erode developer trust in ALL categories, even accurate ones.
- Defining explicit severity levels with concrete code examples per level → consistent classification.

### Few-shot prompting
- Few-shot examples are the most effective technique when detailed prose instructions alone produce inconsistently formatted or inconsistent-quality output.
- Effective for demonstrating handling of AMBIGUOUS cases (e.g. tool selection under ambiguity, borderline test-coverage gaps) — this generalizes the model's judgment to novel similar cases, not just the exact examples shown.
- Reduces hallucination in extraction tasks by showing handling of varied real-world document structures (e.g. informal measurements, inline citations vs bibliographies).
- Typical quantity: 2–4 targeted examples for ambiguous scenarios, ideally showing the REASONING for why one action was chosen over plausible alternatives (not just input→output pairs).
- Still advisory, not deterministic — connects to the cross-domain theme; raises reliability but doesn't guarantee compliance the way tool schemas or hooks do.

### Structured output via tool_use and JSON schemas
- `tool_use` with a JSON schema is the MOST RELIABLE way to get guaranteed schema-compliant output — it eliminates JSON *syntax* errors entirely (unlike asking for JSON in prose, which is advisory and can still produce malformed JSON).
- `tool_choice`: `"auto"` (model may respond with text instead of calling the tool), `"any"` (must call a tool, model picks which), forced `{"type": "tool", "name": "..."}` (must call that specific tool).
- Set `tool_choice: "any"` to guarantee structured output is produced when MULTIPLE extraction schemas/tools exist and the document type is unknown in advance — the model still picks which schema fits, but text-only output is prevented.
- Schema compliance via tool_use eliminates SYNTAX errors but NOT semantic errors (e.g. line items that don't sum to the stated total, correct-looking values in the wrong field) — semantic correctness still needs separate validation.
- Design fields as OPTIONAL/nullable when the source document may not contain that information — prevents the model from fabricating values just to satisfy a required field.
- Use enum fields with an `"other"` + free-text detail-string pattern for extensible/open-ended categories, and values like `"unclear"` for genuinely ambiguous cases.
- Include format-normalization instructions in the prompt alongside the strict schema, to handle inconsistent source formatting.

### Validation, retry, and feedback loops
- Retry-with-error-feedback: on a failed validation, append the SPECIFIC validation error(s) to the follow-up prompt (not just "try again") to guide model self-correction.
- Retries work for FORMAT/structural errors but are ineffective when the required information is simply ABSENT from the source document — no amount of retrying recovers information that was never there.
- Track a `detected_pattern` field on structured findings to enable systematic analysis of which patterns get dismissed by reviewers (feedback loop design).
- Self-correction design pattern: extract both a "calculated_total" and a "stated_total" (or similar derived-vs-stated pair) to let the system flag discrepancies automatically, plus a `conflict_detected` boolean for inconsistent source data.

### Batch processing (Message Batches API)
- ~50% cost savings vs synchronous calls; processing window up to 24 hours; **no guaranteed latency SLA**.
- Appropriate for non-blocking, latency-tolerant workloads (overnight reports, weekly audits, nightly test generation) — INAPPROPRIATE for blocking workflows (e.g. a pre-merge check developers are waiting on).
- The Batch API does NOT support multi-turn tool calling within a single request (can't execute a tool mid-request and get results back inside that same batch call).
- `custom_id` correlates each batch request with its response, including for resubmitting only the FAILED subset (e.g. re-chunking documents that exceeded context limits) rather than resubmitting the whole batch.
- Exam pattern: when one workflow is blocking (pre-merge check) and another is not (overnight report), move ONLY the non-blocking one to batch — don't force both, and don't add a "timeout fallback to real-time" as unnecessary complexity.
- Official exam SLA-math pattern: given a batch processing window (up to 24h) and a downstream SLA (e.g. must guarantee results within 30 hours), calculate the required batch SUBMISSION frequency (e.g. submitting every 4 hours guarantees any given item is picked up and completed within the 30-hour SLA) — this is a quantitative reasoning pattern the exam tests, not just the qualitative blocking/non-blocking choice.
- Run a prompt-refinement pass on a small sample set BEFORE submitting a large batch, to maximize first-pass success rate and avoid costly iterative resubmission of the full volume.

### Multi-instance / multi-pass review
- Self-review limitation: a model that generated code retains reasoning context from that generation, making it LESS likely to question its own decisions in the same session.
- An independent review instance (no prior reasoning context) catches subtle issues better than self-review instructions or extended thinking on the same session.
- Multi-pass review for large changes: split into per-file LOCAL analysis passes + a separate cross-file INTEGRATION pass — avoids "attention dilution" (inconsistent depth per file, contradictory findings across files) that happens in one single large pass.

---

## D5 — Context Management & Reliability (15%)

### Preserving critical information across long interactions
- Progressive summarization risk: condensing numerical values, percentages, dates, and customer-stated expectations into vague prose loses exactness.
- "Lost in the middle" effect: models reliably attend to the beginning and end of long inputs but may under-weight or omit findings buried in the middle.
- Tool results accumulate in context and consume tokens disproportionately to their relevance (e.g. a 40+ field order-lookup response when only 5 fields matter) — trim to relevant fields before they pile up.
- Extract transactional facts (amounts, dates, order numbers, statuses) into a persistent "case facts" block included in every prompt, kept OUTSIDE the summarized/compacted history so it survives compaction.
- Mitigate position effects: place key-findings summaries at the START of aggregated inputs, and use explicit section headers for detailed results.
- Require subagents to include metadata (dates, source locations, methodology) in structured output to support accurate downstream synthesis; have upstream agents return structured facts/citations instead of verbose prose+reasoning when the downstream agent has a limited context budget.

### Escalation and ambiguity resolution
- Legitimate escalation triggers: explicit customer request for a human, a genuine policy exception/gap (not just "this is complex"), or inability to make further progress.
- Honor an EXPLICIT customer request for a human immediately — don't attempt investigation first in that case.
- If the issue is within the agent's capability, acknowledge frustration and offer to resolve; escalate only if the customer reiterates their preference for a human.
- Escalate when policy is silent/ambiguous about the specific request (e.g. competitor price-matching when policy only covers same-site adjustments).
- Sentiment analysis and the model's own self-reported confidence score are UNRELIABLE proxies for actual case complexity — do not use them as the primary escalation trigger. (Official exam example: low first-contact-resolution rate caused by mis-calibrated escalation — the fix is explicit escalation criteria + few-shot examples in the system prompt, not a confidence-threshold auto-router, a separate ML classifier, or sentiment analysis.)
- When multiple customer records match, ask for additional identifying info rather than heuristically picking one.

### Error propagation across multi-agent systems
- Return structured error context to the coordinator: failure type, the query/action that was attempted, any partial results, and possible alternative approaches — this enables an intelligent recovery decision.
- Distinguish ACCESS failures (e.g. a timeout — needs a retry/recovery decision) from VALID EMPTY RESULTS (a successful query that legitimately found nothing) — conflating them is a bug.
- Anti-patterns: silently suppressing an error by returning "empty result = success", OR terminating the entire multi-agent workflow on a single subagent failure. Correct: try local recovery in the subagent first, propagate to the coordinator only what can't be resolved locally, and let the coordinator proceed with partial results, annotating coverage gaps in the final output.

### Managing context in large codebase exploration
- Context degradation in long sessions: the model starts giving inconsistent answers or referencing "typical patterns" instead of the SPECIFIC classes/files it discovered earlier in the session.
- Scratchpad files persist key findings across context boundaries — have agents write and re-reference them to counteract degradation.
- Delegate exploration to subagents (e.g. "find all test files", "trace refund flow dependencies") while the main/coordinating agent keeps only high-level understanding — isolates verbose output.
- For crash recovery: each agent exports its state to a known location; the coordinator loads a manifest on resume and re-injects it into agent prompts.
- `/compact` reduces context usage mid-session when verbose discovery output has filled the context window.

### Human review workflows and confidence calibration
- An aggregate accuracy metric (e.g. "97% overall") can MASK poor performance on a specific document type or specific field — always segment.
- Stratified random sampling of high-confidence extractions is used to measure real error rates and catch novel error patterns that a single aggregate number would hide.
- Field-level confidence scores, calibrated against a labeled validation set, are used to route items to human review — not a single overall document-level confidence.
- Validate accuracy BY document type and BY field segment before reducing/automating away human review for any given segment.

### Information provenance and multi-source synthesis
- Source attribution is commonly lost during summarization steps when findings get compressed without preserving the claim → source mapping.
- Require structured claim-source mappings (claim, evidence excerpt, source URL/doc name) that the synthesis step must preserve and merge, not just prose paraphrase.
- Conflicting statistics from two credible sources: ANNOTATE the conflict with both source attributions — do not arbitrarily pick one value.
- Require publication/collection dates in structured outputs so genuine temporal differences aren't misread as contradictions.
- Render different content types appropriately in synthesis output (financial data as tables, news as prose, technical findings as structured lists) rather than flattening everything into one uniform format.

---

## Out-of-Scope Topics (official — never generate quiz questions from these)

The exam guide explicitly excludes these. Do not build questions around:
- Fine-tuning Claude models or training custom models
- Claude API authentication, billing, or account/quota management
- Deep implementation details of specific programming languages/frameworks (beyond what's needed for tool/schema config)
- Deploying/hosting MCP servers (infra, networking, container orchestration)
- Claude's internal architecture, training process, or model weights
- Constitutional AI, RLHF, or safety training methodology
- Embedding models or vector database implementation details
- Computer use (browser automation, desktop interaction)
- Vision/image analysis capabilities
- Streaming API implementation or server-sent events
- Rate limiting, quotas, or API pricing calculations
- OAuth, API key rotation, or authentication protocol details
- Specific cloud provider configuration (AWS/GCP/Azure)
- Performance benchmarking or model comparison metrics
- Prompt caching implementation details (knowing it exists is fine; internals are not)
- Token counting algorithms or tokenization specifics

---

## Official Sample Question Patterns (reference only — do not reuse verbatim)

Section 9 of the official exam guide includes 12 worked sample questions with explanations, covering Scenarios 1, 2, 3, and 5. These reveal the exam's exact calibration of difficulty and distractor style. Use these PATTERNS to shape new quiz questions — do NOT copy the exact scenarios/numbers verbatim into the daily quiz (that would make the user memorize answers instead of reasoning).

Recurring distractor patterns observed across all 12 official questions:
1. **The "proportionate first step" distractor**: an option that would genuinely work but represents disproportionate effort/infrastructure for a problem that has a simpler untried fix (e.g. building a classifier/routing layer when nobody has tried improving a tool description yet). This is marked wrong even though it's not technically incorrect.
2. **The "right mechanism, wrong layer" distractor**: an option that correctly identifies deterministic enforcement is needed but applies it to the wrong failure mode (e.g. a hook that prevents an action when the real problem is explaining why an action already failed).
3. **The "unreliable proxy" distractor**: using model self-reported confidence or sentiment analysis as a stand-in for actual complexity/correctness — the guide repeatedly flags LLM self-assessment as poorly calibrated and NOT a reliable signal on its own.
4. **The "solves a different problem" distractor**: technically sound advice that targets a cause other than the one described in the scenario (e.g. expanding tool descriptions when the real issue is tool COUNT, or a sentiment-analysis fix when the issue is policy ambiguity, not customer mood).
5. **The "non-existent feature" distractor**: CLI flags/config options that sound plausible but don't exist (`--batch`, `CLAUDE_HEADLESS`, a `.claude/config.json` commands array) — the guide confirms these are deliberately placed as distractors.
6. Correct answers are consistently the option that (a) matches the root cause stated/implied in the scenario text exactly, (b) is the lowest-effort fix that directly addresses that root cause, and (c) is deterministic/structural when the scenario asks for a "guarantee."

Scenario-coverage note: official sample questions exist for Scenarios 1 (Customer Support), 2 (Code Generation), 3 (Multi-Agent Research), and 5 (CI/CD) — none shown for Scenario 4 (Developer Productivity) or 6 (Structured Data Extraction), though both are still in-scope per the Task Statements and Domain blueprint. Keep generating quiz questions for scenarios 4 and 6 using the Task Statement knowledge already in this file.

### Per-domain distractor patterns (for shaping new questions, not reusing verbatim)

**D1 — Agentic Architecture & Orchestration:**
- Hook event confusion: a distractor names the wrong hook event for the job (e.g. PostToolUse when PreToolUse is needed to BLOCK an action before it runs, since PostToolUse only runs after the tool already executed).
- Model-driven vs. pre-configured confusion: a distractor describes a fixed decision tree or hardcoded tool sequence as if it were "the agentic loop," when the defining feature is Claude dynamically deciding the next tool call based on context.
- Completion-signal confusion: a distractor checks for assistant text content, keyword parsing ("done"/"finished"), or an iteration cap as the loop-termination signal, instead of `stop_reason == "end_turn"`.
- Decomposition-gap vs. execution-gap confusion: when subagents each execute correctly but overall coverage is incomplete, a distractor blames subagent execution/communication/context-sharing instead of the coordinator's decomposition never having created a subtask for the missing area in the first place.

**D2 — Tool Design & MCP Integration:**
- Count vs. quality confusion: a distractor treats a reliability problem caused by too MANY tools (decision complexity) as if it were caused by weak/short descriptions, or vice versa — these are different root causes with different fixes (scope tools per role vs. improve description text).
- Error-handling layer confusion: a distractor proposes a PreToolUse hook (which PREVENTS an action before it happens) to fix a problem that is actually about explaining/categorizing a failure that already occurred — hooks don't communicate error reasons, structured error metadata does.
- Resources vs. tools confusion: a distractor uses an MCP tool (action-oriented, consumes a tool call) where an MCP resource (browsable content catalog) would reduce exploratory tool calls more efficiently, or vice versa.
- Effort-order confusion: a distractor jumps to renaming/splitting tools or adding a routing classifier when the untried, lower-effort fix (improving the tool description) hasn't been attempted yet.

**D3 — Claude Code Configuration & Workflows:**
- CLAUDE.md hierarchy-level confusion: a distractor places team-wide conventions at user-level (`~/.claude/CLAUDE.md`, personal/not version-controlled) instead of project-level, or vice versa — classic "new teammate doesn't get the convention" root cause.
- Skills vs. commands vs. rules confusion: a distractor picks the wrong configuration mechanism for a requirement — e.g. a custom slash command (invoked manually) when `.claude/rules/` with path globs is needed for AUTOMATIC, path-conditional application regardless of directory.
- Plan mode threshold confusion: a distractor recommends direct execution for a task whose complexity is already evident upfront (large architectural change), suggesting "start simple and switch to plan mode if it gets complicated" — wrong because the complexity signal was already there before starting.
- Directory-bound CLAUDE.md confusion: a distractor proposes directory-level CLAUDE.md files to handle conventions for files (like tests) that are scattered across MANY different directories — doesn't scale the way a glob-pattern `.claude/rules/` file does.

**D4 — Prompt Engineering & Structured Output:**
- Syntax vs. semantic error confusion: a distractor claims `tool_use` + JSON schema guarantees full correctness, when it only eliminates SYNTAX errors — semantic errors (values swapped between fields, totals that don't sum) still need separate validation.
- Few-shot vs. schema-constraint confusion: a distractor uses few-shot examples or a stronger prompt instruction to fix a fabrication problem that's actually a schema design issue (a required field should be optional/nullable).
- Vague-instruction confusion: a distractor strengthens vague wording ("be conservative," "only report high-confidence findings") instead of replacing it with explicit, specific categorical criteria — repeating/intensifying vague instructions does not reliably improve precision.
- Batch-suitability confusion: a distractor applies the Message Batches API to a BLOCKING workflow (e.g. pre-merge check) for its cost savings, ignoring that batch has no latency SLA — or over-engineers a "timeout fallback to real-time" instead of simply keeping the blocking workflow synchronous.

**D5 — Context Management & Reliability:**
- Access-failure vs. valid-empty-result confusion: a distractor treats a timeout (needs a retry decision) the same as a successful query that legitimately found nothing — conflating these prevents correct recovery logic.
- Aggregate-metric masking confusion: a distractor trusts a single high aggregate accuracy number (e.g. "97% overall") without checking segment-level (per document-type or per-field) performance, which can hide poor performance on a specific slice.
- Compaction/summarization-loss confusion: a distractor relies on general conversation history or `/compact` alone to preserve exact transactional facts (amounts, dates, order numbers), instead of extracting them into a persistent "case facts" block kept outside the summarized history.
- Escalation-trigger confusion: a distractor uses model self-reported confidence or sentiment analysis as the primary escalation trigger, instead of explicit escalation criteria with few-shot examples — self-assessment and sentiment are both poorly-calibrated proxies for actual case complexity.

---

## Mock Exam Mode (on-demand only — not part of the daily cron quiz)

When the user explicitly asks for a "mock exam" / "simulasi ujian" / "full exam":
- Generate 60 questions total (not the daily 5), simulating the real exam format.
- Draw from exactly 4 of the 6 official Exam Scenarios, chosen at random, per the real exam structure (not all 6).
- Mix multiple-choice and multiple-response items; multiple-response items state how many to select, matching real exam format.
- Weight questions by domain per the official blueprint: D1 27%, D2 18%, D3 20%, D4 20%, D5 15% (≈16-17 D1, 11 D2, 12 D3, 12 D4, 9 D5 out of 60 — round sensibly).
- Follow the same Real Exam Style and distractor-pattern guidance as the daily quiz (see "Official Sample Question Patterns" above), scaled up to 60 items — avoid obvious repetition of the same subtopic+scenario combo across the 60.
- Do not reveal answers inline. Deliver all 60 questions, then wait for the user's full answer set before grading (same reply format: "1A 2C 3B...").
- After grading, report: overall score estimate against the 720/1000 passing scale (approximate: percent correct × 1000, since exact scaling is unknown), plus percent-correct broken down by domain (D1-D5), matching the official score report format.
- Because of length, consider delivering in a single message or asking the user if they want it batched into sections — use judgment based on platform constraints (Telegram message length).
- This mode is NOT scheduled automatically; only run it when the user explicitly asks on-demand.

---



## Question Generation Rules

1. Send exactly 5 multiple-choice questions, one correct answer each — EXCEPT occasionally (roughly 1 in 5 quizzes) include one multiple-RESPONSE question that requires selecting 2 correct answers out of 4-5 options; clearly state "(Select 2)" or similar when you do, matching the real exam's mixed item format.
2. Weight by domain: 2 from D1, then 1 each from the other domains
   — but skip any domain whose section is still empty and take from D1/D2 instead.
3. Every question MUST follow the "Real Exam Style": ground it in one of the 6 official Exam Scenarios (or a close variant) — a client/team has a concrete problem or requirement inside that scenario → which fix or design is best? No pure definition or "what does X do" questions. Prefer reusing the official scenario names/tools (e.g. get_customer/lookup_order/process_refund, the multi-agent research pipeline, CI/CD code review) so practice matches real exam framing.
   Example shape:
   "A client's support agent sometimes issues refunds above the allowed limit, even though
   the system prompt says 'NEVER refund more than $100'. What is the most reliable fix?
   A) Repeat the rule in CLAUDE.md in capital letters
   B) Add few-shot examples of refusing large refunds
   C) Add a PreToolUse hook that blocks refund calls above $100
   D) Lower the model temperature"
4. Wrong options must be plausible (common misconceptions or weaker fixes — e.g. "add more few-shot examples", "use a stronger prompt", "add a routing classifier", "switch to a bigger context model") not obviously silly. Favor distractors that fix a DIFFERENT problem than the one actually described (a classic real-exam pattern) or that are technically real but the wrong first step.
5. Do NOT show answers in the quiz message.
6. Avoid correct-answer position bias: do not let the correct answer cluster on one letter (LLMs tend to default to B or C). Before sending, deliberately vary the correct answer's position across the 5 questions so it's spread across A/B/C/D over the course of a quiz (and over consecutive quizzes) — do not just write the "natural" answer position and leave it.
7. When I reply (e.g. "1A 2C 3B 4D 5A"):
   - Grade each answer.
   - Explain in Indonesian (short, with an analogy if it helps).
   - Append each missed topic to weak-topics.md with the date.
8. Check weak-topics.md before each quiz and include at least 1 question on a recent weak topic.
9. Never generate a question from an "Out-of-Scope Topics" item above, even if it seems related.
10. Avoid topic/scenario repetition across recent quizzes: before generating, skim the last 2-3 quizzes sent (check cron output history and/or this session's recent turns if available) and avoid reusing the same narrow subtopic+scenario combo back-to-back (e.g. don't generate another "CI/CD flag combination" question if the previous quiz already had one) — pick a different subtopic within the same domain weight instead. This does not apply to intentional spaced-repetition of a listed weak-topics.md item, which should still recur periodically on purpose.
