# YOLO26 BPU Model Conversion Guide (x86 Machine)

This guide provides step-by-step instructions for converting YOLO26 ONNX models to BPU-compatible `.bin` models using an x86 Linux machine.

## Overview

```
[RDK X5 Device]                    [x86 Linux Machine]
      |                                    |
      |  1. Export ONNX (done)             |
      |  --------------------------->      |
      |     yolo26n_det_bpu.onnx           |
      |                                    |
      |                              2. Run hb_mapper
      |                                 (Docker/Pip)
      |                                    |
      |  3. Transfer .bin file             |
      |  <---------------------------      |
      |     yolo26n_det_bpu_*.bin          |
      |                                    |
      |  4. Run inference                  |
      v                                    v
```

---

## Prerequisites

### x86 Machine Requirements
- **OS**: Ubuntu 20.04 / 22.04 (64-bit)
- **RAM**: 16GB+ recommended
- **Disk**: 20GB+ free space
- **Network**: Internet access for Docker/pip downloads

### Required Files (from RDK X5)
Copy these files from your RDK X5 device to the x86 machine:

| File | Source Path | Description |
|------|-------------|-------------|
| ONNX Model | `samples/vision/yolo26/model/yolo26n_det_bpu.onnx` | Exported BPU-optimized ONNX |
| Mapper Script | `samples/vision/yolo26/conversion/mapper.py` | Conversion automation script |
| Calibration Images | 20-50 COCO images | For quantization calibration |

---

## Method 1: Pip Installation (Recommended)

Choose either **Option A (uv)** or **Option B (Miniconda)** for Python environment setup.

### Option A: Using uv (Faster)

[uv](https://github.com/astral-sh/uv) is a fast Python package manager. Recommended for users who prefer lightweight tooling.

```bash
# Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create virtual environment with Python 3.10
uv venv --python 3.10 .venv

# Activate
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# Install RDK X5 toolchain and dependencies
uv pip install rdkx5-yolo-mapper opencv-python numpy onnxruntime

# Verify installation
hb_mapper --version
# Expected: hb_mapper, version 1.24.3+
```

If download is slow, use a mirror:
```bash
uv pip install rdkx5-yolo-mapper opencv-python numpy onnxruntime \
  --index-url https://mirrors.aliyun.com/pypi/simple/
```

### Option B: Using Miniconda

```bash
# Download Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3

# Initialize
$HOME/miniconda3/bin/conda init bash
source ~/.bashrc

# Create environment with Python 3.10
conda create -n rdk_x5 python=3.10 -y

# Activate
conda activate rdk_x5

# Install RDK X5 toolchain and dependencies
pip install rdkx5-yolo-mapper opencv-python numpy onnxruntime

# Verify installation
hb_mapper --version
# Expected: hb_mapper, version 1.24.3+
```

If download is slow, use a mirror:
```bash
pip install rdkx5-yolo-mapper -i https://mirrors.aliyun.com/pypi/simple/
```

---

## Method 2: Docker Installation

### Step 1: Install Docker

```bash
# Install Docker (if not already installed)
sudo apt update
sudo apt install -y docker.io

# Add user to docker group (to run without sudo)
sudo usermod -aG docker $USER
newgrp docker
```

### Step 2: Pull the RDK X5 Toolchain Image

```bash
# CPU version (recommended for most users)
docker pull openexplorer/ai_toolchain_ubuntu_20_x5_cpu:v1.2.8

# GPU version (requires NVIDIA Docker)
# docker pull openexplorer/ai_toolchain_ubuntu_20_x5_gpu:v1.2.8
```

### Step 3: Run the Container

```bash
# Create a working directory
mkdir -p ~/rdk_workspace
cd ~/rdk_workspace

# Copy your files here (ONNX model, mapper.py, calibration images)

# Run Docker with mounted volume
docker run -it --rm \
  -v $(pwd):/workspace \
  openexplorer/ai_toolchain_ubuntu_20_x5_cpu:v1.2.8 \
  /bin/bash
```

Inside the container:
```bash
cd /workspace
```

---

## Conversion Steps

### Step 1: Prepare Working Directory

```bash
# Create directory structure
mkdir -p ~/rdk_workspace/{model,calibration_images}
cd ~/rdk_workspace

# Copy ONNX model (from RDK X5 via scp)
scp sunrise@<RDK_X5_IP>:/app/github/rdk_model_zoo/samples/vision/yolo26/model/yolo26n_det_bpu.onnx ./model/

# Copy mapper script
scp sunrise@<RDK_X5_IP>:/app/github/rdk_model_zoo/samples/vision/yolo26/conversion/mapper.py ./
```

### Step 2: Prepare Calibration Images

You need 20-50 images for quantization calibration. Options:

**Option A: Download COCO128 Sample**
```bash
# Download COCO128 dataset
wget https://ultralytics.com/assets/coco128.zip
unzip coco128.zip -d ./
mv coco128/images/train2017/* ./calibration_images/
rm -rf coco128 coco128.zip
```

**Option B: Copy from RDK X5**
```bash
scp -r sunrise@<RDK_X5_IP>:/app/COCO/coco128/images/train2017/* ./calibration_images/
```

**Option C: Use Custom Images**
Place 20-50 representative images (JPG/PNG) in `./calibration_images/`

### Step 3: Run the Conversion

```bash
# Activate environment (choose one)
source .venv/bin/activate  # if using uv
conda activate rdk_x5      # if using Miniconda

# Run mapper
python3 mapper.py \
  --onnx ./model/yolo26n_det_bpu.onnx \
  --cal-images ./calibration_images/ \
  --quantized int8 \
  --optimize-level O3
```

**Expected Output:**
```
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] Starting conversion for: ./model/yolo26n_det_bpu.onnx
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] hb_mapper tool is verified.
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] Model Input Resolution: 640x640
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] Generating binary calibration data...
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] Running: hb_mapper makertbin --config config.yaml --model-type onnx
...
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] BPU Model saved to: ./model/yolo26n_det_bpu_bayese_640x640_nv12.bin
[YOLO26_Mapper] [HH:MM:SS.xxx] [INFO] Conversion Workflow Completed.
```

### Step 4: Transfer .bin to RDK X5

```bash
# Copy the converted model back to RDK X5
scp ./model/yolo26n_det_bpu_bayese_640x640_nv12.bin \
  sunrise@<RDK_X5_IP>:/app/github/rdk_model_zoo/samples/vision/yolo26/model/
```

---

## Conversion Options Reference

| Option | Default | Description |
|--------|---------|-------------|
| `--onnx` | (required) | Path to BPU-optimized ONNX model |
| `--cal-images` | `./cal_images` | Directory with calibration images |
| `--output-dir` | (same as onnx) | Output directory for .bin |
| `--quantized` | `int8` | Quantization: `int8` or `int16` |
| `--optimize-level` | `O3` | Compiler optimization: O0-O3 |
| `--jobs` | `16` | Parallel compilation jobs |
| `--cal-sample-num` | `20` | Number of calibration images to use |
| `--save-cache` | `False` | Keep temporary files for debugging |

---

## Troubleshooting

### Error: `hb_mapper: command not found`
- Ensure virtual environment is activated:
  - uv: `source .venv/bin/activate`
  - Miniconda: `conda activate rdk_x5`
- Or verify Docker container is running with the toolchain image

### Error: `No valid images found in calibration directory`
- Check that calibration images are JPG/PNG format
- Ensure at least 20 images are present

### Error: Out of Memory
- Reduce `--jobs` parameter (e.g., `--jobs 4`)
- Close other applications
- Use Docker with `--shm-size="15g"` flag

### Conversion is Slow
- Use `--optimize-level O2` instead of O3 for faster conversion
- Ensure you have enough RAM (16GB+)

### Model Compatibility Issues
- Verify ONNX was exported with `export_yolo26_detect_bpu.py`
- Check toolchain version matches libdnn.so on RDK X5

---

## Quick Reference Commands

### Linux/macOS

```bash
# Full conversion command (uv)
source .venv/bin/activate && \
python3 mapper.py \
  --onnx ./model/yolo26n_det_bpu.onnx \
  --cal-images ./calibration_images/

# Full conversion command (Miniconda)
conda activate rdk_x5 && \
python3 mapper.py \
  --onnx ./model/yolo26n_det_bpu.onnx \
  --cal-images ./calibration_images/

# Docker one-liner (Linux/macOS)
docker run -it --rm \
  -v $(pwd):/workspace \
  openexplorer/ai_toolchain_ubuntu_20_x5_cpu:v1.2.8 \
  bash -c "cd /workspace && python3 mapper.py --onnx ./model/yolo26n_det_bpu.onnx --cal-images ./calibration_images/"
```

### Windows (Docker Desktop)

```powershell
# Download calibration images (PowerShell)
Invoke-WebRequest -Uri "https://ultralytics.com/assets/coco128.zip" -OutFile "coco128.zip"
Expand-Archive -Path "coco128.zip" -DestinationPath "."
New-Item -ItemType Directory -Force -Path "calibration_images"
Move-Item -Path "coco128\images\train2017\*" -Destination "calibration_images\"
Remove-Item -Recurse -Force "coco128", "coco128.zip"

# Docker one-liner (Windows - use explicit path)
docker run --rm -v "d:/path/to/rdk_model_zoo:/workspace" openexplorer/ai_toolchain_ubuntu_20_x5_cpu:v1.2.8 bash -c "cd /workspace && python3 samples/vision/yolo26/conversion/mapper.py --onnx ./yolo26n_det_bpu.onnx --cal-images ./calibration_images/"
```

> **Note for Windows users:**
> - Docker Desktop must be running
> - Use forward slashes `/` in the volume path (e.g., `d:/path/to/...`)
> - Do not use `-it` flag when running non-interactively (use `--rm` only)

---

## Next Steps (on RDK X5)

After transferring the `.bin` file back to RDK X5:

```bash
cd /app/github/rdk_model_zoo/samples/vision/yolo26/runtime/python/

python3 main.py --task detect \
  --model-path ../../model/yolo26n_det_bpu_bayese_640x640_nv12.bin \
  --test-img ../../test_data/bus.jpg \
  --score-thres 0.25 \
  --nms-thres 0.7
```

---

## References

- [uv - Fast Python Package Manager](https://github.com/astral-sh/uv)
- [Docker Hub: openexplorer/ai_toolchain_ubuntu_20_x5_cpu](https://hub.docker.com/r/openexplorer/ai_toolchain_ubuntu_20_x5_cpu)
- [D-Robotics RDK Documentation](https://developer.d-robotics.cc/rdk_doc/en/)
- [D-Robotics GitHub](https://github.com/D-Robotics)
