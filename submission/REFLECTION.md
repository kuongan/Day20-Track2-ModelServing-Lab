# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Trần Khương An
**Cohort:** _<A20-K1 >_
**Ngày submit:** _<2026-05-06>_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 11
- **CPU:** Intel(R) Core(TM) i7-10750H @ 2.60GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, FMA, F16C
- **RAM:** 16 GB
- **Accelerator:** NVIDIA RTX 3060 Laptop GPU (6GB)
- **llama.cpp backend đã chọn:** CPU (planned CUDA but chưa cài Toolkit)
- **Recommended model tier:** Qwen2.5-1.5B (Q4_K_M)

**Setup story (≤80 chữ):**
Ban đầu build lỗi do thiếu `cmake` và CUDA Toolkit. Sau khi cài CMake thì build CPU thành công, nhưng CUDA vẫn chưa hoạt động vì thiếu `nvcc`. Do đó mình fallback sang CPU-only để hoàn thành benchmark trước, sau đó tối ưu bằng thread tuning.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

Settings: `n_threads=6`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 2964 | 386 / 469 | 84.1 / 152.3 | 5622 / 6980 / 7186 | 11.9 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 731 | 526 / 675 | 72.7 / 77.2 | 5154 / 5346 / 5398 | 13.8 |

## Observations

Q2_K có decode rate cao hơn (~13.8 vs 11.9 tok/s) và E2E latency thấp hơn, nhưng TTFT lại cao hơn do load nhanh nhưng kém ổn định ở bước đầu. Q4_K_M chậm hơn ~15% nhưng cho chất lượng output tốt hơn, nên là lựa chọn cân bằng hơn cho inference.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 1.44 | 5100 | 7800 | 9600 | 0 |
| 50 | 1.53 | 9400 | 23000 | 25000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): Dựa vào log metrics, peak `llamacpp:n_busy_slots_per_decode` duy trì ở mức **3.66596**, và `llamacpp:requests_deferred` là **0.0**. Điều này có nghĩa là ở concurrency 50, mặc dù hệ thống bị nghẽn cổ chai về năng lực tính toán khiến độ trễ (E2E P95/P99) tăng vọt gấp 3 lần và RPS không tăng đáng kể, nhưng không có request nào bị đẩy vào hàng đợi chờ cấp phát (deferred = 0). Lượng KV-cache và số slots cấu hình vẫn đủ để tiếp nhận các request này cùng lúc mà không gây ra lỗi (0 failures).
---

## 4. Track 03 — Milestone integration

* **N16 (Cloud/IaC):** stub — chạy local
* **N17 (Data pipeline):** stub — in-memory dict
* **N18 (Lakehouse):** stub — SQLite
* **N19 (Vector + Feature Store):** stub — TOY_DOCS

**Latency breakdown (thực tế từ log):**

* retrieve: **~0.0 - 0.1 ms**
* llama-server (llm): **~2916 - 6532 ms** (biến động tùy theo câu hỏi và độ dài output sinh ra)
* total: **~2916 - 6532 ms**

**Reflection:**
Bottleneck rõ ràng và tuyệt đối nằm ở llama-server (inference), chiếm gần như 100% tổng thời gian xử lý của cả pipeline. Quá trình retrieve diễn ra ngay lập tức (gần 0 ms) do đang sử dụng dữ liệu giả lập (stub/in-memory). Điều này đúng với kỳ vọng vì LLM text-generation luôn chiếm phần lớn độ trễ so với các bước xử lý dữ liệu và truy xuất ban đầu.

---

## 5. Bonus — The single change that mattered most

**Change:**
Tune số lượng threads (`-t`) để match với khả năng xử lý thực tế của CPU (chọn giá trị tối ưu thay vì mặc định hoặc oversubscribe)

---

**Before vs after:**

```
before:
-t 1  → 6.49 tok/s
-t 6  → 16.07 tok/s

after:
-t 12 → 17.21 tok/s  (best)

speedup: ~2.65× (1 → 12 threads)
```

---

**Tại sao nó work:**

LLM decode chủ yếu bị giới hạn bởi **memory bandwidth**, không phải compute thuần. Khi tăng số thread từ 1 lên gần số core (6–12), CPU tận dụng tốt hơn các pipeline xử lý và cache, giúp tăng throughput đáng kể (~2.6×).

Tuy nhiên, khi vượt quá mức tối ưu (ví dụ `-t 24`), hiệu năng giảm (17.21 → 15.88 tok/s) do **oversubscription**: các thread logic tranh chấp cùng memory channel và cache (L2/L3), gây cache miss và context switching overhead. Điều này cho thấy điểm tối ưu không phải là max threads mà là mức cân bằng giữa compute và memory bandwidth.

