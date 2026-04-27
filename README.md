# Agent Trust Verification Provider Pattern

A community-authored integration pattern for plugins that implement agent trust verification on the OpenClaw plugin surface. Trust-model agnostic. Anchored on the canonical governance vocabulary at [aeoess/agent-governance-vocabulary](https://github.com/aeoess/agent-governance-vocabulary).

**Specification:** [`SPEC.md`](./SPEC.md) (v0.1)
**License:** CC-BY-4.0 (specification text); reference implementations licensed independently
**Editors:** Tymofii Pidlisnyi (APS by AEOESS), Lars Kroehl (MolTrust / CryptoKRI GmbH)

## What this is

A documented pattern for how Agent Trust Verification Provider (TVP) plugins should register on the OpenClaw plugin SDK hooks consistently, so multiple providers compose without conflicting and so user policy stays portable across providers.

The pattern was extracted from the design conversations in OpenClaw issues [#49971](https://github.com/openclaw/openclaw/issues/49971) (RFC by MolTrust) and [#63430](https://github.com/openclaw/openclaw/issues/63430) (multi-agent trust boundaries). Both were closed by the OpenClaw maintainer with the same direction: the plugin layer is the right home, and the existing hook surface is sufficient. This document organizes the plugin layer for that purpose. It does not propose any change to OpenClaw core.

## Status

v0.1 community draft. Not endorsed by any plugin platform. Co-edited by two providers building reference implementations (APS and MolTrust). Editor entry path documented in [`SPEC.md`](./SPEC.md) §1.

Open structural items being walked through during the v0.1 phase are tracked as issues in this repository.

## Reference implementations

Two reference implementations launch concurrent with v0.1 publication:

- **MolTrust v2.0** (behavior-based, `@moltrust/openclaw`)
- **Agent Passport System OpenClaw plugin** (authority-based, `agent-passport-system-openclaw-plugin`)

Other providers building agent trust verification are invited to contribute reference summaries under `providers/{name}.md` on the same footing.

## Contributing

The pattern is co-edited under CC-BY-4.0 with two-editor consensus. Contributions welcome:

- **Editorial fixes** (typos, clarity): open a PR against `main`
- **Substantive proposals** (new sections, semantic changes): open an issue first
- **Reference implementation entries**: PR adding `providers/{name}.md` matching the shape in [`SPEC.md`](./SPEC.md) §10

Editor entry path for new editors during a v0.x line is documented in [`SPEC.md`](./SPEC.md) §1.

## Acknowledgments

The OpenClaw plugin surface, designed by Peter Steinberger and the OpenClaw maintainer team, exposes exactly the hooks a trust verification layer needs. The pattern composes against that surface and adds nothing to it. The original RFC framing in OpenClaw #49971 by Lars Kroehl identified the problem space and the hook coverage. The contributors to [aeoess/agent-governance-vocabulary](https://github.com/aeoess/agent-governance-vocabulary) ground the cross-provider semantics this pattern references.
