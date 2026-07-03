# RAG Prep — a full-pipeline RAG exercise

Hands-on practice for building **and being able to discuss** a Retrieval-Augmented Generation
pipeline end to end. A companion to [`edgar_nlp`](../edgar_nlp) (model-adaptation techniques);
this repo is about the **LLM application pipeline**, not modeling.

The exercise is scoped to a **1–2 day** build. The model just has to work well enough — the
goal is fluency across every stage of the pipeline so you can whiteboard and defend it:

```
01 Ingestion  →  02 Embedding  →  03 Vector store  →  04 Retrieval  →  05 LLM
```

## The notebooks

Same teaching pattern as `edgar_nlp` — a **practice** notebook with the core code blanked out,
and an **answer key** to check yourself against *after* attempting.

| Notebook | What it is |
|----------|------------|
| **`RAG_SEC_Pipeline_PRACTICE.ipynb`** | **Phase 1** — the pipeline. The RAG core at each stage is a `### YOUR CODE HERE` blank; SEC-fetch and model-load boilerplate are provided. |
| **`RAG_SEC_Pipeline.ipynb`** | Phase 1 answer key — full working code. |
| **`RAG_SEC_Eval_PRACTICE.ipynb`** | **Phase 2** — evaluation & experimentation. Gold set, retrieval metrics, a Claude LLM-judge, and one controlled experiment. |
| **`RAG_SEC_Eval.ipynb`** | Phase 2 answer key. |
| `Module_09_..._Langchain_Mistral...ipynb` | The course RAG notebook Phase 1 is built from (arXiv corpus, happy-path). |

Each stage follows: **Why** → **write the blank** → **instrument it** (measure the effect) →
a **🎤 Talk-track** note framing how you'd discuss the decision in an interview.

## What it covers

Every node of the pipeline, on real **SEC 10-K filings** (AAPL / MSFT / NVDA):

- **01 Ingestion** — clean filing HTML, recursive chunking, attach metadata.
- **02 Embedding** — `all-mpnet-base-v2`, with a similarity sanity check.
- **03 Vector store** — in-memory Qdrant + a **metadata filter** (scope a query to one company).
- **04 Retrieval** — top-k over-fetch → **cross-encoder rerank**.
- **05 LLM** — Mistral-7B (4-bit) with a ground-only, cite-the-source, decline-when-unknown prompt.

## Running it

- **Platform:** Google Colab, **T4 GPU** (Runtime → Change runtime type → T4 GPU).
- **Gated model:** Stage 05 uses `mistralai/Mistral-7B-Instruct-v0.3` — accept its license on
  Hugging Face and add your token to Colab **Secrets** as `HF_TOKEN`.
- **SEC:** set `SEC_USER_AGENT` to your name/email (SEC requires a real contact).
- **Workflow:** run top-to-bottom **once** (re-running the model-load cell can OOM the T4);
  fill the blanks, then diff against the answer key.

## Phase 2 — the evaluation / experimentation layer

Phase 1 stops at a *working* pipeline; Phase 2 makes it a *measured* one — the part of the diagram
most candidates skip:

- **Gold set** — synthetic Q/A generated from known chunks (ground truth without hand-labeling).
- **Retrieval metrics** — recall@k / MRR (runs first, no GPU — survives a T4 OOM).
- **Answer eval** — faithfulness & relevance scored by an **LLM-judge (Claude `claude-opus-4-8`)**,
  with structured output. Local Mistral generates; Claude judges (hybrid).
- **One experiment** — rerank on vs. off, held-out on the gold set, reported as an effect size with
  a bootstrap 95% CI.

Same runtime requirements as Phase 1, plus an `ANTHROPIC_API_KEY` Colab secret for the judge.
Set `USE_LOCAL_LLM = False` in the generator cell to fall back to Claude generation if the T4 OOMs.
