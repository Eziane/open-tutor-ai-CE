# Getting Started — Run Open TutorAI Locally (No Docker)

This guide walks a complete beginner through running Open TutorAI on their own computer,
step by step. Follow every step in order and you will have the app running.

---

## What You Will Have at the End

| What | Where |
|------|-------|
| Frontend (the UI you use) | http://localhost:5173 |
| Backend API | http://localhost:8080 |
| Interactive API docs | http://localhost:8080/docs |

---

## Step 0 — Check Your Computer Has the Right Software

You need two things installed before you start.

### Python 3.11 or 3.12

> **Why not the latest Python?**
> Some libraries this project uses do not yet have pre-built packages for Python 3.13 or 3.14.
> You must use **3.11 or 3.12**.

Check what you have:

```
python --version
```

- If it says `Python 3.11.x` or `Python 3.12.x` → you are good.
- If it says `Python 3.13.x` or `Python 3.14.x` → install 3.11 from https://www.python.org/downloads/release/python-3119/
- If the command is not found → install Python 3.11 from the link above.

**Windows only:** After installing multiple Python versions, you can pick one with:

```
py --list
```

This shows all installed versions. You will use `py -3.11` (not `python`) in the commands below.

### Node.js 18 or newer

Check:

```
node --version
```

- If it says `v18.x.x` or higher → you are good.
- If not found → install from https://nodejs.org (choose the "LTS" version).

---

## Step 1 — Get the Code

If you have not already cloned the repository:

```bash
git clone https://github.com/Open-TutorAi/open-tutor-ai-CE.git
cd open-tutor-ai-CE
```

All the commands below assume you are **inside the `open-tutor-ai-CE` folder**.

---

## Step 2 — Create the Environment File

The app reads its settings from a file called `.env`. Copy the example file to create it:

**Mac / Linux:**
```bash
cp .env.example .env
```

**Windows (PowerShell):**
```powershell
Copy-Item .env.example .env
```

The default settings work fine for local development. You do not need to change anything now.

---

## Step 3 — Create a Python Virtual Environment

A virtual environment keeps this project's Python packages separate from the rest of your computer.

**Mac / Linux:**
```bash
python3.11 -m venv .venv
```

**Windows — if `python --version` already shows 3.11 or 3.12:**
```powershell
python -m venv .venv
```

**Windows — if you have multiple Python versions (use the py launcher):**
```powershell
py -3.11 -m venv .venv
```

You only need to do this once. A `.venv` folder will appear in the project.

---

## Step 4 — Install Python Dependencies

Activate the virtual environment first, then install.

**Mac / Linux:**
```bash
source .venv/bin/activate
pip install -r requirements-ci.txt
pip install requests==2.32.3 aiohttp==3.11.11 async-timeout aiocache alembic==1.14.0 argon2-cffi==23.1.0 APScheduler==3.10.4 RestrictedPython==8.0 openai anthropic tiktoken
```

**Windows (PowerShell):**
```powershell
.venv\Scripts\pip.exe install -r requirements-ci.txt
.venv\Scripts\pip.exe install requests==2.32.3 aiohttp==3.11.11 async-timeout aiocache alembic==1.14.0 argon2-cffi==23.1.0 APScheduler==3.10.4 RestrictedPython==8.0 openai anthropic tiktoken
```

> **Why two install commands?**
> `requirements-ci.txt` has the core packages. The second command adds the runtime packages
> needed to actually run the app (AI clients, task scheduler, etc.).
> We skip `psycopg2` (the PostgreSQL driver) because local development uses SQLite — no
> PostgreSQL needed.

This step takes 2–5 minutes. You will see a lot of text — that is normal.

---

## Step 5 — Create the Runtime Directories

The app stores uploaded files and its database in a `var/` folder. Create it:

**Mac / Linux:**
```bash
mkdir -p var/uploads var/vector_db
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force var\uploads
New-Item -ItemType Directory -Force var\vector_db
```

---

## Step 6 — Install Frontend Dependencies

```bash
cd ui
npm install
cd ..
```

This takes 1–3 minutes and installs about 1000 packages. You may see security warnings —
they are expected and do not affect local development.

---

## Step 7 — Start the App

You need **two terminal windows open at the same time**, both pointing to the project folder.

### Terminal 1 — Backend (Python API)

**Mac / Linux:**
```bash
.venv/bin/uvicorn main:app --reload --port 8080
```

**Windows (PowerShell):**
```powershell
.venv\Scripts\uvicorn.exe main:app --reload --port 8080
```

Wait until you see a line like:
```
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```

### Terminal 2 — Frontend (SvelteKit UI)

```bash
cd ui
npm run dev
```

Wait until you see:
```
  ➜  Local:   http://localhost:5173/
```

---

## Step 8 — Open the App and Create Your Account

Open your browser and go to: **http://localhost:5173**

You will see a signup page. Create an account — **the first account you create automatically
becomes the administrator**.

---

## Step 9 — Add AI Models

The app needs an AI model to work. You have two options:

---

### Option A — Ollama (free, runs on your computer)

**Best for:** offline use, privacy, no API costs.
**Requires:** a reasonably modern computer (8 GB RAM minimum).

1. Download and install Ollama from https://ollama.com/download
2. After installation, open a new terminal and pull a model:

   ```bash
   ollama pull llama3.2
   ```

   This downloads about 2 GB. Other good models: `mistral`, `qwen2.5:3b`.

3. Ollama runs automatically in the background on `http://localhost:11434`.
4. In the app, go to **Admin → Settings → Providers**. You will see `http://localhost:11434`
   already listed. Click **Save**.
5. Go back to the chat — your model will appear in the model selector dropdown.

---

### Option B — Cloud API (OpenAI, GroqCloud, etc.)

**Best for:** powerful models without a high-end computer.
**Requires:** an API key from the provider.

1. In the app, go to **Admin → Settings → Providers**.
2. Toggle **OpenAI** on.
3. Enter the URL and API key for your provider:

   | Provider | Base URL | Where to get a key |
   |----------|----------|--------------------|
   | OpenAI | `https://api.openai.com/v1` | https://platform.openai.com |
   | GroqCloud (free tier) | `https://api.groq.com/openai/v1` | https://console.groq.com |
   | LMStudio (local) | `http://localhost:1234/v1` | No key needed |

4. Click **Save**. Models from that provider appear in the chat selector immediately.

---

## How to Restart the App Next Time

You do not need to repeat Steps 1–6. Just open two terminals and run:

**Terminal 1 (backend):**

```bash
# Mac / Linux
.venv/bin/uvicorn main:app --reload --port 8080

# Windows
.venv\Scripts\uvicorn.exe main:app --reload --port 8080
```

**Terminal 2 (frontend):**

```bash
cd ui
npm run dev
```

---

## Troubleshooting

### "Module not found" or import errors when starting the backend

Make sure you are running uvicorn from inside the project folder (not from inside `ui/` or
any subdirectory), and that you are using `.venv/Scripts/uvicorn.exe` (not a global one).

### The model list is empty

- **Ollama:** Make sure Ollama is running (`ollama list` in a terminal should show your models).
- **OpenAI:** Check that you saved the API key in Admin → Settings → Providers and that the key is valid.

### Port already in use

If you see `address already in use` for port 8080 or 5173, another process is using that port.
Find and stop it, or change the port:

```bash
# Use a different backend port
.venv/bin/uvicorn main:app --reload --port 8081
```

Then update `CORS_ALLOW_ORIGIN` in `.env` to include `http://localhost:5173`.

### Frontend shows "cannot connect to API"

The backend must be running before you use the frontend. Check Terminal 1 is still active and
shows no errors.

### Windows: `py -3.11` not found

Install Python 3.11 from https://www.python.org/downloads/release/python-3119/ and check
"Add Python to PATH" during installation. Then check `py --list`.

---

## Summary of What Each Piece Does

| Piece | What it is | Port |
|-------|-----------|------|
| **Backend** | Python FastAPI server — handles all data, auth, AI calls | 8080 |
| **Frontend** | SvelteKit web app — the interface you see in the browser | 5173 |
| **Ollama** | Optional local AI model server | 11434 |
| **SQLite DB** | Auto-created at `var/tutorai.db` on first run | — |
