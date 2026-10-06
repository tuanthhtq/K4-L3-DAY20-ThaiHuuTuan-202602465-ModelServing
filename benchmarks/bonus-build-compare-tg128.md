# Bonus B1 - Prebuilt vs source build

Host `Windows-AMD64` � CPU `AMD Ryzen 7 H 255 w/ Radeon 780M Graphics`
Vector extensions detected: none
llama.cpp `b10488` both sides � `threads=8` �
**both pinned to `ngl=0`** so this isolates the compiler �
metric `tg128`, 3 repetitions

> **Backend mismatch, handled.** The prebuilt binary sees
> `['Vulkan0: AMD Radeon 780M Graphics (16792 MiB, 15952 MiB free)']` and your source build sees `(no devices)`.
> Left at `-ngl 99` this comparison would have measured the accelerator and printed
> it under a compiler headline, so both sides were pinned to `-ngl 0`.

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 23.0 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 27.4 | 1.19x |

On this machine, the source build is **1.19x faster**.

before: 23.0 tok/s (prebuilt release)
after:  27.4 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.19x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## Your explanation

Bản source build nhanh hơn prebuilt 1.19x, từ 23.0 lên 27.4 tok/s. Hai bản dùng cùng
model, revision, 8 thread và đều bị khóa ở `ngl=0`, nên khác biệt chủ yếu đến từ
`-DGGML_NATIVE=ON`. Compiler có thể tối ưu mã máy cho đúng CPU AMD hiện tại thay vì
giữ khả năng tương thích rộng như prebuilt. Decode vẫn phụ thuộc nhiều vào memory
bandwidth, nên mức tăng 19% là hợp lý thay vì tăng nhiều lần.
