# SQLSmith

**On-premises text-to-SQL analytics copilot using small open language models**

> **Status: in progress.** Baselines on the full Spider benchmark, the training dataset and the first QLoRA fine-tuning run are done. Fine-tuned model evaluation, DPO, schema linking, self-correction and optimised serving are next. See [Progress](#progress) and [Roadmap](#roadmap).

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
| B.3 QLoRA 7B | Fine-tuning on a 16 GB T4 | In progress (smoke test done) |
| B.4 Fine-tuned evaluation | Base vs fine-tuned on the full Spider dev set | Pipeline built, results pending |
| C. DPO | Preference tuning on correct vs incorrect SQL | Planned |
| D. Schema linking + self-correction | Inference-time improvements | Planned |
| E. Serving and cost | vLLM, AWQ / GGUF, latency and cost per 1K queries | Planned |
| F. BIRD + healthcare set | Harder benchmark and a small synthetic healthcare schema | Planned |

## Results so far

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

### QLoRA run: Qwen2.5-Coder-1.5B

- 4-bit NF4 base model with LoRA adapters: **18.5M trainable parameters (1.18% of 1.56B)**.
- Loss on the SQL answer tokens only, so the model is not rewarded for repeating the question or schema.
- 250 steps in about 26 minutes on a T4, peak 3.8 GiB GPU memory.
- Training loss 0.191, validation loss 0.202.

Loss is not accuracy. The execution accuracy of the fine-tuned model will be reported once Notebook 10 finishes.

## Evaluation rules

- **Execution accuracy, not string match.** A query counts as correct if it returns the right result on the official test-suite databases.
- **No leakage.** The Spider dev set is never used for training or for choosing checkpoints, and gold SQL is never shown in the prompt.
- **Same prompt and settings** for every model, so differences come from the model, not the setup.
- **Honest uncertainty.** Small samples are reported with Wilson 95% intervals. Paired significance tests will only be claimed once per-question results exist for both models.

## Roadmap

- [ ] Finish the 7B QLoRA run and score both fine-tuned models on the full Spider dev set
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
| 09 | [QLoRA_SFT_7B](notebooks/09_QLoRA_SFT_7B.ipynb) | QLoRA fine-tuning of the 7B model (in progress) |
| 10 | [SFT_Comparative_Evaluation](notebooks/10_SFT_Comparative_Evaluation.ipynb) | Base vs fine-tuned evaluation (in progress) |

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
- Spider dev scores are a development benchmark, not a production accuracy figure.
- Some GPU operations are not bit-for-bit reproducible across Colab sessions, even with fixed decoding.

## Author

**Prajwal Nerkar**, Data Scientist | AI/ML Engineer
[LinkedIn](https://linkedin.com/in/prajwal-nerkar) · [GitHub](https://github.com/Prajwaln08)
