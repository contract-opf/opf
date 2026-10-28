# OPF JSON Schemas

These files are **byte-for-byte vendored copies** from the reference
implementation, [`contract-opf/playbook-engine`](https://github.com/contract-opf/playbook-engine)
(its `spec/` directory). Their canonical `$id`s point there —
`https://contract-opf.github.io/playbook-engine/spec/…` — so this repo is a
convenience mirror, not a second source of truth. **Re-vendor by copying
byte-for-byte; never hand-edit.**

| File | Version | sha256 |
|---|---|---|
| `playbook.schema-0.6.json` | OPF **0.6** / `digest_version` 5 (current) | `b2a925811fd661c5d7c11eaa768444fa63764b479c34fff9a50187c580668bf2` |
| `clause-tree.schema.json` | the compiler's intermediate clause tree (not an OPF document) | — |

A conformant validator accepts `opf_version` "0.6" only and rejects every
other version as unsupported (see [`../OPF-SPEC.md`](../OPF-SPEC.md) §10 and §11).

## `retired/`

OPF 0.1–0.3 were retired on **2026-10-07** (issue #238), and 0.4 and 0.5 were
replaced in place by their successors (issues #233, #247), so no version before
0.6 is supported. Retirement never edits a format: the three schemas this repo
published are kept here, byte-unchanged, for anyone reading an old document.

| File | Version | sha256 (unchanged) |
|---|---|---|
| `retired/playbook.schema-0.3.json` | OPF 0.3 (digest_version 2) | `d2d81ca1c4f7547b508b2a22310906ce9a3bf43a2436e8730f0b1e4c9b0a0e15` |
| `retired/playbook.schema-0.2.json` | OPF 0.2 | `eae5f882f9289f2144cc784109d3dd04de7673d6e563d195fd693fd38ae1138d` |
| `retired/playbook.schema.json` | OPF 0.1 | `c1a25b477eeb71c9a6daa2d2d390793301df5e4e6539503e622b72d2f8276962` |

0.4 and 0.5 were never vendored here; they are in the reference engine's git
history. Their `$id`s no longer resolve.
