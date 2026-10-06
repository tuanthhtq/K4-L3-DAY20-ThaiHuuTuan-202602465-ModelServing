# 01 - Tinh chỉnh số lượng thread

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · máy `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 core vật lý · 16 core logic** · `ngl=99` · metric `tg128`

| Số thread (-t) | tg128 (tok/s) | So với tốt nhất |
|:--|--:|--:|
| 1 | 37.5 | 99% |
| 4 | 37.8 | 100% |
| 8 | 37.9 | 100% |
| 16 | 37.5 | 99% |
| 32 | 37.6 | 99% |

**Tốt nhất**: `-t 8` đạt 37.9 tok/s
**Chậm nhất trong các cấu hình đã thử**: `-t 16` đạt 37.5 tok/s (chênh lệch 1.01 lần)
**So với cấu hình mặc định theo số core vật lý** (`-t 8`, 37.9 tok/s): 1.00 lần

Dùng cấu hình sau cho lần chạy:

```powershell
$env:LAB_N_THREADS = '8'
.\lab.ps1 bench
```

## Giải thích

Đỉnh đo được ở `-t 8`, đúng bằng số core vật lý, nhưng đường cong gần như phẳng:
37.5, 37.8, 37.9, 37.5 và 37.6 tok/s cho 1, 4, 8, 16 và 32 thread. Vì vậy
không có một knee rõ ràng; chênh lệch giữa cấu hình tốt nhất và chậm nhất chỉ khoảng
1.01 lần.

Từ 16 thread trở lên, thêm thread không làm tăng throughput vì decode bị giới hạn bởi
băng thông bộ nhớ. Các thread vượt số core vật lý phải chia sẻ cùng tài nguyên bộ nhớ
và tạo thêm chi phí lập lịch, nên kết quả giảm nhẹ. Trên máy này, `-t 8` là lựa chọn
hợp lý: đạt kết quả cao nhất và tránh oversubscription, dù lợi ích so với 4 thread
chỉ khoảng 0.3%.
