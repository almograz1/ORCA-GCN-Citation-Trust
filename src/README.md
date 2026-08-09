# ORCA-GCN CAIF

Implementation of `ORCA_GCN_CAIF.ipynb`, per the PRD at
`phase_B_yuval_remove&clean_later/PRD for development/PRD.md`. Target runtime
is Google Colab (see PRD §5 for why: the bottleneck is CPU-bound optimal
transport, not GPU).

## Getting an OpenAlex API key

Since 2026-02-13 OpenAlex requires a key ($1.00/day free tier vs $0.10/day
without one). Get one in ~30 seconds at https://openalex.org/settings/api.

**Do not paste the key into a notebook cell or commit it anywhere.** In
Colab:

1. Open the **Secrets** panel (key icon in the left sidebar).
2. Add a secret named `OPENALEX_API_KEY` with your key as the value.
3. Toggle "Notebook access" on for this notebook.

Cell 1 reads it via `google.colab.userdata.get("OPENALEX_API_KEY")`, falling
back to the `OPENALEX_API_KEY` environment variable for local/CI runs (never
hardcoded — see the global security rule this repo's owner uses).

## Running

Open `ORCA_GCN_CAIF.ipynb` in Colab and run top to bottom (`Runtime > Run
all`). Every expensive stage checkpoints to Google Drive under
`CFG.CACHE_ROOT`, so a runtime disconnect never costs more than one stage —
see PRD §2.1.

Recommended runtime tier: **L4** for routine runs, **A100 40GB** (bought for
its 12 vCPUs, not its tensor cores) for the reported strict-null-model run.
See PRD §5.2–5.4.

## Local development

`requirements.txt` pins the same dependencies for local editing/linting, but
this is not how the notebook is validated — validation happens on Colab
itself, since Colab's preinstalled package versions can differ from any
local environment (PRD §3.4, risk R5).
