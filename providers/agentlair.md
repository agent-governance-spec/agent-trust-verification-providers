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
| `before_install` | REQUIRED — implemented | Author AAT verification against behavioral trust registry |
| `before_tool_call` | REQUIRED — implemented | Pre-delegation behavioral trust gate; primary signal surface |
| `gateway_start` | REQUIRED — implemented | AAT load, JWKS reachability check, cold-start prior init |
| `inbound_claim` | OPTIONAL — planned | Inbound AAT signature verification |
| `before_dispatch` | OPTIONAL — planned | Outbound `X-AGENTLAIR-*` header attachment |

`before_tool_call` is the primary signal surface. It runs the behavioral trust check against the AAT holder's accumulated score before each tool dispatch — the exact pre-delegation moment that produces the `behavioral_trust` signal.

---

## Grade scale

Scores are integers in [0, 100] per dimension (wire format). Aggregate trust is a weighted combination (default weights: restraint 42.9%, consistency 35.7%, transparency 21.4%).

| Aggregate range | Default policy |
|----------------|---------------|
| 0–40 | Warn (permissive by default, configurable to block) |
| 41–100 | Pass-through |

Cold-start prior: 0.30 (skeptical default). Converges with evidence; full override at ~100 observations.

---

## Configuration shape

Follows SPEC.md §8 schema-fields exactly (`provider`, `endpoints`, `credentials`). Provider-specific `policy.*` fields:

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

Default policy: permissive-with-warnings per SPEC.md §9 conformance requirement 3.

---

## Verifier endpoint

`POST https://agentlair.dev/v1/trust/verify`

Accepts an AAT (EdDSA JWT). Returns three-dimensional scores and aggregate in [0, 100] range. Response cached per AAT claim set; cache TTL 60s (typical-case `before_tool_call` latency under 100ms after first call).

Related crosswalk endpoints: `GET /v1/trust/{agentId}` (full trust profile) and `GET /v1/trust/{agentId}/check` (gate check). The `/v1/trust/verify` endpoint is a convenience form for AAT-based lookup without requiring a resolved `agentId`.

Gateway RPC methods:

- `agentlair.getTrustScore(agentId)` — returns scores by dimension
- `agentlair.verifyAAT(token)` — verifies EdDSA signature against JWKS
- `agentlair.signMessage(payload)` — signs outbound payload with session AAT key

---

## Production status

Live as of 2026-04-28. Behavioral event ingestion active. Non-test agents have non-null scores across all three dimensions.

Vocabulary crosswalk at `aeoess/agent-governance-vocabulary` (PR #46, merged 2026-04-25): `behavioral_trust` exact, `trust_verification` partial, `governance_attestation` partial. Ten canonical terms with explicit `no_mapping` entries and technical rationale.
