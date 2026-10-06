# Reflection - Day 20 Lab (Báo cáo cá nhân)

**Họ tên:** Thái Hữu Tuán  
**MSSV:** 202602465  
**Cohort:** A20-K4  
**Ngày nộp:** 2026-10-06

---

## 1. Phần cứng và runtime

- **OS:** Windows 11
- **CPU:** AMD Ryzen 7 H 255 with Radeon 780M Graphics
- **Core:** 8 nhân / 16 luồng
- **CPU extensions:** AVX2 / AVX-512 (Zen 4)
- **RAM:** 30.8 GB
- **Accelerator:** Vulkan; base track dùng `ngl=99`
- **llama.cpp asset:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** `UD-Q4_K_XL` + `UD-Q2_K_XL`

**Môi trường:** Laptop Windows chạy local.

**Setup:** Script probe chọn Gemma 4 E2B vì máy có đủ RAM. Setup tải runtime
Vulkan b10488 và hai file GGUF. PowerShell launcher gặp lỗi đường dẫn có khoảng trắng khi
khởi động `llama-server`, nên chạy trực tiếp cùng binary với các flag đã sinh cho các
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

**B1 - Build llama.cpp từ source**

```text
before: 23.0 tok/s (prebuilt release)
after:  27.4 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.19x
```

Bản source build nhanh hơn 19%. Hai bản dùng cùng llama.cpp b10488, model, 8 thread
và `ngl=0`; khác biệt chính là source build được tối ưu cho CPU hiện tại. Kết quả này
cho thấy tối ưu instruction set cải thiện decode, dù workload vẫn bị giới hạn đáng kể
bởi memory bandwidth.

**B2 - GPU layer-offload sweep**

```text
before: 25.1 tok/s (-ngl 0, CPU-only)
after:  36.5 tok/s (-ngl 99, full Vulkan offload)
speedup: 1.45x
```

Full offload cho kết quả tốt nhất. Các mức `-ngl 8` đến `-ngl 24` chậm hơn CPU-only
do lợi ích xử lý trên GPU chưa bù được chi phí chia workload và truyền dữ liệu. Từ
`-ngl 32`, throughput mới vượt CPU-only. Model vừa trong bộ nhớ GPU dùng chung nên
không thấy dấu hiệu thiếu bộ nhớ khi tăng tới `-ngl 99`.

**B4/C5 - Model nhỏ nhất vẫn hữu ích**

So sánh Q4 và Q2 bằng cùng năm prompt độc lập. Cả hai trả đúng phép tính, JSON và
hàm Python; cả hai đều yếu ở Goodput@SLO. Q4 còn mở rộng sai TTFT thành `Time to First
Touch`, trong khi Q2 mô tả gần đúng ý nhưng không nêu đủ `Time to First Token`. Vì Q2
giảm kích thước từ 2.97 xuống 2.24 GB, tăng decode từ 35.9 lên 40.3 tok/s và không làm
giảm số prompt đạt yêu cầu, tôi chọn Q2 cho workload này. Failure Goodput@SLO xuất hiện
ở Q2 nhưng cũng có ở Q4, nên chưa thể quy nguyên nhân cho quantization. Chi tiết nằm
trong `bonus/c5-smallest-useful-model.md`.

**B5/C9 - Embedding serving**

Embedding endpoint trả vector 1536 chiều và xếp đúng tài liệu liên quan nhất với
cosine similarity 0.846. Khi tăng batch từ 1 lên 16, latency của cả batch tăng từ
2191.8 lên 2977.7 ms, chỉ 1.36x, trong khi throughput tăng từ 0.5 lên 5.4 texts/s,
tương đương 10.8x. Kết quả phù hợp với cơ chế prefill-bound: embedding chỉ cần một
forward pass, không có decode loop và KV cache như chat serving. Static batching vì
thế tăng throughput rõ rệt. Giới hạn của thí nghiệm là dùng chat GGUF ở pooling mode;
production cần embedding model chuyên dụng. Chi tiết nằm trong
`bonus/c9-embedding-serving.md`.

## 7. Kết quả bất ngờ nhất

Tải đưa vào tăng 5x nhưng throughput gần như không tăng, còn P95 tăng gần 5 lần; queueing
hiện rõ qua Little's Law.

## 8. Tự kiểm tra

- [x] Đã tạo artifact base track
- [x] Đã chụp đủ 5 screenshot bắt buộc
- [x] `verify` pass sau commit cuối

## 9. Khai báo sử dụng AI

AI được dùng để giải thích hướng dẫn lab, phân tích các lượt chạy. Toàn bộ benchmark và screenshot được tạo trên máy này.
