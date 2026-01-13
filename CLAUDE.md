# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**slime** 是一个用于 LLM 后训练(特别是强化学习扩展)的高性能框架,是 GLM-4.7、GLM-4.6、GLM-4.5 等大模型背后的 RL 训练框架。

### 核心架构

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│  Training       │──────│   Data Buffer    │──────│   Rollout       │
│  (Megatron)     │      │                  │      │  (SGLang +      │
│                 │──────│                  │──────│   router)       │
└─────────────────┘      └──────────────────┘      └─────────────────┘
```

- **Training (Megatron/FSDP)**: 主训练流程,从 Data Buffer 读取数据,训练后同步参数到 rollout
- **Rollout (SGLang + router)**: 生成新数据(包括奖励/验证器输出),存储到 Data Buffer
- **Data Buffer**: 桥接模块,管理提示初始化、自定义数据和 rollout 生成方法

### 支持的模型

- Qwen3 系列 (Qwen3Next, Qwen3MoE, Qwen3), Qwen2.5 系列
- DeepSeek V3 系列 (DeepSeek V3, V3.1, DeepSeek R1)
- Llama 3

## 常用命令

### 环境安装

**Docker 方式(推荐):**
```bash
docker pull slimerl/slime:latest
docker run --rm --gpus all --ipc=host --shm-size=16g \
  --ulimit memlock=-1 --ulimit stack=67108864 \
  -it slimerl/slime:latest /bin/bash
```

**手动安装:**
```bash
bash build_conda.sh  # 创建 Python 3.12 conda 环境,安装依赖
pip install -e .      # 安装 slime
```

### 运行训练

**标准流程:**
```bash
# 1. 加载模型配置
source scripts/models/qwen3-4B.sh

# 2. 运行训练脚本
bash scripts/run-qwen3-4B.sh
```

**训练入口:**
- `train.py`: 同步训练(rollout → train → update_weights 循环)
- `train_async.py`: 异步训练(rollout 与 train 重叠,更高效率)

### 权重转换

```bash
# HF 格式 → Megatron torch_dist 格式
python tools/convert_hf_to_torch_dist.py \
  --hf-ckpt-path /path/to/hf/model \
  --torch-dist-ckpt-path /path/to/output \
  --target-model-type qwen3 \
  --target-tensor-model-parallel-size 1 \
  --target-pipeline-model-parallel-size 1
```

### 运行测试

```bash
# 运行所有测试
pytest tests/ -v

# 运行特定测试
pytest tests/test_qwen3_4B_ppo.py -v

# 使用标记过滤
pytest -m "unit"              # 仅单元测试
pytest -m "not skipduringci"  # 排除 CI 跳过的测试
```

### 代码质量检查

```bash
# 安装 pre-commit hooks
pre-commit install

# 运行检查
pre-commit run --all-files --show-diff-on-failure --color=always
```

## 关键目录结构

```
slime/
├── train.py / train_async.py    # 训练入口
├── slime/                       # 核心代码包
│   ├── backends/               # 训练后端(megatron_utils, fsdp_utils, sglang_utils)
│   ├── ray/                    # Ray 分布式管理(rollout.py 是核心)
│   ├── rollout/                # 数据生成层(sglang_rollout.py, generate_hub, rm_hub)
│   ├── router/                 # 路由层(middleware_hub)
│   └── utils/                  # 工具模块(arguments.py 是关键,74KB)
├── slime_plugins/              # 插件系统(mbridge, megatron_bridge)
├── scripts/
│   ├── models/                 # 模型配置文件(29个模型)
│   └── run-*.sh                # 各模型训练启动脚本
├── tools/                      # 权重转换工具
└── examples/                   # 示例集合(20+个)
```

## 参数系统

slime 参数分为三类:

1. **Megatron 参数**: 直接传递,如 `--tensor-model-parallel-size 2`
2. **SGLang 参数**: 必须加 `--sglang-` 前缀,如 `--sglang-mem-fraction-static`
3. **slime 特定参数**: 在 `slime/utils/arguments.py` 中定义

**脚本参数分组:**
- `MODEL_ARGS`: 模型架构参数
- `CKPT_ARGS`: 检查点和路径配置
- `ROLLOUT_ARGS`: 数据生成配置(prompt 数据, batch size, 采样策略)
- `EVAL_ARGS`: 评估配置
- `PERF_ARGS`: 性能优化配置(并行策略, activation checkpointing)
- `GRPO_ARGS`: GRPO/RL 算法参数
- `OPTIMIZER_ARGS`: 优化器配置

## 核心数据结构

定义在 `slime/utils/types.py`:

- **Sample**: 表示生成的样本,包含 prompt, response, reward, log_probs 等
- **RolloutBatch**: 从 rollout 到训练的批处理数据
- **MultimodalType**: 多模态类型支持(image, video, audio)

## 技术栈

- **Python**: 3.10+ (推荐 3.12)
- **PyTorch**: 2.9.1
- **CUDA**: 12.9/12.1
- **Ray**: 分布式执行框架
- **SGLang**: 高性能推理引擎(特定 commit: 24c9100)
- **Megatron-LM**: 训练后端(特定 commit: 3714d81)

## 编码规范

- **行长度**: 119 (black/isort), 320 (ruff)
- **Python 版本**: 3.10+
- **导入顺序**: FUTURE, STDLIB, THIRDPARTY, FIRSTPARTY, LOCALFOLDER
- **第一方包**: slime, slime_plugins
- **第三方包**: megatron, wandb, ray, transformers

## 扩展点

- **自定义 Reducer**: 继承并在 `examples/` 中实现,如 `examples/DrGRPO/custom_reducer.py`
- **自定义 Rollout**: 继承 `slime/rollout/sglang_rollout.py`
- **自定义 RM**: 在 `slime/rollout/rm_hub/` 中添加
- **插件系统**: `slime_plugins/` 目录

## 重要注意事项

1. **版本依赖**: SGLang 和 Megatron-LM 使用特定 commit,不是最新版本
2. **配置匹配**: 模型配置参数(如 `--rotary-base`)必须与模型完全匹配
3. **性能优化**:
   - 支持 colocation 模式(训练和推理在同一 GPU)
   - 支持 offloading (CPU offloading)
   - 支持 NVLink 检测和优化
4. **硬件支持**:
   - NVIDIA H100/H200 (官方支持,有 CI 保护)
   - NVIDIA B200 系列 (支持,无 CI 保护)
   - AMD GPU (通过单独的教程)
