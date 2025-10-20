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

### Trigger Build

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

### Download Wheels

Download the wheel file for your configuration from the GitHub Releases page, then install:

```bash
pip install mmcv-{version}+cu{cuda}torch{torch}cxx11abi{abi}-{platform}.whl
```

### Select the Right Wheel

Choose the wheel based on your environment:

1. **Python Version**: Match your Python version (3.9/3.10/3.11/3.12)
2. **CUDA Version**: Match your CUDA installation (11.8 or 12.x)
3. **PyTorch Version**: Match your installed PyTorch version
4. **C++11 ABI**:
   - `FALSE`: For standard PyTorch wheels (recommended)
   - `TRUE`: For NVIDIA NGC containers or PyTorch compiled from source

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

