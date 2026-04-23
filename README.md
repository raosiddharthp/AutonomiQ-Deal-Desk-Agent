# AutonomIQ Deal Desk

**Autonomous, GCP-native agentic pipeline for end-to-end revenue-cycle quote generation.**

No human in the loop below a configurable deal-value threshold. A Salesforce opportunity is created. Ninety seconds later, a compliant, margin-verified quote is written back — with full provenance, a PDF artefact in GCS, and an immutable audit trail in BigQuery.

---

## What it does

A Salesforce `Opportunity.Created` event fires. AutonomIQ takes it from there:

1. **Pulls CRM context** — account segment, deal value, product line IDs from Salesforce
2. **Retrieves pricing rules** — tiered pricing and COGS from ERP
3. **RAG over contract precedents** — ANN vector search across a redacted corpus of historical deals in AlloyDB pgvector
4. **Computes margin deterministically** — parameterised BigQuery SQL against versioned pricing rules; no LLM arithmetic
5. **Synthesises the quote** — Gemini 1.5 Pro renders a compliant quote document from a Legal-approved Jinja2 template
6. **Routes or posts** — below threshold: writes directly to Salesforce; above threshold: creates an SF Approval Task + Slack Block Kit notification
7. **Logs everything** — every tool call, input, result, and decision written to a partitioned BigQuery audit table

Target end-to-end latency: **under 90 seconds** for straight-through deals. Zero human interactions.

---

## Architecture

Four layers. Each one owns a specific concern.

```
┌─────────────────────────────────────────────────────────────────┐
│  L1  PRESENTATION & EXPERIENCE                                  │
│      Salesforce Quote Record · SF Approval Task · Slack Block   │
│      Cloud Monitoring Ops Dashboard                             │
├─────────────────────────────────────────────────────────────────┤
│  L2  AGENT ORCHESTRATION                                        │
│      Cloud Run · Vertex AI Agent SDK · ReAct loop (max 12 turns)│
│      Cloud Workflows · Pub/Sub push subscription                │
├─────────────────────────────────────────────────────────────────┤
│  L3  MLOps & INTELLIGENCE                                       │
│      Gemini 1.5 Pro (synthesis) · Gemini 1.5 Flash (planning)   │
│      RAG: AlloyDB pgvector · text-embedding-004 · Cloud DLP     │
│      Margin Engine: BigQuery parameterised SQL                  │
├─────────────────────────────────────────────────────────────────┤
│  L4  INFRASTRUCTURE                                             │
│      Cloud Run (6 services) · AlloyDB HA · BigQuery · Firestore │
│      Cloud Storage · Secret Manager · Cloud Armor · Apigee      │
│      VPC · Private Service Connect · CMEK · VPC-SC              │
└─────────────────────────────────────────────────────────────────┘
```

Full C4 diagrams, sequence flows, ADRs, and Day 2 runbooks are in the [architecture design document](./GCP_DealDesk_Architecture.html).

---

## Services

| Service | Runtime | Purpose | Min instances |
|---|---|---|---|
| `orchestrator` | Cloud Run | Vertex AI Agent SDK · ReAct loop · tool dispatch | 1 |
| `intelligence-svc` | Cloud Run | Gemini API client · RAG query builder · prompt templates | 1 |
| `margin-engine` | Cloud Run | BigQuery pricing query · deterministic margin computation | 0 |
| `quote-generator` | Cloud Run | Jinja2 template rendering · PDF generation · GCS write | 0 |
| `approval-router` | Cloud Run | Threshold evaluation · SF Task creation · Slack notify | 0 |
| `sf-connector` | Cloud Run | Salesforce REST/Bulk API v2 · OAuth 2.0 JWT · mTLS | 1 |

---

## Data stores

| Store | Service | Purpose |
|---|---|---|
| AlloyDB PostgreSQL + pgvector | `vector-store` | Contract embeddings · HNSW index · 10M vector capacity baseline |
| BigQuery | `deal_desk_audit_events` | Immutable audit log · partitioned by date · clustered by opp_id |
| BigQuery | `pricing_rules`, `discount_matrix`, `fx_rates` | Deterministic margin computation |
| Cloud Firestore | `deal_desk_config` | Approval threshold · routing rules · idempotency dedup state |
| Cloud Storage | `contracts-raw`, `contracts-redacted`, `quotes-output` | Contract corpus and quote PDF artefacts |

---

## Agent tool manifest

The orchestrator dispatches these tools in a Vertex AI Agent SDK ReAct loop:

```python
get_crm_opportunity(opp_id: str)           # → Account + Opportunity context JSON
get_pricing_rules(acct_segment: str)        # → Tiered pricing and COGS JSON
retrieve_contract_precedents(ctx: dict)     # → 3 precedent excerpts + provenance IDs
compute_margin(inputs: dict)               # → {gm_pct, total_rev, total_cogs, ...}
generate_quote_document(state: dict)       # → quote JSON + GCS PDF URI
evaluate_approval_threshold()              # → PASS | ROUTE | REJECT
post_quote_to_salesforce(payload: dict)    # → SF Quote record + Chatter attachment
```

Max turns: 12. At turn 12, forced `FAILURE` state — full context dumped to DLQ, P0 alert fired.

---

## Pub/Sub event schema

```json
{
  "specversion": "1.0",
  "type": "com.salesforce.opportunity.created",
  "source": "urn:salesforce:org:00D5f0000004C9Z",
  "id": "evt_01J4KM3R9T...",
  "time": "2026-04-14T10:22:31Z",
  "data": {
    "opportunity_id": "006Qk000003Kp7Q",
    "account_id": "001Qk000002Lm9R",
    "amount": 48500.00,
    "stage": "Needs Analysis",
    "close_date": "2026-06-30",
    "product_line_ids": ["PLI-0042", "PLI-0087"],
    "account_segment": "MID_MARKET"
  }
}
```

---

## Approval thresholds

Stored in Firestore — no redeployment required to change.

```
auto_approve_max:   $75,000
margin_floor_pct:   18%
approval_sla_hours: 24
escalation_target:  Sales VP (after SLA breach)
```

Update the Firestore document. Changes take effect within 60 seconds.

---

## RAG pipeline

**Offline ingestion** (triggered on GCS upload via Eventarc):

```
GCS raw PDFs → Cloud DLP redaction → 768-token chunking
→ text-embedding-004 → AlloyDB pgvector HNSW upsert → BQ metadata write
```

**Online retrieval** (synchronous tool call, target <400ms p99):

```
Agent context → embed query → AlloyDB ANN cosine search (top-k=5)
→ metadata rerank (segment + value tier filter) → top-3 returned to agent
```

PII policy: all named entities (PERSON, EMAIL, PHONE, ADDRESS) are replaced with category tokens before embedding. Original documents are retained in a restricted GCS bucket.

---

## Margin computation

The LLM does not perform arithmetic. Full stop.

Margin is computed via a parameterised BigQuery query against a versioned `pricing_rules` table. The result — `{gm_pct, total_rev, total_cogs, discount_applied, bq_row_provenance}` — is passed to Gemini as grounding context. Every quote line item is traceable to a BigQuery row ID.

---

## Salesforce quote record

Fields written back on every successful run:

| Field | Source |
|---|---|
| `QuoteNumber__c` | UUID from `OpportunityId + timestamp` — idempotency key |
| `TotalDealValue__c` | Margin Engine |
| `GrossMarginPct__c` | Margin Engine + BQ provenance pointer |
| `PrecedentDealIds__c` | Top-3 RAG contract IDs |
| `AgentRunId__c` | Cloud Workflows execution ID → links to full audit trail |
| `ApprovalStatus__c` | PENDING_REVIEW / AWAITING_APPROVAL / APPROVED / REJECTED |
| `QuotePDFUrl__c` | Signed GCS URL (72h expiry) |

---

## Security

- All service-to-service communication uses dedicated GCP service accounts with least-privilege IAM bindings
- Salesforce OAuth credentials, ERP API keys, and AlloyDB passwords live in Secret Manager — CMEK-encrypted, rotated every 90 days
- AlloyDB, BigQuery, and GCS are inside a VPC Service Controls perimeter
- Cloud Armor WAF sits in front of the Apigee ingress gateway
- Cloud Audit Logs captures all `DATA_READ` and `DATA_WRITE` events with 90-day retention

---

## Observability

| Signal | Tool | What to watch |
|---|---|---|
| Agent traces | Cloud Trace | E2e latency, ReAct turn count, per-tool breakdown |
| Service logs | Cloud Logging | Error rate, latency, correlation by `opportunity_id` |
| Business metrics | BigQuery → Looker | STP rate, approval rate, margin distribution, token spend |
| Infrastructure | Cloud Monitoring | Cloud Run scaling, AlloyDB lag, BQ slot utilisation, DLQ depth |
| Alerts | PagerDuty / Slack | P0: DLQ non-empty · P1: failure rate >5% or p95 >120s |

---

## SLAs

| SLI | Target |
|---|---|
| Straight-through completion rate | >99% (rolling 7 days) |
| E2e latency p95 (below threshold) | <90s (rolling 24h) |
| Agent error rate (→ DLQ) | <1% (rolling 7 days) |
| Quote provenance completeness | 100% (per audit event) |
| Approval notification delivery | <60s from threshold decision |

---

## Repository structure

```
autonomiq-deal-desk/
├── services/
│   ├── orchestrator/          # Vertex AI Agent SDK · ReAct loop
│   ├── intelligence-svc/      # Gemini client · RAG query builder
│   ├── margin-engine/         # BigQuery pricing computation
│   ├── quote-generator/       # Jinja2 templates · PDF render
│   ├── approval-router/       # Threshold eval · SF Task · Slack
│   └── sf-connector/          # Salesforce REST/Bulk API adapter
├── infra/
│   ├── terraform/             # GCP resource definitions
│   ├── alloydb/               # Schema · HNSW index config
│   └── bigquery/              # Dataset · table schemas · views
├── pipelines/
│   ├── ingestion/             # DLP → chunk → embed → upsert
│   └── workflows/             # Cloud Workflows step definitions
├── templates/
│   └── quote/                 # Jinja2 quote templates · clause library
├── evaluation/
│   └── golden-dataset/        # 500-deal harness for model upgrade testing
├── docs/
│   └── GCP_DealDesk_Architecture.html
└── README.md
```

---

## Day 2

| Trigger | Action |
|---|---|
| New model release | Shadow-traffic 5% for 72h · eval on golden dataset · repin version |
| Corpus > 5M chunks | Migrate ANN to Vertex AI Vector Search · AlloyDB → metadata-only |
| Services > 20 | GKE Autopilot + Cloud Service Mesh (Istio) migration |
| Multi-region DR needed | AlloyDB replica in us-east1 · BQ multi-region · Global LB |
| DLQ non-empty | `dlq-processor` writes BQ record + SF Task + P0 alert |
| Quarterly threshold review | BQ audit query → Firestore config update · no deployment |

---

## Key design decisions

Nine ADRs with full rationale and rebuttals are in the architecture doc. The short version:

- **Vertex AI Agent SDK** over LangGraph — GCP-native tracing, quota enforcement, no portability requirement
- **AlloyDB pgvector** over Vertex AI Vector Search — SQL metadata filters, sub-ms ANN at 500K chunk scale
- **BigQuery for margin** — deterministic, auditable, version-controlled; LLM does not perform arithmetic
- **Cloud Run** over GKE — sufficient for 6 services; GKE is the documented graduation path at 20+
- **Firestore for config** — threshold changes take effect in 60s with no deployment
- **Pub/Sub push** over pull — lower latency, no polling loop, native Cloud Run scale-out trigger
- **Gemini Flash for planning, Pro for synthesis** — cost/quality routed by task complexity
- **Cloud DLP before embedding** — GDPR-safe corpus; rule-based redaction is auditable and certifiable
- **BigQuery for audit** — 30-day Cloud Logging retention is insufficient; cross-deal analytics require SQL

---

## References

- [TOGAF 10 Standard](https://www.opengroup.org/togaf) — architectural framework
- [C4 Model](https://c4model.com) — diagram notation
- [ReAct: Reasoning and Acting in LLMs](https://arxiv.org/abs/2210.03629) — agent loop design
- [RAG paper (Lewis et al., 2020)](https://arxiv.org/abs/2005.11401) — retrieval-augmented generation
- [Vertex AI Agent SDK](https://cloud.google.com/vertex-ai/docs/agents) — orchestration runtime
- [AlloyDB pgvector](https://cloud.google.com/alloydb/docs/pgvector) — vector store
- [Gemini 1.5 Technical Report](https://deepmind.google) — model capabilities and context window

---

*Architecture design document: [`GCP_DealDesk_Architecture.html`](./GCP_DealDesk_Architecture.html)*
