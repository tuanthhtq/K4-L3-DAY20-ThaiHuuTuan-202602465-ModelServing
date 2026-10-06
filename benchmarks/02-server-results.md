# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 70 | 1.19 | 6900 | 9500 | 12000 | 8.5 | 0.0% |
| 50 | 65 | 1.10 | 32000 | 46000 | 48000 | 32.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users Locust simulated. It includes queued
requests, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation, use the server gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.93x** (19% of linear) |
| P95 latency | **4.84x** |
| Effective concurrency at 50 users | 32.6 vs `--parallel 4` slots (occupancy/slot ratio 8.14) |

**Saturated.** Offered load increased 5x, but delivered throughput fell to 0.93x, while
effective concurrency reached 32.6 and metrics recorded
`n_busy_slots_per_decode = 3.98/4`. Effective concurrency was already 8.5 against four
slots at 10 users, so these two runs only support the conclusion that saturation occurs
at or below 10 users. Additional load mostly became queue time instead of throughput.

Throughput moved 0.93x while P95 moved 4.84x. This gap shows that the added latency is
mainly queue time rather than compute time: the slots were almost fully occupied while
effective concurrency greatly exceeded four. Past saturation, much more latency bought
almost no throughput. I chose a P95 SLO of at most 10 seconds: the 10-user run met it at
9.5 seconds, while the 50-user run missed it at 46 seconds. To raise goodput@SLO, I
would test a higher `--parallel` value first, then repeat the same load test to determine
whether queueing falls without causing an earlier throughput plateau.

## Your reading

The server saturates at or below 10 users; the two tested load levels are insufficient
to locate the exact onset. The clearest evidence is that a 5x load increase reduced RPS
from 1.19 to 1.10 (0.93x), raised P95 from 9,500 to 46,000 ms (4.84x), and drove
`n_busy_slots_per_decode` to 3.98/4. By Little's Law, effective concurrency of 32.6 is
far above four slots, so the added latency is queue time. With a P95 SLO of at most 10
seconds, 10 users pass and 50 users fail. I would test a higher `--parallel` value first
because all four slots were busy; increasing threads is not the first choice because CP2
showed nearly unchanged throughput from 1 to 32 threads.
