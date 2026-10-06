# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 37.5 | 99% |
| 4 | 37.8 | 100% |
| 8 | 37.9 | 100% |
| 16 | 37.5 | 99% |
| 32 | 37.6 | 99% |

**Best**: `-t 8` at 37.9 tok/s
**Slowest tested**: `-t 16` at 37.5 tok/s (1.01x spread)
**Against the physical-core default** (`-t 8`, 37.9 tok/s): 1.00x

Use this in your run:

```powershell
$env:LAB_N_THREADS = '8'
.\lab.ps1 bench
```

## Your explanation

The peak is at `-t 8`, matching the physical-core count, but the curve is nearly flat:
37.5, 37.8, 37.9, 37.5 and 37.6 tok/s for 1, 4, 8, 16 and 32 threads. There is
therefore no sharp knee; the spread between the best and slowest configurations is only
about 1.01x.

From 16 threads onward, adding threads does not increase throughput because decode is
memory-bandwidth-bound. Threads beyond the physical-core count share the same memory
resources and add scheduling overhead, so throughput drops slightly. On this machine,
`-t 8` is the sensible choice: it is the best result and avoids oversubscription, even
though the gain over 4 threads is only about 0.3%.
