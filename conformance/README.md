# OPF conformance vectors — canonicalization + digest

The vectors under [`0.6/`](0.6/README.md) are the **normative definition**
of two things for OPF 0.6 / `digest_version` 5 — the one format the
reference engine reads and writes:

1. **Canonicalization and content addressing** (`playbook_engine/canonicalize.py`)
   — the whole-document canonical form, `identity.content_hash`, and the
   three `identity.section_digests` values (`evidence` / `posture` / `floor`).
2. **The `digest` section** (`playbook_engine/digest.py`, OPF §3.12.2) — the
   compact model-facing projection of the precedent record.

An independent implementation (a different language, or a hand-maintained
port like the ones this suite exists to catch drift in — see issue #115)
that reproduces every `expected.*` value in `0.6/vectors/*.json` from that
vector's `input` **is conformant** with canonicalization and digest
construction for this format version. It needs no dependency on this repo
or on Python — every vector is plain JSON.

`tests/test_conformance_vectors.py` is the reference check: it recomputes
each vector from `input` using the engine's own `canonicalize.py`/`digest.py`
and asserts the result equals the vector's frozen `expected.*` values
byte-for-byte. The `expected.*` values are **frozen at generation time**
(via `scripts/generate_conformance_vectors.py`, a dev-only tool — see its
docstring) and never recomputed live by the test; that separation is what
makes the suite an actual drift detector instead of a tautology that always
passes.

## Canonicalization algorithm

`0.6/manifest.json`'s `algorithm` text says canonicalization is "exactly as
the 0.3 set (../manifest.json)". That manifest belonged to the OPF 0.3 /
digest 2 vector set, retired with that format (issue #238; git history
keeps it). The algorithm it stated, which the 0.6 vectors use unchanged:

- **canonical:** `json.dumps(value, sort_keys=True, separators=(',', ':'),
  ensure_ascii=False)` over the whole document after removing the top-level
  `identity` key and the `compiler.generated_at` /
  `compiler.run_id` sub-keys from a copy of the input (see
  `canonicalize.py`). Non-ASCII is emitted literally as UTF-8; Unicode is
  never normalized (NFC and NFD spellings hash differently); a whole-number
  float keeps its `.0`; array order is semantic and never sorted.
- **content_hash:** `'sha256:' + hex(sha256(canonical.encode('utf-8')))`.
- **section_digests[name]:** `'sha256:' + hex(sha256(canonical(input.get(name,
  {})).encode('utf-8')))` for `name` in `evidence` / `posture` / `floor` —
  excluding nothing (a section has no self-referential fields).

## Format version binding

Per the OPF-SPEC.md §11 immutability rule, these vectors are never edited in
place for the same format-version stamp. A format-version bump (a new
`opf_version` or a new `DIGEST_VERSION`) gets a **new, separately-stamped**
vector set, not an overwrite — exactly like a schema file itself. See
[`0.6/README.md`](0.6/README.md) for the vector-by-vector rationale and file
layout.
