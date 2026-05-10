# Decisions Log

Short architectural decision records (ADR-lite). Each entry captures
a tradeoff that wouldn't be obvious from reading the code alone.
Newest first.

---

## 2026-05 — Hybrid data layer, shipped in two stages

**Decision:** The agent will use a hybrid data layer — Google Sheets
for structured data + Qdrant for semantic search — but it rolls out
in two stages:

- **Stage 1 (Phase 1–2, current):** Google Sheets only. Single tool
  for menu, prices, availability, business settings, and order log.
- **Stage 2 (Phase 4):** Adds Qdrant as a second tool for semantic
  search over unstructured prose (product stories, FAQ, policies,
  ingredient deep-dives).

**Why hybrid is the destination (not Sheets-only forever):**
Prose answers — ingredient stories, allergen nuance, edge-case
delivery policies, "why is it like that" backstory — don't fit
cleanly in a spreadsheet cell, and the corpus grows past what the
LLM can hold in context. RAG handles a fundamentally different query
shape: open-ended phrasing where the answer is a paragraph, not a
value.

**Why hybrid is the destination (not RAG-only):**
RAG is *less* reliable than direct
Sheets reads for structured questions like *"how much does this cost?"*. Similarity search may retrieve four chunks and miss the
one with the price; direct row access deterministically sees every
item. Use each tool for what it does best.

**Why ship Sheets-first instead of full hybrid on day one:**
- De-risks the launch — conversational logic, ordering flow, and
  WhatsApp integration get validated before retrieval-quality bugs
  enter the debugging surface.
- The first ~15–20 menu items fit comfortably in the LLM context,
  so Sheets-only achieves 100% recall on structured questions.
  Customers don't notice the second tool is missing yet.
- Adding Qdrant later is purely additive: new tool node in the
  workflow, new ingestion sub-workflow, small prompt update.
  No refactor of the existing agent.

Full reasoning in [`docs/data-architecture.md`](docs/data-architecture.md).

---

## 2026-05 — n8n over a custom Node service

**Decision:** Build the agent as n8n workflows rather than a custom
backend (Node + Express, FastAPI, etc.).

**Why:**
- Built-in nodes for WhatsApp, Google Sheets, OpenAI, and Qdrant
  remove a week of glue code.
- Hosting is one Docker container on a VPS — no separate app server
  to operate.

**Tradeoffs accepted:**
- JSON workflow exports are noisier than code in PRs.
- Versioning prompts across files is awkward — addressed by treating
  `prompts/system-prompt.md` as the source of truth and copying into
  the workflow on edit.
- Heavy custom logic (multi-step state machines) gets clunky in n8n;
  if the order flow grows much more complex, a small companion
  service may be worth introducing.

---

## 2026-05 — Portuguese-only system prompt

**Decision:** Write the system prompt in the same language as the
end-user conversation (Portuguese), not English.

**Why:**
- Mixed-language prompts caused occasional English bleed-through in
  customer-facing replies during early prototyping.
- The non-technical owner can read and edit the prompt directly
  without a translation round-trip.

**Tradeoff:** English-speaking reviewers need a separate explanation
of the prompt's structure (provided in
[`prompts/README.md`](prompts/README.md)).

---

## 2026-05 — Knowledge base will be re-indexed wholesale on change (Phase 4)

**Decision:** When Qdrant is added in Phase 4, the
`ingest-knowledge` workflow will drop and rebuild the collection on
each run rather than doing incremental upserts.

**Why:**
- Knowledge files change rarely (weekly at most) and the corpus is
  small (~5 files, well under 100 chunks total). Re-embedding the
  whole thing costs cents.
- Incremental sync would require tracking content hashes, deletions,
  and chunk-level identity — complexity that buys nothing at this
  scale.

**Re-evaluate when:** the corpus grows past ~500 chunks, or full
re-indexing exceeds ~$1 per cycle.