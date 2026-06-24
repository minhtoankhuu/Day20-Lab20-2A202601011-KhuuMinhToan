# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** _Khưu Minh Toàn_
**Cohort:** _A20-K2-2A202601011_
**Ngày submit:** _2026-06-25_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** _Windows 11 / Ubuntu 22.04 (WSL2)_
- **CPU:** _AMD Ryzen 9 5900HX_
- **Cores:** _16 physical / 16 logical_
- **CPU extensions:** _AVX2_
- **RAM:** _7.5 GB_
- **Accelerator:** _CPU only (NVIDIA RTX 3070 bị lỗi build C++)_
- **llama.cpp backend đã chọn:** _CPU_
- **Recommended model tier:** _TinyLlama-1.1B_

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn:
_Chạy trên WSL2 Ubuntu. Gặp lỗi conflict C++ compiler (GCC 11 vs NVCC) khi build bản CUDA, nên tôi đã fallback sang bản cài đặt CPU-only (`llama-cpp-python`). Đổi port từ 8080 sang 8082 do bị trùng port Docker, và fix lỗi CRLF (`\r`) do copy script từ Windows sang._

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| (Q4_K_M) | 822 | 391 / 2102 | 110.7 / 216.2 | 7896 / 13766 / 15312 | 9.0 |
| (Q2_K)   | 100 | 337 / 460 | 91.2 / 146.1 | 5605 / 7842 / 8266 | 11.0 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

_Mô hình Q4_K_M chạy chậm hơn Q2_K khoảng 20% (decode rate 9 tok/s so với 11 tok/s), nhưng bù lại chất lượng câu văn sinh ra tốt hơn hẳn. Với cấu hình CPU hiện tại, mức 9 tok/s hoàn toàn đủ dùng, nên việc đánh đổi lấy Q4 là hoàn toàn xứng đáng._

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.14 | 25000 | 46000 | 46000 | 0 |
| 50 | (skip) | | | | |

**Batching observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` ở concurrency 50 = _<…>_, nghĩa là …

_Ở concurrency 10, CPU phải phân chia tài nguyên để xử lý đồng thời 10 luồng nên thời gian chờ (TTFB) bị đẩy lên rất cao (25 giây). Server nhận tải tốt không báo lỗi nhưng throughput bị nghẽn ở giới hạn tính toán của CPU._

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** _stub: localhost only_
- **N17 (Data pipeline):** _stub: in-memory dict_
- **N18 (Lakehouse):** _stub: SQLite_
- **N19 (Vector + Feature Store):** _stub: TOY_DOCS_

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _0.0 ms_
- retrieve: _0.0 ms_
- llama-server: _6745.9 ms_

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

_Bottleneck nằm hoàn toàn ở khâu llama-server (gọi API sinh chữ của LLM) chiếm hơn 99% thời gian. Điều này hoàn toàn khớp với lý thuyết vì prefill và decode là các tác vụ ngốn compute nhất, đặc biệt khi chạy mô hình trên CPU thay vì GPU._

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Fallback sang CPU-only wheel thay vì compile CUDA_

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: 0 tok/s (lỗi build `llama-cpp-python` do conflict C++)
after:  ~10 tok/s (chạy trực tiếp trên CPU AMD Ryzen 9)
speedup: ~N/A×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

_Thay vì mất thời gian sửa lỗi thư viện C++ compiler (nvcc vs gcc) trên Windows/WSL2, việc đổi hướng sang tận dụng ngay sức mạnh CPU (16 luồng của AMD Ryzen 9) với thư viện CPU-only đã giúp bài lab chạy mượt mà ngay lập tức. Mặc dù CPU có bandwidth thấp hơn GPU, nhưng nhờ số luồng lớn và tập lệnh AVX2, nó hoàn toàn đủ khả năng gánh vác mô hình 1.1B parameters ở tốc độ ~10 token/giây (vượt mức đọc của con người)._

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

_Việc chạy một Pipeline AI hoàn chỉnh (gồm LLM Server, RAG) ngay trên một chiếc Laptop Windows (thông qua WSL2 CPU) vẫn cho tốc độ xử lý rất tốt và ổn định._

---

## 7. Self-graded checklist

- [ ] `hardware.json` đã commit
- [ ] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [ ] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
