# Docker Compose: Q6 cache with RDNA3 attention

The root `docker-compose.yml` serves the OpenAI-compatible TabbyAPI endpoint on
port 8080 with Qwen3.8-27B EXL3 3.5 bpw, DFlash2 draft 7, a 131,072-token
context, target KV cache Q6, and draft KV cache Q4. It enables the experimental
Q6 attention path with `EXL3_RDNA3_ATTN_Q6=1`.

The Compose file is intended for Linux and the tested RX 7900 XTX / gfx1100
setup. Paths are relative to the repository root. The accompanying config
matches the benchmark setup, including disabled API authentication.

## Required server image

`qwen-exl3-server:20261012` is a **local image**, not a published registry image.
It contains the ROCm/PyTorch toolchain and the patched TabbyAPI used in the
benchmarks. This repository does not currently provide a self-contained build
for that image. Obtain the same image from its owner before starting Compose.
Cloning this repository alone is not sufficient.

For example, its owner can export the image:

```bash
docker save -o qwen-exl3-server.tar qwen-exl3-server:20261012
```

Import it on the destination machine:

```bash
docker load -i qwen-exl3-server.tar
```

## Prepare the source, extension, and models

```bash
git clone --branch rdna3-q6-attn-20261017 \
  https://github.com/marbink/exllamav3-rocm.git
cd exllamav3-rocm

# Build inside the server image to match its Python/PyTorch/ROCm ABI.
# Generated extension binaries are intentionally not committed to Git.
docker run --rm \
  -e PYTORCH_ROCM_ARCH=gfx1100 \
  -e MAX_JOBS=2 \
  -v "$PWD:/opt/exllamav3" \
  -w /opt/exllamav3 \
  qwen-exl3-server:20261012 \
  python3 setup.py build_ext --inplace

# Requires the Hugging Face CLI on the host; see the main quick start.
rocm/scripts/download_models.sh models
mkdir -p triton-cache logs
```

The download script creates these directories under `models/`:

- `Qwen3.8-27B-EXL3-3.5bpw`
- `Qwen3.8-27B-DFlash2-EXL3-5.0bpw`

## Start and inspect

From the repository root:

```bash
docker compose up -d
docker compose logs -f qwen38
```

Health endpoint: `http://localhost:8080/health`.
OpenAI-compatible API base: `http://localhost:8080/v1`.

The source tree and compiled extension are mounted read-only while serving.
After changing the native source, rebuild the extension in the matching image
and recreate the service to load the new binary.
