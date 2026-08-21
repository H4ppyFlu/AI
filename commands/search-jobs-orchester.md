# Job Search Command — Orchestrator (parallel subagents)

This is the **thorough, higher-cost variant** of `/search-jobs`, meant to be run occasionally, not as the default. It spawns parallel `job-scout` subagents instead of searching sequentially yourself, which is faster and keeps search noise out of your own context, but each subagent starts cold and costs more overall than the plain `/search-jobs` command. For routine searches, prefer `/search-jobs`; use this one when you want a wider, more thorough sweep.

You are a job search assistant for a German engineering professional. Your task is to search for relevant job postings in Germany, evaluate each one against the candidate's profile, and organize results into three fit categories.

## Arguments

The user may pass an optional keyword: `$ARGUMENTS`

- If `$ARGUMENTS` is empty → **Profile mode**: search broadly based on candidate's core skills
- If `$ARGUMENTS` is not empty → **Keyword mode**: focus the search on that keyword, still scoring against full profile

## Step 1 – Read Candidate Profile

Read the file `.github/instructions/resources/currentKnowhow.md` to understand the candidate's current skills, experience, and background. Never modify this file.

Key profile facts to extract:
- Core technical tools (MATLAB, Simulink, Python, Git)
- Domain (automotive, simulation, energy systems)
- Education level (Master of Science, engineering background)
- Completed training (Machine Learning, Python)

## Geographic Constraint

**Only include postings that meet one of these two conditions:**
1. **Location within ~30 km of Freiburg im Breisgau** — cities and towns in this radius include: Freiburg im Breisgau, Breisach, Staufen, Bad Krozingen, Müllheim, Lahr, Offenburg, Emmendingen, Waldkirch, Titisee-Neustadt, Lörrach (border), Weil am Rhein. When in doubt about a location, include it and note the approximate distance.
2. **Fully remote** — the posting explicitly states "remote", "Homeoffice", "100% Homeoffice", or "ortsunabhängig". Hybrid roles (e.g., "2 Tage Homeoffice") do not qualify unless the office location is also within the 30 km radius.

Discard postings from other German cities (e.g., Munich, Stuttgart, Hamburg, Berlin) that are neither remote nor within the radius.

## Step 2 – Spawn Parallel Search Subagents

Do not use a fixed, pre-written keyword list, and do not search sequentially yourself. Determine the search terms, then dispatch the actual searching to parallel `job-scout` subagents (via the Agent tool, `subagent_type: job-scout`, `run_in_background: false`, all spawned in a single message so they run concurrently). Each subagent works in its own isolated context and reports back a compact raw list — this keeps your own context free of the underlying search noise (aggregator pages, dead links, retries).

### Determine search terms
- **Profile mode** (no argument): from the profile facts extracted in Step 1, pick 4–6 of the most relevant terms (core tools, domain, role title).
- **Keyword mode** (argument provided): `$ARGUMENTS` is the one search term. Do not append assumed-domain words (e.g. "Ingenieur", "Entwickler", "Simulation") unless they appear in `$ARGUMENTS` itself or in the candidate profile — this keeps keyword mode usable for any role the user searches for, not just engineering titles.

### Build a short profile summary (2–3 lines)
From Step 1's facts, write a compact summary (core tools, domain, role title, education) to hand to each subagent for coarse relevance filtering. This is **not** the full `currentKnowhow.md` — subagents don't score fit, they just need enough signal to skip obviously irrelevant postings (e.g. warehouse, sales, apprenticeships) instead of returning everything or nothing.

### Spawn one subagent per search term — portal assignment
For each term (1 in keyword mode, 4–6 in profile mode), spawn a `job-scout` subagent with a prompt containing:
- The profile summary (for coarse filtering only)
- The Geographic Constraint (verbatim from above)
- Its one assigned term `[Begriff]`, and these structured URLs to `WebFetch` first (spaces encoded as `+` or `%20`):
  1. **Bundesagentur für Arbeit**: `https://www.arbeitsagentur.de/jobsuche/suche?was=[Begriff]&wo=Freiburg im Breisgau&umkreis=30`
  2. **Indeed, local**: `https://de.indeed.com/jobs?q=[Begriff]&l=Freiburg im Breisgau&radius=30`
  3. **Indeed, remote**: `https://de.indeed.com/jobs?q=[Begriff] remote&l=Deutschland` — apply the Geographic Constraint manually afterward, this query returns hybrid/loosely-remote postings too
- Fallback `WebSearch` queries for when a structured URL returns a poor-quality page (CAPTCHA, empty shell, no listings), or to catch portals without a reliable URL pattern (StepStone, Jooble, LinkedIn, heise Jobs): `[Begriff] Stellenangebot Freiburg`, `[Begriff] Stelle Freiburg Breisgau`, `[Begriff] remote Homeoffice Stellenangebot Deutschland`, `[Begriff] Freiburg OR remote Stellenangebot`, `[Begriff] Teilzeit remote Stellenangebot Deutschland`
- Link resolution instructions: for any posting found via a portal, try one `WebSearch` for `[Unternehmen] careers jobs Stelle site:[company-domain]` and one `WebFetch` on the result to find the direct company posting URL; if that fails, keep the portal link and note it with `(via [Portal])`

### Spawn 2 subagents for the company list — batch assignment
Read `.claude/commands/resources/ListeFirmen.md` and split it into 2 roughly equal batches (e.g. first half / second half of the rows). Spawn one `job-scout` subagent per batch with a prompt containing:
- The profile summary (for coarse filtering only)
- The Geographic Constraint, but with the radius widened to 60 km for this list (the user has pre-selected these companies as interesting; note the actual distance so they can judge commute feasibility instead of filtering them out)
- Its batch of companies (name + Sitz)
- Per-company instructions: `WebSearch` for `[Firmenname] Stellenangebote Karriere careers site:[company-domain]`, `WebFetch` the careers page; if that returns poor data quality (cookie banner only, empty shell, 404, redirect), fall back to `WebSearch` for `[Firmenname] Stellenangebote [profile summary keywords]` and extract from snippets, noting `(via WebSearch, Karriereseite nicht auswertbar)`; if still unreachable, try one more `WebSearch` for `[Firmenname] offene Stellen Ingenieur 2026`
- The instruction that a posting does **not** need to contain the assigned search term(s) literally — coarse profile relevance (per the summary) is enough to include it; only skip a company silently if it has no plausible profile-relevant opening at all
- The instruction to mark the Link column as the direct company careers URL, and to add `(Firmenliste)` to Schlüsselanforderungen

### After all subagents return
You'll have 1 (keyword mode) or 6 (profile mode: 4–6 term subagents + 2 batch subagents) reports. Move to Step 3 to merge them.

## Step 3 – Merge, Deduplicate, and Verify Coverage

Combine every subagent's raw postings into one pool. Skip exact duplicates (same title + company — this can happen when a term subagent and a batch subagent both surface the same company posting). Skip postings with no extractable requirements. Aim for 15–25 distinct postings across all reports; if far short of that, it's a genuine signal about market thinness for this term, not a search failure — say so in the output rather than padding with weak matches.

## Step 4 – Score Each Posting

Do not use a fixed skill/domain list (e.g. "MATLAB/Simulink", "automotive"). Score every posting by comparing its requirements directly against the full candidate profile read from `currentKnowhow.md` in Step 1 (Tätigkeiten, Fähigkeiten, Studium, Fortbildung, Privat/Freizeit). Re-derive the comparison fresh each run so scoring stays accurate as the profile evolves — never credit a requirement the candidate can't back with a concrete entry in `currentKnowhow.md` (same authenticity rule as the other commands: no fabricated fit). This scoring is done centrally by you, the orchestrator, never by the subagents.

For each posting, work out:
- **Kernanforderungen erfüllt**: how many of the posting's core/must-have requirements are directly backed by an entry in `currentKnowhow.md`
- **Domänen-Nähe**: whether the posting's domain matches the candidate's current domain, or an adjacent one with a credible, documented bridge (not an assumed one)
- **Teilzeit-Kompatibilität**: 80% Teilzeit explicitly acceptable → bonus

### Most Fitting ★★★
Strong direct match:
- Most of the posting's core requirements (roughly 4 or more) are directly backed by entries in `currentKnowhow.md`
- Domain matches the candidate's current domain, or the profile documents a direct, concrete bridge (e.g. a specific tool, method, or project that maps onto the requirement)
- Seniority fits (Senior/Lead/vergleichbar oder unspezifiziert)
- Teilzeit 80% akzeptiert → bonus

### Fitting ★★
Good overlap, some adaptation needed:
- Roughly half of the posting's core requirements are backed by `currentKnowhow.md`
- Domain is adjacent, with at least one genuine bridge documented in the profile, even if the overall field differs

### Medium ★
Transferable skills apply but a meaningful gap exists:
- Only one or two requirements are backed by `currentKnowhow.md`, the rest would need to be learned on the job
- Domain is clearly different from the candidate's documented experience, and the bridge is thin but honest (not invented)

Discard postings with no overlap at all against `currentKnowhow.md`.

### Location bonus
When two postings have similar skill fit, prefer the one closer to Freiburg or explicitly remote. Note in the "Ort" column whether a posting is remote ("Remote/Homeoffice") or local ("Freiburg", "Emmendingen", etc.).

## Step 5 – Save Results

Determine the output filename:
- Profile mode: `jobs/YYYY-MM-DD.md` (use today's date: replace with actual date)
- Keyword mode: `jobs/YYYY-MM-DD-$ARGUMENTS.md` (e.g., `jobs/2026-05-11-Python.md`)

Write the following markdown file to that path:

```
# Jobsuche – [Datum] [Keyword or "Profil-basiert"] (orchestriert)

Suche durchgeführt am [Datum] mit dem Orchestrator-Command (parallele job-scout-Subagents). Kandidatenprofil: Senior-Entwicklungsingenieur, MATLAB/Simulink/Python, Automobilindustrie.
Geografischer Filter: Freiburg im Breisgau ±30 km oder vollständig remote.

---

## Most Fitting ★★★

| # | Stelle | Unternehmen | Ort | Ref.-Nr. | Schlüsselanforderungen | Link |
|---|--------|-------------|-----|----------|------------------------|------|
| 1 | ...    | ...         | ... | ...      | ...                    | ...  |

---

## Fitting ★★

| # | Stelle | Unternehmen | Ort | Ref.-Nr. | Schlüsselanforderungen | Link |
|---|--------|-------------|-----|----------|------------------------|------|
| 1 | ...    | ...         | ... | ...      | ...                    | ...  |

---

## Medium ★

| # | Stelle | Unternehmen | Ort | Ref.-Nr. | Schlüsselanforderungen | Link |
|---|--------|-------------|-----|----------|------------------------|------|
| 1 | ...    | ...         | ... | ...      | ...                    | ...  |

---

## Suchparameter

- **Modus:** Profil-basiert / Keyword: [keyword]
- **Suchbegriffe (an Subagents verteilt):** [list the actual terms used]
- **Subagents:** [N] Begriffs-Subagents + 2 Firmenlisten-Batch-Subagents
- **Geografischer Filter:** Freiburg im Breisgau ±30 km oder vollständig remote
- **Gefundene Stellen gesamt:** [N]
- **Datum:** [date]
```

After saving, tell the user:
- The file path where results were saved
- How many postings are in each category
- Any notable "Most Fitting" highlights worth calling out
- How many subagents were spawned, for transparency about cost
