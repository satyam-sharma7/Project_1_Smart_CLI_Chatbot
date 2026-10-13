# Smart CLI Chatbot

A small chatbot API built with **FastAPI** that you can talk to from the command line (PowerShell, curl, or any HTTP client). It supports multiple **personas**, keeps **per-session conversation history**, and tracks **token usage** and **context usage** for every chat.

The API is container-ready (Docker) and designed to be hosted on [Render](https://render.com).

---

## Features

- **Three built-in personas**: `professional`, `casual`, and `socratic`
- **Custom system prompt**: set your own persona when starting a session
- **Session memory**: follow-up messages remember earlier conversation (in memory)
- **Usage tracking**: total input/output tokens and context-window percentage returned with each reply
- **Optional API-key protection** via the `X-API-Key` header
- **PowerShell helper** (`chat.ps1`) for chatting straight from the terminal
- **Docker support** for easy deployment

---

## Tech Stack

| Part | Tool |
|------|------|
| Language | Python 3.11 |
| Web framework | FastAPI |
| Server | Uvicorn |
| LLM client | OpenAI Python SDK (pointed at an OpenAI-compatible endpoint) |
| Config | python-dotenv |
| Validation | Pydantic |
| Deployment | Docker, Render |

---

## Project Structure

```
Project_1_Smart_CLI_Chatbot/
├── app.py             # FastAPI app: personas, sessions, and API routes
├── chat.ps1           # PowerShell helper to chat from the terminal
├── dockerfile         # Container build for deployment
├── requirements.txt   # Python dependencies
├── .gitignore
└── README.md
```

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/satyam-sharma7/Project_1_Smart_CLI_Chatbot.git
cd Project_1_Smart_CLI_Chatbot
```

### 2. Install dependencies

```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

pip install -r requirements.txt
```

### 3. Set up environment variables

Create a `.env` file in the project root:

```env
HF_TOKEN=your_token_here
API_KEY=your_optional_shared_secret
```

| Variable | Required | Description |
|----------|----------|-------------|
| `HF_TOKEN` | Yes | Token used to authenticate with the LLM endpoint |
| `API_KEY` | No | If set, every request must send a matching `X-API-Key` header |
| `PORT` | No | Port to run on (defaults to `8000`; Render sets this automatically) |

> Never commit your `.env` file. It is already covered by `.gitignore`.

### 4. Run the server

```bash
python app.py
```

The API will be available at `http://localhost:8000`. Interactive docs are at `http://localhost:8000/docs`.

---

## Using the Chatbot

### From PowerShell (recommended)

```powershell
# One-time setup for this PowerShell session
$BaseUrl = "http://localhost:8000"     # or your deployed Render URL
$ApiKey  = "your-api-key"              # only if you set API_KEY

# Load the helper function
. .\chat.ps1

# Chat
Chat-Bot "Hello, who are you?"
Chat-Bot "Follow-up question"                      # reuses the same session automatically
Chat-Bot "Start fresh" -NewSession -Persona casual # new session with a different persona
```

Each reply prints the bot's answer plus the session ID and context usage.

### With a raw request

```powershell
$body = @{ message = "Hello, testing" } | ConvertTo-Json
Invoke-RestMethod -Uri "http://localhost:8000/chat" -Method Post -ContentType "application/json" -Body $body
```

### With curl

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{"message": "Explain recursion simply", "persona": "socratic"}'
```

---

## Personas

| Persona | Behaviour |
|---------|-----------|
| `professional` | Concise, formal, and precise (default) |
| `casual` | Friendly and relaxed, short sentences |
| `socratic` | Never gives direct answers; asks guiding questions instead |

To use your own behaviour, send a `custom_prompt` when starting a new session. It overrides the persona.

> The persona or custom prompt is applied only when a **new** session is created.

---

## API Reference

### `GET /`
Returns a plain-text status page with example usage and system info.

### `GET /health`
Health check.

```json
{ "status": "ok", "model": "auto", "active_sessions": 2 }
```

### `POST /chat`
Send a message and get a reply.

**Request body**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | Yes | Your message |
| `session_id` | string | No | Omit to start a new session |
| `persona` | string | No | `professional` (default), `casual`, or `socratic` |
| `custom_prompt` | string | No | Custom system prompt (new sessions only) |

**Response**

```json
{
  "session_id": "b1f2c3d4-...",
  "reply": "Hello! How can I help you today?",
  "persona": "professional",
  "context_pct": 0.4,
  "total_input_tokens": 25,
  "total_output_tokens": 12
}
```

### `GET /session/{session_id}`
Returns the full session: persona, messages, token counts, and creation time.

### `DELETE /session/{session_id}`
Deletes a session.

**Errors**

| Code | Meaning |
|------|---------|
| `401` | Missing or invalid `X-API-Key` |
| `404` | Session not found |
| `500` | `HF_TOKEN` is not configured on the server |
| `502` | The upstream model returned an error |

---

## Docker

```bash
docker build -t smart-cli-chatbot -f dockerfile .
docker run -p 8000:8000 --env-file .env smart-cli-chatbot
```

---

## Deploying to Render

1. Push this repo to GitHub.
2. In Render, create a new **Web Service** and connect the repo.
3. Choose **Docker** as the runtime (Render will use the `dockerfile`).
4. Add environment variables: `HF_TOKEN` (and optionally `API_KEY`).
5. Deploy. Render sets `PORT` automatically.

Then point `$BaseUrl` in PowerShell at your Render URL.

---

## Limitations

- **Sessions are stored in memory.** They are lost whenever the server restarts or spins down (this happens on Render's free tier). Swap in a database such as PostgreSQL if you need persistent history.
- **Context usage is an estimate** (about 4 characters per token), so `context_pct` is approximate. The context limit is set to 10,000 tokens.
- **Old messages are not trimmed**, so very long sessions will eventually exceed the model's context window.
- **CORS is open** (`*`) for convenience. Restrict `allow_origins` in `app.py` before using this in production.

---

## Future Improvements

- Persistent storage for sessions
- Automatic trimming or summarising of long conversations
- Streaming responses
- A cross-platform Python or Bash CLI client
- Unit tests and CI

---

## Author

**Satyam Sharma** — [@satyam-sharma7](https://github.com/satyam-sharma7)
