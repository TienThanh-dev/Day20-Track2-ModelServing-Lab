# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Vũ Tiến Thanh
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 11 Home (AMD64)
- **CPU:** unknown (AMD64 architecture)
- **Cores:** 20 physical / 20 logical
- **CPU extensions:** AVX2 (via CPU build)
- **RAM:** CPU-only build — detect-hardware.py không đọc được RAM chính xác trên máy này
- **Accelerator:** CPU only (no GPU detected)
- **llama.cpp backend đã chọn:** CPU (AVX/NEON tuning) — prebuilt llama-cpp-python wheel
- **Recommended model tier:** TinyLlama-1.1B (Q4_K_M) — được chọn do không detect được RAM > 32GB nên fallback về tier nhỏ nhất

**Setup story** (≤ 80 chữ): Dùng prebuilt llama-cpp-python wheel (CPU-only). Không cần CUDA/Vulkan vì máy không có GPU rời. Tải llama.cpp release bin (llama-server.exe) để chạy server thay vì `python -m llama_cpp.server` cho ổn định hơn. Tất cả scripts Python đều chạy trực tiếp trên Windows PowerShell.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 738 | 69 / 84 | 19.9 / 35.8 | 1283 / 1352 / 1357 | 50.1 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 332 | 113 / 144 | 16.5 / 18.7 | 1137 / 1193 / 1207 | 60.6 |

Settings: `n_threads=20`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=0` (CPU-only).

**Một quan sát** (≤ 50 chữ): Q2_K decode nhanh hơn (60.6 vs 50.1 tok/s) nhưng TTFT chậm hơn nhiều (113 vs 69 ms). Q4_K_M cân bằng hơn — TTFT thấp, decode chấp nhận được, quality tốt hơn đáng kể.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | ~22 | 3100 | 5500 | 6900 | 0 |
| 50 | ~39 | 7400 | 13000 | 15000 | 4 |

*(Số liệu từ locust headless output — xem 04-locust-10.png và 05-locust-50.png. Concurrency 50 có 4 failures do timeout.)*

**Smoke test tổng hợp** (từ 02-smoke-test sau load test cuối):
- prompt_tokens_total: 3509 (tổng prompt tokens đã xử lý)
- tokens_predicted_total: 26827 (tổng tokens đã generate)
- n_decode_total: 7331 (số decode calls)
- n_busy_slots_per_decode: **3.75** trung bình (~4 concurrent requests mỗi decode call)
- prompt_tokens_seconds: 340.58 tok/s (prefill throughput)
- predicted_tokens_seconds: 36.16 tok/s (decode throughput)

**KV-cache observation** (từ `record-metrics.py`): metric `llamacpp:kv_cache_usage_ratio` **không được expose** trên CPU-only build của llama-cpp-python prebuilt wheel. Tuy nhiên, `n_busy_slots_per_decode = 3.75` và `requests_processing = 4` cho thấy engine đang xử lý ~4 concurrent requests đồng thời — chứng tỏ KV cache đang active và hoạt động. Peak `reqs_proc=4` với `deferred=4` ở concurrency 50 cho thấy hệ thống bắt đầu queue requests khi tất cả slots bận.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only (không kết nối cloud cluster)
- **N17 (Data pipeline):** stub: in-memory TOY_DOCS dict (không dùng Airflow/batch job)
- **N18 (Lakehouse):** stub: TOY_DOCS (không dùng Delta Lake / Parquet)
- **N19 (Vector + Feature Store):** stub: keyword-overlap retrieval trên TOY_DOCS (không dùng Qdrant / Chroma / FAISS)

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: _N/A (stub — keyword overlap, không có embedder thật)_
- retrieve: _<1ms (toy keyword match trên 5 docs)_
- llama-server: ~3200ms (trung bình từ 3 queries trong pipeline run, xem 09-pipeline-output.png)

**Reflection** (≤ 60 chữ): llama-server là bottleneck chính (~3-4s per query). Retrieve gần như instant vì toy data. Nếu dùng real embedder (N19) thì embed ms sẽ tăng, nhưng llama-server vẫn dominate.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** _Chưa thực hiện bonus track — vui lòng chạy `python BONUS-llama-cpp-optimization/benchmarks/thread-sweep.py` và điền kết quả vào đây._

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: <số liệu default threads>
after:  <số liệu optimal threads>
speedup: ~<X.Y>×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

_Giải thích ở đây bằng mental model hardware: memory bandwidth ceiling, physical vs logical cores, cache locality._

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Q2_K decode nhanh hơn Q4_K_M (60.6 vs 50.1 tok/s) nhưng TTFT lại chậm hơn gần gấp đôi (113 vs 69 ms). Điều này ngược với trực giác — tưởng quantization nhỏ hơn thì mọi thứ nhanh hơn. Thực tế Q2_K có prefill cost cao hơn vì decompression overhead vượt qua lợi ích của fewer weights.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-metrics.csv` đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.