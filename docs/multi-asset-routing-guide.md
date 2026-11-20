# Multi-Asset Routing: Input/Output Guide

This guide walks through one complete routing call — a single `AssetPairRequest`
against a small candidate pool — and explains how `route_multi_asset` produces
its result. All values are taken from or directly consistent with the fixtures
used in `src/multi_asset_routing.rs` tests.

---

## Key types

| Type | Role |
|---|---|
| `AssetPairRequest` | One corridor to route, with strategy and minimum reputation |
| `CandidateQuote` | An on-chain quote record available for selection |
| `AssetPairQuote` | The winning quote for a filled corridor |
| `MultiAssetRoutingResult` | Collects `filled` and `unfilled` corridors |

---

## Routing strategies

Exactly four strategy labels are accepted. Any other string — including
different casing or trailing whitespace — causes `ValidationError` before
candidate selection runs.

| Label | Selection rule |
|---|---|
| `"LowestFee"` | Candidate with the smallest `fee_percentage` |
| `"FastestSettlement"` | Candidate with the smallest `average_settlement_time` |
| `"HighestReputation"` | Candidate with the largest `reputation_score` |
| `"WeightedScore"` | Highest composite score: `fee_score·fw + speed_score·sw + rep_score·rw` |

For `WeightedScore`, each dimension is normalised against a **fixed ceiling**
(`fee` and `reputation` capped at 100, settlement time at `f32::MAX`), not
against the current candidate pool. This means a candidate's score is
independent of how many others are present.

When `route_multi_asset` calls `select_best` internally it passes equal weights
`(0.333, 0.333, 0.334)`. Callers invoking `select_best` directly can supply
their own weights.

---

## Candidate filtering

Before strategy selection, `route_multi_asset` filters the full quote list down
to candidates that satisfy all of the following for a given request:

1. `base_asset` and `quote_asset` normalize to the request's corridor.
2. `base_asset != quote_asset` (self-routes excluded).
3. `rate > 0`.
4. `valid_until > now_timestamp` (expired quotes excluded).
5. `amount >= minimum_amount`.
6. `maximum_amount == 0` **or** `amount <= maximum_amount` (`0` means no upper bound).
7. `reputation_score >= min_reputation`.

---

## Worked example

### Input

Three candidates are available for the `XLM/USDC` corridor:

```rust
// quote_id, base, quote, fee_percentage, reputation_score, average_settlement_time
candidate(1, "XLM", "USDC", fee=50, rep=80, time=60)
candidate(2, "XLM", "USDC", fee=20, rep=90, time=120)
candidate(3, "XLM", "USDC", fee=35, rep=70, time=30)
```

All three have:
- `rate = 1_000_000`
- `minimum_amount = 1`, `maximum_amount = 0` (unbounded)
- `valid_until = u64::MAX`

The request:

```rust
AssetPairRequest {
    base_asset:     "XLM".into(),
    quote_asset:    "USDC".into(),
    amount:         100,
    strategy:       "LowestFee".into(),
    min_reputation: 0,
}
```

`now_timestamp = 0`

### Selection per strategy

This table matches the assertions in `every_accepted_strategy_maps_to_its_rule`:

| Strategy | Winner | Reason |
|---|---|---|
| `LowestFee` | quote_id **2** | `fee=20` is lowest |
| `FastestSettlement` | quote_id **3** | `time=30` is smallest |
| `HighestReputation` | quote_id **2** | `rep=90` is highest |
| `WeightedScore` (fee-only weights: `fw=1.0, sw=0.0, rw=0.0`) | quote_id **2** | best fee score |

### Output for `LowestFee`

```rust
MultiAssetRoutingResult {
    filled: [
        AssetPairQuote {
            pair_key:         "XLM/USDC",
            anchor:           "anchor-2",
            fee_percentage:   20,
            rate:             1_000_000,
            quote_id:         2,
            strategy_applied: "LowestFee",
            routing_reason:   None,
        }
    ],
    unfilled: [],
}
```

---

## Unfilled corridors

When no candidate passes all filters for a corridor, that corridor's key is
appended to `unfilled` and no entry is added to `filled`. The call still
returns `Ok` — an empty candidate list is not an error.

```rust
// No candidates exist for BTC/USDC
AssetPairRequest { base_asset: "BTC", quote_asset: "USDC", amount: 100,
                   strategy: "LowestFee", min_reputation: 0 }

// Result:
MultiAssetRoutingResult {
    filled:   [],
    unfilled: ["BTC/USDC"],
}
```

Callers must inspect `unfilled` to detect corridors that received no quote.

---

## Error propagation

`route_multi_asset` processes requests in submission order and returns the
**first error** encountered. A single invalid request in a batch stops
processing for the entire call; no partial `filled` list is returned.

| Condition | Error code |
|---|---|
| Empty, >12-char, or non-ASCII-alphanumeric asset code | `InvalidAssetCode` |
| `base_asset == quote_asset` after normalization | `InvalidAssetPair` |
| `amount == 0` | `InvalidAmount` |
| Unrecognised strategy label | `ValidationError` |
| A candidate quote has an empty asset code | `InvalidAssetCode` |

---

## Multi-pair call

Requests are independent: each `AssetPairRequest` can carry a different
strategy and `min_reputation`. Results in `filled` are in submission order.

```rust
route_multi_asset(
    &[
        AssetPairRequest { base_asset: "XLM",  quote_asset: "USDC", strategy: "LowestFee", .. },
        AssetPairRequest { base_asset: "USDC", quote_asset: "BTC",  strategy: "HighestReputation", .. },
    ],
    &all_quotes,
    now_timestamp,
)
```

If the second request has no eligible candidates, `filled` will contain the
XLM/USDC result and `unfilled` will contain `"USDC/BTC"`.

---

## Source references

- Types and engine: `src/multi_asset_routing.rs`
- Strategy selection: `src/multi_asset_routing.rs` — `select_best`
- Candidate filtering: `src/multi_asset_routing.rs` — `route_multi_asset`
- Strategy correctness test: `src/multi_asset_routing.rs` — `every_accepted_strategy_maps_to_its_rule`
- Unfilled / error propagation tests: module-level doc comment in `src/multi_asset_routing.rs`
