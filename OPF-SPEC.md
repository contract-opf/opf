# Open Playbook Format (OPF) — Specification

**Version:** 0.3 (additive over 0.2 — §3.12 `digest`; the 0.2 shape is unchanged)
**Status:** Current specification (0.3; 0.2 documents remain valid). Breaking changes possible until 1.0.
**Serialization:** JSON (canonical). YAML permitted for authoring; tools MUST accept both and treat them as equivalent.
**Issue references:** #NNN citations throughout are design provenance from the project's original development tracker.
**License:** This file is specification text and is additionally licensed under the Creative Commons Attribution 4.0 International License (CC-BY-4.0), per the repository `LICENSE` — alongside the Apache-2.0 license covering this repository.
**Reference implementation:** In-text references to `playbook_engine/…`, `tests/…`, and `docs/OPF-BUNDLE-BOUNDARY.md` point into the reference compiler, [`contract-opf/playbook-engine`](https://github.com/contract-opf/playbook-engine). The JSON schemas below live in this repo under [`schema/`](schema/) and are byte-vendored from that implementation (their canonical `$id`s resolve there).

---

## 0. What changed in 0.2, and why

v0.1 modeled a playbook as a set of **clause positions with a frozen, operative posture per clause** (`rollup.position ∈ {standard, acceptable_variants_exist, negotiable, hold_firm}`). That made sense when the consumer was a less-capable reviewer that needed the answer pre-computed. The consuming review engine is now a SOTA model. Pre-freezing a posture per clause is no longer the right interface: it is rigid where the model is flexible, and it hides the negotiation intent that actually drives a decision.

v0.2 makes one structural move: **determinism migrates out of the knowledge and into the guardrails.**

A playbook is now **one document with three sections**, authored at different times by different owners:

| Section | What it carries | Author | Determinism at runtime |
|---|---|---|---|
| **Evidence** (§3.5) | What the corpus shows — accepted variants, concessions, rejections, each cited to precedent. *Descriptive, not prescriptive.* | Auto-derived by the compiler | **Advisory** — the model reasons over it |
| **Posture** (§3.6) | Negotiation intent as a system-prompt-style prose block (rounds, leverage, risk appetite, what's sacred vs. flexible, output audience). | Compiler **drafts** from a short interview; a legal owner edits | **Soft** — shapes judgment, not a gate |
| **Floor** (§3.7) | The red lines that must *never* slip: judge-evaluated natural-language invariants. | Legal owner authors (compiler may *propose* candidates) | **Hard** — deterministic coverage gate + forced consequence; the model cannot override |

Plus one new top-level concern: **composition** (§3.4) — an OPF may declare *composed* external clause-intelligence modules (e.g. a privacy/DPA module) rather than re-encoding that knowledge itself: a pinned, validated dependency recorded for lineage under a fixed governance contract. (Behavior wiring is deferred — see §3.4's open note; today `composes` is declared and verified, not yet executed.)

Everything that made v0.1 trustworthy is retained and, in the Floor and citation rules, sharpened: every asserted text still cites precedent (§4); the provenance rule still holds (§2.3); out-of-scope documents are still retained (§3.8). The change is *not* "less governance because the model is smarter" — it is *more* model freedom in the soft middle, bounded by a hard floor it can never cross and a citation it must always give.

This section summarizes the reframe.

---

## 1. Purpose and scope

OPF is an open format for representing a **negotiation playbook** for a single agreement type, compiled from a corpus of real negotiated agreements. A playbook tells a reviewer — human or LLM — how to evaluate a clause in a new agreement: what the preferred position is, what the corpus shows we have accepted, conceded, and rejected, what our negotiation posture is, and what red lines must never be crossed — each factual claim grounded in cited precedent.

OPF describes **what the playbook knows and intends**, not the review engine's internals (thresholds, retrieval, model policy). OPF is the interface between the corpus→playbook compiler and any downstream review engine.

### Non-goals
- OPF is not a contract format and does not represent agreements themselves.
- OPF does not encode the review engine's thresholds, retrieval, or model policy.
- OPF is not a trained model; it carries no weights.
- OPF does not own the *runtime lifecycle* of an edited Posture. The OPF carries the **initial, compiler-generated** Posture (the genesis record); once a playbook is installed, the consuming app's release-bundle governance owns subsequent Posture versions (§8).

> **OPF vs. the consuming app's release bundle:** OPF is the canonical
> playbook format — knowledge, intent, red lines, perspective, and de
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
| **Floor** | The deterministic red lines that must never slip. (§3.7) |
| **Composition** | A pinned dependency on an external clause-intelligence module. (§3.4) |
| **Observation** | A single occurrence of a clause variant in the corpus, with its outcome and provenance. |
| **Deviation** | How much an observed clause differs from our standard: `none` / `reworded_equivalent` / `substantive`. |
| **Risk delta** | The risk change relative to our standard: a `direction` (`better`/`neutral`/`worse`) and a `magnitude` (`none`/`minor`/`material`). |
| **Outcome** | What happened to an observed variant: `signed` or `proposed_then_reversed` (an explicit rejection). |
| **Provenance** | `our_paper` or `counterparty_paper`. |
| **Historical stance** | A *descriptive* summary of what the corpus shows we have done on a clause (§2.2). Replaces v0.1's prescriptive `rollup.position`. |
| **Curation pin** | An attorney-asserted clause position, embedded in the OPF, that survives recompile and wins over the recomputed rollup. (§3.11) |

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
  "opf_version": "0.2",
  "agreement_type": { "id": "...", "name": "...", "description": "...", "aliases": ["..."] },
  "baseline": { ... },          // §3.2
  "taxonomy": { ... },          // §3.3
  "composes": [ Composition ],  // §3.4 — pinned external module deps (NEW)
  "perspective": { ... },       // §3.1 — whose side this playbook reviews from (NEW)
  "de_minimis": [ "..." ],      // §3.1 — change categories accepted even if novel (NEW)
  "evidence": {                 // §3.5 — the compiled, cited knowledge
    "clauses": [ ClausePosition ],        // template-anchored
    "clause_library": [ ClauseConcept ]   // concept-indexed (counterparty paper)
  },
  "posture": { ... },           // §3.6 — generated negotiation intent (NEW)
  "floor": { ... },             // §3.7 — red lines: judged NL invariants (NEW)
  "corpus": { ... },            // §3.8 — provenance/audit + content addresses
  "compiler": { ... },          // §3.9 — generation metadata
  "identity": { ... },          // §3.10 — content hash + section digests (NEW)
  "curation": { ... }           // §3.11 — embedded attorney-pinned positions (NEW, OPTIONAL)
}
```

### 3.1 Top-level fields
- `opf_version` (string, required) — `"0.2"`.
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
  "notes": "free text"
}
```
If `has_canonical_template` is `false`, the playbook is *emergent*. The provenance rule still applies; with no template, `our_standard` is set only from our-paper deals, and most stances will be `mixed`/`no_signal`.

### 3.3 `taxonomy`
A curated clause taxonomy (unchanged from v0.1). Entries may be inactive so curation survives upstream taxonomy upgrades. A compiler MUST only classify clauses into `active` or `custom` entries.
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
2. A composed module MAY supply Evidence-equivalent guidance and Floor-equivalent red lines *for its `applies_to_taxonomy` scope only*. It MUST NOT widen scope beyond what the OPF declares.
3. Where a composed module and the OPF's own Floor conflict, **the stricter rule wins** (a Floor is a one-way ratchet; composition can only add red lines, never relax them).

> **Open (resolve before implementing composition):** the concrete interface a module exposes — does it hand back structured clause positions, a sub-prompt, a detector set, or all three? — is **not** fixed in 0.2. This section pins the *governance contract* (pin + fail-closed + scope + stricter-wins); the wiring is deferred to one investigation pass against the actual module structure. Until then, `composes` is a declared, validated dependency the runtime records on every review for lineage, even if it does not yet alter behavior.

### 3.5 `evidence` — the compiled, cited knowledge

`evidence` holds the two structures v0.1 carried at top level (`clauses`, `clause_library`), now grouped and reframed as **descriptive evidence**.

#### 3.5.1 `ClausePosition` (template-anchored)
```jsonc
{
  "id": "clause.indemnification",
  "taxonomy_id": "indemnification",
  "title": "Indemnification",
  "our_standard": {
    "text": "…canonical language…",
    "source_ref": { "document_id": "template", "clause_path": "8", "char_span": [s, e] }
  },
  "observed_positions": [
    {
      "text_summary": "Mutual indemnification, negligence-based.",
      "example_ref": { "document_id": "…", "version": 4, "clause_path": "8.1", "char_span": [s, e] },
      "deviation": "substantive",
      "risk_delta": { "direction": "worse", "magnitude": "minor" },
      "provenance": "our_paper",
      "outcome": "signed",
      "precedent_count": 3
    }
  ],
  "summary": {
    "historical_stance": "usually_held",     // §2.2 — DESCRIPTIVE, not an instruction
    "acceptable_if": [                       // cited tolerance conditions — {if,to,rationale} triples (#141)
      { "if": "mutual, negligence-limited indemnification",
        "to": "…the full accepted alternative clause language…",
        "rationale": "Signed with neutral risk_delta; 3x precedent in the corpus.",
        "observation_ref": { "document_id": "…", "version": 3, "clause_path": "8.1", "char_span": [s, e] } }
    ],
    "fallbacks": [ /* worse-risk signed observations, ordered least-to-most costly */ ],
    "rejected": [ /* observations with outcome = proposed_then_reversed */ ],
    "confidence": { "score": 0.0, "basis": "precedent_count + provenance_mix",
                    "n_our_paper": 3, "n_counterparty_paper": 1 }
  }
}
```
- `summary` replaces v0.1's `rollup`. The only substantive change is `rollup.position` → `summary.historical_stance` and the renaming of `rollup` to `summary` to signal "this describes the corpus" rather than "this is your position."
- `acceptable_if`, `fallbacks`, `rejected` are retained verbatim from v0.1's semantics, as **cited evidence summaries** the model reasons over.
- All `*_ref` objects are **citations** (§4) and are REQUIRED wherever text is asserted.

#### 3.5.2 `ClauseConcept` (concept library, for counterparty paper)
Unchanged from v0.1. Used when reviewing a document drafted on the counterparty's template, matched by concept and risk rather than diffed against our template.
```jsonc
{
  "concept_id": "concept.governing_law",
  "taxonomy_id": "governing_law",
  "description": "Which state's law governs and where disputes are heard.",
  "risk_profile": "Venue/forum exposure; cost of litigating away from home.",
  "accepted_forms": [
    { "text_summary": "Counterparty's home state law, no forum clause.",
      "example_ref": { "document_id": "…", "version": 5, "clause_path": "…", "char_span": [s, e] },
      "provenance": "counterparty_paper",
      "risk_delta_vs_our_standard": { "direction": "worse", "magnitude": "minor" } }
  ],
  "notes": "Tolerated in N of M counterparty-paper deals."
}
```

#### 3.5.3 Negotiation dynamics (NEW in 0.2)

Evidence carries the *dynamics* of how positions moved, not just what
signed. All dynamics fields are OPTIONAL and omitted when underivable —
they are never fabricated (same discipline as `identity`).

Per **observation**:

```jsonc
{
  "proposed_by": "counterparty",          // "us" | "counterparty" | "unknown" — who introduced the change
  "observed_at": "2025-11-04",            // ISO-8601 date, for recency; never fabricated
  "counterparty_ref": {                    // per-deal counterparty segment
    "alias": "counterparty-007",           // entity-registry pseudonym (born-safe: raw names never appear)
    "counterparty_type": "public university"
  }
}
```

- `proposed_by` derives from tracked-changes authorship mapped through the
  party aliases, else from which side's draft first carries the span —
  never guessed from filenames.
- `observed_at` precedence: e-sign certificate date on the signed copy >
  embedded document metadata > trustworthy filename date.
- `counterparty_ref` lets a consumer weight "we conceded this to a
  hospital system, not a community college."

Per **ClausePosition**:

```jsonc
"negotiation_trail": [                     // round-by-round ask→landing, one entry per round the clause changed
  { "document_id": "university-of-example",
    "round": 2,                            // version-transition ordinal (v2→v3 = round 2)
    "moved_by": "counterparty",
    "change_summary": "Cap raised from 1x fees to 2x fees.",
    "risk_delta": { "direction": "worse", "magnitude": "minor" },
    "ref": { "document_id": "university-of-example", "version": 3, "clause_path": "8.1", "char_span": [s, e] } }
],
"summary": {
  "stance_detail": { "held": 7, "of": 9, "basis": "our_paper" }   // held-rate behind historical_stance (resolves Appendix A.3)
}
```

- `negotiation_trail` surfaces the compiler's per-round diffs instead of
  discarding them: each entry cites the post-move state (a §4 citation
  that MUST resolve). Together with `outcome: "proposed_then_reversed"`,
  the trail lets a consumer distinguish deal-breakers from trading chips
  **without a prescriptive label** — the dynamics stay descriptive, per
  §2.2.
- `stance_detail` is the count pair behind the `historical_stance` enum
  (`held` of `of` opportunities, on the stated `basis`); a validator
  rejects `held > of` or inconsistency with the enum bucket.

### 3.6 `posture` (NEW) — negotiation intent as generated prose

The Posture is the "smart, system-prompt-style directions" layer: a prose block that tells the review engine *how to negotiate this agreement type*, generated by the compiler from a short interview (§7) and grounded in the Evidence.

```jsonc
"posture": {
  "system_prompt": "This is a generally low-risk agreement type; default toward ACCEPT. We typically go two negotiation rounds before escalating. Hold firm on the liability cap and on declining indemnification (see Floor). Term, notice periods, and renewal mechanics are flexible to close. Write rationale tersely for a GC audience…",
  "version": 1,                            // governed counter (NEW, issue #156) — bumped each re-run of the interview
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
1. `system_prompt` is **legal behavior**. The OPF carries the *initial, compiler-generated* version (the genesis record). A consumer that lets a human edit the Posture MUST treat each edit as a governed version bump (§8) — versioned, diffed, re-approved, rollback-able — never a free-text field that silently changes production behavior. The producer itself already versions its own genesis Posture: `version` starts at `1` and is incremented by 1 each time the compiler's interview (§7) is re-run against an existing Posture (issue #156) — this is the OPF-side half of "governed"; a consumer's edit-time versioning (§8) picks up from there.
2. The `generation.interview` record is **provenance**: it lets an auditor see *why* the Posture says what it says. It MUST be retained.
3. The Posture MUST NOT restate or contradict the Floor. The Floor is the authority on red lines; the Posture may *reference* it ("hold firm on X, see Floor") but a Floor invariant binds regardless of Posture text (§5). Per issue #156's decided direction (2026-07-10): a Posture that appears to soften language around a Floor-protected concept is a **SHOULD-warn** (judgment-first, non-blocking) that a conformant validator surfaces for human review — not a hard validation error.
4. The Posture is **soft** at runtime (§5): it shapes the model's judgment; it is not a gate and cannot, by itself, force or suppress a decision the way a Floor rule does.

### 3.7 `floor` (NEW) — the red lines

The Floor is the small, explicit set of things that must **never** slip — un-overridable by the model under review and by the Posture. It absorbs what an earlier design split out as a separate "review-overlay"; here it is a section of the one knowledge artifact, authored by the legal owner.

A Floor entry is a **natural-language invariant**: one checkable statement a judge can evaluate against a clause in isolation. There is no lexical detector grammar — an earlier draft's term-matching design was superseded (2026-07-09, #145) because real red lines ("never accept uncapped liability") are semantic, not lexical.

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

**Normative rules:**
1. **Evaluation is judgment; coverage and consequence are deterministic.** A dedicated Floor judge — separate from the model doing the review — evaluates each invariant against the clause under review and returns `clear`, `violation`, or `needs_review`. The consumer's deterministic obligations are the **coverage gate** (every invariant present MUST be evaluated and logged on every review; an unevaluated invariant fails the run, fail-closed) and the **consequence rule** (a `violation` verdict forces the negotiation-unacceptable outcome; `needs_review` forces human escalation; the model under review and the Posture can override neither).
2. One Floor, both paper contexts: the same invariants are judged whether reviewing our paper's diff or the counterparty paper's extracted clauses — not two modes.
3. The invariants are OPTIONAL as a section (a corpus-only compile ships zero invariants) but never fabricated: the engine MUST NOT derive active invariants without sign-off.
4. The compiler MAY **propose** Floor candidates (every `outcome: proposed_then_reversed` in the Evidence is a candidate red line). The legal owner finalizes and signs. A compiler MUST NOT auto-promote a candidate to an active invariant without sign-off.

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

- `version_files[].sha256` addresses the **original source file bytes**
  (the staged input, pre-extraction), keyed by the same inferred ordinal
  citations use. One entry per mined version; failed-ingest versions stay
  visible in the producer's ingest record instead.
- **Publication rule:** hashes of confidential files leak nothing, so
  `version_files` (and `snapshot`) are publication-safe. `source_uri` — a
  path/URI into someone's DMS — is NOT, and the public export profile
  strips it.
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
  "section_digests": { "evidence": "sha256:…", "posture": "sha256:…", "floor": "sha256:…", "curation": "sha256:…" }
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
  *whole document*, excluding three things so the hash is neither
  self-referential nor perturbed by non-content run/curation metadata:
  the `identity` object itself (it is where `content_hash` is written),
  `compiler.generated_at` / `compiler.run_id` (wall-clock/run-id, not
  content), and the `curation` object (§3.11 — an attorney pin surviving a
  recompile, or a conflict flag being raised/cleared, is not itself a change
  to the corpus-derived content). Two compiles of byte-identical corpus
  content hash identically regardless of when, under what run, or with what
  curation pins they were produced.
- **`section_digests`** — `sha256:` + hex digest of each of `evidence` /
  `posture` / `floor` / `curation`'s own canonical bytes, computed
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

### 3.11 `curation` (NEW, OPTIONAL) — embedded attorney-pinned positions

```jsonc
{
  "pins": [
    {
      "clause_id": "clause.governing_law",  // evidence.clauses[].id this pin applies to
      "item_id": "C3",                      // viewer item number at pin time — informational
      "position": "consistently_held",      // attorney-asserted position (free-form)
      "baseline_stance": "no_signal",        // historical_stance for this clause AT PIN TIME
      "pinned_at": "2026-07-10T00:00:00Z",
      "pinned_by": "…",                      // OPTIONAL
      "comment": "…",                        // OPTIONAL
      "conflict": {                          // OPTIONAL — set/cleared on recompile
        "flagged_at": "2026-08-01T00:00:00Z",
        "recomputed_historical_stance": "usually_conceded",
        "reason": "historical_stance changed from 'no_signal' to 'usually_conceded' since this position was pinned"
      }
    }
  ]
}
```

Makes the viewer feedback loop real: an attorney's pinned position is
EMBEDDED here (not a sidecar file), so it survives a recompile and a
consumer can treat it as authoritative over the recomputed
`summary.historical_stance` for that clause.

- A pin's `baseline_stance` records the clause's `historical_stance` at the
  moment the pin was made — what the attorney was overriding *from* — not
  the attorney's own `position`. This is deliberate: a pin usually asserts a
  position that already differs from the corpus rollup, so comparing the
  *recomputed* stance against `position` would raise a "conflict" on every
  single recompile even when nothing about the underlying evidence changed.
- On each recompile, the engine's merge layer preserves every pin and
  recomputes `conflict` deterministically (no judge/LLM call — a live
  version is future work): cleared when the freshly recomputed
  `historical_stance` still matches `baseline_stance` (evidence unchanged),
  set when it no longer does (evidence moved since the pin was made). The
  pin's `position` is never silently overridden either way.
- `curation` is OPTIONAL at the top level (absent until the first pin
  exists) and is excluded from `identity.content_hash` — see §3.10 — but
  digested separately as `identity.section_digests.curation` so a consumer
  can still track its lineage independently.

### 3.12 `digest` (NEW in 0.3, OPTIONAL) — the compact model-facing projection

A full OPF document carries every observation's `full_text` and can run to
millions of characters on a real corpus — far beyond a model context. The
`digest` section is the compact projection designed to BE the system-prompt
payload of a consuming review application. Per clause: `taxonomy_id`, `title`,
`historical_stance` + `stance_detail`, `our_standard`, the `acceptable_if`
entries verbatim (`preferred_variations`), and three deduplicated observation
lists — `concessions` (from `fallbacks`), `unacceptable` (from `rejected`),
and `exemplar_forms` (from `observed_positions`) — each grouped by normalized
text with an observation count `n` (precedent-count-weighted), a frequency
`band` (`often` n≥10 / `sometimes` 2–9 / `rare` 1), capped at the top 5
groups by `n` plus every material-risk group, and carrying an `example_ref`
citation resolving into the full playbook for on-demand drill-down.

Normative rules:

- The digest MUST NOT contain `full_text` anywhere — drill-down goes through
  `example_ref` against the full document.
- When present, `digest.clauses[].id` MUST exactly match
  `evidence.clauses[].id` (same set).
- `digest_version` (string, required) versions the digest shape itself,
  independent of `opf_version`.
- The digest is a pure function of `evidence` and participates in
  `identity.content_hash` like any other content section.
- OPTIONAL in the schema; the reference compiler always emits it. Producers
  targeting ~40K tokens (chars/4) satisfy the intent.

The single-file bundle artifact (`playbook.opf.html`, produced by
`playbook view bundle`) embeds the canonical OPF JSON and the digest in
`<script type="application/json">` blocks (ids `opf-canonical`/`opf-digest`,
with `</` escaped as `<\/`; JSON parsing restores the value). The bare
`playbook.opf.json` remains the canonical artifact — a consumer extracts the
block and verifies `identity.content_hash` over the canonical serialization.

## 4. Citations

Every asserted clause text MUST be traceable.
```jsonc
{ "document_id": "string", "version": 4, "clause_path": "8.1", "char_span": [start, end] }
```
- `document_id`, `version`, and `clause_path` are REQUIRED; `char_span` is optional.
- `clause_path` is the dotted clause numbering in the normalized document, not the raw PDF page.
- `char_span`, when present, indexes into the document's full normalized text (document-relative — same coordinate system as `ClauseNode.char_span` in the clause-tree artifact), not the clause's own text.
- `version` is the inferred ordinal (1-based); `"template"` is reserved for the baseline. Every citation's `(document_id, version)` MUST resolve against `corpus.documents` — dangling citations are non-conformant. When the cited document publishes `version_files` (§3.8), the cited version MUST have an entry there — a citation naming bytes no consumer can verify is likewise non-conformant.

### 4.1 Resolution algorithm (NEW)

A consumer holding the corpus files resolves a citation to verified bytes
without the compiler's workspace:

1. Read the citation's `(document_id, version, clause_path, char_span)`.
2. Look up `corpus.documents[document_id].version_files` and select the
   entry whose `version` matches; its `sha256` is the content address of
   the exact source file the compiler read. (For `"template"`, the address
   is `baseline.template_ref.sha256`.)
3. Locate a file in the consumer's own corpus copy whose bytes hash to
   that address — the hash, not any filename or directory layout, is the
   key. No match means the consumer's copy differs from the compiled-from
   corpus: fail loud, do not fall back to a near-name file.
4. Open the verified file and navigate by `clause_path` (dotted numbering
   in the normalized document) and, when present, `char_span`.

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

After compiling Evidence, the producer runs a short structured interview (the answers it cannot derive from the corpus — *forward-looking intent*) and drafts `posture.system_prompt` grounded in the Evidence. The canonical starter set (3–6 questions; a producer MAY prune or extend):

1. **Rounds** — "How many negotiation rounds do you typically go on this agreement type before escalating or walking?"
2. **Leverage** — "What's your default leverage posture? (take-it-or-leave-it standard form / collaborative / we usually need the deal more than they do)"
3. **Risk appetite** — "When a counterparty change is non-material, do you default to accept-to-close, or hold the line?"
4. **Sacred clauses** — "Which clause types are non-negotiable regardless of deal value?" *(seeds Floor candidates — confirmed/signed by the author, never auto-promoted.)*
5. **Flexible clauses** — "Which clause types are you happy to concede to move a deal?"
6. **Deal-size sensitivity & audience** — "Does your posture change above a deal-value threshold? Who reads the output — a GC who wants terse rationale, or a junior reviewer who needs it explained?"

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

- A **conformant playbook** validates against its version's schema — `schema/playbook.schema-0.2.json` for 0.2, `schema/playbook.schema-0.3.json` for 0.3 — and obeys the normative rules in §2.2, §2.3, §3.4, §3.6, §3.7, §3.8, §3.12, and §5. (Validators dispatch on the document's `opf_version`: `"0.1"` documents validate against `schema/playbook.schema.json`, `"0.2"` against `schema/playbook.schema-0.2.json`, `"0.3"` against `schema/playbook.schema-0.3.json` — one validator, three schemas, as `playbook_engine/validator.py` implements.)
- A **conformant producer** emits conformant playbooks, records out-of-scope documents, drafts the Posture from a recorded interview, and only *proposes* (never auto-promotes) Floor candidates.
- A **conformant consumer** honors the determinism boundary (§5), honors provenance (§2.3), surfaces confidence (§9) and citations (§4), loads only pinned+verified composed modules (§3.4), and records the lineage fields (§8).

### 10.1 Vendor extensions (`x_*`)

The schema reserves an `x_*` prefix for vendor extensions at designated
levels (the document root, `evidence.clauses[]` and their observations,
`clause_library[]` entries, `posture`, `floor` and its invariants,
`curation.pins[]`, and `corpus.documents[]` entries). Extensions are not
permitted where hash integrity or mechanical resolvability depends on a
closed shape (`identity`, citations, `agreement_type`, `taxonomy` entries,
`compiler`).

- Conformant consumers MUST ignore unknown `x_*` fields.
- Producers MUST NOT put normative behavior behind `x_*` fields — a
  playbook stripped of every `x_*` field must mean the same thing.
- `x_*` fields ARE content: they participate in `identity.content_hash`
  and the section digests, so two documents differing only in an `x_*`
  value are different playbooks.

## 11. Versioning & migration

OPF uses semantic versioning. `opf_version` is required. Pre-1.0 minor versions may break compatibility.

**Immutability rule (normative, effective 2026-07-16):** a published
`opf_version` is immutable once a consumer exists. Shape or *semantic*
changes — including changes to what existing fields mean or how their
contents are selected — get a NEW `opf_version`, never an in-place edit of a
released schema; "old data still validates" is not sufficient. The `digest`
section's `digest_version` follows the same rule independently. Every
spec-affecting change gets an entry in `spec/CHANGELOG.md`, whose schema-hash
pins are CI-enforced (`tests/test_spec_consistency.py`). OPF 0.3 is frozen as
of `digest_version` 2.

**0.1 → 0.2 migration:**
- `clauses` and `clause_library` move under a new `evidence` object.
- `clauses[].rollup` → `clauses[].summary`; `rollup.position` (enum) → `summary.historical_stance` (descriptive enum, §2.2). `acceptable_if` / `fallbacks` / `rejected` / `confidence` carry over unchanged.
- New top-level: `composes` (§3.4), `perspective` (§3.1), `de_minimis` (§3.1), `posture` (§3.6), `floor` (§3.7), `identity` (§3.10), `curation` (§3.11).
- The §2.2 provenance rule now references `historical_stance` instead of `rollup.position`.

**0.2 → 0.3 migration:**
- Purely additive: a new OPTIONAL top-level `digest` section (§3.12) — the
  compact model-facing projection of `evidence`. A 0.2 document is a valid
  0.3 document once its `opf_version` is bumped (and its `content_hash`
  recomputed if a digest is added). Validators MUST accept both 0.2 and 0.3;
  schema: `playbook.schema-0.3.json`.

## Appendix A — Open questions (for review, not yet decided)

1. **Composition mechanics (§3.4).** The governance contract is fixed; the *module interface* is not. Needs one investigation pass against a real consuming application's module structure before composition alters runtime behavior. Until then, `composes` is recorded for lineage but inert.
2. **Floor minimality (§3.7.1).** The admission test is the guard against the Floor regrowing into a rigid playbook. Worth pressure-testing on a mature production Floor: how many of its accumulated hard rejections survive *both* (a) categorical and (b) statable as a single judge-checkable invariant? The ones that fail (a) should become Posture; the ones that fail (b) should become Evidence guidance.
3. **`historical_stance` vs. a numeric tendency.** ~~`mixed` is coarse. A future version might carry a held-rate (e.g. "held in 7 of 9 our-paper deals") instead of/alongside the enum. Deferred.~~ **Resolved:** `summary.stance_detail` carries the held-rate alongside the enum — see §3.5.3.

## Appendix B — Changelog
- **0.3** — `digest` section (§3.12): compact model-facing projection of `evidence` (stances, preferred variations verbatim, text_summary-only concession/unacceptable summaries, frequency-banded deduplicated exemplar forms with `example_ref` drill-down); single-file bundle artifact `playbook.opf.html` embedding the canonical JSON + digest; additive over 0.2.
- **0.2** — Three-section model (Evidence / Posture / Floor); `historical_stance` (descriptive) replaces `rollup.position` (prescriptive); `composes` (pinned external modules); determinism boundary (§5); producer/author/consumer responsibilities (§6); Posture interview (§7); lineage boundary with the consumer (§8); `identity` — canonical serialization, `content_hash`, per-section digests, producer-assigned `id`/`version`/`supersedes` (§3.10, issue #143); `curation` — embedded attorney-pinned positions surviving recompile with deterministic conflict-flagging (§3.11, issue #147).
- **0.1** — Initial draft. Risk-delta model, provenance rule, dual structure (clause positions + clause library), citation requirement, taxonomy curation model.
