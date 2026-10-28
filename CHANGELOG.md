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
| `playbook.schema-0.6.json` | `b2a925811fd661c5d7c11eaa768444fa63764b479c34fff9a50187c580668bf2` |
| `spec/conformance/0.6/` (manifest.json + vectors/*.json, concatenated) | `c5e3828e7d59c961aca2d633b390b3ea536902e7818efe0dd778a0916cf8be96` |

Current `DIGEST_VERSION`: **5** (`playbook_engine/digest.py`) — the only
digest the engine builds.

## History

### 2026-10-21 — OPF 0.6 / digest 5: verified absence counts (issue #251)

In-place change to the unfrozen `playbook.schema-0.6.json` (epic #245; 0.6 has
no consumer yet); `digest_version` stays "5". A deal with no precedent record
for a clause is no proof it signed without it (the clause may be unclassified),
so absence is recorded as two facts. Facts only.

- **`evidence.absence`** (required; `$defs.absence`, `absenceRecord`,
  `absenceCandidate`): `guard` and `records[]`, one per signed deal and clause
  type with no signed text, with `status` (`absent_verified` or
  `possibly_present`), `basis` (`scored` or `unverifiable`), `n_checked` and up to
  three `candidates` (`ref`, `score`, `filed_under`: the clause type the classifier
  filed the node under, or null).
- **Digest**: required `n_signed_without_verified`, `n_signed_without_possible`
  and `verified_absent_deals` on each clause (`$defs.digestClause`,
  `digestAbsentDeal`); conformance vectors 001-013 are re-stamped and vector 014
  is new.
- **New validator MUSTs**: the records are complete (exactly the in-scope signed
  deals crossed with the clause types, less the pairs with a `signed_text`), ids
  recompute from `(agreement_type.id, document_id, taxonomy_id)` and are unique,
  one record per pair, a record names a clause and a signed deal, `status`
  follows `basis` and `candidates`, candidates are ordered best score first, none
  scores below `guard` and each cites its own deal and resolves (§4).

### 2026-10-21 — OPF 0.6 / digest 5: reconstructed former forms (issue #250)

In-place change to the unfrozen `playbook.schema-0.6.json` (epic #245; 0.6 has
no consumer yet); `digest_version` stays "5". When the owner supplies no older
form files, the producer rebuilds our former forms from the corpus' first drafts
on our paper and flags them. Facts only: no reason for a change is claimed.

- **`baseline.prior_templates[]`** gains optional `basis` (`supplied`, the
  default when absent, or `reconstructed`), `n_deals` (>= 1) and
  `first_signed` / `last_signed` (`YYYY-MM`); the last three are for a
  reconstructed form only. A reconstructed form has no `sha256`,
  `in_use_from` or `in_use_until`.
- **`prior_standards[].source_ref`** of a reconstructed form is an ordinary
  corpus citation (a `corpus.documents[]` document and version), never
  `prior_template`.
- **Digest**: required `basis` on each `standard_history.forms[]` entry
  (`$defs.digestStandardForm`); conformance vector 012 is re-stamped.
- **New validator MUSTs**: a reconstructed form carries no `sha256` or owner
  dates and a supplied form no `n_deals` / `first_signed` / `last_signed`; a
  reconstructed form's prior standards cite a corpus document; its `n_deals`
  equals the distinct deals whose precedent records name it in
  `opened_with_form` and its `first_signed` / `last_signed` the earliest and
  latest signing month among them; no refused ask of a clause type has the
  grouping key of a reconstructed form's prior standard of that clause type
  (our former wording struck before signing is counted under
  `removed_prior_standard`, as for a supplied form); supplied and reconstructed
  forms never appear together. A deal reads as opened on a reconstructed form
  by the same exact first-draft check as a supplied one, on any paper side.

### 2026-10-21 — OPF 0.6 / digest 5: prior standards (issue #249)

In-place change to the unfrozen `playbook.schema-0.6.json` (epic #245; 0.6 has
no consumer yet); `digest_version` stays "5". The clauses of OUR former forms
(`baseline.prior_templates`, issue #246) are recorded as ours and judged once
against today's standard. Facts only: no reason for a change is ever claimed.

- **`baseline.prior_templates[]`** (optional): `{label, in_use_from?,
  in_use_until?, sha256?}` for each supplied form, in config order.
  `$defs.inUseDate` (`YYYY`, `YYYY-MM` or `YYYY-MM-DD`).
- **`evidence.clauses[]`** gains required `prior_standards[]`
  (`$defs.priorStandard`: `form_label`, `text`, `parts[]` with the
  `our_standard.parts` rule, `source_ref`, `vs_standard`), required
  `why_changed` (`{text, basis: "owner"}` or null) and the required counts
  `n_opened_prior`, `n_kept_prior`, `n_moved_to_current`.
- **`evidence.precedent[].opened_with`** gains `prior_standard`; optional
  `opened_with_form` (present exactly when it is `prior_standard`);
  `opening_text` is recorded for it like for `standard` and `non_standard`.
- **Digest**: required `standard_history` per clause
  (`$defs.digestStandardHistory`: `forms[]` of `$defs.digestStandardForm`,
  `why_changed`); an opening on a former form is never a changed opening.
- **New validator MUSTs**: `baseline.prior_templates` labels are unique;
  `prior_standards[].form_label` and `opened_with_form` name a supplied form
  and a clause has one entry per form; `opened_with_form` is present exactly
  when `opened_with` is `prior_standard`; a prior standard's `text`, `order`
  and `part_id`s are those of its parts; its `vs_standard` is null when the
  form reads as today's standard; the three new counts equal what `precedent`
  implies. The digest is recomputed whole as before.
- **A former form is cited in its own bytes.** `$defs.citation.version`
  gains `prior_template`. A prior standard's `source_ref` (and its digest copy,
  `standard_history.forms[].ref`) is `{document_id: "prior_template:<label>",
  version: "prior_template", clause_path, char_span?}`, resolved through that
  form's `baseline.prior_templates[].sha256` (OPF-SPEC §4, §4.1), never
  `template`, which always resolves to today's form. New validator MUSTs: a
  prior standard cites its own form, and a former form's citation names a
  listed form that carries its `sha256`.
- **Per-deal form, per-form dates.** `opened_with_form` is decided per deal
  (among the forms a clause matches, the one the most of the deal's clause
  types match, then config order), and a form's inferred in-use dates come
  from every deal that opened on it in any clause, so they are the same in
  every clause's `standard_history`. Description wording only; no shape change.
- Conformance vectors regenerated (every clause gains the empty
  `prior_standards`, `why_changed` and the counts; every digest clause an empty
  `standard_history`); new vector 012 pins three forms, the three outcomes of a
  deal that opened on one, an unchanged form, the three kinds of in-use dates
  and the owner's note.

### 2026-10-20 — OPF 0.6 / digest 5: how settled each judgment is (issue #248)

In-place change to the unfrozen `playbook.schema-0.6.json` (epic #245; 0.6 has
no consumer yet); `digest_version` stays "5". No change to the evidence: the
new value is derived from `vs_standard.basis` and `vs_standard.check`, never
stored.

- **`settled`** — `agreed` | `adjudicated` | `unchecked` | `owner`: `owner`
  when `basis` is `owner` (whatever its `check` says), else `adjudicated`
  when `check.adjudicated` is true, `agreed` when `check.agreed` is true,
  `unchecked` otherwise (no check, or a disagreement awaiting adjudication).
- **Digest**: required `settled` (the value, or null when the text is not
  judged) on `digestVariant`, `digestRefusedAsk` and `digestChangedOpening`;
  required `settled_counts` (`{agreed, adjudicated, unchecked, owner}`,
  summing to `n_texts`) on the collapsed `digestEquivalentVariants` entry;
  `part_map_legend` gains a required `settled` object stating the four
  values.
- **Dossiers**: a dossier excerpt's non-null `vs_standard` gains a required
  `settled`.
- **Validator**: the digest and the dossiers are already recomputed from the
  document and compared whole, so a wrong or missing `settled` /
  `settled_counts` is rejected with them.
- Conformance vectors regenerated (every judged digest entry and excerpt now
  carries `settled`); new vector 011 pins all four outcomes on each carrier, a
  disagreement awaiting adjudication read as `unchecked`, an owner judgment
  (which carries no check) read as `owner`, and the collapsed entry's counts.

### 2026-10-18 — OPF 0.6 / digest 5: signing dates (issue #247)

New schema file `playbook.schema-0.6.json` replaces `playbook.schema-0.5.json`
(deleted) and `spec/conformance/0.6/` replaces `0.5/`, following the 0.5
pattern; the reference engine, validator, `playbook precedent` and
`find_precedent()` support 0.6 only, and a 0.5 document is rejected as an
unsupported `opf_version`. `digest_version` is "5".

- **`corpus.documents[].signed_on`** (`YYYY-MM-DD`, `YYYY-MM` or null) and
  **`signed_on_basis`** (`document_text` | `signature_block` | `filename` |
  `file_metadata` | null; non-null exactly when `signed_on` is), both required
  on every in-scope document (null when the date is unknown, so "never
  recorded" is distinguishable from "unknown") and absent on an out-of-scope
  document. `$defs.signedOn` /
  `signedOnOrNull` (pattern `^\d{4}-\d{2}(-\d{2})?$`).
- **`evidence.precedent[].signed_on`** is required (nullable) and replaces the
  optional `signed_at`; the `YYYY-Qn` quarter form is removed.
  `provenance_index.documents[].signed_at` is now `signed_on`.
- **Digest 5**: `first_signed` and `last_signed` (`YYYY-MM`,
  `$defs.digestMonth` / `digestMonthOrNull`) on every signed variant, the
  collapsed equivalent entry, every refused-ask group and every clause;
  `digest.corpus.first_signed` / `last_signed` are months too. Ordering keys
  that used `signed_at` use `signed_on` (a month sorts before every day in it).
- **New validator MUSTs**: both keys are present on every in-scope document
  (null when unknown); `signed_on` is a real calendar date or month with a
  year in 1980-2099; `signed_on_basis` is present exactly when `signed_on` is
  non-null and is one of the four bases; a document with `signed_version` null
  has no `signed_on`; a precedent's `signed_on` equals its document's and is
  null when `signed` is false.
- Conformance vectors regenerated (dates in 002, 003 and 008 are `signed_on`;
  vector 002's quarter becomes a month); new vector 010 pins every basis, both
  precisions, a signed deal with no date, an unsigned deal and the months of
  the corpus, a clause, a signed variant and a refused ask.

### 2026-10-10 — OPF 0.5: `vs_standard` part by part, derived label (issue #243)

In-place change to the unfrozen `playbook.schema-0.5.json` (epic #236; 0.5
has no consumer until contract-opf/contract-toaster#128 vendors it, which
freezes it; the owner's 2026-10-10 direction is to change the format now when
that serves the consumer). `digest_version` stays "4"; its shape changes in
place.

- **`our_standard.parts[]`** (required on a non-null standard): `{part_id,
  order, text}` per template passage of the clause type, in order; `text` is
  the parts joined with a blank line. `part_id` is the first 8 hex chars of
  `sha256` over the part's §3.5.4 grouping key (with `perspective.party`),
  `-2`, `-3`, ... on a repeat within one standard.
- **`vsStandardVerdict`**: gains required `parts[]` (each `present` with a
  `relation` or `omitted` with an `effect`, `reason` only when not
  equivalent/neutral) and `additions[]` (`{summary, effect}`); `label` gains
  `mixed` and is DERIVED from the parts and additions (OPF-SPEC §3.5.6);
  optional `owner_label {label, reason}` (basis `owner` only) overrides it;
  `check` gains required `label_agreed` and `parts_agreed` (`agreed` is
  both). The cache key's last field is the ordered list of part grouping
  keys instead of the joined standard.
- **New validator MUSTs**: the label equals the derived label (or
  `owner_label.label`); `parts` judges every standard part exactly once, in
  order, and names no other; a present part has a relation; `owner_label`
  needs basis `owner`; `check.agreed` = `label_agreed and parts_agreed`;
  `our_standard.text` is its parts joined and every `part_id` / `order` is the
  computed one; with the verdict store, the whole judgment must be the stored
  one.
- **Digest 4**: `our_standard` is `{parts: [{part_id, text}], source_ref}`
  (the joined text is not repeated); top-level `part_map_legend` (the tokens
  and the `label_by` values); every variant, refused ask and changed opening
  gains `part_map`, `additions` (tokens) and `reason` (all three null when
  unjudged), and `label_by: "owner"` only when `owner_label` is set (`reason`
  is then the owner's; a label derived from the parts, the default the legend
  states, carries no `label_by`); `positions` gains `mixed`;
  signed variants order `less_protective` and `mixed` first, then
  `different_concept`, unjudged, `more_protective`, the collapsed equivalent
  entry.
- **Dossiers**: `our_standard` is `{parts: [{part_id, text}]}`; every excerpt
  gains `vs_standard` (`{label, label_by, reason, part_map, part_reasons,
  additions}` or null; `reason` the owner's when `label_by` is `owner`).
- Conformance vectors regenerated (vectors 001-008 have one-part standards;
  vector 007's changed opening is `mixed`; vector 008's excerpts carry
  `vs_standard: null`). New vector 009 pins a standard of four parts with a
  repeated passage (`<hash>-2`), part maps of four tokens with omissions and
  additions, dossier part reasons naming parts other than the first, and an
  owner label carried as `label_by: "owner"` with the owner's reason.

### 2026-10-09 — OPF 0.5: `our_standard` is the complete template clause (issue #242)

In-place semantic change to the unfrozen `playbook.schema-0.5.json` (epic #236;
0.5 has no consumer until contract-opf/contract-toaster#128 vendors it, which
freezes it). No schema shape changes; `digest_version` stays "4".

- **`our_standard.text`** (and the digest copy of it) is every template clause
  of the type joined in document order, no longer the first one alone. A
  form's fill-in cover table (bracketed blanks, no operative sentence, ahead
  of the form's first operative clause) is front matter and is never a
  standard. It still decides a struck clause's origin and the `standard`
  fact, as every template clause does.
- **`our_standard.source_ref.clause_path`** of a standard joined from several
  template clauses lists their paths, comma separated (`"2, 5, 7"`; `?` for an
  unnumbered clause), and the reference carries no `char_span`. A one-clause
  standard is unchanged. OPF-SPEC section 4 and 4.1 define the list form.
- A consumer that resolves `source_ref` must accept the list form.
  Equivalence keys include the standard, so `vs_standard` labels are
  re-derived for every clause whose standard changed.

### 2026-10-09 — OPF 0.5: hard-rule manifest, critic dossiers, provenance index (issue #228)

In-place addition to the unfrozen `playbook.schema-0.5.json` (epic #236; 0.5
has no consumer until contract-opf/contract-toaster#128 vendors it, which
freezes it). `digest_version` stays "4".

- **New optional top-level sections** `manifest`, `dossiers` and
  `provenance_index` (OPF-SPEC section 3.12.3): `manifest.hard_rules[]`
  `{rule_id, clause_id, taxonomy_id, statement, required_presence, condition,
  permissible_proof, fallback_language}`, one per `floor.invariants[]` entry;
  `dossiers` a map from clause id to a dossier `{clause_id, taxonomy_id, title,
  our_standard, n_floor_rules, floor_rules[], excerpts[], n_omitted,
  omitted_precedent_ids}` of at most two excerpts `{precedent_id, kind,
  opening, signed, outcome}` within a budget (below), no text ever cut; `provenance_index` `{compiler,
  corpus_manifest_hash, documents[], dossiers}`, where `dossiers` maps a clause
  id to the `{precedent_id, document_id, kind, omitted}` of each excerpt the
  dossier selected, kept (`omitted` false) then dropped for size (`omitted`
  true), and `documents` lists every deal behind those rows, so each
  `omitted_precedent_ids` entry resolves to its deal.
- **Floor invariants** may carry the vendor keys `x_required_presence`,
  `x_condition` (a `required_phrases`, `numeric_bound` or `cross_reference`
  predicate spec, or `"judged"`) and `x_permissible_proof`, besides the existing
  `x_taxonomy_id`. An invariant with `x_required_presence` true or a predicate
  `x_condition` MUST carry `x_taxonomy_id` (a rule with no clause cannot be
  enforced outside both models); the schema's `hardRule` likewise requires a
  string `taxonomy_id` for such a rule. `floor.invariants[]` is otherwise
  unchanged.
- **New normative validator rules:** a present `manifest`, `dossiers` or
  `provenance_index` MUST equal its reference construction over the document
  (`playbook_engine/dossiers.py`, vector 008); every dossier MUST carry at most
  two excerpts and be within its budget unless it holds exactly one excerpt
  (one with two excerpts, or none, over its budget is refused); a
  malformed `x_required_presence`, `x_condition` or `x_permissible_proof`, or
  a presence or predicate rule naming no clause, is an error. A writer that changes the
  Floor MUST re-derive the sections.
- **Dossier construction** (normative through that recomputation): each
  excerpt shows one record of its group, the latest `signed_at` then the
  lowest `precedent_id` (a concession's record taken only among the members
  that opened with our standard). **No text is ever cut part-way**: every
  excerpt text, our standard and each listed Floor rule is whole, because a
  cut-off clause can lose its carve-out or cap and so plant a false fact. The
  budget is `max(1000, 3 * tokens(our_standard.text))` (1,000 with no
  standard); over budget, whole parts are dropped: first the listed Floor
  rules, last first, while the dossier holding only its first excerpt is
  still over budget (`n_floor_rules` still counts them all), then excerpts in
  reverse selection order, keeping the first, until it fits. Our standard and
  the identifiers are never dropped. A dossier always keeps one complete excerpt
  (when the clause has any), even if that alone puts it over budget: that is
  the only case where a dossier may exceed its budget. Each dossier names the
  excerpts it dropped: `n_omitted` and `omitted_precedent_ids` (sorted).
- **Conformance:** `spec/conformance/0.5/` `expected` gains `manifest`,
  `dossiers` and `provenance_index` in every vector; vector 008 is new and
  also pins a refused-ask excerpt, a presence rule naming its clause, a dropped
  and named excerpt, a single over-budget excerpt kept whole with its Floor
  rule dropped, a Floor rule dropped whole from a dossier with no excerpt so
  that it fits, and a provenance index with a corpus snapshot hash,
  source-file hashes and the dropped excerpt indexed as `omitted` with its
  deal.
- **Unchanged:** the digest, the precedent record, the grouping key,
  canonicalization and `identity` (the sections join `content_hash` like any
  other top-level content).

### 2026-10-09 — OPF 0.5: digest 4 (issue #234)

In-place change to the unfrozen `playbook.schema-0.5.json` and a new
`digest_version` "4" that **replaces digest 3** (epic #236; 0.5 has no consumer
until contract-opf/contract-toaster#128 vendors it, and digest 3 never
shipped to one). The digest-3 construction is deleted; `spec/conformance/0.5/`
is regenerated for digest 4 rather than kept beside a digest-3 set.

- **`digest.clauses[]`** gains `n_opened_standard`, `n_kept_standard`,
  `positions` `{standard, equivalent, more_protective, less_protective,
  different_concept, unjudged}`, `changed_openings[]` `{text, n_deals,
  n_to_standard, n_struck, label, ref, precedent_ids}` and
  `n_changed_openings_total`. Each `signed_variants[]` entry gains
  `n_from_standard`, `n_unchanged` and `label`; each `refused_asks[]` entry
  gains `label`. The signed variants labelled `equivalent` (#240) collapse
  into one entry `{label, n_deals, n_texts, n_from_standard, n_unchanged,
  last_signed, exemplars (at most two {text, ref}), precedent_ids}`, listed
  last; `less_protective` and `different_concept` come first, then unjudged,
  then `more_protective`; the cap applies after collapsing. The top level
  gains `uncovered_clause_types` `[{taxonomy_id, label}]`.
- **`evidence.clauses[]`** gains the required counts `n_opened_standard`,
  `n_kept_standard` and `n_changed_openings`, derived from
  `evidence.precedent` like the existing `n_*` counts.
- **New normative validator rules:** the digest MUST equal `build_digest_v4`
  over the document; the three new clause counts MUST equal what
  `evidence.precedent` implies.
- **Unchanged:** the grouping key, precedent ids, the `standard` fact, the
  `vs_standard` field and its rules, canonicalization and `identity`.

### 2026-10-09 — OPF 0.5: the `vs_standard` equivalence label (issue #240)

In-place addition to the unfrozen `playbook.schema-0.5.json` (epic #236; 0.5
has no consumer until contract-opf/contract-toaster#128 vendors it, so
nothing published binds the previous shape). `digest_version` stayed "3" here;
the digest hook (collapsing equivalent variants, label counts) landed with
digest 4 (issue #234, above).

- **New optional field `vs_standard`** on `evidence.precedent[].signed_text`,
  `.opening_text` and each `.refused_asks[]` entry: `{label, reason, basis,
  check} | null`. `label` is `equivalent` | `more_protective` |
  `less_protective` | `different_concept`, relative to `perspective.party`;
  `reason` is one sentence naming the operative difference; `basis` is
  `judge` | `agent` | `owner`; `check` is `{agreed, by, adjudicated} | null`
  (the independent blind check of the verdict, `by` the checking model's id).
  `null` means "not yet judged", never a guess. A reference producer always
  writes the key (null when nothing was judged); a document that omits it is
  read as null.
- **Owner decision #227 (c) narrowed for this one label only.** Judged
  verdicts stay off the consumer path everywhere else. The label is an index
  the consuming model checks against the cited text, never an instruction.
- **New normative validator rules**, each blocking: a non-null `vs_standard`
  MUST NOT sit on a signed text whose `standard` is true, nor on a text of a
  clause whose `our_standard` is null; `check.adjudicated` true implies
  `check.agreed` false; and, when the run's verdict store is present, every
  non-null `vs_standard` MUST trace to a stored verdict under its cache key
  (sha256 over `agreement_type.id`, `taxonomy_id`, `perspective.party`, the
  text's §3.5.4 grouping key and our standard's) carrying the same label.
- **Reference compiler.** New judge kind `equivalence`
  (`playbook judge` queues one item per distinct eligible text once a
  template is configured; `playbook judge --check equivalence` and
  `playbook judge-apply --check` run the blind check and adjudication with a
  pinned `claude-opus-5-5` / `xhigh` agent); `playbook project` reads the
  verdict store and writes the labels; `playbook scorecard` (shape v4) gains
  an `equivalence` section. Paper side, document id and role play no part in
  the cache key. The NDA example ships canned equivalence verdicts.
- **Unchanged:** the grouping key, precedent ids, the `standard` fact,
  canonicalization, `identity`, and all conformance vectors.

### 2026-10-08 — OPF 0.5: what every clause opened with; 0.5 replaces 0.4 (issue #233)

New schema file `playbook.schema-0.5.json` (`opf_version` "0.5"). **0.4 is
removed from this repository**, not kept beside it: the owner confirmed
(2026-10-06, epic #236) that there are zero installed consumers, and
contract-opf/contract-toaster accepts only 0.2/0.3 today, so nothing ever
bound 0.4. `playbook.schema-0.4.json` and `spec/conformance/0.4/` are deleted
(they stay in git history); the conformance set is regenerated as
`spec/conformance/0.5/` (the four 0.4 documents with the new field, plus
vector 005). The validator, assembler, `playbook precedent` and
`find_precedent()` support 0.5 only. OPF-SPEC §3.5.5 documents it. **0.5 has
no consumer until contract-opf/contract-toaster#128 vendors it;** #234
(digest 4) and #228 (hard-rule manifest, critic dossiers) extend 0.5 before
that, so `digest_version` stays "3" here and nothing publishes in between.

- **Precedent record.** `opened_with` is REQUIRED on every
  `evidence.precedent[]` record: `"standard"` (the first-draft text of the
  clause type is our standard language, the exact check `standard` uses),
  `"non_standard"` (present but not standard, including a clause type with no
  template clause), `"absent"` (the first draft has no node of the clause
  type: added during negotiation), or `null` (not determined: the deal has no
  detected executed copy, or the store predates the field; never `"absent"`).
  `opening_text` keeps its `{text, ref} | null` shape and its place in the
  record, but its meaning widens: it is the first-draft text of the clause
  type (every first-draft node bound into its aligned rows, joined with a
  newline in `char_span` order, citing the first), **whatever its origin and
  whatever paper the deal is on**, recorded when `opened_with` is `standard`
  or `non_standard` and either `signed_text` is null or its §3.5.4 grouping
  key differs from `signed_text`'s. In 0.4 it was recorded only for our
  standard language struck before signing, and an edited clause lost it. The
  record states facts only: held, conceded or moved-to-standard meanings are
  derived by the consumer or the digest, never classified here.
- **`curation` removed.** The optional top-level `curation` section, its
  `curationPin` definition and `identity.section_digests.curation` are gone
  from the 0.5 schema (the review loop that produced pins was retired in
  #239; #239's handoff said 0.5 drops them). OPF-SPEC §3.11 is withdrawn.
  `content_hash` now excludes only `identity` and `compiler.generated_at` /
  `compiler.run_id`, and `compute_section_digests` returns three digests
  (evidence, posture, floor). A 0.5 document carrying `curation` fails
  schema validation. The conformance vectors are regenerated: their
  `expected.section_digests` lose the `curation` key (content hashes are
  unchanged: no vector input carried a `curation` section).
- **New normative validator rules (0.5 only)**, each blocking: `signed`
  false implies `opened_with` and `opening_text` are null; `opened_with`
  `"absent"` or null implies `opening_text` is null; a non-null
  `opening_text` has a grouping key different from `signed_text`'s (or
  `signed_text` is null) and `moved` is true.
- **Reference compiler.** Observations gain an engine-internal outcome
  `opening` and an optional `opened_with`; `opening_text` no longer comes
  from `conceded_before_signing` rows. A struck clause whose origin is
  undetermined is now precedent with `signed_text` null, but is still not
  claimed as a refused ask. `moved` is also true whenever `opening_text` is
  non-null. The precedent id is unchanged (it hashes `signed_text` only).
  `_DEVIATION_VS_TEMPLATE_VERSION` moves to 19.
- **Unchanged:** `digest_version` "3" and its construction, the grouping key,
  canonicalization and `identity`. The digest does not project opening
  evidence (that is digest 4's job, #234). Vector 005 therefore leaves
  `expected.digest` untouched by the new fields.

### 2026-10-07 — retire OPF 0.1–0.3 and digest 2: one format (issue #238)

Owner decision (2026-10-06, epic #236): there are zero installed consumers,
so nothing needs backward compatibility. The engine now emits and validates
exactly one format, OPF 0.4 / digest_version 3.

- **Removed from this repository** (they stay in git history and in
  contract-opf/opf): `playbook.schema.json` (0.1),
  `playbook.schema-0.2.json`, `playbook.schema-0.3.json`, and the
  OPF 0.3 / digest_version 2 conformance set
  (`spec/conformance/manifest.json` + `spec/conformance/vectors/`). Their
  pins are gone from the table above; the history entries below still
  record them. `playbook.schema-0.4.json` and `spec/conformance/0.4/` are
  byte-identical (pins unchanged).
- **Validator.** A document whose `opf_version` is anything but "0.4" —
  including "0.1", "0.2" and "0.3" — gets one blocking "unsupported
  opf_version" error and no other check (previously 0.1–0.3 validated
  against their frozen schemas).
- **Compiler.** `playbook project` has no `--opf-version`; the 0.3
  assembler, the digest_version 2 builder, the clause-library compiler and
  every 0.3-only surface of the clause-position compiler (observed
  positions, rollup, `historical_stance` / `stance_detail`,
  `acceptable_if`, fallbacks, rejected, `negotiation_trail`) are deleted.
  `provenance.min_evidence_n`, which only capped the 0.3 stance, is no
  longer a config key. OPF 0.4 output is unchanged: the NDA reference
  playbook's `evidence`, `digest` and `floor` regenerate byte-identically.
- `docs/OPF-SPEC.md` §3.5.1–§3.5.3 and §3.12's digest 2 rules are replaced
  by a history note. The normative rule change (the validator rejects every
  `opf_version` but "0.4") is recorded under `### Normative rule changes` in
  the root `CHANGELOG.md`.

### 2026-10-05 — OPF 0.4 + digest_version 3: the verdict-free per-deal precedent record (issue #223)

New schema file `playbook.schema-0.4.json` (`opf_version` "0.4"); every
earlier schema file is byte-identical (pins above unchanged) and 0.1/0.2/0.3
documents keep validating against them. `playbook.schema.json` is the frozen
0.1 schema, not a pointer, so it is untouched: the validator's
`_SCHEMA_PATH_BY_VERSION` maps "0.4" to the new file. The reference compiler
emits 0.4 by default (`playbook project --opf-version 0.3` keeps the 0.3
assembler for one release). OPF-SPEC §3.5.4, §3.12.1, §10, §10.2 and §11
document it.

- **Evidence shape.** `evidence` is `{clauses, precedent}`.
  `clauses[]` = `{id, taxonomy_id, title, our_standard, n_deals,
  n_signed_standard, n_variants, n_refused}`; `precedent[]` = one record per
  (deal, clause type): `{id, taxonomy_id, document_id, counterparty_ref?,
  paper ("ours"|"theirs"|"unknown"), paper_basis, paper_confidence, signed,
  signed_at?, rounds, signed_text ({text, ref} | null), opening_text
  ({text, ref} | null), standard, moved, refused_asks[{text, round, ref}]}`.
  No `observed_positions`, `clause_library`, `summary` or
  `negotiation_trail`; no stance, band, risk or deviation verdict.
  `signed_text` is null when the clause was struck before signing (the
  ticket's sketch did not cover that case; the record still needs it, e.g.
  our standard struck — carried as `opening_text`). Precedent and clause
  records are closed (no `x_*`): judged verdicts from an opt-in judged run
  go only under the root `x_judgments` extension, keyed by precedent id.
- **Precedent id.** `"prec." + sha256(canonical([agreement_type.id,
  document_id, taxonomy_id, signed_text.text or ""]))[:16]`.
- **digest_version 3.** `{digest_version, perspective (copied; null when
  absent), agreement_type {id, name}, corpus {n_deals, n_signed,
  first_signed, last_signed}, clauses[{id, taxonomy_id, title, our_standard,
  n_deals, n_signed_standard, signed_variants[{text, n_deals, last_signed,
  ref, precedent_ids}], refused_asks[{text, n_deals, ref, precedent_ids}],
  n_variants_total, n_refused_total}]}`. Grouping (the clause counts and
  the digest) is an exact key over the text and the document alone
  (`precedent.normalize_variant_text`, OPF-SPEC §3.5.4 "Grouping key"):
  every `Counterparty-<n>` registry alias becomes the token `counterparty`,
  then `normalize_for_standard` runs with the document's `perspective.party`
  as its only party name (rewritten to `party`). It is NOT the standard
  check's party-name substitution: that list (`our_party_aliases` +
  `known_entities`) is producer configuration the document does not carry,
  so a validator or port could not recompute a key built from it; and
  without the alias rewrite, deals that signed the same words with
  different counterparties split into per-deal singletons. Our party and
  the counterparty are distinct tokens, so a party-swapped text never
  merges. Variant/ask `text` is the ≤ 300-char sentence-boundary summary;
  lists are capped 5 → 1 to fit the budget, totals always uncapped. No
  `clause_count` key (the clause list's length).
- **New normative validator rules (0.4 only)**, each blocking: precedent ids
  unique and equal to their recomputation; at most one precedent per
  (document_id, taxonomy_id); every precedent's taxonomy_id names a clause
  and its document_id a corpus document, with `signed` agreeing with
  `signed_version`; `standard: true` requires `signed_text`; the clause
  `n_*` counts equal what `precedent` implies; `signed_at` is a real date or
  quarter; a present digest equals `build_digest(document)`. The 0.2/0.3
  §2.2 provenance and evidence-depth rules do not apply to 0.4 (no stance;
  paper side gates nothing).
- **Conformance.** A new, separately stamped vector set
  `spec/conformance/0.4/` (manifest + 4 vectors, pinned above) generated by
  `scripts/generate_conformance_vectors.py --opf-version 0.4`. The 0.3 set is
  byte-identical (its pin is unchanged), and the script's default mode still
  reproduces it exactly — it now stamps that set with `DIGEST_VERSION_V2`
  because `DIGEST_VERSION` is 3.
- **Transforms re-derive.** `playbook publish` and `export_profile` re-derive
  a 0.4 document's precedent ids, counts and digest after rewriting text
  (`precedent.refresh_derived`), so the digest never carries a stale copy of
  pre-transform text or the real `perspective.party`; `publish` coarsens
  `precedent[].signed_at` to `YYYY-Qn` like `observed_at`.

### 2026-09-25 — digest v2 `n` counts distinct deals (issue #216) — OWNER-AUTHORIZED IN-PLACE EXCEPTION

**Deliberate, owner-authorized exception to OPF-SPEC §11 (published-version
immutability) and to this file's versioning policy.** The semantics of the
digest's `n` change in place under `digest_version` **2** — no
`digest_version` bump, no `opf_version` bump, and no schema file changes
(every `playbook.schema*.json` is byte-identical; its pin above is
unchanged). The owner authorized this on 2026-09-25 for issue #216 only.

- **Semantic change:** in every digest list (`exemplar_forms`,
  `concessions`, `unacceptable`, `preferred_variations`), `n` is now the
  number of **distinct deals** — distinct `example_ref.document_id` values
  (for `preferred_variations`: the entry's own `observation_ref` deal plus
  every `observed_positions` deal that SIGNED its `to` text — a deal where
  that text was `proposed_then_reversed` refused it and is not acceptance
  precedent) — in the group.
  It previously summed each member's `precedent_count`. The compiler
  stamps each row of a text with that text's deal count, so the old rule
  reported a text signed in k deals as n = k×k (the NDA example's
  assignment clause showed `n: 16`, band "often", on a 4-deal group). A
  row whose citation carries no `document_id` is non-conforming (every
  playbook schema requires `example_ref.document_id`), so it is never
  counted as a guessed deal: `build_digest` raises `ValueError` on it.
  `band` thresholds are unchanged and now apply to deal counts. The
  `n` description strings in `playbook.schema-0.3.json` (which still say
  `n` counts observations sharing the same normalized text,
  precedent_count-weighted) are superseded by this entry; they stay
  unchanged only because that schema file is frozen.
- **Why in place:** the owner decided on 2026-09-13 that the deal is the
  unit of precedent (epic #227). The old `n` over-counted deals, so digests
  already bound under `digest_version` 2 mis-stated precedent strength.
  Correcting `n` under the same version fixes those digests on the next
  compile instead of carrying the over-count forward.
- **Conformance vectors regenerated** (`scripts/generate_conformance_vectors.py`):
  `expected.digest` changes in 003–008 and 011–012, where each clause's
  single observed row with `precedent_count: 3` is now `n: 1`, band
  "rare". Their `input`, `canonical`, `content_hash` and `section_digests`
  are unchanged. Vector 013's input was rebuilt so every observed row is
  its own deal's single signed row for the clause, on a distinct
  `document_id` (the shape the compiler now emits), with
  `confidence.n_our_paper` set to the distinct our-paper deal count (29);
  it still pins the `n=10` "often" and `n=9`/`n=2` "sometimes"
  boundaries. One exception is deliberate: its collision group adds a
  second signed row, with another spelling the digest merges, to a deal
  that already carries one. The compiler never emits that shape; it
  models a hand-curated or legacy-store `observed_positions` input, which
  the digest must still count once per deal. 001, 002, 009 and 010 carry no clauses
  and are byte-identical. The generator now pins the vector set's
  `engine_version` stamp at `1.0.0` so a re-run reproduces the committed
  files exactly. The pin above is updated.

### 2026-09-25 — clause-tree `ClauseNode` gains optional `heading_span`; `char_span` covers the whole clause (issue #217)

`spec/clause-tree.schema.json`'s `$defs.ClauseNode` gained an optional
`heading_span` property (a 2-integer array, or `null`) recording the
heading line alone, and its `char_span` description now states what the
engine emits: the whole clause, from the start of its heading line through
the end of its own body text (children excluded) — the span OPF citations'
`char_span` resolve to (OPF-SPEC §4, whose wording already said so; the
ingesters previously emitted heading-only spans). `heading_span` is absent
when a node has no separate heading line (the synthetic pre-heading `"0"`
node, sub-clauses promoted from body text, LLM/agent-grounded nodes);
when present, `heading_span[0] == char_span[0]` and
`heading_span[1] <= char_span[1]` (`ClauseTree.validate` invariant 6).
Additive/optional — older serialized clause-tree files with no
`heading_span` key still validate and load unchanged. Like the `page`
entry below, this is the intermediate clause-tree artifact, not
`playbook.schema-0.3.json`: no `opf_version`/`digest_version` bump, and the
file stays outside the **Current pins** table.

### 2026-08-22 — Conformance vectors for canonicalization + digest (issue #115)

Added `spec/conformance/` (`manifest.json` + `vectors/*.json`): the
normative, plain-JSON, standalone-consumable conformance vector set for
canonicalization/content-hashing (`canonicalize.py`) and `digest` section
construction (`digest.py`), stamped `opf_version` 0.3 / `digest_version` 2 /
reference `engine_version` 1.0.0. `docs/OPF-SPEC.md` gained §10.2
documenting them as the normative definition. No schema, `opf_version`, or
`digest_version` change — this entry exists because canonicalization is
explicitly within this changelog's stated scope.
`tests/test_conformance_vectors.py` checks this engine against the same
frozen vectors; see `spec/conformance/README.md` for what edge case each
vector isolates. This is the mechanism issue #113's normative-rule-change
policy anticipated ("an independent [implementation] checked against the
conformance vectors") — the drift class it defends against is
Contract Toaster's hand-maintained source-level ports of these two files,
pinned to an engine commit by docstring rather than to anything
mechanically checkable until now.

**Fix round 1 amendment (same day):** added vector 013 (digest dedupe/rank/
top-N-plus-material capping and the "often"/"sometimes"/"rare" frequency-
band boundaries — `_dedupe_rank`/`_preferred_variations` in `digest.py`),
which vectors 001-012 left completely unexercised despite the normative
digest-construction conformance claim. Also added this table's
`spec/conformance/` pin, mechanically enforcing the "never edited in place"
rule `spec/conformance/README.md` already stated but nothing previously
checked (`tests/test_spec_consistency.py::test_spec_changelog_pins_conformance_vectors`).

### 2026-08-22 — OPF 1.0: stability policy and normative-rule-change policy (issue #113)

OPF-SPEC §11 gained two new normative statements, effective at 1.0: the
**stability policy** (1.x changes are additive-only — new OPTIONAL fields
and new `x_*` extensions permitted; no new REQUIRED field, no removing or
retyping an existing field, no changing a normative MUST without a 2.0
release) and the **normative-rule-change policy** (any new or changed
MUST — whether or not it touches the schema — gets its own entry under a
`### Normative rule changes` heading in `CHANGELOG.md`, in the release it
ships under). §3.13's id-uniqueness rule, recorded in `CHANGELOG.md` for
the first time by this same release, is the motivating case: it shipped as
a blocking validator rule with no schema change and no version bump, the
exact drift class both new policies exist to make greppable. No schema,
`opf_version`, or `digest_version` change — the document shape
(`opf_version` "0.3") is unchanged; this entry exists because both new
rules are normative validator rules, within this changelog's stated scope,
even though their durable record lives in `CHANGELOG.md`.

### 2026-08-03 — clause-tree `ClauseNode` gains optional `page` (issue #86)

`spec/clause-tree.schema.json`'s `$defs.ClauseNode` gained an optional
`page` property (`integer >= 1`, or `null`) recording the 1-based source
page the clause begins on, mirroring `playbook_engine/clause_tree.py`'s
new `ClauseNode.page`. Additive/optional — older serialized clause-tree
files with no `page` key still validate and load unchanged. This is the
intermediate clause-tree artifact ("intermediate artifact, not OPF" per
the schema's own `description`), not `playbook.schema-0.3.json` — no
`opf_version`/`digest_version` bump, and the file is intentionally outside
the **Current pins** table's CI-enforced hash check above (that table, and
`tests/test_spec_consistency.py::test_spec_changelog_pins_every_schema`,
cover only `playbook.schema*.json`).

### 2026-07-29 — sibling-id uniqueness is now a blocking normative rule (issue #70)

OPF-SPEC §3.13 (new): `evidence.clauses[].id`, `evidence.clause_library[].concept_id`,
`floor.invariants[].id`, and `corpus.documents[].document_id` MUST be unique
among their siblings, in every OPF version (0.1's top-level `clauses`/
`clause_library` shape included). Not schema-expressible, so enforced only by
`validator.validate_document` (`_check_duplicate_ids`, fail-closed) — a
duplicate sibling id made `export_profile`'s id-keyed sample/rewrite paths
collapse two entries onto one, silently shipping the first duplicate's
flagged text unmodified. No schema or `opf_version`/`digest_version` bump —
purely a new normative validator rule, same category as the digest
consistency check (§3.12).

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
consumer had bound an artifact in the gap. Flagged by a downstream
review-engine team — the pin-and-verify discipline that caught it is the
intended consumer posture.

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
