# Demand Forecasting & Supply-Chain Reorder Agent (n8n)

An n8n workflow that forecasts weekly demand with simple rules, finds products that need a reorder, and asks an AI Agent (Gemini) to recommend the order quantity, urgency and a note for the supplier.

▶ **Demo video:** https://drive.google.com/file/d/1akq3TcQmvV7MKWYNE8h_GyJQN7IkAP-q/view

## How it works

1. Mock inventory arrives (stock, supplier lead time, last 4 weeks of sales)
2. A Code node forecasts weekly demand (average plus trend), the reorder point and days of cover
3. An IF node lets only products at or below the reorder point reach the AI, so healthy stock costs no AI call
4. The AI Agent returns reorder quantity (about 6 weeks of demand), urgency and a supplier note as JSON
5. A Code node validates the answer, then Slack alert and Postgres log

```
Manual Trigger -> Mock Inventory -> Forecast -> Needs Reorder?
   yes -> AI Agent (Gemini) -> Parse Advice -> Slack Alert -> Save Plan
   no  -> (skipped)
```

## AI Agent

AI Agent node without tools (JSON answer only). Forecasting is plain code, the AI only recommends the order.

## Tech stack

n8n, AI Agent, Gemini, Supply Chain, Forecasting, Slack, Postgres

## Setup

1. Copy the JSON from `workflow.md` into a file named `demand-forecast-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `supply-alerts`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: 4 mock inputs run through all branches

## Database

```sql
create table if not exists public.supply_plans (
  id bigint generated always as identity primary key,
  sku text,
  product text,
  stock numeric,
  weekly_forecast numeric,
  reorder_point numeric,
  days_of_cover numeric,
  status text,
  reorder_qty numeric,
  urgency text,
  note text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| SKU-100 Wireless Mouse (stock 40, rising sales) | Reorder, AI recommends quantity and urgency |
| SKU-200 USB-C Hub (stock 250, flat sales) | OK, no AI call |
| SKU-300 Laptop Stand (stock 15, 21-day lead time) | Reorder, likely high urgency |
| SKU-400 Desk Lamp (stock 90, steady sales) | OK, no AI call |

## Notes

- Replace Mock Inventory with Shopify orders and stock from your sheet or database for client use
- Quantities and urgency come from the AI and are validated in code (whole numbers, allowed urgency values only)
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
