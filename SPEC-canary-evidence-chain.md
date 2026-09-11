# SPEC — Canary Evidence Chain

Status: **public behaviour specification, version 1.0.** Specification release only.
No implementation, Tenable listing, Exchange submission, PR, or Contribution Agreement
acceptance is authorized by this document.

Product form: skill + MCP server.
Written from product behaviour and capability intent. No harvested body was quoted,
translated, or structurally reproduced in producing this specification.

---

## 1. Purpose

An operator wants to know whether their **own fabricated test records** are leaking
inside their **own systems** — appearing in an agent's output, a log, a support reply, a
model response, or a downstream index they never authorized.

The product mints canary markers into package-generated synthetic datasets, detects those
markers in text the operator hands it, and records each trip as a hash-linked,
payload-free evidence entry compatible with the Agent Action Evidence Ledger.

**Hard product boundary:** the package can only mark data it fabricated itself, and it can
only look at text the operator already holds. It has no path to third-party data, no
network capability, and no ability to affect anyone outside the calling process.

Non-goals: general DLP, secret scanning, surveillance, identity resolution, notification,
or active response.

## 2. Definitions

- **Synthetic dataset** — a record set produced by the package's own seeded generator from
  an explicit schema. The only kind of data a canary may be embedded in.
- **Canary marker** — an opaque token embedded in a synthetic record.
- **Recipient label** — an opaque operator-chosen string identifying an internal
  destination. Never resolved, contacted, or joined against anything.
- **Trip** — a detection of a marker in operator-supplied text.
- **Evidence entry** — a payload-free, hash-linked record of a trip.

## 3. Inputs

### 3.1 `generate_synthetic_dataset`
`{ schema, recordCount ≤ 100000, seed }` → a `SyntheticDataset` handle.
The generator reads no file, no network, no environment, and no clipboard. Identical
`(schema, seed)` produces identical records.

### 3.2 `mint_canary`
`{ dataset: SyntheticDataset, recordRef, recipientLabel, method }`

- `dataset` **must** be a handle returned by `generate_synthetic_dataset`. There is no
  public constructor, no cast path in the published type surface, and no overload
  accepting a raw string, object, buffer, or file path. A caller holding real data has no
  type-legal way to reach this call.
- `recipientLabel` matches `^[A-Za-z0-9_-]{1,64}$` and is additionally **rejected** when it
  matches a real-world identifier shape: email, URL, hostname, IP, phone, IBAN, payment
  card, cloud ARN, or cloud access-key pattern.
- `method` ∈ `field-value`, `record-append`, `unique-key`.

### 3.3 `scan_for_canaries`
`{ text: string | AsyncIterable<string>, maxChars? }` — operator-supplied text only. The
scanner never opens a file, fetches a URL, or subscribes to a stream. The default and hard
maximum character limits are returned by `describe_policy`; crossing either stops before
examining more input and returns `E_LIMIT`.

### 3.4 `register_honeypot`
`{ dataset, recordRef, label }` — the record reference must be a handle into a synthetic
dataset. A honeypot cannot be registered against a record the package did not fabricate.

### 3.5 Evidence sink adapter
An optional caller-supplied sink provides atomic `append(entry, expectedHead)` and returns
confirmation or failure. Without a sink, evidence is session-local and explicitly reported
as non-durable. The package never opens a sink itself.

### 3.6 `report_access`, `get_stats`, `reset_state`, `export_evidence`, `describe_policy`
Local operations over the current session's own records. `describe_policy` returns input
limits, clock-skew tolerance, identifier-shape classes, and format versions.

## 4. Outputs

### 4.1 Mint result
```
{ "canaryId": "c-0007", "datasetId": "d-91", "method": "unique-key",
  "recipientLabel": "internal-eval-b", "mintedAt": "2026-09-11T18:00:00Z",
  "markerDigest": "9f2c…" }
```
The marker value itself is returned **once**, only to the caller, only from `mint_canary`,
and never appears in evidence, stats, logs, or exports. Loss is unrecoverable by design; the
package never resurfaces or reconstructs it.

### 4.2 Trip evidence entry
```
{ "sequence": 3, "kind": "canary-trip", "canaryId": "c-0007", "datasetId": "d-91",
  "method": "unique-key", "recipientLabel": "internal-eval-b",
  "mintedAt": "2026-09-11T18:00:00Z", "trippedAt": "2026-09-12T09:41:02Z",
  "markerDigest": "9f2c…", "prevDigest": "1a0b…", "entryDigest": "7e41…" }
```
Deliberately absent: the scanned text, the surrounding context, the match offset, any
adjacent field, any characters of the marker value.

### 4.3 Scan result
`{ trips: [ … ], scannedChars: 182004, containment: { quarantined: true } }`
`quarantined` is only a local session observation flag the caller may read. It performs no
revoke, block, notification, write, or remote action.

### 4.4 Stats
Counts per dataset and per recipient label; digests only.

## 5. Invariants

1. **Synthetic-only origin.** A canary can only be embedded in package-generated data.
   There is no admission path for external data.
2. **Opaque recipients.** Recipient labels are never resolved, contacted, geolocated,
   enriched, or joined. Real-world identifier shapes are rejected at admission.
3. **Zero egress.** No network, process-spawn, DNS, or socket symbol is reachable from
   package code or its dependency tree. The package declares zero runtime dependencies.
4. **No ambient persistence.** Evidence is returned, not written. The package ships no
   default writer; durability requires an operator-supplied sink.
5. **Payload-free evidence.** Marker values and scanned content never appear in evidence,
   stats, exports, errors, or logs.
6. **Local containment only.** No revoke, block, notify, ticket, deploy, or remote action
   exists in the public surface.
7. **Chain integrity.** Evidence entries are hash-linked with SHA-256 using the same
   canonical form and chain rule as the Agent Action Evidence Ledger, and verify in a
   ledger export unchanged.
8. **Deterministic generation.** Same schema and seed ⇒ identical dataset. Non-deterministic
   seed sources are rejected.
9. **Export allowlist.** The public entry point exports exactly the reviewed surface.
   Adding an export fails the build.
10. **Detection is local comparison** over text the caller supplied — no collection, no
    callback, no telemetry.

## 6. State transitions

```
(session) --generate_synthetic_dataset--> DATASET(seeded)
DATASET --mint_canary--> ARMED(canaryId)
ARMED --scan_for_canaries(no match)--> ARMED
ARMED --scan_for_canaries(match)--> TRIPPED(evidence appended, quarantined=true)
TRIPPED --scan_for_canaries(match)--> TRIPPED (new evidence entry, idempotent flag)
DATASET --register_honeypot--> HONEYPOT_ARMED
HONEYPOT_ARMED --report_access--> HONEYPOT_ACCESSED (evidence appended)
any --reset_state--> (session cleared; prior exported evidence unaffected)
```

## 7. Failure modes

| Condition | Behaviour |
|---|---|
| Dataset handle not package-generated | rejected; type-level in the published surface, runtime-checked as defence in depth |
| Recipient label matches a real-world identifier shape | `E_IDENTIFIER_SHAPE`, naming the matched shape class, never the value |
| Honeypot registered against a non-synthetic record | `E_ORIGIN` |
| Non-deterministic seed supplied | `E_SEED` |
| Scan input not a string or async iterable of strings | `E_INPUT` |
| Record count above limit | `E_LIMIT` |
| Evidence sink write failure | `E_STORE`; chain head unchanged, entry discarded |
| Scan exceeds the published character limit | `E_LIMIT`; no remaining input is consumed |
| `trippedAt` precedes `mintedAt` beyond tolerance | evidence is retained with `clockAnomaly: true`; ordering is not silently rewritten |

Errors never echo the offending value. Identifier-shape rejection is a documented,
versioned negative corpus and is not claimed to recognize every Unicode confusable or every
real-world identifier format.

## 8. Security boundaries

**In scope:** unauthorized copying, leakage, or reappearance of the operator's own
synthetic test records inside the operator's own systems and agent outputs.

**Explicitly out of scope and refused:** third-party surveillance, deception of external
parties, covert deployment, tracking or attribution of outside actors, honeytokens placed
in data the operator does not own, and any notification or action against a third party.

**Honest limit, to appear verbatim in the shipped README:** an operator can manually copy
a synthetic canary into production data. No library can prevent that. This package refuses
to help — it will not fabricate a marker for data it did not generate, will not resolve a
recipient, and cannot reach the network — and that refusal is the whole of its guarantee.

## 9. Worked example

1. `generate_synthetic_dataset({ schema: supportTickets, recordCount: 500, seed: 42 })`
   → dataset `d-91`, 500 fabricated tickets.
2. `mint_canary({ dataset: d-91, recordRef: 118, recipientLabel: "internal-eval-b",
   method: "unique-key" })` → `c-0007`; the marker value is handed to the caller once.
3. The operator uses `d-91` as the corpus for an internal agent evaluation.
4. Two days later the operator pipes an agent transcript into `scan_for_canaries`.
5. The marker appears. A trip entry is appended: `c-0007`, `internal-eval-b`, marker
   **digest** only, hash-linked to the previous entry. `quarantined: true` is set locally.
6. The operator now knows an agent surfaced evaluation-corpus content in a transcript, and
   can prove when, with an entry that verifies inside their Evidence Ledger export — while
   the transcript itself never entered the evidence record.

## 10. Externally testable properties

| ID | Property |
|---|---|
| P1 | Passing raw external data to the mint path fails to compile (type fixture) and is rejected at runtime. |
| P2 | Every real-world-identifier-shaped recipient label in the negative corpus is rejected. |
| P3 | A static scan of the published build and its dependency tree finds zero network, DNS, socket, or process-spawn symbols; a fixture that adds one fails the build. |
| P4 | For random marker values and random scanned text, no marker character and no scanned substring appears in any evidence entry, stat, export, error, or log. |
| P5 | Same schema and seed produce byte-identical datasets across runs and platforms. |
| P6 | A non-deterministic seed source is rejected. |
| P7 | Adding a new public export fails the API-allowlist check. |
| P8 | The full test suite passes with a read-only filesystem and no network interface. |
| P9 | Canary trip entries verify inside an Agent Action Evidence Ledger export with no modification. |
| P10 | No public operation produces an outbound effect: a process-level egress monitor records zero attempts across the whole suite. |
| P11 | A honeypot cannot be registered against a record not produced by the generator. |
| P12 | Scans stop at the published character bound and do not consume later async chunks. |
| P13 | A lost marker cannot be retrieved through any public operation. |
