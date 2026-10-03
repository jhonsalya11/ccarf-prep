# CCAR-F Prep

Study material and quiz source for the **Claude Certified Architect – Foundations (CCAR-F)** exam.

A daily quiz agent (Hermes + the `ccarf-quiz` skill) reads this folder, generates exam-style questions, grades answers, and logs missed topics.

## Files

| File | Purpose |
|------|---------|
| `knowledge.md` | The only source of quiz content: exam overview, the 6 official scenarios, the advisory-vs-deterministic theme, facts for domains D1–D5, out-of-scope topics, sample question patterns, mock exam mode, and the question generation rules. |
| `weak-topics.md` | Auto-appended log of missed topics and a per-domain accuracy tracker. Written by the quiz agent after grading, so don't edit it by hand. |
| `AGENTS.md` | Rules for any AI agent editing `knowledge.md`. |

## Exam at a glance

- Proctored, 60 questions, 120 minutes, multiple-choice and multiple-response
- 4 scenarios per exam, drawn from a bank of 6
- Passing score: 720 on a 100–1,000 scale

| Domain | Weight |
|--------|--------|
| D1 Agentic Architecture & Orchestration | 27% |
| D2 Tool Design & MCP Integration | 18% |
| D3 Claude Code Configuration & Workflows | 20% |
| D4 Prompt Engineering & Structured Output | 20% |
| D5 Context Management & Reliability | 15% |

## How the quiz works

1. 5 scenario-based questions per quiz, weighted by domain (2 from D1, 1 each from the others).
2. Reply with answers like `1A 2C 3B 4D 5A`.
3. The agent grades them, explains in Indonesian, and appends missed topics to `weak-topics.md`.
4. Recent weak topics are included in later quizzes for spaced repetition.

## Editing the knowledge base

Keep these intact so quiz generation keeps working:

- The `## D1` … `## D5` headings and their `(weight%)`.
- `### Subtopic` headings with short, self-contained bullet facts.
- The "Cross-Domain Theme" and "Question Generation Rules" sections.
- No answer keys or "correct answer" markers next to facts.
- No invented facts. Use `<!-- TODO -->` for gaps.
- Update the `Last updated:` date at the top of `knowledge.md` on every change.

See `AGENTS.md` for the full rules.
