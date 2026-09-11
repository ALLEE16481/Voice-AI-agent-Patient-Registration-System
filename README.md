Voice AI agent Patient Registration system
A voice agent answers a real U.S. phone number, collects patient demographics through natural conversation, confirms them, and persists them through the same REST API a reviewer can query later. Data survives process restarts.

This is a take-home assessment, not a HIPAA production system. Do not store real patient information.

Live demo
Fill these in after you provision Vapi (see Go live):

Item	Value
Phone number	set after python scripts/provision_vapi.py
API base URL	public HTTPS origin, e.g. https://<ngrok-or-railway-host>
Dashboard	<API base URL>/
Health	GET <API base URL>/health
Example queries after a call:

GET /patients?last_name=Doe
GET /patients?phone_number=4155550101
GET /patients/{patient_id}
Architecture
Caller ──dial──► Vapi (STT + LLM + TTS + phone number)
                    │  tool-calls / end-of-call-report
                    ▼
              FastAPI webhooks  ──same service layer──► SQLite
                    ▲
Reviewer ──HTTP──► REST /patients  +  dashboard UI
Separation of concerns:

Layer	Responsibility
Telephony / STT / TTS	Vapi. We do not build speech engines.
Conversational policy	app/vapi/prompt.py — persona, confirmation, corrections, optional-field offer.
Tools	app/vapi/tools.py — lookup, save, update, appointment slots.
Domain + validation	app/validators.py, app/schemas.py, app/services/
HTTP	app/routers/ — JSON envelope { "data", "error" }
Persistence	SQLAlchemy models in app/models.py
The voice agent never writes SQL. After the caller confirms, it calls save_patient / update_patient, which run the same validators and service functions as POST /patients and PUT /patients/:id.

Tech stack (and why)
Choice	Why
Vapi	Fastest path to a dialable U.S. number plus barge-in, STT, TTS, and LLM tool calling. Matches the challenge advice to spend time on prompt, tools, and backend.
GPT-4o via Vapi	Strong at messy spoken intake, corrections, and out-of-order answers.
FastAPI	Typed request validation, clear routers, easy webhook handler.
SQLite	Persistent, zero-ops, survives restarts. Swap DATABASE_URL for Postgres later.
Vanilla dashboard	No frontend build step. Reviewers can see records immediately.
Data model
Required: first/last name, DOB (MM/DD/YYYY, not future), sex (Male / Female / Other / Decline to Answer), U.S. 10-digit phone, street, city, 2-letter state, ZIP or ZIP+4.

Optional: email, address line 2, insurance, preferred language (default English), emergency contact.

Auto: patient_id (UUID), created_at, updated_at. DELETE is a soft delete (deleted_at); subsequent GET/PUT return 404.

Seeded demo records (Jane Doe, Marcus Nguyen) are inserted when the database is empty.

REST API
All responses use { "data": ..., "error": null } on success and { "data": null, "error": { "message", "details?" } } on failure.

Method	Path	Status
GET	/patients	200 — optional last_name, date_of_birth, phone_number
GET	/patients/:id	200 / 404
POST	/patients	201 / 422
PUT	/patients/:id	200 / 400 / 404 / 422 — partial updates
DELETE	/patients/:id	200 / 404 — soft delete
GET	/appointments/slots	200
POST	/appointments	201 / 404 / 409
GET	/calls	200 — transcripts from Vapi end-of-call reports
GET	/health	200
POST	/webhooks/vapi	Vapi tool calls and call events
Server-side validation is independent of the voice agent (bad phone, future DOB, invalid state, etc. are rejected).

Voice agent behavior
Prompt (commented design notes + full system message): app/vapi/prompt.py

Natural intake, not an IVR menu. Extra or out-of-order details are accepted.
Invalid values get a specific re-prompt (e.g. a 3-digit phone).
Full read-back and explicit confirmation before any write; the save/update tools also reject calls without confirmed_by_caller=true.
Optional insurance / emergency contact / language are offered once, not interrogated.
Duplicate phone numbers: lookup_patient_by_phone then “Would you like to update instead?”
Save failures are spoken back; the agent does not pretend the write succeeded.
After success: “You’re all set, {first name}.” Optional mock first-appointment booking.
Spanish: if the caller says they prefer Spanish, the prompt switches language (STT remains English-primary — see limitations).
Call drop / hangup: incomplete demographics are not saved. The end-of-call-report still stores the transcript in call_records.
“Start over” is handled in the prompt: discard the working copy and begin again.
Local setup
Requirements: Python 3.11+, a Vapi account for the phone number.

cd C:\Users\I T world\Projects\VoiceAI
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
If port 8000 is already taken, use --port 8080 instead.

API: http://127.0.0.1:8000
Dashboard: http://127.0.0.1:8000/
Health: http://127.0.0.1:8000/health

python -m pytest -q
Environment variables
See .env.example. The important ones:

Variable	Purpose
DATABASE_URL	Default SQLite file under ./data/
PUBLIC_BASE_URL	Public HTTPS origin Vapi can reach
VAPI_API_KEY	Vapi dashboard API key
VAPI_WEBHOOK_SECRET	Random 24+ character secret sent by Vapi as X-Vapi-Secret
VAPI_AREA_CODE	Preferred area code for a free Vapi number
VAPI_ASSISTANT_ID / VAPI_PHONE_NUMBER_ID	Written by the provision script
Never commit API keys. .env is gitignored.

Go live
Reviewers will call the number. The API must be on a public URL.

Start the API (locally or on Railway / Render / Fly).

If local, expose it:

ngrok http 8000
Put the HTTPS origin in .env as PUBLIC_BASE_URL, set VAPI_API_KEY, and replace VAPI_WEBHOOK_SECRET with a random value of at least 24 characters.

Provision assistant + inbound number:

python scripts/provision_vapi.py
Dial the printed number. Register a test patient (fake data only). Confirm with GET /patients.

Vapi’s first free U.S. number does not require a card. Calls consume Vapi credits. Add an OpenAI key in the Vapi dashboard if the org does not already have LLM credentials.

Docker
docker build -t voiceai .
docker run -p 8000:8000 --env-file .env -v voiceai-data:/app/data voiceai
Known limitations and trade-offs
Not HIPAA. No BAAs, encryption-at-rest beyond OS defaults, or audit export. Assessment scope.
SQLite is the shortcut that still persists across restarts. For concurrent production traffic, switch DATABASE_URL to Postgres — the models are already SQLAlchemy.
Free Vapi numbers are U.S. inbound. International / outbound needs an imported Twilio (or similar) number.
Spanish is handled in the prompt; Deepgram is configured en. Accented English is fine; a full Spanish STT switch was out of scope for the time box.
Name rules follow the assessment exactly: letters, hyphens, and apostrophes.
ngrok URLs change unless you have a reserved domain. Re-run the provision script when the public URL changes.
Partial registrations are intentionally not written. A dropped call yields a transcript, not a half-empty row.
Next steps (if more time)
Postgres + hosted deploy with a stable hostname.
Deepgram multilingual / language-detect for a real Spanish path.
Idempotent save keyed by (call_id, phone_number).
Auth on the REST API (token for reviewers vs open demo).
Richer appointment calendar instead of mock weekday slots.
Project layout
app/
  main.py              # FastAPI app, envelope error handlers
  models.py            # patients, appointments, call_records
  schemas.py           # request/response models
  validators.py        # shared demographic rules
  routers/             # REST
  services/            # DB writes used by REST and Vapi
  vapi/prompt.py       # system prompt + design comments
  vapi/tools.py        # tool schemas + execution
  vapi/webhook.py      # Vapi Server URL
  static/              # dashboard
scripts/provision_vapi.py
tests/
