# WhatsApp AI Support Agent

An AI-powered WhatsApp ordering assistant for a Brazilian restaurant,
built with n8n, WhatsApp Business Cloud API, Google Sheets, and RAG.

## What It Does

- Answers customer questions about the menu, hours, delivery, and payments
- Guides customers through a complete ordering flow via WhatsApp
- Manages a dynamic menu catalog from Google Sheets
- Notifies the restaurant owner of new orders in real-time
- Escalates to the owner when the AI can't help

**Stack:** n8n (self-hosted) · WhatsApp Business Cloud API · Google Sheets
· OpenAI · Qdrant *(vector store, added in Phase 4)*

### Hybrid Data Architecture (staged rollout)

The agent's data layer is designed as a **hybrid** — two tools, the
agent picks which to call per question — but it ships in two stages:

- **Phase 1–2 (current):** Google Sheets only. Structured lookups for
  menu, prices, availability, business settings, and order logging.
  With ~15–20 menu items, the entire catalog fits in the LLM's
  context, giving 100% recall on every query.
- **Phase 4 (planned):** Adds a **Qdrant vector store** as a second
  tool for semantic search over unstructured knowledge — ingredient
  deep-dives, allergen details, FAQ, product stories, delivery
  policies. The agent will choose between the two tools per turn.

Why staged, why hybrid (and not RAG-only or Sheets-only forever):
see [`docs/data-architecture.md`](docs/data-architecture.md) and
the [decisions log](DECISIONS.md).

## How to Read This Repo

This is a portfolio repo: the runtime (n8n, Qdrant, the WhatsApp webhook)
lives on a self-hosted server, not in this directory. The repo is the
**design and configuration artifact** behind the agent. If you're browsing
to understand how it's built, read in this order:

1. [`docs/architecture.md`](docs/architecture.md) — the end-to-end flow:
   customer message → n8n → tool routing → response
2. [`docs/data-architecture.md`](docs/data-architecture.md) — why the agent
   splits between Google Sheets (structured) and Qdrant (semantic)
3. [`prompts/system-prompt.md`](prompts/system-prompt.md) — the agent's
   personality, tool-selection rules, and ordering flow (Portuguese)
4. [`workflows/agent-main.json`](workflows/agent-main.json) — the primary
   n8n workflow that wires everything together
5. [`DECISIONS.md`](DECISIONS.md) — short ADRs explaining the bigger
   tradeoffs (n8n vs custom code, Sheets + Qdrant hybrid, etc.)

## Deployment

The agent runs on a self-hosted n8n instance — these workflows are not
intended to run locally. To stand up your own copy, follow the setup
guides in [`docs/setup/`](docs/setup/):

- [Hostinger + n8n](docs/setup/hostinger.md) (server provisioning)
- [WhatsApp Business Cloud API](docs/setup/whatsapp.md)
- [Google Sheets](docs/setup/google-sheets.md)
- [Qdrant + knowledge ingestion](docs/setup/qdrant.md)

## Documentation

- [Architecture & Data Flow](docs/architecture.md)
- [Data Architecture Decision](docs/data-architecture.md)
- [Decisions Log](DECISIONS.md)
- [Setup guides](docs/setup/)
- [Troubleshooting](docs/troubleshooting.md)

## System Prompt

The AI agent's personality and behavior are defined in
[`prompts/system-prompt.md`](prompts/system-prompt.md) (in Portuguese —
the agent serves Brazilian customers). See
[`prompts/README.md`](prompts/README.md) for design rationale in English.

## License

MIT