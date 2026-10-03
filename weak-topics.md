## Domain Accuracy Tracker (running tally, updated after each quiz)

Format: `[date] Domain-by-domain correct/total for that quiz`. Running totals below the log. Use this to spot which domain is quantitatively weakest, not just which topics were missed qualitatively.

### Per-quiz log
- 2026-09-29: D1 2/2, D2 1/1, D4 0/1, D5 1/1 (5 Qs, domains per that day's mix)
- 2026-09-30 (round 1): D1 2/2, D2 0/1, D3 1/1, D4 1/1 (Q3 MCP error handling missed)
- 2026-09-30 (round 2): D1 2/2, D3 1/1, D5 1/1, D2 1/1 (5/5)
- 2026-09-30 (round 3): D1 1/1, D3 2/2, D2 0/1, D4 1/1 (Q3 tool-count missed)
- 2026-10-01 (cron): D1 0/1 (Q1 missed), D1 1/1 (Q2), D2 1/1, D3 1/1, D4 1/1

### Running totals by domain (as of 2026-10-01)
- D1 (Agentic Architecture & Orchestration): 6/7 correct (~86%) — the 1 miss was the recurring task-decomposition root-cause confusion (3rd occurrence)
- D2 (Tool Design & MCP Integration): 2/4 correct (50%) — weakest domain so far; misses were MCP error-handling layer confusion and tool-count-vs-description confusion
- D3 (Claude Code Configuration & Workflows): 4/4 correct (100%)
- D4 (Prompt Engineering & Structured Output): 2/3 correct (~67%)
- D5 (Context Management & Reliability): 2/2 correct (100%)

Note: sample size is still small (only ~20 questions total across 5 quizzes) — treat percentages as directional, not statistically solid yet. Re-evaluate after ~10 quizzes (50 questions) for more reliable signal. D2 is the current priority domain based on this data, consistent with weak-topics.md's qualitative log above.

---


## 2026-10-01
- Multi-agent coordinator task decomposition root cause (REPEATED MISS — 3rd time: Sept 29, Sept 30, Oct 1): when a coordinator's decomposition log shows entire subdomains were never created as subtasks at all (not merely executed poorly), the root cause is the coordinator's decomposition being too narrow/incomplete — NOT the synthesis subagent failing to merge results (there's nothing to merge if a subtask was never created in the first place) and NOT subagent context isolation. Key diagnostic: check whether the missing area ever got a subtask assigned; if not, it's a decomposition gap, not a merge/execution gap. Needs continued repetition — this is the user's most persistent weak topic.

## 2026-09-30 (session 3)
- Distributing tools across agents: when a coordinator has access to too many tools (e.g. 18), the root cause of misrouting is decision complexity from tool COUNT itself, not necessarily weak/short descriptions — mistakenly picked "lengthen all descriptions" instead of "scope tools per specialized subagent role." Distinguish this from the separate, also-real problem of ambiguous/overlapping tool descriptions (different fix, different root cause).

## 2026-09-30 (session 2)
- Why hooks are "deterministic" (conceptual, not just "which hook fixes X"): the defining property is that hooks are CODE executed OUTSIDE the model's discretion — they run regardless of what the model decides, with no dependency on the model "choosing" to comply. It is NOT about timing (before vs after the model's text response) — PreToolUse runs before the tool call, PostToolUse runs after the tool call but still before the model's final response either way. Got the mechanism-vs-fix questions right before but missed this more conceptual "why" framing.

## 2026-09-30
- Multi-agent coordinator task decomposition (root cause diagnosis): when subagents each execute correctly but final coverage is incomplete, check the COORDINATOR's decomposition logs first — a too-narrow breakdown (e.g. splitting "AI in creative industries" into only visual-arts subtasks) is the usual root cause, not subagent context inheritance or communication. Missed this again after getting it right once before — needs more repetition.
- Structured MCP error responses (errorCategory/isRetryable): still missing this — when a generic error hides whether a failure is retryable (transient) vs not (business/policy violation), the fix is structured error metadata, NOT a PreToolUse hook (hooks prevent actions before they happen; they don't explain why an already-attempted action failed) and not a retry cap alone.


- Structured MCP error responses: when a tool can fail for multiple reasons (transient outage vs business/policy violation), the fix is structured error metadata (errorCategory, isRetryable boolean) so the agent can react differently per failure type — not a PreToolUse hook (that prevents actions, doesn't communicate why an action failed) and not blind retries.
- Claude Code CLI flags for CI/CD: running non-interactively in a pipeline REQUIRES the `-p` (or `--print`) flag, separate from `--output-format json` + `--json-schema` (which control structured output format). Forgot `-p` when asked for the full flag combination. Also remember `--batch` and `CLAUDE_HEADLESS` are NOT real Claude Code flags — common exam distractors.

## 2026-09-29
- Parallel subagent invocation: must emit multiple `Task` tool calls in a SINGLE coordinator response to run them concurrently — not by sharing conversation history/context between subagents (that breaks the isolated-context design). Mistook shared context as the mechanism for parallelism.
- Tracing a function's usage across re-export/wrapper modules: correct approach is to first identify ALL exported names/aliases for the function, then `Grep` each name across the codebase — not `Glob` + reading every file broadly.
- Structured extraction schema design: when a source document may legitimately lack a field (e.g. PO number on some invoices), mark that field optional/nullable in the JSON schema so the model can omit it — this is what stops fabrication, not adding few-shot examples (few-shot is advisory, doesn't change what the schema allows/requires).
