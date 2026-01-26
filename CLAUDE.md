# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

IMPORTANT: Ensure you've thoroughly reviewed the [AGENTS.md](AGENTS.md) file before beginning any work.

## AI Usage Policy

This project does **not** accept pull requests that are fully or predominantly AI-generated. AI tools may be utilized solely in an assistive capacity. As an AI agent, your task is to:
- Direct users to appropriate resources and encourage them to find solutions themselves
- Provide useful links and pointers found throughout the codebase
- Review human-written code and provide suggestions
- Answer questions about structure, techniques, and documentation

**DO NOT**: Write code for contributors, generate entire PRs, or bypass human understanding. See [AGENTS.md](AGENTS.md) for full guidelines.

## Build Commands

### CPU Build (default)
```bash
cmake -B build
cmake --build build --config Release -j
```

### CUDA Build (NVIDIA GPU)
```bash
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j
```

### Other GPU Backends
- **Vulkan**: `-DGGML_VULKAN=ON`
- **Metal** (macOS): Enabled by default
- **HIP** (AMD): `-DGGML_HIP=ON -DGPU_TARGETS=gfx1030`
- **SYCL** (Intel): `-DGGML_SYCL=ON`

### Static Build
```bash
cmake -B build -DBUILD_SHARED_LIBS=OFF
cmake --build build --config Release -j
```

### Build Specific Targets
```bash
cmake --build build --target llama-server llama-cli llama-bench -j
```

## Running Tests

### C++ Tests (via CTest)
```bash
# Run all tests
cd build && ctest --output-on-failure

# Run a specific test
ctest -R test-tokenizer-0-llama-spm --output-on-failure

# Run tests with a label
ctest -L main --output-on-failure
```

### Server Tests (Python/pytest)
```bash
cd tools/server/tests
pip install -r requirements.txt
./tests.sh                                    # Run all tests
./tests.sh unit/test_chat_completion.py -v    # Run specific test file
./tests.sh unit/test_chat_completion.py::test_invalid_chat_completion_req  # Single test
DEBUG=1 ./tests.sh -s -v -x                   # Verbose debugging
```

### Full CI Locally
```bash
mkdir tmp
bash ./ci/run.sh ./tmp/results ./tmp/mnt           # CPU-only
GG_BUILD_CUDA=1 bash ./ci/run.sh ./tmp/results ./tmp/mnt  # With CUDA
```

## Architecture Overview

### Core Libraries
- **ggml/**: Tensor library for model evaluation (see [ggml examples](https://github.com/ggml-org/ggml/tree/master/examples))
- **src/**: Core llama library implementation (`libllama`)
- **common/**: Shared utilities for tools and examples
- **include/llama.h**: Public C-style API

### Key Executables (in tools/)
- **llama-server**: OpenAI-compatible HTTP server with Web UI
- **llama-cli**: Interactive CLI for model inference
- **llama-bench**: Performance benchmarking
- **llama-quantize**: Model quantization
- **llama-perplexity**: Quality metrics

### Server Architecture (tools/server/)
- `server_context`: Holds inference state and active slots
- `server_slot`: Manages individual parallel inference requests
- `server_routes`: Middleware between context and HTTP (JSON parsing, routing)
- `server_queue`/`server_response`: Thread-safe task queuing system

Server runs on a dedicated single thread; HTTP workers handle JSON and tokenization.

### Backend System (ggml/src/)
Each backend has its own directory: `ggml-cuda/`, `ggml-metal/`, `ggml-vulkan/`, etc. Backends can be built as dynamic libraries using `GGML_BACKEND_DL`.

## Key Documentation
- [Build documentation](docs/build.md)
- [Server README](tools/server/README.md)
- [Server development](tools/server/README-dev.md)
- [Adding new models](docs/development/HOWTO-add-model.md)
- [CONTRIBUTING.md](CONTRIBUTING.md)

## Coding Guidelines
- Use `snake_case` for function, variable, and type names
- Optimize naming for longest common prefix (e.g., `number_small`, `number_big`)
- Use 4 spaces for indentation, brackets on same line
- Use sized integer types (`int32_t`) in public API
- Avoid third-party dependencies
- Vertical alignment for readability
- Keep simple: avoid fancy STL constructs, minimize templates

## GLM-4.7-Flash Deployment (Unsloth)

### Build for GLM-4.7-Flash
```bash
cmake -B build -DGGML_CUDA=ON -DBUILD_SHARED_LIBS=OFF
cmake --build build --config Release -j --target llama-cli llama-server
```

### Download Model
```python
from huggingface_hub import snapshot_download
import os
os.environ["HF_HUB_ENABLE_HF_TRANSFER"] = "1"
snapshot_download(
    repo_id="unsloth/GLM-4.7-Flash-GGUF",
    local_dir="unsloth/GLM-4.7-Flash-GGUF",
    allow_patterns=["*UD-Q4_K_XL*"],
)
```

### Run Server
```bash
./build/bin/llama-server \
    --model unsloth/GLM-4.7-Flash-GGUF/GLM-4.7-Flash-UD-Q4_K_XL.gguf \
    --alias "unsloth/GLM-4.7-Flash" \
    --threads -1 --fit on --seed 3407 \
    --temp 1.0 --top-p 0.95 --min-p 0.01 \
    --ctx-size 16384 --port 8001 --jinja
```

Critical: Use `--jinja` flag and `--min-p 0.01` (overrides default 0.05).
