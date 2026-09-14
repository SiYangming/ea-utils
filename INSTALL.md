# ea-utils / fastq-join 安装与使用

## 归档说明

- 本仓库是 `ExpressionAnalysis/ea-utils` 的 fork，代码树即上游源码（`clipper/` 等），根目录的 `README.md`、`Dockerfile` 为上游文件，未做改动
- 归档的发布包 `ea-utils.1.1.2-537.tar.gz`（326,269 B，MD5 `b8d3bcc39edc7fe66d00ed54d9e59830`，SHA-256 `cd88c4fcee2182bdc09984abeaff012703e05c2de8514d3df12dfdbad1080b96`）**不入代码树**，见本仓库 Release `v1.1.2.537` 附件；包内为 `ea-utils.1.1.2-537/`，共 108 个文件（含 `google/`、`samtools/`、`sparsehash/`、`tidx/` 等随包依赖）
- 为什么要归档这一版：**1.1.2.537 无法从 GitHub 取得**。上游仓库带 tag 的唯一源码归档是 `1.04.807.tar.gz`，其源码实为 1.1.2-779；bioconda 现行 recipe（`version: 1.1.2.779`）也是指向 `https://github.com/ExpressionAnalysis/ea-utils/archive/1.04.807.tar.gz`。精确的 537 版本只存在于旧的发布包中
- 许可：见包内与上游仓库的许可文件（ea-utils 采用 MIT 类许可，`fastq-join` 等程序随上游发布）
- 说明：下文出现的 `native/main.py`、`native/install.sh`、`native/test/run_test.sh` 属于**本地封装层**（bioskills 模块 `modules/ea-utils/`），不在本仓库内；本仓库只提供上游源码与归档的发布包

# fastq-join（双端序列拼接）

```bash
tar zxf path/to/ea-utils.1.1.2-537.tar.gz -C /path/to/install/
cd /path/to/install/ea-utils.1.1.2-537/
sudo yum install gsl*        # 依赖 GSL（Debian/Ubuntu: libgsl-dev）
make -j 4
echo 'PATH=$PATH:/path/to/install/ea-utils.1.1.2-537/' >> ~/.bashrc
source ~/.bashrc
```

## 实战示例：QIIME 1.x 双端拼接与清洗

fastq-join 依据双端重叠把 paired-end reads 拼成单端序列，是 QIIME 1.x `join_paired_ends.py` 的核心引擎；
等价能力由 `native/main.py` 的 `join` 子命令提供，批量用法如下（与 `join_paired_ends.py` 对应）。

### 1. 批量双端拼接（逐样本 → 拼接产物汇总）

```bash
mkdir -p 02.join_paired_ends
for s in F3D0 F3D1 F3D2 F3D141 F3D142 F3D143
do
    python main.py join 00.qiime_data/${s}_L001_R1_001.fastq.gz \
                        00.qiime_data/${s}_L001_R2_001.fastq.gz \
                        -o 02.join_paired_ends/${s}.
done
# 产物：<s>.join（拼接）/ <s>.un1、<s>.un2（未拼接），即 fastq-join 的默认三件套
```

### 2. 接头切除 + 质量过滤（fastq-mcf）

```bash
python main.py mcf adapters.fa reads.R1.fq.gz reads.R2.fq.gz \
    -o clean.R1.fq.gz -o clean.R2.fq.gz -q 30 -l 50 --qual-mean 30
```

### 3. 参数说明（fastq-join / fastq-mcf）

| 参数 | 适用             | 说明                                                         |
| ---- | ---------------- | ------------------------------------------------------------ |
| `-o` | join/mcf/clipper | join 为文件名模板（含 `%` 或后缀 join/un1/un2）；mcf 每个输入一个 |
| `-m` | join             | 最小重叠长度（默认 6）                                       |
| `-p` | join/mcf/clipper | 允许的最大差异百分比（join 8 / mcf 10）                       |
| `-v` | join             | 校验 read id 匹配到第 C 个字符（Illumina 用空格）             |
| `-r` | join             | verbose stitch length report 输出                             |
| `-q` | mcf              | 触发碱基切除的质量阈值（默认 10）                             |
| `-l` | mcf              | 过滤后最小剩余长度（默认 19）                                 |
| `-s` | mcf              | 接头最小匹配长度的 log 尺度（默认 2.2）                       |
| `-t` | mcf              | 接头切除 occurrence 阈值（默认 0.25）                         |

## 环境安装（官方镜像优先，不维护本地配方）

官方已维护（bioconda → quay.io/biocontainers → depot.galaxyproject.org），直接拉取官方镜像运行工具二进制；`main.py` 驱动在宿主机跑。

> ⚠️ **版本差异**：bioconda 提供 `1.1.2.537` 与 `1.1.2.779` 两个版本；官方**容器（quay/depot）目前仅构建了 1.1.2.779**（`ea-utils:1.1.2.779--h9dd4a16_0`），`1.1.2.537` 无官方镜像 tag（2026-09 核实）。

### 1. Conda（包管理器安装）

```bash
mamba create -n ea-utils-native -c conda-forge -c bioconda ea-utils=1.1.2.537
conda activate ea-utils-native
fastq-join    # 断言
```

> Homebrew：homebrew-core 与 brewsci/bio 两源均**无 ea-utils 公式**（2026-09 核实 404），故不登记 brew 块。
>
> 一键安装也可直接运行 `native/install.sh`（现代规范：有 conda/mamba 时建 bioconda 环境 `ea-utils`，无 conda 时回退官方 GitHub 源码编译到 `~/software/ea-utils-<ver>` 并写 PATH；版本默认 1.1.2.537，与 `software_versions` 对齐。用法：`bash native/install.sh --help`）。

### 2. Docker（官方镜像）

```bash
docker pull quay.io/biocontainers/ea-utils:1.1.2.779--h9dd4a16_0
# 注意：必须 -u $(id -u):$(id -g) 挂载宿主用户，否则产物归 root
docker run --rm -u $(id -u):$(id -g) -v $PWD:/data -w /data \
    quay.io/biocontainers/ea-utils:1.1.2.779--h9dd4a16_0 \
    fastq-join R1.fastq R2.fastq -o joined.
```

### 3. Apptainer / Singularity

depot.galaxyproject.org 已预构建好 sif，直接拉取现成镜像即可（无需本地从 docker 转换）：

```bash
apptainer pull ea-utils.sif docker://depot.galaxyproject.org/singularity/ea-utils:1.1.2.779--h9dd4a16_0
apptainer run -B $PWD:/data -H /data ea-utils.sif fastq-join /data/R1.fastq /data/R2.fastq -o /data/joined.
```

### 4. 官方源码编译（无官方预编译二进制）

ea-utils 官方**无预编译二进制包**（GitHub 无 release assets，仅源码仓库），宿主机无 conda/docker 时走源码编译：

- **官网**：`https://expressionanalysis.github.io/ea-utils/`
- **GitHub 源码**：`https://github.com/ExpressionAnalysis/ea-utils`

```bash
# 官方源码归档（GitHub tag 1.04.807；内容实为 1.1.2-779 源码，与 bioconda recipe 同源）
wget https://github.com/ExpressionAnalysis/ea-utils/archive/1.04.807.tar.gz -P path/to/
tar zxf path/to/1.04.807.tar.gz -C path/to/
cd path/to/ea-utils-1.04.807/clipper
make -j 4     # 依赖 g++ / gsl / zlib（apt: build-essential libgsl-dev zlib1g-dev）
echo 'export PATH=$PATH:'"$HOME"'/path/to/ea-utils-1.04.807/clipper' >> ~/.bashrc
source ~/.bashrc
fastq-mcf -h | grep Version   # 断言
```

> 说明：GitHub 官方唯一带 tag 的源码归档为 `1.04.807.tar.gz`，其 `ea-utils.spec` 实为 **1.1.2-779** 源码；如需精确的 `1.1.2.537` 请走 conda 路线或使用本仓库 Release `v1.1.2.537` 的归档包。亦可直接运行 `native/install.sh --method source` 自动完成上述步骤。

## 测试

```bash
bash test/run_test.sh   # argv 构造验证为真实回归；fastq-join/clipper 已安装时追加真实执行
```

## 容器与 Conda 链接

- **Bioconda 页面**：`https://anaconda.org/channels/bioconda/packages/ea-utils/overview`
- **Docker**：`docker pull quay.io/biocontainers/ea-utils:1.1.2.779--h9dd4a16_0`
- **Singularity**：`https://depot.galaxyproject.org/singularity/ea-utils%3A1.1.2.779--h9dd4a16_0`
- 安装方式（本地）：`mamba create -n ea-utils -c conda-forge -c bioconda ea-utils=1.1.2.537`
