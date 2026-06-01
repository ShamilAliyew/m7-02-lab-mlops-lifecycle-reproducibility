# ADR 0001: Reproducibility strategy for NorthStar models

## Status
Accepted

## Context

NorthStar Logistics currently manages two production models (ETA and dispatcher-routing) with no systematic versioning or reproducibility guarantees. Models are stored in S3 with names like `eta_v2_FINAL.onnx`, and there is no record of the exact data, code, or environment used to train them. When a model fails in production or needs to be debugged, the team cannot reliably reproduce the training conditions. This creates operational risk: if a model must be retrained or an older version restored, there is no guarantee the team can reproduce the exact model that was deployed.

The four-person ML team ships new model versions every 2 weeks (ETA) and monthly (dispatcher-routing), making reproducibility critical for rapid iteration and debugging.

## Decision

The team commits to pinning and recording these four layers for every training run:

### 1. Environment (Container + Dependency Versions)

**Tool:** Docker + `requirements.txt` (pip freeze)

- Every model training run executes in a fixed Docker image (built weekly from `Dockerfile` in the model repo).
- Before training, `pip freeze > requirements-locked.txt` is captured and committed to the training run artifact.
- The Docker image ID (SHA256 digest) and `requirements-locked.txt` content hash are recorded in the model lineage.
- Python version is pinned in the Dockerfile (`FROM python:3.11.5-slim`).
- All non-Python dependencies (e.g., libxml, system tools) are listed in the Dockerfile.

**Outcome:** Any team member can reproduce the exact environment by pulling the named Docker image and installing the frozen requirements.

### 2. Data (Dataset Versioning)

**Tool:** Git LFS for data + SHA256 hashing

- Training and evaluation datasets live in a versioned S3 bucket (`s3://northstar-datasets/`) organized by date and dataset type (e.g., `events/2025-01/events.parquet`).
- Each dataset file is hashed with SHA256. The hash is computed once at upload and stored as object metadata in S3.
- Before training, the training run fetches the dataset and verifies the hash.
- The dataset path and SHA256 hash are recorded in the model lineage (fields: `dataset_version`, `dataset_hash`).
- Evaluation datasets are frozen snapshots (tagged with version `yyyy-mm-vN`) and never modified after creation.

**Outcome:** The exact rows, transformations, and schema used for training are retrievable by hash. Any model can be rebuilt by fetching the pinned dataset.

### 3. Code (Git Commit Tracking)

**Tool:** Git commit SHA (required in lineage)

- All training code (feature engineering, model architecture, hyperparameters) lives in the `ml-models` Git repository.
- Every training run is triggered by a tagged Git commit or explicitly specifies the git SHA in the training command.
- The `git_sha` (commit hash) is recorded in the model lineage, alongside the exact CLI arguments or `params.yaml` file used for that run.
- The `params.yaml` file (containing learning rate, max depth, etc.) is committed to Git at the time of the training run.

**Outcome:** `git checkout <sha> && cat params.yaml` fully specifies the code and hyperparameters used for any model.

### 4. Randomness (Fixed Random Seeds)

**Tool:** Single-point seed initialization

- A single random seed is chosen at training start (e.g., `RANDOM_SEED=42`) and used to initialize `numpy.random.seed()`, `tensorflow.random.set_seed()`, and `random.seed()` at the top of the training script.
- The seed value is recorded in the model lineage (field: `random_seed`).
- Train/test split, feature sampling, and dropout are all seeded from this single value.
- No randomness is introduced during model loading or inference; inference is deterministic.

**Outcome:** Two training runs with identical seed, code (git_sha), data (hash), and environment (Docker digest) produce byte-identical model weights.

### Recording & Access

All four pieces of information are stored in the model registry lineage record:

```yaml
lineage:
  git_sha: "a3f2e9c1d8b4..."
  dataset_hash: "sha256:7e3a9d..."
  dataset_version: "2025.01.v1"
  framework_version: "xgboost==1.7.6"
  python_version: "3.11.5"
  random_seed: 42
  training_start_timestamp: "2025-02-15T10:23:45Z"
  docker_image_sha: "sha256:f9e2..."
  requirements_hash: "sha256:b1c3..."
```

When debugging or reproducing a model, a team member queries the registry for the model version, retrieves the lineage, and then:
- Pulls the Docker image by SHA
- Checks out the Git commit (or reads params.yaml from artifact storage)
- Downloads the dataset by hash from S3
- Runs the training script with the recorded seed

This approach is **deterministic by design** — not by hope.

## Alternatives Rejected

1. **No dataset versioning; rely on S3 timestamps and folder naming**
   - **Why rejected:** S3 timestamps are not queryable from code; they don't prevent accidental overwrites; they provide no checksum guarantee. If a dataset file is re-uploaded with the same name, the old version is lost. We cannot reliably say "train on the data as it existed on Jan 15 at 10:00 UTC."

2. **Track reproducibility in experiment tracking platform (e.g., MLflow) without pinning code or data**
   - **Why rejected:** Experiment tracking systems log *what was run*, not *what environment it ran in* or *what data was used*. They are audit logs, not reproducibility systems. If the dataset S3 path or Git repository is deleted, the experiment log becomes a post-mortem, not a retraining blueprint.

3. **Use a single monolithic "environment" snapshot (AMI, pre-built VM image) instead of Docker**
   - **Why rejected:** VM snapshots are heavyweight, difficult to audit (no file list), and non-portable (tied to cloud provider). Docker is lightweight, auditable (Dockerfile), and works locally for development. We choose tooling that the team will actually use.

4. **Accept "best-effort" reproducibility — document what you remember, commit to rebuilding from scratch if needed**
   - **Why rejected:** For a team shipping models every 2 weeks, "rebuild from scratch" is not operationally acceptable. If a model regresses, the team needs to understand *why* in hours, not days. We need deterministic reproduction, not best-effort archaeology.

## Consequences

1. **Process burden:** Every training run incurs a ~5-minute overhead (dataset hash verification, Docker pull, requirements check). The team absorbs this cost upfront to avoid costly debugging later.

2. **Storage cost:** Pinning datasets by SHA256 requires retaining multiple versions of training data. Budget increases by ~20 GB/month. Retention policy is 90 days for training artifacts, 365 days for evaluation datasets.

3. **Discipline requirement:** The team must commit to never overwriting dataset files, never deleting Git commits without archiving them, and never training without recording the seed. This is a cultural commitment; tooling alone cannot enforce it. Code review and CI checks (automated lineage validation) help but require discipline.

4. **Debugging is now faster:** When a model fails, the team can reproduce the exact training run in under 30 minutes (pull Docker + data, run training). This is a net win operationally.

5. **Onboarding for new team members is easier:** A new engineer can understand how a model was built by reading the lineage record and running a single training command; they no longer need to guess or ask colleagues.

## Revisit if

- **Scale:** If the team grows to >10 engineers and is training models daily (vs. bi-weekly), the 5-minute reproducibility overhead may be worth accelerating via a centralized artifact repository (e.g., Weights & Biases, Sagemaker) instead of Git LFS + S3 metadata. Re-evaluate after team doubles.
- **Compliance:** If NorthStar is subject to regulatory audit (e.g., GDPR for EU logistics), the lineage record and 365-day retention for evaluation datasets may need extension. Update retention policy if auditors require longer history.
- **Real-time adaptation:** If models must adapt to live data (online learning) rather than batch retraining, the "frozen evaluation dataset" strategy must be revisited; re-evaluate after online learning is considered.

---

## Implementation Checklist for Sprint

- [ ] Create `.github/workflows/train.yaml` to enforce lineage recording on every training run
- [ ] Update training entry point to require `--seed` and validate it's not None
- [ ] Create `dataset_hash.py` utility to compute S3 object SHA256 before training
- [ ] Add lineage validation gate in model registry (reject submissions missing git_sha, dataset_hash, random_seed)
- [ ] Migrate existing `eta_v2_FINAL` and `dispatcher_v1` models to registry with best-effort lineage (mark "incomplete" if missing fields)
- [ ] Train with pinned requirements on the next ETA retraining cycle (2-week window)
