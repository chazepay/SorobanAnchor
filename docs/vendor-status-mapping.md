# Vendor Status Mapping

This document describes the `VendorStatusMap` data structure in `src/sep6.rs`
and the procedure for adding a vendor-specific status mapping safely.

---

## Background

SEP-6 defines a fixed set of transaction status strings (`pending_external`,
`completed`, `error`, etc.). Individual anchors sometimes return proprietary
status strings that have no direct SEP-6 equivalent — for example
`"ach_processing"`, `"kyc_required"`, or `"fx_pending"`. Without a mapping
layer these strings fall through to `TransactionStatus::Error`, which is
technically correct (unknown = failed) but loses meaningful intermediate state.

`VendorStatusMap` lets you register anchor-specific strings once and resolve
any raw status against them, falling back to the standard
`TransactionStatus::from_str` parser for values that are not registered.

---

## Data structure

```rust
// src/sep6.rs

pub struct VendorStatusEntry {
    pub vendor_status: String,       // normalized (trimmed, lowercased) key
    pub canonical:     TransactionStatus,
}

pub struct VendorStatusMap {
    entries: Vec<VendorStatusEntry>, // searched linearly; order is insertion order
}
```

The map stores entries as a `Vec`. Lookups are O(n) — appropriate for the small
number of vendor-specific statuses a single anchor typically uses.

All keys are stored and matched in trimmed, lowercased form. `register`,
`resolve`, `contains`, and `remove` all normalize their input through
`normalize_status_token` (trim + `to_ascii_lowercase`) before any comparison.

---

## Resolution order

`resolve(raw)` applies this sequence:

1. Normalize `raw` (trim whitespace, ASCII-lowercase).
2. Search the vendor map for an exact match on the normalized key.
3. If found, return the registered `canonical` value with the original trimmed
   raw string preserved in `vendor_status`.
4. If not found, call `TransactionStatus::from_str(raw)`, which covers all
   standard SEP-6 strings and maps anything unrecognized to
   `TransactionStatus::Error`.

The original raw string is always preserved in the returned `VendorStatusEntry`
so callers can log or forward vendor detail without losing it.

---

## Procedure for adding a mapping

### Step 1 — Identify the exact raw string

Obtain the status string exactly as the anchor returns it. Do not guess or
derive it from documentation alone; use a real response or an anchor-provided
test fixture. Record the string verbatim, including any casing.

### Step 2 — Choose the canonical SEP-6 status

Select the `TransactionStatus` variant that best describes the state. Consult
the SEP-6 specification for definitions:
https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md#transaction-history

The full set of variants and their SEP-6 meanings:

| Variant | SEP-6 string | Meaning |
|---|---|---|
| `PendingExternal` | `pending_external` | Waiting for action outside Stellar |
| `PendingAnchor` | `pending_anchor` | Anchor is processing |
| `PendingTrust` | `pending_trust` | Waiting for user to establish trustline |
| `PendingUser` | `pending_user` | Waiting for user action |
| `PendingStellar` | `pending_stellar` | Waiting for on-chain confirmation |
| `WaitingCustomerAction` | `waiting_customer_action` | Customer must act |
| `Pending` | `pending` | Generic pending |
| `Incomplete` | `incomplete` | Not yet started |
| `Completed` | `completed` | Successfully finished |
| `Refunded` | `refunded` | Returned to sender |
| `Expired` | `expired` | Timed out |
| `NoMarket` | `no_market` | No market for this pair |
| `TooSmall` | `too_small` | Below minimum |
| `TooLarge` | `too_large` | Above maximum |
| `Error` | `error` | Failed or unrecognized |

Prefer a specific variant over a broad one. `"ach_processing"` maps more
accurately to `PendingExternal` (waiting on a bank) than to the generic
`Pending`.

### Step 3 — Register the mapping

```rust
let mut map = VendorStatusMap::new();
map.register("ach_processing", TransactionStatus::PendingExternal);
map.register("kyc_required",   TransactionStatus::PendingUser);
```

`register` returns `false` and does nothing if the key is empty after
normalization. If the key is already registered, the existing entry is
**replaced** in-place.

### Step 4 — Write tests covering three cases

Every new mapping requires tests for:

**a) Exact match** — the registered string resolves to the expected canonical value:

```rust
let r = map.resolve("ach_processing");
assert_eq!(r.canonical, TransactionStatus::PendingExternal);
assert_eq!(r.vendor_status, "ach_processing");
```

**b) Normalization** — casing and whitespace variants of the same string resolve
identically:

```rust
// From test_vendor_status_map_lookup_is_case_insensitive in src/sep6.rs
assert_eq!(map.resolve("ACH_PROCESSING").canonical, TransactionStatus::PendingExternal);
assert_eq!(map.resolve("Ach_Processing").canonical, TransactionStatus::PendingExternal);
assert!(map.contains("ACH_Processing"));
```

**c) Unknown fallback** — a string that is not in the map and is not a standard
SEP-6 string resolves to `TransactionStatus::Error`, not to the new mapping:

```rust
let r = map.resolve("totally_unknown");
assert_eq!(r.canonical, TransactionStatus::Error);
assert_eq!(r.vendor_status, "totally_unknown");
```

These three cases mirror the pattern established in
`test_vendor_status_map_lookup_is_case_insensitive` and the `VendorStatusMap`
doc-comment examples in `src/sep6.rs`.

---

## What not to do

**Do not register a catch-all alias.** A mapping like
`map.register("", TransactionStatus::Pending)` is silently rejected (`register`
returns `false` for empty keys). Do not work around this by mapping `"unknown"`
or `"*"` to a non-error status; unknown strings must remain `TransactionStatus::Error`
so upstream callers can detect and log them.

**Do not add speculative vendor strings.** Only register strings you have
observed in real anchor responses or in an anchor's published test fixtures.
Registering a guessed string that the anchor never actually returns is harmless
in isolation, but it clutters the map and can silently absorb typos that should
surface as errors.

**Do not share one map across anchors.** Each `VendorStatusMap` should cover
one anchor's quirks. Merging mappings from different anchors risks one anchor's
proprietary string aliasing a standard SEP-6 string from another.

---

## Removing a mapping

```rust
let removed = map.remove("ach_processing");
assert!(removed); // true when the key existed
```

`remove` returns `false` if the key was not registered. The same normalization
applies, so `map.remove("ACH_PROCESSING")` removes the entry registered as
`"ach_processing"`.

---

## Source references

- Data structure and all methods: `src/sep6.rs` — `VendorStatusMap`, `VendorStatusEntry`
- Normalization helper shared with status parsing: `src/sep6.rs` — `normalize_status_token`
- Case-insensitivity test: `src/sep6.rs` — `test_vendor_status_map_lookup_is_case_insensitive`
- Doc-comment examples (exact match, standard fallback, unknown fallback): `src/sep6.rs` — `VendorStatusMap` rustdoc
- SEP-6 status definitions: https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0006.md#transaction-history
