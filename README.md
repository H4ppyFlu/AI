# AI

Personal collection of Claude Code agents, commands, skills, and project
rules — mainly built to support a job search workflow (finding postings,
generating tailored German cover letters, reviewing them) plus a couple of
general-purpose dev-workflow helpers.

## Agents

- **architect-review** — read-only senior-architect assessment of a
  codebase's structure, maintainability, and tech debt.
- **code-reviewer** — reviews a diff for bugs/quality issues, reports
  findings without editing.
- **cover-letter-reviewer** — critiques a drafted cover letter against a
  job posting from an HR-screener and hiring-lead perspective.
- **job-scout** — parallel worker spawned by `/search-jobs-orchester` to
  search a single portal query or company batch for job postings.

## Commands

- **/generate-cover-letter** — generates a DIN A4 German cover letter
  tailored to a pasted job description.
- **/search-jobs** — searches for relevant German job postings and scores
  them against the candidate profile.
- **/search-jobs-orchester** — thorough variant of `/search-jobs` that
  fans out parallel `job-scout` subagents for a wider sweep.

## Skills

- **grill-me** — interviews the user about a plan/design until reaching
  shared understanding across every branch of the decision tree.

## Rules

- **rules/CLAUDE.md** — reusable code review standards and style rules
  pulled into other projects.
