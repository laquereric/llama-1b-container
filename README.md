# llama-1b-container

Run **Llama-3.2-1B-Instruct** with GPU acceleration on any Apple Silicon Mac using a Podman container and `libkrun`. The Linux container accesses your Mac's Metal GPU via the Venus Vulkan translation layer — no CUDA, no cloud, fully local.

## Requirements

- Apple Silicon Mac (M1/M2/M3/M4)
- [Homebrew](https://brew.sh)
- ~10 GB free disk space (image build + model)

---

## 1. Install Dependencies

```bash
brew install podman
brew tap slp/krunkit
brew install krunkit
```

---

## 2. Initialize the Podman Machine

> **Important:** The `libkrun` provider must be set **before** running `podman machine init`. If you already have a default machine, remove it first.

```bash
# Persist the provider in your shell profile (do this first)
echo 'export CONTAINERS_MACHINE_PROVIDER="libkrun"' >> ~/.zshrc
source ~/.zshrc
```

If you already initialized a machine with the default provider, remove it:

```bash
podman machine stop
podman machine rm
```

Then initialize a new machine with `libkrun`:

```bash
podman machine init --cpus 4 --memory 8192 --disk-size 100
podman machine start
```

Verify GPU access:

```bash
podman machine ssh ls /dev/dri
# Expected: by-path  card0  renderD128
```

If `/dev/dri` is missing, the machine is not using `libkrun`. Run `podman machine rm` and retry after confirming `echo $CONTAINERS_MACHINE_PROVIDER` outputs `libkrun`.

---

## 3. Get the Container Image

Choose one path:

### Option A — Download pre-built image (faster)

```bash
podman pull ghcr.io/laquereric/llama-cpp-vulkan:latest
podman tag ghcr.io/laquereric/llama-cpp-vulkan:latest llama-cpp-vulkan
```

### Option B — Build from source

```bash
git clone https://github.com/laquereric/llama-1b-container.git
cd llama-1b-container
podman build -t llama-cpp-vulkan -f Containerfile.llama-vulkan .
```

The build compiles `glslc` (from [google/shaderc](https://github.com/google/shaderc)) and `llama-server` from source with Vulkan enabled. Expect ~20–30 minutes on first run.

---

## 4. Download the Model

```bash
mkdir -p ~/models
curl -L -o ~/models/Llama-3.2-1B-Instruct-Q4_K_M.gguf \
  "https://huggingface.co/unsloth/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_K_M.gguf"
```

---

## 5. Run the Server

```bash
podman run --rm -it \
  --device /dev/dri \
  -v ~/models:/models \
  -p 8080:8080 \
  llama-cpp-vulkan \
  -m /models/Llama-3.2-1B-Instruct-Q4_K_M.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --n-gpu-layers 99
```

| Flag | Purpose |
|------|---------|
| `--device /dev/dri` | Passes the virtualized GPU into the container |
| `-v ~/models:/models` | Mounts your local models folder |
| `--n-gpu-layers 99` | Offloads all layers to the Vulkan GPU |

---

## 6. Query the API

The server exposes an OpenAI-compatible API:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

---

## How It Works

```
macOS Metal GPU
      │
  libkrun VM  (krunkit hypervisor)
      │  virtio-gpu (Venus protocol)
  AlmaLinux 9 container
      │  Vulkan (patched mesa-krunkit driver)
  llama.cpp (llama-server)
```

`libkrun` creates a lightweight VM that exposes the host GPU as a `virtio-gpu` device. The patched Mesa driver inside the container translates Vulkan calls over that device back to Metal on the host.
