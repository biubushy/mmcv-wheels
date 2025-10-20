# MMCV Wheels 构建项目

[English](./README_EN.md) | 简体中文

## 项目简介

本项目提供了一个自动化构建系统，用于为 MMCV（OpenMMLab 计算机视觉基础库）构建不同配置的 Python wheels。通过 GitHub Actions 工作流，可以自动编译和发布针对不同 Python 版本、CUDA 版本、PyTorch 版本和 C++11 ABI 设置的预编译 wheels。

## 功能特性

- **多版本支持**：支持 Python 3.9、3.10、3.11、3.12
- **CUDA 兼容性**：支持 CUDA 11.8.0 和 12.9.1
- **PyTorch 版本**：支持 PyTorch 2.4.0、2.5.1、2.6.0、2.7.1
- **C++11 ABI**：同时构建启用和禁用 C++11 ABI 的版本
- **NGC 容器支持**：支持从 NVIDIA NGC PyTorch 容器镜像构建
- **自动化发布**：自动上传构建的 wheels 到 GitHub Release
- **优化构建环境**：包含磁盘空间优化和 swap 空间配置

## 项目结构

```
mmcv-wheels/
├── .github/
│   ├── workflows/
│   │   ├── _build.yml              # 标准构建工作流模板
│   │   ├── _build_in_container.yml # 容器内构建工作流模板
│   │   └── publish.yaml            # 发布工作流（主入口）
│   └── scripts/
│       ├── build.sh                # 构建脚本
│       └── check_for_ngc_images.sh # NGC 镜像检测脚本
└── README.md
```

## 工作流说明

### 1. 发布工作流 (`publish.yaml`)

主工作流，在推送标签时触发（格式：`v*`）。包含以下步骤：

- **创建 Release**：基于 git 标签自动创建 GitHub Release
- **构建标准 Wheels**：为所有配置矩阵构建标准 wheels
- **构建 NGC Wheels**：为 NVIDIA NGC PyTorch 容器构建 wheels

**构建矩阵配置**：
- 操作系统：ubuntu-22.04
- Python 版本：3.9, 3.10, 3.11, 3.12
- PyTorch 版本：2.4.0, 2.5.1, 2.6.0, 2.7.1
- CUDA 版本：11.8.0, 12.9.1
- C++11 ABI：FALSE, TRUE

### 2. 标准构建工作流 (`_build.yml`)

在 GitHub Actions runner 上直接构建：

1. 设置 Python 环境
2. 安装 CUDA Toolkit
3. 安装对应的 PyTorch 版本
4. 编译 MMCV wheel
5. 上传到 GitHub Release

**关键特性**：
- 磁盘空间优化（删除不需要的工具）
- 配置 10GB swap 空间
- 网络模式安装 CUDA（提高稳定性）

### 3. 容器构建工作流 (`_build_in_container.yml`)

在 Docker 容器内构建（用于 NGC 镜像）：

1. 最大化构建空间
2. 拉取 NGC PyTorch 容器镜像
3. 在容器内提取环境配置
4. 在容器内编译 MMCV wheel
5. 上传到 GitHub Release

**优势**：
- 使用 NVIDIA 官方优化的 PyTorch 镜像
- 自动匹配容器内的 CUDA 和 PyTorch 版本
- 与 NGC 容器环境完全兼容

## 构建脚本

### build.sh

核心构建脚本，执行以下操作：

1. 安装构建依赖（setuptools、ninja、packaging、wheel）
2. 根据 `CXX11_ABI` 环境变量设置编译标志
3. 从 OpenMMLab 官方源构建 MMCV wheel
4. 重命名 wheel 文件以包含 CUDA、PyTorch 版本和 ABI 信息

**环境变量**：
- `MMCV_VERSION`：要构建的 MMCV 版本
- `WHEEL_CUDA_VERSION`：CUDA 主版本号（如：12）
- `MATRIX_TORCH_VERSION`：PyTorch 版本（如：2.6）
- `TORCH_CUDA_VERSION`：完整 CUDA 版本（如：126）
- `CXX11_ABI`：是否启用 C++11 ABI（TRUE/FALSE）
- `MATRIX_PYTHON_VERSION`：Python 版本（如：312）

**Wheel 命名格式**：
```
mmcv-{version}+cu{cuda_version}torch{torch_version}cxx11abi{abi}-{platform}.whl
```

例如：`mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl`

### check_for_ngc_images.sh

检测最近 7 个月的 NVIDIA NGC PyTorch 镜像是否存在：

1. 生成过去 7 个月的镜像标签（格式：`YY.MM-py3`）
2. 使用 `docker manifest inspect` 检查镜像是否存在
3. 输出 JSON 格式的可用镜像列表

**输出示例**：
```json
["nvcr.io/nvidia/pytorch:24.10-py3", "nvcr.io/nvidia/pytorch:24.09-py3"]
```

## 使用方法

### 快速安装指南

#### 第一步：检查您的环境

在安装前，请先确认您的环境配置：

```bash
# 查看 Python 版本
python --version
# 输出示例：Python 3.12.0

# 查看 PyTorch 和 CUDA 版本
python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA: {torch.version.cuda}')"
# 输出示例：
# PyTorch: 2.6.0+cu126
# CUDA: 12.6

# 查看 C++11 ABI 设置（如果已安装 PyTorch）
python -c "import torch; print(f'C++11 ABI: {torch._C._GLIBCXX_USE_CXX11_ABI}')"
# 输出示例：C++11 ABI: False
```

#### 第二步：选择正确的 Wheel 文件

访问 [Releases 页面](https://github.com/biubushy/mmcv-wheels/releases)，根据您的环境选择对应的 wheel 文件。

**Wheel 文件命名规则**：
```
mmcv-{版本号}+cu{CUDA版本}torch{PyTorch版本}cxx11abi{ABI设置}-cp{Python版本}-cp{Python版本}-linux_x86_64.whl
```

**示例文件名解析**：
```
mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
│         │  │  │         │            │    │
│         │  │  │         │            │    └─ Python 3.12
│         │  │  │         │            └────── Python 3.12
│         │  │  │         └─────────────────── C++11 ABI 禁用
│         │  │  └───────────────────────────── PyTorch 2.6.0
│         │  └──────────────────────────────── CUDA 12.x
│         └─────────────────────────────────── MMCV 版本 2.1.0
└───────────────────────────────────────────── 包名
```

**选择标准**：

1. **Python 版本匹配**
   - `cp39` → Python 3.9
   - `cp310` → Python 3.10
   - `cp311` → Python 3.11
   - `cp312` → Python 3.12

2. **CUDA 版本匹配**
   - 如果您的 CUDA 是 11.x（11.0-11.8），选择 `cu11` 开头的文件
   - 如果您的 CUDA 是 12.x（12.0-12.9），选择 `cu12` 开头的文件
   - **提示**：PyTorch 的 CUDA 版本（如 `cu126`）中的数字对应 CUDA 12.6

3. **PyTorch 版本匹配**
   - 选择与您安装的 PyTorch **主版本和次版本**相同的文件
   - 例如：PyTorch 2.6.0、2.6.1 都选择 `torch2.6.0` 的文件
   - 支持的版本：2.4.0、2.5.1、2.6.0、2.7.1

4. **C++11 ABI 匹配**
   - **绝大多数情况**：选择 `cxx11abiFALSE`（适用于通过 pip 安装的标准 PyTorch）
   - **特殊情况**：仅在以下情况选择 `cxx11abiTRUE`
     - 使用 NVIDIA NGC PyTorch 容器
     - 从源码编译的 PyTorch，且编译时启用了 C++11 ABI
   - **如何验证**：运行上面的检查命令，如果输出 `False`，选择 `cxx11abiFALSE`

#### 第三步：下载并安装

**完整示例（环境：Python 3.12 + PyTorch 2.6.0 + CUDA 12.6）**：

```bash
# 1. 下载 wheel 文件
wget https://github.com/biubushy/mmcv-wheels/releases/download/v2.1.0/mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl

# 2. 安装（使用完整文件名）
pip install mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl

# 3. 验证安装
python -c "import mmcv; print(mmcv.__version__)"
```

**或者直接使用 URL 安装**：

```bash
pip install https://github.com/biubushy/mmcv-wheels/releases/download/v2.1.0/mmcv-2.1.0+cu12torch2.6.0cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
```

### 常见环境配置示例

#### 示例 1：标准 PyTorch 环境
```bash
# 环境信息
Python 3.11.0
PyTorch 2.5.1+cu118 (通过 pip 安装)
CUDA 11.8

# 选择的 wheel
mmcv-2.1.0+cu11torch2.5.1cxx11abiFALSE-cp311-cp311-linux_x86_64.whl
```

#### 示例 2：新版 CUDA 环境
```bash
# 环境信息
Python 3.12.0
PyTorch 2.7.1+cu128 (通过 pip 安装)
CUDA 12.8

# 选择的 wheel
mmcv-2.1.0+cu12torch2.7.1cxx11abiFALSE-cp312-cp312-linux_x86_64.whl
```

#### 示例 3：NVIDIA NGC 容器
```bash
# 环境信息
Docker 容器：nvcr.io/nvidia/pytorch:24.10-py3
Python 3.10（容器内）
PyTorch 2.x（容器内预装，启用 C++11 ABI）

# 选择的 wheel（注意：cxx11abiTRUE）
mmcv-2.1.0+cu12torch{对应版本}cxx11abiTRUE-cp310-cp310-linux_x86_64.whl
```

### 故障排除

#### 问题 1：导入错误 "undefined symbol"
```
ImportError: .../mmcv/_ext.cpython-312-x86_64-linux-gnu.so: undefined symbol: _ZN...
```
**原因**：C++11 ABI 不匹配  
**解决方案**：
- 如果您使用的是 `cxx11abiFALSE`，尝试 `cxx11abiTRUE` 版本
- 反之亦然

#### 问题 2：CUDA 版本不匹配
```
RuntimeError: CUDA version mismatch
```
**原因**：wheel 的 CUDA 版本与 PyTorch 的 CUDA 版本不匹配  
**解决方案**：
- 检查 PyTorch 的 CUDA 版本：`python -c "import torch; print(torch.version.cuda)"`
- 下载对应 CUDA 版本的 wheel（cu11 或 cu12）

#### 问题 3：找不到合适的 wheel
**解决方案**：
- 查看 [所有 Releases](https://github.com/biubushy/mmcv-wheels/releases) 找到对应版本
- 如果没有您需要的版本组合，请提交 [Issue](https://github.com/biubushy/mmcv-wheels/issues)
- 或者参考官方 [MMCV 安装文档](https://mmcv.readthedocs.io/en/latest/get_started/installation.html)从源码编译

### 触发构建（维护者）

1. 确定要构建的 MMCV 版本（如 `2.1.0`）
2. 创建并推送对应的 git 标签：

```bash
git tag v2.1.0
git push origin v2.1.0
```

3. GitHub Actions 将自动：
   - 创建 Release
   - 构建所有配置的 wheels
   - 上传 wheels 到 Release

## 技术细节

### C++11 ABI 说明

C++11 ABI 是 GCC 5 引入的新二进制接口。MMCV 需要与 PyTorch 使用相同的 ABI：

- **标准 PyTorch wheels**：不使用 C++11 ABI（`_GLIBCXX_USE_CXX11_ABI=0`）
- **NVIDIA NGC 镜像**：使用 C++11 ABI（`_GLIBCXX_USE_CXX11_ABI=1`）

本项目为两种 ABI 都构建 wheels 以确保兼容性。

### 构建优化

1. **磁盘空间管理**：
   - 删除 .NET、Android、Haskell、CodeQL 等不需要的工具
   - 使用 `maximize-build-space` action 扩展可用空间

2. **内存优化**：
   - 配置 10GB swap 空间
   - 限制并行编译任务数（`MAX_JOBS=2`）

3. **构建超时**：
   - 设置 5 小时超时（GitHub Actions 最大允许 6 小时）

### PyTorch CUDA 版本映射

脚本会自动将 CUDA 版本映射到对应的 PyTorch wheel：

```python
minv = {'2.4': 118, '2.5': 118, '2.6': 118, '2.7': 118, '2.8': 126}
maxv = {'2.4': 124, '2.5': 124, '2.6': 126, '2.7': 128, '2.8': 129}
```

例如：对于 PyTorch 2.6，CUDA 11.x 使用 cu118，CUDA 12.x 使用 cu126。

## 限制说明

- **仅构建 Wheels**：本项目仅构建预编译的 wheels，不发布到 PyPI
- **官方包优先**：用户应优先使用 OpenMMLab 官方维护的 mmcv 包
- **测试跳过**：假设 OpenMMLab 已充分测试，跳过本地测试步骤
- **Linux Only**：当前仅支持 Linux（ubuntu-22.04），使用 glibc 2.35

## 注意事项

1. **Glibc 兼容性**：使用 ubuntu-22.04（glibc 2.35）以获得更好的兼容性
2. **CUDA 安装方式**：使用网络安装模式（network method）以提高稳定性
3. **Setuptools 版本**：固定使用 setuptools 68.0.0 以避免 CUDA 版本不匹配问题
4. **构建资源**：每个构建配置需要大量磁盘空间和时间

## 相关链接

- [MMCV 官方仓库](https://github.com/open-mmlab/mmcv)
- [OpenMMLab 官网](https://openmmlab.com/)
- [NVIDIA NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch)
- [PyTorch 官网](https://pytorch.org/)

## 许可证

本项目用于构建和分发 MMCV wheels。MMCV 的许可证请参考其官方仓库。

## 贡献

如需改进构建流程或添加新的配置支持，欢迎提交 Issue 或 Pull Request。

---

**注意**：本项目是 MMCV wheels 的构建工具，不是 MMCV 本身的开发项目。MMCV 的开发和维护由 OpenMMLab 团队负责。

