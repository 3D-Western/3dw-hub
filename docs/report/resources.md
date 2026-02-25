# Resources & Operational Costs

This document catalogues all infrastructure, hosting solutions, subscriptions, and third-party services required to run the full 3D Western proof of concept system, including the web platform and the AI pipeline. The AI and pipeline's cost estimations is subject to change due to future migrations away from N8N and future AI agent integrations into the system.

---

## Summary

| Category              | Service              | Hosting model    | Cost model             |
| --------------------- | -------------------- | ---------------- | ---------------------- |
| Reverse proxy         | Traefik              | Self-hosted      | Free (open source)     |
| Frontend              | Next.js              | Self-hosted      | Free (open source)     |
| Backend               | Spring Boot / Kotlin | Self-hosted      | Free (open source)     |
| Database              | MariaDB              | Self-hosted      | Free (open source)     |
| Temp file storage     | SeaweedFS            | Self-hosted      | Free (open source)     |
| Pipeline orchestrator | N8N                  | Self-hosted      | Free (community)       |
| Job queue             | Redis                | Self-hosted      | Free (open source)     |
| Pipeline DB           | PostgreSQL           | Self-hosted      | Free (open source)     |
| 3D slicer             | OrcaSlicer CLI       | Self-hosted      | Free (open source)     |
| 3MF file storage      | AWS S3               | Cloud (AWS)      | Pay-per-use            |
| Email                 | AWS SES              | Cloud (AWS)      | Pay-per-use            |
| AI model training     | AWS EC2 instance (G6, P3/P4/P5)     | Cloud (GPU)      | On demand instance pricing |
| AI dataset storage    | AWS S3               | Cloud (AWS)      | Pay-per-use            |
| AI dataset versioning | DVC                  | Self-hosted      | Free (open source)     |
| AI model serving      | FastAPI + vLLM       | Self-hosted/Cloud | Depends on deployment  |

---

> **Note:** The self-hosted services are currently accessible on the local Western network only. A networking solution to expose the platform to students outside the local network has not yet been established and requires discussion with IT.

## Self-Hosted Infrastructure

All core application services run on a local machine as Docker containers orchestrated by Docker Compose. There is no managed cloud compute cost for these components — the only ongoing costs are hardware amortisation and electricity.

| Service          | Image                      | Resource notes                                    |
| ---------------- | -------------------------- | ------------------------------------------------- |
| Traefik          | `traefik:v3.0`             | Lightweight; negligible CPU/RAM                   |
| Frontend         | custom (Node 24 Alpine)    | Low memory footprint in production (SSR)          |
| Backend          | custom (JVM 17)            | ~256–512 MB heap typical for Spring Boot workload |
| MariaDB          | `mariadb:11.2`             | Storage grows with job and user records           |
| SeaweedFS (×4)   | `seaweedfs:latest / 4.07`  | Local disk for temporary file staging             |
| N8N (×3)         | custom                     | Queue mode; worker scales with job throughput     |
| OrcaSlicer CLI   | custom                     | CPU-bound during slicing; bursty usage            |
| PostgreSQL       | `postgres:16.1`            | N8N persistence; small dataset                    |
| Redis            | `redis:7-alpine`           | In-memory job queue; low memory for typical load  |

**Estimated hardware requirements (minimum):**

| Resource | Minimum         | Recommended           |
| -------- | --------------- | --------------------- |
| CPU      | 4 cores         | 8+ cores              |
| RAM      | 8 GB            | 16 GB                 |
| Storage  | 50 GB SSD       | 200 GB+ SSD           |
| Network  | 100 Mbps uplink | 1 Gbps uplink         |

---

## AWS Services

### AWS S3

Used for two purposes: permanent storage of validated 3D model and output files (application), and storage of training datasets and rendered images (AI pipeline).

| Cost component       | Rate (us-east-1, Standard)           |
| -------------------- | ------------------------------------ |
| Storage              | ~$0.023 / GB / month                 |
| PUT / COPY / POST    | ~$0.005 / 1,000 requests             |
| GET / SELECT         | ~$0.0004 / 1,000 requests            |
| Data transfer out    | First 100 GB/month free; ~$0.09 / GB after |

Cost is primarily driven by stored file volume and download frequency. For a small club workload (hundreds of 3D model files, occasional downloads), monthly S3 costs are expected to be **< $5/month**.

### AWS SES

Used to send OTP codes, email verification links, password reset links, and job status notifications.

| Cost component           | Rate                                 |
| ------------------------ | ------------------------------------ |
| Outbound emails          | $0.10 / 1,000 emails                 |
| Inbound emails           | $0.10 / 1,000 emails (if configured) |
| Attachments              | $0.12 / GB of attachment data        |

For a small club (tens to low hundreds of active users), monthly SES costs are expected to be **< $1/month**.

---

## AI Section

### Overview

The AI pipeline trains a vision-language model (VLM) to classify 3D model renders as `nsfw`, `sfw`, or `grey_area`. It involves dataset curation and storage, image rendering, model benchmarking and fine-tuning on cloud GPU, and serving the resulting model as a microservice.

---

### Dataset Storage (AWS S3)

Training data (raw STL files, rendered images, dataset manifests) is stored in a dedicated S3 bucket and versioned with DVC.

| Data type               | Estimated size                                  |
| ----------------------- | ----------------------------------------------- |
| Raw STL files (1,000 samples, multi-file) | Variable; STL files range from ~1 MB to 100+ MB per model. Budget ~50–200 GB |
| Rendered PNG images     | ~2–10 images per sample × 1,000 samples ≈ 2,000–10,000 PNGs. Budget ~5–20 GB at mid resolution |
| Dataset manifests       | Negligible (JSONL text files)                   |

**Estimated S3 cost for AI datasets: $1–$5/month** depending on total stored volume.

---

### Model Training (AWS EC2)

Fine-tuning is run on an **AWS EC2 GPU instance**. Training is a one-time or periodic job — the instance is started for the training run and stopped immediately after to avoid idle charges.

| Cost component      | Notes                                                                                         |
| ------------------- | --------------------------------------------------------------------------------------------- |
| Instance family     | G6 (NVIDIA L4), P3 (V100), P4 (A100), or P5 (H100) depending on model size and VRAM needs   |
| Recommended         | `p3.2xlarge` (1× V100 16 GB) or `g6.xlarge` (1× L4 24 GB) for LoRA/QLoRA with Unsloth       |
| On-demand pricing   | `p3.2xlarge`: ~$3.06/hr &nbsp;·&nbsp; `g6.xlarge`: ~$0.80/hr (us-east-1, on-demand)         |
| Spot pricing        | 60–80% discount available; suitable for fault-tolerant training jobs                         |
| Estimated duration  | Typical VLM LoRA/QLoRA fine-tune on ~500 samples: 1–6 hours depending on instance and model  |
| Estimated cost      | ~$1–$20 per training run (spot) · ~$3–$20 per training run (on-demand)                       |
| Frequency           | One-time for initial model; re-run only if dataset is significantly updated                   |

Benchmarking (zero-shot/few-shot inference on 500 samples per candidate model) is run on the same EC2 instance type or locally if hardware allows. Benchmarking is cheaper than training — inference-only passes with no gradient computation.

---

### Fine-Tuning Framework (Unsloth AI)

Unsloth AI is the fine-tuning library used to train the VLM. It is **open-source and free**. It provides memory-efficient LoRA/QLoRA fine-tuning for supported VLM architectures and reduces GPU VRAM requirements significantly compared to standard training.

| Item          | Cost     |
| ------------- | -------- |
| Unsloth AI    | Free (open source, Apache 2.0) |

---

### Model Serving (FastAPI + vLLM)

The fine-tuned model is served as a microservice via FastAPI and vLLM. This will be deployed on a serverless GPU runtime for cost efficiency. Outlined below are other deployment
options:

| Deployment option     | Cost model                                                                       |
| --------------------- | -------------------------------------------------------------------------------- |
| Self-hosted (local GPU) | No additional cloud cost; hardware amortisation only                            |
| Cloud GPU (e.g. Beam, RunPod, Lambda Labs) | Pay-per-hour; ~$0.50–$2/hour for a consumer-grade GPU suitable for VLM inference |
| Serverless inference  | Per-request billing; suitable for low-volume club usage                          |

For the current scope (club-scale, low-concurrency inference), a **low-cost cloud GPU instance is sufficient**. The inference service currently only needs to be live when the pipeline is vetting for NSFW models.

---

### DVC (Dataset Versioning)

DVC is **open-source and free**. It stores metadata/pointer files in git and pushes large binary assets to the S3 remote. No additional subscription is required beyond the S3 storage costs already accounted for above.

| Item | Cost     |
| ---- | -------- |
| DVC  | Free (open source) |

---

## Total Estimated Monthly Cost

| Item                          | Estimated monthly cost         |
| ----------------------------- | ------------------------------ |
| Self-hosted hardware          | Hardware + electricity (local) |
| AWS S3 (application files)    | < $5                           |
| AWS SES (email)               | < $1                           |
| AWS S3 (AI datasets)          | $1–$5                          |
| AWS EC2 (model training)      | $0 (one-time / periodic; not monthly) |
| Model serving (cloud GPU)     | $0–$30 (depends on deployment choice) |
| **Total cloud (steady state)** | **~$5–$40 / month**            |

Training costs are incurred once (or when the dataset is retrained) rather than monthly for EC2 instances used for fine tuning AI models. Steady-state cloud costs consists of S3 storage and the serverless payment model for GPU usage is pay-as-you-go.
