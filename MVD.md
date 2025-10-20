# 1) What MLOps is (short, citeable)

MLOps = the engineering practice that makes ML products **reproducible, automated, monitored, and continuously improved**—combining ML, DevOps, and Data Engineering across the full lifecycle (design → train → deploy → serve → monitor → retrain).

# 2) Principles you must show in the MVD

The paper lists nine principles—your MVD should demonstrate at least a minimal slice of each: **CI/CD automation, workflow orchestration, reproducibility, versioning (data/model/code), collaboration, continuous training & evaluation, metadata tracking, continuous monitoring, and feedback loops**.

# 3) Components you must include (minimal versions)

Map MVD to the canonical components (C1–C9) so reviewers see 1:1 alignment:

- **C1 CI/CD**: GitLab pipeline stages (build/test/train/eval/package/deploy).
- **C2 Code repo**: Git(Lab) project for code + configs.
- **C3 Orchestrator**: For MVD, simple GitLab DAG or a tiny Airflow/Kubeflow pipeline (one DAG).
- **C4 Feature store**: For MVD, skip a full FS—use a small, versioned CSV (or DVC) to prove the pattern.
- **C5 Training infra**: One GPU (or CPU for toy model) runner; scalable later to HPC.
- **C6 Model registry**: Minimal = store artifact in GitLab Packages or MLflow Registry.
- **C7 Metadata store**: Minimal = metrics.json + run metadata (or MLflow tracking).
- **C8 Serving**: Containerized FastAPI with `/predict`.
- **C9 Monitoring**: `/metrics` (Prometheus format) + basic dashboard.

# 4) Minimal pipeline you can implement now

**Stages:** `lint → unit → train → evaluate(gate) → package(container) → deploy(manual) → smoke`

**Artifacts:** `model.pkl`, `metrics.json`, `model_card.md`, container image

**Endpoints:** `/predict`, `/metrics`

**Feedback loop:** If `evaluate` fails thresholds, pipeline breaks (human-in-the-loop). Later, wire drift→retrain trigger.

# 5) “Definition of Done” (acceptance criteria)

- Reproducible **train→eval→serve** run with pinned deps and seed.
- Versioned **code + data sample + model** (tags/releases).
- CI gate: pipeline **fails** if accuracy/F1 < threshold (we need to discuss and define metrics).s).
- Deployed container exposes **/predict** and **/metrics**; Prometheus scrapes metrics.
- Run metadata (params + metrics) stored (file or MLflow).
- Short **Model Card** describing data, metrics, and risks.
