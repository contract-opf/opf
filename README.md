# Open Playbook Format (OPF)

A machine-readable format for expressing **contract-review playbooks** — the
standard positions, acceptable variations, and hard-rejection rules a reviewer
applies when comparing a counterparty's draft against a standard form.

OPF lets a playbook be versioned, hashed, governed, and executed by tooling
rather than living only in a reviewer's head or a static checklist.

## What's here

```
opf/
├── opf/                              # the OPF schema (playbook.schema-0.2.json) + notes
└── schema/
    ├── playbook.schema.json          # the generalizable playbook schema
    ├── output.schema-v1.json         # the model-response / decision output contract
    ├── bundle.schema-v2.json         # governed release-bundle schema (hashes + approval)
    ├── pen-rules.defaults.json       # default penalty/rule configuration
    ├── registry.example.json         # example playbook registry
    └── synthetic-knowledge.example.json   # a small, fully synthetic example
```

## Core ideas

- **Topics** carry a standard position, `acceptable_variations` (changes that
  don't require a redline), and `reject_if_proposed` / `must_preserve` rules.
- **Anchors** tie each topic to a section of the standard form so detectors
  scope to the right clause.
- **Governed bundles** bind a playbook hash, prompt hash, standard-form hash,
  model-policy hash, and an approval record, so any review is traceable to the
  exact inputs it used.

## Reference example & implementation

- **Example playbook:** [`contract-opf/eiaa-playbook`](https://github.com/contract-opf/eiaa-playbook)
  — a real EIAA (Educational Affiliation Agreement) playbook expressed in OPF.
- **Reference implementation:** [`contract-opf/contract-toaster`](https://github.com/contract-opf/contract-toaster)
  — an LLM-assisted reviewer that executes OPF playbooks and emits tracked-changes redlines.

## License

Apache License 2.0. See [LICENSE](LICENSE).
