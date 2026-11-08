# PharmAgent

PharmAgent is a multi-agent framework for budgeted molecular optimization.
This package runs two system experiments: **E1**, target-only optimization,
and **E2**, ADMET-constrained optimization with adaptive penalties, curriculum
activation, retrieval, and Chemical Experience Replay (CER).

The PharmAgent framework is extensible to additional ADMET-constrained molecular optimization tasks.

## Installation

Use a dedicated **Python 3.10** environment on Linux or macOS:

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python check_installation.py
python -m unittest discover -s tests -v
```

Run these commands from the package directory. The local predictors and their
dependencies are substantial; allow several GB of installation space. E2 runs
ADMET-AI locally, so each concurrent E2 worker also consumes model memory.
A GPU is not required. Start with two workers and increase concurrency only
when memory and API capacity permit it.

The public retrieval snapshot and three PMO oracle model files are included.
Their provenance is described in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
To test retrieval, all five local PMO oracles, and ADMET-AI without any model
API requests:

```bash
python check_installation.py --local-predictors
```

## Run E1 and E2

First inspect the execution plan. This command makes **no API requests** and
does not ask for a credential:

```bash
python run.py --experiment both --dry-run
```

Launch both experiments with one command:

```bash
python run.py --experiment both --run
```

The launcher displays the run size, asks for confirmation, then prompts for
your own MiniMax API key with hidden input. The key is used only in the running
process environment; it is not saved to a configuration file or command line.
Alternatively, supply `MINIMAX_API_KEY` through your job environment or secret
manager. Do not paste a key into a shell command or a shared file.

The fixed configuration uses **MiniMax-M3** at
`https://api.minimaxi.com/v1`, with the Standard service tier and thinking
disabled. Your account must have access to that model and endpoint. A key for
an unrelated provider is not interchangeable. The launcher checks model
availability before starting workers and stops if it cannot verify access.
Hosted-model outputs are stochastic and may change with provider updates;
repeated runs need not produce identical numerical results.

E1 and E2 can also run independently:

```bash
python run.py --experiment E1 --run
python run.py --experiment E2 --run
```

For concurrent execution of both arms, use a single controller:

```bash
python run.py --experiment both --workers 10 --run
```

`--yes` skips the interactive cost confirmation for a scheduled job. It requires
`--run` and does not supply a credential. All calls are billed to the account
configured by the person running the package.

## Experimental protocol

| Setting | Value |
| --- | --- |
| Ordered tasks | GSK3beta, JNK3, DRD2, Sitagliptin MPO, Albuterol similarity |
| Seeds | 0, 1, 2, 3, 4 |
| Unique oracle budget | 1,000 per task and seed |
| One experiment | 25 task runs, up to 25,000 oracle evaluations |
| Both experiments | 50 task runs, up to 50,000 oracle evaluations |
| Round safety limit | 20,000 per task |
| HTTP-attempt safety limit | 60,100 per task |

An oracle evaluation is not an API call. Invalid or repeated proposals and
transport retries can require many API requests without increasing the oracle
count. The combined HTTP-attempt ceiling is 3,005,000; it is a safety fuse,
**not an expected request count, token estimate, or monetary spending limit**.
Set account-level cost controls before launching a full run.

Each worker owns one experiment and one seed. Its five tasks run in the stated
order and share that seed's CER state. Different seeds and experiments have
independent state. Task-level sharding is not equivalent to this protocol.

The E2 curriculum activates no constraints for rounds 0-19, the easy constraint
set for rounds 20-59, and all seven constraints from round 60. The exact
constraint thresholds, penalty updates, generation settings, and retry limits
are specified in `configs/pmo_admet.yaml`; E1 uses `configs/pmo_standard.yaml`.
The launcher validates these settings rather than silently substituting
fallback scorers or a different model.

## Checkpoints and continuation

Outputs default to `results/pharmagent`. Use `--output-dir` to choose a
different location. The run manifest, native journals, HTTP-attempt ledgers,
CER databases, and worker logs are written under that output directory.

To continue an interrupted run, first make sure its previous controller and
workers have stopped, then repeat the same command with the same output
directory. Completed checkpoints are validated and reused; incomplete tasks
resume from execution records matched to the same configuration. An inconsistent or indeterminate
checkpoint is rejected instead of being silently discarded or counted as
complete. Do not delete journals or reset ledgers to bypass a recovery error.

Keep the source, configuration, assets, task order, seeds, and dependencies
unchanged during a run. This package computes its own content-based execution
identity. Checkpoints from another source bundle are not interchangeable.
One launcher may own an output directory at a time. E1-only and E2-only
invocations may use it sequentially; use `--experiment both` to run them together.

For long runs, use a persistent server session or a batch scheduler.

## Read results

```bash
python summarize.py --experiment E1
python summarize.py --experiment E2
python summarize.py --experiment both
```

The summary prints per-seed metrics and five-seed means as JSON. E1 reports
Top-1, Top-10, and AUC Top-10. E2 additionally reports AA-AUC and Pareto
hypervolume. All listed metrics are higher-is-better. Combined result files
are published only once all five seeds and all five tasks in that experiment
pass completion checks. Until then, consult `run_manifest.json` and the worker
checkpoints for progress; partial results are not labeled as complete means.

## Core implementation

- `pharmagent/agents/`: proposal, verification, review, and orchestration.
- `pharmagent/optimization/`: adaptive penalties, curriculum, CER, and recovery.
- `pharmagent/rag/`: matched-pair retrieval and context construction.
- `pharmagent/evaluation/`: oracle accounting, metrics, journals, and identity.
- `pharmagent/llm/`: model transport, request accounting, and response parsing.

## Package integrity

`MANIFEST.json` lists the packaged files, sizes, and SHA-256 checksums.
`provenance/package_reference.json` records predictor and retrieval asset
checksums and the required scientific dependencies. To check file integrity
using only the Python standard library:

```bash
python check_installation.py --integrity-only
```

`check_installation.py` also validates the pinned environment, execution
identity, and retrieval provenance. Its optional `--local-predictors` check
loads the actual local models with network access disabled. The unit tests
exercise the launcher, protocol, checkpoint recovery, and evaluation metrics;
the model transport is mocked in those tests.
