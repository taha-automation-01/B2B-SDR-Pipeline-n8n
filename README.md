B2B SDR Pipeline — Automated Lead Qualification & Outreach
Manual lead qualification is slow, inconsistent, and doesn't scale. Sales teams spend hours researching inbound leads, guessing at intent, and writing first-touch emails by hand — and top-of-funnel quality suffers as volume grows.
B2B SDR Pipeline is an end-to-end n8n automation that solves this. It ingests inbound leads through a Webhook, enriches them with live web intelligence (Tavily), scores buying intent with an LLM via OpenRouter, routes leads by priority, drafts a personalized outreach email, and persists every qualified lead to a Baserow database.
What once took an SDR 15–20 minutes per lead now happens in seconds — consistently, auditable, and with a full paper trail in the database.
Stack: n8n · Tavily (enrichment) · OpenRouter (LLM scoring & drafting) · Baserow (persistence) · Discord (notifications)
Architecture & Data Flow
The pipeline is a linear data flow with a single decision point. Each stage owns one responsibility and passes a normalized JSON payload downstream, which keeps the workflow debuggable: any failure is isolated to exactly one node.
Webhook ─▶ Sanitize ─▶ Enrichment HTTP (Tavily) ─▶ LLM Scoring (OpenRouter)
                                                        │
                                                        ▼
                                                      Router
                                              ┌─────────┴─────────┐
                                              ▼ (qualified)       ▼ (low priority)
                                        Email Draft            discard / log
                                              │
                                              ▼
                                    Code (normalize + parse)
                                              │
                                              ▼
                                    Baserow ──▶ Discord Notify

Key design decisions:
Sanitize before anything else. Raw webhook payloads are untrusted. The Sanitize node trims whitespace, normalizes casing, and drops malformed entries before any API call is made — so enrichment and LLM credits are never wasted on junk input.
Enrichment grounds the LLM. Instead of asking the model to guess a company's profile, the Tavily HTTP node pulls real, current context. This measurably improves scoring accuracy and makes the reasoning explainable.
The LLM returns structured JSON. Scoring and reasoning are requested in a strict JSON schema so the downstream Code node can parse deterministically.
Persistence is defensive. The final Code node extracts JSON safely (string-index based, not regex), handles both content and reasoning output fields from reasoning models, and maps fields case-insensitively to Baserow's column names.
Pipeline Stages in Detail
Webhook — The entry point. Accepts a POST payload with name, company, email, and optional notes. Any form, CRM, or manual trigger can feed it.
Sanitize (Code node) — Validates required fields, trims and normalizes input, and rejects empty payloads early. This node exists to protect the paid API calls downstream.
Enrichment HTTP (Tavily) — Performs a targeted web search on the lead's company and role to surface firmographic context: industry, size, recent activity, and buying signals. The result is injected into the scoring prompt as grounding context.
LLM Scoring (OpenRouter) — Calls the model (e.g., openai/gpt-oss-120b) with a prompt that returns:
{ "score": 90, "reason": "VP of Growth at an established company..." }

The score reflects budget, authority, need, and timing; the reason is a human-readable justification stored alongside the lead.
Router — A conditional node that splits the flow by score threshold. Qualified leads continue to email drafting; low-priority leads are logged and skipped, keeping the SDR's queue clean.
Email Draft (OpenRouter) — Generates a personalized subject line and first-touch email, grounded in the enrichment context and the scoring rationale. Output follows the same strict-JSON contract.
Code (normalize + parse) — The reliability layer. It:
Safely extracts the JSON block from the model output using indexOf/lastIndexOf — immune to markdown backticks and stray text that break regex-based parsing.
Reads both content and reasoning fields, so reasoning models work without changes.
Case-insensitively maps fields (name → Name, email_draft → Email_Draft) to match Baserow column names exactly.
Baserow (Create row) — Writes the final record into the Leads_DB table. Per Baserow's n8n integration docs, the node supports create, read, update, and delete operations, authenticating once against either Baserow Cloud or a self-hosted instance.
Discord Notify — Pushes a summary of each qualified lead to a team channel, so the SDR sees hot leads in near real time without opening the database.
Getting Started
Setup takes about ten minutes: import the workflow, attach four credentials, and create the database table.
1. Import the workflow
Open your n8n instance.
Click Add workflow → Import from File.
Select workflow/b2b-sdr-pipeline.json.
2. Configure credentials
Create each credential once in n8n under Credentials, then attach it to the relevant nodes:
CredentialUsed inTavily / Search APIEnrichment HTTPOpenRouter API keyLLM Scoring, Email DraftBaserow credentialBaserow (Create row)Discord Webhook URLDiscord Notify
For Baserow, set the Host URL to https://api.baserow.io (cloud) or your self-hosted API endpoint. Your numeric Table ID appears in Baserow's auto-generated API documentation (three-dot menu → API docs).
3. Create the database table
Create a table named Leads_DB with these exact fields:
FieldTypeNameSingle line textCompanySingle line textEmailEmail / TextScoreNumberReasonLong textSubjectSingle line textEmail_DraftLong text
4. Test
Send the sample payload from examples/sample-lead.json to your Webhook URL and confirm a new row appears in Leads_DB with a score, reason, and email draft populated.
Field Mapping Reference
Baserow column names are capitalized; model and webhook outputs frequently are not. The final Code node resolves this automatically, but the canonical mapping is:
StageInput FieldOutput Field (Baserow)Webhook / Sanitizename / NameNameWebhook / Sanitizecompany / CompanyCompanyWebhook / Sanitizeemail / EmailEmailLLM ScoringscoreScoreLLM Scoringreason / reasoningReasonEmail DraftsubjectSubjectEmail Draftemail_draft / bodyEmail_Draft
If you add columns to Leads_DB, extend the mapping object in the Code node — the case-insensitive lookup means only the target column name must match exactly.
Security Notes
Never hardcode keys in the workflow JSON. The exported workflow/b2b-sdr-pipeline.json must contain no API keys, tokens, or webhook URLs. Use n8n's credential system exclusively.
.gitignore excludes .env, *.env, credentials.json, and editor/OS artifacts.
Rotate any credential that has ever appeared in a screenshot or export.
Restrict the Webhook (header auth or an IP allowlist) if it is exposed publicly — it is an unauthenticated write path into your pipeline by default.
Repository Structure
b2b-sdr-pipeline/
├── README.md
├── LICENSE
├── .gitignore
├── workflow/
│   └── b2b-sdr-pipeline.json      # Exported n8n workflow (credential-free)
├── assets/
│   ├── workflow-overview.png      # Clean screenshot of the n8n canvas
│   └── database-demo.png          # Baserow table with populated rows
├── examples/
│   ├── sample-lead.json           # Webhook input payload
│   └── sample-output.json         # Final record written to Baserow
└── docs/
    ├── setup.md                   # Step-by-step setup guide
    └── field-mapping.md           # Field mapping reference
