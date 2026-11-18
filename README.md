# ClustraComp 

<a id="top"></a>

Research framework for grouping similar file chunks before lossless compression with LZ4 or Zstandard.

基于相似文件块分组的无损压缩实验框架，支持 LZ4 和 Zstandard。

<p align="center">
	<a href="#readme-中文"><img src="https://img.shields.io/badge/中文-点击切换-1f6feb?style=for-the-badge" alt="切换到中文" /></a>
	<a href="#readme-english"><img src="https://img.shields.io/badge/English-Switch-238636?style=for-the-badge" alt="Switch to English" /></a>
</p>

### NPC 2026 论文 / Paper

[Similarity-Aware Client-Side Reordering for Server-Side Storage Compression in Distributed File Systems](https://arxiv.org/abs/2611.22847)  
Yuanhong Huang, Zicong Chen, Yili Ma, Jinwu Yang, Yulong Chen, Guangming Tan, Dingwen Tao  
[NPC 2026](https://npc-conference.github.io/accepted-papers/) · [arXiv:2611.22847](https://arxiv.org/abs/2611.22847)

<details>
<summary>BibTeX</summary>

```bibtex
@misc{huang2026similarityaware,
  title = {Similarity-Aware Client-Side Reordering for Server-Side Storage Compression in Distributed File Systems},
  author = {Yuanhong Huang and Zicong Chen and Yili Ma and Jinwu Yang and Yulong Chen and Guangming Tan and Dingwen Tao},
  year = {2026},
  eprint = {2611.22847},
  archivePrefix = {arXiv},
  primaryClass = {cs.DC},
  note = {Accepted at NPC 2026},
  url = {https://arxiv.org/abs/2611.22847}
}
```

</details>

---

<a id="readme-中文"></a>

## 中文说明

### 开发者信息

- 开发者：黄远鸿，杨锦武
- 指导老师：陶鼎文，谭光明

### 项目简介

ClustraComp 是用 C 实现的文件块聚类压缩实验框架。它将 Odess 和 Palantir 的块相似性检测方法用于分组无损压缩，并与按文件顺序分组的基线比较压缩比和运行时间。

NPC 2026 [论文](https://arxiv.org/abs/2611.22847)在 SkyFS 客户端对相似数据块重排，由服务端的 OpenZFS 使用 Zstandard 压缩。与按逻辑顺序写入相比，实验中物理存储占用最多减少 42.9%。

核心流程分为两阶段：

1. 特征提取与分组
2. 分组压缩与结果分析

### 方法来源与实现

- **Odess（IEEE ICDE 2021）**通过基于内容的采样提取用于检测块相似性的特征。Xiangyu Zou 等，[Odess: Speeding up Resemblance Detection for Redundancy Elimination by Fast Content-Defined Sampling](https://doi.org/10.1109/ICDE51399.2021.00048)。
- **Palantir（ACM ASPLOS 2024）**使用分层超特征匹配不同相似程度的数据块。Hongming Huang 等，[Palantir: Hierarchical Similarity Detection for Post-Deduplication Delta Compression](https://doi.org/10.1145/3620665.3640353)。

ClustraComp 的两种模式共用 AVX2 向量化的子特征提取实现。Palantir 模式在此基础上构建三层超特征。框架根据匹配结果将相似块收集到同一组，使用 LZ4 或 Zstandard 压缩每组，并通过 OpenMP 并行处理各组。

---

### 主要特性

- 支持两种分组策略：Odess / Palantir
- 支持两种压缩后端：LZ4 / ZSTD
- 支持顺序基线压缩对照
- 支持 OpenMP 并行执行
- 支持 CPU 核心绑定（特征提取 / 聚类分组阶段可分别绑核）
- 提供批量自动测试脚本与配置文件驱动

---

### 项目结构

~~~text
.
├── main.c                 # 程序入口，解析参数并串联两阶段流程
├── src/                   # 源代码目录
│   ├── core_types.c       # 核心数据结构、块模型、特征提取
│   ├── indexing.c         # Odess/Palantir 索引与分组查询
│   ├── compression.c      # 分组压缩、顺序压缩、统计分析
│   ├── pipeline.c         # 端到端流程编排
│   ├── feature_api.c      # 特征提取 API 模块
│   └── grouping_api.c     # 聚类分组 API 模块
├── include/               # 头文件目录
│   ├── core_types.h
│   ├── indexing.h
│   ├── compression.h
│   ├── pipeline.h
│   ├── feature_api.h
│   ├── grouping_api.h
│   ├── gear.h             # 预计算 Gear 哈希表
│   └── verstable.h        # 第三方哈希表实现
├── build/                 # 编译中间产物（.o 文件）
├── lib/                   # 静态库输出
│   ├── libfeature_api.a   # 特征提取静态库
│   └── libgrouping_api.a  # 聚类分组静态库
├── auto_test.sh           # 批量自动测试脚本
├── test_config.cfg        # 测试配置文件
└── Makefile               # 构建脚本
~~~

---

### 环境要求

- Linux（推荐）
- GCC（支持 C11、AVX2、OpenMP）
- LZ4 开发库
- ZSTD 开发库

示例（Debian/Ubuntu）：

~~~bash
sudo apt-get update
sudo apt-get install -y build-essential liblz4-dev libzstd-dev
~~~

---

### 编译

在项目根目录执行：

~~~bash
make clean && make -j
~~~

默认生成：

- 可执行文件：`ClustraComp`
- 静态库：`lib/libfeature_api.a`、`lib/libgrouping_api.a`

---

### 运行

命令格式：

~~~bash
./ClustraComp [输入文件] -c 块大小 -m 方法 --feature-core 核心ID --group-core 核心ID
~~~

参数说明：

| 参数 | 说明 | 默认值 |
|------|------|--------|
| 输入文件 | 待测试文件路径；如果省略，会自动生成 test_data.bin | - |
| `-c` | 块大小（正整数，字节） | 8192 |
| `-m` | 方法选择：`odess` / `palantir`；不指定则根据文件类型自动选择（`.ibd` → Odess，其它 → Palantir） | 自动 |
| `--feature-core` | 绑定特征提取阶段到指定 CPU 核心（非负整数） | 不绑定 |
| `--group-core` | 绑定分组阶段到指定 CPU 核心（非负整数） | 不绑定 |

示例：

~~~bash
./ClustraComp data/sample.bin -c 8192 -m odess
./ClustraComp data/sample.bin -c 4096 --feature-core 0 --group-core 1
./ClustraComp -m palantir
~~~

---

### 自动化批量测试

脚本与配置：

- `auto_test.sh` — 批量测试脚本
- `test_config.cfg` — 测试配置文件

用法：

~~~bash
./auto_test.sh [--config <配置文件>]
~~~

配置文件格式（`KEY=VALUE`，`#` 开头为注释）：

~~~ini
# 数据目录
DATA_DIR=./data
# 输出目录
OUTPUT_DIR=./logs
# 块大小（字节）
CHUNK_SIZE=8192
# 可执行文件路径
EXECUTABLE=./ClustraComp
# 特征提取绑核（-1 表示不绑定）
FEATURE_CORE=-1
# 聚类分组绑核（-1 表示不绑定）
GROUP_CORE=-1
~~~

示例：

~~~bash
# 使用默认配置文件 test_config.cfg
./auto_test.sh

# 指定自定义配置文件
./auto_test.sh --config ./my_config.cfg
~~~

说明：

- 算法由程序内部根据文件类型自动选择（`.ibd` → Odess，其它 → Palantir）
- `FEATURE_CORE` / `GROUP_CORE` 设为非负整数时绑定对应 CPU 核心，`-1` 表示不绑定
- 日志输出到配置的输出目录中
- 单文件超时阈值为 8 小时

---

### 输出与结果解读

程序会输出：

- 特征提取与分组阶段耗时
- 顺序基线压缩比（LZ4 / ZSTD）
- 聚类分组压缩比（LZ4 / ZSTD）
- 分组分布统计（组数、最大组、平均组等）
- 分策略对比与提升倍率

可用于评估：

- 不同块大小对压缩收益的影响
- 不同分组策略对压缩率和速度的影响
- 基线方案与聚类方案的收益差异

---

### 许可与第三方依赖

- 本项目使用第三方头文件 `verstable.h`（Verstable）作为哈希结构实现，位于 `include/` 目录。
- 使用时请遵循对应第三方依赖的许可证要求。

[返回顶部 / Back to Top](#top)

---

<a id="readme-english"></a>

## English

### Developer Information

- Developers: Yuanhong Huang, Jinwu Yang
- Advisors: Dingwen Tao, Guangming Tan

### Overview

ClustraComp is a C-based framework for experiments with clustered file compression. It adapts chunk similarity detection from Odess and Palantir to grouped lossless compression and measures compression ratio and runtime against a baseline that groups chunks in file order.

The [NPC 2026 paper](https://arxiv.org/abs/2611.22847) evaluates client-side reordering in SkyFS with Zstandard compression on the OpenZFS storage servers. The reordered writes used up to 42.9% less physical storage than writes in logical order.

The workflow has two stages:

1. Feature extraction and grouping
2. Grouped compression and result analysis

### Prior Work and Implementation

- **Odess (IEEE ICDE 2021)** uses content-defined sampling to extract features for detecting similar chunks. Xiangyu Zou et al., [Odess: Speeding up Resemblance Detection for Redundancy Elimination by Fast Content-Defined Sampling](https://doi.org/10.1109/ICDE51399.2021.00048).
- **Palantir (ACM ASPLOS 2024)** uses hierarchical super-features to match chunks at different degrees of similarity. Hongming Huang et al., [Palantir: Hierarchical Similarity Detection for Post-Deduplication Delta Compression](https://doi.org/10.1145/3620665.3640353).

Both ClustraComp modes share an AVX2-vectorized subfeature extraction implementation. The Palantir mode builds three levels of super-features from those subfeatures. ClustraComp groups similar chunks using the matches and compresses each group with LZ4 or Zstandard, processing groups in parallel with OpenMP.

---

### Key Features

- Two grouping strategies: Odess / Palantir
- Two compression backends: LZ4 / ZSTD
- Sequential baseline compression for comparison
- OpenMP parallel execution
- CPU core pinning support (separate for feature extraction and grouping stages)
- Batch automation script driven by configuration file

---

### Project Layout

~~~text
.
├── main.c                 # Program entry and argument parsing
├── src/                   # Source code
│   ├── core_types.c       # Core containers, chunk model, feature extraction
│   ├── indexing.c         # Group indexing for Odess/Palantir
│   ├── compression.c      # Grouped compression, sequential baseline, analytics
│   ├── pipeline.c         # End-to-end orchestration
│   ├── feature_api.c      # Feature extraction API module
│   └── grouping_api.c     # Grouping API module
├── include/               # Header files
│   ├── core_types.h
│   ├── indexing.h
│   ├── compression.h
│   ├── pipeline.h
│   ├── feature_api.h
│   ├── grouping_api.h
│   ├── gear.h             # Precomputed gear hash tables
│   └── verstable.h        # Third-party hash table implementation
├── build/                 # Build artifacts (.o files)
├── lib/                   # Static library output
│   ├── libfeature_api.a   # Feature extraction static library
│   └── libgrouping_api.a  # Grouping static library
├── auto_test.sh           # Batch testing script
├── test_config.cfg        # Test configuration file
└── Makefile               # Build script
~~~

---

### Requirements

- Linux (recommended)
- GCC with C11, AVX2, and OpenMP support
- LZ4 development library
- ZSTD development library

Example (Debian/Ubuntu):

~~~bash
sudo apt-get update
sudo apt-get install -y build-essential liblz4-dev libzstd-dev
~~~

---

### Build

Run in the project root:

~~~bash
make clean && make -j
~~~

Default outputs:

- Executable: `ClustraComp`
- Static libraries: `lib/libfeature_api.a`, `lib/libgrouping_api.a`

---

### Run

Command format:

~~~bash
./ClustraComp [input_file] -c chunk_size -m method --feature-core CORE_ID --group-core CORE_ID
~~~

Arguments:

| Argument | Description | Default |
|----------|-------------|---------|
| input_file | Input path; if omitted, test_data.bin is auto-generated | - |
| `-c` | Chunk size in bytes (positive integer) | 8192 |
| `-m` | Method: `odess` / `palantir`; if omitted, auto-selected by file type (`.ibd` → Odess, others → Palantir) | Auto |
| `--feature-core` | Pin feature extraction stage to specified CPU core (non-negative integer) | Unpinned |
| `--group-core` | Pin grouping stage to specified CPU core (non-negative integer) | Unpinned |

Examples:

~~~bash
./ClustraComp data/sample.bin -c 8192 -m odess
./ClustraComp data/sample.bin -c 4096 --feature-core 0 --group-core 1
./ClustraComp -m palantir
~~~

---

### Batch Testing

Script and configuration:

- `auto_test.sh` — Batch testing script
- `test_config.cfg` — Test configuration file

Usage:

~~~bash
./auto_test.sh [--config <config_file>]
~~~

Configuration file format (`KEY=VALUE`, `#` for comments):

~~~ini
# Data directory
DATA_DIR=./data
# Output directory
OUTPUT_DIR=./logs
# Chunk size (bytes)
CHUNK_SIZE=8192
# Executable path
EXECUTABLE=./ClustraComp
# Feature extraction core pinning (-1 = unpinned)
FEATURE_CORE=-1
# Grouping core pinning (-1 = unpinned)
GROUP_CORE=-1
~~~

Examples:

~~~bash
# Use default config file test_config.cfg
./auto_test.sh

# Specify a custom config file
./auto_test.sh --config ./my_config.cfg
~~~

Notes:

- Method is auto-selected by file type (`.ibd` → Odess, others → Palantir)
- `FEATURE_CORE` / `GROUP_CORE` set to a non-negative integer to pin to that CPU core; `-1` means unpinned
- Per-file logs are written to the configured output directory
- Per-file timeout is 8 hours

---

### Outputs and Evaluation

The program reports:

- Feature extraction and grouping time
- Sequential baseline ratios (LZ4 / ZSTD)
- Grouped compression ratios (LZ4 / ZSTD)
- Group distribution statistics (group count, max/avg size, etc.)
- Improvement comparison across strategies

This helps evaluate:

- Impact of chunk size on compression gain
- Impact of grouping strategy on ratio and throughput
- Gain difference between baseline and clustered workflows

---

### License and Third-Party Components

- The project includes `verstable.h` (Verstable) as a third-party hash table implementation, located in `include/`.
- Please follow the corresponding third-party license terms when distributing or integrating.

[Back to Top / 返回顶部](#top)
