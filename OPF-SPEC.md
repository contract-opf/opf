# Open Playbook Format (OPF) — Specification

**Version:** 1.0
**Status:** Stable — the first non-beta release. 1.0 is a stability commitment, not a shape change. Format changes from here are governed by the stability policy in §11 (a breaking shape or normative-rule change requires 2.0). The reference compiler emits and validates exactly one document shape, `opf_version` "0.6" (the verdict-free per-deal precedent record, §3.5.4, with the opening evidence of §3.5.5 and a `digest_version` "5" digest, §3.12.2). The 0.1–0.4 shapes and digest 2 were retired (owner decision 2026-10-06: no installed consumer needs them); their schemas and text stay in git history and in contract-opf/opf.
**Serialization:** JSON (canonical). YAML permitted for authoring; tools MUST accept both and treat them as equivalent.
**Issue references:** #NNN citations throughout are design provenance from the project's original development tracker.
**License:** This file is specification text and is additionally licensed under the Creative Commons Attribution 4.0 International License (CC-BY-4.0), per the repository `LICENSE` — alongside the Apache-2.0 license covering this repository.

---

## 0. What changed in 0.2, and why

v0.1 modeled a playbook as a set of **clause positions with a frozen, operative posture per clause** (`rollup.position ∈ {standard, acceptable_variants_exist, negotiable, hold_firm}`). That made sense when the consumer was a less-capable reviewer that needed the answer pre-computed. The consuming review engine is now a SOTA model. Pre-freezing a posture per clause is no longer the right interface: it is rigid where the model is flexible, and it hides the negotiation intent that actually drives a decision.

v0.2 makes one structural move: **determinism migrates out of the knowledge and into the guardrails.**

A playbook is now **one document with three sections**, authored at different times by different owners:

| Section | What it carries | Author | Determinism at runtime |
|---|---|---|---|
| **Evidence** (§3.5) | What the corpus shows — accepted variants, concessions, rejections, each cited to precedent. *Descriptive, not prescriptive.* | Auto-derived by the compiler | **Advisory** — the model reasons over it |
| **Posture** (§3.6) | Negotiation intent as a system-prompt-style prose block (rounds, leverage, risk appetite, what's sacred vs. flexible, output audience). | Compiler **drafts** from a short interview; a legal owner edits | **Soft** — shapes judgment, not a gate |
| **Floor** (§3.7) | The hard lines that must *never* slip: judge-evaluated natural-language invariants. | Legal owner authors (compiler may *propose* candidates) | **Hard** — deterministic coverage gate + forced consequence; the model cannot override |

Plus one new top-level concern: **composition** (§3.4) — an OPF may declare *composed* external clause-intelligence modules (e.g. a privacy/DPA module) rather than re-encoding that knowledge itself: a pinned, validated dependency recorded for lineage under a fixed governance contract. (Behavior wiring is deferred — see §3.4's open note; today `composes` is declared and verified, not yet executed.)

Everything that made v0.1 trustworthy is retained and, in the Floor and citation rules, sharpened: every asserted text still cites precedent (§4); the provenance rule still holds (§2.3); out-of-scope documents are still retained (§3.8). The change is *not* "less governance because the model is smarter" — it is *more* model freedom in the soft middle, bounded by a hard floor it can never cross and a citation it must always give.

This section summarizes the reframe.

---

## 1. Purpose and scope

OPF is an open format for representing a **negotiation playbook** for a single agreement type, compiled from a corpus of real negotiated agreements. A playbook tells a reviewer — human or LLM — how to evaluate a clause in a new agreement: what the preferred position is, what the corpus shows we have accepted, conceded, and rejected, what our negotiation posture is, and what hard lines must never be crossed — each factual claim grounded in cited precedent.

OPF describes **what the playbook knows and intends**, not the review engine's internals (thresholds, retrieval, model policy). OPF is the interface between the corpus→playbook compiler and any downstream review engine.

### Non-goals
- OPF is not a contract format and does not represent agreements themselves.
- OPF does not encode the review engine's thresholds, retrieval, or model policy.
- OPF is not a trained model; it carries no weights.
- OPF does not own the *runtime lifecycle* of an edited Posture. The OPF carries the **initial, compiler-generated** Posture (the genesis record); once a playbook is installed, the consuming app's release-bundle governance owns subsequent Posture versions (§8).

> **OPF vs. the consuming app's release bundle:** OPF is the canonical
> playbook format — knowledge, intent, hard lines, perspective, and de
> minimis. The consuming app's release bundle wraps an
> OPF document and owns model policy, the output/leakage contract, and
> release (version/sign-off/activation/rollback). See
> `docs/OPF-BUNDLE-BOUNDARY.md` for the full boundary statement (supersedes
> the converter framing in #115).

## 2. Core concepts

| Term | Meaning |
|---|---|
| **Agreement type** | One playbook covers exactly one type (e.g. "Educational Affiliation Agreement"). |
| **Baseline / our paper** | The party's own canonical template, when one exists. Edits against it are *deviations* — the strongest signal. |
| **Counterparty paper** | A document drafted on the other side's template. Clauses surviving here were *tolerated*, not necessarily *endorsed*. |
| **Evidence** | The compiled, cited record of what the corpus shows. Descriptive. (§3.5) |
| **Posture** | Forward-looking negotiation intent, as system-prompt-style prose. (§3.6) |
| **Floor** | The deterministic hard lines that must never slip. (§3.7) |
| **Composition** | A pinned dependency on an external clause-intelligence module. (§3.4) |
| **Observation** | A single occurrence of a clause variant in the corpus, with its outcome and provenance. |
| **Deviation** | How much an observed clause differs from our standard: `none` / `reworded_equivalent` / `substantive`. |
| **Risk delta** | The risk change relative to our standard: a `direction` (`better`/`neutral`/`worse`) and a `magnitude` (`none`/`minor`/`material`). |
| **Outcome** | What happened to an observed variant: `signed` or `proposed_then_reversed` (an explicit rejection). |
| **Provenance** | `our_paper` or `counterparty_paper`. |
| **Historical stance** | A *descriptive* summary of what the corpus shows we have done on a clause (§2.2). Replaces v0.1's prescriptive `rollup.position`. |

### 2.1 The risk-delta model (unchanged from v0.1)

OPF does not carry flat `acceptable_variant` / `concession` labels. An observation carries orthogonal attributes — `deviation`, `risk_delta`, `provenance`, `outcome` — and the human-facing distinction is **derived**:

- *Acceptable variant* = signed observation with `risk_delta.direction = neutral`.
- *Concession* = signed observation with `risk_delta.direction = worse` (magnitude tells you how reluctant).
- *Rejected ask* = observation with `outcome = proposed_then_reversed`.

This keeps the format auditable (you can always see *why* something is a concession) and general across agreement types.

### 2.2 Historical stance is descriptive, not an instruction (the key 0.2 change)

v0.1's `rollup.position` enum tried to be the operative instruction ("hold firm here"). v0.2 replaces it with **`historical_stance`** — a purely *descriptive* summary of the corpus:

> `historical_stance ∈ { consistently_held, usually_held, mixed, usually_conceded, no_signal }`

It answers "**what has the corpus shown we do here?**", never "what must you do here?". A consumer MUST treat `historical_stance` as evidence, not as a directive. What to actually *do* on a live clause is decided by the model from Evidence + Posture + the specifics of the deal in front of it — except where a Floor rule applies, which is non-negotiable (§3.7, §5).

Rationale: the descriptive signal ("we have consistently held the liability cap") is genuinely useful and worth compiling; the *prescription* baked into v0.1's enum is what made it rigid. Splitting description from prescription keeps the signal and removes the rigidity.

### 2.3 The provenance rule (normative — unchanged)

> Only **our-paper** drafting (the canonical template and our-paper deals) MAY define an *opening position* (`our_standard`, and any `historical_stance` stronger than `mixed`).
> **Counterparty-paper** observations MAY only inform tolerance bounds (`fallbacks`, `acceptable_if`) and populate the `clause_library`. A counterparty-paper-only clause MUST NOT set `our_standard`.

Rationale: a clause surviving in the counterparty's template means we failed to strike it, not that we'd propose it. Survivorship is not endorsement.

## 3. Document structure

A playbook is a single JSON object:

```jsonc
{
  "opf_version": "0.6",
  "agreement_type": { "id": "...", "name": "...", "description": "...", "aliases": ["..."] },
  "baseline": { ... },          // §3.2
  "taxonomy": { ... },          // §3.3
  "composes": [ Composition ],  // §3.4 — pinned external module deps (NEW)
  "perspective": { ... },       // §3.1 — whose side this playbook reviews from (NEW)
  "de_minimis": [ "..." ],      // §3.1 — change categories accepted even if novel (NEW)
  "evidence": {                 // §3.5 — the compiled, cited knowledge
    "clauses": [ Clause ],                // one per clause type, with counts (§3.5.4)
    "precedent": [ Precedent ]            // one verdict-free record per (deal, clause type)
  },
  "posture": { ... },           // §3.6 — generated negotiation intent (NEW)
  "floor": { ... },             // §3.7 — hard lines: judged NL invariants (NEW)
  "corpus": { ... },            // §3.8 — provenance/audit + content addresses
  "compiler": { ... },          // §3.9 — generation metadata
  "identity": { ... },          // §3.10 — content hash + section digests (NEW)
  "digest": { ... },            // §3.12 — compact model-facing projection (OPTIONAL)
  "manifest": { ... },          // §3.12.3 — hard-rule manifest, from the Floor (OPTIONAL)
  "dossiers": { ... },          // §3.12.3 — per-clause critic dossiers, bounded by a budget (OPTIONAL)
  "provenance_index": { ... }   // §3.12.3 — where the dossier excerpts came from (OPTIONAL)
}
```

### 3.1 Top-level fields
- `opf_version` (string, required) — `"0.6"`, the one supported version.
- `agreement_type` (object, required) — `id` (slug), `name`, `description`, `aliases[]`.
- `perspective` (object, OPTIONAL) — whose side this playbook is reviewed *as*: `party` (our legal entity/party name) and `counterparty_type` (what the other side typically is, e.g. `"Educational Institution"`). An open-standard OPF instance must say who "us" is — negotiation knowledge is meaningless without it. Owned by OPF (see `OPF-BUNDLE-BOUNDARY.md`).
- `de_minimis` (array of strings, OPTIONAL) — categories of change accepted even when technically novel (e.g. `"typo fixes"`, `"renumbering with no substantive change"`). This is negotiation knowledge, not a runtime policy, so it lives in OPF rather than the consumer's bundle.

  `agreement_type` is the shared cross-tool key for "which contract type is this" — a
  consuming app (e.g. a review tool with its own playbook registry) matches on it rather
  than maintaining a hand-joined mapping between its own key and OPF's.

  - `id` (string, required) — a lowercase slug, `^[a-z0-9-]+$`. Ids are **self-assigned**:
    there is no central registry. An id MAY be bare (`"educational-affiliation"`) or
    self-namespaced by convention (e.g. `"fixturecorp-eiaa"`, org-prefix + hyphen) to reduce the
    chance of collision; namespacing is optional and OPF does not reserve or validate any
    prefix. Two implementers minting the same bare id for different agreement types is a
    collision the format does not prevent — resolve it consumer-side (§`aliases` below, or
    a namespaced id).
  - `name` (string, required) — human-readable label.
  - `description` (string, optional).
  - `aliases` (array of string, optional) — other identifiers this same agreement type is
    known by elsewhere, e.g. a consuming app's own registry/dial key (`"eiaa"`). Aliases
    are free-form strings (no slug pattern enforced) since they mirror whatever key the
    other system already uses. A consumer SHOULD match on `id` first and fall back to
    `aliases` membership before concluding two playbooks describe different agreement
    types.

### 3.2 `baseline`
```jsonc
{
  "has_canonical_template": true,
  "template_ref": { "document_id": "...", "title": "...", "source": "path or URI" },
  "prior_templates": [                       // OPF 0.6, issues #249, #250; absent when there is no former form
    { "label": "2019 form", "in_use_from": "2019" | "2019-03" | "2019-03-14",
      "in_use_until": "2021-06", "sha256": "sha256:…" },   // SUPPLIED: dates and sha256 OPTIONAL
    { "label": "reconstructed 2021-03 to 2022-11", "basis": "reconstructed",
      "n_deals": 7, "first_signed": "2021-03", "last_signed": "2022-11" }   // RECONSTRUCTED, §3.5.7
  ],
  "notes": "free text"
}
```
`prior_templates` lists OLDER versions of our own form the producer was given, in
the order it was given them (§3.5.7): each one's unique `label`, the in-use
dates the owner stated (at the precision stated) and the content address of the
form's source file. Dates the owner did not state are absent here; the digest
infers them from deal dates and says so (§3.12.2). A form's `sha256` is what a
citation of the form resolves through (§4.1), so a SUPPLIED form any citation
names MUST carry it. An entry without `basis` is supplied; `basis:
"reconstructed"` marks a form the producer rebuilt from the corpus' first
drafts because none was supplied (§3.5.7): it has no file, so no `sha256` and no
owner dates, and carries `n_deals` and the `first_signed` / `last_signed` span
(`YYYY-MM`, absent when no deal is dated) of the deals that opened on it. A
supplied form always wins: the two kinds never appear together.

If `has_canonical_template` is `false`, the playbook is *emergent*. The provenance rule still applies; with no template, `our_standard` is set only from our-paper deals, and most stances will be `mixed`/`no_signal`.

### 3.3 `taxonomy`
A curated clause taxonomy (unchanged from v0.1). Entries may be inactive so curation survives upstream taxonomy upgrades. A compiler MUST only classify clauses into `active` or `custom` entries.

How a compiler reached a clause's `taxonomy_id` is the compiler's own record, not part of the format. The reference engine's classification bases are: an exact match of the clause heading to an entry label; heading similarity; a judge's verdict; inheritance from the clause's classified parent; the LLM segmenter's combined pass; and `content_similarity`, a deterministic fallback that assigns a clause every other path left unclassified when its text is close enough to the compiler's own standard text for exactly one entry (a token-overlap score of at least 0.25 and at least twice the runner-up's, at a confidence below 0.70). A content-similarity assignment is a compiler heuristic, never a verified verdict, and the engine assigns nothing on a guess: a clause that scores below the threshold, or whose best and runner-up entries are close, stays unclassified. The engine keeps the basis as the vendor key `x_classification_basis` on its observation store; it is not written into `evidence`, and no section is partitioned by it (prose only; no schema change).

The reference engine repairs segmentation before it builds precedent, because a precedent text that is half another clause, or a clause cut short, is compared as if it were one whole clause. It re-segments each version after classification, in three deterministic steps that use no judge and no key: it splits a sentence or list item that is another clause type's vocabulary out of the node it sits in (a unit moves only when at least half its content words, and at least five of them, are characteristic of the other type, it leads the node's own type and every other type by a margin, and the other type is seen in at least three deals; the new node carries `classification.basis` `resegmented` at a confidence below 0.70); it rejoins a node cut off mid-sentence or mid-list with the next one when that one continues it (starts in lower case or continues the list numbering) and both share a type or the next is unclassified, and it stitches the pieces of one author's tracked insertion that runs across the cut when the text continues across it (the piece before ends open, or the next opens in lower case or with a back-reference; a later list marker on the next counts only when the piece before ends open) and the next piece is positively of the node's type (typed so, or unclassified with at least two of that type's characteristic words and more of them than of any other type's, or a lead-in followed by such a piece; the insertion alone is no such evidence, and a run of whole clauses of one type that each read as complete stays separate clauses); a piece that opens with a heading of its own is never joined (the segmenter's label alone is not a clause start), nor is a piece, typed or not, that the not-a-clause test below rejects (a lead-in or a fragment aside), nor an unclassified piece whose words are another clause type's; and it marks an unclassified node that is not a clause (a bare section number, a signature block, a running header, a lead-in, a leftover fragment, extraction junk) as `x_non_clause` with a reason, leaving it out of classification, precedent and absence counts. Nothing is deleted. Every split and join is recorded on the node (`x_resegmented`: `split_from` or `joined_with`, `basis`, `origin_span` or `origin_spans`, and `scores`, which is empty for a join and for the node a passage was cut out of; a join also records `at`, where each joined passage starts in the node's text), the node's `char_span` lies inside the span it was cut from (or covers the two it was joined from), and a node cut out of clause `N` is numbered `N.r<k>`, so a citation still resolves to the original bytes (§4.1). These are vendor keys on the engine's observation store and clause trees; they are not written into `evidence` and no section is partitioned by them (prose only; no schema change). A sample of the splits and joins is put to the `resegment_check` judge kind, which measures the pass and gates nothing. After these three steps, a classified node that one of its sentences or list items ties to a second clause type (at least two, and at least a fifth, of the unit's content words characteristic of another type that is seen in at least three deals) is put to the `subsplit` judge kind: the node's text cut into numbered sentences and list items and the allowed clause types, answered with a partition into contiguous spans of one type each (or no type) or "no split". An answer is grounded the way a segmenter's block references are (the spans cover every unit exactly, in order, none empty, with ids from the allowed list), and only an owner's answer, or one whose independent blind check agreed or that was adjudicated, is applied: the first span stays the node, each later span becomes a node numbered `N.s<k>` with `classification.basis` `subsplit`, and every one records `x_resegmented` (`subsplit_from`, `basis` `agent`, `origin_span`) and cites a span inside the original block. A false clause boundary is worse than a mixed node, so the agent may always answer "no split".
```jsonc
{
  "source": "CUAD-v1",
  "entries": [
    { "id": "indemnification", "label": "Indemnification", "status": "active",
      "cuad_origin": "Indemnification", "description": "Who bears third-party claim risk." }
  ]
}
```

### 3.4 `composes` (NEW) — pinned external clause-intelligence modules

An OPF MAY compose external modules so it does not re-encode clause knowledge that a maintained module already carries (e.g. a privacy/DPA module supplies data-processing clause intelligence to a commercial-agreement playbook).

```jsonc
"composes": [
  {
    "module": "privacy-legal/dpa-review",       // module identifier
    "version": "1.4.0",                          // exact version
    "integrity": "sha256:…",                     // REQUIRED — pinned content hash
    "applies_to_taxonomy": ["data_processing", "confidentiality"],
    "role": "clause_intelligence",               // what it contributes
    "notes": "Supplies DPA/privacy clause positions for data-handling sections."
  }
]
```

**Normative rules:**
1. A composed module is **legal behavior**. Every entry MUST carry an exact `version` and an `integrity` hash. A consumer MUST refuse to load a module that is unpinned, or whose content does not match `integrity` (**fail-closed**) — never silently fall back to "latest". This is the same anti-drift discipline the consuming app applies to its own vendored code.
2. A composed module MAY supply Evidence-equivalent guidance and Floor-equivalent hard lines *for its `applies_to_taxonomy` scope only*. It MUST NOT widen scope beyond what the OPF declares.
3. Where a composed module and the OPF's own Floor conflict, **the stricter rule wins** (a Floor is a one-way ratchet; composition can only add hard lines, never relax them).

> **Open (resolve before implementing composition):** the concrete interface a module exposes — does it hand back structured clause positions, a sub-prompt, a detector set, or all three? — is **not** fixed in 0.2. This section pins the *governance contract* (pin + fail-closed + scope + stricter-wins); the wiring is deferred to one investigation pass against the actual module structure. Until then, `composes` is a declared, validated dependency the runtime records on every review for lineage, even if it does not yet alter behavior.

### 3.5 `evidence` — the compiled, cited knowledge

`evidence` is the compiled, cited record of what the corpus shows —
descriptive, never an instruction. Its shape is §3.5.4's verdict-free
per-deal precedent record.

> **History (retired 2026-10-07, issue #238).** OPF 0.1–0.3 carried a
> different evidence shape: per clause `observed_positions` and a `summary`
> (`historical_stance`, `stance_detail`, `acceptable_if`, `fallbacks`,
> `rejected`, `confidence`; v0.1's `rollup`), a `clause_library` of concepts
> for counterparty paper, and a `negotiation_trail` (formerly §3.5.1–§3.5.3).
> Those categories needed judged verdicts the consumer path no longer
> carries, and the engine no longer reads or writes them. Their text and
> schemas are in this repository's git history and in contract-opf/opf.

#### 3.5.4 The verdict-free per-deal precedent record (OPF 0.4, issue #223; carried by 0.5 and 0.6)

In `opf_version` "0.6" (`spec/playbook.schema-0.6.json`; the record is 0.4's
plus the opening evidence of §3.5.5 and, since 0.6, the signing date) judged
deviation/risk verdicts are off the consumer path (the consuming model does
the judging) and the deal is the unit of precedent, so `evidence` carries
facts only:

```jsonc
"evidence": {
  "clauses": [ {                       // one per clause type
    "id": "clause.governing_law", "taxonomy_id": "governing_law", "title": "Governing Law",
    "our_standard": { "text": "...", "source_ref": Citation,
                      "parts": [ { "part_id": "3f9a2c1d", "order": 1, "text": "..." } ] } | null,
    "prior_standards": [ PriorStandard ],   // our former forms' clause, §3.5.7 (empty when none)
    "why_changed": { "text": "...", "basis": "owner" } | null,   // the owner's note, §3.5.7
    "n_deals": 6,                      // distinct deals with a precedent for this clause
    "n_signed_standard": 2,            // distinct signed deals that signed our standard
    "n_variants": 2,                   // distinct non-standard texts signed (signed deals)
    "n_refused": 0,                    // distinct refused-ask texts
    "n_opened_standard": 4,            // distinct signed deals whose clause opened with our standard (§3.5.5)
    "n_kept_standard": 2,              // of those, deals that signed our standard
    "n_opened_prior": 3,               // distinct signed deals whose clause opened on a former form of ours (§3.5.7)
    "n_kept_prior": 2,                 // of those, deals that signed the text they opened with
    "n_moved_to_current": 1,           // of those, deals whose signed text is our current standard
    "n_changed_openings": 0            // distinct non-standard openings not signed as proposed (§3.12.2)
  } ],
  "precedent": [ {                     // one per (deal, clause type)
    "id": "prec.7377a6430df64a98",
    "taxonomy_id": "governing_law", "document_id": "beta-industries",
    "counterparty_ref": { "alias": "Counterparty-2" },        // OPTIONAL
    "paper": "ours" | "theirs" | "unknown", "paper_basis": "provenance_detection",
    "paper_confidence": 0.93 | null,
    "signed": true, "signed_on": "2025-03-14" | "2025-03" | null,   // §3.8
    "rounds": 1,
    "signed_text":  { "text": "...", "ref": Citation } | null,
    "opened_with": "standard" | "prior_standard" | "non_standard" | "absent" | null,   // §3.5.5
    "opened_with_form": "2019 form",                                // OPTIONAL: present iff prior_standard, §3.5.7
    "opening_text": { "text": "...", "ref": Citation } | null,      // §3.5.5
    "standard": false, "moved": true,
    "refused_asks": [ { "text": "...", "round": 1, "ref": Citation } ]
  } ],
  "absence": { "guard": 0.1, "records": [ AbsenceRecord ] }   // signed deals without a clause, §3.5.8
}
```

Field semantics:

- `our_standard` — the canonical template's complete language for the clause
  type: every template clause of that type, in document order, joined into one
  `text` (a lead-in sentence and the operative limbs under it are one
  standard). A form's fill-in front matter (a cover table of bracketed
  blanks with no operative sentence, ahead of the form's first operative
  clause) is no clause and is never a standard; a fill-in table after that
  point (a fee or notices schedule) is a clause like any other. A
  standard joined from several template clauses cites them all
  (`source_ref.clause_path` lists their paths, comma separated) and carries no
  `char_span`; a single-clause standard cites that clause and its span.
  `parts` (OPF 0.6, issue #243) are those template passages in order, one part
  each (a one-passage standard is one part): `text` is the parts' texts joined
  with a blank line (`"\n\n"`), `order` counts from 1, and `part_id` is the
  first 8 hex characters of `sha256` over the part's §3.5.4 grouping key (with
  `perspective.party`), so it survives a template edit elsewhere; a later part
  of the same standard with the same hash is `<hash>-2`, `<hash>-3`, ....
  A validator MUST reject a standard whose `text` is not its parts joined, or
  whose `part_id` or `order` is not the one so computed.
- `signed_text` — the deal's terminal text for the clause: the executed copy
  when `signed` is true, the last draft when it is false. `null` when the
  clause was struck before signing.
- `opened_with`, `opening_text` — what the clause opened with, a fact about
  the deal's first draft, whatever its origin and whatever paper the deal is
  on. Defined in §3.5.5. Never fabricated.
- `standard` — `signed_text` is our standard language: an EXACT match after
  normalization (whitespace collapsed; the producer's known party names
  rewritten to one neutral token; case and punctuation dropped; word order,
  negators, modals and numerals kept). Deterministic; never a similarity
  score and never judged. `false` when `signed_text` is `null`. The party
  names are producer configuration (the reference compiler's
  `provenance.our_party_aliases` and `known_entities`), which the document
  does not carry, so `standard` is a producer fact a validator does not
  recompute — and it is NOT the normalization that groups texts (the
  grouping key below).
- `rounds` — distinct negotiation rounds in which the clause changed
  (round `r` is the version transition `r → r+1`); `moved` — the clause
  changed, was struck, or carried a refused ask (always true when
  `opening_text` is non-null).
- `signed_on` — when the deal was signed (OPF 0.6, issue #247): an ISO-8601
  date (`YYYY-MM-DD`) or, when only the month is known, `YYYY-MM`; `null`
  when the deal has no detected executed copy (`signed` false) or no source
  states a date. It is the deal's `corpus.documents[].signed_on` (§3.8) and a
  validator MUST reject a record whose value differs from its document's, is
  not a real calendar date or lies outside 1980-2099, or is set on a record
  with `signed` false. A date is a **fact** read off the executed copy and
  never invented; its basis is on the document.
- `refused_asks[]` — text proposed during the negotiation and struck before
  signing; `round` is the cited draft's version ordinal − 1 (0 = the
  opening draft).
- `paper` — whose paper the deal is on, three-valued (an ambiguous
  detection is `"unknown"`, never coerced to a side). **Metadata only:** no
  count, list or rule partitions, gates or weights by paper side.
- `id` — `"prec." + sha256(canonical([agreement_type.id, document_id,
  taxonomy_id, signed_text.text or ""]))[:16]` (hex; `canonical` is §3.10's
  canonical JSON). Stable across recompiles of the same evidence.

**Grouping key** (normative). The clause counts below and §3.12.2's digest
group texts — signed variants, refused asks, changed openings — by an exact key computed from
the text and the document alone, so a validator or an independent port
recomputes it with nothing but the document. It differs from `standard`'s
normalization only in its party names: `standard` neutralizes the
producer's configured names, which the document does not carry (a key built
from them could not be recomputed), while the key neutralizes only the party
identifiers the document itself carries — the counterparty's entity-registry
alias and `perspective.party` — each to its own token. In order (steps 2–4
are `standard`'s normalization with `perspective.party` as its only party
name):

1. Every entity-registry alias — `Counterparty-` followed by one or more
   ASCII digits, matched case-insensitively, with no word character (a
   Unicode alphanumeric character or underscore) immediately before or
   after — is replaced by the token `counterparty`. Each deal's
   counterparty carries its own alias in the clause text, so without this
   the same words signed with two counterparties would be two variants.
2. Every whitespace run is collapsed to one space.
3. When the document has a non-blank `perspective.party` (§3.1), every
   occurrence of it (its own whitespace runs collapsed to one space;
   matched case-insensitively, with no word character immediately before
   or after) is replaced by the token `party`. No other party name is
   applied.
4. The text is lowercased, every character that is neither a word
   character nor whitespace is replaced by a space, whitespace runs are
   collapsed to one space, and the ends are trimmed.

Two texts group together only when their keys are equal: word order,
negators, modals and numerals all survive, and our party (`party`) and the
counterparty (`counterparty`) stay distinct tokens, so the same words with
the parties' places swapped are a separate group. The reference
implementation is `precedent.normalize_variant_text`; 0.6 conformance
vector 004 pins it. The same key decides whether an opening is distinct
(§3.5.5).

Normative rules (a conformant validator MUST enforce each; the schema
cannot express them):

- Every `precedent[].id` is unique and equals its recomputed value.
- At most one precedent per (`document_id`, `taxonomy_id`) — the deal is the
  unit of precedent.
- Every `precedent[].taxonomy_id` names an `evidence.clauses[]` entry; every
  `precedent[].document_id` resolves to `corpus.documents[]`, and `signed`
  agrees with that document's `signed_version` (non-null ⇔ `signed`) when
  recorded. `standard: true` requires a non-null `signed_text`.
- Each clause's `n_deals` / `n_signed_standard` / `n_variants` / `n_refused` /
  `n_opened_standard` / `n_kept_standard` / `n_opened_prior` / `n_kept_prior` /
  `n_moved_to_current` / `n_changed_openings`
  equal what `precedent` implies (grouping texts by the grouping key above —
  not by `standard`'s normalization, whose party names are not in the
  document). Every count is distinct deals or distinct texts — never rows.
- Every citation in `evidence` resolves (§4).
- The opening-evidence rules of §3.5.5 and the prior-standards rules of §3.5.7.

No stance, band, risk or deviation verdict appears anywhere in a 0.6
`evidence` or `digest`. A producer that ran a judged pass MAY carry its
verdicts under the root vendor extension `x_judgments` (`[{precedent_id,
...}]`, §10.1) — never inside `evidence.precedent`, whose records are
closed (`additionalProperties: false`, no `x_*`). The reference producer
(`playbook-engine`) runs no such pass and emits none.

#### 3.5.5 OPF 0.6 — what each clause opened with (issue #233)

A signed variant means much more when the record says where the deal
started: "we opened with our standard and signed X" makes X a concession on
record, and "they opened with Y and we signed our standard" means Y did not
survive. 0.5 and 0.6 therefore record, for every precedent, what the clause opened
with — as a **fact**, never a meaning. Whether an opening was held,
conceded or moved to standard is derived by the consumer (or the digest) from
`opened_with`, `opening_text` and `signed_text`; the record classifies none of
it. Our paper and third-party paper yield the same kind of evidence: the
opening on counterparty paper is almost never our standard, so a rule gated
on origin would starve it. Paper side stays metadata.

Definitions (normative):

- **First draft.** The first version in the deal's negotiation order (the
  net diff's before side). For a single-version signed deal it is the signed
  copy itself.
- **First-draft text of a clause type.** Every first-draft node bound into an
  aligned row of that `taxonomy_id` — removed rows, and rows whose text still
  occurs in the signed copy, included — joined with a newline in the order of
  the nodes' `char_span` (path order when spans are absent). Its `ref` cites
  the first of those nodes. The row's `taxonomy_id` is the alignment's, the
  same one the terminal record uses.
- **`opened_with`**, required, one of:
  - `"standard"` — the first-draft text is our standard language. This is the
    check `standard` uses (an exact match after normalization against the
    whole template clause, with the producer's party names).
  - `"prior_standard"` (issues #249, #250) — it is not our current standard but IS
    the text of one of OUR former forms for the clause (§3.5.7), by the same
    exact check, against that form's whole clause (for a RECONSTRUCTED form: its
    grouping key (§3.5.4) is that of the clause's recurring first-draft wording
    the form was built from, §3.5.7, whatever the deal's paper side). Its origin is ours, never
    the counterparty's. The current standard is tested first and wins when
    both match. Which form a deal opened on is a fact of the deal, not of each
    clause: when several former forms match a clause, the form that the most
    of the deal's first-draft clause types match names the opening
    (`opened_with_form`), then the first listed in `baseline.prior_templates`;
    so forms that read alike in one clause never split a deal across two forms.
  - `"non_standard"` — it is present but neither, including when the clause
    type has no template clause.
  - `"absent"` — the first draft has no node bound to the clause type, but a
    later draft or the signed copy does: the clause was added during the
    negotiation. A first-draft clause that the producer left unclassified
    reads as absent too: `"absent"` is a fact about the clause type's
    aligned nodes, not about the document text.
  - `null` — not determined: the deal has no detected executed copy
    (`signed` false, an unsigned deal's version order is not anchored), or
    the producer predates this field. `null` is never `"absent"`.
- **`opened_with_form`** — the `label` of the former form `opened_with:
  "prior_standard"` matched. Present exactly when `opened_with` is
  `"prior_standard"`, and always a `baseline.prior_templates[].label`.
- **`opening_text`** — `{text, ref}` of the first-draft text. Recorded when
  `opened_with` is `"standard"`, `"prior_standard"` or `"non_standard"` AND either `signed_text`
  is `null` (the clause was struck, or relocated away) or the §3.5.4 grouping
  key of the opening differs from that of `signed_text`. Otherwise `null`: a
  clause that opened with its signed text, or whose only edit is case,
  whitespace or punctuation, has no distinct opening. `null` still means
  "not recorded / not distinct", never "opened with the signed text" when
  `opened_with` is `null`.

Normative rules (a conformant validator MUST enforce each):

- `signed` false ⇒ `opened_with` is `null` and `opening_text` is `null`.
- `opened_with` is `"absent"` or `null` ⇒ `opening_text` is `null`.
- `opened_with_form` is present ⇔ `opened_with` is `"prior_standard"`.
- `opening_text` non-null ⇒ its grouping key differs from that of
  `signed_text` (or `signed_text` is `null`), and `moved` is `true`.

A struck opening needs no origin claim: a clause type's first-draft text that
was struck before signing is recorded as `opening_text` with `signed_text`
`null` even when the producer cannot tell whether it was our language or the
counterparty's. It is not claimed as a refused ask, which keeps its
origin-gated definition (`refused_asks[]`, §3.5.4). A first-draft node whose
text is relocated, not struck (it occurs verbatim in the signed copy under
another clause), produces no opening and no struck record.

#### 3.5.6 OPF 0.6 — `vs_standard`, the equivalence judgment (issues #240, #243)

The exact-match `standard` fact answers only "did they sign our words
unchanged". On counterparty paper that is almost always false even where the
concept is the same, and grouping signed variants by exact text turns one
concept phrased five ways into five one-deal variants. 0.5 and 0.6 therefore let a
producer judge each DISTINCT text once against our standard and carry the
answer as an index the consumer's model can check against the cited text. It
is an index, never an instruction, and it is the one place a judged verdict
appears in `evidence` (owner decision (c) of issue #227, narrowed for this
judgment only; every other fact in the record stays verdict-free).

The judgment is made **part by part** (issue #243). A single whole-text label
cannot say that a text keeps some of our passages, changes others, drops one
and adds something, which is the usual case; it says "can't rank", and the
consumer has to redo the comparison on every review. So the judge says, for
EVERY part of our standard (§3.5, `our_standard.parts`), whether the text
keeps it and how it compares, lists what the text adds, and the label is
DERIVED from those facts.

`vs_standard` is an OPTIONAL key (a producer SHOULD always write it; absent
is read as `null`) on `signed_text`, `opening_text` and each `refused_asks[]`
entry of a precedent record, and on each `prior_standards[]` entry of a clause
(§3.5.7):

```jsonc
"vs_standard": {
  "parts": [                                  // every part of our standard, once, in order
    { "part_id": "3f9a2c1d", "status": "present",
      "relation": "equivalent" | "more_protective" | "less_protective",
      "reason": "<one sentence; only when relation is not equivalent>" },
    { "part_id": "77b0e4aa", "status": "omitted",
      "effect": "more_protective" | "less_protective" | "neutral",
      "reason": "<one sentence; only when effect is not neutral>" }
  ],
  "additions": [ { "summary": "<one sentence>",
                   "effect": "more_protective" | "less_protective" | "neutral" } ],
  "label":  "equivalent" | "more_protective" | "less_protective" | "mixed" | "different_concept",
  "reason": "<one sentence: the single most important difference; no deal names>",
  "basis":  "judge" | "agent" | "owner",
  "check":  { "agreed": true, "label_agreed": true, "parts_agreed": true,
              "by": "<checker model id>", "adjudicated": false } | null,
  "owner_label": { "label": "...", "reason": "..." }       // OPTIONAL, basis "owner" only
} | null
```

- Every direction is relative to `perspective.party`. A `present` part's
  `relation` compares the text's version of that part with ours; an
  `omitted` part's `effect` is what dropping it does (dropping an obligation
  that falls on the party can be `more_protective`; dropping a recital is
  `neutral`). An addition is something the text adds that no part covers.
- **The derived label.** `label` is a pure function of `parts` and
  `additions`: `different_concept` when no part is `present`; `equivalent`
  when at least one part is present, every present part is `equivalent` and
  every omission and addition is `neutral`; `more_protective` /
  `less_protective` when every non-equivalent, non-neutral effect points that
  way; `mixed` when they point both ways. A text that only drops a neutral
  recital is therefore still rankable. An owner MAY override the label
  (`owner_label`, with the owner's reason); `label` is then the owner's and
  the parts stay as judged. Every model-facing projection that carries the
  parts beside the label (the digest, §3.12.2; the dossiers, §3.12.3) marks
  such a label `label_by: "owner"` and carries the owner's reason, so the
  consumer never reads a label beside a reason that contradicts it. An owner
  MAY instead change the parts (or additions), and the label is re-derived
  from them; the judgment's `reason` is then the owner's, or, when the owner
  gives none, it is kept only if the part map and addition tokens it was
  written for are unchanged and is otherwise replaced by a fallback (it
  explained the old map and, perhaps, the old label).
- `basis` names who produced the judgment. `owner` is an after-the-fact
  correction and wins over any agent or check answer; it is never checked.
- `check` is the independent blind check: a checker that never saw the
  judgment answers the same structured question. `label_agreed`: it derived
  the same label; `parts_agreed`: it found the same set of `(part_id, status,
  direction)` over the parts that are not equivalent or not a neutral
  omission; `agreed` is both. `agreed: false, adjudicated: false`: the
  disagreement awaits adjudication (the judgment shown is still the draft).
  `agreed: false, adjudicated: true`: a fresh agent saw both judgments and its
  answer is the judgment. `null`: not checked yet. A consumer reads an
  unchecked or disputed judgment as weaker evidence than an agreed or
  adjudicated one, and never blocks on it.
- **`settled`** (issue #248) is how settled a judgment is, a pure function of
  `basis` and `check` that the digest and the dossiers carry beside the label
  (it is never stored on the evidence): `owner` when `basis` is `owner`
  (an owner's correction wins over any check), else `adjudicated` when
  `check.adjudicated` is `true` (the blind check disagreed and a fresh agent
  decided: the label was contested), `agreed` when `check.agreed` is `true`,
  and `unchecked` otherwise (no `check`, or a disagreement still awaiting
  adjudication). A conformant validator MUST recompute it with the digest
  and the dossiers (§3.12.2, §3.12.3), which MUST equal what the document
  derives.
- `null` means "not yet judged", never a guess. It is the only value for: a
  signed text whose `standard` fact is true (an exact match is equivalent by
  definition); any text of a clause whose `our_standard` is `null` (emergent
  mode: nothing to compare with); an opening whose `opened_with` is
  `"standard"` or `"prior_standard"` (the latter is our own former form, which
  its `prior_standards[]` entry judges); a former form's clause that reads,
  under the grouping key, as today's standard. A refused ask, a non-standard
  opening, a non-standard signed text and each former form's clause that
  differs from today's are judged.
- **Cache key (the identity of "the same text").** A judgment is shared by
  every deal, paper side and role carrying the same text against the same
  parts: `sha256` over the JSON array `[agreement_type.id, taxonomy_id,
  perspective.party, K(text), [K(part.text) for each part of our_standard,
  in order]]`, where `K` is the §3.5.4 grouping key. The paper side, the
  document id, the role and the opening draft are not part of it. What a
  judge is shown carries none of them either: the taxonomy id and title,
  `perspective.party`, the numbered parts of our standard (`{n, part_id,
  text}`), the candidate text and its roles.

Normative rules (a conformant validator MUST enforce each):

- A non-null `vs_standard` MUST NOT appear on a signed text whose `standard`
  is `true`, nor on a text of a clause whose `our_standard` is `null`.
- `parts` names every part of the clause's `our_standard` exactly once, in
  order, and no other `part_id`; a `present` part has a `relation`.
- `label` equals the derived label of `parts` and `additions`, unless
  `owner_label` is set (then it equals `owner_label.label`); `owner_label`
  requires `basis` `owner`.
- `check.adjudicated` `true` ⇒ `check.agreed` is `false`; `check.agreed` is
  `label_agreed and parts_agreed`.
- When the producing run's verdict store is available, every non-null
  `vs_standard` MUST be the judgment stored under its cache key (a label or a
  part the run never produced is invented).

The reference compiler drafts verdicts with the `playbook-from-corpus` skill
(`playbook judge` queues one `equivalence` item per distinct eligible text
once a template is configured), checks every draft with a separate
`claude-opus-5-5` / `xhigh` agent (`playbook judge --check equivalence`,
`playbook judge-apply --check`), writes the judgments at `playbook project`
and reports drafted / checked / agreed (on the label and on the parts,
separately) / adjudicated / unchecked counts in `playbook scorecard`. Digest
4 (issue #234) uses the labels to collapse equivalent variants into one entry
and carries every other judged text's compact part map (§3.12.2); the critic
dossiers carry the part reasons (§3.12.3).

#### 3.5.7 OPF 0.6 — our former forms: prior standards (issue #249)

Past deals mostly opened on OLDER versions of our own form. Read against
today's standard alone, a clause that opened on last year's indemnity is
`non_standard`, and our own former wording reads as the counterparty's. When
the producer is given those forms (`baseline.prior_templates`, §3.2) it
records the clause as ours and gives the consumer a factual history of the
standard. Only facts are recorded; **no reason is ever claimed**: the only
reason a change is ever shown is the owner's own `why_changed`.

**Reconstructed forms (issue #250).** When the producer is given NO former form
(a supplied form always wins), it rebuilds them from the corpus, because our
former wording is what we SENT: the first draft of a signed deal on our paper
(`paper: "ours"`) that is not today's standard. Per clause type with an
`our_standard`, those first drafts (the distinct opening, else the signed text)
are clustered by a deterministic method: the §3.5.4 grouping key split into
words; two wordings are near-identical when the Jaccard similarity of their word
3-gram sets is at least 0.85 AND they hold the same numbers (digits or spelled
out) and the same count of negators; wordings are visited most-deals-first and
each joins the first cluster whose seed it matches. A cluster is a
**reconstructed clause** when it is the first draft of at least `MIN_DEALS` (3)
distinct deals, its **medoid** (the member wording closest to all the others,
weighted by deals) is the first draft of at least half of them, and the medoid
differs from today's standard. Clause-level reconstructions that co-occur (the
smaller one's deals at least 60% inside the other's, their signing-date spans
overlapping or undated, never two of one clause type) are one **form**; the rest
stay a form per clause. The form is `basis: "reconstructed"` in
`baseline.prior_templates` with its label from what is known of it (`reconstructed
2021-03 to 2022-11`; never a year the data does not give, and no claim of why
it changed), `n_deals` and the signing-month span; each of its clauses is a
`prior_standards[]` entry whose text is the medoid and whose `source_ref` is the
ordinary citation of the medoid where it sits in a corpus document (it has no
file of its own); every signed deal whose first draft of the clause has the medoid's §3.5.4 grouping
key reads `opened_with: "prior_standard"` with `opened_with_form` the form, on
whatever paper side: the paper side chose the candidates the form was built
from, never which deals opened on it. It feeds everything a supplied form feeds (the
`vs_standard` judgment against today's standard, the counts below, the digest's
`standard_history` with `basis: "reconstructed"`), including the removed-row
rule below: a struck clause whose text is the form clause's wording, or one
contiguous piece of it, is no refused ask and is counted under
`removed_prior_standard`.

A recurring first draft on our paper might still be a counterparty's wording, so
each cluster is first put to the `prior_form_origin` judge kind (see the engine's
`REFERENCE.md`): "is this really OUR former language?", asked of the medoid
against today's standard, with no deal, counterparty or paper side in the payload
or the key. It is drafted by an agent, blind-checked by a separate pinned agent,
and adjudicated on disagreement. **Only confirmed clusters become prior
standards**; a cluster with no verdict, an unchecked draft or a disputed one is
pending and changes nothing in the playbook.

**Prior standards.** Each form is segmented and classified exactly like the
current template (same producer, same front-matter rule), so a clause type's
text in a form is comparable, part for part, with today's standard. Per clause
type and form with text for it, `evidence.clauses[].prior_standards[]` holds a
`PriorStandard`:

```jsonc
{ "form_label": "2019 form",                  // a baseline.prior_templates[].label
  "text": "...",                               // the parts joined with a blank line
  "parts": [ { "part_id": "3f9a2c1d", "order": 1, "text": "..." } ],   // as our_standard.parts
  "source_ref": Citation,                      // the clause in the FORM's own bytes: document_id
                                               // "prior_template:2019 form", version "prior_template" (§4)
  "vs_standard": VsStandard | null }           // this clause against TODAY'S standard, §3.5.6
```

`parts` follow `our_standard.parts` exactly (a form's classified, non-empty,
non-front-matter passages of the type in form order; `part_id` the content
hash of the part under §3.5.4's grouping key, so a passage unchanged since the
form has the same id in both). A clause has at most one entry per form. A
clause type the form has no standard for has no entry for it.

`source_ref` cites the clause in the form's own bytes, never in today's
template: its `document_id` is `prior_template:` followed by the form's label,
its `version` is `prior_template`, and it resolves through that form's
`baseline.prior_templates[].sha256` (§4.1); its `clause_path` and `char_span`
are the form's own coordinates. (`document_id` `"template"` always names
today's form, so a former form cited as `"template"` would resolve to today's
wording.)

**Judging each form once.** A prior standard's `vs_standard` is the §3.5.6
judgment of the form's clause against TODAY's standard, part by part: the
parts it names are today's `our_standard.parts`, the cache key is the §3.5.6
key (so the same words signed in a deal share one judgment with the form's
clause), and the subject's role is `prior_standard`. A clause that reads, under
the grouping key, exactly as today's standard is never judged and its
`vs_standard` MUST be `null`; so is one on a clause with no `our_standard`.

**Openings and counts.** A deal that opened on a former form is recorded as
`opened_with: "prior_standard"` with `opened_with_form` (§3.5.5), and its
origin is ours: it is never a changed opening (§3.12.2), and a clause of ours
struck before signing is no refused ask (the producer counts it under
`corpus.stats.dropped_observations`, `removed_prior_standard`). Held and
conceded are computed against the form we actually sent, over distinct signed
deals:

- `n_opened_prior` — `opened_with` is `"prior_standard"`;
- `n_kept_prior` — of those, the deal signed the text it opened with (a
  non-null `signed_text` and a null `opening_text`);
- `n_moved_to_current` — of those, `standard` is `true`: the deal moved from
  our former form to today's.

`n_opened_prior − n_kept_prior` is what was conceded; it is derived, not
recorded.

**`why_changed`.** An OPTIONAL note per clause, `{text, basis: "owner"}`,
written only by the owner (the Review tab's "Why we changed this" field, saved
as an `overrides.json` entry, `basis: owner`); `null` otherwise. It is never
required and the engine never writes one.

Normative rules (a conformant validator MUST enforce each):

- `baseline.prior_templates[]` labels are unique.
- Every `prior_standards[].form_label` and every `opened_with_form` is a
  `baseline.prior_templates[].label`; a clause has at most one prior standard
  per form.
- A SUPPLIED form's prior standard's `source_ref` cites its own form:
  `document_id` `prior_template:<form_label>` and `version` `prior_template`,
  and that form carries its `sha256` (§4).
- Supplied and reconstructed forms never appear together in one document.
- A RECONSTRUCTED form has no `sha256`, `in_use_from` or `in_use_until`; its
  prior standards' `source_ref` is an ordinary corpus citation (never
  `prior_template` or `template`); its `n_deals` is the number of distinct deals
  whose precedent records name it in `opened_with_form`, and its `first_signed` /
  `last_signed` the earliest and latest signing month among them (absent when
  none is dated); and no `refused_asks[].text` of a clause type has the §3.5.4
  grouping key of that form's prior standard of the clause type (its `text` or
  one of its `parts`): our own former wording struck before signing is no
  refused ask.
- `prior_standards[].text` is its `parts` joined in order, `order` counts from
  1, and each `part_id` is the part's recomputed content hash (as
  `our_standard`).
- A prior standard's `vs_standard` follows §3.5.6's rules, and is `null` for a
  clause that reads as today's standard.

#### 3.5.8 OPF 0.6 — verified absence: "we signed N deals without this clause" (issue #251)

A new contract may lack a clause type entirely, and the consumer then needs to
know how often we have signed without it. A deal with no precedent record for a
clause is not enough: the clause may be in the deal's signed copy and never have
been classified (a missed classification must not read as an absence). So
`evidence.absence` records, for **every signed deal and clause type with no
signed text**, which of two **facts** holds. Neither is a verdict.

```jsonc
"absence": {
  "guard": 0.1,                         // the score at or above which a node keeps a deal out of the verified count
  "records": [ {                        // sorted by (document_id, taxonomy_id)
    "id": "abs.0f3a9c2b17d4e655",       // "abs." + first 16 hex of sha256(canonical([agreement_type.id, document_id, taxonomy_id]))
    "taxonomy_id": "assignment", "document_id": "deal-b",
    "status": "absent_verified" | "possibly_present",
    "basis": "scored" | "unverifiable",
    "n_checked": 24,                    // signed-text nodes examined
    "candidates": [ { "ref": Citation, "score": 0.222, "filed_under": "term" | null } ]   // at most three, best first; [] when verified
  } ]
}
```

- **Which pairs.** The signed deals are the in-scope `corpus.documents[]` with a
  non-null `signed_version`. A pair (deal, clause type) has a record exactly when
  the deal has no `evidence.precedent` record for the clause with a non-null
  `signed_text`. A clause struck before signing therefore has a record (the deal
  did sign without it); an unsigned deal has none. The records are complete: a
  validator MUST reject a document missing the record of any such pair, one for a
  pair that has a signed text, a duplicate, an id that is not recomputed from
  `(agreement_type.id, document_id, taxonomy_id)`, and a `candidates[].ref` that
  does not resolve (§4) or cites another deal.
- **The nodes.** Every clause node of the deal's executed copy except those filed
  under this very clause type: the ones the classifier left unclassified AND the
  ones it filed under another clause type (a clause stated inside a block filed
  under a neighbouring type is as good as unclassified for "is it in this deal?").
  A node that is no clause (a bare section number, a signature block, a running
  header; see `x_non_clause`) is not examined. A node filed under this very clause
  type that was too short to reach `evidence.precedent` scores 1.0: it is the
  classifier's own finding of the clause. (A node filed under this very clause type
  that is long enough to be precedent gives the deal a `signed_text` and so no
  record.) `n_checked` counts the nodes examined: every node not filed under
  this clause type, and each too-short one filed under it. A deal whose executed
  copy has no node to examine is `unverifiable` with `n_checked` 0.

- **The score** of a node against a clause type is the content-word Jaccard
  similarity that the content-similarity classifier uses (stop words dropped,
  numerals kept), taken as the best over the node as a whole and each of its
  paragraphs, sentences and list items (a big node that holds the clause among
  others would otherwise dilute it), and over every reference text of the clause
  type: the parts and the whole of our standard and of each former form's
  clause, and every text any deal signed, opened with or had refused for the
  clause. It is rounded to three places and compared with `guard` after rounding.
  `guard` is well below the assign threshold (0.25): assigning a node to a clause
  needs confidence, declining to verify an absence needs only plausibility.
- **Status.** `absent_verified` — `basis` is `scored` and no node scores at or
  above `guard` (`candidates` is empty). `possibly_present` — otherwise: at least
  one node does (`candidates`, with the cited node, its score and `filed_under`, the clause type the
  classifier filed it under or `null`), or
  `basis: "unverifiable"`, nothing could be checked (the deal's executed copy
  produced no readable text or has 1,000 characters or more of passages that
  extracted as junk, such as columns of digits, where the clause may sit; or the
  clause type has no reference text anywhere).
  A validator MUST reject a `status` the `basis` and `candidates` do not imply,
  candidates not ordered best score first, and a score below `guard`.
- **Conservative by design.** A neighbouring clause's shared vocabulary can score
  at or above `guard` against a clause it does not state, which keeps a deal out
  of the verified count: the error is always towards `possibly_present`, never
  towards a false absence. `absent_verified` therefore means "no passage of the
  signed copy resembles the clause", a lower bound on the deals that signed
  without it.
- **Pure and paper-blind.** Both facts are a pure function of the store; paper
  side takes no part in them.
- **The spot-check.** The producer measures its guard with the `absence_check`
  judge kind ("is clause type X really absent from this signed text?"), asked of
  a deterministic sample of the verified records (all of them when there are 40 or
  fewer, else the 40 with the lowest payload hash); the answer gates nothing and
  is not recorded in the document. A confirmed-present item means the guard is
  too weak: it is tightened until the checked sample has none.

### 3.6 `posture` (NEW) — negotiation intent as generated prose

The Posture is the "smart, system-prompt-style directions" layer: a prose block that tells the review engine *how to negotiate this agreement type*, generated by the compiler from a short interview (§7) and grounded in the Evidence.

> **What "grounded" and "generated" mean for the reference producer:** this spec does not mandate free-form LLM drafting. The reference producer (`playbook-engine`) assembles `system_prompt` **deterministically** — one governed prose sentence per interview answer, concatenated in canonical order, not model-written. "Grounded in the Evidence" here means the interview options presented to the author are themselves computed from the compiled Evidence section, and `generation.grounded_in` records the Evidence digest the interview was run against — not that an LLM read Evidence text while composing prose. A conformant producer MAY instead offer a richer, LLM-authored draft; if it does, that draft is still subject to the same versioning and human-approval discipline as any Posture edit (§8).

```jsonc
"posture": {
  "system_prompt": "This is a generally low-risk agreement type; default toward ACCEPT. We typically go two negotiation rounds before escalating. Hold firm on the liability cap and on declining indemnification (see Floor). Term, notice periods, and renewal mechanics are flexible to close. Write rationale tersely for a GC audience…",
  "version": 1,                            // governed counter (NEW, issue #156) — bumped when a re-run of the interview actually changes the content (issue #132: a byte-identical re-run is a no-op)
  "generation": {                          // provenance: how the prompt was produced
    "generated_by": "playbook-engine vX.Y.Z",
    "generated_at": "ISO-8601 (supplied by caller)",
    "interview": [
      { "q": "rounds", "question": "How many rounds…", "answer": "Usually 2." },
      { "q": "leverage", "question": "Default leverage posture?", "answer": "Collaborative; we often want the deal." }
      // … see §7 for the canonical question set
    ],
    "grounded_in": "evidence@<digest>"     // the Evidence state the draft was written against
  }
}
```

**Normative rules:**
1. `system_prompt` is **legal behavior**. The OPF carries the *initial, compiler-generated* version (the genesis record). A consumer that lets a human edit the Posture MUST treat each edit as a governed version bump (§8) — versioned, diffed, re-approved, rollback-able — never a free-text field that silently changes production behavior. The producer itself already versions its own genesis Posture: `version` starts at `1` and is incremented by 1 each time the compiler's interview (§7) is re-run against an existing Posture **with different answers** (issue #156) — this is the OPF-side half of "governed"; a consumer's edit-time versioning (§8) picks up from there. A re-run whose answers are byte-identical to the existing Posture is a no-op: `version` is left untouched rather than bumped for a revision that never happened (issue #132) — a producer MUST NOT advance `version` on a re-run it can determine made no actual change.
2. The `generation.interview` record is **provenance**: it lets an auditor see *why* the Posture says what it says. It MUST be retained.
3. The Posture MUST NOT restate or contradict the Floor. The Floor is the authority on hard lines; the Posture may *reference* it ("hold firm on X, see Floor") but a Floor invariant binds regardless of Posture text (§5). Per issue #156's decided direction (2026-07-10): a Posture that appears to soften language around a Floor-protected concept is a **SHOULD-warn** (judgment-first, non-blocking) that a conformant validator surfaces for human review — not a hard validation error.
4. The Posture is **soft** at runtime (§5): it shapes the model's judgment; it is not a gate and cannot, by itself, force or suppress a decision the way a Floor rule does.

### 3.7 `floor` (NEW) — the hard lines

The Floor is the small, explicit set of things that must **never** slip — un-overridable by the model under review and by the Posture. It absorbs what an earlier design split out as a separate "review-overlay"; here it is a section of the one knowledge artifact, authored by the legal owner.

A Floor entry is a **natural-language invariant**: one checkable statement a judge can evaluate against a clause in isolation. There is no lexical detector grammar — an earlier draft's term-matching design was superseded (2026-07-09, #145) because real hard lines ("never accept uncapped liability") are semantic, not lexical.

```jsonc
"floor": {
  "invariants": [
    {
      "id": "no-uncapped-liability",
      "statement": "Never accept uncapped liability.",
      "rationale": "Uncapped exposure is categorically unacceptable regardless of deal value."
    },
    {
      "id": "no-one-way-indemnity",
      "statement": "Never give one-way indemnification flowing only from us.",
      "rationale": "We do not give indemnification on this paper without reciprocity."
    }
  ]
}
```

An invariant MAY carry vendor-namespace keys (§10.1) that feed the hard-rule manifest (§3.12.3): `x_taxonomy_id` (the clause type it is about), `x_required_presence` (boolean), `x_condition` (a predicate spec or `"judged"`) and `x_permissible_proof` (list of strings). A validator rejects a malformed one. `playbook floor sign --clause ... --requires-presence --condition ... --proof ...` writes them.

**Normative rules:**
1. **Evaluation is judgment; coverage and consequence are deterministic.** A dedicated Floor judge — separate from the model doing the review — evaluates each invariant against the clause under review and returns `clear`, `violation`, or `needs_review`. The consumer's deterministic obligations are the **coverage gate** (every invariant present MUST be evaluated and logged on every review; an unevaluated invariant fails the run, fail-closed) and the **consequence rule** (a `violation` verdict forces the negotiation-unacceptable outcome; `needs_review` forces human escalation; the model under review and the Posture can override neither).
2. One Floor, both paper contexts: the same invariants are judged whether reviewing our paper's diff or the counterparty paper's extracted clauses — not two modes.
3. The invariants are OPTIONAL as a section (a corpus-only compile ships zero invariants) but never fabricated: the engine MUST NOT derive active invariants without sign-off.
4. The compiler MAY **propose** Floor candidates (every `outcome: proposed_then_reversed` in the Evidence is a candidate hard line). The legal owner finalizes and signs. A compiler MUST NOT auto-promote a candidate to an active invariant without sign-off. This bars promoting a *compiler-derived* candidate — a machine inference read off the Evidence — without an explicit accept decision; it does not bar recording a *human-authored* statement (e.g. the Posture interview's `sacred_clauses` answer, OPF §7) as an active invariant directly, since the human act of writing it is itself the sign-off, not an auto-promotion of a machine inference.

#### 3.7.1 Floor admission test (keep the Floor minimal)

The Floor's danger is that it slowly grows back into the rigid per-clause playbook v0.2 set out to retire. A rule belongs in the Floor **only if both** hold:

> **(a) Categorical** — crossing it is unacceptable *regardless of deal value, leverage, or round* (no "it depends"); **and**
> **(b) Statable as a single checkable invariant** — one sentence a judge can evaluate against a clause in isolation, without deal context or cross-clause reasoning.

Anything that fails (a) is **Posture** (intent the model weighs). Anything that fails (b) is **Evidence** — cited material the model reasons over in context (e.g. "payment terms are usually negotiable but watch the offsets" is guidance, not an invariant). A conformant producer SHOULD warn when a Floor is large relative to the taxonomy (a smell that prescription is leaking back into the Floor).

### 3.8 `corpus` (audit trail + content addresses)
```jsonc
{
  "documents": [
    { "document_id": "…", "title": "…", "provenance": "our_paper",
      "in_scope": true, "scope_rationale": "…", "scope_confidence": 0.93,
      "versions": 5, "signed_version": 5, "version_order_basis": "edit_distance_chain+signed_anchor",
      "signed_on": "2025-03-14", "signed_on_basis": "signature_block",   // OPF 0.6: both null when unknown
      "version_files": [                         // per-version content addresses (§4.1)
        { "version": 1, "sha256": "sha256:…", "media_type": "application/pdf" },
        { "version": 2, "sha256": "sha256:…", "media_type": "application/pdf",
          "source_uri": "dms://…" }              // OPTIONAL; private profiles only
      ] }
  ],
  "stats": { "documents_total": 12, "documents_in_scope": 10, "versions_total": 37 },
  "snapshot": {                                  // names the exact corpus state compiled from
    "manifest_hash": "sha256:…"                  // sha256 of canonical JSON of sorted
  }                                              //   [(document_id, version, sha256), …]
}
```
Out-of-scope documents MUST be retained here with `in_scope: false` and a `scope_rationale` — never silently dropped.

- `signed_on` / `signed_on_basis` (OPF 0.6, issue #247) — when the deal was
  signed, and where that was read. `signed_on` is a real `YYYY-MM-DD` date or a
  `YYYY-MM` month in 1980-2099, or `null`; `signed_on_basis` is one of
  `document_text`, `signature_block`, `filename`, `file_metadata` and is
  present exactly when `signed_on` is non-null. The bases are tried in that
  order of trust: the date the document states for itself in its preamble
  ("dated as of", "made and entered into on", "Effective Date", "this 3rd day
  of March, 2021"); a date in its signature block (the latest, when
  signatories dated their signatures differently); a date in the executed
  copy's file name; the executed copy's own authorship timestamp (DOCX core
  properties, PDF info dictionary). When the text says nothing the reference
  compiler asks the `deal_date` judge to read the date passages (it names which
  of the two text sources it used and quotes the words, which must occur in the
  passages it was shown); a source that finds two different dates, a numeric
  date that reads two ways (`03/04/2021`) or an impossible one yields nothing.
  When the preamble states only a month and the signature block
  gives a day in that same month, the day is recorded, with the block as its
  basis. Only a deal with a detected executed copy has a date: `signed_version`
  null means `signed_on` null. A date is never guessed. Both keys are REQUIRED
  on every in-scope document (`null` / `null` when the date is unknown, so a
  producer that never recorded dates cannot be mistaken for one that found none)
  and absent on an out-of-scope document.
- `version_files[].sha256` addresses the **original source file bytes**
  (the staged input, pre-extraction), keyed by the same inferred ordinal
  citations use. One entry per mined version; failed-ingest versions stay
  visible in the producer's ingest record instead.
- **Publication rule:** hashes of confidential files leak nothing, so
  `version_files` (and `snapshot`) are publication-safe. `source_uri` — a
  path/URI into someone's DMS — is NOT, so a producer preparing a
  document for release strips it.
- `snapshot.manifest_hash` is the OPF-side analogue of a consumer's
  `corpus_snapshot_version`: one value naming the corpus state, stable
  across identical recompiles.

### 3.9 `compiler`
```jsonc
{ "name": "playbook-engine", "version": "x.y.z", "run_id": "…", "generated_at": "ISO-8601 (supplied by caller)" }
```

### 3.10 `identity` (NEW) — content hash + section digests

```jsonc
{
  "id": "…",             // OPTIONAL — producer-assigned playbook identifier
  "version": "…",        // OPTIONAL — producer-assigned version label
  "supersedes": "…",     // OPTIONAL — the playbook this one supersedes
  "content_hash": "sha256:…",
  "section_digests": { "evidence": "sha256:…", "posture": "sha256:…", "floor": "sha256:…" }
}
```

Gives the playbook artifact identity: a canonical serialization, a content
hash, and per-section digests, so a consumer can record which exact playbook
governed which document and lineage is reconstructible end to end (§8).

- **Canonical form** (normative): the JSON value with object keys sorted
  recursively, no insignificant whitespace, UTF-8. Array order is untouched
  (semantic). See `playbook_engine/canonicalize.py` for the reference
  implementation.
- **`content_hash`** — `sha256:` + hex digest of the canonical form of the
  *whole document*, excluding two things so the hash is neither
  self-referential nor perturbed by non-content run metadata: the
  `identity` object itself (it is where `content_hash` is written), and
  `compiler.generated_at` / `compiler.run_id` (wall-clock/run-id, not
  content). Two compiles of byte-identical corpus content hash identically
  regardless of when or under what run they were produced.
- **`section_digests`** — `sha256:` + hex digest of each of `evidence` /
  `posture` / `floor`'s own canonical bytes, computed
  independently of the rest of the document. This is the `<digest>`
  referenced by `posture.generation.grounded_in: "evidence@<digest>"` (§7)
  and by the lineage fields a consumer must record (§8).
- **`id` / `version` / `supersedes`** are producer-assigned lineage
  metadata — like `compiler.run_id`, the engine cannot derive them from the
  corpus, so they are recorded only when a caller supplies them and are
  never fabricated. They deliberately do NOT participate in `content_hash`:
  identical content compiled twice under a new version/supersedes label
  still hashes identically (mirrors excluding `compiler.run_id`/
  `generated_at`).
- `identity` is OPTIONAL at the top level (not every producer populates it),
  but when present `content_hash` and `section_digests` are both required —
  a partial identity block is not conformant.

### 3.11 (removed)

OPF 0.4 and earlier carried an optional top-level `curation` section of
attorney-pinned clause positions. OPF 0.6 has no `curation` section (issue
#233; the review loop that produced pins was retired in issue #239). The
0.5 schema rejects a document that carries a top-level `curation` key, and
`identity.section_digests` has no `curation` digest. Section number 3.11 is
left unassigned so the numbering of later sections is stable.

### 3.12 `digest` (OPTIONAL) — the compact model-facing projection

A full OPF document carries every precedent's text and can run to millions
of characters on a real corpus — far beyond a model context. The `digest`
section is the compact projection designed to BE the system-prompt payload
of a consuming review application.

Normative rules:

- The digest MUST NOT contain `full_text` anywhere — drill-down goes through
  its citations against the full document.
- When present, `digest.clauses[].id` MUST exactly match
  `evidence.clauses[].id` (same set).
- `digest_version` (string, required) versions the digest shape itself,
  independent of `opf_version`.
- The digest is a pure function of the document and participates in
  `identity.content_hash` like any other content section.
- OPTIONAL in the schema; the reference compiler always emits it. Producers
  targeting ~40K tokens (chars/4) satisfy the intent.

An OPF 0.6 document carries `digest_version` "5" (§3.12.2).

> **History (retired 2026-10-07, issue #238).** `digest_version` "2" was the
> digest of an OPF 0.3 document (stances, preferred variations, frequency-
> banded exemplar forms). It was retired with 0.3; its definition and
> conformance vectors are in git history.
>
> **History (replaced in place 2026-10-09, issue #234).** `digest_version` "3"
> (per clause: our standard, the deal counts, the grouped signed variants and
> refused asks) is replaced by digest 4, which is digest 3 plus the fields of
> §3.12.2. Nothing consumed digest 3; the engine emits and validates only
> digest 4, and its definition and vectors are in git history.

#### 3.12.1 `digest_version` "3" (retired)

Replaced in place by digest 4 (§3.12.2, issue #234). Section number left
unassigned so the numbering of §3.12.2 is stable.

#### 3.12.2 `digest_version` "5" (OPF 0.6, issues #223, #234, #240, #243, #247, #248)

The verdict-free projection of §3.5.4's precedent record, plus the one judged
index of §3.5.6 (`vs_standard`: its label and compact part map):

```jsonc
"digest": {
  "digest_version": "5",
  "perspective": { "party": "...", "counterparty_type": "..." } | null,  // copy of §3.1
  "agreement_type": { "id": "...", "name": "..." },
  "corpus": { "n_deals": 6, "n_signed": 6, "first_signed": "2022-03", "last_signed": "2024-11" },
  "part_map_legend": { "part_map": { "=": "...", "+": "...", "-": "...",
                                     "x+": "...", "x-": "...", "x0": "..." },
                       "additions": { "+": "...", "-": "...", "0": "..." },
                       "label_by": { "parts": "...", "owner": "..." },
                       "settled": { "agreed": "...", "adjudicated": "...",
                                    "unchecked": "...", "owner": "..." } },
  "clauses": [ {
    "id": "...", "taxonomy_id": "...", "title": "...",
    "our_standard": { "parts": [ { "part_id": "...", "text": "..." } ],
                      "source_ref": Citation } | null,
    "n_deals": 6, "first_signed": "2022-03", "last_signed": "2024-11",   // span of the clause's evidence
    "n_signed_standard": 2,
    "n_opened_standard": 4, "n_kept_standard": 2,
    "standard_history": {                      // issue #249, §3.5.7
      "forms": [ { "form": "2019 form",
                   "in_use_from": "2019-03" | "2019" | null, "in_use_until": "2021-06" | null,
                   "basis": "supplied" | "reconstructed",
                   "in_use_basis": "config" | "deal_dates" | "mixed" | null,
                   "n_opened": 3, "n_kept": 2, "n_conceded": 1, "n_moved_to_current": 1,
                   "unchanged": false,
                   "label": ... | null, "part_map": [...] | null, "additions": [...] | null,
                   "label_by": "owner", "reason": "..." | null, "settled": ... | null,
                   "parts": [ { "part_id": "...", "text": "..." | null } ],
                   "ref": Citation } ],
      "why_changed": { "text": "...", "basis": "owner" } | null },
    "positions": { "standard": 2, "equivalent": 0, "more_protective": 0,
                   "less_protective": 0, "mixed": 1, "different_concept": 1, "unjudged": 2 },
    "signed_variants": [
      { "text": "...", "n_deals": 2, "first_signed": "2022-03", "last_signed": "2023-07",
        "n_from_standard": 2, "n_unchanged": 0, "label": "mixed" | ... | null,
        "part_map": ["=", "-", "x0"] | null, "additions": ["+"] | null,
        "label_by": "owner",               // only when the owner set the label
        "reason": "..." | null,
        "settled": "agreed" | "adjudicated" | "unchecked" | "owner" | null,
        "ref": Citation, "precedent_ids": ["prec.…"] },
      { "label": "equivalent", "n_deals": 3, "n_texts": 2, "n_from_standard": 1,
        "n_unchanged": 2, "first_signed": "2022-03", "last_signed": "2023-07",
        "settled_counts": { "agreed": 1, "adjudicated": 1, "unchecked": 0, "owner": 0 },
        "exemplars": [ { "text": "...", "ref": Citation } ],       // at most two
        "precedent_ids": ["prec.…"] }                               // the collapsed entry
    ],
    "refused_asks":    [ { "text": "...", "n_deals": 1, "first_signed": "2023-09",
                           "last_signed": "2024-11", "label": ... | null,
                           "part_map": [...] | null, "additions": [...] | null,
                           "label_by": "owner", "reason": "..." | null,
                           "settled": ... | null,
                           "ref": Citation, "precedent_ids": ["prec.…"] } ],
    "changed_openings": [ { "text": "...", "n_deals": 3, "n_to_standard": 1, "n_struck": 1,
                            "label": ... | null, "part_map": [...] | null,
                            "additions": [...] | null, "label_by": "owner",
                            "reason": "..." | null, "settled": ... | null,
                            "ref": Citation, "precedent_ids": ["prec.…"] } ],
    "n_variants_total": 2, "n_refused_total": 0, "n_changed_openings_total": 1,
    "n_signed_without_verified": 12, "n_signed_without_possible": 3,       // §3.5.8
    "verified_absent_deals": [ { "document_id": "...", "signed_on": "2025-03" | "2025-03-14" | null,
                                 "absence_id": "abs.…" } ]
  } ],
  "uncovered_clause_types": [ { "taxonomy_id": "...", "label": "..." } ]
}
```

- `perspective` is always present (`null` when the document has none): a
  consumer reading only the digest still knows which side it reviews for.
- `corpus.n_deals` / `n_signed` count in-scope `corpus.documents` (signed:
  `signed_version` non-null); `first_signed`/`last_signed` are the
  earliest/latest precedent `signed_on` of signed deals, **to the month**
  (`YYYY-MM`, `null` when no deal is dated). Every digest date is a month:
  a `signed_on` of `2025-03-14` and one of `2025-03` both read `2025-03`, and
  a month sorts before every day in it when records are ordered.
- `clauses[].first_signed` / `last_signed` — the same span over the clause's
  own precedent records (`null` when none is dated).
- Every count is distinct signed deals read from `evidence.precedent` with
  `signed: true`; paper side partitions nothing.
- `n_opened_standard` — deals whose `opened_with` is `"standard"`;
  `n_kept_standard` — of those, deals whose `standard` is `true`. Both equal
  the same-named `evidence.clauses[]` counts.
- `standard_history` (issue #249, §3.5.7) — always present; `forms` has one
  entry per former form with text for the clause, in config order (never
  capped: a handful of entries, and `parts` carries text only where a passage
  differs from today's), `why_changed` is the clause's owner note (a copy of
  the evidence's) or `null`. Per form: `form` is its label; `n_opened`,
  `n_kept` and `n_moved_to_current` are distinct signed deals that opened on
  that form (`opened_with_form`), kept what they opened with, and ended on the
  current standard; `n_conceded` is `n_opened − n_kept`; `unchanged` is true
  when the form's clause reads, under the grouping key, as today's standard
  (then there is no part map). `label`, `part_map`, `additions`, `reason` and
  `settled` (and `label_by` when the owner set the label) are the form's
  `vs_standard` against today's standard exactly as for any other entry,
  `null` when not judged or unchanged. `parts` lists the form's own passages
  in order as `{part_id, text}`; `text` is `null` when the `part_id` is a part
  of today's standard (the passage is unchanged, see `our_standard`).
  **Dates.** `in_use_from` / `in_use_until` are the config's own values
  (`baseline.prior_templates`, at the precision given) when given; an end the
  owner did not give is inferred, once per form, from the deals that opened on
  the form in ANY clause type: the earliest (`in_use_from`) and latest
  (`in_use_until`) signing month of those deals, else `null` — never guessed.
  A form carries the same dates in every clause's `standard_history`.
  `basis` is the form's `baseline.prior_templates[].basis` (`supplied`, or
  `reconstructed` from the corpus' first drafts, §3.5.7): a reconstructed form has
  no owner dates, so its dates are always inferred (`in_use_basis` `deal_dates`).
  `in_use_basis` says where the dates came from: `config` (every date shown is the owner's), `deal_dates` (every
  one is inferred), `mixed`, or `null` when there is no date. `ref` cites the
  clause inside the form.
- `positions` — the signed deals that signed a text for the clause, by what
  that text is: `standard` (the signed text is our standard language; equals
  `n_signed_standard`), each `vs_standard.label` (§3.5.6) of a non-standard
  signed text, and `unjudged` (a non-standard signed text with no label).
  All seven keys are always present.
- `our_standard` — the clause's standard as its ordered parts
  (`{part_id, text}`, §3.5) and its `source_ref`; the joined `text` is not
  repeated (it is the parts joined with a blank line).
- `signed_variants` — the non-standard `signed_text` of signed deals,
  grouped by §3.5.4's grouping key (exact; counterparty aliases and
  `perspective.party` neutralized — not `standard`'s configured party
  names); `n_deals` distinct deals; `text` is the sentence-boundary summary
  (≤ 300 chars) of the group's representative (latest `signed_on`, then
  lowest `document_id`) and `ref` its citation. Within a tier the order is
  `n_deals` desc, `last_signed` desc (unknown last), then grouping key.
  `first_signed` / `last_signed` are the earliest / latest signing month
  among the group's deals (`null` when none is dated).
  - `n_from_standard` — deals in the group whose `opened_with` is
    `"standard"`: the variant is a concession from our standard on record.
  - `n_unchanged` — deals in the group whose `opened_with` is
    `"non_standard"` and whose `opening_text` is `null`: signed exactly as it
    opened. (`"absent"` and `null` count in neither.)
  - `label` — the representative's `vs_standard.label`, or `null` when it is
    not judged. An index to check against the cited text, never an
    instruction. The label is orthogonal to the opening facts: a variant can
    be both a concession and `equivalent`.
  - **Part map** (issue #243). A judged entry carries `part_map`, one token
    per part of our standard in order: `=` present and equivalent, `+` / `-`
    present but more / less protective, `x+` / `x-` / `x0` omitted with a
    more protective / less protective / neutral effect; `additions`, one
    token per addition (`+`, `-`, `0`); and `reason`, one sentence. The
    label is derived from the part map and additions (§3.5.6, `parts`, the
    default), and `reason` is then the judgment's most important difference;
    such an entry carries no `label_by`, which saves a key on every entry of
    a budgeted digest. `label_by: "owner"` is present only when the owner
    overrode the derived label (`owner_label`): the part map and additions
    stay as judged and may imply another label, and `reason` is the owner's
    (`owner_label.reason`), never the judgment's reason for the label the
    owner overrode, which would contradict the label shown. `part_map`,
    `additions` and `reason` are `null` on an unjudged entry, which carries
    no `label_by`. Per-part reasons are not in the digest (they are in the
    evidence and the dossiers). `part_map_legend` states the tokens, the
    `label_by` values and the `settled` values (a constant).
  - **Settled** (issue #248). A judged entry carries `settled`, how settled
    its judgment is, derived from the judgment's `basis` and `check`
    (§3.5.6; never stored on the evidence): `owner` when `basis` is `owner`
    (whatever its `check` says), else `adjudicated` when `check.adjudicated`
    is true, `agreed` when `check.agreed` is true, and `unchecked` otherwise
    (no `check`, or a disagreement still awaiting adjudication). An
    `adjudicated` label was contested between two independent readings and
    deserves more scrutiny at review time; an `unchecked` one is weaker
    evidence than an `agreed` or `adjudicated` one. It is an index like the
    label, never an instruction, and a consumer never blocks on it. It is
    `null` on an unjudged entry.
  - **Collapse.** Every group whose `label` is `equivalent` is replaced by ONE
    entry (always last): `{label: "equivalent", n_deals (distinct deals
    across the collapsed groups), n_texts (groups collapsed), n_from_standard,
    n_unchanged (summed), settled_counts (how many of the n_texts collapsed
    texts are in each `settled` state: `{agreed, adjudicated, unchecked,
    owner}`, always all four, summing to n_texts), first_signed (earliest),
    last_signed (latest), exemplars (the first two groups in the order
    above, each {text, ref}), precedent_ids (all)}`. It has no `text`, `ref`
    or `settled` of its own.
  - **Order.** Individual entries come in four tiers: `less_protective` and
    `mixed` first (worse for us in some part), then `different_concept`, then
    those with a `null` label, then `more_protective`; the collapsed entry
    closes the list.
- `refused_asks` — every refused ask, grouped the same way, representative
  the earliest-round ask (then lowest `document_id`), with the
  representative's `label`; ordered `n_deals` desc, then grouping key.
  `first_signed` / `last_signed` are the earliest / latest signing month of
  the deals whose records carry an ask in the group.
- `changed_openings` — non-standard opening language that was not signed as
  proposed: precedents with `opened_with` `"non_standard"` and a non-null
  `opening_text`, excluding a precedent whose opening's grouping key equals
  the grouping key of one of its own `refused_asks` (already shown as a
  refused ask). Grouped by the grouping key of `opening_text.text`; `text`
  and `ref` are the summary and citation of the representative (lowest
  `document_id`); `n_to_standard` counts deals whose `standard` is `true`,
  `n_struck` deals whose `signed_text` is `null`; `label` is the
  representative's. Ordered `n_deals` desc, then grouping key. An opening on
  one of our former forms (`opened_with: "prior_standard"`) is never listed:
  its origin is ours (§3.5.7).
- The three lists are capped together (top 5, tightened stepwise to 1 until
  the digest fits the ~40K-token budget), the cap applying after the
  equivalent variants collapse. `n_variants_total` / `n_refused_total` /
  `n_changed_openings_total` are always the uncapped totals of distinct texts
  and equal `evidence.clauses[].n_variants` / `n_refused` /
  `n_changed_openings`.
- `n_signed_without_verified` / `n_signed_without_possible` (issue #251, §3.5.8)
  — distinct signed deals with no signed text for the clause, split into those it
  is **verified absent** from (`absent_verified`) and those it is **possibly
  present** in (`possibly_present`: a passage resembles it, or nothing could be checked). Facts, with no verdict:
  with the deals that signed the clause they make every signed deal.
  `verified_absent_deals` lists the verified deals (`absence_id` resolves into
  `evidence.absence.records[]`), latest `signed_on` first (undated last), then
  `document_id`, capped with the other lists; the counts are always the uncapped
  totals.
- `uncovered_clause_types` — every classifier-eligible `taxonomy.entries[]`
  entry (`status` `active` or `custom`) with no `evidence.clauses[]` entry,
  `{taxonomy_id, label}`, sorted by `taxonomy_id`, never capped. It means "a
  recognised clause type with no precedent in this corpus", nothing more.
- No `full_text`; no stance, band, risk or deviation field.
- **When present, the digest MUST equal the reference construction over the
  document** (`build_digest_v5`, defined by the 0.6 conformance vectors,
  §10.2) — a validator recomputes it and rejects any difference. A
  transform that edits evidence text (publication, residue redaction) MUST
  re-derive the digest, precedent ids and counts afterwards.

The single-file page artifact (`index.html`, produced by
`playbook view bundle`) embeds the canonical OPF JSON and the digest in
`<script type="application/json">` blocks (ids `opf-canonical`/`opf-digest`,
with every `<` escaped as `\u003c`; JSON parsing restores the value). The bare
`playbook.opf.json` remains the canonical artifact — a consumer extracts the
block and verifies `identity.content_hash` over the canonical serialization.

#### 3.12.3 `manifest`, `dossiers`, `provenance_index` (OPF 0.6, issue #228)

Three more OPTIONAL top-level sections the reference compiler always emits
beside the digest, so a consumer can enforce hard-rejection recall outside
both models and give its critic pass bounded, deeper context. Like the digest
each is a **pure function of the document**: a validator recomputes it
(`playbook_engine/dossiers.py`, defined by vector 008, §10.2) and rejects any
difference, and each participates in `identity.content_hash`. None holds a
judged verdict. A writer that changes the Floor after assembly
(`playbook floor sign`, the Posture interview's Q4 promotion) MUST re-derive
them before restamping `identity`.

**`manifest`** — `{hard_rules: [...]}`, one rule per `floor.invariants[]`
entry, in Floor order, and **only from the signed Floor, never from precedent
counts**: `{rule_id, clause_id, taxonomy_id, statement, required_presence,
condition, permissible_proof, fallback_language}`.

- `rule_id` is the invariant's `id`; `statement` its text, verbatim.
- `taxonomy_id` is the invariant's `x_taxonomy_id` and `clause_id` the
  `evidence.clauses[].id` of that clause type; both `null` when the invariant
  names none, and `clause_id` `null` when the corpus has no evidence for it.
- `required_presence` — the clause's absence or deletion is a hard rejection.
  The invariant's `x_required_presence`; `false` when unstated (an invariant
  that does not say the clause must appear never demands its presence).
- **A rule that demands presence or carries a predicate names its clause.** An
  invariant with `x_required_presence` `true`, or an `x_condition` other than
  `"judged"`, MUST also carry `x_taxonomy_id`: a predicate is checked against
  the text of the clause it names, and a presence rule rejects that clause's
  absence, so a rule with no clause could not be enforced by a consumer
  outside both models. The validator, `playbook floor sign` and the schema
  (`$defs.hardRule`: `taxonomy_id` is a string whenever `required_presence`
  is `true` or `condition` is a predicate) all refuse the unanchored form. An
  invariant that states neither is a `"judged"` rule and may name no clause.
- `condition` — a deterministic predicate spec the consumer evaluates outside
  any model, or the string `"judged"` for a rule that is not machine-evaluable
  (the default, and the only value when `x_condition` is absent). The specs:
  `{"type": "required_phrases", "phrases": [...], "match": "all"|"any"}` (the
  clause text contains every, or any, phrase, compared case-insensitively on
  normalized whitespace); `{"type": "numeric_bound", "pattern": "<regex with
  exactly one capture group>", "min"?: n, "max"?: n, "unit"?: s}` (the number
  captured is within the bounds); `{"type": "cross_reference", "clause_id":
  "..."}` (the clause text cross-references that clause).
- `permissible_proof` — what a reviewed document may show to satisfy the rule
  (`x_permissible_proof`, else `[]`).
- `fallback_language` — the clause's `our_standard.text` to insert when the
  protection is absent; `null` when the clause has no standard.

**`dossiers`** — `{clause id: dossier}`, one per `evidence.clauses[]` entry,
bounded by a **budget** and carrying **at most two precedent excerpts**:
`{clause_id, taxonomy_id, title, our_standard: {parts: [{part_id, text}]}|null, n_floor_rules,
floor_rules: [{rule_id, statement, rationale?}] (at most three, fewer when
dropped for size; `n_floor_rules` counts every one), excerpts, n_omitted,
omitted_precedent_ids}`. The rationale is the signed Floor's own
words for the clause (its `x_taxonomy_id` invariants) and the excerpts are the
edge cases and counter-arguments on record. A dossier carries no count (the
digest has them). Nothing is judged and nothing is summarised by a model.

**No text is ever cut part-way.** Every excerpt text (`opening`, `signed`),
our standard and every listed Floor rule appears **whole** or not at all: a
cut-off clause can drop its operative carve-out or cap and so plant a false
fact. There are no prefix cuts and no ellipsis truncation.

An **excerpt** is one precedent record rendered as an opening-to-signed pair,
`{precedent_id, kind, opening, signed, outcome, vs_standard}`: `opening` is the record's
`opening_text` and `signed` its `signed_text`, verbatim; with no recorded
`opening_text` the excerpt is the signed text alone (`opening` `null`); a
record whose clause was struck before signing has `signed` `null` and
`outcome` `struck_before_signing`. `vs_standard` (issue #243) is the judgment
of the text the excerpt's group stands for (the signed text of a signed
variant, the opening of a changed opening, the ask of a refused ask):
`{label, label_by, reason, settled, part_map, part_reasons: [{part_id, reason}],
additions: [{effect, summary}]}` — the digest's part map, `label_by`,
`reason` (the owner's when the owner set the label, §3.12.2) and `settled`
(`agreed`, `adjudicated`, `unchecked` or `owner`, §3.12.2, issue #248) plus the reason
of every part that is not equivalent or not a neutral omission and every
addition's summary — or `null` when that text is not judged. Selection is deterministic, in this order,
each candidate contributing one record of its group, and a group or a record
already chosen being skipped, until two are chosen:

1. the first digest-4 `signed_variants` group with `n_from_standard` > 0 (a
   concession on record; `kind` `signed_variant`), its record taken only
   among the members whose clause opened with our standard (`opened_with`
   "standard"), so the excerpt is the concession's own opening-to-signed
   pair;
2. the first `changed_openings` group (language that did not survive as
   proposed; `kind` `changed_opening`);
3. the remaining `signed_variants` groups, in digest-4 order, then the
   `refused_asks` groups (`kind` `refused_ask`: `opening` is the ask, `signed`
   what the deal signed instead, `outcome` `ask_refused`).

A group's record is its (eligible) member with the latest `signed_on`
(unknown last), **ties broken on the lowest `precedent_id`** (code-point
order). This is not the record the digest cites (§3.12.2 breaks its ties on
`document_id`). A `refused_asks` group's record is chosen the same way among
the records that made the ask, and the excerpt shows that record's
earliest-round ask in the group. The collapsed `equivalent` entry of the
digest stands for its groups in group order: step 1 takes the first of them
with `n_from_standard` > 0, step 3 the first of them. Adding precedents can
change which excerpts appear and so the texts the dossier holds, never add
to it: a precedent that leaves the selection unchanged leaves the dossier
byte-identical.

**Budget.** A dossier's budget is `max(1000, 3 × tokens(our_standard.text))`
tokens (a text's tokens are its characters / 4; a dossier's size is its
canonical JSON characters / 4), or 1,000 when the clause has no
`our_standard`: a clause whose own standard is long is not forced to drop the
evidence beside it. When a dossier is over budget, **whole parts are
dropped**, never cut, in this order:

1. the **listed Floor rules, last first**, while the dossier holding only its
   first excerpt (none, when the clause has none) is still over budget: a
   rule gives way only to the parts that are never dropped, which are our
   standard, the clause's identifiers and that first excerpt. `n_floor_rules`
   still counts every rule, and the manifest states each one verbatim;
2. then the **excerpts, in reverse selection order, keeping the first**,
   until the dossier fits. A dossier **always keeps at least one complete
   excerpt** (when the clause has any).

So a Floor rule is listed in preference to a second excerpt, and a second
excerpt is never dropped when it fits beside the rules left listed. Our
standard and the clause's identifiers are never dropped (the budget is at
least three times our standard's tokens). A dossier may exceed its budget in
**one case only**: it holds its single kept excerpt, every listed Floor rule
dropped, and keeping that excerpt whole puts it over. The scorecard counts
every such dossier as `over_budget_single_excerpt`. Any other dossier over its
budget is invalid: a validator refuses one with two excerpts, or with none
(its standard and identifiers alone exceed the budget). Each dropped excerpt
is named: `n_omitted` is their number and `omitted_precedent_ids` their
`precedent_id`s, sorted, so the critic knows more evidence exists; each
resolves through the provenance index (below) to its deal, and by id to its
record in `evidence.precedent`. Selection order, determinism and
byte-identical re-runs are unchanged by the budget.

**`provenance_index`** — not sent to a model: `{compiler: {name, version},
corpus_manifest_hash, documents, dossiers}`. `dossiers` maps a clause id to
the `{precedent_id, document_id, kind, omitted}` of each excerpt the dossier
selected, in selection order: first the excerpts it lists (`omitted` `false`,
in dossier order), then the ones it dropped whole to fit its budget
(`omitted` `true`), so every `omitted_precedent_ids` entry resolves here to
its deal and kind. `documents` has one `{document_id, signed_on,
version_files: [{version, sha256}]}` per deal behind those rows, kept or
omitted (sorted by id).

### 3.13 Identifier uniqueness (normative, effective 2026-07-29)

Every sibling-scoped id below MUST be **unique among its siblings**:

- `evidence.clauses[].id` — unique across the clause list.
- `evidence.precedent[].id` — unique across the precedent record (§3.5.4).
- `floor.invariants[].id` — unique across the Floor's invariant list.
- `corpus.documents[].document_id` — unique across the corpus.

The JSON Schema cannot express "unique across siblings," so this is enforced
only by a conformant validator, the same way §3.12's digest-consistency rule
is: **a conformant validator MUST reject a document containing a duplicate
sibling id** (fail-closed). Two siblings sharing an id make citation
resolution (§4) and any id-keyed export/digest projection genuinely
ambiguous — a consumer (or the compact `digest` of §3.12) keying off that id
alone cannot tell the siblings apart.

## 4. Citations

Every asserted clause text MUST be traceable.
```jsonc
{ "document_id": "string", "version": 4, "clause_path": "8.1", "char_span": [start, end] }
```
- `document_id`, `version`, and `clause_path` are REQUIRED; `char_span` is optional.
- `clause_path` is the dotted clause numbering in the normalized document, not the raw PDF page. A clause the reference engine cut out of another (§3.3, re-segmentation) is numbered `<clause>.r<k>`; its `char_span` lies inside the span of the clause it was cut from. One citation form is a list: an `our_standard` joined from several template clauses (§3.5) cites them all, so its `clause_path` is their dotted paths in document order separated by `, ` (for example `2, 5, 7`), and it carries no `char_span`. A `?` entry stands for a clause the compiler could not number; it has no navigation target. Every other citation names exactly one clause.
- `char_span`, when present, indexes into the document's full normalized text (document-relative — same coordinate system as `ClauseNode.char_span` in the clause-tree artifact), not the clause's own text. It spans the whole cited clause — from the start of its heading line through the end of its own body text (sub-clauses excluded; each carries its own span) — so a consumer resolving it lands on the clause language, not only its heading. (The reference engine also cuts an execution/signature block out of the clause that precedes it, so that block is not part of the clause text or its span.)
- `version` is the inferred ordinal (1-based); `"template"` is reserved for the baseline, today's form (`document_id` `"template"`), and `"prior_template"` for one of our former forms (§3.5.7), whose `document_id` is `prior_template:` followed by the form's `baseline.prior_templates[].label` (for example `{"document_id": "prior_template:2019 form", "version": "prior_template", ...}`). A former form is never cited as `"template"`: that names today's bytes. Every citation's `(document_id, version)` MUST resolve against `corpus.documents` (or, for `"template"`, `baseline.template_ref`; for `"prior_template"`, a `baseline.prior_templates[]` entry that carries a `sha256`) — dangling citations are non-conformant. When the cited document publishes `version_files` (§3.8), the cited version MUST have an entry there — a citation naming bytes no consumer can verify is likewise non-conformant.

### 4.1 Resolution algorithm (NEW)

A consumer holding the corpus files resolves a citation to verified bytes
without the compiler's workspace:

1. Read the citation's `(document_id, version, clause_path, char_span)`.
2. Look up `corpus.documents[document_id].version_files` and select the
   entry whose `version` matches; its `sha256` is the content address of
   the exact source file the compiler read. (For `"template"`, the address
   is `baseline.template_ref.sha256`: today's form. For `"prior_template"`,
   it is the `sha256` of the `baseline.prior_templates[]` entry whose `label`
   follows the `prior_template:` prefix of the `document_id`: that former
   form's own bytes, never today's template.)
3. Locate a file in the consumer's own corpus copy whose bytes hash to
   that address — the hash, not any filename or directory layout, is the
   key. No match means the consumer's copy differs from the compiled-from
   corpus: fail loud, do not fall back to a near-name file.
4. Open the verified file and navigate by `clause_path` (dotted numbering
   in the normalized document; for a comma-separated list, each path in turn,
   skipping `?`, the cited text being those clauses joined in order) and, when
   present, `char_span`.

`playbook resolve-citation <playbook> --clause <id> --obs <n>
--corpus-dir <dir>` is the reference implementation of these steps.

## 5. The determinism boundary (NEW — normative)

A conformant consumer MUST treat the three sections with three different bindings:

| Section | Binding | The consumer MUST… |
|---|---|---|
| **Floor** | **Hard** | run the fail-closed coverage gate in code — every invariant evaluated and logged on every review, an unevaluated invariant fails the run; a `violation` verdict forces the outcome; the model and Posture cannot override it. The verdict itself is an LLM judgment (a dedicated Floor judge, with adversarial second-pass/escalation per the live-eval policy), not lexical matching — the determinism is in **coverage and consequence, not detection**. |
| **Posture** | **Soft** | compose into the model's instructions to shape judgment; never let it, alone, gate a decision. |
| **Evidence** | **Advisory** | surface to the model as cited material to reason over; never treat `historical_stance` as a directive. |

This is the load-bearing contract of v0.2: it is what lets a stochastic model be pointed at high-stakes legal work. The model gets freedom in the soft middle; the Floor is the guarantee underneath it.

## 6. Producer / author / consumer responsibilities (NEW)

| Concern | Producer (compiler) | Author (legal owner) | Consumer (review engine) |
|---|---|---|---|
| Evidence | **populates** (auto-derived, cited) | — | surfaces to model; honors provenance + confidence |
| Posture | **drafts** from interview, grounded in Evidence | **edits & approves** | composes as soft instructions; versions edits under governance |
| Floor | **proposes** candidates from reversals/rejections | **authors & signs** | judges every invariant on every review (fail-closed coverage gate), forces the outcome on `violation` |
| Composition | **records** declared deps + integrity | **approves** module deps | loads only pinned+verified modules (fail-closed) |
| Corpus/compiler | **populates** | — | records lineage on every review |

## 7. Posture generation: the interview (NEW)

After compiling Evidence, the producer runs a short structured interview (the answers it cannot derive from the corpus — *forward-looking intent*) and drafts `posture.system_prompt` grounded in the Evidence (see §3.6 for what "drafts" and "grounded" mean for the reference producer, which assembles the prose deterministically from the answers rather than generating it with an LLM). The canonical starter set (3–6 questions; a producer MAY prune or extend):

1. **Rounds** — "How many negotiation rounds do you typically go on this agreement type before escalating or walking?"
2. **Leverage** — "What's your default leverage posture? (take-it-or-leave-it standard form / collaborative / we usually need the deal more than they do)"
3. **Risk appetite** — "When a counterparty change is non-material, do you default to accept-to-close, or hold the line?"
4. **Sacred clauses** — "Which clause types are non-negotiable regardless of deal value?" *(seeds Floor candidates — confirmed/signed by the author, never auto-promoted.)*
5. **Flexible clauses** — "Which clause types are you happy to concede to move a deal?"
6. **Output audience & deal-size sensitivity** — "Who will read the output, and does anything change above a deal-value threshold? (GC audience, terse rationale, no change by deal size / GC audience, terse rationale, but tightens above a threshold — say what threshold / junior-reviewer audience, needs it explained, no change by deal size / junior-reviewer audience, needs it explained, and tightens above a threshold — say what threshold)"

The mapping is explicit: **Q4** seeds Floor candidates; **Q1–3, 5–6** shape the Posture prose. The producer MUST record every question/answer in `posture.generation.interview` (§3.6).

## 8. Governance & lineage (NEW — boundary with the consumer)

OPF owns the *format*; the consuming app owns the *runtime lifecycle*. The boundary:

- The OPF carries the **genesis** Posture (compiler-generated) and the **signed** Floor. These are the starting state.
- Once installed, the consumer's release-bundle governance owns subsequent Posture/Floor versions. Each app-side version MUST record its parent OPF section digest, so the lineage **corpus → OPF(evidence+posture+floor) → installed bundle → edited posture version → decision** is reconstructible end to end.
- A consumer MUST record, on every review: the OPF identity + section digests, any edited-Posture version, the active Floor digest, and every composed module's `module@version#integrity`.
- Rollback and quarantine operate on the **bundle** (all sections + bound standard form + model policy), not on a single section in isolation.

(The concrete bundle/sign/activate/rollback mechanics live in the consuming app's governance docs; OPF only fixes the *lineage fields* the consumer must record.)

## 9. Confidence (unchanged)

Confidence is advisory, not statistical authority. It is a function of precedent count and provenance mix, with our-paper observations weighted above counterparty-paper. With small corpora most entries will be low-confidence; consumers MUST surface confidence and MUST NOT treat a single precedent as a rule.

## 10. Conformance

- A **conformant playbook** carries `opf_version` "0.6", validates against `spec/playbook.schema-0.6.json`, and obeys the normative rules in §2.3, §3.4, §3.5.4, §3.5.5, §3.6, §3.7, §3.8, §3.12, §3.13 and §5. A conformant validator rejects any other `opf_version` — including the retired "0.1" to "0.5" — as unsupported (`playbook_engine/validator.py`). The §2.2 evidence-depth rule capped a stance that 0.5 and 0.6 no longer carry, and paper side gates nothing.
- A **conformant producer** emits conformant playbooks, records out-of-scope documents, drafts the Posture from a recorded interview, and only *proposes* (never auto-promotes) Floor candidates.
- A **conformant consumer** honors the determinism boundary (§5), honors provenance (§2.3), surfaces confidence (§9) and citations (§4), loads only pinned+verified composed modules (§3.4), and records the lineage fields (§8).

### 10.1 Vendor extensions (`x_*`)

The schema reserves an `x_*` prefix for vendor extensions at designated
levels (the document root, `evidence.clauses[]` and their observations,
`clause_library[]` entries, `posture`, `floor` and its invariants,
and `corpus.documents[]` entries). Extensions are not
permitted where hash integrity or mechanical resolvability depends on a
closed shape (`identity`, citations, `agreement_type`, `taxonomy` entries,
`compiler`).

- Conformant consumers MUST ignore unknown `x_*` fields.
- Producers MUST NOT put normative behavior behind `x_*` fields — a
  playbook stripped of every `x_*` field must mean the same thing.
- `x_*` fields ARE content: they participate in `identity.content_hash`
  and the section digests, so two documents differing only in an `x_*`
  value are different playbooks.

### 10.2 Conformance vectors — canonicalization and digest (normative)

`spec/conformance/0.6/` is the **normative definition** of the two
mechanical algorithms everything in §3.10 (`identity`) and §3.12 (`digest`)
rests on: canonical serialization + content hashing (reference
implementation `playbook_engine/canonicalize.py`) and `digest` section
construction (reference implementation `playbook_engine/digest.py`).
`spec/conformance/0.6/manifest.json` stamps the exact `opf_version` /
`digest_version` / reference `engine_version` the vectors bind to, per the
§11 immutability rule below — a format-version bump gets a new,
separately-stamped vector set, never an in-place edit of an existing one.

Each vector under `spec/conformance/0.6/vectors/` is a self-contained,
plain-JSON `{input, expected}` pair (`canonical`, `content_hash`,
`section_digests`, `digest`, and, since issue #228, `manifest`, `dossiers`, `provenance_index`) — **an independent, non-Python implementation
that reproduces every vector's `expected.*` from its `input` is conformant**
with canonicalization and digest construction for that format version, with
no dependency on this repo. `tests/test_conformance_vectors.py` is this
engine's own check against the same frozen vectors; see
`spec/conformance/0.6/README.md` for what each vector isolates: canonical
serialization of 0.6-shaped documents and digest 5 construction — variant
grouping by §3.5.4's grouping key (including its counterparty-alias and
`perspective.party` neutralization), the `n_deals`/`last_signed` ordering,
the cap with uncapped totals, refused-ask grouping across deals, the
sentence-boundary summary, and `perspective` carried as `null` when the
document has none; vector 005 carries the opening evidence of §3.5.5; vector 006 pins the opening rules (`n_opened_standard`/`n_kept_standard`, `n_from_standard`/`n_unchanged`, `changed_openings` and its refused-ask exclusion); vector 007 pins the `vs_standard` collapse, tier order, the part maps (a mixed changed opening) and `uncovered_clause_types`; vector 008 pins the hard-rule manifest (a presence or predicate rule names its clause), the dossier excerpt order (the concession's own record, the `signed_on` then lowest-`precedent_id` record order and a refused-ask excerpt) and the budget (a whole excerpt dropped and named, a single over-budget excerpt kept whole with its Floor rule dropped, and the last Floor rule dropped whole from a dossier with no excerpt so that it fits), and the provenance index with its corpus snapshot hash, source-file hashes and the dropped excerpt indexed as `omitted` with its deal (§3.12.3); vector 009 pins the part rules on a standard of several parts (a repeated passage's `-2` `part_id`, multi-token part maps with omissions and additions, part reasons naming parts other than the first) and an owner label carried with `label_by` `owner` and the owner's reason (§3.5.6, §3.12.2, §3.12.3); vector 010 pins the signing dates (§3.8, §3.12.2): month-only and full dates, every `signed_on_basis`, a signed deal with no date, an unsigned deal, and the `first_signed` / `last_signed` months of the corpus, a clause, a signed variant and a refused ask; vector 011 pins `settled` (§3.5.6, §3.12.2, §3.12.3): all four outcomes on signed variants, a refused ask, a changed opening and a dossier excerpt, a disagreement awaiting adjudication read as `unchecked`, an owner judgment (which carries no check) read as `owner`, and the collapsed equivalent entry's `settled_counts`. The canonical-serialization algorithm itself is stated in
`spec/conformance/README.md`. (The OPF 0.3 / digest 2 vector set, which also
pinned key ordering, Unicode emission and float/int formatting edge cases,
was retired with that format, issue #238; those edge cases are unit-tested
in `tests/test_canonicalize.py`.)

This exists because a hand-maintained downstream port of `canonicalize.py`
or `digest.py` (Contract Toaster carries one, pinned to an engine commit by
docstring) fails silently on drift: a changed algorithm doesn't raise an
import error, it just produces a different `content_hash` — surfacing later
as an ingest-hash-verification failure, or worse, a hash that agrees when it
shouldn't.

## 11. Versioning & migration

OPF uses semantic versioning. `opf_version` is required.

**Stability policy (normative, effective at 1.0):** 1.0 is the first
non-beta release — the format is no longer "breaking changes possible until
1.0." Within the 1.x series:
- A 1.x release MAY add a new OPTIONAL field, or a new `x_*` vendor
  extension (§10.1).
- A 1.x release MUST NOT add a new REQUIRED field, remove or retype an
  existing field, or change what an existing field means or how its
  contents are selected.
- Removing or retyping a field, or changing a normative MUST, requires a
  2.0 release — never an in-place edit or a same-major-version reinterpretation
  of an existing rule.

**Normative rule changes (normative, effective at 1.0):** the drift class
that actually bites a consumer is a normative rule changing *without* a
schema change or a version bump — JSON Schema validation alone cannot catch
it. §3.13's id-uniqueness rule shipped in exactly this shape (a blocking
validator rule, no schema change, no version bump). Going forward, any new
or changed MUST — whether or not it touches the schema — MUST get an entry
in `CHANGELOG.md` under a dedicated `### Normative rule changes` heading in
the release it ships under, so a downstream implementation (including an
independent one checked against the conformance vectors, §10.2) has a
durable, greppable record of what changed.

**Immutability rule (normative, effective 2026-07-16):** a published
`opf_version` is immutable once a consumer exists. Shape or *semantic*
changes — including changes to what existing fields mean or how their
contents are selected — get a NEW `opf_version`, never an in-place edit of a
released schema; "old data still validates" is not sufficient. The `digest`
section's `digest_version` follows the same rule independently. Every
spec-affecting change gets an entry in `spec/CHANGELOG.md`, whose schema-hash
pins are CI-enforced (`tests/test_spec_consistency.py`).

**Retirement (2026-10-07, issue #238; 2026-10-08, issue #233).** With no
installed consumer, the owner retired OPF 0.1–0.3 and `digest_version` 2,
and 0.5 then replaced 0.4 in place (nothing consumed 0.4), and 0.6 replaced
0.5 the same way (issue #247): the reference engine emits and validates only
0.6, and a conformant validator rejects any other `opf_version` as
unsupported. Retirement removes a format from the
reference implementation; it never edits one — the retired schemas and
vectors are unchanged in git history and in contract-opf/opf.

**How the 1.x series relates to `opf_version` (normative):** the engine's
1.x series and the document shape's `opf_version` are versioned
independently, and the stability policy and the immutability rule above
compose as follows — an additive 1.x change (a new OPTIONAL field or `x_*`
extension, permitted by the stability policy) ships as a NEW `opf_version`
(e.g. 0.5 → 0.6), per the immutability rule; it is never an in-place edit
of a frozen shape. A consumer pinned to the earlier `opf_version` is
unaffected by such a change; a consumer that wants the new field upgrades
its pin to the new `opf_version`. "A published version is frozen" and "1.x
is additive-only" are therefore both true without conflict.

**Migrations.** The 0.1 → 0.2, 0.2 → 0.3, 0.3 → 0.4, 0.4 → 0.5 and 0.5 → 0.6
migration notes described upgrade paths between formats the engine no longer
reads (issues #238, #233); they are in git history. A document in a retired
format is not converted: re-mine and re-project (`playbook mine`, then
`playbook project`), which emits 0.6. A store mined before 0.6 carries no
signing-date evidence (and one mined before 0.5 no opening evidence), so it
must be re-mined, not only re-projected.

## Appendix A — Open questions (for review, not yet decided)

1. **Composition mechanics (§3.4).** The governance contract is fixed; the *module interface* is not. Needs one investigation pass against a real consuming application's module structure before composition alters runtime behavior. Until then, `composes` is recorded for lineage but inert.
2. **Floor minimality (§3.7.1).** The admission test is the guard against the Floor regrowing into a rigid playbook. Worth pressure-testing on a mature production Floor: how many of its accumulated hard rejections survive *both* (a) categorical and (b) statable as a single judge-checkable invariant? The ones that fail (a) should become Posture; the ones that fail (b) should become Evidence guidance.
3. **`historical_stance` vs. a numeric tendency.** ~~`mixed` is coarse. A future version might carry a held-rate (e.g. "held in 7 of 9 our-paper deals") instead of/alongside the enum. Deferred.~~ **Resolved, then superseded:** 0.2/0.3 carried the held-rate as `summary.stance_detail`; 0.4 and 0.5 carry no stance at all, only per-clause deal counts (`n_signed_standard` of `n_deals`, §3.5.4).

## Appendix B — Changelog
- **0.6 (issue #251)** — Verified absence (§3.5.8, §3.12.2): `evidence.absence` (`guard` and one `records[]` entry per signed deal and clause type with no signed text: `absent_verified` or `possibly_present`, its `basis`, `n_checked` and cited `candidates`); the digest's per-clause `n_signed_without_verified`, `n_signed_without_possible` and `verified_absent_deals`. A missed classification is never read as an absence. OPF 0.6 and digest 5 are replaced in place; there is no backward compatibility.
- **0.6 (issue #250)** — Reconstructed former forms (§3.2, §3.5.5, §3.5.7, §3.12.2): `baseline.prior_templates[]` entries gain `basis` (`supplied` by default, or `reconstructed` for a form the producer rebuilt from the corpus' first drafts on our paper when none was supplied) with `n_deals`, `first_signed` and `last_signed`; a reconstructed form has no `sha256`, its `prior_standards[]` cite the corpus document the wording was read from, and the digest's `standard_history.forms[]` gain `basis`. Only clusters the `prior_form_origin` judgment confirmed (blind-checked, adjudicated on disagreement) become forms.
- **0.6 (issue #249)** — Prior standards (§3.2, §3.5.4, §3.5.5, §3.5.7, §3.12.2): `baseline.prior_templates[]` (our older forms' labels, owner dates and content addresses); `evidence.clauses[].prior_standards[]` (each form's clause as parts with stable `part_id`s, cited in the form's own bytes — `version` `"prior_template"`, `document_id` `prior_template:<label>`, resolved through the form's `sha256` (§4.1) — and its `vs_standard` against today's standard, judged once, subject role `prior_standard`), `why_changed` (the owner's optional note) and the counts `n_opened_prior` / `n_kept_prior` / `n_moved_to_current`; `opened_with` gains `"prior_standard"` with `opened_with_form`, so our own former wording is never a changed opening or a refused ask; the digest's per-clause `standard_history`. OPF 0.6 and digest 5 are replaced in place; there is no backward compatibility.
- **0.6 (issues #247, #248)** — Signing dates (§3.5.4, §3.8, §3.12.2): `corpus.documents[].signed_on` (an ISO date, or `YYYY-MM` when only the month is known, or `null`) and `signed_on_basis` (`document_text` | `signature_block` | `filename` | `file_metadata`, present exactly when the date is); every precedent record gains a required `signed_on` equal to its document's, which replaces the optional `signed_at` (the `YYYY-Qn` quarter form is gone). `digest_version` "5": `first_signed` / `last_signed` (`YYYY-MM`) on every signed variant (and the collapsed equivalent entry), every refused ask and every clause, and the corpus span is to the month. New validator MUSTs: both keys present on every in-scope document (null when unknown), a well-formed real date in 1980-2099, a basis exactly when there is a date, no date without a detected executed copy, and a precedent's date equal to its document's. New schema file `playbook.schema-0.6.json` and conformance set `spec/conformance/0.6/` (vector 010); they replace 0.5 in the reference engine. Issue #248 adds `settled` (§3.5.6, §3.12.2, §3.12.3): how settled each judged text's `vs_standard` is (`agreed`, `adjudicated`, `unchecked` or `owner`), derived from its basis and blind check and carried on every non-equivalent digest variant, refused ask and changed opening and on each dossier excerpt's `vs_standard`; the collapsed equivalent entry carries `settled_counts`; `part_map_legend.settled` states the values; conformance vector 011.
- **0.5 (issue #234)** — `digest_version` "4" (§3.12.2) replaces digest 3 in place: per clause `n_opened_standard` / `n_kept_standard` / `positions`, `n_from_standard` and `n_unchanged` on each signed variant, `changed_openings` (with `n_changed_openings_total`), the `vs_standard` `label` on variants, asks and openings (equivalent variants collapse into one entry), and top-level `uncovered_clause_types`; `evidence.clauses[]` gains `n_opened_standard`, `n_kept_standard` and `n_changed_openings`. The 0.5 conformance set was regenerated for digest 4 with vectors 006 and 007.
- **0.5 (issue #240)** — Equivalence label (§3.5.6): an optional `vs_standard` `{label, reason, basis, check} | null` on `signed_text`, `opening_text` and each `refused_asks[]` entry, judged once per distinct text against our standard and blind-checked by an independent model; three new validator MUSTs.
- **0.5 (issue #243)** — `vs_standard` is judged part by part (§3.5.6): `our_standard.parts[]` (ordered template passages with content-hash `part_id`s, §3.5); each judgment carries `parts` (present with a `relation` or omitted with an `effect`) and `additions`, and its `label` — gaining `mixed` — is DERIVED from them; `check` records `label_agreed` and `parts_agreed`; an owner may override the label (`owner_label`). The cache key covers the ordered parts. Digest 4 carries our standard as its parts, a `part_map_legend`, and on every judged variant, ask and opening a compact `part_map`, `additions` tokens and the `reason`, plus `label_by: "owner"` (and then the owner's reason) only when the owner set the label; `mixed` leads with `less_protective`; dossier excerpts carry their judgment's part map, `label_by`, reason and part reasons. The 0.5 conformance vectors are regenerated, and vector 009 pins a standard of several parts and an owner label. Replaced in place: nothing outside the engine had vendored 0.5.
- **0.5** — Opening evidence (§3.5.5): every precedent record gains `opened_with` (`standard` | `non_standard` | `absent` | `null`) and `opening_text` is recorded for any distinct opening, whatever its origin (0.4 recorded it only for our standard struck before signing). Three new validator MUSTs. The optional top-level `curation` section (§3.11) and `identity.section_digests.curation` are removed: the 0.5 schema rejects them (issue #233, after the review loop that produced pins was retired in issue #239). New schema file `playbook.schema-0.5.json`; it replaced 0.4 in the reference engine (no consumer ever bound 0.4). `digest_version` "3" is unchanged until digest 4 lands (epic #236); 0.5 has no consumer until contract-opf/contract-toaster#128 vendors it.
- **Retirement (issue #238)** — The reference engine reads and writes only 0.4 (now 0.5); OPF 0.1–0.3 and `digest_version` "2" are retired (validators reject them as unsupported). Not a new shape: 0.4 is unchanged. §3.5.1–§3.5.3 and §3.12's digest 2 rules are replaced by history notes.
- **0.4** — Verdict-free per-deal precedent record as the evidence shape (§3.5.4): `evidence.{clauses, precedent}` with distinct-deal counts, deterministic `standard`, three-valued paper metadata that gates nothing, and recomputable precedent ids; `digest_version` "3" (§3.12.1) with `perspective`, grouped signed variants and refused asks, and a digest-equals-recomputation rule; judged verdicts only under `x_judgments`. New schema file `playbook.schema-0.4.json`; 0.3 frozen.
- **1.0** — Stability commitment, not a shape change: the document shape (`opf_version` "0.3") is unchanged. New §11 stability policy (1.x changes are additive-only — new OPTIONAL fields and new `x_*` extensions permitted; no new REQUIRED field, no removing/retyping a field, no changing a normative MUST without a 2.0 release) and new §11 normative-rule-change policy (any new or changed MUST, whether or not it touches the schema, gets a `CHANGELOG.md` entry under a `### Normative rule changes` heading in the release it ships under).
- **0.3** — `digest` section (§3.12): compact model-facing projection of `evidence` (stances, preferred variations verbatim, text_summary-only concession/unacceptable summaries, frequency-banded deduplicated exemplar forms with `example_ref` drill-down); single-file bundle artifact `playbook.opf.html` embedding the canonical JSON + digest; additive over 0.2.
- **0.2** — Three-section model (Evidence / Posture / Floor); `historical_stance` (descriptive) replaces `rollup.position` (prescriptive); `composes` (pinned external modules); determinism boundary (§5); producer/author/consumer responsibilities (§6); Posture interview (§7); lineage boundary with the consumer (§8); `identity` — canonical serialization, `content_hash`, per-section digests, producer-assigned `id`/`version`/`supersedes` (§3.10, issue #143); `curation` — embedded attorney-pinned positions surviving recompile with deterministic conflict-flagging (§3.11, issue #147).
- **0.1** — Initial draft. Risk-delta model, provenance rule, dual structure (clause positions + clause library), citation requirement, taxonomy curation model.
