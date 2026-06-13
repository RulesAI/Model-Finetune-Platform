# 09-Mac 本机试验指南

> **创建**：2026-05-25
> **用途**：使用 MacBook（Apple Silicon）本机进行微调试验，零成本验证工具集
> **三件套地位**：[06-涉密内网部署](./06-涉密内网部署指南.md)（涉密交付）/ [07-云训练资源选型](./07-云训练资源选型与成本对比.md)（已脱敏上云）/ **09-Mac 本机**（纯本地试验）
> **关联**：[02-工具栈-v4](./02-工具栈选型-v4.md) / [08-基座模型规模化指南](./08-基座模型规模化指南.md)
> **核心结论**：Mac 上跑微调**完全可行**，工具集 80% 复用 + 20% 用 MLX-LM 替代 CUDA 栈

---

## 一、本文档的定位

### 1.1 三种部署路径的对照

```
┌──────────────────────────────────────────────────────────────────┐
│           Model-Finetune-Platform 三种实施路径                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  09 Mac 本机（本文档）   │  07 云训练（AutoDL/PAI）│ 06 涉密内网    │
│  ─────────────────────  │  ────────────────────  │  ───────────  │
│  • 零成本               │  • ¥1500-2500 / 5 周    │ • ¥180-280 万 │
│  • 纯本地（零数据风险） │  • 已脱敏上云           │ • 物理隔离    │
│  • 小模型 1B-7B         │  • prod 全规模          │ • prod 全规模 │
│  • 工程师 onboarding    │  • 业务 POC / 试点      │ • 商业交付    │
│  • smoke test           │  • 中等成熟度           │ • 高成熟度    │
│  ⭐ 用 MLX-LM 替代       │  ⭐ 工具栈 100% 一致     │ ⭐ 离线包部署  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

  Mac 试验 → AutoDL POC → 涉密内网    ← 完整演进路径
```

### 1.2 Mac 试验的独特价值

| 价值维度 | 说明 |
|---------|------|
| **零成本** | 不用充值任何云账号 / 不消耗算力预算 |
| **零数据风险** | 数据 100% 本地，无任何上云/协议风险 |
| **工程师 onboarding** | 新人 1-2 天就能跑通完整 5 阶段，理解工具集设计 |
| **工具集 smoke test** | 验证 pipeline 设计合理性，跑过一遍 prod 路径就清楚 |
| **业务 demo 草稿** | 即使 3B 小模型，可拿来给业务方看"大致效果"|
| **持续可用** | Mac 在桌上一直能跑，云资源还得抢 / 启动 |
| **隐私敏感场景** | 即使数据未脱敏，Mac 试验是唯一合规选项 |

### 1.3 Mac 试验**不能**做什么

| 不能做 | 替代方案 |
|--------|---------|
| ❌ 跑 prod 同款 GLM-4-9B 完整 SFT（除非 128GB Mac）| 上 AutoDL（07）|
| ❌ 用 ms-swift 训练（CUDA only）| 用 MLX-LM 替代 |
| ❌ 用 LMDeploy 推理（CUDA only）| 用 mlx_lm.server 替代 |
| ❌ DeepSpeed 多卡训练 | Mac 单 SoC，单卡概念不存在 |
| ❌ Unsloth 加速 | MLX 自带优化 |

---

## 二、硬件需求与能力

### 2.1 Apple Silicon 推荐档位

| 芯片 | 推荐内存 | 可训练模型上限 | 推理速度（tok/s）|
|------|---------|--------------|--------------|
| M3 / M4 | 16-24 GB | Qwen2.5-1.5B / 3B（LoRA）| ~30-50 |
| M3 Pro / M4 Pro | 24-48 GB | Qwen2.5-7B（LoRA 紧）| ~50-80 |
| **M3 Max / M4 Max** ⭐ | **64-128 GB** | Qwen2.5-14B / GLM-4-9B（LoRA） | **~80-120** |
| M3 Ultra / M4 Ultra | 128-256 GB | GLM-4-32B（QLoRA）| ~100-150 |

### 2.2 Mac 微调可行性矩阵

| 模型 | 全参 SFT | LoRA | QLoRA | 推理（FP16）| 推理（INT4 量化）|
|------|---------|------|-------|------------|---------------|
| Qwen2.5-1.5B | ✅ 16GB | ✅ 8GB | ✅ 4GB | ✅ 4GB | ✅ 1GB |
| **Qwen2.5-3B** ⭐ 甜点 | ✅ 24GB | ✅ 12GB | ✅ 6GB | ✅ 8GB | ✅ 2GB |
| Qwen2.5-7B | ⚠️ 48GB（紧） | ✅ 24GB | ✅ 12GB | ✅ 14GB | ✅ 4GB |
| **Qwen2.5-14B** | ❌ 96GB | ✅ 48GB | ✅ 24GB | ✅ 28GB | ✅ 8GB |
| GLM-4-9B-Chat-1M | ⚠️ 64GB（紧） | ✅ 32GB | ✅ 16GB | ✅ 18GB | ✅ 5GB |
| Qwen2.5-32B | ❌ | ⚠️ 100GB | ✅ 50GB | ⚠️ 64GB | ✅ 18GB |

**所需总内存 ≈ 上述显存 × 1.5**（系统 + 训练 buffer）。

### 2.3 你的 Mac 能跑什么？

基于 memory 推断（你的 Mac 跑过 MLX-LM @11435 实测 105 tok/s）→ 至少 M3/M4 Max 64GB：

**强烈推荐**：从 **Qwen2.5-3B**（甜点）→ **Qwen2.5-7B**（充分试验）。

128GB Mac 可挑战 Qwen2.5-14B 或 GLM-4-9B-Chat-1M（HR 同款基座）。

---

## 三、工具栈替代（v4 工具集 + MLX 适配）

### 3.1 工具栈对照表

| Phase | v4 prod 工具（CUDA）| Mac 替代工具 | 兼容性 |
|-------|------------------|-------------|------|
| 1 数据工程 | MarkItDown | **MarkItDown**（同）| ✅ 100% |
| 1 数据工程 | Distilabel | **Distilabel**（同）| ✅ 100% |
| 1 数据工程 | Presidio | **Presidio**（同）| ✅ 100% |
| 2 SFT 训练 | ms-swift | **MLX-LM**（mlx_lm.lora）| ⚠️ 命令不同 |
| 2 加速 backend | Unsloth | 不需要（MLX 自带优化）| ⚠️ 概念变化 |
| 2 多卡分片 | DeepSpeed | 不需要（Mac 单 SoC）| ⚠️ 不适用 |
| 3 DPO | ms-swift swift rlhf | **MLX-LM**（mlx_lm.lora --use-dpo）| ⚠️ |
| 4 评估 | OpenCompass | OpenCompass（部分需 MPS）| ⚠️ 70% |
| 4 评估 | DeepEval | **DeepEval**（同）| ✅ 100% |
| 4 评估 | QualBench | QualBench 数据集 + 自跑 | ✅ |
| 4 人工评估 | Argilla | **Argilla**（Docker 本地）| ✅ 100% |
| 5 推理服务 | LMDeploy | **mlx_lm.server**（OpenAI 兼容）| ⚠️ 命令不同 |
| 5 量化 | LMDeploy INT4 AWQ | **MLX quantize**（INT4/INT8）| ⚠️ |
| 监控 | SwanLab / Prometheus | SwanLab 本地 | ✅ |

### 3.2 工具集兼容性总评

```
Mac 试验栈支撑度评估：

数据工程（Phase 1）：     ████████████████████  100%
SFT/DPO 训练（Phase 2/3）：████████████████░░░░   80%（换 MLX-LM）
评估（Phase 4）：         ███████████████░░░░░   75%（部分需替代）
推理部署（Phase 5）：     ████████████████░░░░   80%（换 mlx_lm.server）

综合：v4 工具集 Mac 兼容度 ~85%
（核心 pipeline 通，工具替换 5 处）
```

---

## 四、完整 5 阶段流水线（Mac 实操）

### 4.1 环境准备

```bash
# 1. 安装 Python 3.10（推荐 pyenv 或 conda）
brew install pyenv
pyenv install 3.10.13
pyenv local 3.10.13

# 2. 创建项目 venv
cd ~/GitHub_Source_Code/_AI训练/Model-Finetune-Platform
python -m venv venv-mac
source venv-mac/bin/activate

# 3. 安装 Mac 试验栈
pip install mlx mlx-lm                           # MLX 核心
pip install markitdown[all]                       # 文档解析
pip install distilabel                            # 数据增广
pip install presidio-analyzer presidio-anonymizer # 脱敏
pip install deepeval                              # 评估
pip install transformers peft datasets            # HF 生态（部分 MPS 支持）
pip install rouge-chinese                         # 中文 ROUGE

# 4. 验证 MLX 可用
python -c "import mlx.core as mx; print(mx.default_device())"
# 期望输出：Device(gpu, 0)
```

### 4.2 Phase 1 数据工程（与 prod 完全相同）

```bash
# 1.1 文档解析（如需，HR 158 md 已经是 md，跳过）
python scripts/data_prep/parse_documents.py \
  --input scenes/hr-salary/data/raw \
  --output data/processed/hr-salary

# 1.2 二次脱敏（F4 校正）
python scripts/data_prep/run_presidio.py \
  --scene hr-salary \
  --rules scenes/hr-salary/pii_rules/

# 1.3 指令构造（Distilabel + Claude API）
python scripts/data_prep/build_sft_dataset.py \
  --scene hr-salary \
  --teacher claude-4.6 \
  --output data/instructions/hr-salary/sft_v1_trial.jsonl \
  --samples 500   # Mac 试验用 500 条即可
```

⚠️ **HR Mac 试验数据合规**：客户数据即使脱敏，蒸馏到 Claude API 仍需业务方书面确认。如不便，**可跳过蒸馏**用纯 158 md 直接构造 instruction。

### 4.3 Phase 2 SFT 训练（MLX-LM 替代 ms-swift）

#### 4.3.1 模型下载

```bash
# MLX 社区已有 Qwen2.5 转换版（推荐）
python -m mlx_lm.convert \
  --hf-path Qwen/Qwen2.5-3B-Instruct \
  --mlx-path ~/.cache/mlx/Qwen2.5-3B-Instruct

# 或直接从 MLX 社区下载预转好的
huggingface-cli download mlx-community/Qwen2.5-3B-Instruct-4bit
```

#### 4.3.2 LoRA 训练

```bash
# 数据格式转换（HF jsonl → MLX 格式）
python scripts/training/hf_to_mlx_dataset.py \
  --input data/instructions/hr-salary/sft_v1_trial.jsonl \
  --output data/instructions/hr-salary/mlx/

# LoRA 训练
mlx_lm.lora \
  --train \
  --model ~/.cache/mlx/Qwen2.5-3B-Instruct \
  --data data/instructions/hr-salary/mlx/ \
  --train-type lora \
  --num-layers 16 \
  --batch-size 2 \
  --iters 1000 \
  --learning-rate 1e-5 \
  --adapter-path checkpoints/hr-salary/mac-trial-qwen3b/
```

#### 4.3.3 MLX 配置参数对照 prod

| 参数 | Mac MLX-LM | Prod ms-swift |
|------|-----------|--------------|
| 模型 | Qwen2.5-3B | GLM-4-9B-Chat-1M |
| 训练方法 | LoRA | LoRA |
| LoRA 层数 | --num-layers 16 | target_modules: q,k,v,o |
| Batch size | 2 | 8（per-device）|
| Learning rate | 1e-5 | 2e-4 |
| 训练步数 | 1000 iters | 3 epochs |
| 上下文长度 | 默认 | 8192 |

注：MLX 的 lr 通常比 ms-swift 低 1-2 个量级（实现差异）。

### 4.4 Phase 3 DPO（可选）

```bash
# DPO 偏好对训练
mlx_lm.lora \
  --train \
  --model checkpoints/hr-salary/mac-trial-qwen3b/ \
  --data data/preferences/hr-salary/mlx/ \
  --use-dpo \
  --dpo-cpo-loss-type sigmoid \
  --num-layers 16 \
  --iters 500 \
  --learning-rate 1e-6 \
  --adapter-path checkpoints/hr-salary/mac-trial-qwen3b-dpo/
```

### 4.5 Phase 4 评估

```bash
# 4.1 F2 校正：base 模型 baseline
python scripts/evaluation/baseline.py \
  --scene hr-salary \
  --model ~/.cache/mlx/Qwen2.5-3B-Instruct \
  --benchmarks business_test \
  --output logs/eval/hr-salary/mac-trial-baseline.md

# 4.2 微调后模型评估
python scripts/evaluation/run_eval.py \
  --scene hr-salary \
  --model ~/.cache/mlx/Qwen2.5-3B-Instruct \
  --adapter checkpoints/hr-salary/mac-trial-qwen3b-dpo/ \
  --benchmarks business_test \
  --output logs/eval/hr-salary/mac-trial-finetuned.md

# 4.3 对比报告
python scripts/evaluation/compare_reports.py \
  --baseline logs/eval/hr-salary/mac-trial-baseline.md \
  --finetuned logs/eval/hr-salary/mac-trial-finetuned.md
```

### 4.6 Phase 5 推理服务（mlx_lm.server 替代 LMDeploy）

```bash
# 5.1 合并 LoRA 权重（可选）
mlx_lm.fuse \
  --model ~/.cache/mlx/Qwen2.5-3B-Instruct \
  --adapter-path checkpoints/hr-salary/mac-trial-qwen3b-dpo/ \
  --save-path checkpoints/hr-salary/mac-trial-qwen3b-merged/

# 5.2 量化（INT4 / INT8）
mlx_lm.convert \
  --hf-path checkpoints/hr-salary/mac-trial-qwen3b-merged/ \
  --mlx-path checkpoints/hr-salary/mac-trial-qwen3b-int4/ \
  -q --q-bits 4

# 5.3 启动 OpenAI 兼容服务
mlx_lm.server \
  --model checkpoints/hr-salary/mac-trial-qwen3b-int4/ \
  --port 8016 \
  --host 0.0.0.0

# 5.4 测试
curl http://localhost:8016/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "央企工资总额备案制和核准制有什么区别？"}
    ]
  }'
```

---

## 五、HR 场景具体 walkthrough

### 5.1 完整试验时间表（M4 Max 64GB / Qwen2.5-3B）

```
Day 1（半天）：环境准备 + 数据
├── 0-1h:  装 MLX-LM + Qwen2.5-3B-Instruct 下载（4GB）
├── 1-2h:  HR 158 md 转 instruction 数据（200 条）
└── 2-4h:  数据格式转 MLX

Day 2（一天）：SFT 训练
├── 9:00-13:00:  mlx_lm.lora 训练 1000 iters
├── 13:00-14:00: 午休（训练继续）
└── 14:00-17:00: 评估 + 推理测试

Day 3（半天）：服务化 + Demo
├── 0-1h:  合并 + INT4 量化
├── 1-2h:  mlx_lm.server 启动
└── 2-4h:  写 demo 网页 + 业务方演示
```

**总投入：2.5-3 天**（一个工程师全程跟）

### 5.2 训练后产出

| 产出物 | 大小 | 用途 |
|--------|------|------|
| LoRA adapter | ~50-100 MB | 可挂载到 base 模型 |
| INT4 量化模型 | ~1.5-2 GB | 部署用 |
| 评估报告 | ~10 KB | baseline vs finetuned 对比 |
| Demo API 端点 | `http://localhost:8016/v1/...` | 业务方演示 |

### 5.3 业务方 demo 注意事项

- 3B 模型质量**显著弱于** 9B prod 版本（不要承诺）
- demo 目的是验证"**流程能跑通 + 大致方向对**"，不是展示**最终效果**
- 强调"这是 Mac 试验版，prod 会用更大模型"
- 可附带 Qwen2.5-7B 试验（如内存允许）对比效果

---

## 六、与 Prod 流水线的迁移路径

### 6.1 配置文件对应关系

```
configs/
├── ms_swift/scenes/hr-salary/          ← prod 用
│   └── sft_glm4_9b.yaml
└── mlx_lm/scenes/hr-salary/            ← Mac 用（新增）
    └── sft_qwen3b.yaml
```

### 6.2 命令对照

```bash
# Mac 试验
mlx_lm.lora --train --model Qwen2.5-3B --data ...

# Prod (AutoDL / 涉密内网)
swift sft --model_id_or_path glm-4-9b-chat-1m --dataset ...
```

### 6.3 数据 / Prompt / 评估集**完全复用**

| 资产 | Mac 试验 | Prod |
|------|---------|------|
| `scenes/hr-salary/prompts/*.jinja` | ✅ 同 | ✅ 同 |
| `scenes/hr-salary/instructions/rules.yaml` | ✅ 同（只是 samples 较少）| ✅ 同 |
| `scenes/hr-salary/eval_set/benchmark.jsonl` | ✅ 同 | ✅ 同 |
| `scenes/hr-salary/system_prompts/default.md` | ✅ 同 | ✅ 同 |

**意义**：Mac 试验积累的 prompt / 数据规则 / 评估集**可无缝迁移到 prod**，不浪费。

### 6.4 迁移到 Prod 的步骤

```
1. Mac 试验跑通（2-3 天）
   ↓
2. 收集真实踩坑点 → 更新 docs/
   ↓
3. 数据规则 / Prompt / 评估集打磨成熟
   ↓
4. 客户启动信号到 → 上 AutoDL 或涉密内网
   ↓
5. 复用 scenes/ 全部内容
   ↓
6. 切换 configs/ms_swift（替代 configs/mlx_lm）
   ↓
7. Prod 训练（GLM-4-9B 全量数据）
```

**核心收益**：Mac 试验阶段 80% 的工作（数据、prompt、评估、流程）**直接迁移**到 prod。

---

## 七、典型试验场景

### 7.1 工程师 onboarding

| Day | 任务 |
|-----|------|
| Day 1 | Read docs/ 00-09 + 装环境 + 跑通推理 |
| Day 2 | 跑通完整 5 阶段流水线（用 Mac + Qwen2.5-3B）|
| Day 3 | 理解项目架构 + 提交第一个改进 |

**1 周内**新工程师即可独立维护工具集，且对所有 docs 有实操理解。

### 7.2 工具集 smoke test

每次 v4 → v5 工具栈升级时，**先在 Mac 上跑一遍**验证流水线不破坏，再正式 release。

### 7.3 业务 demo 草稿

业务方提"想看看效果" → Mac 上 2-3 天出 v0.1，先看大方向是否对，再投入 prod 资源。

### 7.4 隐私敏感场景

如客户数据**未脱敏 + 严禁上云**，Mac 是唯一合规选项（但 prod 训练仍需上客户内网，见 06）。

### 7.5 工具集版本升级回归

ms-swift / MLX-LM 升级后，Mac 上跑一遍历史 benchmark 看是否退化。

---

## 八、性能预期（M4 Max 实测推算）

### 8.1 训练速度

| 模型 | 数据量 | 训练时长（1 epoch ≈ 1000 iters）|
|------|-------|------------------------------|
| Qwen2.5-1.5B | 500 条 | **30-60 分钟** |
| Qwen2.5-3B | 500 条 | **1-2 小时** |
| Qwen2.5-7B | 500 条 | **3-5 小时** |
| Qwen2.5-14B | 500 条 | 8-12 小时 |
| GLM-4-9B-Chat-1M | 500 条 | 6-10 小时 |

### 8.2 推理速度（INT4 量化）

| 模型 | 单条响应延迟 | 吞吐 |
|------|-----------|------|
| Qwen2.5-3B-INT4 | ~1-2s | ~100 tok/s |
| Qwen2.5-7B-INT4 | ~2-3s | ~70 tok/s |
| Qwen2.5-14B-INT4 | ~4-6s | ~40 tok/s |

参考：你的 Mac MLX-LM 实测 105 tok/s（user memory），与上表一致。

---

## 九、Mac 试验的局限

### 9.1 不能完整覆盖的场景

| 局限 | 影响 | 弥补方案 |
|------|------|---------|
| 不能跑 ms-swift / DeepSpeed | 训练框架不一致 | 上 AutoDL 跑 1 次 prod 工具验证 |
| 不能跑 LMDeploy INT4 量化 | 推理优化不同 | 推理部署阶段在 prod 测 |
| 不能多卡分布式训练 | 大模型训练上限低 | 大模型只能上 prod |
| OpenCompass 部分不可用 | C-Eval / CMMLU 评估受限 | 评估在 prod 跑一次 |
| 不能验证生产负载 | P95 延迟 / 并发不可测 | prod 部署时验证 |

### 9.2 误区警示

| ❌ 误区 | ✅ 正解 |
|--------|--------|
| Mac 试验通过 → prod 必然成功 | Mac 验证流程通，prod 需重新验证工具差异 |
| Qwen2.5-3B 效果好 = GLM-4-9B prod 效果一定好 | 3B 是流程验证用，效果不可外推 |
| Mac 上 demo 给客户看 = 最终交付物 | 这只是草稿，prod 才是交付 |

---

## 十、风险与避坑

| 风险 | 规避策略 |
|------|---------|
| MLX-LM 版本与模型不兼容 | 用 MLX 社区预转换模型（mlx-community/*）|
| 训练中途内存爆 | 降 batch_size 到 1 / 减少 LoRA 层数 |
| 训练 loss 不下降 | 检查学习率（MLX 通常 1e-5，不是 prod 的 2e-4）|
| 推理速度慢 | 启用 INT4 量化 |
| 数据格式 HF ≠ MLX | 用 scripts/training/hf_to_mlx_dataset.py 转 |
| MLX 部分算子不支持 | 切换 PyTorch + MPS backend（次选）|
| 误把 Mac 试验当 prod | 文档清晰标"Mac 试验 v0.x" |
| HR 数据上 Claude API 蒸馏 | 业务方书面确认，否则跳过蒸馏 |
| 试验完忘记清理 cache | `rm -rf ~/.cache/mlx/`（释放磁盘）|

---

## 十一、Mac 试验完整 Checklist

### 11.1 启动前

- [ ] Mac 内存 ≥ 24 GB（最低 Qwen2.5-3B）/ ≥ 64 GB（推荐 Qwen2.5-7B）/ 128 GB（Qwen2.5-14B / GLM-4-9B）
- [ ] 磁盘空间 ≥ 50 GB（模型 + 数据 + checkpoint）
- [ ] Python 3.10 venv 就位
- [ ] MLX-LM 已安装并跑通推理 smoke test
- [ ] 业务方确认（如使用 Claude API 蒸馏）

### 11.2 试验中

- [ ] Phase 1 数据工程产出 ≥ 200 条 instruction
- [ ] Phase 2 SFT loss 收敛
- [ ] Phase 3 DPO 完成（可选）
- [ ] Phase 4 评估 baseline + finetuned 对比报告
- [ ] Phase 5 mlx_lm.server 服务化跑通

### 11.3 试验后

- [ ] 写 demo 网页 / 录视频（给业务方）
- [ ] 把踩坑点更新到 docs/
- [ ] 数据规则 / Prompt / 评估集打磨
- [ ] **决策**：是否启动 prod 训练（07 上云 / 06 涉密内网）
- [ ] 清理本地 cache（如需）

---

## 十二、一句话摘要

**Mac 本机试验是工具集"三件套部署路径"的第一档（零成本本地）—— 用 MLX-LM 替代 ms-swift / LMDeploy（其他工具不变）、推荐 Qwen2.5-3B (甜点) 或 7B (充分试验)、2-3 天跑通完整 5 阶段流水线，价值在工程师 onboarding / 工具集 smoke test / 业务 demo 草稿，scenes/ 与 prompts 完全可迁移到 prod。**

---

## Sources

- [MLX 官方文档（Apple）](https://ml-explore.github.io/mlx/)
- [MLX-LM GitHub](https://github.com/ml-explore/mlx-lm)
- [MLX Community Hugging Face 模型](https://huggingface.co/mlx-community)
- [Qwen2.5 系列 (HF + MLX Community)](https://huggingface.co/Qwen)
- [MLX-LM LoRA 微调指南](https://github.com/ml-explore/mlx-lm/blob/main/mlx_lm/LORA.md)
- [Apple Silicon LLM 微调实战](https://medium.com/@ishaafsalman/fine-tuning-qwen-qwen3-vl-30b-a3b-moe-architecture-with-lora-2365359e870f)
