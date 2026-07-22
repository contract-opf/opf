# OPF JSON Schemas

These files are **byte-for-byte vendored copies** from the reference
implementation, [`contract-opf/playbook-engine`](https://github.com/contract-opf/playbook-engine)
(its `spec/` directory). Their canonical `$id`s resolve there —
`https://contract-opf.github.io/playbook-engine/spec/…` — so this repo is a
convenience mirror, not a second source of truth. **Re-vendor by copying
byte-for-byte; never hand-edit.**

| File | Version | Notes |
|---|---|---|
| `playbook.schema-0.3.json` | OPF **0.3** (current) | Additive over 0.2 — adds the `digest` section. Frozen at `digest_version` 2. |
| `playbook.schema-0.2.json` | OPF 0.2 | The three-section model (Evidence / Posture / Floor). |
| `playbook.schema.json` | OPF 0.1 | Initial draft. |

A conformant validator dispatches on a document's `opf_version` — one
validator, three schemas. See [`../OPF-SPEC.md`](../OPF-SPEC.md) §10 and
[`../CHANGELOG.md`](../CHANGELOG.md).
