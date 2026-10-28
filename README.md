# Open Playbook Format (OPF)

**A machine-readable standard for contract-review playbooks — the negotiation
knowledge a legal team already has, made portable, auditable, and safe to hand
to an LLM.**

> **OPF 1.0 (stable)** · document shape `opf_version` **0.6** · `digest_version` **5** ·
> Apache-2.0 (schemas & code) + CC-BY-4.0 (spec text)

Every legal team already knows what it will accept. That knowledge lives in the
one place nobody can query: years of redlines, tracked changes, and signed PDFs.
A *playbook* is the distilled answer — for each clause, our standard position,
the variants we've accepted, what we've conceded under pressure, and what we've
refused. Historically it lived in a senior reviewer's head or a static checklist.

**OPF makes that playbook a document a machine can read.** One versioned,
hashed, JSON-Schema-validated file that any tool can execute — so a playbook can
be governed, diffed, audited, and pointed at a live contract, instead of living
only in institutional memory.

OPF is the **interface** between the corpus→playbook compiler and any downstream
review engine. Any tool that reads OPF can use your playbook; you are never
locked into one engine.

---

## The core idea: three sections, three bindings

A modern review model (a SOTA LLM) is capable enough that pre-freezing a
position for every clause is the *wrong* interface — it's rigid where the model
is flexible, and it hides the negotiation intent that actually drives a decision.

So OPF makes one structural move: **determinism migrates out of the knowledge
and into the guardrails.** A playbook is one document with three sections, each
authored by a different owner and each binding differently at review time:

| Section | What it carries | Author | Binding at review time |
|---|---|---|---|
| **Evidence** | What the corpus *shows*, as one verdict-free precedent record per (deal, clause): what the deal opened with, what it signed, whether that is our standard language, the asks refused before signing, and when it was signed — each cited to the document that proves it. *Descriptive, not prescriptive.* | Auto-derived by the compiler | **Advisory** — the model reasons over it |
| **Posture** | Negotiation *intent* as a system-prompt-style prose block: how many rounds, leverage, risk appetite, what's sacred vs. flexible, who reads the output. | Compiler drafts from a short interview; counsel edits & approves | **Soft** — shapes judgment, never a gate |
| **Floor** | The red lines that must **never** slip, as judge-checkable natural-language invariants ("never accept uncapped liability"). | Legal owner authors & signs (compiler may *propose* candidates) | **Hard** — a violation forces the outcome; the model and the Posture cannot override it |

This is the load-bearing contract of OPF, and the reason it's safe to point a
stochastic model at high-stakes legal work: **the model gets freedom in the soft
middle, bounded by a hard floor it can never cross and a citation it must always
give.**

---

## Why it's built this way

- **Descriptive, not prescriptive.** Evidence tells the model *what the corpus
  shows* ("9 of 12 signed deals kept our liability cap; 3 signed this variant"),
  never *what it must do*. Since 0.4 the evidence carries no stance, band, risk
  or deviation verdict at all — only facts and counts.
  What to actually do on a live clause is decided from Evidence + Posture + the
  deal in front of the model — except where a Floor invariant applies, which is
  non-negotiable. This keeps the useful signal and removes the rigidity that a
  frozen per-clause position baked in.

- **Every claim is cited.** No asserted clause text exists without a pointer to
  the exact `(document, version, clause, character span)` that proves it.
  Citations must resolve or the playbook is non-conformant. A playbook is
  auditable *by construction* — you can always see *why* something is recorded
  as a concession or a red line.

- **The Floor is judged, but coverage and consequence are deterministic.** Red
  lines are semantic, not lexical ("never accept uncapped liability" can't be a
  keyword match), so a dedicated judge evaluates each invariant. What's
  guaranteed in code is the **coverage gate** (every invariant is evaluated and
  logged on every review — an unevaluated one fails the run, fail-closed) and
  the **consequence** (a violation forces the negotiation-unacceptable outcome).

- **Portable and governed.** A canonical serialization + `content_hash` +
  per-section digests give the artifact a stable identity, so you can prove
  which exact playbook governed which review and reconstruct the full lineage:
  **corpus → playbook → decision.** No lock-in: the format is the contract, not
  the tool.

---

## What's already built

OPF isn't a paper standard. There's a working, end-to-end ecosystem around it:

```
   corpus of negotiated        playbook-engine          playbook.opf.json           contract-toaster
   agreements (signed    ──►   (deterministic     ──►   + digest + HTML bundle ──►  (or any OPF-aware
   + drafts; DOCX/PDF)         + LLM judgment)           every claim cited            reviewer / LLM)
                                                         schema-validated
          └───────────────────────────── all speak OPF ─────────────────────────────┘
```

- **The spec** *(this repo)* — OPF 0.6: the JSON schema ([`schema/`](schema/)),
  the normative specification ([`OPF-SPEC.md`](OPF-SPEC.md)), the
  [conformance vectors](conformance/), the shipped [taxonomies](taxonomy/), and
  the [changelog](CHANGELOG.md).

- **[`playbook-engine`](https://github.com/contract-opf/playbook-engine)** — the
  reference **compiler**. Point it at a folder of negotiated agreements (the
  signed copy plus the drafts exchanged along the way) and it compiles a
  validated OPF playbook — deterministic alignment plus LLM judgment, every
  statement cited. Runs from a Claude Code skill or Docker; there's a
  **one-minute, no-API-key quickstart** over a synthetic corpus.

- **[`contract-toaster`](https://github.com/contract-opf/contract-toaster)** —
  a reference **consumer**. An LLM-assisted reviewer that executes an OPF
  playbook against a counterparty's draft and returns **ACCEPT** or a
  tracked-changes redline with footnoted rationale.

- **[`playbooks`](https://github.com/contract-opf/playbooks)**
  — a real, browsable playbook compiled from negotiated
  educational-affiliation agreements and published pseudonymized as a
  pedagogical example. Every position cites real observations.
  **[See it rendered →](https://contract-opf.github.io/playbooks/)**
  *(The published example is still the OPF 0.3 compile; a 0.6 refresh is
  pending sign-off under that repo's publication policy.)*

---

## The format at a glance

```jsonc
{
  "opf_version": "0.6",
  "agreement_type": { "id": "educational-affiliation",
                      "name": "Educational Affiliation Agreement", "aliases": ["eiaa"] },
  "perspective":    { "party": "…", "counterparty_type": "Educational Institution" },
  "baseline":       { /* our template + any prior_templates (our former forms) */ },

  "evidence": {                       // DESCRIPTIVE · advisory — facts only, every text cited
    "clauses": [                      //   per clause type: our standard (as parts) + deal counts
      { "taxonomy_id": "assignment", "our_standard": { "text": "…", "parts": [ /* … */ ] },
        "n_deals": 12, "n_signed_standard": 9, "n_variants": 2, "n_refused": 1,
        "n_opened_standard": 10, "n_kept_standard": 8, "prior_standards": [ /* … */ ] }
    ],
    "precedent": [                    //   one verdict-free record per (deal, clause type)
      { "document_id": "…", "paper": "ours", "signed": true, "signed_on": "2025-01-10",
        "opened_with": "standard", "opening_text": null,
        "signed_text": { "text": "…", "ref": { /* document, version, clause_path, char_span */ },
                         "vs_standard": { "label": "less_protective", "parts": [ /* … */ ] } },
        "standard": false, "moved": true, "rounds": 2, "refused_asks": [ /* cited */ ] }
    ],
    "absence": { "guard": 0.1, "records": [ /* absent_verified | possibly_present, cited */ ] }
  },

  "posture": { "system_prompt": "…how to negotiate this agreement type…", "version": 1 },  // SOFT
  "floor":   { "invariants": [ { "statement": "Never accept uncapped liability." } ] },     // HARD

  "digest":   { "digest_version": "5", /* compact model-facing projection — designed to BE the prompt */ },
  "manifest": { /* the Floor as a hard-rule list */ }, "dossiers": [ /* per-clause critic briefs */ ],
  "corpus":   { /* per-document provenance, signing dates, content addresses, snapshot hash */ },
  "identity": { "content_hash": "sha256:…", "section_digests": { "evidence": "sha256:…", "...": "..." } }
}
```

A few things worth calling out:

- **Verdict-free precedent.** Each `evidence.precedent[]` record states facts —
  what the deal opened with (`opened_with`: `standard` / `prior_standard` /
  `non_standard` / `absent`), what it signed, whether that is our standard
  (`standard`), whether the clause `moved`, the `refused_asks`, and `signed_on`.
  It never says "held" or "conceded"; a consumer or the digest derives those.
- **`vs_standard`** is the one judged field outside `x_judgments`: each distinct
  non-standard text is compared to our standard *part by part*, blind-checked
  by an independent model, and its `settled` state (`agreed` / `adjudicated` /
  `unchecked` / `owner`) travels with it.
- **Absence is verified, not assumed.** A deal with no record for a clause may
  just be a missed classification, so `evidence.absence` scores the deal's
  unclassified text against the clause and records `absent_verified` or
  `possibly_present` with cited candidates.
- **Our former forms** (`baseline.prior_templates`, `prior_standards`) keep our
  own older wording from reading as a counterparty concession.
- **`digest`** (`digest_version` 5) is the compact projection of `evidence`
  designed to *be* the system-prompt payload of a review app, with
  `example_ref` / `precedent_ids` that drill back into the full document.
  `manifest`, `dossiers` and `provenance_index` give a reviewer the Floor as
  rules and a budgeted, cited brief per clause.
- **The provenance rule.** Only *our-paper* drafting may define an opening
  position; a clause that survived only in the counterparty's template MUST NOT
  set our standard. Survivorship is not endorsement.
- **`x_*` vendor extensions** are reserved at designated levels; conformant
  consumers ignore unknown ones, and a playbook stripped of every `x_*` field
  must mean the same thing.

The full normative detail — the precedent record (§3.5.4–§3.5.8), the digest
(§3.12), citations (§4), the determinism boundary (§5),
producer/author/consumer responsibilities (§6), the Posture interview (§7),
governance & lineage (§8), conformance (§10), versioning (§11) — is in
**[`OPF-SPEC.md`](OPF-SPEC.md)**.

---

## What's in this repo

```
opf/
├── README.md                        # this file
├── OPF-SPEC.md                      # the normative specification (CC-BY-4.0)
├── CHANGELOG.md                     # spec changelog, with the current sha256 pins
├── LICENSE                          # Apache-2.0
├── schema/
│   ├── playbook.schema-0.6.json     # current — OPF 0.6 / digest_version 5
│   ├── clause-tree.schema.json      # the compiler's intermediate clause tree (not OPF)
│   └── retired/                     # OPF 0.1–0.3, retired 2026-10-07 (kept for history)
├── conformance/
│   ├── README.md                    # what the vectors define and how to use them
│   └── 0.6/                         # manifest.json + 14 {input, expected} vectors
└── taxonomy/                        # the clause taxonomies the reference compiler ships
```

Every file except this README and [`schema/README.md`](schema/README.md) is
**byte-vendored** from the reference implementation's `spec/` and
`docs/OPF-SPEC.md`, where the canonical `$id`s resolve. Paths inside the
vendored text (`playbook_engine/…`, `tests/…`, `docs/…`) refer to
[`playbook-engine`](https://github.com/contract-opf/playbook-engine).

| File | sha256 |
|---|---|
| `schema/playbook.schema-0.6.json` | `b2a925811fd661c5d7c11eaa768444fa63764b479c34fff9a50187c580668bf2` |
| `conformance/0.6/` (manifest.json + vectors/*.json, concatenated) | `c5e3828e7d59c961aca2d633b390b3ea536902e7818efe0dd778a0916cf8be96` |

These match the **Current pins** in [`CHANGELOG.md`](CHANGELOG.md).

---

## Status & roadmap

The spec text is **OPF 1.0 (stable)**: a stability commitment, not a shape
change. The one current document shape is **`opf_version` 0.6** with
**`digest_version` 5**; the reference engine reads and writes exactly that.

- **Retirement (2026-10-07).** With no installed consumer to stay compatible
  with, the owner retired OPF 0.1–0.3 and `digest_version` 2 (issue #238);
  0.5 then replaced 0.4 in place, and 0.6 replaced 0.5 in place (issues #233,
  #247), so **0.1–0.5 are all retired** and a conformant validator rejects them
  as unsupported. Retirement never edits a format: the 0.1–0.3 schemas this repo
  published are kept unchanged under [`schema/retired/`](schema/retired/); 0.4
  and 0.5 never left the reference engine and live in its git history. A
  document in a retired format is not converted — re-compile it.
- **Stability policy.** Within 1.x, a release may add an OPTIONAL field or an
  `x_*` extension, never a REQUIRED field, a removal, a retype or a changed
  meaning; such an additive change still ships as a new `opf_version`. Any new
  or changed MUST is recorded under `### Normative rule changes`.
- **Immutability rule.** Once a consumer exists, a published `opf_version` is
  immutable. Spec-affecting changes are pinned by hash and CI-enforced.
- **Open questions** (see the spec's Appendix A): the concrete *interface* a
  composed clause-intelligence module exposes (the governance contract is fixed;
  the wiring is deferred), and pressure-testing Floor minimality on a mature
  production Floor.

---

## Get involved

We're building an open community around evidence-derived, governable
negotiation playbooks — and we'd love implementers, reviewers, and critics.

- **Read** the spec ([`OPF-SPEC.md`](OPF-SPEC.md)) and the
  [schema](schema/playbook.schema-0.6.json).
- **Implement** it in another language: reproduce every `expected.*` value in
  [`conformance/0.6/vectors/`](conformance/0.6/) from its `input` and your
  canonicalization and digest are conformant — no Python or engine dependency.
- **Try** the reference compiler's one-minute quickstart in
  [`playbook-engine`](https://github.com/contract-opf/playbook-engine) — no API
  key needed.
- **Browse** a real playbook:
  [contract-opf.github.io/playbooks](https://contract-opf.github.io/playbooks/).
- **Open an issue or discussion** for: a new agreement type, a consumer
  implementation, a schema question, or any of the open questions above.
  Agreement-type `id`s are self-assigned — there's no central registry; a
  namespacing convention keeps them from colliding.

---

## License

- **Schemas and code:** Apache License 2.0 — see [`LICENSE`](LICENSE).
- **Specification text** (`OPF-SPEC.md`): additionally licensed under
  [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/).

## Not legal advice

OPF is a data format, and a playbook expressed in it encodes one organization's
negotiating positions for reference only. Any review an OPF playbook drives is a
tool recommendation, not legal advice — attorney approval is always required.
