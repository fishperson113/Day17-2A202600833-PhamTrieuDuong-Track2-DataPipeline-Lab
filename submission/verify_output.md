# Verification Output

## `make verify` — 16/16 ALL PASS

```
=== verify.py: Day 17 lab smoke test ===
=== Day 17 pipeline (lite) ===
  bronze rows in      : 13
  duplicates dropped  : 5  (Silver dedup)
  records quarantined : 3  (failed the gate)
  silver rows         : 8
  gold daily rows     : 5

Gold (completed orders by day):
order_date  n_orders  n_users  revenue  avg_order_value
2026-06-01         2        2    69.49            34.74
2026-06-02         1        1    19.99            19.99
2026-06-03         1        1    49.50            49.50
2026-06-04         1        1     5.00             5.00
2026-06-06         1        1    49.50            49.50
  [OK ] extract loaded raw rows
  [OK ] Silver dropped duplicates (the hook)
  [OK ] gate quarantined bad records
  [OK ] Gold produced daily rows
  [OK ] no duplicate order_id remains in Silver
  [OK ] streaming consumer is idempotent
  [OK ] doc->chunk->embedding ingestion
  [OK ] agent traces flattened into Bronze spans
  [OK ] eval golden set curated from traces
  [OK ] decontamination drops eval-overlapping pairs
  [OK ] at least one clean preference pair survives
  [OK ] ASOF point-in-time join avoids future leakage
  [OK ] knowledge graph built from docs
  [OK ] graph query answers 'what is returnable?' = widget only
  [OK ] graph answers a real 2-hop question (widget->accessory->warehouse)
  [OK ] vector foil: no single chunk answers the multi-hop question

RESULT: 16/16 checks — ALL PASS
```

## `make test` (`pytest -q`) — 18 passed

```
..................                                                        [100%]
18 passed in 12.89s
```
