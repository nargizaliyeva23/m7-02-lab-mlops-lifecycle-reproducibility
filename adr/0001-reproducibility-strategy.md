# ADR 0001: Reproducibility strategy for NorthStar models

## Context

NorthStar Logistics currently stores models directly in S3 using inconsistent naming conventions such as `eta_v2_FINAL.onnx`. There is no dataset versioning, registry lifecycle, or reproducibility standard, making rollback, debugging, and auditability difficult.

The ML team currently lacks consistent lineage tracking between datasets, training code, model artifacts, and deployed production versions.

---

## Decision

### Environment

All training and inference environments will run inside Docker containers with pinned image versions. Container images will be stored in Amazon ECR and referenced by immutable image digests instead of mutable tags such as `latest`.

Python dependencies will be pinned using exact package versions in `requirements.txt`. CI pipelines in GitHub Actions will rebuild containers only when dependency or infrastructure changes occur.

Training and inference containers must use the same major framework versions to prevent training-serving skew.

---

### Data

Training datasets will be versioned using DVC (Data Version Control) with storage backed by Amazon S3. Every training run must include:

* dataset hash
* feature pipeline version
* evaluation dataset version
* feature store snapshot identifier

Feature generation pipelines will create immutable snapshots rather than querying live production tables during training.

Evaluation datasets will remain frozen and immutable across retraining cycles to ensure metric comparability between model versions.

Dataset hashes and feature versions will be stored as lineage metadata inside MLflow and the model registry.

---

### Code

All experiments must originate from committed Git branches. Every training run will store:

* git commit SHA
* training configuration
* hyperparameters
* framework versions
* MLflow run ID

Experiment tracking will be managed with MLflow. Every registered model version must link back to a reproducible MLflow run.

Direct manual uploads to S3 are prohibited. Models may only enter the registry through CI/CD pipelines executed by GitHub Actions.

All deployment manifests and infrastructure configuration files will remain under version control.

---

### Randomness

All training pipelines must explicitly set deterministic random seeds for:

* Python
* NumPy
* model training frameworks

Data splitting procedures must use deterministic partitioning logic to ensure reproducible train/validation/test datasets.

Training jobs will log random seeds as lineage metadata in MLflow and registry records.

Any intentionally non-deterministic training configuration must be explicitly documented in the experiment metadata.

---

## Alternatives rejected

* "Store models directly in S3 with manual naming"

  * Rejected because lineage tracking and rollback history become unreliable.

* "Use timestamps instead of dataset versioning"

  * Rejected because timestamps do not guarantee identical dataset snapshots.

* "Allow mutable Docker tags such as latest"

  * Rejected because environments may silently change between retraining runs.

* "Allow manual production deployments"

  * Rejected because deployment state becomes difficult to audit and reproduce.

---

## Consequences

* Engineers must maintain DVC metadata and S3 synchronization processes.
* CI/CD pipelines become mandatory for all production model releases.
* Additional infrastructure and storage costs will occur due to immutable artifact retention.
* Training pipelines may become slightly slower due to deterministic configuration requirements.

---

## Revisit if

* Regulatory compliance requirements, team size, or model volume increase to the point where centralized governance systems such as Kubeflow or Amazon SageMaker Model Registry become operationally necessary.
