# Open Playbook Format (OPF)

**A machine-readable standard for contract-review playbooks — the negotiation
knowledge a legal team already has, made portable, auditable, and safe to hand
to an LLM.**

> **Version 0.3** · Apache-2.0 (schemas & code) + CC-BY-4.0 (spec text) ·
> pre-1.0, breaking changes possible until 1.0

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
| **Evidence** | What the corpus *shows*: accepted variants, concessions, rejections, per-round negotiation trails — each cited to the document that proves it. *Descriptive, not prescriptive.* | Auto-derived by the compiler | **Advisory** — the model reasons over it |
| **Posture** | Negotiation *intent* as a system-prompt-style prose block: how many rounds, leverage, risk appetite, what's sacred vs. flexible, who reads the output. | Compiler drafts from a short interview; counsel edits & approves | **Soft** — shapes judgment, never a gate |
| **Floor** | The red lines that must **never** slip, as judge-checkable natural-language invariants ("never accept uncapped liability"). | Legal owner authors & signs (compiler may *propose* candidates) | **Hard** — a violation forces the outcome; the model and the Posture cannot override it |

This is the load-bearing contract of OPF, and the reason it's safe to point a
stochastic model at high-stakes legal work: **the model gets freedom in the soft
middle, bounded by a hard floor it can never cross and a citation it must always
give.**

---

## Why it's built this way

- **Descriptive, not prescriptive.** Evidence tells the model *what the corpus
  shows* ("we've consistently held the liability cap"), never *what it must do*.
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

- **The spec** *(this repo)* — OPF 0.3: the JSON schemas ([`schema/`](schema/)),
  the normative specification ([`OPF-SPEC.md`](OPF-SPEC.md)), and the
  [changelog](CHANGELOG.md).

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

- **[`eiaa-example-playbook`](https://github.com/contract-opf/eiaa-example-playbook)**
  — a real, browsable playbook, compiled from **44 negotiated
  educational-affiliation agreements (161 versions)** and published pseudonymized
  as a pedagogical example. Every position cites real observations.
  **[See it rendered →](https://contract-opf.github.io/eiaa-example-playbook/)**

---

## The format at a glance

```jsonc
{
  "opf_version": "0.3",
  "agreement_type": { "id": "educational-affiliation",
                      "name": "Educational Affiliation Agreement", "aliases": ["eiaa"] },
  "perspective":    { "party": "…", "counterparty_type": "Educational Institution" },

  "evidence": {                       // DESCRIPTIVE · advisory — every claim cited
    "clauses": [                      //   template-anchored, per clause:
      { "our_standard": { "...": "..." },
        "observed_positions": [ /* variant + deviation + risk_delta + provenance + outcome */ ],
        "negotiation_trail": [ /* round-by-round ask → landing, each citing the post-move state */ ],
        "summary": { "historical_stance": "usually_held",   // descriptive, NOT an instruction
                     "acceptable_if": [ /* cited tolerance conditions */ ],
                     "fallbacks": [ ], "rejected": [ ] } }
    ],
    "clause_library": [ /* concept-indexed positions from counterparty paper */ ]
  },

  "posture": { "system_prompt": "…how to negotiate this agreement type…", "version": 1 },  // SOFT
  "floor":   { "invariants": [ { "statement": "Never accept uncapped liability." } ] },     // HARD

  "digest":   { /* compact model-facing projection of evidence (0.3) — designed to BE the prompt */ },
  "corpus":   { /* per-document provenance, content addresses, snapshot hash — out-of-scope docs retained */ },
  "identity": { "content_hash": "sha256:…", "section_digests": { "evidence": "sha256:…", "...": "..." } }
}
```

A few things worth calling out:

- **`historical_stance`** (`consistently_held` / `usually_held` / `mixed` /
  `usually_conceded` / `no_signal`) answers *"what has the corpus shown we do
  here?"* — never *"what must you do?"*. A consumer treats it as evidence, not a
  directive.
- **The provenance rule.** Only *our-paper* drafting may define an opening
  position; a clause that survived only in the counterparty's template informs
  tolerance bounds but MUST NOT set our standard. Survivorship is not
  endorsement.
- **`digest`** (new in 0.3) is the compact projection of `evidence` designed to
  *be* the system-prompt payload of a review app — capped to a token budget,
  with `example_ref` citations that drill back into the full document.
- **`x_*` vendor extensions** are reserved at designated levels; conformant
  consumers ignore unknown ones, and a playbook stripped of every `x_*` field
  must mean the same thing.

The full normative detail — citations (§4), the determinism boundary (§5),
producer/author/consumer responsibilities (§6), the Posture interview (§7),
governance & lineage (§8), conformance (§10) — is in
**[`OPF-SPEC.md`](OPF-SPEC.md)**.

---

## What's in this repo

```
opf/
├── README.md                      # this file
├── OPF-SPEC.md                    # the normative specification (CC-BY-4.0)
├── CHANGELOG.md                   # spec changelog
├── LICENSE                        # Apache-2.0
└── schema/
    ├── playbook.schema-0.3.json   # current — OPF 0.3 (frozen at digest_version 2)
    ├── playbook.schema-0.2.json   # OPF 0.2 (three-section model)
    └── playbook.schema.json       # OPF 0.1 (initial draft)
```

Validators dispatch on the document's `opf_version` — one validator, three
schemas. The schema files are byte-vendored from the reference implementation,
where their canonical `$id`s resolve; see [`schema/README.md`](schema/README.md).

---

## Status & roadmap

OPF is **0.3**, frozen at `digest_version` 2 and additive over 0.2 (a 0.2
document is a valid 0.3 document once its version is bumped). It's **pre-1.0**:
minor versions may still break compatibility.

- **Immutability rule.** Once a consumer exists, a published `opf_version` is
  immutable — shape or *semantic* changes get a new version, never an in-place
  edit. Spec-affecting changes are pinned by hash and CI-enforced.
- **Stable:** the three-section model, the risk-delta/provenance model,
  citations, `identity`/content-hashing, the conformance rules.
- **Open questions** (see the spec's Appendix A): the concrete *interface* a
  composed clause-intelligence module exposes (the governance contract is fixed;
  the wiring is deferred), and pressure-testing Floor minimality on a mature
  production Floor.

---

## Get involved

We're building an open community around evidence-derived, governable
negotiation playbooks — and we'd love implementers, reviewers, and critics.

- **Read** the spec ([`OPF-SPEC.md`](OPF-SPEC.md)) and the
  [schema](schema/playbook.schema-0.3.json).
- **Try** the reference compiler's one-minute quickstart in
  [`playbook-engine`](https://github.com/contract-opf/playbook-engine) — no API
  key needed.
- **Browse** a real playbook:
  [contract-opf.github.io/eiaa-example-playbook](https://contract-opf.github.io/eiaa-example-playbook/).
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
