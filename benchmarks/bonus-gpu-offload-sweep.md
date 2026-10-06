# Bonus - GPU offload sweep

Host `Windows-AMD64` � backend(s) `vulkan` �
llama.cpp `b10488` � `threads=8` � metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 25.1 | 1.00x | 69% |
| 8 | 21.6 | 0.86x | 59% |
| 16 | 21.2 | 0.84x | 58% |
| 24 | 23.0 | 0.92x | 63% |
| 32 | 28.7 | 1.14x | 79% |
| 99 | 36.5 | 1.45x | 100% |

Best: `-ngl 99` at 36.5 tok/s
-- 1.45x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload là cấu hình tốt nhất trên máy này. `-ngl 99` đạt 36.5 tok/s, nhanh hơn
1.45x so với CPU-only ở 25.1 tok/s. Partial offload từ 8 đến 24 layer còn chậm hơn
CPU-only vì phần tính toán được chia giữa CPU và GPU nhưng vẫn phải trả chi phí truyền
dữ liệu qua bộ nhớ dùng chung. Từ 32 layer, lợi ích GPU mới vượt chi phí đó. Model vừa
trong bộ nhớ của Radeon 780M nên đường cong tiếp tục tăng tới full offload, không cho
thấy VRAM là giới hạn trong thí nghiệm này.
