# Review context for ORCA-GCN Citation Trust

## What this project is

A single Google Colab notebook (`src/ORCA_GCN_CAIF.ipynb`) implementing CAIF
(Curvature-Aware Impact Factor) - a trust-weighted replacement for the
Journal Impact Factor. Every citation edge is audited along four axes
(Ollivier-Ricci curvature, a curvature-gated GCN embedding score, temporal
plausibility, semantic similarity), fused into a feature vector, and passed
through a small interpretable Trust Head (`sigmoid(W^T*Phi + b)`) to produce
a bounded trust weight `w in (0,1)` per citation. CAIF is the mean trust
weight per journal; NCAIF normalizes it against a peer-journal field mean;
a degree-preserving null model and Z-score gate decide whether to
PUBLISH or FLAG the result.

**Source of truth:** `phase_B_yuval_remove&clean_later/PRD for
development/PRD.md`. This PRD is what the notebook was built against
cell-by-cell (Cells 0-14 map directly to its Section 6). It also defines
the 18 Phase-A project-book tests the notebook's Cell 13 implements as
executable assertions - these are the hard pass/fail gate, not optional.

## Review priorities

- **CFG-driven, no magic numbers.** Every tunable value belongs in the
  `Config` dataclass (Cell 1). A hardcoded literal appearing anywhere below
  that cell is a defect, not a style nit.
- **Fail loud, not silent, on bad input or bad state.** Prefer a clear
  exception with context (which stage, which config, which identifiers)
  over a value that silently degrades (e.g. a lookup that falls back to
  `0.0` instead of raising when the fallback masks a real bug).
- **No em dashes** in code, comments, or commit messages - use `" - "`
  (repo-wide convention).
- Flag genuine DRY violations (the same computation done two different ways
  in two places) as high-priority - that exact pattern caused a real bug
  here before (see below).

## Review history on this PR - don't re-flag these as new

1. **First Copilot review** (6 comments) is already fully addressed:
   `sh()` now fails fast instead of swallowing pip-install failures; a
   missing torch install now raises instead of printing a warning; the
   OpenAlex API-key assert now tolerates a cached `raw_works.pkl` for
   offline/CI runs; `ORCAConv.gate()` is epsilon-clamped so `exp`/`sigmoid`
   modes can't underflow to exactly `0.0` (verified locally: kappa=-1000
   returned exactly `0.0` before the fix); the null model's `h(u,v)` lookup
   used to read a dict (`H_MAP`) that only covered the original graph's
   edges and silently returned `0.0` for every rewired edge, which
   distorted the null distribution - fixed with `h_of(u, v)`, computed from
   stored per-node embeddings, valid for any node pair; and Tests 04/08 now
   report `SKIP` (not `FAIL`) on the documented Windows
   `multiprocessing.get_context('fork')` limitation in `GraphRicciCurvature`
   instead of misreporting an environment gap as a real test failure.
2. **Follow-up refactor pass** (DRY/KISS + error handling): consolidated
   `H_MAP` away entirely in favor of `h_of()` as the single source of truth
   (it duplicated the exact value `h_of()` computes, which is what enabled
   the bug in item 1); extracted `_register_node()` to de-duplicate the
   near-identical journal/author node-registration blocks in the graph
   builder; `OpenAlexClient._get()`'s HTTP errors now carry the endpoint and
   call count instead of a bare `requests.HTTPError`; `compute_orc()`
   failures now report which component (node/edge count) and ORC config
   triggered them; an unrecognized `CFG.SEMANTIC_BACKEND` now raises
   `ValueError` instead of silently running the scibert branch.

Please still double-check these are correct, but they're not undiscovered
issues - the goal of a re-review is to catch anything *new* the last two
commits introduced, and anything the first review missed.

## Known, accepted gaps (documented in the PR description, not TODOs to invent)

Cell 11's `weak_labels()` doesn't implement the bulk-citation weak-negative
rule (`CFG.WEAK_BULK_REF_MIN` is defined but unused); there's no
grouped-by-journal-pair validation split for the weak-supervision labels;
`edge_audit.parquet` (PRD deliverable 4) is never written; there's no
coefficient-stability/bootstrap reporting for the learned `W`; and the
peer-journal field-normalization pipeline (`FIELD_PEER_RESULTS`) is a
manual stub, so `NCAIF == CAIF` until it's populated. These are real,
tracked gaps against the PRD - feel free to comment on them, but they are
already known and are follow-up work, not blockers on their own.

## Verification status

Cells 0-6 were run for real on live Google Colab against the real OpenAlex
API. Cells 6-14 were verified via a local Windows test harness (cached
pipeline stages, small synthetic graphs) rather than a full live Colab
re-run - Ollivier-Ricci curvature itself needs `multiprocessing`'s `fork`
context, which doesn't exist on Windows, so that one step was verified on
Colab separately and stubbed for local runs of everything downstream of it.
The `sh()` fail-fast and gate-epsilon fixes were also verified in isolation
(outside the notebook, via plain torch/subprocess scripts) since a full
Colab re-run wasn't done for the latest two commits.
