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

## Spreadsheets used

| Sheet | Purpose | Read/Write |
|---|---|---|
| `menu` | Items, categories, prices, availability flag | Read |
| `settings` | Hours, delivery zones, payment methods | Read |
| `orders` | Order log with timestamp, items, total, status | Write |

## Setup steps

1. **Create a Google Cloud project** and enable the Google Sheets API.
2. **Create a service account**, download the JSON key, and share each
   spreadsheet with the service account's email (Editor permission).
3. **Create the three spreadsheets** above. Use the CSVs in
   [`templates/seed/`](../../templates/seed/) as the column schema.
4. **Add the credentials in n8n**: Credentials → New → "Google Sheets
   Service Account" → paste the JSON key.
5. **Set the sheet IDs as env vars** (`MENU_SHEET_ID`, `SETTINGS_SHEET_ID`,
   `ORDERS_SHEET_ID`) referenced by the workflows.

## Seeding the menu

The seed CSVs in `templates/seed/` are formatted for direct import:

1. Open the empty `menu` spreadsheet
2. **File → Import → Upload `menu-catalog.csv`**
3. Choose **"Replace current sheet"**, separator type **comma**
4. Repeat with `whatsapp-messages.csv` for the message templates sheet

There is no programmatic seeder — the agent treats Sheets as the source
of truth, so manual import keeps the workflow obvious to the restaurant
owner who maintains it.
