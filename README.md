# Resume Bot

An AI-powered Telegram bot that acts as a personal resume manager. It parses your existing resume, maintains a persistent structured profile in a database, lets you update it through natural conversation, and generates ATS-friendly PDF resumes and cover letters on demand — including tailored versions optimized for specific job descriptions.

---

## Overview

Resume Bot is built as an n8n automation workflow. It connects a Telegram interface to a Supabase database and an Ollama-served large language model, orchestrating a full pipeline from raw resume input to compiled PDF output.

The core idea is that most people maintain a single static resume document. This bot replaces that with a living, structured profile stored in a database. You can update it conversationally, view it at any time, and export it as a professionally formatted PDF. When applying for a specific role, you can provide the job description and the bot will generate a temporarily optimized version of your resume without touching your original data.

---

## Features

### Resume Parsing
Drop a PDF into the Telegram chat. The bot extracts the text, passes it to an AI agent, and maps the unstructured content into a structured profile schema. The parsed profile is saved to your Supabase database and becomes your permanent Master Profile.

### Conversational Profile Updates
You do not need commands to update your profile. Send a plain message like "I just completed a machine learning course" or "Add my new frontend internship at Acme Corp" and the AI agent will detect the new information, update the relevant fields in your profile, and confirm what changed.

### Job Description Optimization
Send `/optimize` followed by a full job description. The bot performs a gap analysis between your master profile and the requirements of the role, rewrites your summary and bullet points to emphasize relevant experience, reorders your skills to prioritize JD-relevant categories, and saves the result as a temporary Tailored Profile. Your master profile is never modified during this process.

### PDF Export
Request a PDF at any time. The bot generates a LaTeX document from your profile, compiles it on a remote server via SSH, and sends you the resulting PDF. If compilation fails due to special character escaping issues, the bot automatically invokes a secondary AI agent to fix the LaTeX and retries the compilation.

### Cover Letter Generation
Send `/coverletter` after running `/optimize`. The bot uses your tailored profile to generate a structured, professional cover letter, compiles it as a PDF, and sends it directly to the chat.

### Profile Preview
View your full profile in formatted text directly in the Telegram chat before exporting, for both the master and tailored versions.

---

## Commands

| Command | Description |
|---|---|
| `/info` | Display the help menu and command list |
| `/view` | Show your master profile as formatted text in chat |
| `/download` | Compile and download your master profile as a PDF |
| `/optimize [job description]` | Analyze the JD and generate a tailored profile |
| `/viewOptimized` | Show your current tailored profile as formatted text |
| `/downloadOptimized` | Compile and download your tailored resume as a PDF |
| `/coverletter` | Generate and download a tailored cover letter PDF |

Any other message (including a PDF upload) is treated as a conversational update to your profile.

---

## Architecture

The bot is implemented as a single n8n workflow exported as `ResumeBot.json`. The workflow consists of the following major components:

### Trigger and Routing
- **Telegram Trigger**: Listens for incoming messages and callback queries (inline button presses).
- **Supabase Fetch**: On every message, fetches the user's profile row from Supabase using their Telegram chat ID.
- **Command Router**: A Switch node that inspects the incoming message or callback data and routes execution to the appropriate branch.

### Profile Branches
- **PDF Upload Branch**: Detects when a document is attached, extracts PDF text, formats a prompt, and sends it to the AI Agent for parsing.
- **Chat Branch**: Handles all plain text messages. Sends them directly to the AI Agent for conversational profile updating.
- **Info Branch**: Returns the help message with the command menu.
- **View Branch**: Formats and returns the master profile as Telegram-formatted text.

### AI Agents (Ollama)
All AI inference runs locally via Ollama using the `qwen3-coder:480b` model served through an OpenAI-compatible API endpoint.

- **AI Agent** (main): Handles both PDF parsing and conversational updates. Given the current profile and the user's message, it returns a JSON object containing an updated `master_profile` and a natural language `chat_reply`. It uses a windowed buffer memory keyed to the user's chat ID for conversation continuity.
- **AI JD Agent**: Receives the user's master profile and a job description. Returns a `tailored_profile` with rewritten content and a `chat_reply` summarizing optimizations and skill gaps.
- **AI Fix LaTeX Agent**: When pdflatex compilation fails, this agent receives the compile log and the original LaTeX source, fixes any errors, and returns corrected LaTeX.
- **AI Cover Letter Agent**: Receives the tailored profile and generates a structured cover letter as a JSON object with distinct paragraphs and bullet points.

### Database (Supabase)
A Supabase `Profiles` table stores one row per user, keyed by `telegram_id`. Each row holds:
- `telegram_id`: The user's Telegram chat ID (used as the primary key for lookups)
- `master_profile`: JSON blob containing the full structured resume profile
- `tailored_profile`: JSON blob containing the most recent JD-optimized profile

The bot performs an upsert pattern: it checks whether a row exists for the user and issues either an UPDATE or an INSERT accordingly.

### LaTeX and PDF Compilation
Profile data is converted to a LaTeX document using a custom template with precise resume formatting. The `.tex` file is sent to a remote Linux server via SSH, compiled with `pdflatex`, and the resulting PDF is base64-encoded and returned to n8n. The PDF binary is then sent to the user via the Telegram `sendDocument` API.

The resume template uses standard LaTeX packages (`tabularx`, `enumitem`, `hyperref`, `tcolorbox`, `titlesec`) and defines custom commands for consistent section and entry formatting.

---

## Data Model

The `master_profile` and `tailored_profile` fields follow this JSON schema:

```json
{
  "name": "string",
  "tagline": "string",
  "contact": {
    "email": "string",
    "phone": "string",
    "linkedin_url": "string",
    "linkedin_display": "string",
    "github_url": "string",
    "github_display": "string",
    "portfolio_url": "string",
    "portfolio_display": "string"
  },
  "summary": "string",
  "skills": [
    { "category": "string", "items": "string" }
  ],
  "work_experience": [
    {
      "title": "string",
      "company": "string",
      "location": "string",
      "start_date": "string",
      "end_date": "string",
      "bullet_points": ["string"]
    }
  ],
  "education": [
    {
      "degree": "string",
      "institution": "string",
      "score": "string",
      "year": "string"
    }
  ],
  "projects": [
    {
      "name": "string",
      "tech_stack": "string",
      "duration": "string",
      "link_url": "string",
      "link_display": "string",
      "bullet_points": ["string"]
    }
  ],
  "certifications": [
    {
      "name": "string",
      "issuer": "string",
      "year": "string"
    }
  ]
}
```

---

## Prerequisites

Before setting up and importing this workflow, ensure the following are available and configured:

- **n8n** (self-hosted or cloud) — version compatible with `typeVersion` 1.x nodes and the LangChain integration nodes
- **Telegram Bot** — created via BotFather; you will need the bot token
- **Supabase project** — with a `Profiles` table containing columns: `telegram_id` (text, primary key), `master_profile` (jsonb), `tailored_profile` (jsonb)
- **Ollama** — running locally or on a server accessible from n8n, with the `qwen3-coder:480b` model pulled. Ollama must be accessible via an OpenAI-compatible API endpoint (default: `http://localhost:11434/v1`)
- **Remote Linux server with pdflatex** — accessible via SSH with password authentication. Must have `texlive-full` or equivalent packages installed, including `latexmk`, `pdflatex`, and all packages used in the template

---

## Setup

### 1. Prepare Supabase

Create a new Supabase project and run the following SQL to create the profiles table:

```sql
CREATE TABLE "Profiles" (
  telegram_id TEXT PRIMARY KEY,
  master_profile JSONB,
  tailored_profile JSONB
);
```

Enable Row Level Security policies as appropriate for your deployment.

### 2. Set Up Ollama

Install Ollama on your server and pull the required model:

```bash
ollama pull qwen3-coder:480b
```

Ensure the Ollama API is accessible from your n8n instance. If Ollama and n8n are on different hosts, expose the API with:

```bash
OLLAMA_HOST=0.0.0.0 ollama serve
```

### 3. Set Up the PDF Compilation Server

On a Linux server, install the required LaTeX packages:

```bash
sudo apt-get install texlive-full
```

Ensure SSH password authentication is enabled and the n8n server can reach this host.

### 4. Configure n8n Credentials

In your n8n instance, create the following credentials:

**Telegram API** (name: `AI Agent Bot`)
- Credential type: Telegram API
- Access Token: your bot token from BotFather

**Supabase API** (name: `Resume Bot`)
- Credential type: Supabase API
- Host: your Supabase project URL
- Service Role Secret: your Supabase service role key

**OpenAI API** (name: `Ollama API`)
- Credential type: OpenAI API
- API Key: any non-empty string (Ollama does not require authentication)
- Base URL: your Ollama server's OpenAI-compatible endpoint (e.g., `http://localhost:11434/v1`)

**SSH Password** (name: `SSH Password account`)
- Credential type: SSH
- Host: your LaTeX compilation server's IP or hostname
- Port: 22 (or your configured SSH port)
- Username: the SSH user
- Password: the SSH password

### 5. Import the Workflow

1. Open your n8n instance and navigate to Workflows.
2. Click "Import from file" and select `ResumeBot.json`.
3. After import, open each credential reference in the nodes and map them to the credentials created in step 4. The credential IDs embedded in the JSON are environment-specific and will need to be re-linked.
4. Activate the workflow.

### 6. Configure the Telegram Webhook

n8n automatically registers the webhook with Telegram when the workflow is activated, provided your n8n instance is reachable from the internet. Ensure:
- n8n is accessible via HTTPS on a public URL
- The Telegram Trigger node has the correct bot token credential

---

## How the Optimize Command Works

The `/optimize` command implements a non-destructive tailoring pattern:

1. The user sends `/optimize` followed by the full job description text in a single message.
2. The bot extracts the JD text, fetches the user's master profile from Supabase, and builds a structured prompt.
3. The AI JD Agent analyzes the gap between the profile and the JD requirements. It is instructed to never fabricate any data. Specifically, it must copy `name`, `contact`, `education`, and `certifications` fields exactly from the master profile.
4. It may only rewrite the `summary`, `tagline`, the order of `skills` categories, and the phrasing of `work_experience` and `projects` bullet points.
5. The resulting tailored profile is saved to the `tailored_profile` column in Supabase, separate from the master profile.
6. The user can view or download the tailored version. Running `/optimize` again with a different JD will overwrite the previous tailored profile.

---

## Error Handling

- **LaTeX Compilation Failure**: If `pdflatex` returns a non-zero exit code, the workflow captures the compile log and invokes the AI Fix LaTeX Agent to identify and correct the error. The corrected LaTeX is then compiled in a second attempt. If the retry also fails, the user receives an error message.
- **Missing Profile**: If a user attempts to view, download, or optimize without an existing profile, they receive a prompt explaining that they need to upload their resume PDF first.
- **JSON Parse Failures**: AI agent outputs are parsed defensively. If the output cannot be parsed as JSON, a fallback response is returned to the user.
- **Cover Letter Without Optimization**: If the user requests a cover letter without first running `/optimize`, the bot detects the missing tailored profile and instructs the user to run `/optimize` first.

---

## Notes

- The bot currently supports one active tailored profile per user at a time. Each call to `/optimize` replaces the previous tailored profile.
- Profile updates via natural conversation use a windowed buffer memory, so the AI retains context across the current session.
- All PDF compilation happens on the remote SSH server. The compiled PDF files are stored temporarily in `/tmp` and are not cleaned up automatically between sessions (only cover letter files are removed after being read). You may want to add a cron-based cleanup job on the compilation server.
- The `qwen3-coder:480b` model is used for all AI tasks. You can substitute a different model by updating the model name in the three Ollama nodes within the workflow.
- Credential IDs in the exported JSON are instance-specific. After importing into a new n8n instance, all credential references must be re-linked manually.
