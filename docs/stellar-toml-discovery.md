# Stellar TOML Discovery

This document covers how `src/stellar_toml.rs` fetches and parses
`stellar.toml` files, what a successful parse produces, how origin validation
works, and how to interpret retry and fallback outcomes.

---

## Overview

Discovery has three distinct steps that callers should not conflate:

1. **Fetch** — retrieve the raw TOML bytes from a well-known URL.
2. **Parse** — extract structured fields from those bytes.
3. **Origin validation** — confirm that declared endpoints belong to the same
   origin the document was fetched from.

A document that passes step 1 may fail step 2 (malformed fields). A document
that passes step 2 may fail step 3 (mismatched origin). Passing all three
establishes that the document is well-formed and structurally self-consistent;
it does **not** establish that the anchor behind those endpoints is trustworthy,
live, or compliant.

---

## Fetch

### URL construction

`fetch_stellar_toml_url(domain)` appends `/.well-known/stellar.toml` to the
domain after stripping any trailing slash. The domain must pass
`validate_anchor_domain` or the function returns `Err(InvalidEndpointFormat)`.

```
fetch_stellar_toml_url("https://anchor.example.com")
  → Ok("https://anchor.example.com/.well-known/stellar.toml")

fetch_stellar_toml_url("https://anchor.example.com/")
  → Ok("https://anchor.example.com/.well-known/stellar.toml")
```

### Size limit

The fetch layer enforces a 100 KiB cap (`MAX_STELLAR_TOML_BYTES = 102_400`).
A response body that exceeds this size is rejected with `InvalidEndpointFormat`
before the bytes reach the parser.

---

## Resilient fetch with retry

`fetch_stellar_toml_with_retry` is the production entry point. It accepts:

| Argument | Type | Purpose |
|---|---|---|
| `primary_domain` | `&str` | Anchor's HTTPS origin |
| `config` | `StellarTomlFetchConfig` | Retry parameters and optional fallback hosts |
| `fetch_fn` | `FnMut(&str) -> Result<String, AnchorKitError>` | Performs the actual HTTP GET |
| `sleep_fn` | `FnMut(u64)` | Called with delay in ms between retries; pass `\|_\| {}` in tests |
| `jitter_source` | `&mut impl JitterSource` | Provides per-attempt jitter seeds |

### Retry behaviour

The function tries the primary domain up to `config.retry.max_attempts` times
with exponential backoff. All errors from `fetch_fn` are treated as transient
(network or DNS failures); the retry loop does not distinguish HTTP 4xx from
5xx. After the primary domain exhausts its budget, each host in
`config.fallback_hosts` is tried with the same retry loop in order.

Default configuration:

```rust
StellarTomlFetchConfig {
    retry: RetryConfig::default(), // max_attempts = 3
    fallback_hosts: vec![],
}
```

### Return value

`fetch_stellar_toml_with_retry` returns `Ok(StellarTomlFetchResult)` on the
first successful response within the size limit:

```rust
pub struct StellarTomlFetchResult {
    pub resolved_url:  String, // the URL that succeeded
    pub raw_content:   String, // raw TOML bytes as a string
    pub used_fallback: bool,   // true when a fallback host was used
}
```

`used_fallback: true` means the primary domain and all its retries were
exhausted before a fallback host responded. The raw content is otherwise
treated identically regardless of which host provided it.

If every host and every retry is exhausted, the function returns the last
`AnchorKitError` from `fetch_fn`.

**Parse errors are not retried here.** `fetch_stellar_toml_with_retry` returns
the raw string; callers call `parse_stellar_toml` (or
`parse_stellar_toml_with_origin`) separately and handle parse errors themselves.

---

## Parse

`parse_stellar_toml(raw)` takes the raw TOML string and returns
`ParsedStellarToml`. The parser is section-aware: it tracks which table it is
currently inside so that currency-scoped keys (`code`, `issuer`, `status`) are
only collected while inside a `[[CURRENCIES]]` block. Keys from unrelated
sections such as `[DOCUMENTATION]` cannot pollute the asset list.

### Parsed fields

```rust
pub struct ParsedStellarToml {
    pub network_passphrase:        Option<String>,
    pub transfer_server:           Option<String>, // SEP-6
    pub transfer_server_sep0024:   Option<String>, // SEP-24
    pub kyc_server:                Option<String>,
    pub web_auth_endpoint:         Option<String>, // SEP-10
    pub signing_key:               Option<String>,
    pub direct_payment_server:     Option<String>, // SEP-31
    pub anchor_quote_server:       Option<String>, // SEP-38
    pub supported_assets:          Vec<String>,    // de-duplicated code list
    pub currencies:                Vec<ParsedCurrency>,
}
```

Every URL field (`transfer_server`, `transfer_server_sep0024`, `kyc_server`,
`web_auth_endpoint`, `direct_payment_server`, `anchor_quote_server`) is
validated with `validate_anchor_domain` as it is parsed. An invalid URL is a
hard parse error; the rest of the document is not returned.

Non-URL fields (`network_passphrase`, `signing_key`) and all currency fields
are optional. When absent they parse to `None` or empty rather than failing.

### Currency entries

Each `[[CURRENCIES]]` block produces one `ParsedCurrency`:

```rust
pub struct ParsedCurrency {
    pub code:   String,         // required; block is dropped if missing
    pub issuer: Option<String>, // empty issuer is a hard error
    pub status: Option<String>,
}
```

A block without a `code` field is a hard parse error. A block with a present
but empty `issuer` field is also a hard parse error.

`supported_assets` is a de-duplicated list of `code` values derived from
`currencies`, kept for backward compatibility. Richer detail is in `currencies`.

### Convenience methods

```rust
parsed.supports_sep6()   // transfer_server.is_some()
parsed.supports_sep24()  // transfer_server_sep0024.is_some()
parsed.supports_sep10()  // web_auth_endpoint.is_some()
parsed.supports_sep31()  // direct_payment_server.is_some()
parsed.supports_sep38()  // anchor_quote_server.is_some()

// Both web_auth_endpoint and signing_key must be present.
parsed.is_sep10_complete()

// Look up a currency by code.
parsed.find_currency("USDC")

// Deduplicated (label, url) pairs for distinct service endpoints.
parsed.discovered_endpoints()
```

`is_sep10_complete()` requires both `web_auth_endpoint` and `signing_key`
because SEP-10 challenge verification needs both. `supports_sep10()` only
checks for the endpoint and is a looser test.

`discovered_endpoints()` deduplicates by URL in declaration order: if two
fields point at the same URL, only the first field's label is returned. This
prevents callers from probing the same host twice.

---

## Origin validation

`parse_stellar_toml_with_origin(raw, source_domain)` calls `parse_stellar_toml`
and then checks that the parsed `TRANSFER_SERVER` URL, if present, shares the
same `scheme://host[:port]` origin as `source_domain`. Origin comparison is
case-insensitive.

If the origins differ the function returns `Err(InvalidEndpointFormat)` with a
context string identifying the mismatched endpoint. All other fields are parsed
exactly as `parse_stellar_toml` would parse them; only the primary endpoint is
pinned.

This check prevents a document fetched from one host from silently redirecting
callers to an unrelated `TRANSFER_SERVER`. It does not establish that the anchor
is authorized to operate that endpoint or that the endpoint is compliant.

### What origin validation establishes and does not establish

| It establishes | It does not establish |
|---|---|
| `TRANSFER_SERVER` is on the same origin as the fetched document | The anchor controls that domain |
| The document is structurally valid TOML with recognized fields | Declared endpoints are live or responding |
| URL fields passed domain validation | The anchor is SEP-compliant |
| Currency blocks have required `code` fields | Issuer accounts exist on the network |

---

## Minimal working example

```rust
use anchorkit::stellar_toml::{
    fetch_stellar_toml_with_retry, parse_stellar_toml_with_origin,
    StellarTomlFetchConfig,
};
use anchorkit::retry::MockJitterSource;

let config = StellarTomlFetchConfig::default(); // 3 attempts, no fallbacks
let mut js = MockJitterSource::new(vec![0u64; 10]);

// Step 1: fetch raw bytes
let fetch_result = fetch_stellar_toml_with_retry(
    "https://anchor.example.com",
    &config,
    |url| {
        // inject real HTTP client here; return raw TOML string
        Ok(format!("TRANSFER_SERVER = \"https://anchor.example.com/sep6\"\n\
                    WEB_AUTH_ENDPOINT = \"https://anchor.example.com/auth\"\n\
                    SIGNING_KEY = \"GABCDEF...\"\n\
                    [[CURRENCIES]]\n\
                    code = \"USDC\"\n\
                    issuer = \"GA5ZSE...\"\n"))
    },
    |_ms| {}, // no real sleep in tests
    &mut js,
).unwrap();

assert_eq!(fetch_result.used_fallback, false);

// Step 2: parse with origin check
let parsed = parse_stellar_toml_with_origin(
    &fetch_result.raw_content,
    "https://anchor.example.com",
).unwrap();

assert!(parsed.supports_sep6());
assert!(parsed.is_sep10_complete());
assert_eq!(parsed.supported_assets, vec!["USDC"]);
assert_eq!(parsed.find_currency("USDC").unwrap().issuer.as_deref(),
           Some("GA5ZSE..."));
```

---

## Failure modes and their meaning

| Error | Trigger | Meaning |
|---|---|---|
| `InvalidEndpointFormat` (fetch) | Domain fails `validate_anchor_domain` | The domain string itself is malformed |
| `InvalidEndpointFormat` (size) | Response body > 100 KiB | Document is unusually large; rejected before parsing |
| Last `fetch_fn` error | All hosts and retries exhausted | Network/DNS failure; no document was retrieved |
| `InvalidEndpointFormat` (parse) | A URL field contains an invalid value | Document declares a malformed endpoint |
| `ValidationError` ("CURRENCIES block missing code field") | A `[[CURRENCIES]]` block has no `code` key | Document is structurally invalid |
| `ValidationError` ("CURRENCIES block has empty issuer field") | A `[[CURRENCIES]]` block has `issuer = ""` | Document is structurally invalid |
| `InvalidEndpointFormat` (origin) | `TRANSFER_SERVER` origin ≠ source domain | Document was redirected to or declares a cross-origin server |

---

## Source references

- Full implementation: `src/stellar_toml.rs`
- URL construction: `src/stellar_toml.rs` — `fetch_stellar_toml_url`
- Parser: `src/stellar_toml.rs` — `parse_stellar_toml`
- Origin check: `src/stellar_toml.rs` — `parse_stellar_toml_with_origin`
- Resilient fetch: `src/stellar_toml.rs` — `fetch_stellar_toml_with_retry`
- Size constant: `src/stellar_toml.rs` — `MAX_STELLAR_TOML_BYTES`
- Shell workflow example: `examples/anchor_info_discovery.sh`
