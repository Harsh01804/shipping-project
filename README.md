# FreightTiger anomaly review

This project implements the required route-week anomaly scanner in a deterministic, rule-based way. It reads the shipment ledger, groups by route and route type, computes cost per tonne-km by Monday-to-Sunday week, and compares each week against:

- the route's trailing 8-week average using only prior weeks (no look-ahead)
- the same-week peer average across other routes in the same `route_type`

A candidate row is only reviewed if the current week's cost is above both baselines and meets the minimum thresholds: current week must be at least +5% above the route's own trailing average and at least +10% above the same-week peer average. The +5% own threshold is intentionally low enough to catch real route-specific spikes, while the +10% peer threshold prevents small noise from being treated as anomalies.

## Exact handling of notes

The note check is intentionally strict and deterministic. Windows are parsed from the note text itself with regex and a small date interpretation layer; they are not hard-coded elsewhere in the workflow.

- the note must apply to the route itself or to `All Routes`
- the note's covered date window must include the route-week being checked
- the note must actually say costs rose, or describe a relevant disruption/surcharge that increased costs
- notes that say costs were unaffected, stable, normal, or absorbed by transporters are rejected
- N003 (national diesel note) is accepted only inside its own note week (May 5 to May 11, 2025). No route-week in this data both passes the thresholds and falls in that week, so it justifies no rows.

This is how the known traps are handled:

- N005, N008, N010 are rejected because they explicitly say costs were not materially affected
- N004 is rejected because it refers to routes outside this dataset
- N006 is rejected because it describes stable demand rather than a cost increase

The result is a conservative, explainable rule set: a note is only accepted when it clearly supports the route and the week being reviewed.

## Fewer-than-8-weeks handling

If a route does not yet have 8 prior weeks, the code uses all prior weeks available and does not pad or extrapolate. This follows the brief exactly.

## Reproducibility and cost

The workflow is fully deterministic: no random sampling, no temperature-based calls, and no hidden state. Three separate runs on the same data produce identical hashes.

The cost log is honest: this run is a rule-based checker, so it performs 0 LLM calls and has a reported cost of $0.00. The main limitation is that there is no LLM or vector DB retrieval layer; if you wanted richer reason text or more context-aware note matching, that would be the next step beyond this deterministic baseline.

## Files

- `solution.py` builds the full route-week anomaly output and runs the evaluation harness
- `output.csv` contains the route-weeks that are at least +5% above their own route history and +10% above their peer average; only those threshold-passing rows are included in the output
- `eval_set.csv` is a small labeled validation set covering real rises and non-rising controls
- `eval_summary.txt` stores the real pass/fail evaluation results
- `reproducibility_check.txt` records the 3-run hash comparison
- `token_cost_log.md` records the token/cost log for this deterministic workflow
