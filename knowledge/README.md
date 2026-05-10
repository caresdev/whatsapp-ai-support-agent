# Knowledge Base — RAG Source Documents

> **Phase 4, not yet live.** These files are the planned source
> material for the agent's semantic-search tool (Qdrant), which is
> added in Phase 4 of the rollout. The Phase 1–2 agent ships with
> Google Sheets only — see [`DECISIONS.md`](../DECISIONS.md) and
> [`docs/data-architecture.md`](../docs/data-architecture.md) for
> why the hybrid is staged.

These markdown files are the **source of truth** for the agent's
semantic knowledge. Once Phase 4 ships, the `ingest-knowledge.json`
workflow will chunk them, embed each chunk via OpenAI, and upsert
the vectors into Qdrant. Re-run that workflow whenever a file here
changes.

Content is written in **Portuguese** because the agent serves
Portuguese speakers — embedding and retrieving in the user's
language gives better semantic match quality than translating at
runtime.

## When this knowledge base, vs. Google Sheets?

The agent has (will have) two data tools and picks per question:

- **Answer is a value** — a price, a yes/no, an opening time, a
  delivery zone — → **Google Sheets** (deterministic row lookup).
- **Answer is a paragraph** — a story, an explanation, a policy
  nuance, a "why is it like that" — → **this knowledge base**
  (semantic search).

Full reasoning in
[`docs/data-architecture.md`](../docs/data-architecture.md).

## What each file covers

| File | Topic |
|---|---|
| `historia-produtos.md` | Origin stories, inspiration, and "why we make it this way" notes for signature items. Used when customers ask backstory questions. |
| `ingredientes-detalhados.md` | Per-item ingredient breakdowns, allergens, sourcing notes. Used for dietary-restriction and "what's in this?" questions. |
| `politicas-entrega.md` | Delivery zones, fees, timing windows, and cancellation policy in prose form. Structured pricing lives in Sheets; this file is for the *explanations*. |
| `faq.md` | Frequently asked questions with full prose answers. The catch-all for things that don't fit the other files. |

## Adding a new knowledge file

1. Create a new `.md` file in this folder. Keep it focused on a
   single topic — smaller, well-scoped files chunk and retrieve
   better than one giant catch-all.
2. Write in Portuguese (or whichever language your customers speak),
   in the same prose style as the existing files.
3. *(Phase 4)* Re-run the `ingest-knowledge` workflow in n8n to
   re-embed everything.
4. *(Phase 4)* Test with a few queries that should hit the new
   content.

## Removing or replacing content

*(Phase 4)* The ingestion workflow does a **full re-index** — drop
and rebuild the collection — so deleting or editing a file here is
enough; no manual Qdrant cleanup needed. Reasoning in
[`DECISIONS.md`](../DECISIONS.md) under
*"Knowledge base will be re-indexed wholesale on change."*

## Tone and structure tips

These files are read by an LLM, not just by humans, so:

- **Use clear section headers.** The chunker splits on headings;
  well-headed files retrieve more accurately.
- **Keep each section self-contained.** Don't write "as mentioned
  above" — a chunk may be retrieved without its neighbors.
- **Write the way customers ask.** If customers say *"é apimentado?"*,
  use that phrasing in the file rather than a clinical equivalent.
  Embedding similarity rewards matching vocabulary.
