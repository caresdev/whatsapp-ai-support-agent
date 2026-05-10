# Data Architecture: Why a Hybrid (Sheets + RAG)

This document explains why the agent uses two data tools — Google
Sheets for structured data and a Qdrant vector store for semantic
search — and *when* it picks each one.

It also explains why the hybrid ships in two stages: Sheets-first
now, RAG added later. For the staged-rollout decision record, see
[`DECISIONS.md`](../DECISIONS.md).

## TL;DR

| | Google Sheets (direct) | Qdrant (RAG) |
|---|---|---|
| **What it stores** | Menu items, prices, availability, business settings, orders | Product stories, FAQ, policies, ingredient deep-dives, preparation guides |
| **Data shape** | Structured rows and columns | Unstructured prose |
| **How the agent queries** | Reads rows directly — deterministic | Embedding + similarity search |
| **Reliability for exact lookups** | 100% recall — sees every item | Variable — depends on embedding match |
| **Handles open-ended phrasing** | Limited | Excellent |
| **Who edits it** | Restaurant owner (directly in Sheets) | Developer (via knowledge-ingestion workflow) |
| **Ships in** | Phase 1–2 (current) | Phase 4 |

## The instinct, and why it's only half right

A reasonable first instinct is *"RAG is the modern, AI-native pattern,
so use RAG for everything."* That's partially right — RAG is a more
impressive architectural pattern than spreadsheet reads, and a
portfolio that shows vector embeddings, semantic search, and
retrieval-augmented generation tells the reader we understand
modern AI infrastructure.

But for **structured data**, RAG is not inherently more reliable.
For ~15–20 menu items, direct Sheets access can give the LLM the
entire menu in its context, every turn. When a customer asks
*"how much does this cost?"*, the agent sees every single row
and gives a deterministic answer. A RAG pipeline does similarity
search against embedded chunks — and might miss the chunk containing
the price if the query phrasing doesn't embed close to the chunk's
phrasing. **Similarity is not lookup.**

RAG shines somewhere else entirely: large bodies of unstructured
prose where the customer's question is unpredictable and the answer
is a paragraph, not a value. Ingredient deep-dives, preparation
methods, allergen explanations, the restaurant's story, edge-case
delivery policies — these don't fit cleanly in a spreadsheet cell,
and the corpus grows past what the LLM can hold in context.

The hybrid uses each tool for what it does best.

## How the agent decides which tool to call

The system prompt teaches the agent the routing rule:

- **Is the answer a value?** (a price, a yes/no, a time, a phone
  number, a delivery zone) → call **Google Sheets**.
- **Is the answer a paragraph?** (a story, an explanation, a nuance,
  a "why is it like that") → call the **Qdrant** tool.
- **Is the customer placing an order?** (item selection, quantity,
  address, payment) → use Sheets to confirm availability and price,
  then write the order row.

If the agent is uncertain, the prompt biases it toward Sheets first
(the deterministic source) and only escalates to RAG when the
question is clearly open-ended.

## The three approaches considered

**Approach 1 — Sheets-only (forever).**
Simple, deterministic, easy for the owner to maintain. The whole
menu fits in context; recall is perfect. But it doesn't scale to
rich prose content, and prose answers (ingredient stories, allergy
nuance) end up copy-pasted into spreadsheet cells where they read
awkwardly and break the structured-data contract.

**Approach 2 — RAG-only.**
Everything goes through embeddings. This handles unstructured
content beautifully, but it's overkill for "how much is X?" — adds
query latency, adds infrastructure (a Qdrant container), and creates
a sync problem: when the owner edits a price, the vectors must be
rebuilt before the new price is queryable.

**Approach 3 — Hybrid (chosen).**
Two tools, agent picks according to the user query. This is how production AI systems are
actually structured.

## Why ship Sheets-first

Even though hybrid is the destination, Phase 1–2 launch with Sheets
only:

- **Faster launch.** Conversational logic, ordering flow, and
  WhatsApp integration get validated before any RAG retrieval-quality
  bugs are mixed into the debugging surface.
- **No customer-visible loss.** With ~15–20 items, Sheets-only gives
  100% recall on every structured question; the kinds of questions
  that most need RAG (long prose answers) can be served from the
  prompt itself in the meantime.
- **Adding RAG later is additive.** A new tool node in the workflow,
  a new ingestion sub-workflow, a small prompt update. No refactor
  of the existing agent.

The current [`workflows/ingest-knowledge.json`](../workflows/ingest-knowledge.json)
is a placeholder for that Phase 4 work; the live agent currently does not yet
call into Qdrant.

## Infrastructure note

When Qdrant is enabled in Phase 4, it runs as an additional Docker
service alongside n8n on the same VPS. Memory footprint for a small
knowledge corpus is ~100–200MB — comfortably within a KVM 2 (8GB)
instance.