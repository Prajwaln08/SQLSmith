# SQLSmith

**On-premises text-to-SQL analytics copilot using small open language models**

> **Status: in progress.** Milestones A and B are done: baselines, a clean training set, and QLoRA fine-tuning of both models, scored on the full Spider dev set. Fine-tuning lifted **Qwen 1.5B from 42.5% to 59.4%** (+16.9 points) and **Qwen 7B from 73.2% to 75.1%** (+1.9 points). DPO, schema linking, self-correction and optimised serving are next. See [Progress](#progress) and [Roadmap](#roadmap).

## The problem

Business teams in banks, insurers and hospitals often wait days for an analyst to answer simple data questions such as *"How many patients enrolled last month, by clinic?"*. Large API models can write that SQL, but many of these companies are not allowed to send their database schemas or data to an external LLM provider.

SQLSmith asks one question: **can a 1.5B to 7B open model, running on one GPU inside the company network, get close to frontier-model accuracy at a fraction of the cost?**

**Who it helps**

- **Business users** (managers, operations, finance, clinical leads) get answers in seconds instead of waiting in an analyst's queue.
- **BI and analytics teams** spend less time on repetitive ad-hoc queries and more on real analysis.
- **Security and compliance teams** keep data, schemas and queries inside their own network, with no external API calls.

**What the problem costs today:** analyst hours spent on repetitive queries, slower decisions, and either large API bills or compliance risk.

## How it will be used

```
Business user asks a question in plain English
        |
        v
SQLSmith (open model on one GPU, inside the company network)
  1. Schema linking: finds the tables and columns the question needs
  2. Generates SQL with a fine-tuned small model
  3. Runs the SQL read-only; if it fails, retries with the error message
        |
        v
Answer table + the SQL it ran (so an analyst can check it)
```

The planned FastAPI service will let a BI tool, a chat app or an internal portal send a question and get back the SQL, the result and the time taken.

## Approach

1. **Build training data.** Use Spider (and later BIRD) training questions. Optionally distil extra examples from a frontier model, keeping only SQL whose execution result matches the gold result.
2. **Fine-tune.** Train Qwen2.5-Coder 1.5B and 7B with **QLoRA** (4-bit base model with LoRA adapters), then try **DPO** with pairs of correct and incorrect SQL.
3. **Improve at inference time.** Add **schema linking** (show the model only the relevant tables) and a **self-correction loop** that re-runs failed SQL with the database error message.
4. **Serve cheaply.** Serve with **vLLM**, quantise to **AWQ or GGUF**, and measure tokens per second, p95 latency and cost per 1,000 queries against a frontier API model.

## Progress

| Milestone | What it covers | Status |
| --- | --- | :---: |
| A. Baselines | Qwen2.5-Coder 1.5B and 7B on Spider with the official test-suite evaluator, plus error analysis | Done |
| A.1 Full benchmark | All 1,034 Spider dev questions for both models | Done |
| B.1 SFT dataset | Clean, database-disjoint training and validation sets | Done |
| B.2 QLoRA 1.5B | First fine-tuning run (250 steps) | Done |
| B.3 QLoRA 7B | Fine-tuning on a 16 GB T4 (200 steps) | Done |
| B.4 Fine-tuned evaluation | Base vs fine-tuned on the full Spider dev set | Done |
| C. DPO | Preference tuning on correct vs incorrect SQL | Planned |
| D. Schema linking + self-correction | Inference-time improvements | Planned |
| E. Serving and cost | vLLM, AWQ / GGUF, latency and cost per 1K queries | Planned |
| F. BIRD + healthcare set | Harder benchmark and a small synthetic healthcare schema | Planned |

## Results so far

### Headline: fine-tuned vs base, official Spider 1.0 dev set (1,034 questions)

Same frozen prompt, greedy decoding, 4-bit inference on one T4, and the same pinned official evaluator for all four runs. The re-scored baselines match Notebook 06 exactly, so the comparison is like for like.

| Model | Base | QLoRA fine-tuned | Change |
| --- | :---: | :---: | :---: |
| Qwen2.5-Coder-1.5B-Instruct | 42.5% | **59.4%** | **+16.9 points** |
| Qwen2.5-Coder-7B-Instruct | 73.2% | **75.1%** | **+1.9 points** |

**By difficulty**

| Model | Easy (248) | Medium (446) | Hard (174) | Extra hard (166) |
| --- | :---: | :---: | :---: | :---: |
| 1.5B base | 65.3% | 40.8% | 25.9% | 30.1% |
| 1.5B fine-tuned | **83.1%** | **61.2%** | **42.5%** | **36.7%** |
| 7B base | 88.3% | 78.0% | 65.5% | 45.8% |
| 7B fine-tuned | **89.5%** | **80.5%** | 64.4% | **50.6%** |

**Speed (generation on one T4, adapter attached but not merged)**

| Model | Mean latency | p95 latency | Tokens / second |
| --- | :---: | :---: | :---: |
| 1.5B base | 1.81 s | 3.29 s | 20.2 |
| 1.5B fine-tuned | 3.61 s | 7.58 s | 9.3 |
| 7B base | 1.90 s | 3.67 s | 15.0 |
| 7B fine-tuned | 3.91 s | 8.47 s | 8.1 |

**What this shows**

- **The small model gains the most.** Fine-tuning closed **55% of the gap** between the 1.5B and 7B base models (16.9 of 30.7 points), with gains at every difficulty level.
- **The 7B model gains less.** It improved on easy, medium and extra-hard questions, but dropped 1.1 points on hard ones. Its run was also lighter (LoRA rank 8, 200 steps, 1,024-token cap) to fit a T4, so the two gains are not a like-for-like comparison of model size.
- **Latency roughly doubled** with the LoRA adapter running unmerged on the 4-bit base model. Merging the adapter and serving with vLLM (Milestone E) are meant to recover this.
- No empty predictions in any of the four runs. A paired significance test on per-question results has not been run yet, so the +1.9-point 7B gain should be read with care.

### Baseline: official Spider 1.0 dev set (1,034 questions)

Execution accuracy with the official [test-suite evaluator](https://github.com/taoyds/test-suite-sql-eval) (`--etype exec`), full schema in the prompt, no examples, no fine-tuning. Both models run in 4-bit on a single Colab T4 GPU.

| Model | Execution accuracy | Mean latency | p95 latency | Tokens / second |
| --- | :---: | :---: | :---: | :---: |
| Qwen2.5-Coder-1.5B-Instruct | **42.5%** | 1.81 s | 3.29 s | 20.2 |
| Qwen2.5-Coder-7B-Instruct | **73.2%** | 1.90 s | 3.67 s | 15.0 |

**Qwen 7B by difficulty**

| Easy (248) | Medium (446) | Hard (174) | Extra hard (166) |
| :---: | :---: | :---: | :---: |
| 88.3% | 78.0% | 65.5% | 45.8% |

The 7B model peaks at about 6.6 GiB of GPU memory, so it fits comfortably on a 16 GB card.

### Error analysis (first 50 questions, 1.5B model)

26 failed queries were reviewed by hand. The most common causes:

- **Invalid column references (12):** the model used a column that does not exist in the table it chose.
- **Wrong table, join or aggregation (6):** for example an unnecessary join, or aggregating after the wrong join.
- **Validator rejections (5):** queries rejected by the query validator before or during replay.
- **Other (3):** wrong meaning of a term, wrong column order, and one possible issue in the benchmark's reference answer.

These findings shaped the roadmap: invalid columns point to **schema linking**, and runtime errors point to a **self-correction loop**.

### Fine-tuning dataset

| Step | Result |
| --- | --- |
| Spider training questions (`train_spider` + `train_others`) | 8,659 |
| After removing exact duplicates | 8,651 |
| After checking every gold query compiles in read-only SQLite | 8,648 |
| **Training split** | **7,931 examples, 131 databases** |
| **Validation split** (split by database, not by question) | **717 examples, 15 databases** |
| Spider dev set (held out for final scoring only) | 1,034 examples, 20 databases |

Checks pass for no database overlap between training, validation and dev, so the model is never scored on a database it trained on.

### QLoRA training runs

Both runs use a 4-bit NF4 base model with LoRA adapters on all attention and MLP projections, the same database-disjoint training set from Notebook 07, and loss on the SQL answer tokens only (so the model is not rewarded for repeating the question or schema). Long examples are dropped, never truncated.

| Setting | Qwen 1.5B (Notebook 08) | Qwen 7B (Notebook 09) |
| --- | :---: | :---: |
| LoRA rank / alpha | 16 / 32 | 8 / 16 |
| Trainable parameters | **18.5M (1.18%)** | **20.2M (0.26%)** |
| Max sequence length | 1,536 tokens | 1,024 tokens |
| Training examples kept | 7,849 (82 too long) | 7,849 (82 too long) |
| Learning rate | 2e-4, cosine | 1e-4, cosine |
| Effective batch size | 8 | 16 |
| Steps (share of one epoch) | 250 (0.25) | 200 (0.41) |
| Training time on a T4 | about 26 minutes | about 1 hour 54 minutes |
| Peak GPU memory | 3.8 GiB | 10.3 GiB |
| Final training loss | 0.191 | 0.135 |
| Validation loss | 0.202 (128 examples) | 0.104 (58 examples, 14 of 15 databases) |

The 7B settings were reduced on purpose to fit a 16 GB T4 (gradient checkpointing, micro-batch 1). 7B validation loss settled at 0.104 to 0.105 from step 150 to step 200 with no upward trend, so the run showed no sign of over-fitting within this budget. Loss is not accuracy: the execution scores above come from Notebook 10.

## Evaluation rules

- **Execution accuracy, not string match.** A query counts as correct if it returns the right result on the official test-suite databases.
- **No leakage.** The Spider dev set is never used for training or for choosing checkpoints, and gold SQL is never shown in the prompt.
- **Same prompt and settings** for every model, so differences come from the model, not the setup.
- **Frozen baselines.** Fine-tuned models are scored with the same prompt, decoding, evaluator commit and test-suite databases as the baselines, and the baselines are re-scored in the same run as a check.
- **Honest uncertainty.** Small samples are reported with Wilson 95% intervals. Significance is only claimed after a paired test on per-question results.

## Roadmap

- [x] Finish the 7B QLoRA run and score both fine-tuned models on the full Spider dev set
- [ ] Paired significance test (McNemar) on per-question results, base vs fine-tuned
- [ ] Error analysis of the fine-tuned models, including the 7B drop on hard questions
- [ ] Longer or higher-rank 7B run on a larger GPU, to match the 1.5B settings
- [ ] Ablation table: base, +SFT, +DPO, +schema linking, +self-correction
- [ ] Error taxonomy before and after each step (wrong join, wrong aggregation, hallucinated column)
- [ ] Serve with vLLM; compare AWQ and GGUF quantisation on speed, memory and accuracy
- [ ] Cost and quality chart against a frontier API model (cost per 1,000 queries vs accuracy)
- [ ] BIRD benchmark and a small set of hard healthcare-style questions on a synthetic schema
- [ ] FastAPI service with read-only database access

**Business metrics to report:** share of business questions answered correctly without an analyst, cost per 1,000 queries, and time to answer.

## Notebooks

| # | Notebook | Purpose |
| :---: | --- | --- |
| 01 | [SQLSmith_Baseline](notebooks/01_SQLSmith_Baseline.ipynb) | First 1.5B baseline on 50 Spider questions |
| 02 | [Spider_Error_Analysis](notebooks/02_Spider_Error_Analysis.ipynb) | Hand review of failed queries |
| 03 | [Spider_Official_Evaluation](notebooks/03_Spider_Official_Evaluation.ipynb) | Official test-suite evaluator set-up |
| 04 | [Qwen_7B_Baseline](notebooks/04_Qwen_7B_Baseline.ipynb) | 7B baseline on the same 50 questions |
| 05 | [Milestone_A_Report](notebooks/05_Milestone_A_Report.ipynb) | 1.5B vs 7B report with intervals and charts |
| 06 | [Full_Spider_Benchmark](notebooks/06_Full_Spider_Benchmark.ipynb) | Both models on all 1,034 dev questions |
| 07 | [SFT_Dataset_Preparation](notebooks/07_SFT_Dataset_Preparation.ipynb) | Clean, database-disjoint training data |
| 08 | [QLoRA_SFT_1_5B](notebooks/08_QLoRA_SFT_1_5B.ipynb) | QLoRA fine-tuning of the 1.5B model |
| 09 | [QLoRA_SFT_7B](notebooks/09_QLoRA_SFT_7B.ipynb) | QLoRA fine-tuning of the 7B model (200 steps) |
| 10 | [SFT_Comparative_Evaluation](notebooks/10_SFT_Comparative_Evaluation.ipynb) | Base vs fine-tuned, official evaluation of all four runs |

## Tech stack

| Purpose | Tools |
| --- | --- |
| Models | Qwen2.5-Coder 1.5B and 7B Instruct |
| Fine-tuning | PyTorch, Hugging Face Transformers, PEFT (LoRA / QLoRA), bitsandbytes; TRL and Unsloth planned for DPO |
| Evaluation | Official Spider test-suite evaluator, SQLite, sqlglot, pandas |
| Serving (planned) | vLLM, AWQ / GGUF quantisation, FastAPI, DuckDB or PostgreSQL |
| Tracking | Google Drive artifacts with checksums; Weights & Biases planned |
| Hardware | Google Colab T4 (16 GB); an A10 or L4 is enough for the later stages |

## Datasets

- [Spider 1.0](https://yale-lily.github.io/spider): cross-database text-to-SQL benchmark (CC BY-SA 4.0).
- [BIRD](https://bird-bench.github.io/): larger, harder benchmark with real-world databases (planned).
- A small synthetic healthcare schema with hard questions (planned). No real patient data is used.

Raw dataset files are not stored in this repository. The notebooks download them.

## Run it

1. Open a notebook from the table above in Google Colab.
2. Choose a **T4 GPU** runtime for generation and training notebooks; scoring and report notebooks run on CPU.
3. Run the cells in order. Results are saved to your Google Drive with checksums so runs can resume after a disconnect.

## Limitations

- Results so far are on Spider only; BIRD and real business schemas are harder.
- Spider dev scores are a development benchmark, not a production accuracy figure. Spider dev was never used for training or checkpoint choice, but it is not a hidden test set.
- The 1.5B and 7B runs used different training settings, so their gains do not isolate the effect of model size.
- Fine-tuned latency is measured with an unmerged adapter; it is not yet an optimised serving number.
- Some GPU operations are not bit-for-bit reproducible across Colab sessions, even with fixed decoding.

## Author

**Prajwal Nerkar**, Data Scientist | AI/ML Engineer
[LinkedIn](https://linkedin.com/in/prajwal-nerkar) · [GitHub](https://github.com/Prajwaln08)
