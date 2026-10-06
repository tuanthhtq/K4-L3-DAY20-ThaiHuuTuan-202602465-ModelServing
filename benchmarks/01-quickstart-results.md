# 01 - Đo độ trễ cơ sở

Model `Gemma 4 E2B` · máy `Windows-AMD64` · llama.cpp `b10488`
Cấu hình: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · không tính lần chạy khởi động
Số request hoàn thành: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Dung lượng (GB) | Thời gian tải (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5677 | 194 / 623 | 27.8 / 28.2 | 1945 / 2376 / 2376 | 35.9 |
| UD-Q2_K_XL | 2.24 | 3587 | 188 / 759 | 24.8 / 25.1 | 1754 / 2302 / 2302 | 40.3 |

- **TTFT** là thời gian prefill. Prompt ngắn giữ chỉ số này thấp; RAG với context dài sẽ làm TTFT tăng mạnh.
- **TPOT** là chi phí decode cho mỗi output token, chủ yếu bị giới hạn bởi băng thông bộ nhớ. `decode tok/s = 1000 / TPOT_p50`.
- Trên máy này, `UD-Q2_K_XL` decode **nhanh hơn 1.12 lần** so với `UD-Q4_K_XL` và chiếm ít hơn 0.73 GB ổ đĩa.

## Nhận xét

Trên máy này, UD-Q2_K_XL nhỏ hơn 0.73 GB (2.24 so với 2.97 GB), tải nhanh hơn
36.8% (3587 so với 5677 ms) và decode nhanh hơn 12.3% (40.3 so với 35.9 tok/s).
TTFT P50 của Q2 thấp hơn 3.1% (188 so với 194 ms), nhưng TTFT P95 lại cao hơn
(759 so với 623 ms). E2E P50 thấp hơn 9.8% (1754 so với 1945 ms), còn E2E P95
thấp hơn 3.1% (2302 so với 2376 ms).

Tôi đã hỏi cả hai quantization cùng một câu: "Define goodput@SLO in one sentence."
Cả hai đều trả lời đúng. Q4 trả lời ngắn gọn hơn với 24 output token; Q2 dùng 31 token
nhưng nội dung vẫn chính xác. Với workload này, Q2 đáng dùng khi cần tiết kiệm bộ nhớ
và tăng tốc decode. Q4 phù hợp hơn khi ưu tiên giữ chất lượng model, dù phép thử một
prompt này chưa cho thấy Q2 bị giảm chất lượng rõ ràng.
