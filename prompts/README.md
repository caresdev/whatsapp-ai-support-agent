# System Prompt — Design Rationale

The agent's behavior is governed by
[`system-prompt.md`](system-prompt.md), written in Portuguese. This
README explains the *why* behind that prompt for English-reading
reviewers.

## Files

- **`system-prompt.md`** — the live system prompt, in Portuguese.
- **`prompt-changelog.md`** — append-only log of meaningful prompt
  edits, with reasoning. Treats prompts as code: each behavior
  change is a dated entry explaining what shifted and why.

## Key design decisions

### Written in Portuguese, not English

The prompt is in the same language as the conversations it governs.
Mixing languages (English instructions, Portuguese conversation)
introduces drift — the model occasionally bleeds the instruction
language into customer-facing outputs. Writing the whole prompt in
Portuguese eliminates that class of bug and reads more naturally to
the non-technical owner who maintains it.

See [`DECISIONS.md`](../DECISIONS.md) for the formal record of this
decision.

### Tool selection rules are explicit

The prompt **names the available tools** and gives concrete examples
of when each is appropriate, rather than relying on the model's
judgment alone. This dramatically reduces "answered from memory"
hallucinations on price questions.

The set of tools the prompt teaches changes by phase:

- **Phase 1–2 (current):** one tool — **Google Sheets**. The prompt
  instructs the agent to look up *every* structured fact (prices,
  availability, hours, delivery zones) in Sheets rather than
  answering from its own training data. Anything outside the
  spreadsheet is either answered from prose written into the prompt
  itself, or escalated to the owner.
- **Phase 4:** adds a second tool — **Qdrant** (semantic search over
  the knowledge base in [`knowledge/`](../knowledge/)). The prompt
  gains a routing rule: *value questions → Sheets; paragraph
  questions → Qdrant.* See
  [`docs/data-architecture.md`](../docs/data-architecture.md) for
  the full routing logic.

When Phase 4 lands, this is a prompt edit (and a workflow tool-node
addition), not a refactor — keep that in mind when reading the
current prompt.

### Ordering flow is structured, conversation is not

The order-taking sub-flow uses a tight script (item → quantity →
confirm → address → payment) because customers expect efficiency
once they've decided to buy. Everything else (FAQ, browsing,
chitchat) is left open so the agent can be warm rather than robotic.

### Escalation is explicit, not a fallback

The prompt has a named "I don't know" path that hands the
conversation to the owner via WhatsApp. The agent is instructed to
use it **early** rather than guess — the cost of a wrong answer to a
customer (especially on allergy or pricing questions) is higher than
the cost of a five-minute owner reply.

This matters extra in Phase 1–2: without the Qdrant tool, the agent
has fewer ways to answer prose questions, so the escalation path
absorbs the gap. As Phase 4 ships and Qdrant fills in those answers,
expect the escalation rate to drop — that's a leading indicator the
RAG layer is pulling its weight.

## Editing the prompt

1. Make the change in `system-prompt.md`.
2. Add an entry to `prompt-changelog.md` with the date, the change,
   and *why* — including any test conversation that exposed the
   problem.
3. Update the n8n workflow's prompt node (the prompt is duplicated
   into `agent-main.json` for portability — keep them in sync).
