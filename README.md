<!-- solve-problem:49de9d3f-3ea9-460b-aaee-ffd7b63c70ef -->
# Counterfactual ground truth for MEV detectors

A [Tangle Solve problem](https://solve.tangle.tools/problems/49de9d3f-3ea9-460b-aaee-ffd7b63c70ef).

## What is the problem?

MEV (Maximal Extractable Value) detectors on Ethereum and similar chains are heuristics that label on-chain transaction sequences as "arbitrage," "sandwich attack," "liquidation," or "benign," and attach a value estimate. There is no ground truth: the same heuristic that produced a label is typically used to evaluate it. The task is to construct **counterfactually certified traces** — labeled transaction-sequence datasets where each label and value is derived independently of the detector, via counterfactual simulation.

Formally, for a transaction sequence σ executed at block height h, one must produce:

1. **Extractable value** EV(σ): the maximum value an optimal adversary could have extracted in σ's position (requires solving an optimization over the block, e.g., via an MEV-aware order-flow auction or dynamic program over a batched AMM/limit-order state).
2. **Realized profit** RP(σ): what the actual extractor netted, including gas/bribes (coinbase transfers, priority fees) — computable from trace data but requiring careful accounting of failed transactions and token price normalization.
3. **User harm** UH(σ): the counterfactual user outcome — the execution price/slippage the user *would* have received absent the extractor's transactions. This is the hard part: it requires re-executing the block with the adversarial transactions removed (or repositioned) on a faithful EVM simulator (e.g., a fork of geth/erigon/reth or a tool like `revm`), then valuing the difference.

The dataset must include **failures** (reverted extraction attempts — these look identical to successes in mempool data and are systematically miscounted) and **benign lookalikes** (e.g., coincidental adjacent trades, arbitrage that *reduces* price dislocation and benefits the user). Each record carries an explicit **uncertainty certificate**: assumptions under which the counterfactual is valid (e.g., "no other agent would have inserted txs in the freed block space").

## Why does it matters?

Every major MEV quantification — the Flash Boys 2.0 arbitrage profits, sandwich-attack incidence estimates, the ~$XXXM figures used in PBS/ePBS design debates, and the evaluations of production detectors — rests on self-labeled heuristics whose error rates are unknown. A certified benchmark would (a) let researchers report detector precision/recall instead of raw counts, (b) settle disputes about how much MEV actually harms users versus is benign arbitrage, and (c) provide the evaluation substrate for protocol interventions (order-flow auctions, encrypted mempools, proposer-builder separation) whose design tradeoffs currently lean on unvalidated measurements.

## What's already known?

- **Daian et al. (2019)** established the measurement paradigm: label atomic arbitrage via a bot-fingerprint heuristic (high-frequency identical addresses, gas-price competition) and compute profits from DEX reserves. No counterfactuals, no sandwich analysis, no failure accounting.
- Subsequent measurement work (e.g., quantitative sandwich studies, EigenPhi-style dashboards,.flashbots MEV-explore) improved labeling heuristics and coverage but all remain **self-evaluating** — no independent ground truth, no published precision/recall against certified labels.
- **Counterfactual simulation tooling exists**: Ethereum state replay (forked reth/erigeron, revm), Flashbots' `mev-inspectors-py` and the FlashbotsBlocks/`flashbots-mocha` RPC for proposer payments. No one has combined block re-execution with optimal-extraction solvers into a labeled, uncertainty-certified public corpus. (Verify: check whether Flashbots or academic groups have released such a corpus post-2023.)
- Known hard issues: gas accounting across reorgs, token price normalization (whose price oracle?), the "counterfactual reordering" problem (removing an extractor's tx changes block packing, so the counterfactual block is itself non-unique), and adversarial causal-inference framing (the extractor's tx is a *treatment*; block construction is confounded by the builder).

## What would count as solving it?

1. **A public dataset** of ≥10⁴–10⁵ labeled MEV events across ≥1 year of Ethereum (and ideally one other chain), each with: raw trace, EV via an optimal-extraction solver, RP with full cost accounting, UH via re-executed counterfactual blocks, a class label (arbitrage / sandwich / liquidation / failure / benign lookalike), and a machine-readable uncertainty statement (e.g., bounds over a set of plausible counterfactual block constructions).
2. **Validation**: at least a stratified human/committee audit of a sample confirming labels, plus sensitivity analysis showing UH bounds are stable across counterfactual construction choices.
3. **Demonstrated use**: precision/recall of ≥2 published detectors measured against the corpus, with the gap from self-labeled estimates quantified.
Solving variants: (a) full corpus as above; (b) a smaller "gold set" (hundreds of events) with exhaustive manual certification — acceptable as a first milestone; (c) a methodological contribution proving impossibility/identifiability limits (which counterfactual quantities *cannot* be point-identified from on-chain data alone) — this alone would be publishable.

## How to start

1. Read Daian et al. (2019) §4–5 (measurement methodology) to understand the original labeling heuristics you must improve on; then survey current detectors (EigenPhi taxonomy, `mev-inspectors-py`) to enumerate label classes and known lookalike confusions.
2. Build the counterfactual engine first on a narrow, high-signal class: **single-pool sandwich attacks on UniswapV2-style AMMs**, where the counterfactual (user trade executed against pre-attack reserves) is mathematically clean and re-execution is straightforward with `revm`/forked reth. Extend to multi-hop and failures.
3. For EV, adapt existing batch-auction optimal-arbitrage solvers (the Flash Boys appendix formulates atomic arb as a shortest-path/optimization problem); for sandwich EV, solve the attacker's optimal front/back-size optimization analytically (closed forms exist for constant-product AMMs).
4. The main obstacle is **block-construction confounding**: the counterfactual block is not unique. Handle it by reporting interval-valued harm over an explicitly enumerated set of plausible fill transactions, and say so in the certificate.

## Key references

- Daian, P., Goldfeder, S., Kell, T., Li, Y., Zhao, X., Bentov, I., Breidenbach, L., Juels, A. (2019). "Flash Boys 2.0: Frontrunning, Transaction Reordering, and Consensus Instability in Decentralized Exchanges." arXiv:1904.05234. Establishes MEV as a concept, measures atomic arbitrage and gas auctions on DEXs with heuristic labels; the measurement baseline this benchmark must supersede. https://arxiv.org/abs/1904.05234
- Qin, K., Zhou, L., Gervais, A. (2022). "Quantifying Blockchain Extractable Value: How dark is the forest?" IEEE S&P. Formalizes EV search over multiple protocols and simulates optimal attacks — closest existing work to "extractable value" computation. https://arxiv.org/abs/2101.05511 (verify version/venue)
- Flashbots, `mev-inspectors-py` (open-source repository). Production heuristic detector whose output exemplifies the self-labeling problem and provides trace-parsing infrastructure to build on. https://github.com/flashbots/mev-inspectors-py
- (Suggested, verify) Works on counterfactual/re-execution measurement of sandwich harm, e.g., analyses accompanying encrypted-memppool proposals (Shutter, threshold encryption literature) — search for "sandwich attack user loss measurement" 2021–2024 for the closest prior counterfactual attempts.

## Warnings

- **Self-labeling is the failure mode being attacked**: never evaluate using the same heuristic that generated labels; a detector agreeing with mev-inspectors-py labels is not evidence of correctness.
- **Counterfactual non-uniqueness**: removing a sandwich's transactions leaves block space that *would* have been filled by something else; a point estimate of user harm is generally unjustified. Use intervals.
- **Failed transactions are systematically mishandled**: reverted extraction attempts still pay gas and appear in mempool-adjacent data; excluding them inflates extractor competence and profit figures.
- **Value normalization**: profits in tokens must be denominated somewhere; the choice of price oracle/time shifts results — fix and disclose it.
- **Benign lookalikes**: much "frontrunning-like" ordering is innocuous or user-beneficial arbitrage; a corpus without a benign class teaches detectors to overfire.
- **Reorg sensitivity**: traces near chain reorganizations can be duplicated or orphaned; deduplicate carefully.

---
*`BCGT-000109` · unverified*

## Acceptance guidance

1. **A public dataset** of ≥10⁴–10⁵ labeled MEV events across ≥1 year of Ethereum (and ideally one other chain), each with: raw trace, EV via an optimal-extraction solver, RP with full cost accounting, UH via re-executed counterfactual blocks, a class label (arbitrage / sandwich / liquidation / failure / benign lookalike), and a machine-readable uncertainty statement (e.g., bounds over a set of plausible counterfactual block constructions).
2. **Validation**: at least a stratified human/committee audit of a sample confirming labels, plus sensitivity analysis showing UH bounds are stable across counterfactual construction choices.
3. **Demonstrated use**: precision/recall of ≥2 published detectors measured against the corpus, with the gap from self-labeled estimates quantified.
Solving variants: (a) full corpus as above; (b) a smaller "gold set" (hundreds of events) with exhaustive manual certification — acceptable as a first milestone; (c) a methodological contribution proving impossibility/identifiability limits (which counterfactual quantities *cannot* be point-identified from on-chain data alone) — this alone would be publishable.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/49de9d3f-3ea9-460b-aaee-ffd7b63c70ef?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000109.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000109)
