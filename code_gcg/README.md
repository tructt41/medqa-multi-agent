# Full source — GCG universal trên Vicuna-7B-v1.3

Toàn bộ code đã chạy trên GPU thuê (Vast.ai instance 50898440, 1× RTX 3090 24GB) để sinh suffix
universal GCG và pack input cho SmoothLLM. Tất cả path trong script là path **trên máy remote**
(`/workspace/...`, `/opt/miniforge3/...`).

## Thứ tự chạy
| Bước | File | Mô tả | Nơi chạy |
|---|---|---|---|
| 1 | `setup_env.sh` | Tạo conda env `gcg` (py3.10) + torch cu121 + transformers 4.28.1 + fschat 0.2.20 … | CPU |
| 2 | `download_model.sh` | Tải Vicuna-7B-v1.3 fp16 từ HuggingFace về `/workspace/models` | mạng |
| 3 | `clone_and_patch.sh` | Clone llm-attacks, `pip install -e`, đặt `transfer_vicuna_single.py`, chạy `patch_attack_manager.py` | CPU |
| 4 | `run_gcg.sh` | Chạy GCG universal 500 bước (nền: `nohup bash run_gcg.sh &`) | GPU (~21h) |
| 5 | `post_run.sh` | Sau khi GCG xong: build input SmoothLLM + verify jailbreak thật | CPU + GPU |
| — | `host_model.sh` | (Tuỳ chọn) host Vicuna fp16 qua FastChat OpenAI API cổng 8080 cho eval C1/C2 | GPU |

## Các file
- **`transfer_vicuna_single.py`** — config GCG (llm-attacks): `transfer=True`, `progressive_goals=True`, 1 model Vicuna, template `vicuna`, `cuda:0`. Đặt vào `llm-attacks/experiments/configs/`.
- **`patch_attack_manager.py`** — patch lớp `ModelWorker`: chạy đồng bộ 1 tiến trình (bỏ `spawn` nhân đôi model) + `requires_grad_(False)`. **Bắt buộc** để 7B fp16 vừa 24GB (nếu không sẽ OOM).
- **`build_smoothllm_inputs.py`** — đọc logfile GCG + advbench → xuất `universal_suffix.json`, `attack_prompts.jsonl/.csv` (100 câu held-out row 25:125), `refusal_prefixes.json`, `smoothllm_config.json` vào `/workspace/results/smoothllm`.
- **`verify_jailbreak.py [K]`** — sinh câu trả lời thật (goal+suffix vs goal-only) cho K câu → `jailbreak_verification.jsonl` + `jailbreak_summary.json` (ASR).
- **`gcg_universal_vicuna.ipynb`** — notebook mô tả toàn bộ pipeline đã chạy (mirror các script trên).

## Kết quả (đã kéo về local)
- `../results/vicuna-7b-v1.3/` — log GCG + `universal_suffix.json`
- `../results/smoothllm/` — pack input cho SmoothLLM + verify jailbreak
- `../vicuna_gcg_run_config.md` — bảng tham số + kết quả + lệnh host

Tóm tắt lần chạy: 500/500 bước (~21h), best loss ≈ 0.26, verify ASR có suffix = **1.00 (25/25)**, baseline sạch = **0.08 (2/25)**.

## Lưu ý
- Suffix tối ưu trên **fp16 HF**. Eval C1/C2 nên chạy trên đúng bản fp16 (dùng `host_model.sh`); nếu chạy Ollama/GGUF quantized phải đo lại ASR.
- `"Passed"` trong log GCG chỉ là heuristic 16-token — dùng `jailbreak_verification.jsonl` để xác nhận jailbreak thật.
- MedQA / C0 / C3 là benign, không gắn suffix.
