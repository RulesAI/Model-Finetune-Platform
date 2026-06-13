# CLAUDE.md - Model-Finetune-Platform 开发规范

本文件定义 Model-Finetune-Platform 项目的技术架构与开发约束。

---

## 一、项目严格定位（v3 收窄版）

**纯大模型微调工具集** —— 让算法工程师把"行业素材 → 微调好的模型 + API"端到端跑通。

### 用户角色

| 角色 | 关系 |
|------|------|
| **算法工程师 / 实施工程师** | ⭐ 本工具集的直接用户 |
| 下游业务系统（PayLens / Nexus / GraphForge / Agent）| 消费工具集产出的模型 API |
| 业务方（HR 部门 / 制造车间 / 供应链运营）| 不直接接触本工具集，用下游业务系统 |

### 边界严守（不做的事）

| ❌ 不做 | 归属 |
|--------|------|
| RAG / 知识图谱 | **GraphForge 项目** |
| 业务系统集成 / Agent 编排 | 下游业务系统 |
| 数据标注平台 | 集成 Argilla |
| 基础模型预训练 | 不在范围 |
| 多模态训练 | v2+ 再考虑 |

**为什么严格定位**：5 周交付首发场景、3 周交付后续场景。范围越广越做不快。

---

## 二、两层架构（关键设计原则）

```
通用工具集（领域无关）            场景插件（领域特定）
├── scripts/                       scenes/<scene>/
├── backend/                       ├── prompts/
├── frontend/                      ├── instructions/
└── configs/<tool>/_base.yaml      ├── eval_set/
                                   └── system_prompts/
                                  +
                                  configs/<tool>/scenes/<scene>/<cfg>.yaml
```

详细架构说明：`docs/00-项目定位与多领域路线图.md`

---

## 三、技术栈（v3 收窄版）

### 训练侧（Python 3.10+）

| 阶段 | 主框架 | 备选 |
|------|--------|------|
| SFT / DPO | **ms-swift v4.0** ⭐ | LLaMA-Factory |
| 数据工程 | Distilabel + MarkItDown + Presidio | DataDreamer |
| 评估 | OpenCompass + DeepEval + QualBench | - |
| 基座（主）| **GLM-4-9B-Chat-1M** | Qwen2.5-14B（Plan B）|

### 服务侧

| 组件 | 技术栈 |
|------|--------|
| 后端 | FastAPI + asyncpg / SQLite |
| 前端 | Next.js 14（App Router）+ Ant Design 5 + Zustand |
| 推理（生产）| **LMDeploy** ⭐（涉密单机首选）|
| 推理（dev）| vLLM |

### 已移除（v2 → v3 收窄）

| 工具 | 移除原因 | 现在归属 |
|------|---------|---------|
| RAGFlow | RAG 不属于"微调" | GraphForge |
| LightRAG | RAG 不属于"微调" | GraphForge |
| BGE-M3 / BGE-reranker | Embedding/Rerank 服务 RAG，不属于"微调" | GraphForge |
| RAGAS | 评估 RAG，与本项目无关 | GraphForge |

### v3 收窄记录

| # | v2 → v3 变更 | 原因 |
|---|-----------|------|
| **R1** | 删除 Phase 4 RAG | RAG 划给 GraphForge，本项目不做 |
| **R2** | Phase 4 评估（原 Phase 5）合并 | 不再需要 RAGAS |
| **R3** | 部署阶段编号 Phase 5（原 Phase 6）| 6 阶段 → 5 阶段 |
| **R4** | 用户角色从"业务方"严格收窄为"算法工程师/实施工程师" | 工具集是内部能力 |
| **R5** | 交付物从"四件套"调整为"三件套" | 移除 RAG 服务 |

详细：`docs/02-工具栈选型-v3.md`

---

## 四、端口约定（不可更改）

| 端口 | 服务 |
|------|------|
| `3006` | 前端管理面板（Next.js） |
| `8006` | 主后端 API（FastAPI，对外唯一入口）|
| `8016` | 推理服务（LMDeploy / vLLM，OpenAI 兼容接口）|

> 已移除 `8086 RAGFlow`（v2 → v3 收窄）。RAG 服务由 GraphForge 提供，端口归 GraphForge 管理。

---

## 五、目录约定（核心架构）

```
docs/        # 项目文档（先读 docs/00-...）
scenes/      # 场景插件（每个领域一个子目录）
configs/     # 工具配置（<tool>/scenes/<scene>/）
data/        # 训练数据（按 scene 拆）
scripts/     # 通用脚本（接受 --scene 参数）
checkpoints/ # 模型 checkpoint（按 scene 拆，gitignore）
logs/        # 日志（按 scene 拆，gitignore）
frontend/    # Next.js 管理面板
backend/     # FastAPI 编排层（scene 为一等公民）
tests/       # 工程测试
```

### 强制约束

| ✅ 必须 | ❌ 禁止 |
|--------|--------|
| 所有 scripts 接受 `--scene <name>` 参数 | 在 scripts/ 内硬编码场景 |
| prompts 放 `scenes/<scene>/prompts/*.jinja` | 把 prompt 硬编码到 Python |
| 训练 config 放 `configs/<tool>/scenes/<scene>/` | configs 内不分场景 |
| 评估集放 `scenes/<scene>/eval_set/` | 评估集放 tests/ |
| checkpoint 路径 `checkpoints/<scene>/<run_id>/` | 平铺到根 checkpoints/ |
| data/raw/ 用软链接客户素材 | 把客户素材复制进 git |
| **凡是涉及 RAG / 知识图谱的需求，转告 GraphForge** | 在本项目实现 RAG |

---

## 六、数据安全约束（涉密场景必读）

1. **客户原始素材永不出本地**：训练、评估、推理全部本地完成
2. **API 蒸馏数据脱敏**：调用外部 API 做数据增广时，仅传通用问答模板，**不传客户脱敏案例原文**
3. **训练数据二次脱敏**：Phase 1 完成后用 Presidio 二次扫描，人工抽查 5% 样本
4. **Checkpoint 不共享**：训练产物只在本地与客户内网间流转，**不上传任何公网仓库**
5. **多场景隔离**：不同客户场景的 checkpoint、数据、日志按 `<scene>/<client>` 隔离

---

## 七、5 阶段开发流程（通用 + 场景实例）

每个 Phase 完成必须出**指标报告**，不允许"凭感觉好"放行下一 Phase。

| Phase | 通用工具 | 场景特定输入 | 评估指标 |
|-------|---------|------------|---------|
| 1. 数据工程 | Distilabel + Presidio | `scenes/<scene>/prompts/` + `instructions/rules.yaml` | 人工抽检 100 条 ≥ 4/5 |
| 2. SFT | ms-swift | 数据 + 基座 + `configs/ms_swift/scenes/<scene>/sft.yaml` | ROUGE-L / 自评 + C-Eval 回归 |
| 3. DPO | ms-swift | SFT 模型 + 偏好对 | win-rate vs SFT ≥ 60% |
| 4. 评估 | OpenCompass + QualBench + DeepEval | `scenes/<scene>/eval_set/` | 通用能力掉点 < 3% |
| 5. 部署 | LMDeploy | 模型 + `configs/lmdeploy/scenes/<scene>/serve.yaml` | P95 延迟 < 3s / 24h 不掉 |

---

## 八、新增场景流程

新增场景必须按 `docs/03-场景扩展指南.md` 6 步执行。**禁止**：
- 不填数据画像 yaml 就开始填代码
- 不写 eval_set 就上线
- 改 scripts/ 通用代码以适配新场景（应改 scenes/ 或 configs/）

---

## 九、开发约束（继承全局 + 项目特殊）

### 继承 `~/.claude/rules/`
- `code-quality.md`：TypeScript 严格类型 / Python 类型注解 / 函数 < 50 行
- `workflow.md`：原子 commit / 测试后 commit / 修改前读上下文
- `security.md`：参数化 SQL / 不硬编码 secret
- `performance.md`：异步 I/O / 批处理 / 设超时

### 项目特殊
- **每个训练任务必须可追溯**：configs/ 下的 YAML + git commit hash + 数据版本 + 场景名，四者绑定
- **每个 checkpoint 必须有评估报告**：放在 `logs/eval/<scene>/<checkpoint_id>.md`
- **训练日志必须 wandb / swanlab 双备份**
- **scene 是一等公民**：所有 API 调用、所有脚本、所有 config 必须区分 scene
- **不要做 RAG 相关代码**（用户来要求时，转告 GraphForge）

---

## 十、与其他项目的关系

| 项目 | 协同模式 |
|------|---------|
| **GraphForge** | ⭐ 消费方：GraphForge 调用本项目模型 API 作 LLM 后端；本项目不调用 GraphForge |
| **HR PayLens** / **HR Knowledge Nexus** | 消费方：调用本项目模型 API |
| **Supply Nexus Agent** | 未来 supply-chain 场景启动后的消费方 |
| **EvoChain-Agent** | 未来 supply-chain 场景的 Agent 编排消费方 |

**所有项目都是本工具集的"下游消费方"**，本项目不依赖任何业务项目。

---

## 十一、提交规范

```
feat(phase1): 添加 Distilabel 数据增广 pipeline
feat(scene-hr): 新增 HR-salary 政策类 prompt 模板
fix(training): 修复 LoRA 配置中的 target_modules 错误
docs: 更新工具栈选型 v3（收窄移除 RAG）
chore(config): 调整 ms-swift 学习率
refactor: 移除 RAG 相关代码（划给 GraphForge）
```

Phase 标签：`phase1` ~ `phase5` / `scene-<name>` / `infra` / `docs` / `eval`
