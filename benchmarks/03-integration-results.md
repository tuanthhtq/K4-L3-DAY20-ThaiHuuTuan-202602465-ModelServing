# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3444.0 | 3444.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3089.7 | 3089.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3086.0 | 3086.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **3206.6** · total **3206.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC:** stubbed; this run used the local machine and no deployed cloud infrastructure.
- **N17 Data pipeline:** stubbed; the documents are an in-memory toy corpus.
- **N18 Lakehouse:** stubbed; no lakehouse or persistent analytical store is connected.
- **N19 Vector + features:** stubbed; retrieval uses keyword overlap instead of real embeddings and a vector index.

The LLM being the dominant stage matches my expectation. Embedding was disabled and
keyword retrieval took less than 0.1 ms, while generation averaged 3206.6 ms and
accounted for effectively 100% of total latency. To halve pipeline latency, I would
target the LLM stage first by reducing generated tokens and testing the faster Q2
quantization. Optimising retrieval would have no measurable effect in this baseline.
