# DSPy Examples — Programming (not Prompting) Language Models

A small, focused collection of standalone [DSPy](https://dspy.ai/) examples, each demonstrating one core DSPy pattern end-to-end: structured output extraction, chain-of-thought reasoning, retrieval-augmented generation, tool-using ReAct agents, and automatic prompt/pipeline optimization.

Every script in this repo is self-contained and runnable on its own — there's no shared package structure to navigate. Pick the concept you want to learn, open that file, and run it.

---

## Table of Contents

1. [What is DSPy?](#what-is-dspy)
2. [What's in This Repo](#whats-in-this-repo)
3. [Prerequisites](#prerequisites)
4. [Setup & Installation](#setup--installation)
5. [Environment Variables](#environment-variables)
6. [How to Run an Example](#how-to-run-an-example)
7. [Suggested Learning Path](#suggested-learning-path)
8. [File-by-File Guide](#file-by-file-guide)
9. [Notes](#notes)

---

## What is DSPy?

[DSPy](https://dspy.ai/) is a framework for programming — rather than hand-writing — prompts for language models. Instead of crafting a giant prompt string, you declare **signatures** (typed input → output contracts) and compose them into **modules** (`Predict`, `ChainOfThought`, `ReAct`, custom `dspy.Module` subclasses). DSPy can then **optimize** those modules automatically against a metric and a small dataset, tuning the underlying prompts/few-shot examples for you.

This repo walks through that whole arc: from a single typed prediction, up to a pipeline that DSPy optimizes on its own.

---

## What's in This Repo

| Concept | File |
|---|---|
| Structured output extraction | `Structured_Output.py` |
| Chain-of-Thought reasoning | `chain_of_thought.py` |
| Retrieval-Augmented Generation (RAG) | `hr_rag_bot.py` |
| Tool-using agent (ReAct) | `react_expense_assistant.py` |
| Automatic optimization of a pipeline | `Self_improving_rag.py` |

All examples run against Google's Gemini models via DSPy's `dspy.LM("gemini/...")` interface (which routes through [LiteLLM](https://www.litellm.ai/) under the hood).

---

## Prerequisites

- Python 3.10+ (recommended)
- A **Gemini API key** (from Google AI Studio) — every example configures `dspy.LM("gemini/gemini-2.5-flash", ...)`, and the RAG examples additionally use `dspy.Embedder("gemini/gemini-embedding-001", ...)` for embeddings
- No local GPU or vector database required — the RAG examples use DSPy's built-in in-memory `dspy.retrievers.Embeddings` retriever over a small in-code corpus

---

## Setup & Installation

```bash
# 1. Clone the repository
git clone <this-repo-url>
cd <repo-folder>

# 2. Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate       # Windows
# source venv/bin/activate  # macOS/Linux

# 3. Install dependencies
pip install -r requirements.txt
```

---

## Environment Variables

Each script calls `load_dotenv(override=True)`, so create a `.env` file in the repo root:

```env
GEMINI_API_KEY=your_gemini_api_key_here
```

This key is picked up by LiteLLM (used internally by `dspy.LM`) whenever a script configures a `gemini/...` model or embedder.

**Never commit your `.env` file** — it holds your personal API key.

---

## How to Run an Example

Every file has a `main()` guarded by `if __name__ == "__main__":`, so you can just run it directly:

```bash
python Structured_Output.py
python chain_of_thought.py
python hr_rag_bot.py
python react_expense_assistant.py
python Self_improving_rag.py
```

Each script prints:
- the model's prediction / final output, and
- `dspy.inspect_history()` — DSPy's log of the actual prompt(s) sent to the LM, which is the best way to see exactly what DSPy generated under the hood for a given signature/module.

> `Self_improving_rag.py` takes noticeably longer to run than the others, since it compiles (optimizes) a module with `dspy.MIPROv2` across a small train/validation set before re-evaluating it.

---

## Suggested Learning Path

Work through the files in this order — each one builds on a DSPy concept introduced before it:

1. **`Structured_Output.py`** — Start here. The simplest possible DSPy program: a `dspy.Signature` with typed fields (including a `Literal` field) run through `dspy.Predict`, turning a messy support email into a structured ticket (subject, priority, product, sentiment).
2. **`chain_of_thought.py`** — Same idea, but swaps `dspy.Predict` for `dspy.ChainOfThought`, which asks the model to reason step-by-step before producing its output — here, deciding a loan-risk category for an applicant profile.
3. **`hr_rag_bot.py`** — Introduces retrieval: a small in-code "HR handbook" corpus is embedded and searched with `dspy.retrievers.Embeddings`, and a custom `dspy.Module` (`RAG`) combines the retrieved context with `dspy.ChainOfThought` to answer questions grounded in that context.
4. **`react_expense_assistant.py`** — Introduces tool use: `dspy.ReAct` is given two plain Python functions wrapped as `dspy.Tool`s (a currency-exchange lookup and a safe calculator) and reasons/acts/observes in a loop to answer a multi-step question.
5. **`Self_improving_rag.py`** — Ties it together and goes further: rebuilds the HR RAG bot as a `MiniHR` module, defines a small labeled eval set and an exact-match metric, evaluates a **baseline** version, then uses `dspy.MIPROv2` to **automatically optimize** the module's prompts, and re-evaluates to show the before/after accuracy.

---

## File-by-File Guide

### `Structured_Output.py`
Extracts structured tickets from raw support emails. Defines a `SupportEmail` signature with `subject`, `priority` (`Literal["low", "medium", "high"]`), `product`, and `negative_sentiment` output fields, run through `dspy.Predict`. Includes three sample emails as a quick demo.

### `chain_of_thought.py`
A financial-risk checker for loan applications. Defines a `LoanRisk` signature (`loan_risk`, `approved`) and demonstrates `dspy.ChainOfThought`, which produces intermediate reasoning before its final structured answer — useful for seeing *why* the model reached a decision, not just what it decided.

### `hr_rag_bot.py`
RAG over a small, hard-coded employee-handbook corpus. Builds embeddings with `dspy.Embedder("gemini/gemini-embedding-001")`, wires them into a `dspy.retrievers.Embeddings` retriever (top-`k=3`), and defines a `RAG` module that retrieves relevant passages and answers via `dspy.ChainOfThought`.

### `react_expense_assistant.py`
A tool-using expense assistant built with `dspy.ReAct`. Two tools are defined as plain functions and wrapped with `dspy.Tool`:
- `get_exchange_rate` — a hard-coded USD conversion table (`USD`, `EUR`, `GBP`)
- `calculate` — a regex-guarded safe `eval()` for arithmetic expressions

The agent is asked to convert a EUR expense to USD and check it against a per-person spending limit — a task that requires chaining both tools together.

### `Self_improving_rag.py`
Takes the same HR-handbook RAG idea from `hr_rag_bot.py` and shows DSPy's optimization loop:
- Rebuilds the RAG pipeline as a `MiniHR` module with an `HRAnswer` signature.
- Defines a tiny labeled `devset` (4 question/answer pairs) and an `exact_match` metric.
- Runs and prints a **Baseline** accuracy report.
- Compiles an **Optimized** version of the bot using `dspy.MIPROv2(metric=exact_match, auto="light")`.
- Re-evaluates and prints the **Optimized** accuracy report for comparison, plus the full DSPy prompt history.

### `requirements.txt`
Pinned dependencies for the whole repo — notably `dspy`, `litellm`, `openai`, `optuna`/`gepa` (used internally by DSPy's optimizers), and standard supporting libraries (`pydantic`, `requests`, `python-dotenv`, etc.).

---

## Notes

- All scripts configure `temperature=1` and a large `max_tokens=128000` on the LM — feel free to tune these per example as you experiment.
- The docstring in `hr_rag_bot.py` mentions "free Hugging Face embeddings," but the code itself uses `dspy.Embedder("gemini/gemini-embedding-001", ...)` — so a valid `GEMINI_API_KEY` is required for that script too, the same as every other example here.
- `react_expense_assistant.py`'s `calculate` tool includes a `yolo` flag that bypasses its regex safety check and calls raw `eval()` — it's there to illustrate the difference between a guarded and unguarded tool, not as something to enable casually.
- `dspy.inspect_history()` is your best friend while learning DSPy — run it after any prediction to see exactly what prompt DSPy actually sent to the model, which is invaluable for understanding how signatures and modules translate into real prompts.

---

This README is intended to help you navigate a small but complete tour of DSPy's core building blocks — signatures, modules, retrieval, tool use, and optimization — one file at a time.
