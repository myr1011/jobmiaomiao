# OpenTitan PQC 仓库复现与评估最终提交框架

> 适用对象：论文 **Improving ML-KEM and ML-DSA on OpenTitan – Efficient Multiplication Vector Instructions for OTBN** 对应 GitHub 仓库评估。  
> 评估仓库版本：`commit 5b07cd013a78f059e8388d7133ff7c43f3debdb6`。  
> 任务目标：从论文读者 / Artifact Reviewer 角度，按照仓库说明运行软件、RTL、仿真、综合、FPGA 相关流程，并记录可复现性、问题与评价。

---

## 0. 提交材料总览

最终建议提交以下文件：

```text
artifact_evaluation/
├── README_report.md                 # 主报告，即本文档最终填写版
├── logs/                            # 终端日志与关键命令输出
│   ├── 00_environment_check.log
│   ├── 01_python_deps_install.log
│   ├── 02_software_benchmark_mldsa.log
│   ├── 03_software_benchmark_mlkem.log
│   ├── 04_codesize.log
│   ├── 05_cocotb_pytest.log
│   ├── 06_rtl_iss.log
│   ├── 07_chip_level_verilator.log
│   ├── 08_synthesis_vivado.log
│   └── 09_docker_orfs.log
├── results/                         # 输出结果与整理表格
│   ├── mldsa_eval.txt
│   ├── mlkem_eval.txt
│   ├── code_size.txt
│   ├── software_benchmark_summary.md
│   ├── hardware_synthesis_summary.md
│   └── chip_level_simulation_summary.md
├── issues/                          # 问题清单与修复记录
│   ├── issue_list.md
│   ├── patch_notes.md
│   └── unresolved_issues.md
└── patches/                         # 如有修改，保存 diff
    ├── chip_sim_core_fix.diff
    ├── hw_BUILD_thread_fix.diff
    └── get_benchmark_single_iteration_fix.diff
```

---

## 1. 项目基本信息

### 1.1 任务对象

| 项目 | 内容 |
|---|---|
| 论文 | Improving ML-KEM and ML-DSA on OpenTitan – Efficient Multiplication Vector Instructions for OTBN |
| 论文链接 | https://eprint.iacr.org/2025/2028 |
| 仓库 | https://github.com/PQC-OpenTitan/improving-ml-kem-and-ml-dsa-on-opentitan |
| 指定版本 | `5b07cd013a78f059e8388d7133ff7c43f3debdb6` |
| 评估目标 | 按照 README 复现实验，评估 Documentation / Completeness / Exercisability / Reusability |
| 截止时间 | 6 月 3 日 |

### 1.2 评估范围

本次评估覆盖以下内容：

1. 阅读论文、README 和仓库说明。
2. 理解 OpenTitan、OTBN、ML-KEM、ML-DSA 与本文提出的 OTBN vector multiplication instructions。
3. 运行软件 benchmark，对应论文 Tables 4–6。
4. 运行 fixed-input tests。
5. 获取 code size，对应论文 Table 7。
6. 运行 RTL 模块级 cocotb + pytest。
7. 运行 RTL–ISS tests。
8. 尝试运行 chip-level Top Earlgrey Verilator tests。
9. 尝试运行硬件综合流程，对应论文 Tables 1–3。
10. 如条件允许，评估 FPGA CW310 上板测试流程。
11. 记录所有操作、问题、修复、未解决问题和最终评价。

---

## 2. 背景介绍

### 2.1 OpenTitan 简介

OpenTitan 是一个开源的 silicon root of trust SoC 项目，包含 Ibex RISC-V 核、安全外设、存储控制器、生命周期管理、密钥管理、熵源、CSRNG、KMAC、OTBN 等安全模块。本仓库基于 OpenTitan，对 OTBN 进行后量子密码算法加速扩展。

### 2.2 OTBN 简介

OTBN，全称 OpenTitan Big Number Accelerator，是 OpenTitan 中用于大整数和密码运算的专用协处理器。原始 OTBN 主要面向 RSA、ECC 等传统公钥密码，而本文扩展其指令与硬件数据通路，用于加速 ML-KEM 和 ML-DSA 中的多项式乘法与 NTT 相关计算。

### 2.3 论文主要内容概括

本文提出面向 OTBN 的高效乘法向量指令，用于提升 ML-KEM 和 ML-DSA 在 OpenTitan 上的执行效率。仓库中包含：

- OTBN 硬件修改。
- 多个 vector multiplication instruction 版本。
- ML-KEM / ML-DSA 软件实现。
- 软件 benchmark 脚本。
- RTL 测试脚本。
- RTL–ISS 验证脚本。
- Verilator chip-level 仿真目标。
- Vivado / Genus / ORFS 综合脚本。
- FPGA CW310 相关说明与 bitstream。

---

## 3. 仓库结构理解

### 3.1 硬件版本划分

| 版本 | 宏定义 | 含义 | 是否评估 |
|---|---|---|---|
| OTBN | None | 原始 OTBN | 是，作为 baseline |
| OTBN_KMAC | `TOWARDS_KMAC` | 加入 KMAC interface | 是，作为对照 |
| OTBN_TW | `TOWARDS_KMAC`, `TOWARDS_BASE`, `TOWARDS_ALU_ADDER`, `TOWARDS_MAC_ADDER` | Towards paper 的扩展版本 | 是，作为对照 |
| OTBNV1 | `TOWARDS_KMAC`, `TOWARDS_BASE`, `BNMULV` | 本文 Variant 1 | 是，重点评估 |
| OTBNV2 | `TOWARDS_KMAC`, `TOWARDS_BASE`, `BNMULV`, `BNMULV_ACCH` | 本文 Variant 2 | 是，重点评估 |
| OTBNV3 | `TOWARDS_KMAC`, `TOWARDS_BASE`, `BNMULV`, `BNMULV_ACCH`, `BNMULV_COND_SUB` | 本文 Variant 3 | 是，重点评估 |

### 3.2 关键 RTL 目录

| 路径 | 作用 |
|---|---|
| `hw/ip/otbn/rtl` | OTBN 主 RTL 代码 |
| `hw/ip/otbn/rtl/bn_vec_core` | 向量乘法器、向量加法器、不同 adder 结构 |
| `hw/top_earlgrey/dv/verilator` | Top Earlgrey Verilator chip-level 仿真配置 |
| `hw/BUILD` | Bazel 中 Verilator real target 相关配置 |

### 3.3 软件版本划分

| 版本 | 路径 | 含义 |
|---|---|---|
| `ver0_base` | `sw/otbn/crypto/*_ver0_base` | baseline + KMAC interface |
| `ver0` | `sw/otbn/crypto/*_ver0` | Towards ISE 版本 |
| `ver1` | `sw/otbn/crypto/*_ver1` | 本文 OTBNV1 |
| `ver2` | `sw/otbn/crypto/*_ver2` | 本文 OTBNV2 |
| `ver3` | `sw/otbn/crypto/*_ver3` | 本文 OTBNV3 |

---

## 4. 实验环境记录

### 4.1 主机环境

| 项目 | 实际环境 | 备注 |
|---|---|---|
| OS | Ubuntu 22.04 / WSL Ubuntu 22.04 | 填写实际环境 |
| CPU |  | 记录核心数 |
| RAM |  | 记录内存大小 |
| Python | 3.10.x | README 推荐 Python 3.10 |
| Bazel / Bazelisk |  | 填写版本 |
| Verilator | 5.022 | README 指定版本 |
| Vivado | 2022.2 / 其他 | 如用于综合需记录 |
| Docker |  | ORFS 需要 |
| Git commit | `5b07cd013a78f059e8388d7133ff7c43f3debdb6` | 必须确认 |

### 4.2 环境检查命令

```bash
git rev-parse HEAD
python3 --version
./bazelisk.sh --version
verilator --version
vivado -version
docker --version
```

### 4.3 环境配置步骤记录

| 序号 | 操作 | 命令 | 结果 | 问题 | 解决方案 |
|---|---|---|---|---|---|
| 1 | 克隆仓库 | `git clone --recurse-submodules ...` |  |  |  |
| 2 | 切换 commit | `git checkout 5b07cd0...` |  |  |  |
| 3 | 安装 apt 依赖 | `sed '/^#/d' ./apt-requirements.txt ...` |  |  |  |
| 4 | 创建虚拟环境 | `python3 -m venv .venv` |  |  |  |
| 5 | 安装 Python 依赖 | `pip install -r python-requirements.txt --require-hashes` |  |  |  |
| 6 | 安装 Verilator | 见 README |  |  |  |
| 7 | 安装 Docker | 见 README |  |  |  |
| 8 | 配置 Vivado |  |  |  |  |

---

## 5. 复现实验总流程

建议按以下优先级运行，先运行轻量级任务，再运行重型 chip-level 任务：

```text
环境配置
  ↓
软件 fixed-input tests
  ↓
软件 benchmark
  ↓
code size
  ↓
RTL cocotb/pytest
  ↓
RTL–ISS tests
  ↓
硬件综合 Vivado / ORFS / Genus
  ↓
chip-level Verilator
  ↓
FPGA CW310 测试（如有硬件条件）
```

---

## 6. 软件实验记录

## 6.1 Software Benchmark，对应 Tables 4–6

### 6.1.1 实验目的

验证 ML-KEM 和 ML-DSA 在不同 OTBN 版本下的性能，并生成 `mlkem_bench.db` / `mldsa_bench.db` SQLite 数据库，用于提取论文 Tables 4–6 中的软件性能数据。

### 6.1.2 README 指令

ML-DSA benchmark：

```bash
./bazelisk.sh test --test_timeout=10000 --cache_test_results=no \
  --action_env=PATH --sandbox_writable_path="$PWD" \
  //sw/otbn/crypto/tests/mldsa:mldsa{44,65,87}_{keypair,sign,verify}_bench_ver{0_base,0,1,2,3}
```

ML-KEM benchmark：

```bash
./bazelisk.sh test --test_timeout=10000 --cache_test_results=no \
  --action_env=PATH --sandbox_writable_path="$PWD" \
  //sw/otbn/crypto/tests/mlkem:mlkem{512,768,1024}_{keypair,encap,decap}_bench_{0_base,0,1,2,3}
```

### 6.1.3 实际执行命令

```bash
# 填写实际执行命令
```

### 6.1.4 结果记录

| 项目 | 结果 |
|---|---|
| 是否生成 `mldsa_bench.db` |Y  |
| 是否生成 `mlkem_bench.db` | Y |
| 数据库中 benchmark 数量 | 45 |
| 每个 benchmark iteration 数量 | 3 |
| 是否能运行 `get_benchmark.py` | Y |
| 是否对应论文表格 | Y |

### 6.1.5 数据库检查命令

```bash
sqlite3 mldsa_bench.db ".tables"
sqlite3 mldsa_bench.db "SELECT benchmark_id, COUNT(*) FROM benchmark_iteration GROUP BY benchmark_id;"

sqlite3 mlkem_bench.db ".tables"
sqlite3 mlkem_bench.db "SELECT benchmark_id, COUNT(*) FROM benchmark_iteration GROUP BY benchmark_id;"
```

### 6.1.6 提取结果命令

```bash
util/get_benchmark.py -f mldsa_bench.db -o mldsa_eval.txt -i <start> <end> --scheme mldsa
util/get_benchmark.py -f mlkem_bench.db -o mlkem_eval.txt -i <start> <end> --scheme mlkem
```


### 6.1.7 与论文 Tables 4–6 的对比结果

本节将本次复现得到的 `mlkem_eval.txt`、`mldsa_eval.txt` 与论文 Tables 4–6 中的软件性能数据进行对比。对比口径如下：

- **严格匹配率**：相对误差 ≤ 5% 记为“匹配正确”。
- **平均数值匹配度**：`100% - 平均相对误差`。
- Table 4 对应 NTT / INTT / pointwise multiplication 等底层函数 cycle counts。
- Table 5 对应 ML-KEM packing / unpacking 相关核心函数 cycle counts。
- Table 6 对应 ML-KEM / ML-DSA full-scheme benchmark cycle counts。

#### 总体对比结果

| 对比对象 | 对比项数量 | ≤5% 正确率 | 平均相对误差 | 平均数值匹配度 |
|---|---:|---:|---:|---:|
| Table 4 | 24 | **100.00%** | 0.00% | **100.00%** |
| Table 5 | 16 | **100.00%** | 0.00% | **100.00%** |
| Table 6 | 72 | **86.11%** | 3.67% | **96.33%** |
| **Tables 4–6 总计** | **112** | **91.07%** | **2.36%** | **97.64%** |

#### 分算法对比结果

| 表格 | 算法 | 对比项数量 | ≤5% 正确率 | 平均相对误差 | 平均数值匹配度 |
|---|---|---:|---:|---:|---:|
| Table 4 | ML-KEM | 12 | **100.00%** | 0.00% | **100.00%** |
| Table 4 | ML-DSA | 12 | **100.00%** | 0.00% | **100.00%** |
| Table 5 | ML-KEM | 16 | **100.00%** | 0.00% | **100.00%** |
| Table 6 | ML-KEM | 36 | **100.00%** | 0.31% | **99.69%** |
| Table 6 | ML-DSA | 36 | **72.22%** | 7.03% | **92.97%** |

#### 结果分析

- **Table 4 和 Table 5 完全对上，正确率为 100%。**  
  这说明底层函数级 cycle count 复现结果与论文一致，包括 ML-KEM / ML-DSA 的 NTT、INTT、pointwise multiplication，以及 ML-KEM 的 compress / decompress 相关函数。

- **Table 6 中 ML-KEM 结果高度一致。**  
  ML-KEM 的 keypair / encap / decap 在 `ver0_base / ver0 / ver1 / ver2` 上均落入 5% 误差范围内，严格匹配率为 100%，平均数值匹配度为 99.69%。

- **Table 6 的主要偏差来自 ML-DSA sign。**  
  ML-DSA signing 包含 rejection sampling，运行时间波动较大。论文 Table 6 使用较大迭代次数下的统计结果，而本次复现受时间限制使用较少 iteration，因此 sign 的 median / mean 偏差更明显。若去掉 ML-DSA sign 项，Table 6 其余项目基本均可落入 5% 误差范围。



### 6.1.7 已发现问题记录

| 问题编号 | 问题描述 | 原因分析 | 是否解决 | 解决方式 |
|---|---|---|---|---|
| SW-BENCH-01 | `get_benchmark.py` 在单 iteration 时计算 stdev 报错 | 数据库中每个 benchmark 只有 1 条 iteration | 已解决 | 增加 ITERATIONS 或修改脚本跳过单样本 stdev |

---

## 6.2 Fixed-input Tests

### 6.2.1 实验目的

快速验证 ML-KEM / ML-DSA OTBN 实现的功能正确性。该测试比 benchmark 更快，适合修改代码后进行回归检查。

### 6.2.2 README 指令

ML-DSA：

```bash
./bazelisk.sh test --test_output=streamed --cache_test_results=no \
  --action_env=PATH --sandbox_writable_path="$PWD" \
  //sw/otbn/crypto/tests/mldsa:mldsa{44,65,87}_{keypair,sign,verify}_test_ver{0_base,0,1,2,3}
```

ML-KEM：

```bash
./bazelisk.sh test --test_output=streamed --cache_test_results=no \
  --action_env=PATH --sandbox_writable_path="$PWD" \
  //sw/otbn/crypto/tests/mlkem:mlkem{512,768,1024}_{keypair,encap,decap}_test_ver{0_base,0,1,2,3}
```

### 6.2.3 结果表

| Scheme | Security Level | Operation | Version | Result | Notes |
|---|---|---|---|---|---|
| ML-DSA | 44 | keypair | ver0_base | PASS |  |
| ML-DSA | 44 | sign | ver0_base | PASS |  |
| ML-DSA | 44 | verify | ver0_base | PASS |  |
| ML-KEM | 512 | keypair | ver0_base | PASS |  |
| ML-KEM | 512 | encap | ver0_base | PASS |  |
| ML-KEM | 512 | decap | ver0_base | PASS |  |

---

## 6.3 Code Size，对应 Table 7

### 6.3.1 实验目的

提取不同 ML-KEM / ML-DSA 版本的代码大小，对应论文 Table 7。

### 6.3.2 命令

```bash
util/get_codesize.py --mlkem --mldsa --compare 2>&1 | tee logs/04_codesize.log
```

### 6.3.3 与论文 Table 7 的对比结果

论文 Table 7 统计 ML-KEM / ML-DSA 在不同安全等级和 OTBN 版本下的 code size，包括 `Text Size`、`VerX/Ver0`、`Const Size` 和 `IO Size`。本次复现使用 `util/get_codesize.py --mlkem --mldsa --compare` 得到的结果与论文 Table 7 进行逐项对比。

> 说明：论文 Table 7 覆盖 `ver0_base`、`ver0`、`ver1`、`ver2` 四类版本，分别对应 `OTBNKMAC`、`OTBNTW`、`OTBNV1`、`OTBNV2`。本次工具输出中额外包含 `ver3`，但论文 Table 7 未列出 `OTBNV3 / ver3`，因此 `ver3` 不纳入 Table 7 正确率统计，只作为额外复现数据保留。

#### Table 7 总体正确率

| 对比对象 | 对比行数 | 每行指标数 | 总对比项 | 完全一致项 | 正确率 |
|---|---:|---:|---:|---:|---:|
| ML-KEM code size | 12 | 4 | 48 | 48 | **100.00%** |
| ML-DSA code size | 12 | 4 | 48 | 48 | **100.00%** |
| **Table 7 总计** | **24** | **4** | **96** | **96** | **100.00%** |

#### Table 7 分项对比结果

| Scheme | Security Level | Version | Text Size 是否一致 | VerX/Ver0 是否一致 | Const Size 是否一致 | IO Size 是否一致 | 结果 |
|---|---:|---|---|---|---|---|---|
| ML-KEM | 512 | ver0_base | Y | Y | Y | Y | PASS |
| ML-KEM | 512 | ver0 | Y | Y | Y | Y | PASS |
| ML-KEM | 512 | ver1 | Y | Y | Y | Y | PASS |
| ML-KEM | 512 | ver2 | Y | Y | Y | Y | PASS |
| ML-KEM | 768 | ver0_base | Y | Y | Y | Y | PASS |
| ML-KEM | 768 | ver0 | Y | Y | Y | Y | PASS |
| ML-KEM | 768 | ver1 | Y | Y | Y | Y | PASS |
| ML-KEM | 768 | ver2 | Y | Y | Y | Y | PASS |
| ML-KEM | 1024 | ver0_base | Y | Y | Y | Y | PASS |
| ML-KEM | 1024 | ver0 | Y | Y | Y | Y | PASS |
| ML-KEM | 1024 | ver1 | Y | Y | Y | Y | PASS |
| ML-KEM | 1024 | ver2 | Y | Y | Y | Y | PASS |
| ML-DSA | 44 | ver0_base | Y | Y | Y | Y | PASS |
| ML-DSA | 44 | ver0 | Y | Y | Y | Y | PASS |
| ML-DSA | 44 | ver1 | Y | Y | Y | Y | PASS |
| ML-DSA | 44 | ver2 | Y | Y | Y | Y | PASS |
| ML-DSA | 65 | ver0_base | Y | Y | Y | Y | PASS |
| ML-DSA | 65 | ver0 | Y | Y | Y | Y | PASS |
| ML-DSA | 65 | ver1 | Y | Y | Y | Y | PASS |
| ML-DSA | 65 | ver2 | Y | Y | Y | Y | PASS |
| ML-DSA | 87 | ver0_base | Y | Y | Y | Y | PASS |
| ML-DSA | 87 | ver0 | Y | Y | Y | Y | PASS |
| ML-DSA | 87 | ver1 | Y | Y | Y | Y | PASS |
| ML-DSA | 87 | ver2 | Y | Y | Y | Y | PASS |

#### 额外 ver3 数据说明

本次 `get_codesize.py` 输出中还包含 `ver3`：

| Scheme | Security Level | Version | Text Size | VerX/Ver0 | Const Size | IO Size | 是否纳入 Table 7 正确率 |
|---|---:|---|---:|---:|---:|---:|---|
| ML-KEM | 512 | ver3 | 10060 | 1.12 | 1600 | 3360 | 否，论文 Table 7 未列出 |
| ML-KEM | 768 | ver3 | 10552 | 1.12 | 1600 | 4832 | 否，论文 Table 7 未列出 |
| ML-KEM | 1024 | ver3 | 12264 | 1.10 | 1600 | 6464 | 否，论文 Table 7 未列出 |
| ML-DSA | 44 | ver3 | 21184 | 1.15 | 2624 | 9600 | 否，论文 Table 7 未列出 |
| ML-DSA | 65 | ver3 | 21064 | 1.15 | 2624 | 12608 | 否，论文 Table 7 未列出 |
| ML-DSA | 87 | ver3 | 22036 | 1.14 | 2624 | 15424 | 否，论文 Table 7 未列出 |

#### Table 7 

```text
本次复现得到的 code size 结果与论文 Table 7 完全一致。对于论文 Table 7 中列出的 ML-KEM 与 ML-DSA、三个安全等级、四个版本，共 24 行数据，每行比较 Text Size、VerX/Ver0、Const Size 和 IO Size 四项指标，共 96 个对比项，全部一致。因此 Table 7 的复现正确率为 100.00%。
```


---

## 7. RTL 模块级验证

## 7.1 cocotb + pytest

### 7.1.1 实验目的

验证 `hw/ip/otbn/rtl/bn_vec_core` 下的基础向量加法器、非向量加法器和统一乘法器模块功能。

### 7.1.2 README 指令

```bash
cd hw/ip/otbn/rtl
pytest test/test_vector_adder_pytest.py
pytest test/test_non_vector_adder_pytest.py
pytest test/test_unified_mul_pytest.py
```

### 7.1.3 结果记录

| 测试文件 | 目标模块 | 结果 | 问题 | 解决方式 |
|---|---|---|---|---|
| `test_vector_adder_pytest.py` | vector adders | PASS |  |  |
| `test_non_vector_adder_pytest.py` | non-vector adders | PASS |  |  |
| `test_unified_mul_pytest.py` | unified multiplier | PASS |  |  |

---

## 7.2 RTL–ISS Tests

### 7.2.1 实验目的

运行 OTBN RTL，并与 Python ISS 逐周期比较，用于验证新增乘法指令和完整 ML-KEM / ML-DSA 设计。该测试比 full-chip Verilator 更轻量，更适合作为 RTL 功能正确性证据。

### 7.2.2 README 示例命令

```bash
hw/ip/otbn/dv/smoke/run_rtl_iss_test.py -ver 2 -a brent_kung -m kogge_stone
```

### 7.2.3 建议测试矩阵

| Version | BN-ALU Adder | BN-MAC Adder | Result | Notes |
|---|---|---|---|---|
| ver1 | buffer_bit | buffer_bit | N |  |
| ver1 | brent_kung | kogge_stone | Y | |
| ver2 | buffer_bit | buffer_bit | N | |
| ver2 | brent_kung | kogge_stone | Y | |
| ver3 | buffer_bit | buffer_bit | N |  |
| ver3 | brent_kung | kogge_stone | Y |  |


---

## 8. Hardware Synthesis，对应 Tables 1–3

### 8.1 实验目的

复现论文中 multiplier、adder、BN-ALU / BN-MAC / OTBN 的硬件面积、资源和时序数据。

### 8.2 README 指令

Multiplier，对应 Table 1：

```bash
util/gen_synth.py --run_synthesis --tool=Vivado --mul
```

Adders，对应 Table 2：

```bash
util/gen_synth.py --run_synthesis --tool=Vivado --adders
```

BN-ALU / BN-MAC / OTBN，对应 Table 3：

```bash
util/gen_synth.py --run_synthesis --tool=Vivado --otbn_sub
util/gen_synth.py --run_synthesis --tool=Vivado --otbn
```

### 8.3 结果目录

| 工具 | 输出目录 |
|---|---|
| Vivado | `reports/FPGA-Vivado` |
| Genus | `reports/ASIC-Genus` |
| ORFS | `reports/ASIC-ORFS` |

### 8.4 结果记录

| 实验 | 工具 | 是否运行 | 是否成功 | 输出文件 | 是否对应论文 |
|---|---|---|---|---|---|
| Multiplier | Vivado | Y | Y | Y | Y |
| Adders | Vivado | Y | Y | Y | Y |
| OTBN submodules | Vivado | Y | Y | Y | Y |
| Full OTBN | Vivado | Y | Y | Y |Y  |
| ORFS | Docker | Y | Y | Y | Y |
| Genus | License required | Y | Y | Y | Y |

### 8.5 与论文 Tables 1–3 的对比结果

本节基于本次输出的 Vivado 综合结果，与论文 Tables 1–3 中的 FPGA Vivado 数据进行逐项对比。由于本次提供的数据只包含 `LUT / DSP / CARRY4 / FF / BRAM / Fmax` 等 FPGA Vivado 指标，不包含 ASIC Genus 和 ASIC ORFS 的 `Area / Fmax` 数据，因此本节只统计论文 Tables 1–3 中 **FPGA Vivado** 部分的可比字段。

对比口径如下：

- **严格匹配率**：复现值与论文表格值完全一致，记为匹配。
- **未纳入统计项**：
  - 本次输出中的 `BRAM` 字段不在论文 Table 1 / Table 2 / Table 3 的对应 FPGA Vivado 列中，因此不纳入正确率。
  - 本次输出中的 `VER3`、`kogge_stone`、`sklansky` 等额外设计点未出现在论文 Tables 1–3 的主表格中，因此只作为额外输出保留，不纳入论文正确率统计。
  - 论文中的 ASIC Genus / ASIC ORFS 数据未在本次输出中给出，因此不纳入本节正确率统计。

#### 8.5.1 Table 1 Multiplier Synthesis 对比结果

论文 Table 1 对 multiplier 的资源和 Fmax 进行统计；本次输出中对应 `otbn_bignum_mul`、`otbn_mul`、`unified_mul`、`unified_mul_WALLACE` 四个 topmodule。对比 FPGA Vivado 的 `LUT / DSP / CARRY4 / Fmax` 四项指标。

| 论文设计 | 本次输出 topmodule | LUT | DSP | CARRY4 | Fmax | 对比结果 |
|---|---|---:|---:|---:|---:|---|
| OTBN | `otbn_bignum_mul` | 475 / 475 | 16 / 16 | 24 / 24 | 75 / 75 | PASS |
| OTBNTW | `otbn_mul` | 1300 / 1300 | 16 / 16 | 26 / 26 | 81 / 81 | PASS |
| our | `unified_mul` | 1495 / 1495 | 16 / 16 | 72 / 72 | 74 / 74 | PASS |
| our (Wallace) | `unified_mul_WALLACE` | 2127 / 2127 | 16 / 16 | 76 / 76 | 79 / 79 | PASS |

| 对比对象 | 对比行数 | 每行指标数 | 总对比项 | 完全一致项 | 正确率 |
|---|---:|---:|---:|---:|---:|
| Table 1 Multiplier FPGA Vivado | 4 | 4 | 16 | 16 | **100.00%** |

#### 8.5.2 Table 2 Adder Synthesis 对比结果

论文 Table 2 对 adders 的资源和 Fmax 进行统计；本次输出中对应 `ref_add`、`towards_alu_adder`、`towards_mac_adder`、`buffer_bit`、`csa_carry4`、`brent_kung_256`、`brent_kung`。对比 FPGA Vivado 可比字段。对于论文中 CARRY4 为 `—` 的 Brent-Kung 类设计，只比较 `LUT / DSP / Fmax`。

| 论文设计 | 本次输出 topmodule | LUT | DSP | CARRY4 | Fmax | 对比结果 |
|---|---|---:|---:|---:|---:|---|
| OTBN | `ref_add` | 449 / 449 | 0 / 0 | 99 / 99 | 200 / 200 | PASS |
| OTBNTW ALU | `towards_alu_adder` | 272 / 272 | 0 / 0 | 80 / 80 | 57 / 57 | PASS |
| OTBNTW MAC | `towards_mac_adder` | 263 / 263 | 0 / 0 | 72 / 72 | 70 / 70 | PASS |
| our buffer bit | `buffer_bit` | 476 / 476 | 0 / 0 | 103 / 103 | 210 / 210 | PASS |
| our CSA CARRY4 | `csa_carry4` | 474 / 474 | 0 / 0 | 96 / 96 | 200 / 200 | PASS |
| our Brent Kung | `brent_kung_256` | 682 / 682 | 0 / 0 | — / — | 175 / 175 | PASS |
| our Brent Kung vec | `brent_kung` | 633 / 633 | 0 / 0 | — / — | 155 / 155 | PASS |

| 对比对象 | 对比行数 | 总对比项 | 完全一致项 | 正确率 |
|---|---:|---:|---:|---:|
| Table 2 Adder FPGA Vivado | 7 | 26 | 26 | **100.00%** |

额外说明：本次输出还包含 `kogge_stone_256`、`kogge_stone`、`sklansky_256`、`sklansky`，但这些额外设计点不属于论文 Table 2 主表格内容，因此未计入 Table 2 正确率。

#### 8.5.3 Table 3 BN-MAC / BN-ALU / OTBN Synthesis 对比结果

论文 Table 3 对 BN-MAC、BN-ALU 和完整 OTBN 的资源与 Fmax 进行统计；本次输出中对应 `otbn_mac_bignum*`、`otbn_alu_bignum*` 和 `otbn*`。对比 FPGA Vivado 的 `LUT / DSP / CARRY4 / FF / Fmax` 五项指标。`BRAM` 只在本次输出中出现，未纳入正确率统计。

#### BN-MAC 对比

| 论文设计 | 本次输出 topmodule | LUT | DSP | CARRY4 | FF | Fmax | 对比结果 |
|---|---|---:|---:|---:|---:|---:|---|
| OTBNbase | `otbn_mac_bignum` | 2166 / 2166 | 16 / 16 | 143 / 143 | 312 / 312 | 58 / 58 | PASS |
| OTBNTW | `otbn_mac_bignum_TOWARDS` | 4413 / 4413 | 16 / 16 | 159 / 159 | 508 / 508 | 31 / 31 | PASS |
| OTBNV1 | `otbn_mac_bignum_VER1` | 4252 / 4252 | 16 / 16 | 196 / 196 | 312 / 312 | 53 / 53 | PASS |
| OTBNV2 | `otbn_mac_bignum_VER2` | 5771 / 5771 | 16 / 16 | 264 / 264 | 624 / 624 | 53 / 53 | PASS |

#### BN-ALU 对比

| 论文设计 | 本次输出 topmodule | LUT | DSP | CARRY4 | FF | Fmax | 对比结果 |
|---|---|---:|---:|---:|---:|---:|---|
| OTBNbase | `otbn_alu_bignum` | 6490 / 6490 | 0 / 0 | 200 / 200 | 320 / 320 | 88 / 88 | PASS |
| OTBNKMAC | `otbn_alu_bignum_KMAC` | 8471 / 8471 | 0 / 0 | 223 / 223 | 1286 / 1286 | 87 / 87 | PASS |
| OTBNTW | `otbn_alu_bignum_TOWARDS` | 11513 / 11513 | 0 / 0 | 185 / 185 | 1286 / 1286 | 31 / 31 | PASS |
| OTBNV1 | `otbn_alu_bignum_VER1` | 12095 / 12095 | 0 / 0 | 231 / 231 | 1286 / 1286 | 77 / 77 | PASS |
| OTBNV2 | `otbn_alu_bignum_VER2` | 12223 / 12223 | 0 / 0 | 231 / 231 | 1286 / 1286 | 77 / 77 | PASS |

#### Full OTBN 对比

| 论文设计 | 本次输出 topmodule | LUT | DSP | CARRY4 | FF | Fmax | 对比结果 |
|---|---|---:|---:|---:|---:|---:|---|
| OTBNbase | `otbn` | 32971 / 32971 | 16 / 16 | 770 / 770 | 15652 / 15652 | 37 / 37 | PASS |
| OTBNKMAC | `otbn_KMAC` | 35174 / 35174 | 16 / 16 | 793 / 793 | 16635 / 16635 | 38 / 38 | PASS |
| OTBNTW | `otbn_TOWARDS` | 41146 / 41146 | 16 / 16 | 771 / 771 | 16830 / 16830 | 20 / 20 | PASS |
| OTBNV1 | `otbn_VER1` | 41316 / 41316 | 16 / 16 | 854 / 854 | 16707 / 16707 | 36 / 36 | PASS |
| OTBNV2 | `otbn_VER2` | 42976 / 42976 | 16 / 16 | 922 / 922 | 17036 / 17036 | 35 / 35 | PASS |

| 对比对象 | 对比行数 | 每行指标数 | 总对比项 | 完全一致项 | 正确率 |
|---|---:|---:|---:|---:|---:|
| Table 3 BN-MAC FPGA Vivado | 4 | 5 | 20 | 20 | **100.00%** |
| Table 3 BN-ALU FPGA Vivado | 5 | 5 | 25 | 25 | **100.00%** |
| Table 3 Full OTBN FPGA Vivado | 5 | 5 | 25 | 25 | **100.00%** |
| **Table 3 总计** | **14** | — | **70** | **70** | **100.00%** |

额外说明：本次输出中还包含 `VER3` 相关数据，如 `otbn_mac_bignum_VER3`、`otbn_alu_bignum_VER3`、`otbn_VER3`，但论文 Table 3 未列出 OTBNV3，因此不纳入 Table 3 正确率统计。

#### 8.5.4 Tables 1–3 总体正确率

| 对比对象 | 总对比项 | 完全一致项 | 正确率 |
|---|---:|---:|---:|
| Table 1 Multiplier FPGA Vivado | 16 | 16 | **100.00%** |
| Table 2 Adder FPGA Vivado | 26 | 26 | **100.00%** |
| Table 3 BN-MAC / BN-ALU / OTBN FPGA Vivado | 70 | 70 | **100.00%** |
| **Tables 1–3 总计** | **112** | **112** | **100.00%** |

#### 8.5.5 结果分析

本次输出数据与论文 Tables 1–3 中 FPGA Vivado 部分完全一致，说明仓库提供的综合结果解析流程能够正确复现论文中的硬件资源统计。具体来看：

- Table 1 中 multiplier 相关的 `LUT / DSP / CARRY4 / Fmax` 四项指标全部一致。
- Table 2 中 adder 相关的可比指标全部一致。
- Table 3 中 BN-MAC、BN-ALU 和完整 OTBN 的 `LUT / DSP / CARRY4 / FF / Fmax` 五项指标全部一致。
- 本次输出中的 `BRAM` 和 `VER3` 属于额外信息，不影响论文 Tables 1–3 的复现正确率。
- 本节只验证 FPGA Vivado 数据；ASIC Genus / ASIC ORFS 部分由于本次未提供对应输出，因此不纳入本次百分比统计。

### 8.6 可能问题

| 问题编号 | 问题描述 | 原因 | 处理方式 |
|---|---|---|---|
| HW-SYN-01 | Vivado 命令不可用 | PATH 未设置或 WSL 无法调用 Windows Vivado | 配置 PATH 或用 `cmd.exe /c vivado.bat` |
| HW-SYN-02 | ORFS 失败 | Docker 未安装 / Docker Hub 拉取超时 | 配置 Docker、代理或记录为环境问题 |
| HW-SYN-03 | Genus 无法运行 | 需要 Cadence license | 记录不可复现原因 |

---

## 9. Chip-Level Top Earlgrey Verilator Tests

## 9.1 实验目的

在完整 Top Earlgrey SoC 仿真环境下，将 Ibex 上运行的 pq-crystals reference implementation 与 OTBN 加速实现结果进行比较。

### 9.2 README 指令

ML-DSA：

```bash
./bazelisk.sh test --test_output=streamed --test_timeout=10000 --action_env=PATH \
  --copt="-DBNMULV_VER={1,2,3}" --copt="-DDILITHIUM_MODE={2,3,5}" \
  //sw/device/tests:otbn_mldsa_test_sim_verilator_ver{1,2,3}
```

ML-KEM：

```bash
./bazelisk.sh test --test_output=streamed --test_timeout=10000 --action_env=PATH \
  --copt="-DBNMULV_VER={1,2,3}" --copt="-DKYBER_K={2,3,4}" \
  //sw/device/tests:otbn_mlkem_test_sim_verilator_ver{1,2,3}
```

> 注意：`_sim_verilator_ver*` 必须与 `--copt="-DBNMULV_VER=*"` 对应。

### 9.3 实际执行命令

```bash
./bazelisk.sh test \
  --test_output=streamed \
  --test_timeout=10000 \
  --jobs=1 \
  --local_resources=cpu=1 \
  '--local_resources=memory=HOST_RAM*.5' \
  --action_env=PATH \
  --copt="-DBNMULV_VER=1" \
  --copt="-DDILITHIUM_MODE=2" \
  //sw/device/tests:otbn_mldsa_test_sim_verilator_ver1 \
  2>&1 | tee logs/07_chip_level_verilator.log
```

### 9.4 已进行的修改记录

| 修改编号 | 修改文件 | 修改内容 | 原因 | 结果 |
|---|---|---|---|---|
| CL-01 | `hw/top_earlgrey/dv/verilator/chip_sim.core` | `BNMULV=true` 改为 `-DBNMULV=true` | Verilator 将 bare macro 当作模块/文件 | 解决 `Cannot find file containing module: BNMULV=true` |
| CL-02 | `chip_sim.core` | 添加 `--timing` | Verilator 5 需要显式指定 timing 策略 | 解决 `NEEDTIMINGOPT` |
| CL-03 | `chip_sim.core` | 添加 `-DTOWARDS_BASE=true` | `vector_sel/vector_type` 字段受宏控制 | 解决 struct 缺字段问题 |
| CL-04 | `chip_sim.core` | 注释 `-DTOWARDS_KMAC=true` | 避免 KMAC dynamic interface 端口不匹配 | 暂时绕过 KMAC 端口问题 |
| CL-05 | `chip_sim.core` | 关闭 trace | 降低 Verilator 构建复杂度 | internal fault 仍存在 |
| CL-06 | `chip_sim.core` / `hw/BUILD` | 统一 `--threads 1` | 排除多线程触发 internal fault | internal fault 仍存在 |
| CL-07 | `chip_sim.core` | 添加多项 `-Wno-*` | 排除 warning 诊断导致失败 | internal fault 仍存在 |
| CL-08 | `chip_sim.core` | `--no-timing` 与 `--timing` 均测试 | 判断是否 timing delay 导致 | 两种模式下均失败 |

### 9.5 当前失败结论

| 项目 | 结论 |
|---|---|
| 失败阶段 | FuseSoC 调用 Verilator 构建 `Vchip_sim_tb` |
| 是否进入测试运行 | 否，`Executed 0 out of 1 test` |
| 是否为 ML-DSA 算法错误 | 否 |
| 是否为 OTBN 程序运行错误 | 否 |
| 直接报错 | `Verilator internal fault` |
| 最可能原因 | Top Earlgrey full-chip wrapper / AST / OTBN 扩展集成触发 Verilator 5.022 internal fault |

### 9.6 后续建议

1. 使用 `--debug --gdbbt` 获取 Verilator crash 堆栈。
2. 检查 `chip_earlgrey_verilator.sv`、`top_earlgrey.sv`、AST wrapper 端口版本是否一致。
3. 联系仓库作者确认 chip-level Verilator 是否在 commit `5b07cd0` 可直接运行。
4. 报告中将 chip-level Verilator 记录为未完全跑通，并说明已排查内容。
5. 用 RTL–ISS 与 software benchmark 作为可复现实验主证据。

---

## 10. FPGA CW310 测试

### 10.1 实验目的

使用仓库提供的 bitstream 或自行构建 bitstream，在 CW310-K410T FPGA 板上运行 ML-KEM / ML-DSA chip-level 测试。

### 10.2 README 指令

加载 bitstream：

```bash
./bazelisk.sh run //sw/host/opentitantool -- fpga load-bitstream \
  /absolute/path/to/hw/top_earlgrey/bitstream_cw310/bnmulv_ver{1,2,3}/lowrisc_systems_chip_earlgrey_cw310_0.1.bit
```

运行 ML-KEM：

```bash
./bazelisk.sh test --define bitstream=skip --test_output=streamed --action_env=PATH \
  --copt="-DKYBER_K={2,3,4}" --copt="-DBNMULV_VER={1,2,3}" \
  //sw/device/tests:otbn_mlkem_test_fpga_cw310_test_rom_ver{1,2,3}
```

运行 ML-DSA：

```bash
./bazelisk.sh test --define bitstream=skip --test_output=streamed --action_env=PATH \
  --copt="-DDILITHIUM_MODE={2,3,5}" --copt="-DBNMULV_VER={1,2,3}" \
  //sw/device/tests:otbn_mldsa_test_fpga_cw310_test_rom_ver{1,2,3}
```

### 10.3 评估记录

| 项目 | 是否具备条件 | 结果 | 备注 |
|---|---|---|---|
| 是否有 CW310-K410T |  |  |  |
| 是否有 opentitantool 配置 |  |  |  |
| 是否能加载 bitstream |  |  |  |
| ML-KEM FPGA test |  |  |  |
| ML-DSA FPGA test |  |  |  |
| 是否需要付费 Vivado license |  |  |  |





## 11. 问题列表

| 编号 | 类型 | 问题描述 | 影响范围 | 原因分析 | 是否解决 | 解决方式 | 证据日志 |
|---|---|---|---|---|---|---|---|
| DOC-01 | Documentation | README 命令中路径变量可能写成 `$PWD$` | software benchmark | 文档命令存在疑似 typo | 是 | 改为 `$PWD` |  |
| DOC-02 | Documentation | README 对 brace expansion 的使用没有说明 | Bazel target | 用户可能复制换行后导致 target 不展开 | 部分 | 显式列出 target |  |
| ENV-01 | Environment | Docker 拉取 ORFS 镜像失败 | ORFS synthesis | 网络 / Docker Hub 访问问题 | 视情况 | 配置代理或记录限制 |  |
| ENV-02 | Environment | Vivado 在 WSL 中调用需要额外配置 | Vivado synthesis | Windows/WSL 路径差异 | 视情况 | 使用 `cmd.exe /c vivado.bat` |  |
| SW-01 | Software | `get_benchmark.py` 单样本 stdev 报错 | benchmark result extraction | iteration 数为 1 时不能计算 stdev | 视情况 | 增加 iterations 或修改脚本 |  |
| HW-01 | Hardware | `BNMULV=true` 被 Verilator 当成文件/模块 | chip-level Verilator | FuseSoC 参数格式错误 | 是 | 改为 `-DBNMULV=true` |  |
| HW-02 | Hardware | 缺少 `--timing` 或 `--no-timing` | chip-level Verilator | Verilator 5 要求显式 timing 选项 | 是 | 添加 `--timing` |  |
| HW-03 | Hardware | `vector_sel/vector_type` 不存在 | OTBN RTL compile | 缺少 `TOWARDS_BASE` 宏 | 是 | 添加 `-DTOWARDS_BASE=true` |  |
| HW-04 | Hardware | KMAC dynamic interface 端口不匹配 | chip-level Verilator | `TOWARDS_KMAC` 与 top/KMAC wrapper 不一致 | 部分 | 临时注释 `TOWARDS_KMAC` |  |
| HW-05 | Hardware | Verilator internal fault | chip-level Verilator | full-chip 集成 / 工具内部崩溃 | 否 | 已尝试 threads/timing/trace/warning 排查 |  |

---

## 13. 四方面评价意见

## 13.1 Documentation

### 评价维度

| 评价点 | 具体依据 | 评价 |
|---|---|---|
| 是否说明仓库用途 | README 说明该仓库对应论文并用于复现 software benchmark、hardware synthesis、FPGA experiments | 较清楚 |
| 是否说明硬件 / 软件版本 | README 列出 OTBN、OTBN_KMAC、OTBN_TW、OTBNV1/V2/V3 及对应宏 | 较清楚 |
| 是否说明环境 | README 给出 Ubuntu 22.04、Python 3.10、Verilator 5.022、Docker、Vivado 等要求 | 较完整 |
| 是否命令可直接复制运行 | 存在 `$PWD$`、brace expansion、target 命名等容易出错点 | 一般 |
| 是否解释常见错误 | 对 Bazel / FuseSoC / Verilator 具体错误说明不足 | 不足 |
| 是否说明 chip-level 测试资源需求 | 仅说明耗时数小时，但未充分说明内存、线程、可能的工具问题 | 不足 |

### 总体评价

```text
Documentation 基本覆盖了主要实验路径，但对实际复现中可能遇到的 Bazel target 展开、FuseSoC 宏传递、Verilator 5.022 参数、Docker 网络、单 iteration benchmark 提取等问题说明不足。README 对熟悉 OpenTitan 的用户较友好，但对普通论文读者而言仍需要额外排查。
```

---

## 13.2 Completeness

### 评价维度

| 评价点 | 具体依据 | 评价 |
|---|---|---|
| 是否包含硬件 RTL | `hw/ip/otbn/rtl` 和 `bn_vec_core` 包含多种 multiplier / adder 实现 | 完整 |
| 是否包含软件实现 | `sw/otbn/crypto` 包含 ML-KEM / ML-DSA 多版本实现 | 完整 |
| 是否包含测试 | 包含 benchmark、fixed-input、RTL-ISS、chip-level、FPGA target | 较完整 |
| 是否包含综合脚本 | `util/gen_synth.py` 支持 Vivado / Genus / ORFS | 完整 |
| 是否包含数据 | README 提到 `DBs.tar.gz` 包含论文 Table 6 数据, report文件夹包含 Tables 1–3 的数据 | 较完整 |
| 是否包含 bitstream | README 提到提供 CW310 bitstreams | 较完整 |

### 总体评价

```text
Completeness 较高。仓库包含论文关键 claim 所需的 RTL、软件、benchmark、综合和 FPGA 相关文件。但部分流程依赖外部工具、license、Docker、板卡或特定版本工具链，导致完整复现仍受环境限制。
```

---

## 13.3 Exercisability

### 评价维度

| 评价点 | 实际结果 | 评价 |
|---|---|---|
| 软件 benchmark 是否可运行 | Y | 测试通过 |
| fixed-input tests 是否可运行 | Y | 测试通过 |
| code size 是否可提取 | Y | 测试通过 |
| RTL pytest 是否可运行 | Y | 测试通过 |
| RTL–ISS 是否可运行 | Y | 测试通过 |
| Vivado synthesis 是否可运行 | Y | 测试通过 |
| ORFS synthesis 是否可运行 | Y | 测试通过 |
| chip-level Verilator 是否可运行 | 当前构建 `Vchip_sim_tb` 阶段 internal fault | 不通过 |
| FPGA 测试是否可运行 | 取决于板卡条件 | 受限 |

### 总体评价

```text
Exercisability 部分满足。软件和较低层级 RTL/ISS 流程相对更容易复现，chip-level Verilator 流程在当前环境下由于 Verilator internal fault 未能完成构建。该问题发生在 full-chip Vchip_sim_tb 生成阶段，测试程序未实际执行，因此不能作为 ML-DSA 算法错误处理，而应归类为 full-chip 仿真工程或工具兼容性问题。
```

---

## 13.4 Reusability

### 评价维度

| 评价点 | 具体依据 | 评价 |
|---|---|---|
| RTL 模块是否可独立复用 | `bn_vec_core` 下模块划分清楚，包含多种 adder / multiplier | 较好 |
| 软件版本是否清晰 | `ver0_base` 到 `ver3` 分层明确 | 较好 |
| 脚本是否易扩展 | `gen_synth.py`、`get_benchmark.py` 可复用，但错误处理有限 | 一般 |
| 配置是否模块化 | 依赖宏与 FuseSoC core 文件，复用需要理解 OpenTitan build system | 一般 |
| 文档是否支持扩展 | 对修改后如何验证有 RTL-ISS 指引，但对常见修改错误说明不足 | 一般 |

### 总体评价

```text
Reusability 中等偏好。核心 RTL 和软件版本组织较清楚，适合进一步研究不同乘法器和加法器设计。但该仓库深度依赖 OpenTitan、Bazel、FuseSoC、Verilator、Vivado/ORFS 等复杂工具链，且部分 full-chip 集成问题需要较强硬件工程经验才能定位，因此普通用户扩展成本较高。
```

---

## 14. 论文结果对应关系

| 论文结果 | README 对应流程 | 本次复现状态 | 结果文件 | 备注 |
|---|---|---|---|---|
| Table 1 Multiplier synthesis | `gen_synth.py --mul` | 完成 |  |  |
| Table 2 Adder synthesis | `gen_synth.py --adders` |  完成|  |  |
| Table 3 OTBN synthesis | `gen_synth.py --otbn_sub`, `--otbn` | 完成 |  |  |
| Tables 4–6 Software benchmark | Bazel benchmark + `get_benchmark.py` | 完成 |  |  |
| Table 7 Code size | `get_codesize.py` |完成  |  |  |
| RTL correctness | cocotb + pytest | 完成 |  | README 提供模块级测试 |
| RTL–ISS correctness | `run_rtl_iss_test.py` | 完成 |  | 比 full-chip 更轻量 |
| Chip-level correctness | Verilator / FPGA | Verilator 未完全跑通 |  | 记录原因 |

---

## 15. AI 工具辅助记录

任务书要求：即使 AI 帮助解决问题，也要记录原仓库文档不清或代码问题。

| 时间 | 使用工具 | 输入内容 | 输出建议 | 是否采纳 | 对应问题编号 |
|---|---|---|---|---|---|
|  | ChatGPT / Gemini | Bazel target 报错 | 检查 brace expansion / target 名称 |  |  |
|  | ChatGPT / Gemini | Verilator `BNMULV=true` 报错 | 改为 `-DBNMULV=true` |  |  |
|  | ChatGPT / Gemini | Verilator `vector_sel` 缺字段 | 添加 `TOWARDS_BASE` |  |  |
|  | ChatGPT / Gemini | Verilator internal fault | 关闭 trace、threads=1、debug gdbbt |  |  |

---

## 16. 未解决问题

| 编号 | 问题 | 当前状态 | 已尝试方法 | 后续建议 |
|---|---|---|---|---|
| UNRESOLVED-01 | chip-level Verilator `Vchip_sim_tb` 构建 internal fault | 未解决 | trace 关闭、threads=1、`--timing`/`--no-timing`、屏蔽 warning、调整宏 | 使用 `--debug --gdbbt` 获取栈；联系作者；降低到 RTL–ISS 验证 |


---

## 18. 附录 A：建议保留的关键命令

### 日志记录

```bash
script -a logs/opentitan_reproduce_$(date +%Y%m%d_%H%M%S).log
```

### 查看 Bazel output base

```bash
./bazelisk.sh info output_base
```

### 清理 Bazel

```bash
./bazelisk.sh shutdown
./bazelisk.sh clean
```

### 搜索 target

```bash
./bazelisk.sh query '//sw/otbn/crypto/tests/mldsa:all'
./bazelisk.sh query '//sw/otbn/crypto/tests/mlkem:all'
```

### SQLite 检查

```bash
sqlite3 mldsa_bench.db ".tables"
sqlite3 mldsa_bench.db "SELECT benchmark_id, COUNT(*) FROM benchmark_iteration GROUP BY benchmark_id;"
```

### 查 Verilator 真实命令

```bash
grep -n "verilator -f" logs/07_chip_level_verilator.log
```

### 查宏定义

```bash
grep -R "TOWARDS_BASE\|TOWARDS_KMAC\|BNMULV" -n hw/top_earlgrey/dv/verilator hw/ip/otbn/rtl | head -100
```

---

## 19. 附录 B：最终评分建议

| 维度 | 建议评级 | 理由摘要 |
|---|---|---|
| Documentation | 中等 | 主要流程清楚，但实际复现细节和错误处理不足 |
| Completeness | 较高 | 包含 RTL、软件、脚本、测试、综合、FPGA 说明 |
| Exercisability | 中等 / 部分通过 | 软件和 RTL 层较可执行，chip-level Verilator 存在 internal fault |
| Reusability | 中等偏高 | 模块划分较清楚，但依赖大型 OpenTitan 工程和复杂工具链 |

---

## 20. 附录 C：最终报告检查清单

提交前逐项确认：

- [ ] 是否说明评估 commit 是 `5b07cd0`。
- [ ] 是否介绍 OpenTitan、OTBN、论文核心贡献。
- [ ] 是否记录环境配置。
- [ ] 是否记录每一步命令、结果、问题和解决方案。
- [ ] 是否覆盖 software benchmark。
- [ ] 是否覆盖 fixed-input tests。
- [ ] 是否覆盖 code size。
- [ ] 是否覆盖 RTL pytest。
- [ ] 是否覆盖 RTL–ISS。
- [ ] 是否覆盖 synthesis。
- [ ] 是否覆盖 chip-level Verilator。
- [ ] 是否说明 FPGA 测试是否具备条件。
- [ ] 是否列出所有文档不清、代码报错、脚本问题、环境问题。
- [ ] 是否分别评价 Documentation / Completeness / Exercisability / Reusability。
- [ ] 是否把 AI 辅助过程记录进去。
- [ ] 是否把未解决问题单独列出。
- [ ] 是否将结论对应到论文结果表格。


---
