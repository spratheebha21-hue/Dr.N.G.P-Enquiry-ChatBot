# Dr. NGP IT Enquiry Chatbot

A browser-based enquiry chatbot branded for **Dr. NGP IT**. It answers from a local JSON knowledge base using classic NLP: preprocessing, TF-IDF, cosine similarity, rapidfuzz fallback, regex entity extraction, and per-session context. It does **not** use an LLM, hosted model, or external AI API.

> **Important:** `backend/data/college_config.json` still contains sample/dummy college facts from the starter project. Update those values with verified Dr. NGP IT information before publishing. The branding name is set to Dr. NGP IT, but the chatbot’s factual answers are only as accurate as the configured knowledge base.

## Features

- 110 intents and 880 example patterns across general enquiries, admissions, courses, fees, academics, placements, facilities, student life, administration, research, and visitors.
- TF-IDF (unigrams + bigrams) with cosine similarity and rapidfuzz fallback.
- Typo/abbreviation normalisation, stop-word removal, optional NLTK WordNet lemmatisation, and offline rule-based fallback.
- Department, program-level, year, and semester entity extraction; short contextual follow-ups.
- Hot reload of `intents.json` and `college_config.json` when files change.
- React 18 + TypeScript responsive full-window chat UI, light/dark mode, quick topics, suggestions, feedback, markdown, and browser-local history.
- FastAPI endpoints, basic per-session rate limiting, unmatched-query and feedback JSONL logs.
- Pytest suite and knowledge-base validation script.
- Manual development/runtime setup only. **No Docker configuration is included.**

## Project structure

```text
.
├── README.md
├── backend/
│   ├── main.py                    # FastAPI routes, CORS, rate limiting
│   ├── matcher.py                 # intent loading, TF-IDF, fuzzy match, context
│   ├── entities.py                # department, program level and year extraction
│   ├── config.py                  # app settings, config reload, placeholder rendering
│   ├── models.py                  # Pydantic API models
│   ├── utils.py                   # text preprocessing and JSONL helpers
│   ├── requirements.txt
│   ├── data/
│   │   ├── intents.json           # 110-intent knowledge base
│   │   └── college_config.json    # branding, facts, and department data
│   ├── scripts/validate_intents.py
│   └── tests/                     # API, matcher, context, entities, fallback, config tests
└── frontend/
    ├── index.html
    ├── package.json
    ├── vite.config.ts             # dev API proxy; default backend port 8010
    ├── tailwind.config.js
    └── src/
        ├── App.tsx
        ├── api/client.ts
        ├── hooks/                  # chat, theme, localStorage
        ├── components/             # chat UI components
        └── utils/                  # markdown and time formatting
```

## Requirements

- Python **3.11+** (Python 3.12 has been tested)
- Node.js **20+** and npm
- A free local port **8010** for the backend and **5173** for Vite by default

Port 8000 is deliberately avoided: another local service may already own it. If the backend fails to bind, check the port and use the same free port for Uvicorn and the Vite proxy.

## Local setup (Windows)

Requirements: Python 3.11 or newer, Node.js 20 or newer, and npm. Start the
backend and frontend in **two separate PowerShell terminals** from the project
root.

### 1. Start the backend

In the first terminal:

```powershell
cd backend
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8010
```

The virtual environment only needs to be created once. If `backend\.venv`
already exists, skip the `py -m venv .venv` command. The backend uses the
rule-based lemmatiser if the optional NLTK WordNet data is not installed.

Check that the backend is running by opening
[http://127.0.0.1:8010/api/health](http://127.0.0.1:8010/api/health). The
response should include `"status":"ok"`, `"intents_loaded":110`, and
`"config_loaded":true`. API documentation is at
[http://127.0.0.1:8010/docs](http://127.0.0.1:8010/docs).

### 2. Start the frontend

In a second terminal, from the project root:

```powershell
cd frontend
npm ci
npm run dev
```

`npm ci` installs the exact dependencies in `package-lock.json`; run it again
only when setting up the project or after dependencies change. Open the URL
printed by Vite, normally [http://localhost:5173](http://localhost:5173).
Vite forwards `/api` requests to the backend at `http://localhost:8010`.

### 3. Stop the app

Press **Ctrl+C** in each terminal.

Before publishing, replace the sample facts in
`backend/data/college_config.json` with verified institutional information.
The backend reloads config changes automatically.

### Changing the backend port

If port 8010 is occupied, pick another port (for example 8011) and use it in
**both** places:

1. Start Uvicorn with `--port 8011`.
2. In `frontend/vite.config.ts`, set `server.proxy['/api'].target` to
   `http://localhost:8011`.
3. Restart Uvicorn and Vite.

On Windows, inspect port ownership with:

```bat
netstat -ano | findstr :8010
tasklist /FI "PID eq <PID>"
```

If an error response says `body.messages` is required, the request is reaching
another service, not this chatbot backend. Check the Uvicorn startup output and
verify `/api/health` at the selected backend port.

## Runtime configuration

Settings are read from environment variables by `backend/config.py`. The
project does **not** automatically load a `.env` file; set variables in the
shell before starting Uvicorn.

| Variable | Default | Description |
| --- | --- | --- |
| `CONFIDENCE_THRESHOLD` | `0.35` | Minimum TF-IDF cosine score for a match. Higher is stricter. |
| `FUZZY_THRESHOLD` | `60` | Minimum rapidfuzz score (0–100) for fallback matching. |
| `FOLLOW_UP_BOOST` | `1.2` | Weight multiplier for intents whose `follow_up_of` includes the previous intent. |
| `SESSION_TTL_SECONDS` | `1800` | Session-context idle time-to-live (30 minutes). |
| `RATE_LIMIT_PER_MINUTE` | `30` | Chat requests allowed per session per rolling minute. |
| `MAX_MESSAGE_LENGTH` | `1000` | Maximum accepted chat message length. |
| `CORS_ORIGINS` | `*` | Comma-separated frontend origins, e.g. `http://localhost:5173`. |
| `INTENTS_PATH` | `backend/data/intents.json` | Optional alternate knowledge-base path. |
| `CONFIG_PATH` | `backend/data/college_config.json` | Optional alternate config path. |
| `UNMATCHED_LOG` | `backend/logs/unmatched.log` | Optional unmatched-query log path. |
| `FEEDBACK_LOG` | `backend/logs/feedback.jsonl` | Optional feedback log path. |

PowerShell example (run from `backend/`):

```powershell
$env:CONFIDENCE_THRESHOLD = "0.38"
$env:CORS_ORIGINS = "http://localhost:5173"
.\.venv\Scripts\python.exe -m uvicorn main:app --reload --host 127.0.0.1 --port 8010
```

## Updating the college configuration

`backend/data/college_config.json` contains two main objects:

### `placeholders`

A flat mapping of values referenced by responses using `{{key}}`. For example:

```json
{
  "college_name": "Dr. NGP IT",
  "bot_name": "Dr. NGP IT Assist",
  "website": "https://www.drngpit.ac.in/",
  "college_email": "replace-with-verified-email",
  "college_phone": "replace-with-verified-phone",
  "accent_color": "#4F46E5",
  "logo_url": ""
}
```

The frontend reads the public branding subset from `GET /api/config/public`.
That endpoint returns branding only; the complete configuration remains on the
backend.

### `departments`

Department-specific facts used by `{{dept_specific_response}}`:

```json
{
  "cse": {
    "name": "Computer Science and Engineering",
    "hod": "Verified HOD name",
    "email": "verified-department-email",
    "phone": "verified-department-phone",
    "seats": "Verified intake",
    "highlights": "Verified department details"
  }
}
```

Department keys should match canonical entity keys recognised in
`backend/entities.py` (for example `cse`, `ece`, `eee`, `mech`, `civil`, `it`,
`aids`, `aiml`, `mba`, and `mca`). Add aliases in `entities.py` if the college
uses additional branch names.

Empty or absent placeholder values are rendered as helpful fallback wording;
raw `{{placeholder}}` syntax should never be shown to the visitor. Review every
sample value and response for accuracy before deployment.

## Adding or editing intents

Edit `backend/data/intents.json`. Each intent has this structure:

```json
{
  "tag": "new_intent_tag",
  "category": "admissions",
  "patterns": [
    "formal question phrasing",
    "casual question phrasing",
    "short keyword query",
    "common typo or abbreviation",
    "..."
  ],
  "responses": [
    "First response variant using {{college_name}} and {{website}}.",
    "Second response variant with verified information."
  ],
  "suggestions": ["existing_intent_tag", "another_existing_tag"],
  "follow_up_of": ["previous_intent_tag"]
}
```

Knowledge-base rules:

- Tags must be unique `snake_case` identifiers.
- Provide at least **8 varied patterns** and **2 response variants** for every
  intent. Mix formal, casual, keyword-only, slang, and likely typo phrasing.
- Provide 2–4 suggestion tags that resolve to real intents. The API turns tags
  into display labels; chip selections are indexed back to their intent.
- `follow_up_of` is optional. Use it when a question is naturally likely after
  another intent; those candidates receive a ranking boost in session context.
- Keep overlapping topics distinct with discriminative terms. For example,
  use `tuition` for `tuition_fee`, `hostel` for `hostel_fee`, and `fee
  structure` for the general `fee_structure` intent.
- Do not add unverified prices, dates, placements, accreditation, rankings, or
  statistics. Put verified facts in config placeholders instead.
- Course, fee, and deadline replies should direct visitors to the official
  website or admission helpline for current details.
- The backend hot-reloads the file when saved. Run validation and tests after
  edits.

## Validation, tests, and quality checks

Run these commands from `backend/` in PowerShell:

```powershell
.\.venv\Scripts\python.exe scripts/validate_intents.py
.\.venv\Scripts\python.exe scripts/validate_intents.py --strict
.\.venv\Scripts\python.exe -m pytest tests -v -s
```

The validator checks unique tags, categories, non-empty data, at least 8
patterns and 2 responses, valid suggestion references, configured response
placeholders, at least five fallbacks, and cross-intent near-duplicate
patterns. `--strict` also fails on near-duplicate warnings.

The pytest suite covers every pattern's intent accuracy (95% minimum target),
confusion reporting, entity extraction, context follow-ups, fallback behavior,
placeholder rendering, and API endpoints. Current project tests are run with:

```powershell
.\.venv\Scripts\python.exe scripts/validate_intents.py
.\.venv\Scripts\python.exe -m pytest tests -q
```

## Matching behavior and tuning

The query pipeline is:

1. Lowercase and normalise punctuation, abbreviations, common typos, and slang.
2. Remove stop words and lemmatise (NLTK WordNet if installed; built-in
   rule-based lemmatiser otherwise).
3. Compare the query against intent patterns with TF-IDF unigram/bigram cosine
   similarity.
4. If the best score is below `CONFIDENCE_THRESHOLD`, try rapidfuzz.
5. For short entity-only follow-ups (such as “and for ECE?”), reuse the last
   session intent and update the extracted entity. `follow_up_of` also boosts
   related candidates.
6. If no intent matches, return one of the fallback messages and popular
   topic chips; write the query and best guess to the unmatched log.

Suggested tuning process:

- Many valid questions falling back: add representative patterns first; only
  then try lowering `CONFIDENCE_THRESHOLD` in small steps (for example 0.35 to
  0.32).
- Wrong-topic matches: improve discriminative patterns or raise the threshold
  slightly. Review validator warnings and `unmatched.log`.
- Fuzzy matching accepting unrelated queries: raise `FUZZY_THRESHOLD`.
- Test every change with pytest; do not tune based solely on one example.

## Logs and feedback

Logs are created under `backend/logs/` at runtime:

- `unmatched.log`: JSON Lines records for unmatched and low-confidence queries
  (timestamp, session id, message, best guess, score, reason). Review recurring
  queries and add appropriate patterns after removing any sensitive data.
- `feedback.jsonl`: one JSON Lines record per thumbs-up/down submission,
  including the user query, reply, intent, rating, and timestamp.

Logs may contain user-provided text. Restrict access, set retention limits, and
remove personally identifiable information before using records to improve
patterns. Feedback is stored locally in append-only files; there is no database
or analytics service.

## API reference

Interactive OpenAPI documentation is available at
[http://127.0.0.1:8010/docs](http://127.0.0.1:8010/docs) while the backend is
running.

| Method | Path | Request / response |
| --- | --- | --- |
| `POST` | `/api/chat` | Request: `{ "message": string, "session_id"?: string }`. Response: `reply`, `intent`, `confidence`, `suggestions`, `entities`, and the effective `session_id`. |
| `GET` | `/api/health` | Service status, loaded intent count, config status. |
| `GET` | `/api/intents/stats` | Total intents, pattern count, count per category. |
| `GET` | `/api/config/public` | Public branding values for the frontend. |
| `POST` | `/api/feedback` | `session_id`, original `message`, `reply`, `intent`, and `rating` (`up` or `down`); returns 204. |

The chat endpoint accepts an absent or blank session ID and generates a fresh
one. The frontend stores the returned ID for subsequent turns. Requests with
blank messages, messages over the configured length limit, or invalid feedback
payloads are rejected by API validation.

## Frontend behavior

- Full-browser responsive layout; topic cards appear before the first message.
- Light/dark toggle in the header; initial theme follows the system preference,
  then the choice is persisted in localStorage.
- Chat history and session ID are stored in localStorage. **Clear chat** starts
  a fresh conversation/session in this browser.
- Suggestions appear below the latest bot answer; pressing Enter sends and
  Shift+Enter inserts a newline.
- Bot replies support safe basic markdown (bold, lists, links opened in a new
  tab); user messages are displayed as plain text.
- Feedback buttons send ratings to the backend. Failed chat requests show a
  retry control.
- ARIA labels/live region, visible keyboard focus, character limit feedback,
  and reduced-motion preferences are supported.

## Production checklist

1. Replace every sample/dummy config value with verified institutional data.
2. Review intent response text, especially regulatory, admissions, fees, dates,
   and placement claims; point visitors to the official site for changing data.
3. Set `CORS_ORIGINS` to the deployed frontend origin rather than `*`.
4. Put the app behind HTTPS and a production-grade reverse proxy/process
   manager; configure host firewall and access to the backend appropriately.
5. Add log rotation/retention and a privacy notice before retaining user
   messages or feedback.
6. Run the validator, all tests, and a manual API/UI check after each config or
   knowledge-base update.
