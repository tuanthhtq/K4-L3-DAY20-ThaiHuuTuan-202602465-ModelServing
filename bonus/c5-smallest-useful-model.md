# C5 - Model nhỏ nhất vẫn hữu ích

## Thiết lập

- Model: Gemma 4 E2B, llama.cpp b10488, Vulkan full offload (`ngl=99`).
- So sánh: `UD-Q4_K_XL` 2.97 GB và `UD-Q2_K_XL` 2.24 GB.
- Cùng năm prompt, mỗi prompt chạy trong một chat mới để lịch sử không ảnh hưởng kết quả.

## Kết quả

| Prompt | Q4 | Q2 | Nhận xét |
|:--|:--|:--|:--|
| `17 * 23` | Pass | Pass | Cả hai trả đúng `391`. |
| JSON `name`, `age` | Pass | Pass | Cả hai trả đúng cấu trúc và giá trị. |
| Giải thích TTFT | Fail | Partial | Q4 nhầm thành `Time to First Touch`; Q2 mô tả đúng ý thời gian tới phản hồi đầu tiên nhưng không mở rộng đúng `Time to First Token`. |
| Hàm Python kiểm tra số chẵn | Pass | Pass | Cả hai sinh hàm modulo hợp lệ. |
| Goodput@SLO | Partial | Partial | Cả hai nhận ra `SLO` và ý nghĩa chung của goodput nhưng không nêu đúng định nghĩa: lượng request hữu ích đáp ứng SLO. |

Số đo từ benchmark base:

| Quantization | Size | Decode |
|:--|--:|--:|
| UD-Q4_K_XL | 2.97 GB | 35.9 tok/s |
| UD-Q2_K_XL | 2.24 GB | 40.3 tok/s |

## Kết luận

Trong hai mức đã thử, `UD-Q2_K_XL` là lựa chọn nhỏ nhất vẫn hữu ích. Nó giảm 0.73 GB
và tăng decode khoảng 12.3% nhưng không làm giảm số prompt đạt yêu cầu so với Q4.
Failure rõ nhất ở Q2 là câu Goodput@SLO; tuy nhiên Q4 cũng thất bại tương tự, nên dữ
liệu này không chứng minh quantization là nguyên nhân. Với workload ngắn hiện tại, tôi
sẽ triển khai Q2 để ưu tiên bộ nhớ và tốc độ. Muốn xác định chính xác điểm chất lượng
bắt đầu gãy cần thử thêm mức thấp hơn như `UD-IQ2_M` trên cùng bộ prompt.
