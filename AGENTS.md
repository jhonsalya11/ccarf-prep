# AGENTS.md — CCAR-F Prep Knowledge Base

This folder feeds a daily quiz system (Hermes agent + `ccarf-quiz` skill).
Any AI agent editing `knowledge.md` here must follow these rules so quiz
generation keeps working correctly.

## Files

- `knowledge.md` — the ONLY source of quiz content. Questions are generated
  ONLY from facts written here. Do not remove the "Question Generation Rules"
  section at the bottom — that's the spec another agent (Hermes) reads to
  build quizzes.
- `weak-topics.md` — auto-appended log of topics the user got wrong, with
  dates. Do not edit manually; Hermes appends to it after grading.

## Format conventions for knowledge.md

- Keep the `## D1` … `## D5` domain headings and their `(weight%)` exactly as
  is — the quiz generator weights questions by these.
- Under each domain, use `### Subtopic` headings with short bullet facts.
  Keep bullets factual and self-contained — each bullet should make sense
  without needing outside context, since questions are built only from what's
  written here.
- Do NOT invent facts to fill gaps. If a domain/subtopic has no solid source
  material yet, leave it as `<!-- TODO -->` rather than guessing — the quiz
  generator is instructed to skip empty/TODO sections rather than hallucinate
  questions from them.
- Keep the "Cross-Domain Theme" section (advisory vs deterministic enforcement)
  — it's a recurring exam pattern the user wants reinforced across domains.
- Never edit the "Question Generation Rules" section unless the user
  explicitly asks — it controls quiz behavior (count, weighting, language,
  grading), not content.
- Update the `Last updated:` date at the top whenever you change this file.

## What NOT to do

- Don't add answer keys or "correct answer" markers next to facts — the quiz
  generator picks the correct answer live at generation time based on facts,
  and any embedded answer hints risk leaking into quizzes.
- Don't restructure into a different format (e.g. flat bullet list, JSON) —
  the heading structure is what the domain-weighting rule parses against.
