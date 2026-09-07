# Data Architecture: Why a Hybrid (Sheets + RAG)

The agent has two data tools: Google Sheets for structured data and a
Qdrant vector store for semantic search. This document covers why there
are two, how the agent picks between them, and why they don't ship at the
same time.

The decision record is in [`DECISIONS.md`](../DECISIONS.md). This is the
long version.

## Two tools, two jobs

|                                   | Google Sheets (direct)                                      | Qdrant (RAG)                                                              |
| --------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------- |
| **What it stores**                | Menu items, prices, availability, business settings, orders | Product stories, FAQ, policies, ingredient deep-dives, preparation guides |
| **Data shape**                    | Structured rows and columns                                 | Unstructured prose                                                        |
| **How the agent queries**         | Reads rows directly — deterministic                         | Embedding + similarity search                                             |
| **Reliability for exact lookups** | 100% recall — sees every item                               | Variable — depends on embedding match                                     |
| **Handles open-ended phrasing**   | Limited                                                     | Excellent                                                                 |
| **Who edits it**                  | Restaurant owner (directly in Sheets)                       | Developer (via knowledge-ingestion workflow)                              |
| **Ships in**                      | Phase 1                                                     | Phase 4                                                                   |

## The tempting wrong answer

A reasonable first instinct is _"use RAG for everything."_ RAG is a far more
impressive and elegant architectural pattern than spreadsheet reads — embeddings, similarity search, retrieval — and
it's what an AI project is "supposed" to look like next to a spreadsheet
read.

It's also the wrong tool for a price list.

This menu has roughly 15–20 items. Read directly from Sheets, the whole
catalog fits in the model's context on every turn. When a customer asks
*"how much does this cost?"*, the agent is looking at every row and
the answer is deterministic. Route that same question through RAG and it
becomes a similarity search: embed the question, pull the top handful of
chunks and hope the one holding the price is among them. Usually it is.
Usually is a bad property for a number a customer is about to pay.

There's a second cost. Prices change. In Sheets, the owner edits a cell
and the next message sees the new price. In a vector store, nothing is
true until the collection is re-embedded, so every price edit becomes a
pipeline run and a window where the agent confidently quotes a stale
number.

## Where RAG actually earns its place

Not every question has a cell that answers it:

> *"My kid is lactose intolerant, can I order this cake?"*
>
> *"How are your products different from other merchants'?"*

The answers are paragraphs. They live in prose that keeps growing — the
ingredient notes, the allergen policy, the restaurant's story, the
delivery edge cases in [`knowledge/`](../knowledge/). Two things make this
RAG's territory rather than Sheets': the phrasing is unpredictable, so
keyword or row matching doesn't get you there, and the corpus eventually
outgrows what's reasonable to keep in context on every turn.

Cramming that into spreadsheet cells works right up until it doesn't. A
paragraph in a cell reads badly, breaks the structured-data contract the
rest of the sheet depends on, and puts the owner in charge of editing
prose inside a grid.

So: each tool does the job it's actually good at.

## How the agent decides

The system prompt gives the agent a routing rule that mostly comes down
to the shape of the answer:

- **The answer is a value** — a price, a yes/no, a time, a delivery zone
  → **Google Sheets**.
- **The answer is a paragraph** — a story, an explanation, a nuance
  → **Qdrant**.
- **The customer is ordering** — item, quantity, address, payment →
  **Sheets** to confirm availability and price, then write the order row.

When it's ambiguous, the prompt biases toward Sheets. The deterministic
source is the safer default always, and a wrong price is a worse failure than a thin answer.

## Why Sheets ships first

Hybrid is the destination, but Phase 1 launches with Sheets alone.

Retrieval quality is its own debugging surface. Chunk sizes, embedding
choice, top-k, the gap between what a customer types and what the corpus
says — none of that is hard to fix, but all of it is hard to fix while
also working out whether the ordering flow drops state on the fourth
message. Shipping Sheets first means that when the agent gives a bad
answer, there's one place it came from.

Nothing customer-facing is lost in the meantime. At this menu size,
Sheets-only gives full recall on every structured question, and the
handful of prose answers that matter early can live in the system prompt
until the corpus justifies a vector store.

And the upgrade is additive: a tool node in
[`agent-main.json`](../workflows/agent-main.json), an ingestion
sub-workflow, a short prompt update. Nothing about the Phase 1 agent gets
rewritten to make room for it.

[`workflows/ingest-knowledge.json`](../workflows/ingest-knowledge.json)
is a placeholder for that work. The live agent does not call Qdrant yet.

## Infrastructure note

Qdrant runs as a second Docker service next to n8n on the same VPS, with
n8n reaching it over the internal network at `http://qdrant:6333`. A
knowledge base this size needs roughly 100–200MB of RAM, which a KVM 2
(8GB) instance absorbs without noticing. The service is already written
into the compose file, commented out — see
[`docs/setup/hostinger.md`](setup/hostinger.md).