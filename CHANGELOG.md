# Spec changelog

One entry per spec-affecting change (schema files, `digest` shape/semantics,
canonicalization, normative validator rules). Downstream consumers vendor
these schemas hermetically — this file is how they diff intent without
reverse-engineering `git log`.

## Versioning policy (normative, effective 2026-07-16)

- **A published `opf_version` is immutable once a consumer exists.** Shape or
  *semantic* changes to what a document's fields mean get a NEW `opf_version`
  (0.3 → 0.4), never an in-place edit of a released schema. "The schema still
  validates old data" is NOT sufficient — two artifacts claiming the same
  version must mean the same thing.
- **`digest_version` governs the digest section independently** and follows
  the same rule: any change to the digest's shape OR selection semantics
  (what the lists include/exclude, how entries are ranked or capped) bumps it.
- CI enforces the paper trail: `tests/test_spec_consistency.py` pins each
  schema file's sha256 to the **Current pins** table below — any schema edit
  fails CI until this changelog gains an entry and updated pins. That
  friction is deliberate.

## Current pins

| File | sha256 |
|---|---|
| `playbook.schema.json` | `c1a25b477eeb71c9a6daa2d2d390793301df5e4e6539503e622b72d2f8276962` |
| `playbook.schema-0.2.json` | `eae5f882f9289f2144cc784109d3dd04de7673d6e563d195fd693fd38ae1138d` |
| `playbook.schema-0.3.json` | `d2d81ca1c4f7547b508b2a22310906ce9a3bf43a2436e8730f0b1e4c9b0a0e15` |

Current `DIGEST_VERSION`: **2** (`playbook_engine/digest.py`).

## History

### 2026-07-16 — OPF 0.3 FROZEN (at digest_version 2, engine PR #230)

`opf_version` 0.3 and `digest_version` 2 are frozen as of this entry. Any
further spec-affecting change goes to 0.4 (or digest_version 3). Consumers
should vendor `playbook.schema-0.3.json` at or after engine commit
`204057e` (merge of PR #230) — earlier same-day 0.3 states are superseded
(see below) and reject current artifacts.

### 2026-07-16 — digest budget enforcement (PR #230; digest_version 1 → 2)

- `preferred_variations` deduplicated/ranked/capped like the other lists;
  digest entries are now `{if, to, observation_ref, n, band}` projections
  (new `$defs.digestPreferredVariation`; the compiler-generated `rationale`
  stays in the full OPF). **Breaking for consumers of the day-old
  digest_version 1 shape**: entries validated as `acceptableIfEntry`
  (which requires `rationale`) no longer appear in digests.
- `build_digest` enforces the ~40K-token budget by construction (per-list
  cap tightens 5 → 4 → 3).
- `digest_version` bumped `"1"` → `"2"`.

### 2026-07-16 — digest list semantics change (PR #229) — RETROSPECTIVE NOTE

`concessions`/`unacceptable` moved from a 1:1 projection of
`summary.fallbacks`/`rejected` to deduplicated, precedent-weighted,
top-5-plus-material capped lists; `digestObservationSummary` gained
`n`/`band`. **This was a semantic change shipped without a
`digest_version` bump — a violation of the policy above, which this
changelog exists to prevent recurring.** It was corrected hours later by
PR #230's bump to digest_version 2 (which covers both changes); no
consumer had bound an artifact in the gap. Flagged by the contract-toaster
team — the pin-and-verify discipline that caught it is the intended
consumer posture.

### 2026-07-16 — OPF 0.3 introduced (PR #227)

- New optional top-level `digest` section (`playbook.schema-0.3.json`,
  OPF-SPEC §3.12); validator accepts 0.1/0.2/0.3; digest covered by
  `identity.content_hash`; normative rules: digest clause ids must match
  evidence, no `full_text` anywhere in the digest.
- Single-file bundle artifact `playbook.opf.html` (`view bundle`) embedding
  the canonical JSON + digest in `<script type="application/json">` blocks
  (ids `opf-canonical`/`opf-digest`, `</` escaped as `<\/`).
- `playbook.schema-0.2.json` unchanged; 0.2 documents remain valid.

### Earlier

- **0.2** — three-section model (Evidence/Posture/Floor); see OPF-SPEC
  Appendix B.
- **0.1** — initial draft; retained for history (`playbook.schema.json`,
  `docs/OPF-SPEC-v0.1.md`).
