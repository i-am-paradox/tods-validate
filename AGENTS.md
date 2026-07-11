# AGENTS.md — ods-validate

> Drop this file at the root of the `ods-validate` repo. It is the source of truth for project
> intent, scope, conventions, and the build plan. Read it fully before writing any code.

## What this is

`ods-validate` is a Python CLI and GitHub Action that validates Operational Data Standard (ODS)
feeds. ODS is an open standard, convened by Compiler LLC under Cal-ITP, that extends GTFS to
describe scheduled transit operations: deadheads, personnel/run assignments, and non-revenue
service. The spec lives at https://ods.calitp.org/ and the spec repo is under the cal-itp GitHub
org. GTFS has a canonical validator (MobilityData's gtfs-validator) that anchors its entire
ecosystem; ODS has no equivalent. This project fills that gap.

## Why it exists (strategic context — read this, it shapes decisions)

This is a portfolio/contribution project by Chelsea Kelly-Reif, built to be genuinely useful to
the ODS working group and, candidly, to open a conversation with Compiler LLC. The audience is
twofold:

1. **Transit schedulers and vendors** who produce or consume ODS feeds and need actionable,
   human-readable validation errors.
2. **The ODS working group / Compiler engineers** who will judge this repo as a work sample.
   Every decision should look like it came from someone who could maintain cal-itp repos
   tomorrow: tested, documented, accessible, boring in the best way.

Deadline pressure: a coffee meeting with Compiler's founder at the end of June 2026. By then the
repo must be public, installable, and able to validate a real or sample ODS feed end to end.
Polish beats feature count. A validator that checks 12 rules flawlessly with great error messages
beats one that half-checks 40.

**Do not** frame this repo anywhere in its docs as a job-search artifact. The README speaks only
to transit data practitioners.

## Ground rules for Codex

- **Never invent spec details.** Before implementing any rule, fetch and read the actual ODS
  spec from ods.calitp.org (and its GitHub source). File names, field names, required/optional
  status, and enum values come from the spec, not from memory or analogy to GTFS. If the spec is
  ambiguous, implement the permissive interpretation and open a `docs/spec-questions.md` entry —
  these ambiguity findings are themselves valuable contributions to the working group.
- Check whether the spec ships sample/reference feeds in the cal-itp ODS repo. If yes, vendor a
  copy into `tests/fixtures/` (respect license; ODS is Apache/CC-style open — confirm). If no,
  hand-build a minimal valid fixture feed and a catalog of invalid variants.
- Match the conventions of the cal-itp Python ecosystem where reasonable (they use Django, pytest,
  pre-commit, GitHub Actions, Dependabot across repos like cal-itp/benefits). This repo should
  feel native to that org.
- Python 3.11+. Keep runtime dependencies minimal: `pydantic` (or stdlib dataclasses) for rule
  config, `click` or `typer` for CLI, CSV handling via stdlib `csv` — avoid pandas in the core
  (heavy dependency for a validator; vendors will run this in CI).
- License: MIT or Apache-2.0 to match the cal-itp ecosystem (check what the ODS spec repo uses
  and mirror it).

## Architecture

```
ods-validate/
├── AGENTS.md                  # this file
├── README.md                  # practitioner-facing; quick start in < 5 min
├── pyproject.toml             # PEP 621, hatchling or setuptools, console_scripts entry
├── src/ods_validate/
│   ├── __init__.py
│   ├── cli.py                 # `ods-validate path/to/feed/ [--format json|text|github]`
│   ├── loader.py              # reads an ODS package (dir or zip), file discovery, encoding
│   ├── schema.py              # spec-derived table & field definitions (generated, see below)
│   ├── rules/
│   │   ├── __init__.py        # rule registry + severity levels (ERROR / WARNING / INFO)
│   │   ├── structure.py       # required files present, no unknown files (warn), parse errors
│   │   ├── fields.py          # required fields, types, enums, ID formats
│   │   ├── references.py      # cross-file foreign keys (and refs into companion GTFS feed)
│   │   └── semantics.py       # logical checks: time ordering, overlapping assignments, etc.
│   ├── gtfs_companion.py      # optional: load the paired GTFS feed for cross-validation
│   └── report.py              # renderers: human text, JSON, GitHub Actions annotations
├── tests/
│   ├── fixtures/              # valid feed + one broken fixture per rule
│   └── test_*.py              # one test module per rules module; aim 90%+ coverage on rules
├── action.yml                 # composite GitHub Action wrapping the CLI
├── .github/workflows/ci.yml   # lint (ruff), type-check (mypy), test matrix 3.11/3.12/3.13
└── docs/
    ├── rules.md               # auto-generated rule catalog: ID, severity, description, example
    └── spec-questions.md      # ambiguities found while implementing — working-group fodder
```

Key design decisions, made now so they don't get relitigated:

- **Rules are data + small functions, not a framework.** Each rule has a stable ID
  (`ODS-E001`-style), severity, spec citation (section link), and a check function that yields
  `Finding` objects (file, row, column, message, suggestion). Resist building a plugin system;
  a registry list is enough.
- **Error messages are the product.** Every finding must say what's wrong, where, and what good
  looks like, in language a transit scheduler (not a programmer) can act on. Example: bad —
  "foreign key violation in deadheads"; good — "deadheads.txt row 14: trip_id 'WKDY-103' does
  not exist in the companion GTFS trips.txt. Deadhead records must reference a scheduled trip."
- **Three output modes:** human-readable text (default, grouped by severity), `--format json`
  (stable schema for tooling), `--format github` (workflow annotations so the Action surfaces
  findings inline on PRs).
- **The GitHub Action is a first-class deliverable.** A vendor maintaining an ODS export should
  be able to add 6 lines of YAML and get PR annotations. This is the thing that makes the tool
  spread.

## Build plan

### Phase 1 — Skeleton + structure rules (week 1)
- Repo scaffolding, CI, pre-commit (ruff, mypy), pyproject with console entry point.
- Loader: accept directory or .zip; handle UTF-8/BOM; clear errors on malformed CSV.
- Spec ingestion: read the ODS spec, transcribe table/field definitions into `schema.py` with
  spec-section citations in comments. If the spec publishes a machine-readable schema, generate
  from it instead and commit the generator script.
- Structure rules: required files present, unknown files warned, header rows match spec.
- Definition of done: `pipx install` from the repo, `ods-validate tests/fixtures/valid/` exits 0
  with a clean report; a deliberately broken fixture exits 1 with findings.

### Phase 2 — Field + reference rules (week 2)
- Field-level: required fields, type coercion checks, enum validation, duplicate primary keys.
- Cross-file references within the ODS package.
- `--gtfs path/to/gtfs.zip` flag: validate ODS references into the companion GTFS feed
  (trip_ids, stop/location ids — whatever the spec says links the two). Use stdlib zip + csv;
  do not pull in a full GTFS library for this.
- `docs/rules.md` generated from the registry via a small script, run in CI to prevent drift.

### Phase 3 — Semantic rules + Action + release (week 3)
- Semantic checks worth their weight: shift/run time ordering, assignments referencing dates
  outside the feed's service window, overlapping duty assignments for the same personnel id
  (severity WARNING — real schedules have legitimate edge cases; never be dogmatic where the
  spec isn't).
- `action.yml` composite action + a demo workflow in the README.
- Tag v0.1.0, publish to PyPI under `ods-validate`. GitHub release notes written for
  practitioners.
- Stretch (only if everything above is done): a `--summary-md` flag emitting a Markdown report
  suitable for pasting into a working-group issue.

### Phase 4 — The contribution play (after coffee, ongoing)
- Open an issue or discussion on the ODS spec repo introducing the tool, linking the rule
  catalog, and attaching `spec-questions.md` findings as respectful, specific questions.
- Offer the tool to the working group; signal willingness to transfer or co-maintain under
  cal-itp if they want it. The goal is adoption, not ownership.

## Quality bar

- Every rule has at least one passing and one failing test with a dedicated fixture.
- `ruff check`, `mypy --strict` on `src/`, pytest all green in CI before any feature work
  continues. No skipped tests on main.
- README contains: what ODS is (two sentences, link out), install, a 30-second usage example
  with real output pasted in, the Action snippet, and a rule catalog link. No marketing voice.
- Conventional commits; PR-sized changes even when working solo (the git history is part of the
  work sample).
- Accessibility of output: no color-only signaling in terminal output; findings readable when
  piped to a file.

## Writing style for docs (README, rule messages, release notes)

Plain, concrete prose. At most one em dash per document, prefer zero. No rule-of-three rhetorical
constructions. No "simply," "just," "blazingly," "powerful." Vary paragraph openings. It is fine
for a sentence to run long or land flat; do not over-polish. Write like a careful engineer, not a
launch tweet.

## Open questions to resolve early (do not guess)

1. Exact ODS v1 file and field inventory — fetch the spec first.
2. Is ODS v2 published in draft? If yes, add a `--spec-version` flag and validate against v1 by
   default; note v2 deltas in docs.
3. Does the spec repo include reference feeds or a conformance suite? Reuse before rebuilding.
4. License and code-of-conduct norms in the cal-itp org — mirror them.
