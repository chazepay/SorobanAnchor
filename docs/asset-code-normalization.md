# Asset Code Normalization

This document describes how asset codes are validated and normalized before
they participate in routing. The behavior is defined in two places:

- `src/errors.rs` — `normalize_asset_code()`: the canonical, fallible version
  used across the codebase whenever an asset code enters the system from an
  external source (deposit response, withdrawal response, etc.).
- `src/multi_asset_routing.rs` — `normalize_asset_code()`: a local, infallible
  helper used internally by the routing engine after validation has already
  passed.

---

## What normalization does

Both functions apply the same transformation:

1. **Trim** surrounding ASCII whitespace.
2. **Uppercase** ASCII letters (`to_ascii_uppercase`).

That is the complete transformation. No characters are removed, transliterated,
or substituted. The result is returned as-is if it passes validation.

---

## Validation rules

`errors::normalize_asset_code` returns `Err(InvalidAssetCode)` when any of the
following are true after trimming:

| Rule | Detail |
|---|---|
| Empty | The trimmed string has length 0 |
| Too long | The trimmed string has length > 12 |
| Non-ASCII-alphanumeric character | Any character that is not `[A-Za-z0-9]` |

`validate_asset_pair_request` in the routing module enforces the same rules
(length 1–12, ASCII alphanumeric only) before a routing call proceeds.

---

## Examples

All examples are derived from tests in `src/errors.rs` and
`src/multi_asset_routing.rs`.

### Accepted inputs

| Input | Normalized output |
|---|---|
| `"usdc"` | `"USDC"` |
| `"XLM"` | `"XLM"` |
| `"Eurc"` | `"EURC"` |
| `"  USDC  "` | `"USDC"` (whitespace trimmed) |
| `"  Xlm "` | `"XLM"` (whitespace trimmed, then uppercased) |
| `"abc123XYZ789"` | `"ABC123XYZ789"` (12 chars — maximum length) |
| `"ABCDEFGHIJKL"` | `"ABCDEFGHIJKL"` (12 chars — maximum length) |

### Rejected inputs

| Input | Reason |
|---|---|
| `""` | Empty after trim |
| `"TOOLONGCODE13"` | 13 characters — exceeds 12-char limit |
| `"BAD CODE"` | Space is not ASCII alphanumeric |
| `"USD-C"` | Hyphen is not ASCII alphanumeric |
| `"USD.C"` | Period is not ASCII alphanumeric |
| `"USD@C"` | `@` is not ASCII alphanumeric |
| `"USD/C"` | Slash is not ASCII alphanumeric |
| `"USDC!"` | `!` is not ASCII alphanumeric |

### Unicode inputs are always rejected

The uppercasing step uses `to_ascii_uppercase`, not Unicode case folding. A
non-ASCII character that looks like a Latin letter — for example the long `s`
(`ſ`, U+017F), which Unicode would map to `S` — is **not** uppercased and
therefore fails the ASCII-alphanumeric check.

| Input | Why it is rejected |
|---|---|
| `"uſdc"` | `ſ` (U+017F) is not ASCII alphanumeric |
| `"ÜSDC"` | `Ü` (U+00DC) is not ASCII alphanumeric |
| `"ＵＳＤＣ"` | Fullwidth letters are not ASCII alphanumeric |
| `"USDC\u{200B}"` | Zero-width space is not ASCII alphanumeric |

This means a non-ASCII spelling can never alias a valid ASCII code, regardless
of Unicode case equivalence.

---

## What normalization does not check

Normalization only verifies the **code string** itself. It does not:

- Validate that an issuer account exists on the Stellar network.
- Confirm that a code/issuer pair is registered with any anchor.
- Check that the asset is live, tradeable, or has liquidity.

An asset code that passes `normalize_asset_code` is structurally valid for
routing purposes. Whether the corresponding asset is real and usable is
determined by the routing candidate pool and any on-chain checks the caller
performs separately.

---

## Pair key construction

The routing engine derives a corridor identifier from two normalized codes:

```
pair_key("usdc", "xlm")  →  "USDC/XLM"
pair_key("  Xlm ", "USDC")  →  "XLM/USDC"
```

The pair key is `"<BASE>/<QUOTE>"` where both sides have been through
`normalize_asset_code`. This key is used as the map key in
`MultiAssetRoutingResult` and in the `unfilled` list.

---

## Source references

- Canonical implementation: `src/errors.rs` — `normalize_asset_code`
- Routing-local version: `src/multi_asset_routing.rs` — `normalize_asset_code`, `pair_key`
- Charset predicate: `src/multi_asset_routing.rs` — `is_asset_code_charset`
- Tests (charset, unicode, punctuation): `src/multi_asset_routing.rs` — `valid_codes_normalize_as_before`, `unicode_asset_codes_rejected`, `punctuation_asset_codes_rejected`
- Tests (length, empty, whitespace): `src/errors.rs` — `test_normalize_asset_code_*`
