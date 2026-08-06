# Kerq  
**Category:** `behavioral_trust` (tool‑level reliability)  
**Type:** Tool Trust Verification Provider  
**Website:** [https://kerq.dev](https://kerq.dev)  
**Maintainer:** `@greenoriginals`

---

## Overview
Adds `providers/kerq.md` per the invitation in **SPEC.md §10**  
- Kerq is a **behavioral trust provider** focused on **tool endpoint reliability at execution time**  
- Existing providers evaluate whether an *agent* is trusted to act  
- **Kerq evaluates whether the *tool* an agent is about to call is reliable enough to execute**  
- Enables agents to enforce **minimum reliability thresholds** before connecting to external APIs, MCP servers, or services

---

## Trust Model
Kerq implements the **`behavioral_trust`** signal class, applied to **tools**, not agents.

**Trust scores are derived from:**
- automated probes (continuous endpoint monitoring)  
- community telemetry (real‑world tool call outcomes)

These signals are aggregated into a **weighted composite score** across **7 reliability metrics**.

**Kerq does *not* evaluate:**
- agent identity  
- delegation or permissions  
- inter‑agent communication  

Kerq strictly evaluates **tool execution reliability**.

---

## Trust Output
Scores are integers in **0–100**.  
Per‑metric scores use the same range.

```json
{
  "success": true,
  "tool_id": 42,
  "slug": "github-mcp-server",
  "trust_score": 91,
  "tier": "certified",
  "score_breakdown": {
    "uptime": 98,
    "response_time": 95,
    "error_rate": 100,
    "consistency": 88,
    "throughput": 85,
    "recovery_behavior": 90,
    "performance_trend": 80
  },
  "calculated_at": "2026-03-30T20:00:00.000Z"
}
```

---

## Hook Coverage

### `before_tool_call` (**REQUIRED**)
Kerq enforces reliability at execution time:

1. Agent initiates tool call  
2. Kerq queries `/api/tools/:id/score`  
3. Score is compared to policy threshold  
4. Execution is allowed or blocked  

### `before_install` (**REQUIRED**)
Kerq provides **advisory trust visibility** at install time:

- inspects tools referenced by the installing skill/plugin  
- retrieves trust scores from Kerq index  
- returns findings:
  - **warning** if score below threshold  
  - **advisory** if tool is not yet scored  

Kerq **does not block installation**.

### `gateway_start` (**REQUIRED**)
- validates API availability  
- verifies credentials  
- confirms readiness  

### `inbound_claim` / `before_dispatch`
Not implemented — Kerq does not handle identity or messaging trust.

---

## Configuration Shape

```json
{
  "provider": "kerq",
  "endpoints": {
    "verifier": "https://kerq.dev/api",
    "jwks": null
  },
  "credentials": {
    "apiKey": "YOUR_API_KEY"
  },
  "policy": {
    "toolCalls": {
      "minScore": 80,
      "blockBelow": 65
    }
  }
}
```

- `"jwks": null`  
- Policy fields follow schema‑shape flexibility

---

## Gateway RPC Methods

### `kerq.getToolScore(toolId)`
Returns trust score via `/api/tools/:id/score`.

### `kerq.verifyAttestation(credential)`
Not implemented — Kerq does not issue attestations.

### `kerq.signMessage(payload)`
Not implemented — Kerq does not sign scores.

---

## Verifier Endpoint

**Primary endpoint:**

```
GET /api/tools/:id/score
```

- returns trust score + breakdown  
- safe for per‑step execution  
- low latency (~27ms average)

---

## Telemetry Loop

Optional telemetry reporting:

```
POST https://kerq.dev/api/v1/report
```

Agents may report:
- tool_id  
- status_code  
- latency_ms  
- timestamp  

Telemetry is:
- fire‑and‑forget  
- non‑blocking  
- improves scoring accuracy over time  

---

## Behavior
- deterministic scoring  
- read‑only enforcement  
- sub‑30ms average latency  

---

## Production Status
- **Score Index:** continuously monitored tool catalog  
- **Scoring API:** live at [https://kerq.dev/api](https://kerq.dev/api)  
- **Telemetry ingestion:** live at `/api/v1/report`  
- **Performance:** ~27ms avg, <35ms p95  
- **SDKs:** Python (PyPI), LangChain, CrewAI  

---

## Positioning
Kerq fills a gap:

- Existing providers verify whether an **agent** should act  
- Kerq verifies whether the **tool** the agent is about to call will reliably execute  

Kerq provides **tool‑level trust scoring** for execution‑time decision points.

---

## Compatibility
Kerq is complementary to:

- governance_attestation providers  
- trust_verification providers  
- agent‑focused behavioral_trust providers  

Kerq extends behavioral trust to **external tool reliability**.
