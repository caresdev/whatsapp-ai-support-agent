# Templates

CSV files that define the column schema for the agent's Google Sheets
data layer. Each one imports into a tab of a **single spreadsheet** — not
into separate documents.

## One spreadsheet, three tabs

| File | Imports into tab | Access | Holds |
|---|---|---|---|
| `seed/menu-catalog.csv` | `cardapio` (menu) | Read | Items, categories, prices, ingredients, availability flags |
| `seed/settings.csv` | `configuracoes` (settings) | Read | Key-value business settings: hours, delivery area, payment methods |
| `seed/orders.csv` | `pedidos` (orders) | Write | Append-only order log |

One document means one service-account share, one ID in the environment
(`GOOGLE_SHEETS_ID`), and one revision history covering all three tabs
together. See [`docs/data-architecture.md`](../docs/data-architecture.md)
for why Sheets holds the structured layer at all.

## Why the tabs and columns are in Portuguese

Filenames and documentation here are English, because the audience for
this repo is English-speaking. The tab names and column headers are not,
because they're the surface the restaurant owner works in. They edit
`preco` (price) and toggles `disponivel` (available) themselves; making them navigate an English
schema to change their own prices would be the wrong tradeoff. Same
reasoning as the Portuguese system prompt — see
[`DECISIONS.md`](../DECISIONS.md).

## How to import

See **"Seeding the sheets"** in
[`docs/setup/google-sheets.md`](../docs/setup/google-sheets.md). In short:
open the spreadsheet, create the tab, then File → Import → Upload →
"Replace current sheet" → comma separator.

## Schema changes

If you add or rename a column:

1. Update the CSV here.
2. Re-import into the live spreadsheet (or edit the column manually).
3. Update the relevant n8n workflow node to read the new column name.
4. If the column is referenced in `prompts/system-prompt.md` (e.g., a
   menu category), update there as well.

## Not here: WhatsApp message templates

Outbound message templates (order confirmation, promotions,
re-engagement) are authored and approved in WhatsApp Manager, and Meta
holds their approval status. A copy in this repo could not stay in sync
with it, so the template bodies are documented in
[`docs/setup/whatsapp.md`](../docs/setup/whatsapp.md) rather than living
here as data.