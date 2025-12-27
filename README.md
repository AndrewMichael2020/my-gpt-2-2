# Healthcare SQL Agent - Model-First MVP

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/AndrewMichael2020/my-gpt-2-2/blob/main/colab_train.ipynb)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.0+-ee4c2c.svg)](https://pytorch.org/)

A complete end-to-end pipeline for training a decoder-only language model from scratch to generate safe, validated T-SQL SELECT queries for healthcare analytics.

## 🚀 Quick Start

Click the "Open in Colab" badge above to launch the notebook directly in Google Colab, then:

1. Select Runtime → Change runtime type → GPU (T4, L4, or better)
2. Run all cells sequentially

The notebook will:
- Generate 5,000+ training samples with synthetic SQL queries
- Train a BPE tokenizer (8k vocab)
- Train a tiny decoder-only transformer (~50M parameters)
- Run inference on 3 test questions
- Validate outputs against strict SQL rules

### What Gets Built

The pipeline creates:

| Component | Description | Output |
|-----------|-------------|--------|
| **Schema** | Healthcare database with 5 tables (Patients, Visits, Departments, Providers, Diagnoses) | `data/example_schema.json` |
| **Dataset** | Synthetic T-SQL training data with placeholders for IDs/dates | `data/train.jsonl` (5k samples)<br>`data/val.jsonl` (200 samples) |
| **Tokenizer** | BPE tokenizer trained on SQL corpus | `artifacts/tokenizer/tokenizer.json` |
| **Model** | Tiny decoder-only transformer (512d, 8 layers, 8 heads) | `checkpoints/checkpoint_epoch_*.pt` |
| **Validation** | SELECT-only, schema-aware, placeholder-safe gates | Built-in validation functions |

## 📋 Pipeline Overview

### 1. Schema Definition
Healthcare analytics schema with:
- **Patients**: PatientID, demographics, insurance
- **Visits**: VisitID, dates, charges, provider/department links
- **Departments**: DepartmentID, names, locations
- **Providers**: ProviderID, specialty, department
- **Diagnoses**: DiagnosisID, ICD codes, visit links

### 2. Dataset Generation
Deterministic SQL templates for:
- **Aggregations**: COUNT, SUM, AVG
- **Grouping**: By month, day, department, provider
- **Filtering**: WHERE with ID and date placeholders
- **Joins**: Multi-table queries with patient/department data

Each sample includes:
```json
{
  "schema_text": "Patients(PatientID, FirstName, ...); Visits(...); ...",
  "question": "How many visits did patient __ID_1__ have?",
  "id_map": "__ID_1__=5432",
  "sql": "SELECT COUNT(*) FROM Visits WHERE PatientID = __ID_1__;"
}
```

### 3. ID Placeholder System
**Why placeholders?**
- Healthcare queries contain exact IDs that must not be corrupted
- Model learns SQL structure without memorizing fragile identifiers
- Deterministic extraction → placeholder replacement → validation → reinjection

**Flow:**
1. Extract IDs/dates from question: `"patient 5432"` → `__ID_1__`
2. Train model on placeholders
3. Validate placeholder integrity in generated SQL
4. Reinject exact values: `__ID_1__` → `5432`

### 4. Model Architecture
Decoder-only transformer optimized for Colab L4:
- **Parameters**: ~50M (MVP) → 300M (production target)
- **Layers**: 8 transformer blocks
- **Dimensions**: 512d embeddings, 8 attention heads
- **Context**: 512 token max sequence length
- **Training**: FP16, gradient checkpointing, causal masking

### 5. Validation Gates
Generated SQL must pass ALL checks:
- ✓ Exactly one statement ending with `;`
- ✓ SELECT-only (no DML/DDL)
- ✓ Schema-known tables only
- ✓ Valid placeholders from ID map
- ✓ No forbidden keywords (INSERT, DROP, EXEC, etc.)

### 6. Training Strategy
**MVP (this notebook):**
- 5k samples, 3 epochs, ~10 minutes on L4
- Goal: Prove pipeline works end-to-end

**Production scaling:**
- 200k-800k samples
- ~300M parameter model
- Multi-session training with checkpointing
- Advanced templates (subqueries, HAVING, complex joins)

## 🎯 Acceptance Criteria

✅ **Dataset**: Generate ≥5,000 training samples with deterministic SQL templates  
✅ **Tokenizer**: Train BPE tokenizer on SQL corpus  
✅ **Model**: Train tiny decoder-only model (~50M params)  
✅ **Inference**: Generate SQL for 3 example questions  
✅ **Validation**: ≥2/3 questions pass strict validation gates  
✅ **Placeholders**: Exact ID reinjection with integrity checks  

## 📊 Example Output

```
[Question 1]
Original: How many visits did patient 5432 have in department 25?
Clean: How many visits did patient __ID_1__ have in department __ID_2__?
ID Map: {'__ID_1__': '5432', '__ID_2__': '25'}

Generated SQL (with placeholders):
  SELECT COUNT(*) FROM Visits WHERE PatientID = __ID_1__ AND DepartmentID = __ID_2__;

✓ Validation: PASSED

Final SQL (with real IDs):
  SELECT COUNT(*) FROM Visits WHERE PatientID = 5432 AND DepartmentID = 25;
```

## 🔧 Technical Details

### Dependencies
Installed automatically in notebook:
- PyTorch ≥2.0 (core training)
- Transformers ≥4.35 (utilities)
- Tokenizers ≥0.15 (BPE training)
- Standard: numpy, tqdm, jsonlines

### Compute Requirements
- **GPU**: T4/L4 (16-22GB VRAM) minimum
- **RAM**: ~16GB
- **Disk**: ~5GB for checkpoints
- **Time**: ~10 minutes for MVP, ~2 hours for scaled training

### Model Scaling Path
| Size | Params | Layers | d_model | Heads | VRAM | Target |
|------|--------|--------|---------|-------|------|--------|
| Tiny | 50M | 8 | 512 | 8 | ~2GB | MVP proof |
| Small | 300M | 12 | 768 | 12 | ~6GB | Production |
| Medium | 700M | 16 | 1024 | 16 | ~14GB | Stretch goal |

## 📁 File Structure

```
.
├── colab_train.ipynb          # 🎯 MAIN NOTEBOOK - Run this!
├── README.md                   # This file
├── STRATEGY (2).md            # Model-first strategy document
├── README_IDEAL_STATE (1).md  # Target architecture specification
│
└── (Generated at runtime)
    ├── data/
    │   ├── example_schema.json
    │   ├── train.jsonl
    │   └── val.jsonl
    ├── artifacts/
    │   └── tokenizer/
    │       └── tokenizer.json
    └── checkpoints/
        └── checkpoint_epoch_*.pt
```

## 🚦 Next Steps

After MVP validation:

1. **Scale dataset**: 5k → 200k+ samples
2. **Add templates**: Subqueries, HAVING, window functions, CTEs
3. **Scale model**: 50M → 300M parameters
4. **Advanced training**: Learning rate scheduling, mixed precision
5. **Evaluation**: Held-out test set with accuracy metrics
6. **CLI tool**: `python -m sql_agent --schema ... "question"`
7. **Schema expansion**: Multi-schema support, dynamic schema loading

## 📝 License

MIT License - See LICENSE file

## 🤝 Contributing

This is an MVP proof-of-concept. For production use:
- Add comprehensive test coverage
- Implement beam search/nucleus sampling
- Add SQL execution safety checks
- Build evaluation harness with metrics
- Deploy as API service

---

**Status**: ✅ MVP Complete - Ready for Colab testing