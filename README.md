# Canary Evidence Chain

> **Specification only — no implementation exists in this repository.**

Synthetic-only canary evidence for detecting reappearance of evaluation data.

This is an independent SHPBL research candidate intended for possible future submission to the Tenable CyberAgents Exchange. It has **not** been submitted to, approved by, or endorsed by Tenable.

## Read the contract

- [SPEC-canary-evidence-chain.md](./SPEC-canary-evidence-chain.md) — version 1.0 public behavior specification
- [PROVENANCE.md](./PROVENANCE.md) — SHPBL/CMPSBL heritage and the public IP boundary
- [STATUS.md](./STATUS.md) — current review and submission state
- [LICENSE](./LICENSE) — MIT grant limited to this repository's specification documents

## Proposed product form

Skill + MCP server. The specification defines inputs, outputs, invariants, state transitions, failure modes, security boundaries, worked examples, and externally testable properties. It deliberately does not prescribe or contain implementation code.

## Why a security practitioner might use it

The contract is designed to become an independently testable defensive tool rather than a policy-only document. Reviewers can challenge the public behavior now, before implementation decisions narrow the design or create accidental IP exposure.

## Current stage

```text
SPECIFICATION RELEASED → IMPLEMENTATION NOT AUTHORIZED → NOT SUBMITTED → NOT APPROVED
```

A separate human approval is required before fresh implementation. A later exact-file IP-surface review and license decision will apply to those new implementation bytes.

## Review

Open an issue with a concrete ambiguity, counterexample, threat-model gap, or externally testable property. Do not report this repository as a working capability or a Tenable listing.

## Origin

Composed by [SHPBL](https://shpbl.com/tenable-submissions) from capability intent discovered across SHPBL and its same-author sister project, CMPSBL. See [PROVENANCE.md](./PROVENANCE.md) for the precise claim.
