# Agent Trust Verification Provider Pattern

**v0.1 — OpenClaw plugin profile, community draft**
**Status:** Proposed (community draft, not endorsed by any plugin platform)
**Date:** 2026-04-27
**Editors:** Tymofii Pidlisnyi (APS by AEOESS), Lars Kroehl (MolTrust / CryptoKRI GmbH)
**Repository:** github.com/agent-governance-spec/agent-trust-verification-providers
**License:** CC-BY-4.0 (specification text); reference implementations licensed independently under their own terms (Apache 2.0 for the APS reference, MolTrust reference license per its package)

---

## 1. Status and authorship

This document is a community-authored integration pattern for plugins that implement agent trust verification. **It is not authored by, sponsored by, or endorsed by any plugin platform.** Platform maintainers (in v0.1, the OpenClaw project) were notified of this draft for transparency before publication; nothing in this document should be read as a statement on behalf of those projects.

The pattern is published as a CC-BY-4.0 specification under shared editorship. The current editor list reflects who has contributed substantive design work to v0.1; the editor slot is explicitly open to other providers building reference implementations against this pattern. The pattern is not owned by any single project or organization. v0.1 is a profile for OpenClaw plugin SDK; future versions may add profiles for other plugin platforms with comparable hook surfaces.

The pattern was extracted from the design conversations in two closed OpenClaw issues that asked for native trust verification (#49971 RFC by MolTrust, #63430 multi-agent trust boundaries). Both were closed by the OpenClaw maintainer with the same direction: the plugin layer is the right home, and the existing hook surface is sufficient. This document organizes the plugin layer for that purpose. It does not propose any change to OpenClaw core.

**Editorial process for substantive disagreements.** Resolution of substantive disagreements between editors follows a two-step path. Editors first attempt convergent revision through normal PR review. If convergence is not reached within two review rounds, the disputed sentence or section is labeled with both editor positions and the resolution is deferred to public WG input over a defined window (typically two weeks). No external arbiter is invoked. If consensus still does not form, the disputed item is marked unresolved in v0.x and revisited at the next minor version boundary.

**Implementation names remain property of respective projects.** Provider names referenced in this specification (APS, MolTrust, AgentID, AgentNexus, AgentLair, and others) remain the property of their respective projects and contributors. Inclusion in this specification does not transfer trademark, naming, or other proprietary rights. Removal of a provider reference may be requested by that provider's named maintainer.

**Editor entry path.** Additional editors may join during a v0.x line by mutual consent of the existing editors, accompanied by substantive design contribution to the specification's open structural items. New editors are added in PR-merge messages and the editor line in this section. Major-version (v1.0+) editor changes follow the editorial process clause above.

## 2. Scope and non-goals

**In scope.** What hooks an Agent Trust Verification Provider (TVP) plugin should register, what each handler does, how multiple TVPs compose when more than one is installed, the configuration schema providers should accept, conformance criteria, and pointers to reference implementations.

**Not in scope.** OpenClaw core itself, the OpenClaw plugin SDK design, what counts as a valid trust signal (a separate cross-vendor effort lives at github.com/aeoess/agent-governance-vocabulary), any specific provider's internal verification methodology, and any commercial framing of trust as a service. The pattern is provider-agnostic and substrate-agnostic.

## 3. Problem statement

Multi-agent OpenClaw deployments (multiple instances, third-party skills from ClawHub, inter-agent communication, autonomous payments) have no transport-level trust verification. Skills are gated by metadata signature scanning at install time, but agent identity is not verified, delegation scope is not enforced at tool-call boundary, and inter-agent messages are not signature-checked.

The OpenClaw plugin SDK already exposes the hooks needed to add this layer: `before_install` for skill author verification, `before_tool_call` for delegation scope enforcement, `inbound_claim` and `before_dispatch` for inter-agent message signatures, `gateway_start` for self-attestation, plus `registerGatewayMethod` for cross-plugin RPC. What is missing is a documented pattern for how trust verification providers should register on those hooks consistently, so multiple providers compose without conflicting and so user policy stays portable across providers.

## 4. Architecture

A TVP plugin SHOULD register on the public OpenClaw plugin hooks named in section 5. Handlers SHOULD return blocking, warning, or pass-through results based on configurable policy. The plugin SHOULD NOT modify other plugins' state, write to user files outside the OpenClaw config directory, or perform network calls outside the verification path. All blocking decisions MUST be deterministic given the same input event and config.

This pattern is intentionally trust-model agnostic. The canonical governance vocabulary at github.com/aeoess/agent-governance-vocabulary distinguishes three first-class signal classes that TVPs may implement: `governance_attestation` for authority-based providers (delegation tokens, scoped permits, revocation chains), `behavioral_trust` for behavior-based providers (scored trust derived from observed behavior over time), and `trust_verification` for identity-based providers (cryptographic identity verification, endorsement chain validation). Different TVPs implement different signal classes; section 7 specifies how their outputs compose at the gateway. The pattern does not canonicalize one trust model over another. It specifies the integration surface that lets multiple trust models coexist on the same OpenClaw deployment.

```
                  +----------------------------------------+
                  |      OpenClaw plugin SDK (core)        |
                  |  registerHook | registerGatewayMethod  |
                  +----------------------------------------+
                                    |
                +-------------------+--------------------+
                |                                        |
        +---------------+                       +---------------+
        | TVP plugin A  |       (...)           | TVP plugin N  |
        | (e.g. APS,    |                       | (e.g. MolTrust|
        |  authority-   |                       |  behavior-    |
        |  based)       |                       |  based)       |
        +---------------+                       +---------------+
```

Each TVP runs independently. Composition is by ordered evaluation (section 7) and gateway-RPC method namespacing (section 6).

## 5. Required and optional hooks

The hook names below are public OpenClaw plugin SDK names as of commit 45146913007d (April 2026). The signatures referenced are from `src/plugins/hook-types.ts` in that commit.

### 5.1 `before_install` — REQUIRED

Triggered when a skill, plugin, or bundle is being installed from any source (npm, ClawHub, local archive). The TVP receives the install target metadata including author identifier where available. The provider verifies the author against its trust registry and returns either a pass-through, a warning finding, or a hard block.

Handler MUST handle missing-author gracefully: skills installed from local archives or unsigned sources may not carry an identifier the provider knows. Default behavior on missing-author SHOULD be permissive with a warning surfaced via the `findings` field.

### 5.2 `before_tool_call` — REQUIRED

Triggered before every agent tool invocation. The TVP receives the tool name and parameters. The provider checks the call against the agent's active delegation or authorization context. For tools outside the active scope, the handler returns `{block: true}` with a reason. For tools inside the scope but flagged as high-risk in config, the handler MAY return `{requireApproval}` to force user confirmation instead of blocking.

The handler MUST complete in under 100ms in the typical case (cached delegation lookup) and under 500ms in the cold case (gateway round-trip). Slower handlers degrade agent responsiveness and SHOULD be optimized via local caching.

### 5.3 `inbound_claim` — OPTIONAL

Triggered when an inbound inter-agent message or claim arrives. The TVP receives sender identity, channel, and payload metadata. The provider verifies any signed envelope on the inbound claim against the sender's published key material. Unsigned or signature-failing messages MAY be blocked, warned, or passed through based on config.

This hook SHOULD be implemented by providers that ship signed inter-agent messaging primitives. Providers that only verify identity at install time MAY omit it.

### 5.4 `before_dispatch` — OPTIONAL

Triggered before an outbound message is dispatched to another agent. The TVP receives the outbound message envelope and MAY add provider-specific signature headers.

Providers SHOULD use namespaced header prefixes (`X-{PROVIDER}-*`) to avoid collision when multiple TVPs are installed.

### 5.5 `gateway_start` — REQUIRED

Triggered when the OpenClaw gateway process starts. The TVP performs its own startup checks: load local credentials, verify they are not revoked, confirm reachability of any required external verification endpoints, and log readiness. Failures here SHOULD NOT block gateway startup but SHOULD be surfaced through the standard plugin diagnostic channel.

## 6. Gateway RPC methods

A TVP SHOULD expose at least the following capabilities via `api.registerGatewayMethod()`, namespaced by provider. Method names within each provider's namespace are at the provider's discretion (schema-shape); the capabilities listed are required (schema-fields):

- A method returning the provider's trust output for a given agent (e.g., `aps.checkGrade(agentId)` for authority-based providers, `moltrust.getTrustScore(agentId)` for behavior-based providers)
- A method verifying a provider-signed attestation under the provider's own scheme (e.g., `{provider}.verifyDelegation(token)` for authority-based providers, `{provider}.verifyAttestation(credential)` for identity-based providers, `{provider}.verifyEndorsement(chain)` for behavior-based providers)
- A method signing an outbound payload with the local TVP key (e.g., `{provider}.signMessage(payload)`)

Cross-plugin callers SHOULD be able to discover available providers by inspecting registered method namespaces. A future revision of this specification MAY define a discovery method.

## 7. Composition: multiple providers

When more than one TVP is installed, the OpenClaw plugin SDK runs all `before_install` and `before_tool_call` handlers in registration order. Each handler returns independently. The plugin SDK aggregates results: any handler returning `block: true` blocks the operation; any handler returning `findings` surfaces them to the user.

TVPs MUST NOT assume they are the only provider running. A TVP MUST NOT mutate global state (environment variables, file system locations outside its plugin directory) in a way that other TVPs would observe.

User policy SHOULD specify per-provider strictness independently. A user running both APS and MolTrust SHOULD be able to configure APS as advisory-only and MolTrust as blocking, or any other combination. The reference config schema in section 8 supports this.

For the `inbound_claim` and `before_dispatch` hooks, handlers run in registration order but only the first handler returning a signature mutation is honored for that field. Subsequent handlers MAY add their own signature in a different namespace.

## 8. Configuration schema

A TVP SHOULD accept configuration from `~/.openclaw/{provider}.config.json` or equivalent provider-namespaced location. The schema below is RECOMMENDED. The schema distinguishes two classes of fields. Schema-fields parts (envelope, endpoints, credentials) require the exact field names given because cross-provider tooling reads them as fixed surfaces. Schema-shape parts (the `policy.*` tree, including grade scales and provider-specific policy fields) require only the structural shape; provider-specific field names within `policy.*` are permitted as long as the strictness-level shape is preserved.

```json
{
  "provider": "{provider-name}",
  "endpoints": {
    "verifier": "https://...",
    "jwks": "https://.../.well-known/jwks.json"
  },
  "credentials": {
    "passportPath": "~/.openclaw/{provider}-credentials.json"
  },
  "policy": {
    "skillAuthor": {
      "minGrade": 0,
      "warnBelow": 1,
      "blockBelow": null
    },
    "toolCalls": {
      "enforceScope": true,
      "highRiskTools": ["bash", "exec", "fetch"],
      "highRiskBehavior": "approval"
    },
    "inboundMessages": {
      "requireSignature": false,
      "warnUnsigned": true
    }
  }
}
```

**Schema-fields (interop-critical, exact field names required):** `provider`, `endpoints.verifier`, `endpoints.jwks`, `credentials.passportPath`. These fields are required by name because cross-provider tooling reads them as fixed surfaces.

**Schema-shape (model-specific, structure-only):** all `policy.*` fields including grade scales (`minGrade`, `warnBelow`, `blockBelow`), behavior mode fields (`enforceScope`, `highRiskBehavior`, `highRiskTools`), and provider-specific keys. Providers MAY use different field names within `policy.*` as long as the strictness-level shape is preserved (a permissive-warnings-blocking gradient or equivalent).

The field-by-field walkthrough refining this split is tracked in a separate issue against this repository.

Default policy SHOULD be permissive-with-warnings so installation does not break existing OpenClaw setups. Strict mode SHOULD be a configuration change, not a code change.

## 9. Conformance

A plugin claiming conformance to this pattern version 0.1 MUST:

1. Register handlers on `before_install`, `before_tool_call`, and `gateway_start`
2. Accept the configuration schema in section 8 with the following split:
   - **2a.** Accept the envelope schema (`provider`, `endpoints`, `credentials`) with the field names given. (Schema-fields, interop-critical.)
   - **2b.** Accept some policy schema with strictness levels covering install-time author checks, runtime tool-call enforcement, and inbound-message verification. The example in section 8 is illustrative; provider-specific field names within the `policy.*` tree are permitted as long as the strictness-level structure is preserved. (Schema-shape, model-specific.)
3. Default to permissive-with-warnings on first install
4. Handle missing-author and missing-credential conditions without crashing the plugin runtime
5. Complete `before_tool_call` handler in under 500ms in the cold case
6. Use namespaced gateway RPC methods (`{provider}.{method}`)
7. Use namespaced outbound headers (`X-{PROVIDER}-*`) if implementing `before_dispatch`
8. Not mutate state outside the plugin's own directory
9. Publish a verifier endpoint or local key material that downstream verifiers can use to independently check the provider's signed outputs
10. Document its trust signal semantics and grade scale in a publicly accessible specification or crosswalk

A plugin claiming conformance SHOULD additionally:

- Implement `inbound_claim` if it ships signed inter-agent messaging
- Implement `before_dispatch` if it adds outbound signature headers
- Cache verification results for at least 60 seconds to keep the typical-case `before_tool_call` latency under 100ms
- Surface user-visible warnings through the OpenClaw `findings` channel rather than custom UI

A conformance test suite is out of scope for v0.1 but is anticipated for a later revision.

## 10. Reference implementations

Reference implementations are published under `providers/` in this specification's repository. The reference suite is open: any provider that implements the pattern is invited to contribute a one-page summary at `providers/{name}.md` documenting the provider's grade scale, hook coverage, configuration shape, and verifier endpoint.

Two reference implementations launch concurrent with v0.1 publication:

**MolTrust v2.0** — combined behavior-based and identity-based trust verification. Maintainer: Lars Kroehl (CryptoKRI GmbH). npm package: `@moltrust/openclaw` (v2.0 upcoming; current latest tag v1.0.1). Trust output: behavior-derived score against the canonical `behavioral_trust` signal class, combined with identity verification against the `trust_verification` signal class. Signing: W3C Verifiable Credentials with DID-based identity, anchored to Base L2.

**Agent Passport System (APS) OpenClaw plugin** — authority-based trust verification. Maintainer: Tymofii Pidlisnyi (APS by AEOESS). npm package: `agent-passport-system-openclaw-plugin` (`github.com/aeoess/openclaw-plugin-aps`). Trust output: passport-grade-scoped delegation against the canonical `governance_attestation` signal class. Trust grades 0-3 (self-asserted, infrastructure-attested, provider-attested, hardware-attested) per the Agent Social Contract paper (Zenodo DOI 10.5281/zenodo.18749779). Signing: Ed25519 envelope signatures, JWS via the public gateway at gateway.aeoess.com.

The two implementations are concurrent first-references, not ordered. Neither is canonical. The pattern itself is the canonical artifact, and the providers/ directory is a catalogue, not a ranking.

Other providers actively building agent trust verification (AgentID, AgentNexus, AgentLair, others) are invited to contribute reference summaries on the same footing as the entries above.

## 11. Open questions for v0.2

- Should there be a discovery method (`__tvp_discover()`) that returns the list of installed TVPs and their reported grade scales?
- Should the `findings` field shape be standardized across providers so the OpenClaw UI can render them uniformly?
- Should there be a conformance test harness that providers run before publishing claims of conformance?
- How does this pattern compose with the `bundle` plugin format (narrower trust boundary) versus native plugins (in-process, more privileged)?
- Should the spec define a standard cache shape so that multiple providers do not each maintain independent caches of overlapping data (such as agent passport grades)?
- Should v0.2 add platform profiles for plugin systems beyond OpenClaw with comparable hook surfaces?

These are deliberately not answered in v0.1. Implementation experience across at least two independent providers should inform v0.2.

## 12. Acknowledgments

This pattern would not exist in this shape without two prior contributions. The OpenClaw plugin surface, designed by Peter Steinberger and the OpenClaw maintainer team, exposes exactly the hooks a trust verification layer needs without requiring core changes; this pattern composes against that surface and adds nothing to it. The original RFC framing in OpenClaw issue #49971 by Lars Kroehl (MolTrust / CryptoKRI GmbH) identified the problem space, the hook coverage, and the multi-issuer composition need that v0.1 documents.

The contributors to `aeoess/agent-governance-vocabulary` ground the cross-provider semantics referenced in section 4 and section 7. The vocabulary's `trust_verification`, `governance_attestation`, and `behavioral_trust` canonical signal definitions are the substrate this pattern composes against.

This specification is published under CC-BY-4.0. Each reference implementation is published separately under its own license; the APS reference implementation is Apache 2.0; the MolTrust reference implementation under its own license per its package.

## 13. Revision history

- **v0.1 (2026-04-27):** initial draft as published to the neutral-org home. Co-edited by Tymofii Pidlisnyi (APS by AEOESS) and Lars Kroehl (MolTrust / CryptoKRI GmbH). Reframing from the working draft (2026-04-26) applied: vocab-class anchoring in section 4 (governance_attestation, behavioral_trust, trust_verification as parallel first-class signal classes), MUST #2 split into 2a schema-fields and 2b schema-shape across sections 8 and 9, MolTrust v2.0 and APS plugin as concurrent first-references in section 10, three additional clauses in section 1 (editorial process, implementation-name, editor-entry), explicit license headers (spec CC-BY-4.0, reference implementations independent). Hook signatures verified against OpenClaw `src/plugins/hook-types.ts` at commit 45146913007d. OpenClaw maintainer notified prior to publication.
