# pyAudioAnalysis CNPort

pyAudioAnalysis CNPort 基于上游 [pyAudioAnalysis](https://github.com/tyiannak/pyAudioAnalysis)，面向国内开源发布和国产化软硬件环境验证。在保留上游 Python API、命令行入口、音频特征提取、分类、分割和可视化能力的基础上，CNPort 对源码同步、依赖安装、原生测试和多平台验收进行统一管理。

> 上游原始说明保存在 [README_UPSTREAM.md](README_UPSTREAM.md)，上游许可证和引用信息以该文件及 [LICENSE.md](LICENSE.md) 为准。

上游仓库：`tyiannak/pyAudioAnalysis`。

## 国产化背景

pyAudioAnalysis 是经典音频分析库，不是深度学习模型框架。其主要计算链路为“音频读取 → 短时/中时特征提取 → NumPy/SciPy 统计计算 → scikit-learn / hmmlearn 分类、分割与评估”。原生版本没有 CUDA、PyTorch 或 Triton 依赖，核心目标是验证原生 CPU 路径能否在不同国产 CPU、运行时和 Python 科学计算栈上稳定工作。

CNPort 的国产化工作遵循以下原则：

- 保持上游 API、CLI 和数据处理语义，不引入专用 GPU 后端作为通过条件；
- 重点验证 `ShortTermFeatures`、`MidTermFeatures`、`audioSegmentation` 等计算热点在不同 CPU 架构和数值库上的兼容性；
- 将平台差异放在运行环境、依赖和测试配置中，避免破坏上游库的通用实现；
- 同时保留上游原始 README，便于对照变更和后续同步。

## 安装

```bash
git clone <CNPort-repository-url> pyAudioAnalysis
cd pyAudioAnalysis
python -m pip install -r requirements.txt
python -m pip install -e .
```

建议每个平台使用独立 Python 虚拟环境，并使用与平台科学计算库匹配的 NumPy、SciPy、scikit-learn 和 hmmlearn 版本。库的基本用法和完整功能说明请参阅 [README_UPSTREAM.md](README_UPSTREAM.md) 及上游 Wiki。

## 国产化评估方案

验收以“同一源码提交 + 平台原生 CPU 环境 + 可复现测试数据”为基本单元，不将不同平台的运行时间或浮点实现差异直接作为功能失败条件。

| 层级 | 验证内容 | 通过标准 |
|---|---|---|
| 环境与安装 | Python、依赖导入、editable install、CLI 启动 | 依赖可导入，包可安装和调用 |
| 单元测试 | `pytests/` 中特征提取和分割测试 | pytest 全部通过 |
| 本地集成 | `cmd_test_00/01/04/05/07/08.sh` | 脚本返回码全部为 0 |
| 外部数据集成 | `cmd_test_02*`、`03`、`06`、`09`–`12_*`、`script_tests.py` | 脚本全部通过，日志无异常关键字 |
| 跨平台一致性 | 在 NVIDIA、昇腾、海光、寒武纪、摩尔线程、平头哥等平台重复上述检查 | 各平台单独记录结果，不以另一平台作为标准答案 |

## 原生测试方法

### 外部测试数据集

上游外部测试脚本依赖作者私有数据路径，该数据没有随上游代码发布。CNPort 使用公开的 pyAudioAnalysis 外部测试数据集：

<http://dataset.audio-native.cn/datasets/opendataset/pyaudioanalysis-datasets.git>

下载前请确认已安装 Git LFS（<https://git-lfs.com>）。数据集仓库公开提供 `pyAudioAnalysisData.tar.gz`，执行以下命令下载并解压：

```bash
# 安装 Git LFS 后初始化
git lfs install

# 下载公开数据集
git clone http://dataset.audio-native.cn/datasets/opendataset/pyaudioanalysis-datasets.git
cd pyaudioanalysis-datasets
git lfs pull

# 解压后，将数据集目录传给测试脚本
tar -xzf pyAudioAnalysisData.tar.gz
export PYAUDIOANALYSIS_DATA_ROOT="$PWD/pyAudioAnalysisData"
cd ../pyAudioAnalysis/tests
python script_tests.py
bash cmd_test_02.sh "$PYAUDIOANALYSIS_DATA_ROOT"
# 其余 cmd_test_02_B.sh、cmd_test_02_C.sh、cmd_test_03.sh、
# cmd_test_06.sh、cmd_test_09.sh、cmd_test_10.sh、cmd_test_11.sh、
# cmd_test_12_*.sh 以同样方式传入数据集根目录
```

### 直接运行

```bash
# 单元测试
python -m pytest -q pytests

# 本地集成测试
cd tests
bash cmd_test_00.sh ..
bash cmd_test_01.sh ..
bash cmd_test_04.sh ..
bash cmd_test_05.sh ..
bash cmd_test_07.sh ..
bash cmd_test_08.sh ..

# 有外部数据集时：先按上文命令下载并解压公开数据集
export PYAUDIOANALYSIS_DATA_ROOT=/path/to/pyAudioAnalysisData
python script_tests.py
bash cmd_test_02.sh "$PYAUDIOANALYSIS_DATA_ROOT"
# 其余 cmd_test_02_B.sh、cmd_test_02_C.sh、cmd_test_03.sh、
# cmd_test_06.sh、cmd_test_09.sh、cmd_test_10.sh、cmd_test_11.sh、
# cmd_test_12_*.sh 以同样方式传入数据集根目录
```

### 使用标准 Docker 镜像

为避免宿主机依赖污染，CNPort 原生验收使用批准的 CPU 镜像，并在独立 worktree 中运行。x86_64/amd64 示例：

```bash
python common/skills/testing/pyaudioanalysis/skill-pyaudioanalysis-native-test-docker/scripts/run_pyaudioanalysis_native_tests_docker.py \
  --repo-root /path/to/pyAudioAnalysis \
  --platform-name nvidia-a40 \
  --data-root /path/to/pyAudioAnalysisData \
  --output-root /path/to/HY_CNPort_Test_Output
```

镜像按宿主机架构自动选择：

- amd64：`open-audio-native-registry.cn-beijing.cr.aliyuncs.com/hy_cnport/pyaudioanalysis-cpu-py39:linux-amd64`
- arm64：`open-audio-native-registry.cn-beijing.cr.aliyuncs.com/hy_cnport/pyaudioanalysis-cpu-py39:linux-arm64`

运行器会：创建 `pyaudioanalysis-docker-<platform>` worktree、拉取对应镜像、运行 pytest 和 shell 集成测试，并生成 Markdown 报告。完整验收时应在目标平台执行并保留报告；不要在主 checkout 中安装依赖或直接运行会生成临时文件的外部脚本。

## 平台测试结果

以下表格记录最近一次可复现实机/容器验收。`4 passed` 表示当前上游 pytest 套件收集到的 4 个用例；本库是原生 CPU 实现，Docker 测试验证的是容器内 CPU 路径，不代表 GPU 加速结果。

| 平台 | 原生单元测试 | 本地集成测试 | 外部数据集成测试 | 结论 |
|---|---:|---:|---:|---|
| NVIDIA A40 | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | 当前平台测试通过 |
| 华为昇腾 910B | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | 当前平台测试通过 |
| 海光 BW1000 / C86 | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | 修复运行器路径问题后，当前平台测试通过 |
| 寒武纪 MLU590（Docker amd64） | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | Docker 原生单元、本地集成及外部数据集测试全部通过 |
| 摩尔线程 S5000 | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | 当前平台测试通过 |
| 阿里平头哥 | ✅ 4/4 | ✅ 6/6 | ✅ 15/15 | 当前平台测试通过 |

NVIDIA A40、华为昇腾 910B、摩尔线程 S5000 和阿里平头哥当前平台测试通过。寒武纪 MLU590 已完成 Docker 原生单元、本地集成和外部数据集集成测试，全部通过。

外部集成测试需要使用上述公开数据集地址；上游原始测试脚本中的作者私有数据路径不作为发布配置。各平台测试前均需先完成注册、登录、下载和解压，再执行外部集成测试。

## 结果与同步说明

- 本仓库保留跨平台原生测试结果和可复现的验证说明；发布时请以当前代码快照和测试报告为准。
- `README_UPSTREAM.md` 保存上游原始 README；本文件记录 CNPort 背景、评估口径、测试流程和已有平台结果。
- pyAudioAnalysis 不依赖 CUDA/Triton；国产化验证关注原生 CPU 兼容性，而非 GPU 加速路径。
- 新增结果应同时记录源码提交、平台/CPU、Python 与关键依赖版本、测试命令、数据集版本、镜像 digest、通过/失败/跳过数量及报告路径。本次阿里平头哥（平头哥 ZW810E PPU 服务器，宿主 CPU 为 x86_64，Python 3.10.13、NumPy 1.25.2、SciPy 1.15.3、scikit-learn 1.7.2、hmmlearn 0.3.3，源码提交 `fc0ec09`，2026-09-28 实机原生执行）外部数据集测试未跳过任何脚本，14 个外部脚本和 `script_tests.py` 均返回 0，报告归档于 `HY_CNPort_Test_Output/pyAudioAnalysis/06-t-head-zw810e/pyaudioanalysis-t-head-zw810e-20260928/`。
