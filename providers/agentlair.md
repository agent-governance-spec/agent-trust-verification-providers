# AgentLair — Trust Verification Provider Summary

**Signal class:** `behavioral_trust` (canonical mapping: `peer_review`, exact)  
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

Scores initialize from a Bayesian cold-start prior (0.50 per dimension). Prior narrows as behavioral events accumulate. Typically converges after 20–30 observed tool calls. Non-test agents running in production have non-null scores; the scoring pipeline is live.

The `peer_review` mapping in the vocabulary crosswalk is exact. AgentLair's pre-delegation behavioral trust check maps directly to this signal class. Production evidence (live endpoints, behavioral event ingestion, non-null scores on non-test agents) is what moved the match type from `partial` to `exact` — per the match-type rationale in the merged crosswalk.

---

## Hook coverage

| Hook | Status | Notes |
|------|--------|-------|
| `before_install` | REQUIRED — implemented | Author AAT verification against behavioral trust registry |
| `before_tool_call` | REQUIRED — implemented | Pre-delegation behavioral trust gate; primary signal surface |
| `gateway_start` | REQUIRED — implemented | AAT load, JWKS reachability check, cold-start prior init |
| `inbound_claim` | OPTIONAL — planned | Inbound AAT signature verification |
| `before_dispatch` | OPTIONAL — planned | Outbound `X-AGENTLAIR-*` header attachment |

`before_tool_call` is the primary signal surface. It runs the behavioral trust check against the AAT holder's accumulated score before each tool dispatch — the exact pre-delegation moment that produces the `peer_review` signal.

---

## Grade scale

Scores are continuous floats in [0.0, 1.0] per dimension. Aggregate trust is a weighted combination (default weights: consistency 0.4, restraint 0.4, transparency 0.2).

| Aggregate range | Default policy |
|----------------|---------------|
| 0.00–0.40 | Warn (permissive by default, configurable to block) |
| 0.41–1.00 | Pass-through |

Cold-start prior: 0.50. Converges with evidence; 20–30 events typical.

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
      "warnBelow": 0.41,
      "blockBelow": null
    },
    "dimensions": ["consistency", "restraint", "transparency"],
    "dimensionWeights": {
      "consistency": 0.4,
      "restraint": 0.4,
      "transparency": 0.2
    },
    "coldStart": {
      "prior": 0.5,
      "convergenceEvents": 25
    },
    "enforceScope": true
  }
}
```

Default policy: permissive-with-warnings per SPEC.md §9 conformance requirement 3.

---

## Verifier endpoint

`POST https://agentlair.dev/v1/trust/verify`

Accepts an AAT (EdDSA JWT). Returns three-dimensional scores and aggregate. Response cached per AAT claim set; cache TTL 60s (typical-case `before_tool_call` latency under 100ms after first call).

Gateway RPC methods:

- `agentlair.getTrustScore(agentId)` — returns scores by dimension
- `agentlair.verifyAAT(token)` — verifies EdDSA signature against JWKS
- `agentlair.signMessage(payload)` — signs outbound payload with session AAT key

---

## Production status

Live as of 2026-04-28. Behavioral event ingestion active. Non-test agents have non-null scores across all three dimensions.

Vocabulary crosswalk at `aeoess/agent-governance-vocabulary` (PR #46, merged 2026-04-25): `peer_review` exact, `behavioral_trust` exact, `trust_verification` partial, `governance_attestation` partial. Eight canonical terms with explicit `no_mapping` entries and technical rationale.
