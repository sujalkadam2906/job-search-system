# jobhunt-agent

**A daily job-search pipeline that fetches public ATS job boards, filters them deterministically, scores the survivors with an LLM, and emails a digest with drafted application kits. It never submits an application.**

> **Attribution.** This project is built on **`PWP_jobhunt`** by Pratyush Narain
> ([perryvegehan/PWP_jobhunt](https://github.com/perryvegehan/PWP_jobhunt)). The pipeline
> architecture, parsers, provider layer and test suite come from that project. This
> repository is a customised copy tuned for fresher Java/backend and full-stack roles
> in India. See [NOTICE.md](NOTICE.md) for exactly what was changed and by whom.

---

## Overview

`jobhunt` reads job postings from the public, unauthenticated APIs of three applicant
tracking systems (Greenhouse, Lever, Ashby), throws away most of them with a free
regex/location/freshness gate, and only then pays for LLM calls. The few jobs that
score well get a drafted kit (fit summary, tailored resume bullets, honest gaps, a cover
note and questions to ask). Everything lands in an HTML email. A human reads it and
applies by hand.

```
fetch ──► prefilter ──► dedupe ──► screen (LLM, batched) ──► draft (LLM, top N) ──► digest ──► email
 3 ATS     regex +        seen.json    cheap model              better model          HTML       SMTP
 APIs      location +                  score 0-10 + reason      one call per job
           freshness
```

## Problem statement

Fresher job hunting means checking many careers pages daily, and most postings are
the wrong seniority, stack or city. Re-reading the same listings wastes time and
cover-note drafting is repetitive. This tool automates the *finding, filtering and
first-draft* steps while leaving the decision and the submit button to the person.

## Key features

- **Three ATS adapters** (Greenhouse, Lever, Ashby) that handle each vendor's field quirks.
- **Deterministic prefilter** (include/exclude title regexes, location substrings, remote flag, max age) runs before any LLM call.
- **Two-stage LLM cost model**: a cheap batched screen over all survivors, an expensive draft only for the shortlist.
- **Pluggable providers**: Anthropic, Google Gemini, Groq, any OpenAI-compatible endpoint, or local Ollama. Screen and draft stages can use different providers.
- **Persistent dedupe + application tracker** (`seen.json`) with CSV export.
- **Resilient by design**: dead company slugs, a failed LLM batch, or a failed draft never abort a run.
- **Offline mode**: `--mock --scorer keyword` runs the full pipeline from bundled fixtures with no network and no API key.
- **Scheduled via GitHub Actions** (weekdays 06:00 IST), with the dedupe store carried in `actions/cache`.
- **63 offline tests** (parsers, prefilter, LLM layer with stubbed providers).

## Technology stack

| Layer | Technology |
|---|---|
| Language | Python 3.12+ (CI uses 3.12) |
| HTTP | `requests` |
| Config | YAML (`PyYAML`), minimal built-in `.env` reader |
| LLM providers | Anthropic SDK; Gemini / Groq / OpenAI-compatible / Ollama over plain REST |
| Email | `smtplib` + `email.message` (STARTTLS) |
| Storage | JSON file (`seen.json`) + CSV export |
| Testing | `pytest` |
| Automation | GitHub Actions (cron, `actions/cache`, `upload-artifact`) |

## Architecture overview

Each stage is a separate module with one responsibility; HTTP is kept out of the
parsers so the same parsing code runs against live responses and mock fixtures.

1. **fetch** — `fetch_all()` polls each board in `companies.yaml`; `parse_*()` turn JSON into `Job` dataclasses.
2. **prefilter** — drops jobs by title, location and age; no LLM.
3. **dedupe** — `Store.unseen()` removes jobs already in `seen.json`.
4. **screen** — batches of jobs (default 8, JD truncated) sent to the screening model, which returns `[{job_id, score, reason}]`.
5. **draft** — jobs scoring at or above the threshold (capped at `max_per_digest`) get one drafting call each.
6. **digest + mail** — an inline-CSS HTML email is written to `out/digest.html` and optionally sent.
7. **track** — every screened job is recorded in `seen.json`; `out/tracker.csv` is regenerated.

If screening produces no scores at all (every batch failed), the run exits without recording
anything so the same jobs are retried next time.

## Project structure

```
.
├── jobhunt/
│   ├── cli.py          argparse entry point: profile / run / applied / stats
│   ├── fetch.py        Job dataclass, strip_html, 3 parsers, fetch_board/fetch_all
│   ├── prefilter.py    deterministic title/location/freshness gate
│   ├── providers.py    provider interface + 5 backends + resolve()
│   ├── llm.py          build_profile / screen / draft / keyword_screen / parse_json
│   ├── digest.py       HTML digest builder (inline CSS)
│   ├── mailer.py       SMTP sender
│   ├── store.py        seen.json dedupe + tracker + CSV export
│   └── mock.py         fixtures in each ATS's native JSON shape
├── tests/              pytest suite (+ fixtures/test_config.yaml)
├── config.yaml         filters, thresholds, file paths
├── companies.yaml      boards to poll
├── profile.example.json  shape of the (gitignored) profile.json
├── .env.example        environment variable template
├── .github/workflows/daily.yml   scheduled run
├── requirements.txt / pyproject.toml
├── README.md / SETUP.md / NOTICE.md
```

## Data model

There is no database. State is a JSON object keyed by `job_id` (`"<ats>:<slug>:<id>"`) in `seen.json`:

| Field | Meaning |
|---|---|
| `first_seen` | UTC timestamp the job was first recorded |
| `company`, `title`, `location`, `url` | copied from the posting |
| `score`, `reason` | LLM screening result (null if unscored) |
| `emailed` | whether the run that recorded it sent the email |
| `applied`, `applied_on` | set by the `applied` command |

`out/tracker.csv` is an export of the same data.

## External APIs and CLI

The project has no HTTP server. It *consumes* these public endpoints (no auth):

| ATS | Endpoint |
|---|---|
| Greenhouse | `GET https://boards-api.greenhouse.io/v1/boards/{slug}/jobs?content=true` |
| Lever | `GET https://api.lever.co/v0/postings/{slug}?mode=json` |
| Ashby | `GET https://api.ashbyhq.com/posting-api/job-board/{slug}?includeCompensation=true` |

Its own interface is the CLI:

| Command | Purpose |
|---|---|
| `python -m jobhunt profile --resume <pdf\|txt\|md> [--out profile.json]` | build `profile.json` from a resume using the draft-stage model |
| `python -m jobhunt run [--mock] [--scorer llm\|keyword] [--no-draft] [--send] [--limit N]` | run the pipeline |
| `python -m jobhunt applied <job_id>` | mark a job as applied |
| `python -m jobhunt stats` | print tracker counts and export CSV |

Global option: `--config <path>` (default `config.yaml`).

## Installation

Prerequisites: Python 3.12+ and Git.

```bash
git clone https://github.com/sujalkadam2906/jobhunt-agent.git
cd jobhunt-agent
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Environment variables

Copy `.env.example` to `.env` (it is gitignored). Real environment variables take precedence over `.env`.

| Variable | Purpose |
|---|---|
| `LLM_PROVIDER` | `anthropic` (default), `gemini`, `groq`, `openai-compatible`, `ollama` |
| `SCREEN_PROVIDER`, `DRAFT_PROVIDER` | per-stage override of `LLM_PROVIDER` |
| `SCREEN_MODEL`, `DRAFT_MODEL` | model ids; built-in defaults exist per provider |
| `ANTHROPIC_API_KEY` / `GEMINI_API_KEY` / `GROQ_API_KEY` | key for the chosen provider (Groq key is also used by `openai-compatible`) |
| `LLM_BASE_URL` | base URL for `openai-compatible` |
| `OLLAMA_HOST` | Ollama URL (default `http://localhost:11434`) |
| `SMTP_HOST`, `SMTP_PORT` | default `smtp.gmail.com`, `587` |
| `SMTP_USER`, `SMTP_PASS` | sender account; for Gmail use an **App Password** |
| `MAIL_TO` | recipient (defaults to `SMTP_USER`) |

Model ids change over time; set `SCREEN_MODEL`/`DRAFT_MODEL` to ids your provider currently offers.

## Running locally

Offline smoke test (no network, no key):

```bash
python -m jobhunt run --mock --scorer keyword
```

Real run:

```bash
cp .env.example .env                              # add your key
python -m jobhunt profile --resume resume.pdf     # writes profile.json (gitignored)
python -m jobhunt run --limit 10 --no-draft       # cheap first run
python -m jobhunt run --send                      # full run + email
```

Run the tests:

```bash
python -m pytest tests -q
```

For a step-by-step beginner walkthrough, see [SETUP.md](SETUP.md).

### Scheduling on GitHub Actions

`.github/workflows/daily.yml` runs on weekdays at 06:00 IST and on manual dispatch. Set repository
secrets `PROFILE_JSON` (contents of your local `profile.json`), your provider API key, and
`SMTP_USER`, `SMTP_PASS`, `MAIL_TO`. Optional repository variables: `LLM_PROVIDER`, `SCREEN_PROVIDER`,
`DRAFT_PROVIDER`, `SCREEN_MODEL`, `DRAFT_MODEL`. `seen.json` persists between runs through `actions/cache`.

## Screenshots

None included. Run the mock command above and open `out/digest.html` to see the digest.

## Future improvements

- Mark only the jobs actually emailed as `emailed` (currently every screened job in a run is recorded with the run's email status).
- Add more ATS adapters (e.g. Workable, SmartRecruiters) and per-company slug health reporting.
- Add rate-limit backoff/retry to the LLM providers and board fetches.
- Add a LICENSE once the licensing of the upstream project is confirmed.

## Author

Customised by **Sujal Kadam** — [github.com/sujalkadam2906](https://github.com/sujalkadam2906).
Original project: Pratyush Narain, [perryvegehan/PWP_jobhunt](https://github.com/perryvegehan/PWP_jobhunt).
