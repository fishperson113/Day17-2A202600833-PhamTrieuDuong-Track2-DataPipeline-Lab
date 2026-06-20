# Reflection — Day 17

**1. The flywheel — most silent failure:** `decontaminate`. If its normalization has a bug, the pipeline writes `preference_pairs.jsonl` with no error while eval prompts silently leak into training. Detect it: assert `len(raw_pairs) > len(clean_pairs)` after every flywheel run when the eval set is non-empty. A zero-drop run with a non-empty eval set is a red flag.

**2. Decontamination — what breaks if skipped:** The 2 dropped pairs share exact prompts with the eval set. The model memorizes those inputs during training; eval accuracy inflates while generalization does not improve. The lie surfaces in production A/B tests: eval score looks high but user-satisfaction metrics don't move.

**3. Point-in-time — dangerous feature without ASOF:** `credit_score` in a fraud model. Joining "current score" to a historical transaction leaks future repayments into the training row. The model learns: high future score → low fraud risk. At inference time that future score doesn't exist, so the model silently over-estimates safety.

**4. Graph vs vector:**
- **Graph wins:** *"Where does a widget ship from?"* — requires widget → IS_A → accessory → SHIPS_FROM → Hanoi (2 hops). No single chunk contains both facts; flat retrieval fails regardless of embedding quality.
- **Graph overkill:** *"Is a widget returnable?"* — one sentence answers it. Single-chunk retrieval is sufficient; graph traversal adds latency with no benefit.
