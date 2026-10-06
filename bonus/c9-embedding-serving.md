# C9 - Embedding serving

## Thiết lập

- Endpoint: llama-server `/v1/embeddings` trên cổng 8081.
- Model: Gemma 4 E2B chat GGUF ở pooling mode, vector 1536 chiều.
- Workload: cùng tập văn bản, tăng batch từ 1 lên 16.

## Retrieval

Với query hỏi embedding serving có dùng KV cache và decode loop như chat hay không,
kết quả đúng được xếp hạng đầu với cosine similarity `0.846`:

> Embedding serving is prefill-bound: one forward pass, no KV cache, no decode loop.

## Throughput sweep

| Batch | Latency (ms) | Throughput (texts/s) |
|--:|--:|--:|
| 1 | 2191.8 | 0.5 |
| 2 | 2208.3 | 0.9 |
| 4 | 2256.8 | 1.8 |
| 8 | 2544.4 | 3.1 |
| 16 | 2977.7 | 5.4 |

## Phân tích

Tăng batch từ 1 lên 16 làm latency của cả batch chỉ tăng khoảng 1.36x, trong khi
throughput tăng từ 0.5 lên 5.4 texts/s, tương đương 10.8x. Embedding chỉ cần một
forward pass cho mỗi văn bản, không có vòng decode tự hồi quy và không cần KV cache.
Vì vậy static batching tận dụng phần cứng tốt hơn, khác với chat serving nơi continuous
batching phải lập lịch nhiều chuỗi decode có độ dài khác nhau.

Thí nghiệm dùng lại chat model ở pooling mode để không tải thêm weights. Kết quả đủ
minh họa serving regime nhưng không đại diện cho chất lượng retrieval production;
triển khai thật nên dùng embedding model chuyên dụng như Qwen3-Embedding hoặc BGE-M3.
