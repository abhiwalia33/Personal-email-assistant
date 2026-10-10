# Weeks 1–2 summary

The cohort project is an AI personal assistant for email and calendar work:

```text
Email/calendar provider → Nylas → AI personal assistant → user reviews or approves an action
```

## Week 1: Project setup (no app code)

Week 1 was preparation only. The branch contains just config files: `.env.example`, `.gitignore`, `pyproject.toml`, `docker-compose.yml`, `AGENTS.md`.

- **Tooling:** Python 3.12+, Git, `uv` for the virtual environment (`uv venv`, `uv sync`), Docker Compose (`docker compose up` prints a ready message to prove Docker works).
- **GitHub:** clone the cohort repo, rename its remote to `cohort`, add my own repo as `origin`. That way I can still fetch new teaching branches from `cohort`.
- **Secrets:** `.env.example` holds variable names only. Real values go in `.env`, which `.gitignore` excludes.
- **Nylas homework:** create a developer account and app, and note the client ID, API key and region URI (US `api.us.nylas.com` / EU `api.eu.nylas.com`). Register the hosted-auth callback `http://localhost:5010/oauth/exchange`.

## Week 2: Capture emails and test LLM calls

### Part 1: email capture (`app/`)

```text
New email → Nylas → Serveo tunnel → main.py /events → saved to requests/events/
                                                    → shown at localhost:8000
```

| File | What it does |
|------|--------------|
| `app/config/config_auth.py` | FastAPI helper on port 5010. `/nylas/auth` runs hosted OAuth and returns the **grant ID**. `/nylas/recent-emails` and `/nylas/send-email` check that the API works. |
| `app/config/config_webhook.py` | Registers `SERVER_URL/events` with Nylas for `message.created` and prints the **webhook secret**. Re-run it whenever the tunnel URL changes. |
| `app/main.py` | FastAPI server on port 8000. `GET /events` answers Nylas's challenge. `POST /events` checks the `x-nylas-signature` HMAC-SHA256 against `WEBHOOK_SECRET`, saves the raw event JSON, parses the email and shows it on `/`. |
| `app/schemas/nylas_webhook_schema.py` | `WebhookEvent`: the Pydantic model for the webhook wrapper (`type`, `id`, `time`, `data`, …). |
| `app/schemas/nylas_email_schema.py` | `EmailObject`: the Pydantic model for the email inside `data.object`. |
| `app/templates/index.html` | Jinja2 page listing the emails received. |

Key ideas:
- **Tunnel:** my laptop isn't reachable from the internet, so Serveo (`ssh -R 80:localhost:8000 serveo.net`) gives it a public HTTPS URL. If the URL changes, update `SERVER_URL`, register a new webhook and delete the old one.
- **Signature check:** `main.py` only accepts requests signed with the webhook secret. It reads `.env` once at startup, so restart it after changing `WEBHOOK_SECRET`. Otherwise every request returns 401.
- **Raw events first:** each event is stored as JSON in `requests/events/` before anything else happens. Those files contain real email content and are git-ignored.
- **My fix:** with no `x-nylas-signature` header, `verify_signature` crashed and returned a 500. It now returns 401, and it also rejects requests when `WEBHOOK_SECRET` is empty.

### Part 2: LLM quickstarts (`week2/`)

| File | Shows | Needs |
|------|-------|-------|
| `openai-quickstart.py` | Hosted generative model via the Responses API | `OPENAI_API_KEY` |
| `structured_output.py` | Typed output: JSON Schema, then a Pydantic model | `OPENAI_API_KEY` |
| `jev-quickstart.py` | Decision model (TypeSafe Jev). Returns typed `Choice` / `Score` / `Noul` decisions with probabilities instead of chat text. | `TYPESAFE_API_KEY` |
| `ollama-quickstart.py` | Open-source model running locally | Ollama + `qwen2.5:3b-instruct` |
| `pydantic-introduction.ipynb` | Pydantic basics: field types and validation, then example models for LLM output (sentiment, recommendations, content metadata) | nothing |

Takeaway: there are three kinds of model call. A generative model writes text. Structured output makes it fill a schema. A decision model answers typed questions directly. Pydantic is the common layer that validates the data in each case.

## Every session

1. Start the tunnel and check its URL against `SERVER_URL`.
2. `cd app` then `uv run main.py`.
3. If the URL changed, run `config_webhook.py`, paste the new secret and restart `main.py`.

See `docs/start-new-session.md` and `docs/webhook-troubleshoot-guide.md`.
