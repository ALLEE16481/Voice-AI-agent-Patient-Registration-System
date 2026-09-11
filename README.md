# 🏥 Voice AI Patient Registration System

**Call a phone number. Speak naturally. Patient registered.**

A conversational voice AI agent that answers a real U.S. phone number, collects patient demographics through natural dialogue (not an IVR menu), reads everything back for confirmation, and persists the data through the same REST API a reviewer can query afterward. Data survives process restarts.

&gt; ⚠️ **Scope:** This is a technical assessment, not a HIPAA production system. **Do not store real patient information.**

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.11+-green.svg)](https://fastapi.tiangolo.com/)
[![Vapi](https://img.shields.io/badge/Voice-Vapi-purple.svg)](https://vapi.ai/)

---

## 📞 Live Demo

| Item | Value |
|---|---|
| **Phone number** | `+1-XXX-XXX-XXXX` *(set after running `python scripts/provision_vapi.py`)* |
| **API base URL** | `https://your-public-host.com` |
| **Dashboard** | `https://your-public-host.com/` |
| **Health check** | `GET https://your-public-host.com/health` |
| **API docs** | `https://your-public-host.com/docs` |

**Example queries after a call:**

```bash
curl https://your-public-host.com/patients?last_name=Doe
curl https://your-public-host.com/patients?phone_number=4155550101
curl https://your-public-host.com/patients/{patient_id}
```

---

## 🏗️ Architecture

```
Caller ──dial──► Vapi (STT + LLM + TTS + U.S. phone number)
                     │  tool-calls / end-of-call-report
                     ▼
               FastAPI webhooks ──same service layer──► SQLite
                     ▲
Reviewer ──HTTP──► REST /patients + dashboard UI
```

### Separation of Concerns

| Layer | Responsibility |
|---|---|
| Telephony / STT / TTS | **Vapi** — we don't build speech engines |
| Conversational policy | `app/vapi/prompt.py` — persona, confirmation, corrections, optional-field offer |
| Tools | `app/vapi/tools.py` — lookup, save, update, appointment slots |
| Domain + validation | `app/validators.py`, `app/schemas.py`, `app/services/` |
| HTTP | `app/routers/` — consistent JSON envelope `{ "data", "error" }` |
| Persistence | SQLAlchemy models in `app/models.py` |

&gt; **Design principle:** the voice agent never writes SQL. After the caller confirms, it calls `save_patient` / `update_patient`, which run the **same validators and service functions** as `POST /patients` and `PUT /patients/:id`.

---

## 🧰 Tech Stack (and Why)

| Choice | Why |
|---|---|
| **Vapi** | Fastest path to a dialable U.S. number plus barge-in, STT, TTS, and LLM tool-calling — spend the time-box on prompt, tools, and backend |
| **GPT-4o** (via Vapi) | Strong at messy spoken intake, corrections, and out-of-order answers |
| **FastAPI** | Typed request validation, clear routers, easy async webhook handlers |
| **SQLite** | Persistent, zero-ops, survives restarts. Swap `DATABASE_URL` for Postgres later — models are already SQLAlchemy |
| **Vanilla dashboard** | No frontend build step; reviewers see records immediately |

---

## 🗃️ Data Model

**Required:** first/last name, DOB (`MM/DD/YYYY`, never future), sex (`Male` / `Female` / `Other` / `Decline to Answer`), U.S. 10-digit phone, street, city, 2-letter state, ZIP or ZIP+4.

**Optional:** email, address line 2, insurance provider + member ID, preferred language (default: English), emergency contact name + phone.

**Auto:** `patient_id` (UUID), `created_at`, `updated_at`.

**DELETE is a soft delete** (`deleted_at`); subsequent GET/PUT on a deleted record return 404.

Two seeded demo records (Jane Doe, Marcus Nguyen) are inserted when the database is empty.

---

## 🔌 REST API

All responses use the envelope:
- Success → `{ "data": ..., "error": null }`
- Failure → `{ "data": null, "error": { "message": "...", "details?": ... } }`

| Method | Path | Status |
|---|---|---|
| `GET` | `/patients` | 200 — optional filters: `?last_name=`, `?date_of_birth=`, `?phone_number=` |
| `GET` | `/patients/{id}` | 200 / 404 |
| `POST` | `/patients` | 201 / 422 |
| `PUT` | `/patients/{id}` | 200 / 400 / 404 / 422 — partial updates allowed |
| `DELETE` | `/patients/{id}` | 200 / 404 — soft delete only |
| `GET` | `/appointments/slots` | 200 — mock availability |
| `POST` | `/appointments` | 201 / 404 / 409 |
| `GET` | `/calls` | 200 — call transcripts from Vapi end-of-call reports |
| `GET` | `/health` | 200 |
| `POST` | `/webhooks/vapi` | Vapi tool calls and call events |

Server-side validation is **independent of the voice agent** — bad phone numbers, future DOBs, invalid states, etc. are rejected at the API layer even if they slip through speech.

---

## 🗣️ Voice Agent Behavior

Full prompt (with commented design notes): [`app/vapi/prompt.py`](app/vapi/prompt.py)

- **Natural intake, not an IVR menu** — extra or out-of-order details are accepted gracefully
- **Specific re-prompts** for invalid values (e.g., a 3-digit phone number)
- **Full read-back + explicit confirmation before any write** — save/update tools also reject calls without `confirmed_by_caller=true`
- **Optional fields are offered once, not interrogated**: insurance, emergency contact, preferred language
- **Duplicate detection**: `lookup_patient_by_phone` → *"It looks like we already have a record for [First Name] [Last Name]. Would you like to update instead?"*
- **Honest failure handling**: save errors are spoken back to the caller — the agent never pretends a write succeeded
- **Graceful close**: *"You're all set, {first name}."* — then optional mock first-appointment booking
- **Spanish**: if the caller prefers Spanish, the prompt switches language (see limitations — STT remains English-primary)
- **Call drop / hangup**: incomplete registrations are never saved; the transcript is still stored in `call_records`
- **"Start over"**: working copy discarded, intake begins again

---

## 🚀 Local Setup

**Requirements:** Python 3.11+, a [Vapi](https://vapi.ai) account (free tier — first U.S. number needs no card).

```bash
git clone &lt;your-repo-url&gt;
cd &lt;repo-name&gt;

python -m venv .venv
# Windows:
.\.venv\Scripts\Activate.ps1
# macOS/Linux:
source .venv/bin/activate

pip install -r requirements.txt
cp .env.example .env        # Windows: copy .env.example .env

python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

&gt; If port 8000 is taken, use `--port 8080`.

| | URL |
|---|---|
| API | http://127.0.0.1:8000 |
| Dashboard | http://127.0.0.1:8000/ |
| Health | http://127.0.0.1:8000/health |
| Swagger docs | http://127.0.0.1:8000/docs |

**Run tests:**

```bash
python -m pytest -q
```

---

## 🔐 Environment Variables

See [`.env.example`](.env.example). Key variables:

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Defaults to SQLite file under `./data/` |
| `PUBLIC_BASE_URL` | Public HTTPS origin Vapi can reach |
| `VAPI_API_KEY` | Vapi dashboard API key |
| `VAPI_WEBHOOK_SECRET` | Random 24+ character secret sent by Vapi as `X-Vapi-Secret` |
| `VAPI_AREA_CODE` | Preferred area code for a free Vapi number |
| `VAPI_ASSISTANT_ID` / `VAPI_PHONE_NUMBER_ID` | Written by the provision script |

&gt; **Never commit API keys.** `.env` is gitignored.

---

## 🌐 Go Live

Reviewers will call the number — the API must be on a public URL.

1. **Start the API** (locally, or on Railway / Render / Fly.io).
2. **If local, expose it:**
   ```bash
   ngrok http 8000
   ```
3. Put the HTTPS origin in `.env` as `PUBLIC_BASE_URL`, set `VAPI_API_KEY`, and replace `VAPI_WEBHOOK_SECRET` with a random value of at least 24 characters.
4. **Provision assistant + inbound number:**
   ```bash
   python scripts/provision_vapi.py
   ```
5. **Dial the printed number.** Register a test patient (fake data only). Confirm with `GET /patients`.

&gt; Vapi calls consume Vapi credits. Add an OpenAI key in the Vapi dashboard if your org doesn't already have LLM credentials.

---

## 🐳 Docker

```bash
docker build -t voiceai .
docker run -p 8000:8000 --env-file .env -v voiceai-data:/app/data voiceai
```

---

## ⚠️ Known Limitations & Trade-offs

- **Not HIPAA.** No BAAs, encryption-at-rest beyond OS defaults, or audit export — assessment scope only.
- **SQLite** is the deliberate shortcut that still persists across restarts. For concurrent production traffic, switch `DATABASE_URL` to Postgres — the models are already SQLAlchemy.
- **Free Vapi numbers** are U.S. inbound. International/outbound requires an imported Twilio (or similar) number.
- **Spanish** is handled in the prompt; Deepgram is configured `en`. Accented English is fine — a full Spanish STT switch was out of scope for the time-box.
- **Name rules** follow the assessment exactly: letters, hyphens, apostrophes.
- **ngrok URLs change** unless you have a reserved domain — re-run the provision script when the public URL changes.
- **Partial registrations are intentionally not written** — a dropped call yields a transcript, not a half-empty row.

---

## 🧭 Next Steps (with More Time)

- Postgres + hosted deploy with a stable hostname
- Deepgram multilingual / language-detect for a real Spanish path
- Idempotent save keyed by `(call_id, phone_number)`
- Auth on the REST API (reviewer token vs open demo)
- Richer appointment calendar instead of mock weekday slots

---

## 📁 Project Layout

```
app/
  main.py              # FastAPI app, envelope error handlers
  models.py            # patients, appointments, call_records
  schemas.py           # request/response models
  validators.py        # shared demographic rules
  routers/             # REST endpoints
  services/            # DB writes used by BOTH REST and Vapi
  vapi/prompt.py       # system prompt + design comments
  vapi/tools.py        # tool schemas + execution
  vapi/webhook.py      # Vapi Server URL handler
  static/              # dashboard
scripts/provision_vapi.py
tests/
```
