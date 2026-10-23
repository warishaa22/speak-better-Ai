# SpeakBetter AI

A personal AI communication coach built with Streamlit, Google Gemini and SQLite. Users practise
conversations, interviews, presentations and workplace English, get structured feedback on every
answer, retry the same question, and track recurring weaknesses over time.

## Features

| Area | What it does |
|---|---|
| Accounts | Register, log in, log out, profile (name, goals, level, explanation language incl. optional Urdu) |
| 8 practice modes | Everyday, interview, presentation, workplace, networking, group discussion, vocabulary, situational |
| Conversation simulator | Gemini plays a partner/interviewer, one question at a time, keeps context, adapts to level, ends with a review |
| Feedback | 7-criterion rubric (1-10), strengths, improvements, corrections, improved answer, explanation, practice tip |
| Try Again | Re-answer the same question; compare scores (computed in Python) plus AI evidence from the actual texts |
| Voice practice | Record or upload audio, transcribe with Gemini, edit transcript, then get feedback |
| Weakness tracker | Counts weakness tags across answers; "recurring" only after 3+ answers, otherwise "occasional" |
| Dashboard / Progress | Sessions, practice time, streaks, score trend, skill averages, most improved, next exercise |
| Daily challenge | One short challenge per day (UTC), personalised to recurring weaknesses |
| History & privacy | Search, delete sessions, clear history, delete account; audio is never stored |

## Project structure

```text
speakbetter-ai/
├── app.py                  # entry point: setup, login gate, sidebar navigation
├── requirements.txt
├── .streamlit/
│   ├── config.toml         # lavender theme
│   └── secrets.toml.example
├── modules/
│   ├── config.py           # model name, limits, rubric, labels, settings reader
│   ├── database.py         # SQLite schema + all queries (every query is user-scoped)
│   ├── auth.py             # scrypt password hashing, register/login
│   ├── ai_coach.py         # ALL Gemini calls, JSON parsing + validation, error mapping
│   ├── conversation.py     # practice modes, simulator prompts, session start/reply/finish
│   ├── speech.py           # audio validation, transcription, measured pace/filler words
│   ├── analytics.py        # Pandas: streaks, trends, weakness patterns, recommendations
│   ├── charts.py           # Plotly figures
│   ├── ui.py               # CSS and shared feedback widgets
│   └── views.py            # one function per Streamlit page
└── tests/                  # 73 unit tests, no network or API key needed
```

Layers: `views.py` (UI) calls `conversation.py`/`ai_coach.py`/`speech.py`/`analytics.py` (logic), which use
`database.py` (storage). Only `ai_coach.py` and `speech.py` talk to Gemini, and only `database.py` runs SQL.

## Database schema

- `users` - username, password_hash, display_name, goals, level, explain_lang
- `sessions` - user_id, mode, difficulty, scenario, planned_turns, status, started_at, ended_at, review_json
- `messages` - session_id, role (assistant/user), content
- `attempts` - user_id, session_id (nullable), parent_attempt_id, attempt_no, source (text/voice), kind (practice/challenge), question, answer, audio_seconds, wpm, filler_count, feedback_json, 7 score columns, overall, weakness_tags
- `comparisons` - first_attempt_id, second_attempt_id, comparison_json
- `daily_challenges` - one row per user per day

All child tables use `ON DELETE CASCADE`, so deleting a session or account removes the related records.

## Setup

1. Install Python 3.10 or newer.
2. Create and activate a virtual environment, then install the dependencies:
   ```bash
   python -m venv .venv
   source .venv/bin/activate        # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```
3. Get a Gemini API key: open <https://aistudio.google.com/apikey>, sign in with a Google account, click
   **Create API key**, and copy it.
4. Configure secrets:
   ```bash
   cp .streamlit/secrets.toml.example .streamlit/secrets.toml
   ```
   Open `.streamlit/secrets.toml` and paste your key as `GEMINI_API_KEY`. Environment variables with the
   same names also work.
5. Run: `streamlit run app.py`

The database file and tables are created automatically on first launch.

### Choosing the model
The model name is `GEMINI_MODEL` (default in `modules/config.py`). Model names change often. If you see
"model was not found", open Google AI Studio, copy a current Flash model name, and put it in secrets.
If JSON answers are cut off on a "thinking" model, raise `MAX_OUTPUT_TOKENS` in `modules/config.py`.

## Running the tests
```bash
python -m unittest discover -s tests -t .
```
(`pytest` also works if installed.) Tests use a fake AI function, so they need no API key or internet.

## Deploying to Streamlit Community Cloud

1. Create a GitHub repository and push the project:
   ```bash
   git init
   git add .
   git status                      # confirm secrets.toml and *.db are NOT listed
   git commit -m "SpeakBetter AI"
   git branch -M main
   git remote add origin https://github.com/YOUR-USERNAME/speakbetter-ai.git
   git push -u origin main
   ```
2. Go to <https://share.streamlit.io>, sign in with GitHub, click **Create app**, choose the repository,
   branch `main` and main file `app.py`.
3. Open **Advanced settings → Secrets** and paste the contents of your `secrets.toml`
   (at least `GEMINI_API_KEY`). Click **Deploy**.
4. If the app shows a missing-package error, check that `requirements.txt` is in the repository root.

## Important limitations

- **SQLite on Streamlit Community Cloud is not durable.** The app's file system is ephemeral: when the app
  restarts or redeploys, `speakbetter.db` can be reset and users would lose accounts and history. Fine for a
  demo or portfolio; not for real users.
- **For real multi-user use**, move to a hosted PostgreSQL database (for example Supabase, Neon or Google
  Cloud SQL). Only `modules/database.py` needs to change because it is the only file that runs SQL.
- SQLite does not encrypt data. Passwords are hashed, but answers and transcripts are stored as plain text.
- Login rate-limiting is per browser session only, which slows casual guessing but is not real protection.
- Days (streaks, daily challenge) use UTC, not the user's local time.
- Scores are AI estimates against a fixed rubric. Text cannot measure pronunciation, vocal confidence or
  body language, and the app does not claim to. Pace and filler-word counts are measured from the
  transcript/audio length; transcription may omit some fillers, and pauses are not measured.
- Audio goes to Google Gemini for transcription and is never saved by this app. Do not use real private
  information in practice answers.
- `st.audio_input` needs a recent Streamlit and browser microphone permission. File upload is the fallback.

## Security notes
- Passwords: scrypt with a random salt per user, constant-time comparison, same error for wrong user or password.
- All SQL is parameterised; every query includes the signed-in `user_id`.
- User text is treated as data: delimiter tags are stripped, input is length-limited, and the system prompt tells
  the model not to follow instructions inside learner text. This reduces, but cannot fully remove, prompt-injection risk.
- Users only see friendly errors, never API keys, prompts or stack traces.
- This app has not had a professional security review.
