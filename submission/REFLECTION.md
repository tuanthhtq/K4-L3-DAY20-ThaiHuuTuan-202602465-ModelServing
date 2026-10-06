# Reflection - Day 20 Lab (Báo cáo cá nhân)

**Họ tên:** Thai Huu Tuan  
**MSSV:** 202602465  
**Cohort:** A20-K4  
**Ngày nộp:** 2026-10-06

---

## 1. Phần cứng và runtime

- **OS:** Windows 11
- **CPU:** AMD Ryzen 7 H 255 with Radeon 780M Graphics
- **Core:** 8 vật lý / 16 logic
- **CPU extensions:** AVX2 / AVX-512 (Zen 4)
- **RAM:** 30.8 GB
- **Accelerator:** Có thiết bị Vulkan; base track dùng `ngl=99`
- **llama.cpp asset:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` + `UD-Q2_K_XL`

**Môi trường:** Laptop Windows chạy local.

**Câu chuyện setup:** Script probe chọn Gemma 4 E2B vì máy có đủ RAM. Setup tải runtime
Vulkan b10488 và hai file GGUF. PowerShell launcher gặp lỗi đường dẫn có khoảng trắng khi
khởi động `llama-server`, nên tôi chạy trực tiếp cùng binary với các flag đã sinh cho các
checkpoint serving, load và pipeline.

---

## 2. Đo lường

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5677 | 194 / 623 | 27.8 / 28.2 | 1945 / 2376 / 2376 | 35.9 |
| UD-Q2_K_XL | 2.24 | 3587 | 188 / 759 | 24.8 / 25.1 | 1754 / 2302 / 2302 | 40.3 |

**Nhận xét:** Q2 nhỏ hơn 0.73 GB, tải nhanh hơn 36.8% và decode nhanh hơn 12.3%. E2E
P50 thấp hơn 9.8%, dù TTFT P95 cao hơn. Cùng một câu hỏi goodput@SLO cho câu trả lời
đúng ở cả hai quantization; Q4 dùng 24 output token, Q2 dùng 31. Với workload này,
Q2 đáng dùng khi ưu tiên bộ nhớ và tốc độ decode.

---

## 3. Serving dưới tải

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Effective concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.19 | 6900 | 9500 | 12000 | 8.5 | 0.0% |
| 50 | 1.10 | 32000 | 46000 | 48000 | 32.6 | 0.0% |

- **Tải đưa vào tăng:** 5x; throughput thực tế thay đổi **0.93x**.
- **P95 tăng:** **4.84x**.
- **Effective concurrency ở 50 users:** 32.6 so với `--parallel=4` slots.
- **Peak `n_busy_slots_per_decode`:** 3.98 / 4 slots.

**Phân tích saturation:** Server bão hòa tại hoặc dưới 10 users. Tải tăng 5x làm RPS
giảm từ 1.19 xuống 1.10, trong khi P95 tăng từ 9.5s lên 46s. Gauge 3.98/4 và
effective concurrency 32.6 cho thấy toàn bộ slot decode bận, đồng thời request phải xếp
hàng. Latency tăng chủ yếu là queue time. Với SLO P95 là 10s, 10 users đạt còn 50 users
không đạt. Tôi sẽ thử tăng `--parallel` trước vì toàn bộ slot đang bận; CP2 cho thấy
tăng thread không cải thiện throughput đáng kể.

---

## 4. Tích hợp

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 | Cloud/IaC | Stub |
| N17 | Data pipeline | Stub |
| N18 | Lakehouse | Stub |
| N19 | Vector + features | Stub; retrieval bằng keyword overlap |
| N20 | Serving | Real; `llama-server` local |

**Tách latency** (trung bình 3 query):

- embed: **0.0 ms**
- retrieve: **0.0 ms**
- llm: **3206.6 ms**
- **Stage lớn nhất:** llm, khoảng **100%** tổng latency

**Nhận xét:** LLM là bottleneck đúng như dự đoán. Embedding bị tắt và keyword retrieval
dưới 0.1 ms, trong khi generation trung bình 3206.6 ms. Để giảm latency 2x, tôi sẽ
tập trung vào generation bằng cách giảm số output token và thử Q2; tối ưu retrieval stub
không tạo khác biệt đáng kể trong baseline này.

---

## 5. Thay đổi có tác động lớn nhất

**Thay đổi:** Đổi từ `UD-Q4_K_XL` sang `UD-Q2_K_XL`.

```text
before: 35.9 tok/s (UD-Q4_K_XL)
after:  40.3 tok/s (UD-Q2_K_XL)
speedup: 1.12x
```

Decode tự hồi quy phải đọc lại trọng số model cho mỗi output token, nên trên máy này nó
chủ yếu bị giới hạn bởi memory bandwidth. Q2 giảm kích thước model từ 2.97 GB xuống
2.24 GB, giảm lượng dữ liệu phải truyền qua bộ nhớ ở mỗi bước decode. Vì vậy throughput
decode tăng 12% dù vẫn có chi phí dequantization. Kiểm tra một prompt cho kết quả đúng ở
cả hai bản, nên đây là trade-off hữu ích nhất; thread tuning chỉ thay đổi throughput
khoảng 1%.

---

## 6. Bonus

Chưa thực hiện.

## 7. Kết quả bất ngờ nhất

Tải đưa vào tăng 5x nhưng throughput gần như không tăng, còn P95 tăng gần 5 lần; queueing
hiện rõ qua Little's Law.

## 8. Tự kiểm tra

- [x] Đã tạo artifact base track
- [x] Đã chụp đủ 5 screenshot bắt buộc
- [ ] `verify` pass sau commit cuối

## 9. Khai báo sử dụng AI

AI được dùng để giải thích hướng dẫn lab, kiểm tra report sinh ra và hỗ trợ diễn đạt nhận
xét từ số liệu. Toàn bộ benchmark và screenshot được tạo trên máy này.
