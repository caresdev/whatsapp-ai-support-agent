# Google Sheets Setup

The agent uses Google Sheets as its **structured data layer**: menu
catalog, prices, availability, business settings, and the order log.
n8n reads and writes to these sheets via the Google Sheets API.

## Why Sheets

- The restaurant owner can easily work with spreadsheets. Edits to the menu,
  prices, or hours don't require a code change or a deploy.
- Read/write is straightforward from n8n's built-in Google Sheets node.
- Order history is human-readable from day one. No admin UI needed.

For the full Sheets-vs-Qdrant split, see
[`docs/data-architecture.md`](../data-architecture.md).

## One spreadsheet, three tabs

| Tab | Purpose | Read/Write |
|---|---|---|
| `cardapio` (menu) | Items, categories, prices, ingredients, availability flag | Read |
| `configuracoes`(settings) | Key-value settings: hours, delivery zones, payment methods | Read |
| `pedidos` (orders) | Order log with timestamp, items, total, status | Write |

Everything lives in a single spreadsheet rather than three. That means one
service-account share to grant, one ID to configure, and one revision
history that restores all three tabs to a consistent point.

The tab and column names are Portuguese because the owner edits them
directly. See
[`templates/README.md`](../../templates/README.md).

## Setup steps

1. **Create a Google Cloud project** and enable the Google Sheets API.
2. **Create a service account**, download the JSON key, and share the
   spreadsheet with the service account's email (Editor permission).
3. **Create the spreadsheet** with the three tabs above. Use the CSVs in
   [`templates/seed/`](../../templates/seed/) as the column schema.
4. **Add the credentials in n8n**: Credentials → New → "Google Sheets
   Service Account" → paste the JSON key.
5. **Set `GOOGLE_SHEETS_ID`** in the environment to the spreadsheet's
   document ID — the segment between `/d/` and `/edit` in its URL.

## Seeding the sheets

The seed CSVs in `templates/seed/` carry the column headers, ready for
direct import. They ship without sample rows; you're filling the catalog
with the restaurant's real menu.

For each of the three files:

1. Create the tab and give it the Portuguese name from the table above
2. **File → Import → Upload** the matching CSV
3. Choose **"Replace current sheet"**, separator type **comma**

`orders.csv` is headers-only by design — the agent appends to it.

There is no programmatic seeder. The agent treats Sheets as the source of
truth, so manual import keeps the workflow obvious to the restaurant owner
who maintains it.
