# Job Search Command

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

## Step 2 – Build Search Queries

Do not use a fixed, pre-written keyword list. Search terms must always come from either `$ARGUMENTS` (keyword mode) or freshly extracted from `currentKnowhow.md` (profile mode), so the search adapts automatically whenever the argument or the profile changes.

### Profile mode (no argument)
From the profile facts extracted in Step 1, pick 4–6 of the most relevant terms (core tools, domain, role title).

### Keyword mode (argument provided)
Use `$ARGUMENTS` itself as the one search term. Do not append assumed-domain words (e.g. "Ingenieur", "Entwickler", "Simulation") unless they appear in `$ARGUMENTS` itself or in the candidate profile from Step 1 — this keeps keyword mode usable for any role the user searches for, not just engineering titles.

### Primary source — structured portal URLs (use these first)
Plain natural-language `WebSearch` queries on job-portal topics mostly return aggregator/overview pages ("94 Python jobs in Freiburg"), not individual postings. Instead, `WebFetch` these location- and radius-aware URLs directly for each search term `[Begriff]` (profile term or `$ARGUMENTS`), with spaces encoded as `+` or `%20`:

1. **Bundesagentur für Arbeit** (official, precise Umkreis radius filter): `https://www.arbeitsagentur.de/jobsuche/suche?was=[Begriff]&wo=Freiburg im Breisgau&umkreis=30`
2. **Indeed, local**: `https://de.indeed.com/jobs?q=[Begriff]&l=Freiburg im Breisgau&radius=30`
3. **Indeed, remote**: `https://de.indeed.com/jobs?q=[Begriff] remote&l=Deutschland` — then apply the Geographic Constraint filter manually, since this query returns hybrid/loosely-remote postings too, not just fully-remote ones

Repeat for each of the 4–6 profile terms (profile mode) or the single `$ARGUMENTS` term (keyword mode). If a fetch returns a poor-quality page (CAPTCHA block, empty shell, no listings), fall back to the WebSearch queries below for that term instead of skipping it.

### Fallback / supplementary — WebSearch
Use these to catch portals without a reliable URL pattern (StepStone, Jooble, LinkedIn, heise Jobs) or when a structured URL above failed:
1. `[Begriff] Stellenangebot Freiburg`
2. `[Begriff] Stelle Freiburg Breisgau`
3. `[Begriff] remote Homeoffice Stellenangebot Deutschland`
4. `[Begriff] Freiburg OR remote Stellenangebot`
5. `[Begriff] Teilzeit remote Stellenangebot Deutschland`

## Step 2b – Direct Company Career Page Search

Read the file `.claude/commands/resources/ListeFirmen.md` to get the full list of target companies and their locations.

For **every company** in that list, do the following — here use the geographic constraint of 60km (the user has pre-selected these companies as interesting; note the distance in the output so they can decide):

### For each company:
1. Use WebSearch to find the company's careers page URL:
   `[Firmenname] Stellenangebote Karriere careers site:[company-domain]`
2. Use WebFetch on that careers page to scan for open positions
3. **If WebFetch returns poor data quality** — e.g. only cookie-consent text, navigation boilerplate, an empty shell, a 404, or a redirect to an unrelated domain (common on JavaScript-rendered career portals like Personio, SmartRecruiters, or Workday) — do not treat the company as having no openings. Instead, fall back to WebSearch with a targeted query, e.g. `[Firmenname] Stellenangebote Python Simulation Ingenieur`, and extract job titles/details from the search snippets instead. Note in the Schlüsselanforderungen cell `(via WebSearch, Karriereseite nicht auswertbar)` when this fallback was used.
4. Extract any postings that are relevant to the candidate profile from `currentKnowhow.md` — in keyword mode, `$ARGUMENTS` only steers which search queries get run (Step 2), it is not a filter on the results. A posting found via the company scan does **not** need to literally contain the keyword to be kept: if it is a genuine profile match (per Step 4's scoring), include it and score it on its own merits. Only drop it if it has no real overlap with the profile at all.
5. If a company has **no matching open positions** (confirmed via WebFetch or the WebSearch fallback), skip it silently — do not list it
6. If a company's careers page is unreachable, try one alternative search: `[Firmenname] offene Stellen Ingenieur 2026`

### Batching for efficiency
Run WebSearch calls for multiple companies in parallel. Suggested batches of 8–10 companies at a time. Then fetch the resulting career pages in parallel.

### Location note for this section
Companies from ListeFirmen.md that are outside the 30 km Freiburg radius (e.g., Ineratec/Karlsruhe, Koehler/Offenburg) — still include matching positions but mark the Ort column with the actual city so the candidate can judge commute feasibility.

### Output
Add all matching positions found via company career pages into the same collection pool as Step 2 results. Mark their source in the Link column as the direct company careers URL (not a job portal). Add a column note `(Firmenliste)` in the Schlüsselanforderungen cell if it helps distinguish the source.

## Step 3 – Collect Job Postings

From the search results, extract individual job postings. For each result, collect:
- **Stelle** (job title)
- **Unternehmen** (company name)
- **Ort** (location / city)
- **Ref.-Nr.** (job reference / requisition number if shown — leave `–` if not found)
- **Schlüsselanforderungen** (2–4 key requirements visible in the snippet)
- **Link** (URL to the posting)

Aim to collect 15–25 distinct postings across all searches. Skip duplicates (same title + company). Skip postings with no extractable requirements.

### Link resolution
For every job found via a portal (Indeed, StepStone, Jooble, Glassdoor, heise Jobs, LinkedIn, etc.):
1. Search for the company's own careers page (e.g., `[Unternehmen] careers jobs Stelle site:[company-domain]`)
2. Use WebFetch on the company's careers page to find the direct posting URL
3. **If WebFetch returns poor data quality** (cookie banner only, empty shell, 404, redirect to an unrelated domain), fall back to WebSearch for the specific job title plus company name to try to locate the direct posting URL from search snippets instead
4. Use the **company careers link** in the results table, not the portal link
5. If no direct company link can be found after one attempt (WebFetch or WebSearch fallback), fall back to the portal link and note it with `(via [Portal])`

## Step 4 – Score Each Posting

Do not use a fixed skill/domain list (e.g. "MATLAB/Simulink", "automotive"). Score every posting by comparing its requirements directly against the full candidate profile read from `currentKnowhow.md` in Step 1 (Tätigkeiten, Fähigkeiten, Studium, Fortbildung, Privat/Freizeit). Re-derive the comparison fresh each run so scoring stays accurate as the profile evolves — never credit a requirement the candidate can't back with a concrete entry in `currentKnowhow.md` (same authenticity rule as the other commands: no fabricated fit).

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
# Jobsuche – [Datum] [Keyword or "Profil-basiert"]

Suche durchgeführt am [Datum]. Kandidatenprofil: Senior-Entwicklungsingenieur, MATLAB/Simulink/Python, Automobilindustrie.
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
- **Suchbegriffe:** [list the actual queries used]
- **Geografischer Filter:** Freiburg im Breisgau ±30 km oder vollständig remote
- **Gefundene Stellen gesamt:** [N]
- **Datum:** [date]
```

After saving, tell the user:
- The file path where results were saved
- How many postings are in each category
- Any notable "Most Fitting" highlights worth calling out
