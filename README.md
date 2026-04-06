# ApplyPilot — Automated Job Application Pipeline

> A fork of [ApplyPilot by Pickle-Pixel](https://github.com/Pickle-Pixel/ApplyPilot) — customized, debugged, and extended by [Sunil Gangupamu](https://github.com/Sunil-3012).

---

## What is ApplyPilot?

ApplyPilot is an end-to-end automated job application pipeline. It scrapes fresh job postings from major job boards, scores them against your resume using an LLM, tailors your resume for each role, generates a cover letter, and autonomously fills out and submits the application — all without manual effort.

---

## How It Works (Pipeline Overview)

```
Discover → Enrich → Score → Tailor → Cover Letter → Auto Apply
```

### Stage 1 — Discover
Scrapes job postings in real time from **LinkedIn, Indeed, and ZipRecruiter** using the [JobSpy](https://github.com/Bunsly/JobSpy) library. Search queries, locations, and job boards are fully configurable via `searches.yaml`. Results are stored in a local SQLite database.

### Stage 2 — Enrich
Visits each job's URL to scrape the full job description and direct application link, since the initial scrape only returns a summary.

### Stage 3 — Score
Sends each job description and your resume to an LLM (Groq API with LLaMA 3.3 70B). The model scores the fit from 1–10. Only jobs scoring **7 or above** move forward in the pipeline — cutting noise and focusing effort on strong matches.

### Stage 4 — Tailor
For each high-scoring job, the LLM rewrites your resume to highlight the most relevant skills and experience. A validation layer checks for hallucinations and fabricated skills before generating a tailored PDF resume.

### Stage 5 — Cover Letter
Generates a personalized cover letter for each job using the LLM, exported as a PDF.

### Stage 6 — Auto Apply
Uses **Claude's computer use API** (Anthropic) and **Playwright** (browser automation) to open the application page, fill out every form field, upload the tailored resume and cover letter, and submit — fully autonomously.

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.11 |
| Job Scraping | [JobSpy](https://github.com/Bunsly/JobSpy) (LinkedIn, Indeed, ZipRecruiter) |
| Database | SQLite |
| LLM (Scoring / Tailoring / Cover Letter) | Groq API — LLaMA 3.3 70B |
| Browser Automation | Playwright |
| Auto-Apply AI Agent | Anthropic Claude API (claude-haiku) |
| Resume / Cover Letter Output | PDF generation |
| Config | YAML + JSON |

---

## My Customizations & Bug Fixes

### 1. Fixed LLM 404 Fallback (`src/applypilot/llm.py`)
The original code only handled HTTP `403` errors to trigger the native Gemini API fallback. The Gemini OpenAI-compatible endpoint was returning `404`, which caused all scoring jobs to silently fail. Extended the error handler to also catch `404`:

```python
# Before
if resp.status_code == 403 and self._is_gemini:

# After
if resp.status_code in (403, 404) and self._is_gemini:
```

### 2. Fixed False Positives in Resume Validator (`src/applypilot/scoring/validator.py`)
The fabrication watchlist — designed to prevent fake skills from being added to resumes — incorrectly included `spring` and `angular`, which are real skills in my profile (Spring Boot, Angular). This caused **19 out of 20** resume tailoring attempts to fail validation. Removed the false positives:

```python
# Removed from FABRICATION_WATCHLIST:
# "spring", "angular", "certif", "certified", "aws certified"
```

### 3. Disabled Workday & Smart Extract Scrapers (`src/applypilot/pipeline.py`)
Added config flags `workday_enabled` and `smartextract_enabled` to `searches.yaml` so scrapers can be toggled without touching code. Workday scraping added significant runtime (48 companies × 6 queries) with minimal return for my job search.

### 4. Configured Full US Job Search (`my-config/searches.yaml`)
- Reduced search queries from 20 down to 7 focused queries to cut discovery time
- Set location to Remote + Tampa + United States for nationwide coverage
- Removed Google Jobs board (low signal-to-noise ratio)
- Added US-wide `accept_patterns` and non-US `reject_patterns`

---

## Configuration

All personal configuration lives in `my-config/` (not committed — see `.gitignore`):

| File | Purpose |
|---|---|
| `profile.json` | Your personal info, skills, work history, salary expectations |
| `searches.yaml` | Search queries, job boards, locations, filters |
| `resume.txt` | Plain text resume used for LLM scoring |
| `.env` | API keys (Groq, Anthropic, etc.) |

Copy your config files to `~/.applypilot/` before running:

```bash
cp my-config/searches.yaml ~/.applypilot/searches.yaml
cp my-config/profile.json ~/.applypilot/profile.json
cp my-config/resume.txt ~/.applypilot/resume.txt
cp my-config/.env ~/.applypilot/.env
```

---

## Running the Pipeline

```bash
# Install dependencies
pip install -e .
pip install python-jobspy
playwright install chromium

# Run each stage
applypilot run discover       # Scrape fresh job postings
applypilot run score          # Score jobs with LLM
applypilot run tailor         # Tailor resumes for top jobs
applypilot run cover          # Generate cover letters
applypilot apply --limit 5    # Auto-apply to 5 jobs
```

### Useful Database Commands

```bash
# View job summary
sqlite3 ~/.applypilot/applypilot.db "SELECT COUNT(*), site FROM jobs GROUP BY site;"

# Delete old unapplied jobs and start fresh
sqlite3 ~/.applypilot/applypilot.db "DELETE FROM jobs WHERE applied_at IS NULL;"

# Apply to a specific job by URL
applypilot apply --url "https://www.indeed.com/viewjob?jk=..."
```

---

## Credits

- **Original Project:** [ApplyPilot by Pickle-Pixel](https://github.com/Pickle-Pixel/ApplyPilot)
- **Job Scraping:** [JobSpy](https://github.com/Bunsly/JobSpy) by the open source community
- **Fork & Customization:** [Sunil Gangupamu](https://github.com/Sunil-3012)
