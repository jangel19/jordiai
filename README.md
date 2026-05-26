# jordiai

A personalized, zero-cost local coding assistant built for embedded ML development at owell, a startup in Ecuador.

## What it is

jordiai combines DeepSeek-Coder-V2 16B (MoE, 2.4B active parameters) with a RAG pipeline backed by PostgreSQL and pgvector. It runs entirely offline on an Apple M3 Pro, costs nothing per query, and learns your coding patterns over time through an accept/reject save flow.

Built as a self-hosted alternative to GitHub Copilot, Claude Code, and OpenAI Codex for engineers working in bandwidth-constrained or offline environments.

## Benchmarks

Evaluated on a custom C++17 benchmark covering algorithms, embedded systems, and machine learning — domains directly relevant to embedded ML development.

| Model | Pass Rate | Cost per Query | Offline |
|---|---|---|---|
| DeepSeek bare (16B MoE) | 60% | $0 | Yes |
| jordiai (RAG-augmented) | 80% | $0 | Yes |
| Claude Sonnet 4.6 | 100% | ~$0.01 | No |

RAG personalization improved pass rate by 20 percentage points over the bare model, closing roughly half the gap to Sonnet 4.6 at zero additional compute cost.

### Hardware performance (M3 Pro, 18GB)

| Metric | Value |
|---|---|
| Time to first token | 99ms |
| Generation rate | 67 tok/s |
| RAG retrieval latency | 194ms avg |
| 100k C++ forward passes | 2.75ms |

## Architecture

```
User query
    -> Context gathering (git diff, file)
    -> RAG retrieval (pgvector similarity search)
    -> Conditional prompt injection (your code only, >= 0.55 similarity)
    -> DeepSeek-Coder-V2 16B via Ollama
    -> Streamed response + reference panel
    -> Quality gate (compile check before save)
    -> Accept/reject save flow
    -> pgvector + Obsidian vault
```

**Stack**

| Layer | Technology |
|---|---|
| Model | DeepSeek-Coder-V2 16B MoE via Ollama |
| Embeddings | all-MiniLM-L6-v2, 384-dim |
| Vector DB | PostgreSQL 16 + pgvector |
| Knowledge store | 34 ML/embedded papers, 2,612 chunks |
| CLI | Python, Click, Rich |

## Key design decisions

**Selective RAG injection.** Paper chunks are never injected into the model prompt — only the developer's own validated solutions above 0.55 cosine similarity are injected. Paper chunks appear as a reference panel after generation. This prevents retrieved prose from degrading generation quality.

**Compilation-based quality gate.** Before any snippet is saved to the database, it is validated by syntax-checking Python via `py_compile` or C++ via `g++ -fsyntax-only`. One buggy snippet containing `int8_t::max` (invalid C++ syntax) caused the RAG-augmented model to reproduce the same compile error across three consecutive attempts before this gate was added.

**MoE efficiency.** DeepSeek-Coder-V2 16B activates only 2.4B parameters per forward pass, producing inference throughput comparable to a dense 7B model while retaining 16B-class knowledge capacity.

## Paper

The full technical writeup covering architecture, evaluation methodology, and benchmark results is available in [`jordiai.pdf`](./jordiai.pdf).

## Usage

```bash
mycode "implement INT8 quantization in C++"
mycode "write a FreeRTOS task with priority scheduling"
mycode --file src/inference.cpp "optimize this forward pass"
```
