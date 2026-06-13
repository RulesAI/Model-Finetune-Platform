# Model-Finetune-Platform

> **纯大模型微调工具集** —— 让算法工程师把"行业素材 → 微调好的模型 + API"端到端跑通
> 架构：**通用工具集 × 场景插件**两层
> 用户：**算法工程师 / 实施工程师**（内部技术团队，不是业务方）
> 当前场景：HR 国企薪酬（首发）；规划：生产制造、供应链

---

## 一、项目严格定位

### 1.1 是什么

把"**让 LLM 学会某行业语言/风格/推理模式**"这件事工程化：从素材到一个可调用 API 的微调模型，5 阶段流水线全自动化（数据 → SFT → DPO → 评估 → 部署）。

### 1.2 不是什么（边界严格）

| ❌ 不做 | 归属 |
|--------|------|
| **RAG / 知识图谱** | GraphForge 项目主业 |
| **业务系统集成 / Agent 编排** | 下游业务系统（PayLens / Nexus / SCI.AI / Agent 系列）|
| **数据标注平台** | 集成 Argilla，不重造 |
| **基础模型预训练** | 不在范围内 |
| **多模态训练**（v1）| v2+ 再说 |

**为什么严格定位**：项目要在 5 周交付首发场景、3 周交付后续场景。范围越广越做不快。

### 1.3 项目用户角色

```
内部用户（本工具集服务的对象）
├── 算法工程师 → 跑 6 阶段流水线，调超参，出模型
└── 实施工程师 → 用工具集为新客户/新场景快速复制能力

外部消费者（不是直接用户）
└── 下游业务系统 → 调用工具集产出的模型 API
    ├── HR PayLens / HR Knowledge Nexus
    ├── GraphForge（RAG 服务用本项目模型作 LLM 后端）
    ├── SCI.AI / Supply Nexus Agent
    └── 自建业务 Agent
```

业务方（HR 部门 / 制造车间 / 供应链运营）**不直接接触本工具集**，他们用的是下游业务系统。

详细定位 + 多领域路线图：[`docs/00-项目定位与多领域路线图.md`](docs/00-项目定位与多领域路线图.md)

---

## 二、技术栈（v3 收窄版）

```
┌────────────────────────────────────────────────────────────────────┐
│           通用工具集（领域无关，5 阶段流水线）                       │
├────────────────────────────────────────────────────────────────────┤
│  Phase 1  数据工程  │ MarkItDown + Distilabel + Presidio           │
│  Phase 2  SFT       │ ms-swift ⭐（国企首选）/ LLaMA-Factory（备选） │
│  Phase 3  DPO       │ ms-swift                                     │
│  Phase 4  评估      │ OpenCompass + DeepEval + QualBench           │
│  Phase 5  部署      │ LMDeploy ⭐（涉密单机）/ vLLM（dev/SaaS）      │
├────────────────────────────────────────────────────────────────────┤
│           场景插件（领域特定，scenes/<scene>/）                      │
├────────────────────────────────────────────────────────────────────┤
│  • 指令构造模板 (prompts/)                                          │
│  • 数据合成配方 (instructions/)                                     │
│  • 领域评估集 (eval_set/)                                           │
│  • 推理 system prompt (system_prompts/)                            │
└────────────────────────────────────────────────────────────────────┘
```

基座模型主选：**GLM-4-9B-Chat-1M**；Plan B：**Qwen2.5-14B-Instruct**（Apache 2.0）。

详细选型理由：[`docs/02-工具栈选型-v3.md`](docs/02-工具栈选型-v3.md)

---

## 三、最终交付物（"三件套"）

每个场景跑完 5 阶段后，本工具集向下游交付：

```
┌─────────────────────────────────────────────────┐
│   <场景> 微调模型交付包                          │
├─────────────────────────────────────────────────┤
│  ① 微调模型                                      │
│     - GLM-4-9B-Chat-1M + 场景 LoRA adapter       │
│     - LMDeploy INT4 量化版（可单机部署）         │
│                                                  │
│  ② 推理 API（OpenAI 兼容）                       │
│     - LMDeploy 服务化部署                        │
│     - 任意业务系统可直接调用                     │
│                                                  │
│  ③ 配套资产                                      │
│     - 评估报告（C-Eval / CMMLU / 领域 benchmark）│
│     - System Prompt 模板（推理时使用）           │
│     - 运维手册 + SLA                             │
└─────────────────────────────────────────────────┘
```

**RAG / 业务集成不在本工具集职责内** —— 下游业务系统自己接 GraphForge RAG 或自建。

---

## 四、当前场景与路线图

| 场景 | 状态 | 数据规模 | 触发时机 |
|------|------|---------|---------|
| **HR 国企薪酬** | 🚧 搭建中（首发） | 158 md / ~140 万 token | 2026-05 |
| **TCM 中医药** | 📋 数据清单就位（L1）| **~8-10 亿 token**（500-700× HR）| 2026-Q3 待资源/算力落定 |
| 生产制造 | 📋 规划中 | TBD | 2026-Q3 待客户落定 |
| 供应链 | 📋 规划中 | TBD | 2026-Q3-Q4 待客户落定 |

新场景如何接入：[`docs/03-场景扩展指南.md`](docs/03-场景扩展指南.md)

---

## 五、端口约定

| 端口 | 服务 | 说明 |
|------|------|------|
| `3006` | 前端管理面板（Next.js） | 训练任务管理 / 评估报告（多场景切换）|
| `8006` | 主后端 API（FastAPI） | 训练编排 / 数据管理 / 评估 |
| `8016` | 推理服务（LMDeploy 主 / vLLM 备） | OpenAI 兼容接口，对**下游业务系统**暴露 |

---

## 六、目录结构

```
Model-Finetune-Platform/
├── docs/                       # 项目文档
│   ├── 00-项目定位与多领域路线图.md    # 总览（先读）
│   ├── 01-训练方案与时间计划.md         # v1（保留审计）
│   ├── 01-训练方案与时间计划-v2.md      # v2（保留审计）
│   ├── 01-训练方案与时间计划-v3.md      # v3（保留审计）
│   ├── 01-训练方案与时间计划-v4.md      # v4 ⭐ 当前权威（F1-F4 落实 + Phase 0 CPT）
│   ├── 02-工具栈选型.md                # v1（保留审计）
│   ├── 02-工具栈选型-v2.md             # v2（保留审计）
│   ├── 02-工具栈选型-v3.md             # v3（保留审计）
│   ├── 02-工具栈选型-v4.md             # v4 ⭐ 当前权威（F1-F4 落实 + CPT 工具）
│   ├── 03-场景扩展指南.md
│   ├── 04-工具栈选型评估报告.md         # v3 时点深度评估 + 4 校正点 (F1-F4)
│   ├── 05-工具集成与依赖管理.md         # 7 种集成方式 + 5 层架构 + 涉密内网调整
│   ├── 06-涉密内网部署指南.md           # HR 客户 IT 部署手册 + 我方实施 SOP（涉密交付路径）
│   ├── 07-云训练资源选型与成本对比.md   # AutoDL / 智星云 / PAI 多平台对比（已脱敏 POC 路径）
│   ├── 08-基座模型规模化指南.md         # 1B-MoE 五档配置矩阵 + F7/F8 校正
│   └── 09-Mac本机试验指南.md            # ⭐ MLX-LM 本地试验（零成本 / 工程师 onboarding）
│
├── scenes/                     # 场景插件
│   ├── hr-salary/              # HR 国企薪酬（首发，5 阶段流水线）
│   ├── tcm/                    # 中医药（数据清单就位，6 阶段流水线含 CPT）
│   ├── manufacturing/          # 生产制造（占位）
│   └── supply-chain/           # 供应链（占位）
│       每个场景含: prompts/ / instructions/ / eval_set/ / system_prompts/
│
├── configs/                    # 工具配置（按工具 → 场景拆）
│   ├── ms_swift/scenes/<scene>/        # 训练
│   ├── llama_factory/scenes/<scene>/   # 备选训练
│   ├── distilabel/scenes/<scene>/      # 数据增广
│   ├── opencompass/scenes/<scene>/     # 评估
│   ├── lmdeploy/scenes/<scene>/        # 推理部署
│   └── vllm/scenes/<scene>/            # 备选推理
│   ↑ 注意：已无 ragflow/lightrag 子目录（v3 收窄后移除）
│
├── data/                       # 训练数据
├── scripts/                    # 端到端脚本（按 Phase 组织，--scene 参数化）
│   ├── data_prep/              # Phase 1
│   ├── training/               # Phase 2 + 3
│   ├── evaluation/             # Phase 4
│   └── deployment/             # Phase 5
├── checkpoints/<scene>/        # 模型 checkpoint（gitignore）
├── logs/<scene>/               # 训练 / 推理日志（gitignore）
├── frontend/                   # 自建管理面板
├── backend/                    # FastAPI 编排层
└── ...
```

---

## 七、Quick Start（HR 场景示例）

```bash
cd /Users/simon/GitHub_Source_Code/_AI训练/Model-Finetune-Platform

# 1. 数据准备（Phase 1）
python scripts/data_prep/build_sft_dataset.py --scene hr-salary --teacher claude-4.6

# 2. SFT 训练（Phase 2）
swift sft --config configs/ms_swift/scenes/hr-salary/sft_glm4_9b.yaml

# 3. DPO 对齐（Phase 3）
swift rlhf --rlhf_type dpo --config configs/ms_swift/scenes/hr-salary/dpo_glm4_9b.yaml

# 4. 评估（Phase 4）
python scripts/evaluation/run_eval.py --scene hr-salary --benchmarks ceval,cmmlu,qualbench

# 5. 部署（Phase 5）
lmdeploy serve api_server --config configs/lmdeploy/scenes/hr-salary/serve.yaml --port 8016

# 交付给下游：API 端点 http://<host>:8016/v1/chat/completions（OpenAI 兼容）
```

---

## 八、与 GraphForge 的协作边界（重要）

```
本项目                                  GraphForge
─────────                              ──────────
微调模型 + 推理 API           ◀────────  消费方（也是协作方）
                                       
                                       职责：
                                       • 知识图谱 + RAG 服务
                                       • LightRAG 集成层
                                       
                                       与本项目交互：
                                       • GraphForge 调用本项目模型 API 作为 LLM 后端
                                       • 本项目不调用 GraphForge（解耦）
```

**简言之**：本项目让 LLM **掌握**领域语言；GraphForge 让 LLM **看见**领域知识。两者通过"模型 API"接口对接，职责零重叠。

---

## 九、当前进度

- [x] 项目骨架（通用 + 场景插件 两层架构）
- [x] 设计文档全套（00 定位 / 01-v3 训练方案 / 02-v3 工具栈 / 03 场景扩展）
- [x] HR 场景占位结构
- [x] 制造 / 供应链 场景路线图占位
- [x] 严格收窄到"纯微调工具集"（v3 定位）
- [ ] **Phase 1**：HR-salary 数据工程脚本 + Distilabel pipeline
- [ ] **Phase 2**：ms-swift 环境 + GLM-4-9B-Chat-1M LoRA 配置
- [ ] **Phase 3**：DPO 偏好对构造
- [ ] **Phase 4**：OpenCompass + DeepEval + QualBench
- [ ] **Phase 5**：LMDeploy 部署 + 前端管理面板

---

## 十、关键设计原则（强约束）

1. **数据不出本地**：所有训练 / 评估在本地完成，不上云
2. **场景插件化**：新增场景不动核心代码
3. **职责严守**：不做 RAG、不做 Agent、不做业务集成
4. **每个 checkpoint 必跑通用回归**：C-Eval / CMMLU 抽 200 题，掉点 < 3% 才放行
5. **交付即用**：产出的 API 是 OpenAI 兼容，下游业务系统接入零成本

---

## 十一、文档导航

- **总览**：[docs/00-项目定位与多领域路线图.md](docs/00-项目定位与多领域路线图.md)
- **训练方案**：[docs/01-训练方案与时间计划-v4.md](docs/01-训练方案与时间计划-v4.md) ⭐ 当前权威（F1-F4 + CPT）
- **工具栈选型**：[docs/02-工具栈选型-v4.md](docs/02-工具栈选型-v4.md) ⭐ 当前权威（96% 评分）
- **场景扩展指南**：[docs/03-场景扩展指南.md](docs/03-场景扩展指南.md)
- **工具栈深度评估**：[docs/04-工具栈选型评估报告.md](docs/04-工具栈选型评估报告.md) ⭐ 综合评分 96% (v4)
- **工具集成与依赖管理**：[docs/05-工具集成与依赖管理.md](docs/05-工具集成与依赖管理.md) ⭐ 7 种集成方式 + 5 层架构（启动前必读）
- **涉密内网部署指南**：[docs/06-涉密内网部署指南.md](docs/06-涉密内网部署指南.md) ⭐ HR 客户 IT 部署手册 + 我方实施 SOP（涉密交付路径）
- **云训练资源选型**：[docs/07-云训练资源选型与成本对比.md](docs/07-云训练资源选型与成本对比.md) ⭐ AutoDL / 智星云 / PAI 多平台对比（已脱敏 POC 路径）
- **基座模型规模化指南**：[docs/08-基座模型规模化指南.md](docs/08-基座模型规模化指南.md) ⭐ 1B → MoE 五档配置矩阵 + F7（ZeRO-2 vs ZeRO-3）+ F8（超参规模化）
- **Mac 本机试验指南**：[docs/09-Mac本机试验指南.md](docs/09-Mac本机试验指南.md) ⭐ MLX-LM 本地试验（零成本 / 工程师 onboarding / 工具集 smoke test）
- **开发规范**：[CLAUDE.md](CLAUDE.md)
