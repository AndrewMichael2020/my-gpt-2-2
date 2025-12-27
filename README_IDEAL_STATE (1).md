# Healthcare SQL Agent (Rewrite) – Ideal State (Model-First)

This repo trains and serves a **small-to-medium language model** that generates **T-SQL SELECT** queries for Microsoft SQL Server.

## What this is
- A **decoder-only LM** trained **from scratch** in Google Colab (L4).
- A minimal Python wrapper for:
  - dataset generation
  - training orchestration and checkpoints
  - inference prompt formatting
  - strict SQL validation
  - ID placeholder vault

## What this is not
- Not an API wrapper.
- Not a multi-agent SQL system.
- Not a general BI assistant.

## Core guarantee
Input: one clear, schema-aware question.

Output: one SQL statement:
- `SELECT ... FROM ... WHERE ... GROUP BY ... ORDER BY ... ;`

## Why IDs use placeholders
Healthcare SQL questions contain identifiers that must be exact.
We enforce correctness via:
- deterministic extraction
- placeholders in prompt and SQL
- post-generation reinjection
- strict validation

## Colab training constraints (target)
- GPU: L4 (~22 GB VRAM)
- Disk: ~240 GB
- RAM: ~56 GB
- Training is checkpointed to support <= 2-hour sessions.

## Repository structure (ideal state)
```
.
├─ data/
│  ├─ schemas/
│  │  └─ example_schema.json
│  ├─ train.jsonl
│  ├─ val.jsonl
│  └─ test.jsonl
├─ notebooks/
│  └─ colab_train.ipynb
├─ sql_agent/
│  ├─ __init__.py
│  ├─ id_vault.py          # extract + placeholders + reinjection
│  ├─ schema.py            # schema loader + helpers
│  ├─ prompt.py            # prompt formatter for the model
│  ├─ validate.py          # SELECT-only + schema + placeholder gates
│  ├─ generate.py          # generate_sql() loads a local checkpoint
│  ├─ cli.py               # python -m sql_agent "..."
│  └─ train/
│     ├─ model.py          # decoder-only architecture config
│     ├─ tokenizer.py      # tokenizer training + loading
│     ├─ dataset.py        # JSONL dataset loader
│     └─ train_loop.py     # PyTorch training loop + checkpointing
├─ scripts/
│  ├─ build_dataset.py     # schema-aware synthetic dataset generator
│  ├─ train_tokenizer.py
│  └─ eval.py              # evaluation harness
├─ checkpoints/            # gitignored
├─ requirements.txt
└─ README.md
```

## Dataset format (JSONL)
Each line is a single training example with explicit fields:
- schema_id
- schema_compact_text
- question_text
- id_map_text
- sql_text

The model is trained on a concatenated text form with stable markers.

## Inference (ideal CLI)
```
python -m sql_agent --schema data/schemas/example_schema.json "Question..."
```

## Validation rules
A query is accepted only if:
- exactly one statement
- SELECT-only
- ends with `;`
- schema-known tables and columns only
- placeholders only from the extracted ID vault
- no DML/DDL keywords

## Checkpointing
Training saves:
- model weights
- optimizer state
- step count
- tokenizer reference
This supports stop/resume for short training sessions.
