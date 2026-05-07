# gguf-deploy

Compose files for serving GGUF models via [llama.cpp](https://github.com/ggml-org/llama.cpp) or [vLLM](https://github.com/vllm-project/vllm).

## Files

| File | Engine | Hardware | Image |
|---|---|---|---|
| [compose-llama-cpp-cpu.yaml](compose-llama-cpp-cpu.yaml) | llama.cpp server | CPU only | `ghcr.io/ggml-org/llama.cpp:server-b9038` |
| [compose-llama-cpp-gpu.yaml](compose-llama-cpp-gpu.yaml) | llama.cpp server | NVIDIA GPU | `ghcr.io/ggml-org/llama.cpp:server-cuda-b9038` |
| [compose-vllm-gpu.yaml](compose-vllm-gpu.yaml) | vLLM OpenAI server | NVIDIA GPU | `vllm/vllm-openai:v0.19.0` |

All three default to `unsloth/Qwen3-14B-GGUF` (IQ4_XS).

## Pre-download (optional)

```zsh
export HF_HOME=${PWD}/.cache/huggingface 
export HF_TOKEN=hf_cool_token
hf download unsloth/Qwen3-14B-GGUF Qwen3-14B-IQ4_XS.gguf
```

## Run

`.env` 
```ini
HF_TOKEN=hf_cool_token
```

```zsh
docker compose -f compose-llama-cpp-cpu.yaml up -d
docker compose -f compose-llama-cpp-gpu.yaml up -d
docker compose -f compose-vllm-gpu.yaml up -d
```

## ⚠️ Warnings

### Port collision
All three files bind host port **8791**. Only run **one at a time**, or edit the `ports:` mapping before bringing a second stack up.

### Mounted volumes
- `${PWD}/models` → `/models` (llama.cpp only) — pre-downloaded GGUF files.
- `${PWD}/.cache/` → `/root/.cache/` (all three) — HuggingFace cache. **Shared across stacks**; switching engines re-uses downloads but also means a corrupt cache affects every container. The container writes here as **root**, so files on the host will be root-owned.

### CPU usage
- [compose-llama-cpp-cpu.yaml](compose-llama-cpp-cpu.yaml) sets `LLAMA_ARG_THREADS=16` and a **16 GB memory limit**. Tune `THREADS` to your **physical** core count (not SMT threads) and raise the limit if the model + KV cache exceeds it — the container will be OOM-killed otherwise.
- Expect low tokens/sec on CPU; intended for testing or small-load serving.

### GPU usage
- Both GPU files request **all** NVIDIA GPUs (`count: all`). Set `count: 1` (or use `device_ids`) to pin a specific card on multi-GPU hosts.
- Requires the NVIDIA Container Toolkit on the host.
- [compose-llama-cpp-gpu.yaml](compose-llama-cpp-gpu.yaml) offloads **all** layers (`N_GPU_LAYERS=999`).
- [compose-vllm-gpu.yaml](compose-vllm-gpu.yaml) reserves **90%** of VRAM (`--gpu-memory-utilization 0.90`) and sets `shm_size: 16gb` plus `ipc: host` — do not run alongside other GPU workloads on the same card.
- vLLM's GGUF support is experimental; the tokenizer is pulled separately from `Qwen/Qwen3-14B`.
