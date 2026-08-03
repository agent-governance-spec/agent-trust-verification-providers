# AgentLair — Trust Verification Provider Summary

**Signal class:** `behavioral_trust` (canonical mapping: `behavioral_trust`, exact)  
**Trust model:** behavior-based  
**Maintainer:** @piiiico  
**Verifier endpoint:** `https://agentlair.dev/v1/trust/verify`  
**JWKS:** `https://agentlair.dev/.well-known/jwks.json`  
**Vocabulary crosswalk:** [aeoess/agent-governance-vocabulary crosswalk/agentlair.yaml](https://github.com/aeoess/agent-governance-vocabulary/blob/main/crosswalk/agentlair.yaml) (merged 2026-04-25, PR #46)

---

## Trust model

AgentLair issues an Agent Authentication Token (AAT) — EdDSA-signed JWT — at session start. Session-scoped, not portable across sessions. Verifiable against a public JWKS endpoint.

Trust scoring is three-dimensional:

- **Consistency** — how predictably the agent behaves across similar task contexts
- **Restraint** — whether the agent stays within its stated scope (tool use, resource consumption, delegation depth)
- **Transparency** — whether observable behavior matches declared intent

Scores initialize from a Bayesian cold-start prior (0.30, skeptical default). Prior narrows as behavioral events accumulate. Typically converges after ~100 observed behavioral events. Non-test agents running in production have non-null scores; the scoring pipeline is live.

The `behavioral_trust` mapping in the vocabulary crosswalk is exact. AgentLair's pre-delegation behavioral trust check maps directly to this signal class. Production evidence (live endpoints, behavioral event ingestion, non-null scores on non-test agents) is what moved the match type from `partial` to `exact` — per the match-type rationale in the merged crosswalk.

---

## Hook coverage

| Hook | Status | Notes |
|------|--------|-------|
| `before_install` | REQUIRED — planned | Author AAT verification against behavioral trust registry |
| `before_tool_call` | REQUIRED — implemented | Pre-delegation behavioral trust gate; primary signal surface |
| `gateway_start` | REQUIRED — planned | AAT load, JWKS reachability check, cold-start prior init |
| `inbound_claim` | OPTIONAL — planned | Inbound AAT signature verification |
| `before_dispatch` | OPTIONAL — planned | Outbound `X-AGENTLAIR-*` header attachment |

1 of the 3 hooks required by SPEC.md §9 is implemented. The shipping plugin (`@agentlair/defenseclaw@0.3.0`) registers `before_tool_call` and `after_tool_call`, and nothing else. `before_install` and `gateway_start` are not implemented; the rows above said `implemented` and were wrong. Full status: Conformance status, below.

`before_tool_call` is the primary signal surface. It runs the behavioral trust check against the AAT holder's accumulated score before each tool dispatch — the exact pre-delegation moment that produces the `behavioral_trust` signal. It verifies the caller's AAT against the JWKS, reads the trust score, and blocks or warns against a configurable threshold.

---

## Grade scale

Scores are integers in [0, 100] per dimension (wire format). Aggregate trust is a weighted combination (default weights: restraint 42.9%, consistency 35.7%, transparency 21.4%).

| Score | Policy |
|----------------|---------------|
| Below the configured threshold (default: < 40) | Block (shipping default is `mode: enforce`, `failOpen: false`) |
| At or above the configured threshold (default: ≥ 40) | Pass-through |

Note: SPEC.md §9.3 requires defaulting to permissive-with-warnings on first install. `@agentlair/defenseclaw@0.3.0` ships `mode: "enforce"` and `failOpen: false`, so it blocks by default and does not meet §9.3 today. Setting `mode: "observe"` gives the spec-intended behaviour. See Conformance status below.

Cold-start prior: 0.30 (skeptical default). Converges with evidence; full override at ~100 observations.

---

## Configuration shape

**Target shape (SPEC.md §8), not yet accepted by the shipping plugin.** `@agentlair/defenseclaw@0.3.0` takes a flat, provider-specific schema — `agentlairUrl`, `agentlairApiKey`, `agentId`, `jwksUrl`, `trustThreshold`, `mode`, `failOpen`, `verifyAat`, `reportTelemetry` — with `additionalProperties: false`. The envelope below would be **rejected** by that schema today. It documents where the plugin is going, not what it accepts. Anyone integrating right now should use the flat fields.

The §8 envelope AgentLair intends to accept, with provider-specific `policy.*` fields:

```json
{
  "provider": "agentlair",
  "endpoints": {
    "verifier": "https://agentlair.dev/v1/trust/verify",
    "jwks": "https://agentlair.dev/.well-known/jwks.json"
  },
  "credentials": {
    "passportPath": "~/.openclaw/agentlair-credentials.json"
  },
  "policy": {
    "trustThresholds": {
      "warnBelow": 41,
      "blockBelow": null
    },
    "dimensions": ["consistency", "restraint", "transparency"],
    "dimensionWeights": {
      "consistency": 0.357,
      "restraint": 0.429,
      "transparency": 0.214
    },
    "coldStart": {
      "prior": 0.30,
      "convergenceEvents": 100
    },
    "enforceScope": true
  }
}
```

Default policy in the shipping plugin is enforce/fail-closed, not the permissive-with-warnings that §9.3 requires. See Conformance status.

---

## Verifier endpoint

`POST https://agentlair.dev/v1/trust/verify`

Accepts an AAT (EdDSA JWT). Returns three-dimensional scores and aggregate in [0, 100] range.

Related crosswalk endpoints: `GET /v1/trust/{agentId}` (full trust profile) and `GET /v1/trust/{agentId}/check` (gate check). The `/v1/trust/verify` endpoint is a convenience form for AAT-based lookup without requiring a resolved `agentId`.

Gateway RPC methods (SPEC.md §6) — **planned, not yet registered.** `@agentlair/defenseclaw@0.3.0` makes no `api.registerGatewayMethod()` calls. The capabilities exist over HTTP today and the intended method names are:

- `agentlair.getTrustScore(agentId)` — returns scores by dimension (live as `GET /v1/trust/{agentId}`)
- `agentlair.verifyAAT(token)` — verifies EdDSA signature against JWKS (live as `POST /v1/trust/verify`)
- `agentlair.signMessage(payload)` — signs outbound payload with session AAT key (no HTTP equivalent yet)

---

## Production status

Live as of 2026-04-28. Behavioral event ingestion active. Non-test agents have non-null scores across all three dimensions.

Vocabulary crosswalk at `aeoess/agent-governance-vocabulary` (PR #46, merged 2026-04-25): `behavioral_trust` exact, `trust_verification` partial, `governance_attestation` partial. Ten canonical terms with explicit `no_mapping` entries and technical rationale.

---

## Conformance status

**AgentLair does not currently meet SPEC.md §9 v0.1.**

Implementation status for `@agentlair/defenseclaw@0.3.0`, verified against the published artifact on 2026-08-02.

| §9 requirement | Status |
|---|---|
| 1. Register `before_install`, `before_tool_call`, `gateway_start` | **Not met** — only `before_tool_call` (plus a non-spec `after_tool_call`) |
| 2a. Accept envelope schema (`provider`, `endpoints`, `credentials`) | **Not met** — flat provider-specific schema, `additionalProperties: false` |
| 2b. Policy schema with strictness levels | **Not met** — runtime tool-call enforcement exists; install-time and inbound-message strictness are not implemented |
| 3. Default to permissive-with-warnings | **Not met** — ships `mode: enforce`, `failOpen: false` |
| 4. Handle missing-author / missing-credential without crashing | **Not met** — missing credentials handled without crashing (note); the author-side check requires `before_install`, which is not registered, so the complete requirement cannot be reached |
| 5. `before_tool_call` under 500ms cold | **Untested** — nothing has been measured. `checkTrust()` runs on every call and its abort budget is 3000ms (`TRUST_TIMEOUT_MS = 3e3`), so the plugin does not enforce the 500ms ceiling — which is not the same as the cold path measuring above it |
| 6. Namespaced gateway RPC methods | **Not met** — no `registerGatewayMethod()` calls |
| 7. Namespaced outbound headers | n/a — `before_dispatch` not implemented |
| 8. No state mutation outside plugin directory | Met — no filesystem writes in the published bundle |
| 9. Publish verifier endpoint / key material | Met — JWKS and verifier live |
| 10. Document signal semantics and grade scale | Met — this file and the vocabulary crosswalk |

The behavioral trust model, scoring pipeline, JWKS and verifier endpoint described above are live and independently checkable. What is not yet true is the *plugin-side* spec conformance: two required hooks, the config envelope, the default posture, and gateway RPC registration. Those are implementation work, not claims, and this file will be updated when they land — not before.
