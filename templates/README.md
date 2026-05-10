# Templates

CSV files used as **importable schemas/seed data** for the Google
Sheets the agent reads from. They define the column structure the
workflows expect — if you change the columns here, you'll need to
update the corresponding `agent-main.json` Sheets node references too.

## Files

| File | Imports into | Used by |
|---|---|---|
| `seed/menu-catalog.csv` | The `menu` spreadsheet | `agent-main.json` reads this on every order to look up items, prices, and availability. |
| `seed/whatsapp-messages.csv` | The `messages` spreadsheet | Pre-approved WhatsApp message templates (greetings, order confirmations, escalation handoffs). Editable without touching the workflow. |

## How to import

See the **"Seeding the menu"** section in
[`docs/setup/google-sheets.md`](../docs/setup/google-sheets.md). In
short: open the empty spreadsheet → File → Import → Upload → "Replace
current sheet" → comma separator.

## Schema changes

If you add or rename a column:

1. Update the CSV here.
2. Re-import into the live spreadsheet (or edit the column manually).
3. Update the relevant n8n workflow node to read the new column name.
4. If the column is referenced in `prompts/system-prompt.md` (e.g., a
   menu category), update there as well.
