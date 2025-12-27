# Quick Start Guide - Healthcare SQL Agent

## Running the Notebook

### Option 1: Google Colab (Recommended)
1. Open `colab_train.ipynb` in Google Colab
2. Runtime → Change runtime type → GPU (T4 or better)
3. Run all cells in order (Runtime → Run all)
4. Wait ~10-15 minutes for training (3 epochs)

### Option 2: Local Jupyter
1. Install dependencies: `pip install torch transformers tokenizers datasets tqdm jsonlines`
2. Start Jupyter: `jupyter notebook colab_train.ipynb`
3. Run all cells in order
4. Requires GPU (CUDA) for reasonable training time

## What to Expect

### Training Output
```
Generating training dataset...
Generated 5000 training samples
Generated 200 validation samples

Training BPE tokenizer...
Vocab size: 8000

Starting training...
Epoch 1: Train Loss = 2.1234, Val Loss = 1.9876
Epoch 2: Train Loss = 1.8765, Val Loss = 1.7654
Epoch 3: Train Loss = 1.6543, Val Loss = 1.5432
Training complete!
```

### Demo Output (Cell 22)
```
[Question 1]
Original: How many visits did patient 5432 have in department 25?
Clean: How many visits did patient __ID_1__ have in department __ID_2__?
ID Map: {'__ID_1__': '5432', '__ID_2__': '25'}

Generated SQL (with placeholders):
  SELECT COUNT(*) FROM Visits WHERE PatientID = __ID_1__ AND DepartmentID = __ID_2__;
✓ Validation: PASSED

RESULTS: 3/3 questions passed validation
✓ SUCCESS: Basic acceptance criteria met
```

### Evaluation Output (Cell 24)
```
Evaluating: 100%|████████████| 100/100

EVALUATION RESULTS
Total questions: 100
Passed: 92
Failed: 8
Pass rate: 92.0%

✓✓✓ EXCELLENT: Pass rate >= 90% - Target achieved!
```

## Troubleshooting

### Issue: "No checkpoint found"
**Cause**: Training hasn't been run yet
**Solution**: Run Cell 16 first (training loop)

### Issue: Pass rate < 90%
**Causes**: 
- Model undertrained (only 3 epochs)
- Random seed variation
- Dataset too simple/complex

**Solutions**:
1. Increase epochs: Change `num_epochs = 3` to `num_epochs = 10` in Cell 16
2. Increase training samples: Change `5000` to `10000` in Cell 8
3. Lower temperature: Change `temperature=0.1` to `temperature=0.05` in Cell 18

### Issue: Out of Memory (OOM)
**Cause**: Batch size too large for GPU
**Solution**: In Cell 14, change `batch_size=8` to `batch_size=4` or `batch_size=2`

### Issue: Tokenizer splitting placeholders
**Cause**: Special tokens not loaded properly
**Solution**: Re-run Cell 10 (tokenizer training) and Cell 14 (dataset creation)

## Key Configuration Options

### Model Size (Cell 12)
```python
MODEL_CONFIG = {
    "vocab_size": tokenizer.get_vocab_size(),
    "d_model": 256,        # ← Increase for larger model (512, 768)
    "n_heads": 8,          # ← Must divide d_model evenly
    "n_layers": 6,         # ← More layers = better quality, slower
    "max_seq_len": 512,
    "dropout": 0.1
}
```

### Training Duration (Cell 16)
```python
num_epochs = 3  # ← Increase for better quality (5, 10, 20)
```

### Generation Settings (Cell 18)
```python
def generate_sql(..., max_length=256, temperature=0.1, ...):
    # max_length: Max tokens to generate (256 is usually enough)
    # temperature: Lower = more deterministic (0.05 - 0.3)
```

### Dataset Size (Cell 8)
```python
train_samples = generate_dataset_samples(EXAMPLE_SCHEMA, 5000)  # ← Increase to 10000, 20000
val_samples = generate_dataset_samples(EXAMPLE_SCHEMA, 200)    # ← Keep at 200-500
```

## Expected Timings (on T4 GPU)

| Task | Time |
|------|------|
| Dataset generation | ~10 sec |
| Tokenizer training | ~20 sec |
| Training (3 epochs) | ~8-12 min |
| Demo (3 questions) | ~5 sec |
| Evaluation (100 questions) | ~1-2 min |
| **Total** | **~10-15 min** |

## Validation Rules

The model output must satisfy:
1. ✓ Starts with `SELECT` (case-insensitive)
2. ✓ Ends with exactly one `;`
3. ✓ No DML/DDL keywords (INSERT, UPDATE, DELETE, DROP, etc.)
4. ✓ All placeholders (`__ID_1__`, etc.) exist in ID map
5. ✓ Contains at least one known table name from schema

## Next Steps After Success

1. **Extend schema**: Add more tables/columns to `EXAMPLE_SCHEMA` (Cell 6)
2. **Add templates**: Create more question/SQL pairs in `templates` list (Cell 8)
3. **Larger model**: Increase `d_model`, `n_layers` in `MODEL_CONFIG` (Cell 12)
4. **More training**: Increase `num_epochs` and dataset size
5. **Fine-tune**: Collect real questions and fine-tune the model

## Getting Help

- Check `VALIDATION_IMPROVEMENTS.md` for detailed technical documentation
- Review error messages in validation output
- Run evaluation (Cell 24) to see failure categories
- Check `data/eval_results.json` for detailed failure examples
