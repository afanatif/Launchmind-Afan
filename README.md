# LaunchMind 🚀

**An autonomous Multi-Agent System that takes a startup idea from concept to launch — writing code, opening GitHub pull requests, posting on Slack, and sending emails — without any human doing it manually.**

---

## Startup Idea

LaunchMind is **idea-agnostic** — when you run `python main.py`, the system prompts you to enter any startup idea you want:

```
Enter your startup idea: _
```

The CEO agent then uses an LLM to decompose your idea into tasks for each sub-agent automatically. No hardcoding, no pre-defined startup.

**Example idea used during development:**
> *Safeline — A women-only ride-sharing platform that ensures safe, vetted transportation through rigorous driver screening, real-time trip sharing, and emergency SOS features. Targeted at working women, students, and solo travellers who face safety concerns with existing ride-share services.*

---

## Agent Architecture

```
                        ┌─────────────────────┐
        startup idea ──▶│    CEO Agent         │◀── revision feedback loops
                        │  (Orchestrator)      │
                        └──────────┬──────────┘
                                   │ task messages (JSON)
               ┌───────────────────┼───────────────────┐
               ▼                   ▼                   ▼
      ┌─────────────────┐ ┌────────────────┐ ┌──────────────────┐
      │  Product Agent  │ │Engineer Agent  │ │ Marketing Agent  │
      │                 │ │                │ │                  │
      │ • Value prop    │ │ • HTML landing │ │ • Tagline/copy   │
      │ • Personas      │ │   page (Gemini)│ │ • Cold email     │
      │ • Features      │ │ • GitHub issue │ │   (Brevo)        │
      │ • User stories  │ │ • Git branch   │ │ • Slack Block Kit│
      │                 │ │ • PR opened    │ │   message        │
      └────────┬────────┘ └───────┬────────┘ └────────┬─────────┘
               │                  │                    │
               └──────────────────┴────────────────────┘
                        result messages back to CEO
```

### Communication Flow
1. **CEO → Product**: task — decompose idea into spec
2. **Product → CEO**: result — product specification JSON
3. **CEO reviews spec via LLM** — sends `revision_request` if not good enough (up to 2 cycles)
4. **CEO → Engineer**: task — build landing page + GitHub actions
5. **CEO → Marketing**: task — generate copy, send email, post to Slack
6. **Engineer → CEO**: result — PR URL + issue URL
7. **Marketing → CEO**: result — all generated copy
8. **CEO → CEO**: confirmation — final Slack summary posted

Every message follows the structured schema defined in `message_bus.py`.

---

## Setup Instructions

### Prerequisites

- Python 3.11+
- A GitHub account with a public repository
- A Slack workspace with a bot app created at [api.slack.com/apps](https://api.slack.com/apps)
- A Brevo account at [brevo.com](https://brevo.com) (free tier: 300 emails/day)
- A Google Gemini API key at [aistudio.google.com](https://aistudio.google.com)

### 1. Clone the repository

```bash
git clone https://github.com/afanatif/Launchmind-Afan.git
cd Launchmind-Afan
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in all values:

| Variable | Where to get it |
|---|---|
| `GEMINI_API_KEY` | [aistudio.google.com](https://aistudio.google.com) → Get API Key |
| `HF_API_KEY` | [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) |
| `GITHUB_TOKEN` | GitHub → Settings → Developer Settings → Personal Access Tokens (Classic) → `repo` scope |
| `GITHUB_REPO` | `your-username/your-repo-name` |
| `SLACK_BOT_TOKEN` | Slack App → OAuth & Permissions → Bot User OAuth Token (`xoxb-...`) |
| `BREVO_API_KEY` | Brevo → Settings → API Keys |
| `BREVO_FROM_EMAIL` | A verified sender email in your Brevo account |
| `TEST_EMAIL` | Any inbox you control (cold outreach goes here) |

### 4. Slack bot setup

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → your app → **OAuth & Permissions**
2. Add bot token scopes: `chat:write`, `chat:write.public`, `channels:read`, `groups:read`
3. Click **Reinstall to Workspace**
4. Create a `#launches` channel in your workspace
5. Type `/invite @YourBotName` inside `#launches`

### 5. Run the system

```bash
python main.py
```

Enter your startup idea when prompted. The full pipeline runs automatically.

---

## Platform Integrations

| Platform | What the agent does |
|---|---|
| **GitHub** | Engineer agent creates a branch, commits a full HTML landing page (`index.html`), opens a GitHub issue with an LLM-generated description, and opens a pull request — all via the GitHub REST API |
| **Slack** | Marketing agent posts a Block Kit formatted launch card (with tagline, description, PR link, status) to the workspace channel. CEO agent posts a final summary after all agents complete. |
| **Brevo (Email)** | Marketing agent sends a cold-outreach email with an LLM-generated subject line and body to the configured `TEST_EMAIL` address via the Brevo REST API |
| **Gemini** | Used by all agents for LLM reasoning: task decomposition (CEO), spec generation (Product), HTML generation + PR writing (Engineer), copy generation (Marketing) |
| **Hugging Face** | Automatic fallback LLM (Llama-3.3-70B) used by all agents when Gemini quota is exhausted |

---

## Repository Structure

```
Launchmind-Afan/
├── agents/
│   ├── ceo_agent.py          # Orchestrator: decomposes idea, reviews outputs, feedback loops
│   ├── product_agent.py      # Generates product spec (value prop, personas, features, stories)
│   ├── engineer_agent.py     # Builds HTML, creates GitHub issue, branch, commit, PR
│   ├── marketing_agent.py    # Generates copy, sends Brevo email, posts Slack Block Kit card
│   └── qa_agent.py           # (Stub) QA reviewer agent
├── tests/
│   ├── test_slack.py         # Standalone Slack integration test (auth, post, verify)
│   └── test_brevo.py         # Standalone Brevo diagnostic (quota, senders, send test)
├── main.py                   # Single entry point — runs the full pipeline end-to-end
├── message_bus.py            # Shared in-memory message bus with full communication logging
├── requirements.txt          # Python dependencies
├── .env.example              # Template for all required environment variables
├── .gitignore                # Ensures .env is never committed
└── README.md                 # This file
```

---

## Message Schema

Every agent-to-agent message is a structured JSON object:

```json
{
  "message_id":        "uuid-v4-string",
  "from_agent":        "ceo",
  "to_agent":          "product",
  "message_type":      "task",
  "payload":           { "idea": "...", "focus": "..." },
  "timestamp":         "2026-04-12T18:00:00Z",
  "parent_message_id": null
}
```

`message_type` is one of: `task` | `result` | `revision_request` | `confirmation`

The full message log (every message sent and received, never cleared) is available in `message_bus.full_message_log`. At the end of every pipeline run, it is simultaneously:
1. Printed to the terminal for the evaluator
2. Automatically saved locally to a `message_log.json` file in the project folder

---

## Dynamic Decision-Making (Feedback Loop)

The CEO agent does **not** just pipe outputs between agents. After the Product agent returns a spec, the CEO:

1. Sends the spec to Gemini with a strict review prompt
2. If the LLM responds with `NEEDS_REVISION`, it extracts the feedback and sends a `revision_request` message back to the Product agent with specific issues
3. The Product agent revises the spec and returns a new `result`
4. This loop repeats up to **2 revision cycles** before the CEO proceeds

This is implemented in `ceo_agent.review_product_spec()` and is fully visible in the terminal log and `full_message_log`.

---

## Live Demo Screenshots

> Run `python main.py`, enter a startup idea, and watch the agents collaborate in real time.

**GitHub Pull Request (opened by EngineerAgent):**
→ https://github.com/afanatif/Launchmind-Afan/pulls

**Slack Workspace:**
> The bot posts to the configured channel using Block Kit formatting after every run.

---

## Group Members & Agent Ownership

| Member | Agent | Responsibility |
|---|---|---|
| Afan Atif | CEO Agent + Engineer Agent | Orchestration, review loops, GitHub integration |
| Muhammad Shariq | Marketing Agent + Product Agent | Copy generation, Brevo email, Slack posting, spec generation |

---

## Grading Notes

- ✅ Real GitHub PR: visible at `https://github.com/afanatif/Launchmind-Afan/pulls`
- ✅ Real Slack message: Block Kit card posted to workspace channel after every run
- ✅ Real email: Brevo delivers to `TEST_EMAIL` with LLM-generated subject and body
- ✅ Feedback loop: CEO review of Product spec with up to 2 `revision_request` cycles
- ✅ Structured JSON messages: full schema with all 7 required fields
- ✅ LLM used ≥ twice in CEO: task decomposition + spec review
- ✅ All secrets in environment variables — never hardcoded
- ✅ HuggingFace fallback when Gemini quota exhausted
