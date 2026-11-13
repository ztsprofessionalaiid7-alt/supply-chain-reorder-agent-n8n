# Demand Forecasting & Supply-Chain Reorder Agent (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `demand-forecast-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "Autonomous demand forecasting & supply-chain reorder agent (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table supply_plans in Supabase SQL Editor\n3. Slack channel: supply-alerts (change in the Slack node)\n4. Click Execute workflow: 4 mock products, rules forecast demand, AI only runs for items that need a reorder\nReal client use: replace Mock Inventory with Shopify orders + stock from your sheet or database\nAI Agent node (Gemini) (no tools, JSON answer)",
        "height": 260,
        "width": 560
      },
      "id": "81df176e-7ed9-4fc8-9a88-5c77fcc0d504",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "117f03ac-e6cc-44e5-9369-56fb70950e13",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { sku: 'SKU-100', product: 'Wireless Mouse', stock: 40, lead_time_days: 14, weekly_sales: [30, 34, 38, 42], unit_cost: 8 } },\n  { json: { sku: 'SKU-200', product: 'USB-C Hub', stock: 250, lead_time_days: 10, weekly_sales: [20, 18, 22, 19], unit_cost: 12 } },\n  { json: { sku: 'SKU-300', product: 'Laptop Stand', stock: 15, lead_time_days: 21, weekly_sales: [10, 12, 15, 18], unit_cost: 15 } },\n  { json: { sku: 'SKU-400', product: 'Desk Lamp', stock: 90, lead_time_days: 7, weekly_sales: [25, 24, 26, 25], unit_cost: 9 } }\n];"
      },
      "id": "cbb3f6f3-6d77-4bf3-8c0d-ab99f659bcc1",
      "name": "Mock Inventory",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return $input.all().map((it) => {\n  const p = it.json;\n  const s = p.weekly_sales;\n  const avg = s.reduce((a, b) => a + b, 0) / s.length;\n  const trend = (s[s.length - 1] - s[0]) / (s.length - 1);\n  const weekly_forecast = Math.max(0, Math.round(avg + trend));\n  const demand_lead = Math.ceil(weekly_forecast * p.lead_time_days / 7);\n  const reorder_point = Math.ceil(demand_lead * 1.2);\n  const days_of_cover = weekly_forecast ? Math.round(p.stock / (weekly_forecast / 7)) : 999;\n  const status = p.stock <= reorder_point ? 'reorder' : 'ok';\n  return { json: Object.assign({}, p, { weekly_sales: s.join(','), weekly_forecast, reorder_point, days_of_cover, status, reorder_qty: 0, urgency: 'none', note: '' }) };\n});"
      },
      "id": "9bc4d5b1-7710-4e75-9a37-85ddd4225264",
      "name": "Forecast",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "463138ee-d2ce-4119-870e-5dcf228110d8",
              "leftValue": "={{ $json.status }}",
              "rightValue": "reorder",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "e0c5accf-175a-413e-9a66-ae59db8f92ca",
      "name": "Needs Reorder?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        660,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a supply-chain planner. Recommend a reorder.\nProduct: {{ $json.product }} ({{ $json.sku }})\nCurrent stock: {{ $json.stock }}\nForecast demand: {{ $json.weekly_forecast }} units per week\nSupplier lead time: {{ $json.lead_time_days }} days\nReorder point: {{ $json.reorder_point }}\nUnit cost: {{ $json.unit_cost }}\n\nReturn reorder_qty (whole units to order, enough for about 6 weeks of demand), urgency (low, medium or high; high if stock runs out before the supplier can deliver) and a one-sentence note for the supplier.",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a supply-chain planner. Respond with ONLY a JSON object, no markdown, no extra text: {\"reorder_qty\":0,\"urgency\":\"low|medium|high\",\"note\":\"one sentence for the supplier\"}. reorder_qty is a whole number of units covering about 6 weeks of demand. urgency is high if stock runs out before the supplier can deliver."
        }
      },
      "id": "69d7d506-3550-4136-b1a7-b6bd67526f31",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        900,
        60
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "75a84b03-13db-4343-829a-4948049ef132",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        880,
        280
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Forecast').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst qty = Math.max(0, Math.round(Number(r.reorder_qty) || (ctx.weekly_forecast * 6)));\nconst urg = String(r.urgency || '').toLowerCase();\nreturn { json: Object.assign({}, ctx, {\n  reorder_qty: qty,\n  urgency: ['low', 'medium', 'high'].includes(urg) ? urg : 'medium',\n  note: String(r.note || '').slice(0, 300)\n}) };"
      },
      "id": "b44eea67-5762-449e-97d4-b7606046f0d2",
      "name": "Parse Advice",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1260,
        60
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "supply-alerts",
          "mode": "name"
        },
        "text": "=:package: *Reorder needed: {{ $json.product }}*\nStock: {{ $json.stock }} | Forecast: {{ $json.weekly_forecast }}/week | Lead time: {{ $json.lead_time_days }} days\nSuggested order: {{ $json.reorder_qty }} units ({{ $json.urgency }} urgency)\nNote: {{ $json.note }}",
        "otherOptions": {}
      },
      "id": "e435829a-10ff-4ab0-b0ab-93d4c85c735c",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1520,
        0
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "supply_plans",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "sku": "={{ $json.sku }}",
            "product": "={{ $json.product }}",
            "stock": "={{ $json.stock }}",
            "weekly_forecast": "={{ $json.weekly_forecast }}",
            "reorder_point": "={{ $json.reorder_point }}",
            "days_of_cover": "={{ $json.days_of_cover }}",
            "status": "={{ $json.status }}",
            "reorder_qty": "={{ $json.reorder_qty }}",
            "urgency": "={{ $json.urgency }}",
            "note": "={{ $json.note }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "sku",
              "displayName": "sku",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "product",
              "displayName": "product",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "stock",
              "displayName": "stock",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "weekly_forecast",
              "displayName": "weekly_forecast",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "reorder_point",
              "displayName": "reorder_point",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "days_of_cover",
              "displayName": "days_of_cover",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "reorder_qty",
              "displayName": "reorder_qty",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "urgency",
              "displayName": "urgency",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "note",
              "displayName": "note",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "551c5356-0f12-43ac-8e9c-ce483ae16f65",
      "name": "Save Plan",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1520,
        380
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Inventory",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Inventory": {
      "main": [
        [
          {
            "node": "Forecast",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Forecast": {
      "main": [
        [
          {
            "node": "Needs Reorder?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Needs Reorder?": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Plan",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Advice": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Plan",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Advice",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
