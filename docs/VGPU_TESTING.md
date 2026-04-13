# Virtual GPU Testing Guide for DSSTNE

## Overview

This guide explains how to set up and run DSSTNE tests using virtual GPUs (vGPUs). Virtual GPUs allow testing in cloud environments and CI/CD pipelines without dedicated physical GPUs.

## Supported Virtual GPU Platforms

| Platform | Support Level | Notes |
|----------|---------------|-------|
| NVIDIA GRID | Full | Recommended for production testing |
| NVIDIA vGPU | Full | Enterprise virtualization |
| AWS EC2 GPU Instances | Full | g4dn, p3, p4 instance types |
| Google Cloud GPU | Full | N1 with attached GPUs |
| Azure NC-series | Full | NCv3 or NCasT4_v3 |
| Container (nvidia-docker) | Full | Docker with NVIDIA runtime |

## Prerequisites

### System Requirements

- Linux kernel 4.15+ (Ubuntu 18.04+, CentOS 7.6+)
- NVIDIA Driver 450.0+ (for CUDA 11.x)
- CUDA Toolkit 9.1+ (11.x recommended)
- cuDNN 7.0+
- vGPU license (if using NVIDIA GRID)

### Verify vGPU Installation

```bash
# Check if NVIDIA driver is loaded
nvidia-smi

# Expected output should show:
# - Driver Version
# - CUDA Version
# - Available GPU(s)

# Check CUDA installation
nvcc --version
```

## Configuration

### Environment Variables

```bash
# Specify which GPU to use (for multi-GPU systems)
export CUDA_VISIBLE_DEVICES=0

# Set GPU memory fraction (useful for vGPU with shared memory)
export TF_FORCE_GPU_ALLOW_GROWTH=true

# Enable vGPU-specific optimizations
export DSSTNE_VGPU_MODE=1
```

### DSSTNE vGPU Configuration

Create or modify `~/.dsstne/config.json`:

```json
{
    "gpu": {
        "device_id": 0,
        "memory_fraction": 0.8,
        "allow_growth": true,
        "vgpu_mode": true
    }
}
```

## Running Tests with vGPU

### Basic Test Execution

```bash
# Build DSSTNE
cd /path/to/amazon-dsstne
make clean && make

# Run GPU tests
cd tst
make
CUDA_VISIBLE_DEVICES=0 make run-tests
```

### Test with Memory Limits (vGPU)

When using vGPU with limited memory, set memory constraints:

```bash
# Set CUDA memory pool limit (in MB)
export CUDA_MEMORY_POOL_SIZE=2048

# Run tests with constrained memory
./build/tst/bin/unittests
```

### CI/CD Integration

#### GitHub Actions Example

```yaml
name: GPU Tests

on: [push, pull_request]

jobs:
  gpu-tests:
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup CUDA
        run: |
          nvidia-smi
          nvcc --version
          
      - name: Build DSSTNE
        run: make
        
      - name: Run GPU Tests
        env:
          CUDA_VISIBLE_DEVICES: 0
          DSSTNE_VGPU_MODE: 1
        run: |
          cd tst
          make run-tests
```

#### Docker with NVIDIA Runtime

```dockerfile
FROM nvidia/cuda:11.4-cudnn8-devel-ubuntu20.04

# Install DSSTNE dependencies
RUN apt-get update && apt-get install -y \
    libopenmpi-dev \
    libjsoncpp-dev \
    libnetcdf-dev \
    libnetcdf-c++4-dev \
    libcppunit-dev

# Copy and build DSSTNE
COPY . /opt/amazon/dsstne
WORKDIR /opt/amazon/dsstne
RUN make

# Run tests on container start
CMD ["bash", "-c", "cd tst && make run-tests"]
```

Run with:
```bash
docker build -t dsstne-test .
nvidia-docker run --rm dsstne-test
```

## vGPU-Specific Considerations

### Memory Management

Virtual GPUs may have different memory characteristics:

1. **Reduced Memory**: vGPUs typically have less memory than physical GPUs
2. **Memory Overcommit**: Some platforms allow memory overcommit
3. **Shared Memory**: Memory may be shared with other vGPUs

**Recommendations:**

```cpp
// In your test code, check available memory
size_t free, total;
cudaMemGetInfo(&free, &total);
printf("GPU Memory: %zu MB free of %zu MB total\n", 
       free / 1024 / 1024, total / 1024 / 1024);

// Adjust batch sizes based on available memory
size_t targetMemory = free * 0.8;  // Use 80% of available
```

### Performance Expectations

vGPU performance may differ from physical GPUs:

| Metric | Physical GPU | vGPU | Notes |
|--------|-------------|------|-------|
| Memory Bandwidth | 100% | 70-90% | Virtualization overhead |
| Compute | 100% | 80-95% | Depends on vGPU profile |
| Latency | Baseline | +5-15% | Context switching |

**Test Timeouts:**

```bash
# Increase test timeout for vGPU (slower execution)
export CPPUNIT_TEST_TIMEOUT=300  # seconds
```

### Error Handling

Common vGPU-specific errors:

1. **CUDA_ERROR_OUT_OF_MEMORY**
   - Reduce batch size
   - Check `nvidia-smi` for memory usage

2. **CUDA_ERROR_NO_DEVICE**
   - Verify vGPU is assigned
   - Check driver installation

3. **CUDA_ERROR_INVALID_DEVICE**
   - Verify CUDA_VISIBLE_DEVICES
   - Check vGPU profile compatibility

## Test Categories

### Tests Safe for vGPU

These tests work well on vGPU with limited resources:

- Unit tests (small batch sizes)
- Activation function tests
- Loss function tests
- Weight initialization tests
- Data loading tests

### Tests Requiring Adjustment

These may need modified parameters for vGPU:

- Large batch training tests
- Multi-GPU tests
- Memory-intensive operations
- Performance benchmarks

### Skipping Tests on vGPU

```cpp
// In test code, detect vGPU and skip heavy tests
#ifdef DSSTNE_VGPU_MODE
    CPPUNIT_TEST_SKIP("Skipped on vGPU: requires >8GB memory");
#else
    CPPUNIT_TEST(testLargeModelTraining);
#endif
```

## Troubleshooting

### Debugging vGPU Issues

```bash
# Enable CUDA debugging
export CUDA_LAUNCH_BLOCKING=1

# Enable verbose error messages
export DSSTNE_DEBUG=1

# Run with memory debugging
cuda-memcheck ./build/tst/bin/unittests
```

### Common Solutions

| Issue | Solution |
|-------|----------|
| Out of memory | Reduce batch size, use streaming |
| Slow performance | Check vGPU profile, reduce model size |
| Driver mismatch | Update NVIDIA driver |
| Tests timeout | Increase timeout, reduce test scope |

## Appendix: vGPU Profiles

### NVIDIA GRID Profiles

| Profile | Memory | vGPUs per GPU | Use Case |
|---------|--------|---------------|----------|
| 1Q | 1GB | 16 | Light inference |
| 2Q | 2GB | 8 | Standard testing |
| 4Q | 4GB | 4 | Training tests |
| 8Q | 8GB | 2 | Full testing |

### Recommended Profile for DSSTNE Tests

- **Minimum**: 2Q (2GB) for unit tests
- **Recommended**: 4Q (4GB) for full test suite
- **Optimal**: 8Q (8GB) for training + inference tests

---

*Document Version: 1.0*
*Last Updated: April 2026*
