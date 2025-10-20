# MMCV Wheels Build Project

English | [简体中文](./README.md)

## Project Overview

This project provides an automated build system for compiling MMCV (OpenMMLab Computer Vision Foundation Library) Python wheels with various configurations. Using GitHub Actions workflows, it automatically compiles and publishes pre-built wheels for different Python versions, CUDA versions, PyTorch versions, and C++11 ABI settings.

## Features

- **Multi-version Support**: Python 3.9, 3.10, 3.11, 3.12
- **CUDA Compatibility**: CUDA 11.8.0 and 12.9.1
- **PyTorch Versions**: PyTorch 2.4.0, 2.5.1, 2.6.0, 2.7.1
- **C++11 ABI**: Builds both C++11 ABI enabled and disabled versions
- **NGC Container Support**: Builds from NVIDIA NGC PyTorch container images
- **Automated Release**: Automatically uploads built wheels to GitHub Releases
- **Optimized Build Environment**: Includes disk space optimization and swap configuration

## Project Structure

```
mmcv-wheels/
├── .github/
│   ├── workflows/
│   │   ├── _build.yml              # Standard build workflow template
│   │   ├── _build_in_container.yml # Container build workflow template
│   │   └── publish.yaml            # Publish workflow (main entry)
│   └── scripts/
│       ├── build.sh                # Build script
│       └── check_for_ngc_images.sh # NGC image detection script
└── README.md
```

## Workflow Description

### 1. Publish Workflow (`publish.yaml`)

Main workflow triggered on tag push (format: `v*`). Includes the following steps:

- **Create Release**: Automatically creates a GitHub Release based on git tag
- **Build Standard Wheels**: Builds standard wheels for all configuration matrix
- **Build NGC Wheels**: Builds wheels for NVIDIA NGC PyTorch containers

**Build Matrix Configuration**:
- OS: ubuntu-22.04
- Python versions: 3.9, 3.10, 3.11, 3.12
- PyTorch versions: 2.4.0, 2.5.1, 2.6.0, 2.7.1
- CUDA versions: 11.8.0, 12.9.1
- C++11 ABI: FALSE, TRUE

### 2. Standard Build Workflow (`_build.yml`)

Builds directly on GitHub Actions runner:

1. Set up Python environment
2. Install CUDA Toolkit
3. Install corresponding PyTorch version
4. Compile MMCV wheel
5. Upload to GitHub Release

**Key Features**:
- Disk space optimization (remove unnecessary tools)
- Configure 10GB swap space
- Network mode CUDA installation (improved stability)

### 3. Container Build Workflow (`_build_in_container.yml`)

Builds inside Docker containers (for NGC images):

1. Maximize build space
2. Pull NGC PyTorch container image
3. Extract environment configuration inside container
4. Compile MMCV wheel inside container
5. Upload to GitHub Release

**Advantages**:
- Uses NVIDIA official optimized PyTorch images
- Automatically matches CUDA and PyTorch versions in container
- Fully compatible with NGC container environment

## Build Scripts

### build.sh

Core build script that performs the following:

1. Install build dependencies (setuptools, ninja, packaging, wheel)
2. Set compilation flags based on `CXX11_ABI` environment variable
3. Build MMCV wheel from OpenMMLab official source
4. Rename wheel file to include CUDA, PyTorch version and ABI information

**Environment Variables**:
- `MMCV_VERSION`: MMCV version to build
- `WHEEL_CUDA_VERSION`: CUDA major version (e.g., 12)
- `MATRIX_TORCH_VERSION`: PyTorch version (e.g., 2.6)
- `TORCH_CUDA_VERSION`: Full CUDA version (e.g., 126)
- `CXX11_ABI`: Whether to enable C++11 ABI (TRUE/FALSE)
- `MATRIX_PYTHON_VERSION`: Python version (e.g., 312)

**Wheel Naming Format**:
```
mmcv-{version}+cu{cuda_version}torch{torch_version}cxx11abi{abi}-{platform}.whl
```

Example: `mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl`

### check_for_ngc_images.sh

Checks for existence of NVIDIA NGC PyTorch images from the last 7 months:

1. Generates image tags for the past 7 months (format: `YY.MM-py3`)
2. Uses `docker manifest inspect` to check image existence
3. Outputs JSON formatted list of available images

**Output Example**:
```json
["nvcr.io/nvidia/pytorch:24.10-py3", "nvcr.io/nvidia/pytorch:24.09-py3"]
```

## Usage

### Quick Installation Guide

#### Step 1: Check Your Environment

Before installation, verify your environment configuration:

```bash
# Check Python version
python --version
# Example output: Python 3.12.0

# Check PyTorch and CUDA versions
python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA: {torch.version.cuda}')"
# Example output:
# PyTorch: 2.6.0+cu126
# CUDA: 12.6

# Check C++11 ABI setting (if PyTorch is already installed)
python -c "import torch; print(f'C++11 ABI: {torch._C._GLIBCXX_USE_CXX11_ABI}')"
# Example output: C++11 ABI: False
```

#### Step 2: Select the Correct Wheel File

Visit the [Releases page](https://github.com/biubushy/mmcv-wheels/releases) and choose the wheel file matching your environment.

**Wheel File Naming Convention**:
```
mmcv-{version}+cu{cuda_ver}torch{torch_ver}cxx11abi{abi}-cp{py_ver}-cp{py_ver}-linux_x86_64.whl
```

**Example Filename Breakdown**:
```
mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
│         │  │  │         │            │    │
│         │  │  │         │            │    └─ Python 3.12
│         │  │  │         │            └────── Python 3.12
│         │  │  │         └─────────────────── C++11 ABI disabled
│         │  │  └───────────────────────────── PyTorch 2.6.0
│         │  └──────────────────────────────── CUDA 12.x
│         └─────────────────────────────────── MMCV version 2.1.0
└───────────────────────────────────────────── Package name
```

**Selection Criteria**:

1. **Python Version Match**
   - `cp39` → Python 3.9
   - `cp310` → Python 3.10
   - `cp311` → Python 3.11
   - `cp312` → Python 3.12

2. **CUDA Version Match**
   - If your CUDA is 11.x (11.0-11.8), choose files starting with `cu11`
   - If your CUDA is 12.x (12.0-12.9), choose files starting with `cu12`
   - **Tip**: PyTorch's CUDA version notation (e.g., `cu126`) indicates CUDA 12.6

3. **PyTorch Version Match**
   - Choose files matching your PyTorch **major and minor version**
   - Example: PyTorch 2.6.0 and 2.6.1 both use `torch2.6.0` files
   - Supported versions: 2.4.0, 2.5.1, 2.6.0, 2.7.1

4. **C++11 ABI Match**
   - **Most common case**: Choose `cxx11abiFALSE` (for standard PyTorch installed via pip)
   - **Special cases only**: Choose `cxx11abiTRUE` if:
     - Using NVIDIA NGC PyTorch containers
     - Using PyTorch compiled from source with C++11 ABI enabled
   - **How to verify**: Run the check command above; if output is `False`, use `cxx11abiFALSE`

#### Step 3: Download and Install

**Complete Example (Environment: Python 3.12 + PyTorch 2.6.0 + CUDA 12.6)**:

```bash
# 1. Download wheel file
wget https://github.com/biubushy/mmcv-wheels/releases/download/v2.1.0/mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl

# 2. Install (using full filename)
pip install mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl

# 3. Verify installation
python -c "import mmcv; print(mmcv.__version__)"
```

**Or install directly from URL**:

```bash
pip install https://github.com/biubushy/mmcv-wheels/releases/download/v2.1.0/mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
```

### Common Environment Examples

#### Example 1: Standard PyTorch Environment
```bash
# Environment info
Python 3.11.0
PyTorch 2.5.1+cu118 (installed via pip)
CUDA 11.8

# Selected wheel
mmcv-2.1.0+cu11torch2.5.1cxx11abiFALSE-cp311-cp311-linux_x86_64.whl
```

#### Example 2: Newer CUDA Environment
```bash
# Environment info
Python 3.12.0
PyTorch 2.7.1+cu128 (installed via pip)
CUDA 12.8

# Selected wheel
mmcv-2.1.0+cu12torch2.7.1cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
```

#### Example 3: NVIDIA NGC Container
```bash
# Environment info
Docker container: nvcr.io/nvidia/pytorch:24.10-py3
Python 3.10 (in container)
PyTorch 2.x (pre-installed in container with C++11 ABI enabled)

# Selected wheel (note: cxx11abiTRUE)
mmcv-2.1.0+cu12torch{matching_version}cxx11abiTRUE-cp310-cp310-linux_x86_64.whl
```

### Troubleshooting

#### Issue 1: Import Error "undefined symbol"
```
ImportError: .../mmcv/_ext.cpython-312-x86_64-linux-gnu.so: undefined symbol: _ZN...
```
**Cause**: C++11 ABI mismatch  
**Solution**:
- If you're using `cxx11abiFALSE`, try the `cxx11abiTRUE` version
- Vice versa

#### Issue 2: CUDA Version Mismatch
```
RuntimeError: CUDA version mismatch
```
**Cause**: Wheel's CUDA version doesn't match PyTorch's CUDA version  
**Solution**:
- Check PyTorch's CUDA version: `python -c "import torch; print(torch.version.cuda)"`
- Download the matching CUDA version wheel (cu11 or cu12)

#### Issue 3: Cannot Find Suitable Wheel
**Solutions**:
- Check [all Releases](https://github.com/biubushy/mmcv-wheels/releases) for the matching version
- If your version combination is not available, submit an [Issue](https://github.com/biubushy/mmcv-wheels/issues)
- Or refer to the official [MMCV installation guide](https://mmcv.readthedocs.io/en/latest/get_started/installation.html) to compile from source

### Trigger Build (For Maintainers)

1. Determine the MMCV version to build (e.g., `2.1.0`)
2. Create and push the corresponding git tag:

```bash
git tag v2.1.0
git push origin v2.1.0
```

3. GitHub Actions will automatically:
   - Create a Release
   - Build wheels for all configurations
   - Upload wheels to Release

## Technical Details

### C++11 ABI Explanation

C++11 ABI is the new binary interface introduced in GCC 5. MMCV needs to use the same ABI as PyTorch:

- **Standard PyTorch wheels**: Don't use C++11 ABI (`_GLIBCXX_USE_CXX11_ABI=0`)
- **NVIDIA NGC images**: Use C++11 ABI (`_GLIBCXX_USE_CXX11_ABI=1`)

This project builds wheels for both ABIs to ensure compatibility.

### Build Optimizations

1. **Disk Space Management**:
   - Remove unnecessary tools (.NET, Android, Haskell, CodeQL)
   - Use `maximize-build-space` action to expand available space

2. **Memory Optimization**:
   - Configure 10GB swap space
   - Limit parallel compilation jobs (`MAX_JOBS=2`)

3. **Build Timeout**:
   - Set 5-hour timeout (GitHub Actions max is 6 hours)

### PyTorch CUDA Version Mapping

The script automatically maps CUDA versions to corresponding PyTorch wheels:

```python
minv = {'2.4': 118, '2.5': 118, '2.6': 118, '2.7': 118, '2.8': 126}
maxv = {'2.4': 124, '2.5': 124, '2.6': 126, '2.7': 128, '2.8': 129}
```

For example: For PyTorch 2.6, CUDA 11.x uses cu118, CUDA 12.x uses cu126.

## Limitations

- **Wheels Only**: This project only builds pre-compiled wheels, does not publish to PyPI
- **Official Package First**: Users should prefer the official mmcv package maintained by OpenMMLab
- **Testing Skipped**: Assumes OpenMMLab has thoroughly tested, skips local testing steps
- **Linux Only**: Currently only supports Linux (ubuntu-22.04) with glibc 2.35

## Notes

1. **Glibc Compatibility**: Uses ubuntu-22.04 (glibc 2.35) for better compatibility
2. **CUDA Installation**: Uses network installation mode for improved stability
3. **Setuptools Version**: Fixed to setuptools 68.0.0 to avoid CUDA version mismatch issues
4. **Build Resources**: Each build configuration requires significant disk space and time

## Related Links

- [MMCV Official Repository](https://github.com/open-mmlab/mmcv)
- [OpenMMLab Website](https://openmmlab.com/)
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch)
- [PyTorch Website](https://pytorch.org/)

## License

This project is for building and distributing MMCV wheels. For MMCV's license, please refer to its official repository.

## Contributing

To improve the build process or add support for new configurations, feel free to submit Issues or Pull Requests.

---

**Note**: This project is a build tool for MMCV wheels, not a development project for MMCV itself. MMCV development and maintenance is handled by the OpenMMLab team.

