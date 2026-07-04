# CLAUDE.md

Guidance for Claude Code working in this repository.

## Overview

SkillSensei is an AI resume parsing/scoring platform. A FastAPI backend parses uploaded PDF/DOCX resumes with Google Gemini into structured data, persists it in SQLite via SQLAlchemy, scores it with spaCy + textstat, and serves AI improvement suggestions. The frontend is a set of static HTML pages using React via CDN (no build step).

## Architecture

**Backend (project root, Python):**
- `main.py` — the entire FastAPI app: all endpoints, file text extraction (`fitz`/PyMuPDF for PDF, `python-docx` for DOCX), and Gemini calls (`gemini-2.5-flash`). No routers; everything is defined here.
- `models.py` — SQLAlchemy ORM models. `Resume` is the hub, with cascade relationships to `PersonalInfo`, `WorkExperience`, `Project`, `Education`, `ResumeScore`, and a many-to-many to `Skill` via `resume_skill_association`. Also `JobPosting` and `ResumeJobMatch`.
- `schemas.py` — Pydantic v2 schemas (`from_attributes=True`). Has `field_validator`s that convert DB shapes to API shapes: `skills` list-of-`Skill`-objects → list of names; `technologies` comma-string → list.
- `crud.py` — `create_or_update_resume` upserts by email (then phone); on update it clears and replaces skills/experience/projects/education. `get_or_create_skill` dedupes skills.
- `database.py` — SQLite engine (`sqlite:///./resume_parser.db`) and `SessionLocal`/`Base`.
- `scoring.py` — `ResumeScorer`: skill matching against a fixed `TARGET_SKILLS` list, `textstat` readability. Grammar check is **disabled** (`check_grammar` returns a hardcoded `(85, [])`) because `language_tool_python` needs a Java runtime unavailable on Railway; the returned `grammar_score`/`grammar_errors` are placeholders and the feedback text says so rather than claiming "Excellent grammar!". `overall_score` is weighted `skills*(4/7) + readability*(3/7)` — grammar is excluded from the weighting since it isn't actually assessed.

**Frontend (`skillsensei-frontend/public/`):**
- `index.html`, `analysis.html`, `suggestions.html` — standalone pages. React 18 + ReactDOM + Babel Standalone + Tailwind, all loaded from CDN `<script>` tags. JSX is transpiled in-browser by Babel. There is **no `package.json`, no node_modules, no bundler**.
- `API_BASE_URL` is hardcoded near the top of each page's script (currently the Railway prod URL `https://ai-resume-parser-production.up.railway.app`). Change it there to point at a local backend.

## Running

```bash
# backend (from project root)
pip install -r requirements.txt
python -m spacy download en_core_web_sm   # or it's pinned in requirements.txt
uvicorn main:app --reload                 # http://127.0.0.1:8000 , docs at /docs

# frontend: open skillsensei-frontend/public/index.html directly,
# or serve the folder. No build step.
```

- Requires `GEMINI_API_KEY` in `.env` (loaded via python-dotenv). Without it the app still starts but Gemini calls fail.
- Python 3.11 (`runtime.txt`). Deployed on Railway via `Procfile`: `uvicorn main:app --host 0.0.0.0 --port $PORT`.
- `resume_parser.db` (SQLite file) is committed; tables auto-create on startup via `Base.metadata.create_all`.

## Gotchas

- There are still two similarly-named pairs of endpoints, kept intentionally: `POST /resumes/{id}/analyze` (`analyze_resume`) vs `POST /analyze-resume/{email}` (`analyze_resume_by_email`), and `DELETE /resumes/{id}` (`delete_resume`) vs `DELETE /resume/{email}` (`delete_resume_by_email`, singular `resume`). The SPA (`index.html`) uses the id-based ones; the standalone report pages (`analysis.html`, `suggestions.html`) use the email-keyed ones. Function names were deduplicated so this is no longer a shadowing risk, just two legitimate paths per feature — don't assume one is dead code.
- In `get_dashboard_analytics`, `func` is referenced before its `from sqlalchemy import func` import inside the function — touch this endpoint with care.
- Gemini responses are parsed by stripping ```json fences then `json.loads`; malformed model output raises 500s. `/get-suggestions/{email}` uses `gemini-1.5-flash` while other calls use `gemini-2.5-flash`.
- No automated tests and no linter config in the repo.
- CORS is wide open (`allow_origin_regex=".*"`) to support opening the frontend from `file://`.
