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
| **`RAG_SEC_Agent_PRACTICE.ipynb`** | **Phase 3** — agentic RAG. A hand-rolled ReAct loop over Claude tool-use: search → self-critique → re-query → cite/abstain, with agentic eval. CPU-only. |
| **`RAG_SEC_Agent.ipynb`** | Phase 3 answer key. |
| **`RAG_SEC_Causal_PRACTICE.ipynb`** | **Phase 4** — causal eval. Treats the agent as an experimental subject: a crossover measuring the ITT and CACE of its *own* re-query decision. CPU-only. |
| **`RAG_SEC_Causal.ipynb`** | Phase 4 answer key. |
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

## Phase 3 — agentic RAG

Phases 1–2 are a *chain*: a chain can't recover from a bad first retrieval. Phase 3 makes it an
**agent** — a hand-rolled **ReAct loop** over **Claude's tool-use API** that plans, searches,
judges whether the context is sufficient, **re-queries** if not, and answers with `[source-id]`
citations or **abstains** when the filings don't contain the answer.

- **Brain:** Claude (`claude-opus-4-8`) via tool-use; **retrieval stays local**. No local LLM →
  **no GPU, no OOM** — a CPU Colab runtime is enough.
- **Guardrail:** every cited id is verified against what was actually retrieved (catches fabricated
  citations).
- **Agentic eval:** grades *behavior* — searches per question, citation grounding, and abstention
  on a deliberately out-of-corpus question — not just the final answer.

You build the tool schema, the ReAct loop (the centerpiece), the citation check, and the agentic
eval. Needs only the `ANTHROPIC_API_KEY` secret.

## Phase 4 — causal evaluation (the agent as an experimental subject)

Input evaluation scores a component on fixed inputs; **model/behavior evaluation** scores the
agent's *own decisions*. Phase 4 measures the causal effect of the agent's choice to **re-query**:

- **Crossover design** — every question runs through both arms (control = capped at one retrieval;
  treatment = free to re-query), so the counterfactual is observed per question.
- **Two estimands** — **ITT** (effect of *offering* self-correction, over all questions) and
  **CACE** (effect among the **compliers** — questions that actually re-queried). `ITT = P(re-query) × CACE`.
  Because the subject is a model, the crossover recovers CACE directly (no IV needed).
- **Correct inference** — repeated runs cut outcome noise but are clustered within a question, so
  the unit of inference is the **question**: a sign-flip permutation test for ITT and a cluster
  bootstrap for the intervals (avoids pseudoreplication).

Outcome is a **blind** completeness judge. You build the capped agent loop, the experiment, the
estimands, and the question-level inference. CPU runtime; `ANTHROPIC_API_KEY` only.
