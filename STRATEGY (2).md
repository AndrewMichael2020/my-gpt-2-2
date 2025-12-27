# Healthcare SQL Agent (Rewrite) – High-Level Strategy (Model-First)

## Goal
Train a small-to-medium **decoder-only language model** from scratch that takes **one clear, schema-aware question** and returns **one** safe, executable **T-SQL SELECT** statement for **Microsoft SQL Server**.

A thin Python wrapper exists only to:
- build datasets,
- run training in Google Colab,
- run inference,
- validate output.

## Hard Product Constraints
- Output must be exactly **one** statement.
- Output must be **SELECT-only** and end with `;`
- Allowed shape: `SELECT ... FROM ... WHERE ... GROUP BY ... ORDER BY ... ;`
- No DML/DDL, no temp tables, no stored procedures, no multi-statement output.
- Input is **not ambiguous** and assumes the user knows the schema.

## Target Compute (your stated budget)
Training environment: **Google Colab L4**
- VRAM: ~22 GB
- Disk: ~240 GB
- RAM: ~56 GB
- Training should be runnable in **<= 2-hour sessions** (checkpoint-resumable).

## Model Size Targets (fits L4 realistically)
Pick a size that is realistic to train repeatedly on L4.
- **Tier A (recommended): ~300M parameters**
- **Tier B (optional): ~700M parameters**

The objective is correctness for a narrow SQL form, not broad language capability.

## Data Strategy (Training From Scratch)
Quality comes from the dataset. We build a synthetic + templated corpus from:
1. **Schema bundle(s)**: tables, columns, keys, join paths.
2. **Question templates**: unambiguous, schema-aware.
3. **SQL templates**: deterministic T-SQL SELECT patterns.
4. **Diversity knobs**: aliases, optional filters, grouping variations, ordering variations.

### Training Record Format (one example)
Each sample is a single text record with stable markers:
- `SCHEMA:` (compact)
- `QUESTION:` (schema-aware)
- `ID_MAP:` (placeholder mapping)
- `SQL:` (single SELECT)

## Best-working ID Passing Pattern (what we implement)
**ID Vault + Placeholder Contract**:
1. Extract IDs/dates deterministically from the question.
2. Replace them with placeholders (`__ID_1__`, `__DATE_1__`).
3. Provide the placeholder map in the prompt.
4. Force the model to output placeholders exactly.
5. Validate placeholder integrity, then reinject exact values.

Why this is strong:
- The model learns structure without being asked to copy fragile identifiers.
- Exactness is enforced deterministically.

## Training Plan (<= 2-hour sessions)
### Phase 0: Sanity
- Generate 5k–20k samples.
- Train a tiny model (50–100M) briefly.
- Confirm end-to-end works.

### Phase 1: Main
- Dataset: 200k–800k samples (depending on diversity and disk).
- Model: ~300M decoder-only.
- Seq length: 512–1024.
- Checkpoint every N steps and resume across sessions.

### Phase 2: Quality
- Expand join patterns and aggregates.
- Add held-out evaluation set (500–2,000 questions).

## Inference Contract
Input:
- schema bundle path
- one question

Pipeline:
1. ID vault extraction -> placeholders
2. Prompt = schema + question + id_map + strict rules
3. Model generates SQL
4. Validator enforces:
   - one statement
   - SELECT-only
   - schema-known entities
   - placeholder integrity
   - ends with `;`
5. Renderer reinjects exact IDs

Output:
- a single validated SELECT statement

## Success Metrics
- Structural validity rate (passes gates)
- Schema validity rate
- Placeholder exactness (100%)
- Task accuracy on held-out questions

## Definition of Done (V1)
Given a schema-aware question, output a single valid T-SQL SELECT query with exact ID fidelity and strict constraint enforcement.
