# Design — Agent-Data Flywheel for a Vietnamese E-Commerce Customer Support Chatbot

## The Problem

A mid-size Vietnamese e-commerce platform (think Tiki or a Shopee seller hub) runs a customer support chatbot that handles ~50,000 chat sessions per day: order status, return/refund requests, product queries, and shipping complaints. The chatbot was launched six months ago with a base LLM and a hand-curated FAQ. Since then, the team has noticed that the model is drifting: it hallucinates return windows for new product categories (accessories added after the initial FAQ), and its refusal rate on legitimate return requests has climbed from 3% to 11%.

**The core problem:** The team has production traffic — 50k traces/day of real user interactions — but no systematic pipeline to turn that traffic into evaluation data and fine-tuning pairs. Every model update is a manual exercise: someone picks 200 conversations by hand, labels them, and runs a training job. The pipeline breaks every time a new product category ships. There is no decontamination; the team didn't know it was needed.

**Why it's hard:**
- Chat logs are in Vietnamese, with regional abbreviations ("tr" = "trả lại" / return), diacritics that differ by IME, and code-switching (Vietnamese + English product names like "iPhone 15 Pro Max").
- PDPL (Luật 91/2025) classifies chat content containing order IDs + user names as personal data. Storing raw traces beyond 90 days requires explicit consent or anonymization.
- The team is small (2 ML engineers) and the infra budget is tight — the pipeline must run cheaply on existing servers, not Spark or a cloud data warehouse.

---

## Architecture Sketch

```
Production chatbot
      │  one trace per session (gen_ai.* spans)
      ▼
┌─────────────────────┐
│   Ingest & Anonymize│  strip PII (user_id → hash, order_id → token)
│   (nightly batch)   │  before Bronze — PDPL gate
└────────┬────────────┘
         │ Bronze: append-only spans table (DuckDB)
         ▼
┌────────────────────┐
│  Validate & Tag    │  Pandera schema check; quarantine malformed spans
│  (quality gate)    │  tag outcome: ok / ToolError / Refusal / Hallucination
└────────┬───────────┘
         │ Silver: clean, typed spans
         ├──────────────────────────────────────┐
         ▼                                      ▼
┌─────────────────┐                  ┌──────────────────────┐
│  Build Eval Set │                  │  Build DPO Pairs     │
│  (split=eval,   │                  │  (ok vs error same   │
│   human-curated │                  │   prompt, decontam.) │
│   holdout)      │                  └──────────┬───────────┘
└────────┬────────┘                             │
         │                                      │
         └──────────────┬───────────────────────┘
                        ▼
               datasets/  (JSONL)
               eval_golden.jsonl
               preference_pairs.jsonl
                        │
                        ▼
               Day 22: SFT / DPO fine-tune
```

---

## Open Questions and Decisions

### Q1: Source & Shape — How dirty is the data, and does the schema drift?

**Decision: Treat each trace as semi-structured; enforce schema at ingest with a Pandera contract, not at the chatbot side.**

The chatbot emits `gen_ai.*` OpenTelemetry spans, but the schema is not frozen: when a new tool is added (e.g., `check_stock`), new span types appear. If we validate at the source, every new tool addition requires a schema change before any data lands. Instead, we validate at ingest with `strict=False` — unknown fields pass through to `attributes_json` — and only enforce the columns the flywheel actually needs (`user_input`, `agent_output`, `status`, `split`). Unknown span types land in Bronze and are ignored by downstream steps.

**Tradeoff: strict schema (catch drift early) vs. permissive schema (tolerate new tools without pipeline changes).** We chose permissive because the team ships new tools monthly and a strict gate would cause weekly outages. We mitigate the risk by alerting when the quarantine count spikes, which catches genuine corruption.

**Rejected alternative: validate at the chatbot SDK level (emit-time schema).** Rejected because it couples the ML pipeline to the application release cycle. A broken ML schema validator would silently drop production traffic.

---

### Q2: Batch or Streaming? — How fresh does the training data need to be?

**Decision: Nightly batch. Streaming is overkill for a training-data pipeline.**

Training runs on Day 22 cadence (weekly or bi-weekly SFT jobs). Data that is 24 hours old is indistinguishable from data that is 30 seconds old for the fine-tuning job. Running a Kafka consumer with real-time ingestion would cost ~3× more in infra (a managed Kafka cluster or Redpanda instance) and require an always-on consumer process with its own failure modes.

**Tradeoff: real-time streaming (sub-minute latency, higher cost, more ops burden) vs. nightly batch (24h latency, cheap, simple).** Latency budget for a weekly training cycle is one day — nightly batch satisfies this with no wasted cost.

**The one exception:** the quarantine alert (Q4 below) is emitted in near-real-time by a lightweight log monitor, so a spike in malformed traces pages the team within minutes even though the full pipeline is batch.

---

### Q3: Contracts & Quality — What do we validate before data reaches the model?

**Decision: Two-layer gate. Layer 1: schema (Pandera). Layer 2: content heuristics (Python rules).**

Layer 1 catches structural problems: missing `user_input`, null `agent_output`, invalid `status` values. These go to `quarantine.csv`.

Layer 2 catches semantic problems that pass schema validation but produce toxic training data:
- Turns where `agent_output` is shorter than 5 tokens (a non-answer that looks like `ok` in the schema).
- Turns where the output contains a PII pattern that the anonymizer missed (regex on CCCD/phone number format).
- Turns where `user_input` is in a non-Vietnamese, non-English language (the model was not trained for Korean or Chinese queries; these would confuse DPO pairs).

**Tradeoff: tight heuristics (more quarantine, safer training data) vs. loose heuristics (higher yield, possible noise).** We err toward tighter because DPO is sensitive to label noise — one bad `chosen`/`rejected` swap degrades the model more than skipping 10 borderline examples.

---

### Q5: Flywheel & Decontamination — How do we build the eval set without poisoning it?

**Decision: Hard split at collection time + exact-match decontamination + n-gram near-dedup.**

The eval set is defined by a `split='eval'` tag applied by human reviewers to ~5% of traces each week. This tag is set before the flywheel runs, so the eval set is structurally separate from the training pool.

Decontamination then removes any preference pair whose prompt appears in the eval set. For Vietnamese, exact-match after Unicode normalization (NFC) + lowercase + whitespace collapse is not enough: "Tôi muốn trả hàng" and "tôi muốn trả hàng." (trailing period) are the same question. We extend the lab's `decontaminate` function with 13-gram Jaccard similarity: any pair with ≥ 0.8 Jaccard overlap with an eval prompt is dropped.

**Tradeoff: exact-match (fast, may miss paraphrases) vs. embedding-similarity (catches semantic duplicates, needs a model call).** We chose 13-gram because it runs zero-key and catches the most common Vietnamese near-duplicate pattern (word reordering, added punctuation) without requiring an embedding API. Embedding-similarity is the extension exercise.

**Vietnamese-specific note:** Vietnamese diacritics are encoded differently depending on the IME (NFC vs. NFD decomposition). We normalize to NFC before any comparison — without this, "hoàn" (precomposed) and "hoàn" (decomposed) are treated as different prompts, and decontamination silently misses the match.

---

### Q10: Vietnamese Context — What changes versus an English pipeline?

**Decision: Anonymize PII at the Bronze boundary, not at query time. Use PDPL-compliant retention (90 days raw, indefinite for anonymized).**

PDPL (Luật Bảo vệ Dữ liệu Cá nhân, Luật 91/2025) requires that personal data (user name, phone, order ID) not be retained in raw form beyond the purpose of collection and without consent. Chat logs that contain order IDs + names are covered.

Our approach: before landing spans into Bronze, a deterministic anonymizer replaces `user.id` with a salted SHA-256 hash and scrubs order IDs and phone numbers from `user_input`/`agent_output` using regex. The raw span (with PII) is never written to disk. Bronze contains only anonymized data and can be retained indefinitely.

**Tradeoff: anonymize at ingestion (simple, irreversible) vs. anonymize at query time (reversible for debugging, higher risk of accidental exposure).** We chose ingestion-time anonymization because the risk of accidental PII exposure in a DuckDB file on a shared server outweighs the debugging convenience. A separate audit log (also anonymized) records which sessions were processed, so re-runs are traceable without re-exposing PII.

---

## Rejected Alternative: Real-time Kafka ingestion of traces

We considered consuming agent traces from a Kafka topic in real-time (producer in the chatbot SDK → Redpanda → DuckDB consumer via Redpanda Connect). This would give sub-minute Bronze latency.

**Rejected because:** The downstream training job runs weekly. Sub-minute ingestion latency does not improve model quality; it only increases infra cost (a managed Redpanda cluster costs ~$200/month on the smallest tier) and operational complexity (consumer lag monitoring, rebalancing, offset management). The nightly batch achieves the same outcome at near-zero marginal cost using a cron job and the existing server.

The only scenario where streaming would be justified is if we wanted a real-time safety filter (e.g., halt the chatbot if hallucination rate in the last 100 turns exceeds a threshold). That use case is handled by the lightweight quarantine spike alert (see Q3), not by a full streaming pipeline.
