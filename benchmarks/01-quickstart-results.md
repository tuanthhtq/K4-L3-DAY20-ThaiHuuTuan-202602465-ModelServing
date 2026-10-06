# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5677 | 194 / 623 | 27.8 / 28.2 | 1945 / 2376 / 2376 | 35.9 |
| UD-Q2_K_XL | 2.24 | 3587 | 188 / 759 | 24.8 / 25.1 | 1754 / 2302 / 2302 | 40.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG increases it sharply.
- **TPOT** = per-output-token decode cost, mainly bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- On this machine, `UD-Q2_K_XL` decodes **1.12x faster** than `UD-Q4_K_XL` and uses 0.73 GB less disk space.

## Your observation

On this machine, UD-Q2_K_XL is 0.73 GB smaller (2.24 vs 2.97 GB), loads 36.8% faster
(3587 vs 5677 ms), and decodes 12.3% faster (40.3 vs 35.9 tok/s). Q2 TTFT P50 is
3.1% lower (188 vs 194 ms), but TTFT P95 is higher (759 vs 623 ms). E2E P50 is
9.8% lower (1754 vs 1945 ms), while E2E P95 is 3.1% lower (2302 vs 2376 ms).

I asked both quantizations the same question: "Define goodput@SLO in one sentence."
Both answers were correct. Q4 was more concise at 24 output tokens; Q2 used 31 tokens
but remained accurate. For this workload, Q2 is worth using when memory footprint and
decode speed matter. Q4 is preferable when preserving model quality matters more,
although this one-prompt check did not show a clear quality loss from Q2.
