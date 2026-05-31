# ADR 0001: Reproducibility strategy for NorthStar models

## Context

NorthStar's current ML artefacts live in S3 as files named `eta_v2_FINAL.onnx` with no registry, no dataset versioning, and no record of which code, environment, or data produced each model. If a production model degrades, the team cannot reconstruct the training conditions to diagnose the regression or roll back to a known-good state.

## Decision

Reproducibility is enforced across four layers:

**Environment:** Every training run executes inside a Docker image built from a pinned base (`python:3.11.9-slim`). Python dependencies are pinned with `pip-compile` into a checked-in `requirements.lock` file. The Docker image SHA and the SHA256 of `requirements.lock` (`python_env_hash`) are stored in every model artefact's metadata in MLflow. A new image is built and pushed to ECR on every merge to `main`; training jobs reference the image by digest, never by a mutable tag like `latest`.

**Data:** All training datasets are managed with DVC backed by S3. Every training run references a specific DVC commit that resolves to a deterministic Parquet snapshot. The SHA256 of that snapshot (`dataset_hash`) is stored in the model's MLflow metadata. Running `dvc checkout <dataset_hash>` on any machine retrieves the exact bytes used to train any historical model.

**Code:** Training jobs only start on clean git state (enforced by a pre-flight check: `git diff --quiet && git diff --cached --quiet`). The HEAD commit SHA (`git_sha`) is logged to MLflow at run start and stored in the model artefact metadata. Any model can be reproduced by checking out that SHA, pulling the matching DVC dataset, and building the matching Docker image.

**Randomness:** A single integer root seed is passed to all stochastic components at training entry-point. The entry-point sets `PYTHONHASHSEED`, `random.seed`, `numpy.random.seed`, and `torch.manual_seed` (+ `torch.cuda.manual_seed_all`) from this root seed before any library is called. The root seed value is logged to MLflow as `random_seed`. Exact bit-for-bit reproducibility is not guaranteed across GPU hardware generations due to non-deterministic CUDA kernels; the strategy guarantees statistical reproducibility (metric variance < 0.1% across re-runs with the same seed on the same hardware class).

## Alternatives rejected

- **No dataset versioning — rely on S3 object timestamps:** S3 timestamps are mutable (overwrite with same key, copy operations, bucket replication). A timestamp does not identify the exact bytes of the training set, and it provides no mechanism to retrieve the data later if the bucket policy changes. Rejected: insufficient for audit or rollback.
- **Conda environment files instead of Docker + pip-compile:** Conda `environment.yml` with version ranges (e.g., `numpy>=1.24`) does not pin the full dependency closure, and Conda solves non-deterministically across platforms. The team uses Linux for training; Docker + pip-compile gives a fully locked, platform-specific environment that is byte-for-byte identical across machines. Rejected: Conda resolution drift makes environment reproduction unreliable.
- **Log everything to a shared spreadsheet:** Proposed informally as a lightweight alternative. Not machine-readable, not enforced at run time, and does not integrate with the CI/CD pipeline or the model registry. Rejected: human-maintained records rot immediately.

## Consequences

- Every training run requires a clean git working tree; engineers cannot run ad-hoc experiments from dirty checkouts without explicitly passing `--allow-dirty` (which sets a flag in MLflow metadata marking the run as non-reproducible).
- DVC adds a checkout step to the training pipeline. On a cold machine, fetching a 50 GB dataset from S3 takes ~10 minutes. Teams must budget for this in CI/CD pipeline time.
- Docker image builds are triggered on every merge to `main`. Build times (~3–5 min) add to the CI loop. Images are cached in ECR; stale images are retained for 90 days to cover any model in staging or production during that window.

## Revisit if

The team grows beyond ~15 engineers and multiple teams begin sharing datasets, at which point a dedicated feature/data platform (e.g., Feast + Delta Lake with ACID versioning) would replace DVC's file-hash approach with row-level data lineage and point-in-time correct feature retrieval.
