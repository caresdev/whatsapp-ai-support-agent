# n8n Workflows

These JSON files are exported n8n workflows. Import them via
**n8n → Workflows → Import from File**. Each workflow is
self-contained but shares credentials (WhatsApp, Google Sheets,
OpenAI, and — once Phase 4 ships — Qdrant) configured at the n8n
instance level.

## Phased rollout

The workflow set is built in stages, mirroring the data-architecture
rollout (see [`DECISIONS.md`](../DECISIONS.md)):

- **Phase 1–2 (current):** `agent-main`, `notify-order`,
  `agent-broadcast`. Single data tool: Google Sheets.
- **Phase 4 (planned):** activates `ingest-knowledge` and adds a
  Qdrant tool node to `agent-main`.

## What's in here

| File | Phase | Trigger | Purpose |
|---|---|---|---|
| `agent-main.json` | 1–2 (current) | WhatsApp webhook | The primary conversational agent. Receives customer messages, looks up structured data in Sheets, generates a reply, and sends it back via the WhatsApp API. Phase 4 adds a Qdrant tool node alongside the Sheets node so the agent can route value vs. paragraph questions. |
| `notify-order.json` | 1–2 (current) | Sub-workflow (called by `agent-main`) | Sends a structured order summary to the owner's WhatsApp when a customer confirms an order. |
| `agent-broadcast.json` | 1–2 (current) | Manual / scheduled | Sends a one-to-many message to a customer list (promos, hours changes). Reads recipients from a Sheets tab. |
| `ingest-knowledge.json` | 4 (placeholder) | Manual | *Not yet active.* Will read markdown files from [`knowledge/`](../knowledge/), chunk them, embed via OpenAI, and upsert to a Qdrant collection. Designed to do a wholesale re-index per run — see [`DECISIONS.md`](../DECISIONS.md). |

## Read order for reviewers

If you're reading the repo to understand how the agent works:

1. **`agent-main.json`** — the heart of the system. In the current
   phase, look at how the prompt and the Sheets tool node work
   together; in Phase 4, this is also where the Qdrant tool node
   will be wired in.
2. **`notify-order.json`** — small, illustrates the sub-workflow
   pattern and the owner-notification path.
3. **`agent-broadcast.json`** — independent utility for outbound
   messaging; safe to skim last.
4. **`ingest-knowledge.json`** — Phase 4 placeholder; useful as a
   shape preview of the RAG ingestion pipeline rather than a
   working flow today.

## Conventions

- **Naming:** `<domain>-<purpose>.json`. `agent-*` are
  conversational, `ingest-*` are data pipelines, `notify-*` are
  outbound messaging.
- **Credentials** are referenced by name, not embedded — importing
  a workflow into a fresh n8n instance requires re-binding them.
- **Env vars** (sheet IDs, Qdrant collection name, owner phone
  number, etc.) are read via the n8n `$env` expression, not
  hardcoded.
- **Prompt copy:** the system prompt is mastered in
  [`prompts/system-prompt.md`](../prompts/system-prompt.md) and
  copied into `agent-main.json`. Keep them in sync on every edit.
