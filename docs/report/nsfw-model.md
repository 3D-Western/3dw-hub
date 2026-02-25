# Overview

This document details data curation, sampling, and training process for the NSFW model, a vision-language model (VLM) fine-tuned to classify 3D models as NSFW, SFW, or grey area 3D files for incoming print requests. Images are rendered from STL files using a Python OpenSCAD script and fed into the model for classification.

---

## Technical Specifications 

| Component             | Technology          | Purpose                                                        |
| --------------------- | ------------------- | -------------------------------------------------------------- |
| Image extraction      | Python + OpenSCAD   | Render STL files into images for VLM input                     |
| Fine-tuning framework | Unsloth AI          | Memory-efficient fine-tuning of open-source VLM candidates     |
| Training runtime      | Beam                | Cloud compute runtime for fine-tuning jobs                     |
| Containerisation      | Docker              | Reproducible training and serving environments                 |
| Model serving         | FastAPI + vLLM      | High-throughput inference serving for the fine-tuned VLM       |
| Dataset versioning    | DVC                 | Version-control datasets stored in S3                          |
| Dataset storage       | AWS S3              | Single bucket for all raw data, rendered images, and manifests |

---

## Data Collection & Sampling

### Sample Definition

A **sample** is one 3D model (identified by a unique ID). A single sample may contain more than one file (e.g., multiple STL parts for the same model). All files belonging to a sample are kept together and treated as one unit during sampling and splitting.

### Label Categories

| Label       | Description                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| `nsfw`      | Model contains explicit adult content (nudity, sexual content, gore, etc.)  |
| `sfw`       | Model is clearly safe for all audiences                                     |
| `grey_area` | Model is ambiguous — potentially offensive but not definitively NSFW        |

### Raw Collection Targets

| Label       | Target Samples |
| ----------- | -------------- |
| `nsfw`      | 400            |
| `sfw`       | 300            |
| `grey_area` | 300            |
| **Total**   | **1,000**      |

### Sampling Process

```
┌─────────────────────────────────────────────────────────────────┐
│  Collect raw samples                                            │
│  400 nsfw + 300 sfw + 300 grey_area = 1,000 samples             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Concatenate into full 1,000-sample dataset                     │
│  Assign schema, extract images via OpenSCAD                     │
│  Version with DVC → push to S3                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              Random sample 500 (stratified)
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  Static benchmark / validation dataset (500 samples)            │
│  Fixed — never modified after creation                          │
│  Version with DVC → push to S3                                  │
└─────────────────────────────────────────────────────────────────┘
```

The 500-sample benchmark dataset is drawn once via random sampling from the full 1,000-sample dataset and then frozen. It is used exclusively for benchmarking pretrained VLM candidates before fine-tuning begins (see [Model Selection & Benchmarking](#model-selection--benchmarking)).

---

## Dataset Schemas

### Full Dataset (1,000 samples)

Stored as a manifest file (e.g., `dataset_full.jsonl`) where each line is one sample.

| Field           | Type            | Description                                                                 |
| --------------- | --------------- | --------------------------------------------------------------------------- |
| `sample_id`     | `string`        | Unique identifier for the sample (e.g., UUID or slug)                       |
| `label`         | `string`        | Classification label: `nsfw`, `sfw`, or `grey_area`                         |
| `files`         | `array[string]` | List of S3 keys for the raw STL files belonging to this sample              |
| `images`        | `array[string]` | List of S3 keys for the rendered image(s) extracted from the STL files      |
| `source`        | `string`        | Origin of the sample (e.g., dataset name, scrape source, manual curation)   |
| `annotator`     | `string`        | ID or name of the person who labelled this sample                           |
| `annotated_at`  | `string`        | ISO 8601 timestamp of when the label was assigned                           |
| `notes`         | `string`        | Optional free-text notes (e.g., reason for grey_area classification)        |
| `split`         | `string`        | Dataset membership: `benchmark` (in 500-sample set) or `train_pool`         |

### Benchmark Dataset (500 samples)

Stored as a separate manifest (e.g., `dataset_benchmark.jsonl`). This is a strict subset of the full dataset. Fields are identical to the full dataset schema above, with the following constraints:

| Field       | Constraint                                          |
| ----------- | --------------------------------------------------- |
| `sample_id` | Must exist in the full dataset                      |
| `split`     | Always `benchmark`                                  |
| `label`     | Reflects the same label as in the full dataset      |

The benchmark manifest is generated once and must not be modified after creation. Any addition, removal, or re-labelling requires creating a new versioned benchmark.

---

## S3 Bucket Layout

All data lives in a single S3 bucket. DVC tracks dataset manifests and large binary assets.

```
s3://<bucket>/
├── raw/
│   └── stl/
│       └── <sample_id>/
│           └── <filename>.stl          # Raw STL files, grouped by sample
├── rendered/
│   └── <sample_id>/
│       └── <view>.png                  # Images rendered by OpenSCAD script
├── datasets/
│   ├── dataset_full.jsonl              # Full 1,000-sample manifest
│   └── dataset_benchmark.jsonl         # Static 500-sample benchmark manifest
└── dvc/
    └── ...                             # DVC cache and remote metadata
```

---

## DVC Setup

DVC is used to version-control the datasets and rendered images stored in S3, keeping large binaries out of git.

```bash
dvc init
dvc remote add -d s3remote s3://<bucket>/dvc
dvc add datasets/dataset_full.jsonl
dvc add datasets/dataset_benchmark.jsonl
dvc push
```

Dataset manifest files and their `.dvc` pointer files are committed to git. The actual binary data (STL files, rendered images) is pushed to S3 via DVC.

---

## Image Extraction

Images are rendered from STL files using a Python script that invokes OpenSCAD programmatically. Each STL file in a sample is rendered from one or more camera angles to produce representative images for VLM input.

| Parameter        | Value / Description                                      |
| ---------------- | -------------------------------------------------------- |
| Renderer         | OpenSCAD (headless)                                      |
| Output format    | PNG                                                      |
| Views per file   | Configurable (e.g., front, side, isometric)              |
| Output location  | `s3://<bucket>/rendered/<sample_id>/<view>.png`          |

---

## Model Selection & Benchmarking

Before fine-tuning, several open-source VLM candidates are evaluated on the static 500-sample benchmark dataset to identify the most suitable base model for this classification task.

### Process

| Step | Action                                                                                       |
| ---- | -------------------------------------------------------------------------------------------- |
| 1    | Select a shortlist of open-source VLM candidates (e.g., LLaVA, Qwen-VL, InternVL variants) |
| 2    | Run zero-shot or few-shot inference on the 500-sample benchmark for each candidate           |
| 3    | Compute classification metrics per candidate (see table below)                               |
| 4    | Choose the candidate with the best overall performance as the fine-tuning base model         |

### Evaluation Metrics

| Metric                  | Description                                                        |
| ----------------------- | ------------------------------------------------------------------ |
| Accuracy                | Overall fraction of correctly classified samples                   |
| Precision (per class)   | Of samples predicted as a class, how many were correct             |
| Recall (per class)      | Of samples that belong to a class, how many were identified        |
| F1 Score (per class)    | Harmonic mean of precision and recall per class                    |
| Macro F1                | Unweighted mean F1 across all three classes                        |
| Confusion matrix        | Full 3×3 matrix (nsfw / sfw / grey_area)                           |

---

## Fine-Tuning

Once the base model is selected, it is fine-tuned using Unsloth AI on the remaining training pool (samples not in the 500-sample benchmark).

### Training Configuration

| Parameter             | Description                                                          |
| --------------------- | -------------------------------------------------------------------- |
| Base model            | Chosen VLM candidate from benchmarking step                          |
| Fine-tuning framework | Unsloth AI (LoRA / QLoRA)                                            |
| Training runtime      | Beam (cloud GPU)                                                     |
| Containerisation      | Docker (reproducible training environment)                           |
| Training data         | Samples with `split = train_pool` from the full 1,000-sample dataset |
| Validation data       | Static 500-sample benchmark dataset                                  |
| Input format          | Rendered PNG image(s) + label                                        |
| Output classes        | `nsfw`, `sfw`, `grey_area`                                           |

### Training Data Split Summary

| Split          | Source                               | Size          | Purpose                       |
| -------------- | ------------------------------------ | ------------- | ----------------------------- |
| `benchmark`    | Random 500 from full dataset (fixed) | 500 samples   | Model selection + validation  |
| `train_pool`   | Remaining 500 from full dataset      | ~500 samples  | Fine-tuning                   |

---

## Model Serving

The fine-tuned model is served via **FastAPI + vLLM** for high-throughput inference.

| Component    | Technology    | Purpose                                            |
| ------------ | ------------- | -------------------------------------------------- |
| Inference    | vLLM          | Optimised batched inference for VLM                |
| API layer    | FastAPI       | REST endpoint for classification requests          |
| Runtime      | Docker        | Containerised serving environment                  |
