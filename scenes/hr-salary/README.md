# Scene: HR-Salary（国企薪酬）

> **状态**：🚧 搭建中（首发场景）
> **启动时间**：2026-05-25
> **预计上线**：2026-Q3（5 周交付，v3 收窄后）
> **基座模型**：GLM-4-9B-Chat-1M
> **数据规模**：158 md / ~140 万 token

---

## 一、场景定位

为**国企（央企 + 省属国企）**薪酬政策解读、方案设计提供 AI 助手的**基础模型能力**。

### 注意：场景产出 ≠ 端到端业务系统

| 本场景产出（在本工具集范围内）| 由下游业务系统产出（不在本工具集范围）|
|---------------------------|------------------------------------|
| 微调后的 GLM-4-9B 模型 | HR 部门用的对话 UI |
| OpenAI 兼容 API（端口 8016）| 政策条款精确引用（RAG 兜底）|
| System Prompt 模板 | 业务流程整合（如自动生成方案 Word 文档）|
| 评估报告 | 与企业 HRIS 系统集成 |

### 下游消费方

- **HR PayLens**（薪酬报告撰写）
- **HR Knowledge Nexus**（HR 知识问答 + 合规）
- **GraphForge**（作为 RAG 服务的 LLM 后端，配合政策图谱）

---

## 二、数据来源

| 维度 | 数值 |
|------|------|
| 文件总数 | **158 个 markdown** |
| 总字节数 | 6,302,340 bytes ≈ 6.3 MB |
| 估算 token 数 | 约 140 万 token |
| 数据路径 | `data/raw/HR-薪酬素材/` → `_HR/文档素材/已处理素材/` |

### 主题结构

| 子目录 | 大小 | 内容 |
|--------|------|------|
| `国企薪酬制度脱敏案例/` | 1.4 MB | 央企 + 省属国企实际薪酬方案 |
| `国有企业工资分配政策汇编/` | 1.3 MB | 6 大类政策法规 |
| `薪酬主题书籍与政策/` | 3.6 MB | 4 本专业书籍 + 案例集 + 政策文件 |

---

## 三、场景特征

| 维度 | 取值 | 影响 |
|------|------|------|
| 多模态需求 | 否（纯文本）| 用 GLM-4-9B 文本模型 |
| 涉密等级 | 高（已脱敏，需二次校验）| 数据本地化 + Presidio 校验 |
| 实时性要求 | 中（小时级响应即可）| LMDeploy 量化部署足够 |

> **v3 移除字段**：~~引用网络密度~~ —— 该维度由下游业务系统通过 GraphForge RAG 解决，本项目不评估。

---

## 四、本场景目录结构

```
scenes/hr-salary/
├── README.md             # 本文件
├── prompts/              # Distilabel 数据增广用的领域 prompt
│   ├── qa.jinja
│   ├── generation.jinja
│   └── extraction.jinja
├── instructions/         # SFT 指令构造规则
│   ├── rules.yaml
│   └── seed_examples.jsonl
├── eval_set/             # 业务评估集
│   ├── benchmark.jsonl
│   └── golden.jsonl
└── system_prompts/       # 推理 system prompt
    └── default.md
```

---

## 五、本场景工具配置（v3 收窄后）

| 工具 | 配置路径 |
|------|---------|
| 训练（主）| `configs/ms_swift/scenes/hr-salary/` |
| 训练（备）| `configs/llama_factory/scenes/hr-salary/` |
| 数据增广 | `configs/distilabel/scenes/hr-salary/` |
| 评估 | `configs/opencompass/scenes/hr-salary/` |
| 部署 | `configs/lmdeploy/scenes/hr-salary/` |

> **v3 移除**：~~configs/ragflow~~ / ~~configs/lightrag~~ 已删除，RAG 归 GraphForge。

---

## 六、HR-Salary 指令分布

```yaml
distribution:
  qa: 0.50                # 政策问答（学领域语言）
  generation: 0.30        # 方案生成（学结构化输出）
  extraction: 0.20        # 信息抽取（简历 → 结构化）
```

> **v3 关键**：政策条款的**精确数字 / 编号**不依赖微调记忆，依赖下游 RAG 检索。本项目的 QA 训练目标是"问对类型 + 答出框架 + 引用形式正确"，不是"记住具体数字"。

---

## 七、HR-Salary 评估特色

| benchmark | 抽样量 | 通过线 |
|-----------|-------|--------|
| C-Eval（通用）| 200 题 | 比 base 掉点 < 3% |
| CMMLU（通用）| 200 题 | 比 base 掉点 < 3% |
| **QualBench HR 资格类** ⭐ | 100 题 | **≥ 80%**（人力资源管理师二级通过线）|
| 业务测试集 | 200 条 | 人工评 ≥ 4.0/5 |

> **v3 移除**：~~RAGAS 四件套~~（评估 RAG 归 GraphForge 与下游业务系统）。

---

## 八、HR-Salary 进度追踪（v3 调整）

- [x] 数据画像完成
- [x] 场景目录骨架建立
- [ ] **Step 3.1** 编写 prompts/*.jinja（W1）
- [ ] **Step 3.2** 编写 instructions/rules.yaml + 50 条种子样例（W1）
- [ ] **Step 3.3** 业务方提供 200 条 eval_set/benchmark.jsonl（W2）
- [ ] **Step 3.4** 编写 system_prompts/default.md（W1）
- [ ] **Step 4** 工具配置（ms-swift / opencompass / lmdeploy）（W2）
- [ ] **Step 5** 跑通 5 阶段流水线（W3-W5）
- [ ] **Step 6** 评估 + API 部署 + 下游消费方对接（W5）

详细时间表：`docs/01-训练方案与时间计划-v3.md` §五。

---

## 九、与 GraphForge 协同（端到端 HR AI 助手）

当业务方需要"完整 HR AI 助手"时，本场景的产出**只是其中一部分**：

```
本场景产出（5 周）：HR 国企薪酬微调模型 + API
                            │
                            ▼
GraphForge（与本项目协同）：HR 政策图谱 + LightRAG 服务，调本场景 API 作 LLM 后端
                            │
                            ▼
HR PayLens / Nexus（业务集成）：UI + 业务流程 + 报告生成
                            │
                            ▼
最终客户产品：国企 HR 智能助手
```

**本场景**的所有承诺仅限"模型 + API + 评估"。RAG / UI / 业务流程**不是本场景的工作**。
