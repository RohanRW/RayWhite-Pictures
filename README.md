# raywhite-ai

Real-estate assistant for Ray White Ltd. One FastAPI backend serving three
channels, one AI pipeline, one database.

```
WordPress Website ───────► /api/chat ───────────────┐
                                                    │
Facebook Messenger ──────► /webhook/facebook ───────┤
                                                    ▼
WhatsApp Cloud API ──────► /webhook/whatsapp    FastAPI Backend
                                                    │
                                                    ├── AI / RAG
                                                    ├── Business Logic
                                                    └── PostgreSQL
                                                          │
                                                          ├── Contacts (users)
                                                          ├── Conversations
                                                          ├── Messages
                                                          └── Leads
```

## Layout

```
raywhite-ai/
├── main.py                  FastAPI app: CORS, router wiring, /health, legacy /webhook
├── requirements.txt
├── .env                     all secrets and settings (never committed)
│
├── app/
│   ├── config.py            every env var, read once into `settings`
│   ├── database.py          engine + session_scope(); no-ops when DATABASE_URL is empty
│   │
│   ├── routers/             transport only - parse, verify, queue, respond
│   │   ├── website.py       POST /api/chat  (+ /chat, the path the live widget uses)
│   │   ├── facebook.py      GET/POST /webhook/facebook
│   │   └── whatsapp.py      GET/POST /webhook/whatsapp
│   │
│   ├── services/            all the behaviour, channel-independent
│   │   ├── ai.py            router agent -> specialist agent -> reply, plus image vision
│   │   ├── rag.py           Google-Sheet knowledge base + retrieval for the apartment agent
│   │   ├── memory.py        SessionRef, sliding-window history, transcript persistence
│   │   ├── leads.py         phone/email capture out of incoming messages
│   │   ├── meta.py          shared webhook plumbing: verify, signature, de-dupe, ad filter
│   │   ├── facebook.py      Messenger Send API client + payload parser
│   │   └── whatsapp.py      WhatsApp Cloud API client + payload parser
│   │
│   ├── models/              SQLAlchemy tables
│   │   ├── base.py          Base, TimestampMixin, Channel/LeadStatus enums
│   │   ├── contact.py       contacts   - one row per person per channel
│   │   ├── conversation.py  conversations - one thread, keyed by session_key
│   │   ├── message.py       messages   - one row per inbound message + its reply
│   │   └── lead.py          leads      - contacts who left a phone or email
│   │
│   └── prompts/             the agent prompts (router, apartments, conversational,
│                            land_price, vision)
```

The rule: **routers never call OpenAI or the database directly** - they build a
`SessionRef` and hand it to a service. That is what makes a fourth channel a
new file in `routers/` and nothing else.

## Request flow

1. A message arrives on one of the three routers.
2. Meta channels verify the signature, drop duplicate `mid`s and absorb ad
   ice-breakers, then work in a `BackgroundTask` - Meta gets its 200 in
   milliseconds and never retries.
3. `ai.reply()` loads the last 10 turns from `messages`, asks the router agent
   for a category, grounds the apartment agent with the rows `rag.retrieve()`
   picked for this question, and answers.
4. The exchange is written to `messages`; any phone or email in it becomes a
   `lead`.

## The tables

`messages` is the one you will read most. **One row per inbound message**, with
the bot's answer on the same row:

| column                | what it holds                                                    |
|-----------------------|------------------------------------------------------------------|
| `id`                  | primary key                                                       |
| `conversation_id`     | FK to `conversations`                                             |
| `channel`             | `website` / `facebook` / `whatsapp`                               |
| `sender_id`           | PSID (Messenger), phone (WhatsApp), session id (website)          |
| `user_message`        | what the customer typed - NULL if they sent only a picture        |
| `image_url`           | the picture they sent, NULL otherwise                             |
| `ai_response`         | the bot's reply - **NULL when no reply was sent**                 |
| `category`            | which agent answered: apartments / conversational / land_…_price  |
| `provider_message_id` | Meta's `mid` / `wamid`, used to drop webhook retries              |
| `is_context`          | true for context the bot injected itself (ad campaign notes)      |
| `created_at`          | when it arrived                                                   |

`ai_response` is NULL in three cases: a picture the bot only looked at, an ad
ice-breaker it absorbed, and any channel switched off in `AUTO_REPLY_CHANNELS`.

The other three:

- **`contacts`** - one row per person per channel (`channel` + `external_id`
  unique). Collects `name`, `phone`, `email` as they come up in chat.
- **`conversations`** - one thread per contact, keyed by `session_key`
  (`fb:<psid>`, `wa:<phone>`, `web:<session id>`), plus `last_category` and
  `last_message_at`.
- **`leads`** - a contact who left a phone number or email, with `interest` and
  a `status` the sales team can move along (`new` → `contacted` → `qualified`).

Useful queries:

```sql
-- everything one customer said and heard, in order
SELECT created_at, user_message, image_url, ai_response
FROM messages WHERE sender_id = '8801711223344' ORDER BY created_at;

-- Messenger messages still waiting for a human answer
SELECT created_at, sender_id, user_message FROM messages
WHERE channel = 'facebook' AND ai_response IS NULL AND NOT is_context
ORDER BY created_at DESC;

-- today's leads
SELECT channel, name, phone, email, interest FROM leads
WHERE created_at::date = CURRENT_DATE;
```

Enum-ish columns (`channel`, `status`) are plain `VARCHAR`, not PostgreSQL
`ENUM` types - a native type outlives its table, collides on the next create,
and needs an `ALTER TYPE` for every new value. Adding a fourth channel is a
code-only change.

## Which channels answer automatically

```bash
AUTO_REPLY_CHANNELS=website,whatsapp    # facebook is logged, not answered
AUTO_REPLY_CHANNELS=website,whatsapp,facebook   # bot answers Messenger too
```

A channel left out is still recorded in full - contact, conversation, message,
lead. It simply gets no AI reply, so a person can answer from the Page inbox.

## Running it

```bash
python -m venv venv && venv/Scripts/activate     # source venv/bin/activate on Linux
pip install -r requirements.txt
# fill in .env, then:
python -m app.database                           # create the tables
uvicorn main:app --reload
```

`python -m app.database` is optional - the app also creates missing tables on
startup (`AUTO_CREATE_TABLES=true`). Running it by hand just gives you the
error message directly if the connection is wrong.

`DATABASE_URL` is optional too: leave it empty and the bot runs exactly as it
did before persistence existed (a 10-message in-process window, no transcripts,
no leads). `/health` reports which mode you are in.

Paste the URL exactly as your host gives it - `postgres://`, `postgresql://` and
`postgresql+psycopg://` all work, because `config.py` rewrites the prefix onto
the psycopg 3 driver this project installs.

> **Render:** use the **External** Database URL when connecting from your
> laptop. The internal one (`dpg-xxxx-a` with no domain) only resolves from
> inside Render's own network and fails locally with
> `failed to resolve host … getaddrinfo failed`.

## Changing the schema

There is no migration tool. `create_all()` creates missing tables and indexes;
it will **not** alter a table that already exists. So:

- **New table or index** - just add the model and restart.
- **New column on a live table** - add it to the model, then apply it once by
  hand:
  ```sql
  ALTER TABLE messages ADD COLUMN IF NOT EXISTS whatever TEXT;
  ```
- **Throwing the data away** - `DROP TABLE messages, conversations, leads,
  contacts;` and restart.

If you later want migrations back, `pip install alembic && alembic init alembic`
and point `target_metadata` at `app.models.Base.metadata`.

## Webhook URLs in the Meta dashboard

Point Messenger at `https://<host>/webhook/facebook` and WhatsApp at
`https://<host>/webhook/whatsapp`. The old combined `https://<host>/webhook`
still works and dispatches by payload type, so the switch can happen whenever it
suits.
