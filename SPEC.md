# Auto Research Skill — Specification

## Overview

三阶段自动化研究流程：Deep Research → Coding & Experimenting → Writing & Review。

**执行模型**：整体流程由 Claude Code / Qwen Code 的 agent 自身驱动。Skills 为纯文本文档（遵循 [Agent Skills Spec](https://agentskills.io/llms.txt)），agent 读取 skill 内容作为行动指南，自行执行搜索、编码、写作等操作。Skill 不启动 agent，而是被 agent 参考。

**Skill 组织结构**：所有 skills 扁平存放（与主流 coding agent 的 skill 发现机制一致，按 name + description 匹配加载）。层级关系通过 skill 内容中的引用表达，而非目录嵌套：

- `auto-research`（顶层入口）：跨阶段调度、决策门、回退、版本管理
- `auto-research-s1-flow` / `auto-research-s2-flow` / `auto-research-s3-flow`（阶段 flow）：被 `auto-research` 调度，控制各自阶段内的子任务顺序和完成判断
- 其余子 skills：被阶段 flow 参考，指导具体操作

**每阶段执行逻辑**：Agent 参考阶段 flow skill 的指示：维护该阶段 TODO 文档 → 参考子 skills 完成子任务 → 判断阶段是否结束 → 交付阶段性产物。

---

## 0. 顶层 Skill：`auto-research`

### 0.1 职责

统领全部三阶段流程的入口 skill。Agent 激活此 skill 后，由其调度整个研究生命周期。

### 0.2 功能

1. **项目初始化**：根据用户输入的 topic，创建 `projects/<project_name>/` 目录及初始文档骨架
2. **阶段调度**：判断当前处于哪个阶段（读取各 stage progress 文档），加载对应阶段 flow skill
3. **决策门执行**：在阶段交界处暂停，向用户展示阶段性产物并请求确认
4. **回退管理**：当阶段 flow 报告失败/回退需求时，执行跨阶段回退逻辑
5. **版本管理**：管理 `topic_gap_idea.md` 的 living → frozen 转换
6. **全局状态追踪**：维护一个 `project_status.md` 记录整体进度

### 0.3 调度逻辑

```
用户输入 topic
    → 初始化项目目录
    → 加载 auto-research-s1-flow，执行 Stage 1
    → 决策门：用户确认 idea
    → 加载 auto-research-s2-flow，执行 Stage 2
    → 决策门：用户确认实验结果
    → 创建 topic_gap_idea_frozen.md
    → 加载 auto-research-s3-flow，执行 Stage 3
    → 交付最终产物

回退路径：
    Stage 2 idea 失败 → idea pool 内切换（不经过 auto-research）
    Stage 2 idea pool 耗尽 → 回退 auto-research → 重新加载 auto-research-s1-flow
    Stage 3 review 要求补实验 → 回退 auto-research → 加载 auto-research-s2-flow（定向补充模式）
```

### 0.4 维护文档：`project_status.md`

```markdown
# Project: [名称]

## Current Stage: Stage N
## Topic: [一句话]

## Stage History
- Stage 1: 完成 (2026-XX-XX) → 选定 idea: [名称]
- Stage 2: 进行中 / 回退 (原因: ...)
- Stage 3: 未开始

## Decision Log
- [日期] 用户确认 idea: ...
- [日期] 回退原因: ...
```

---

## 1. Stage 1: Deep Research

### 1.1 Input

1. 用户选定的 topic（一句话描述研究方向）
2. 可选：用户提供的初始关键词、已知相关论文、约束条件（目标会议、资源限制）

### 1.2 Output

1. `related_work.md` — 文献综述文档（按主题分节，每篇含标题、ID、年份、会议、核心方法、与本研究的关系）
2. `topic_gap_idea.md` — Topic & Gap 分析 & Idea 文档（结构见 §1.5）
3. `assets.md` — 预计使用的数据集 / benchmark / 模型链接文档（含下载方式）
4. `baselines.md` — Baselines 方法文档（方法名、论文链接、开源 repo、核心指标、复现难度评估）

### 1.3 抽象功能

Agent 参考主 flow skill（`auto-research-s1-flow`），自主执行以下工作：

1. 根据 topic 设计初始查询关键词/词组
2. 参考 `auto-research-s1-arxiv-search` skill 进行文献搜索，参考 `auto-research-s1-paper-analysis` skill 对高相关论文进行深度分析
3. 每轮搜索后：反思结果 → 更新 `topic_gap_idea.md` 中的 gap 分析 → 调整下一轮查询关键词
4. 搜索结束后：汇总 gap → 生成 positioning → 提出 3-5 个候选 idea → 可行性 & novelty 分析
5. 参考 `auto-research-s1-modelscope-query` / `auto-research-s1-huggingface-query` skill 整理 assets 和 baselines
6. 维护 `stage1_progress.md`，判断阶段完成条件
7. **决策门**：输出 idea 排序 + 理由，等待用户确认选择后进入 Stage 2

### 1.4 Skills 清单

| Skill | 类型 | 职责 |
|---|---|---|
| `auto-research-s1-flow` | **主 flow** | 控制 Stage 1 整体流程：搜索轮次管理、完成度评估、TODO 维护、产物交付 |
| `auto-research-s1-arxiv-search` | 子 skill | arxiv API 查询语法、HTML 全文下载与解析方法 |
| `auto-research-s1-paper-analysis` | 子 skill | 对单篇论文全文进行结构化分析的指南（提取什么、怎么组织） |
| `auto-research-s1-github-search` | 子 skill | GitHub API 搜索开源 repo、评估可用性（stars/更新/脚本完整度）、记录复现难度 |
| `auto-research-s1-modelscope-query` | 子 skill | ModelScope 模型/数据集信息查询方法 |
| `auto-research-s1-huggingface-query` | 子 skill | HuggingFace papers/models/datasets 查询方法 |

### 1.5 搜索策略与终止条件

**每轮搜索必须做累积综合**（搜索 → 更新 related_work → 更新 GAP → 更新 idea pool → 规划下轮关键词），确保后续轮次基于已有 GAP/idea 认知搜索，防止重复发现或撞 idea。

- **轮次上限**：5 轮
- **每轮搜索策略**：
  - Round 1：宽泛 topic 词 → 建立领域地图
  - 后续轮次：从当前 GAP 分析和 idea pool 衍生关键词 → 深挖空白方向，跳过已饱和方向
- **终止条件**（满足任一即停）：
  1. 轮次 ≥ 5
  2. 本轮新发现论文 < 5（收益递减）
- **每轮必须更新**：
  1. `related_work.md`：追加新论文条目
  2. `topic_gap_idea.md` § Gap：哪些方向已覆盖、哪些空白仍在/新出现
  3. `topic_gap_idea.md` § Idea Pool：标注与已有工作的重叠（`⚠️ overlap`），新增 idea
- **搜索终止后**：锁定 GAP + idea → 找数据集/benchmark（asset preparation）

### 1.6 维护文档：`topic_gap_idea.md` 结构

```markdown
# [研究方向名称]

## Topic
- 领域定位：
- 任务设定：
- 关键约束：（目标会议、资源、时间）

## Gap
- 现有主流路线：（2-3 句概括）
- 具体不足：
- 证据：（论文引用 + 数据支撑）

## Positioning
- 最近工作 A (xxx, 2025)：做了 X，我们区别在 Y
- 最近工作 B (xxx, 2026)：做了 X，我们区别在 Y

## Idea Pool
### Idea 1: [名称]
- 核心主张：
- 为什么能填 gap：
- 可行性：（资源、技术难度、时间）
- Novelty：（与最近工作的区别）
- 风险：（最可能失败的原因）

### Idea 2: ...
### Idea 3: ...

## Status
- 当前阶段：searching / idea_finalization / asset_prep / awaiting_user_decision
- 已完成搜索轮次：N/5
- 已读论文数：N
```

### 1.7 上下文管理

1. 每轮搜索结果：元信息追加到 `related_work.md`（持久化），context 中只保留当前轮
2. 深度分析结果：写入文件后从 context 释放
3. Context 中始终保留：`topic_gap_idea.md` 最新版 + 上一轮反思 + 当前轮搜索结果

### 1.8 进度 TODO 文档：`stage1_progress.md`

```markdown
# Stage 1 Progress

## Search Rounds
- [ ] Round 1: keywords=[...], found=N, high_relevant=N, 反思=...
- [ ] Round 2: keywords=[...], found=N, high_relevant=N, 反思=...
- [ ] ...

## Deep Analysis
- [ ] Paper A (arxiv ID) — 已下载 / 已分析 / 已写入 related_work
- [ ] Paper B ...

## Termination Check
- [ ] Round ≥ 5 reached
- [ ] New papers < 5 in latest round

## Documents
- [ ] related_work.md 完成
- [ ] topic_gap_idea.md 完成（含 3-5 ideas + 排序）
- [ ] assets.md 完成
- [ ] baselines.md 完成

## Decision Gate
- [ ] 用户确认选定 idea(s)：_______________
```

---

## 2. Stage 2: Coding & Experimenting

### 2.1 Input

1. `topic_gap_idea.md`（用户已确认的 idea）
2. `related_work.md`
3. `assets.md`
4. `baselines.md`

### 2.2 Output

1. 实验 repo（代码、脚本、配置）
2. `experiment_results.md` — 实验结果文档（含所有数字、表格、分析）
3. `pre_review_checklist.md` — 预审稿清单（实验设计时自动生成，见 §2.6）

### 2.3 抽象功能

Agent 参考主 flow skill（`auto-research-s2-flow`），自主执行以下工作：

1. 参考 `auto-research-s2-asset-download` skill 下载必要实验资产（数据集、模型、框架）
2. 参考 `auto-research-s2-eval-infrastructure` skill 搭建统一评测基建（数据加载 + 评估指标 + 评估脚本）
3. Sanity check：复现 baseline 数字（与原文 ±2% 内一致）
4. Pilot experiment：100-200 条数据跑通全链路
5. 参考 `auto-research-s2-model-call` / `auto-research-s2-model-training` / `auto-research-s2-vllm-deploy` skill 实现 baselines 和核心方法
6. 生成 pre-review checklist → 设计实验矩阵
7. 参考 `auto-research-s2-experiment-runner` skill 执行实验、收集结果
8. 参考 `auto-research-s2-result-analysis` skill 总结实验结果
9. 维护 `stage2_progress.md`，判断阶段完成条件
10. **回退机制**：若当前 idea 实验失败，从 idea pool 选下一个；全部失败则回退 Stage 1

### 2.4 Skills 清单

| Skill | 类型 | 职责 |
|---|---|---|
| `auto-research-s2-flow` | **主 flow** | 控制 Stage 2 整体流程：资产准备 → 基建 → 实验 → 结果，TODO 维护，完成判断，回退逻辑 |
| `auto-research-s2-asset-download` | 子 skill | 从 ModelScope / HuggingFace 下载模型和数据集的方法 |
| `auto-research-s2-vllm-deploy` | 子 skill | vLLM 模型部署指南（推理服务启动、参数配置） |
| `auto-research-s2-model-training` | 子 skill | 模型训练指南（GRPO / SFT / LoRA，基于 trl + accelerate） |
| `auto-research-s2-model-call` | 子 skill | 模型调用指南（本地 vLLM 推理服务 / 远程 API，并发调用、结果解析） |
| `auto-research-s2-eval-infrastructure` | 子 skill | 统一评测框架搭建指南（数据加载、指标计算、结果汇总） |
| `auto-research-s2-experiment-runner` | 子 skill | 实验脚本编排与执行指南（多实验管理、日志收集） |
| `auto-research-s2-result-analysis` | 子 skill | 实验结果解析与表格/文档生成指南 |

### 2.5 研究类型分支

#### 2.5.1 做题型（在已有 benchmark 上优化指标）

1. 核心实验：在 3 个左右 benchmark 上对选定指标的优化
2. 消融实验：确定各组件有效性
3. 泛化实验：跨模型 / 跨数据集
4. 效率对比：时间 / 显存 / query 数

#### 2.5.2 出题型（合成 benchmark + 评估框架）

1. 核心实验：通过算法合成 benchmark 数据
2. 评估框架设计与验证（人工一致性、难度分布）
3. 多模型在 benchmark 上的表现测试
4. 结果分析 + 发现总结

### 2.6 Pre-review Checklist（实验设计时自动生成）

根据 idea 文档 + 方法组件 + baseline 列表自动推导：

1. 方法有 N 个组件 → 生成 N 行消融 + 1 行 all-without
2. 有未对比的近 1 年 SOTA → 标记为必须 baseline
3. 只在 1 个模型/数据集上测 → 标记"需跨模型/跨数据集验证"
4. 有核心超参 → 标记"需敏感性实验（3-5 个点）"
5. 核心提升 < 5% → 标记"需多 seed（≥3）+ 标准差"
6. 有最显然的简化替代方案 → 标记为 ablation row（"why not just X?"）
7. 有计算开销 → 标记"需效率对比"
8. 输出为文本/结构化内容 → 标记"需 case study（3-5 例）"

输出：实验矩阵，标注 must-run / nice-to-have 优先级。

### 2.7 维护文档：`experiment_results.md` 结构

```markdown
# Experiment Results

## Setup
- 模型：
- 数据集：
- 评估指标：
- 硬件：
- 训练配置：

## Baseline Reproduction (Sanity Check)
| Method | Paper Report | Our Reproduce | Δ |
|--------|-------------|---------------|---|

## Main Results
| Method | Benchmark1 | Benchmark2 | Benchmark3 |
|--------|-----------|-----------|-----------|

## Ablation
| Config | Metric |
|--------|--------|

## Additional Experiments
### Efficiency
### Sensitivity
### Qualitative Examples

## Analysis
- 关键发现：
- 失败实验记录：（什么不 work，为什么）
```

### 2.8 进度 TODO 文档：`stage2_progress.md`

```markdown
# Stage 2 Progress

## Asset Setup
- [ ] 数据集下载 & 预处理完成
- [ ] 模型下载完成
- [ ] 框架环境配置完成

## Eval Infrastructure
- [ ] 评估脚本实现
- [ ] Sanity check 通过（baseline 数字复现）
- [ ] Pilot experiment 通过（100-200 条全链路）

## Baselines
- [ ] Baseline A 实现 & 测试
- [ ] Baseline B 实现 & 测试

## Core Method
- [ ] 核心代码实现
- [ ] 实验脚本编写
- [ ] Pre-review checklist 生成

## Experiments
- [ ] 主实验完成
- [ ] 消融实验完成
- [ ] 泛化实验完成
- [ ] 效率实验完成
- [ ] 敏感性实验完成
- [ ] Case study 完成

## Results
- [ ] experiment_results.md 完成
- [ ] 结果分析完成

## Decision Gate
- [ ] 实验结果支持 idea → 进入 Stage 3
- [ ] 实验结果不支持 → 回退：选择 idea pool 中下一个 / 回退 Stage 1
```

---

## 3. Stage 3: Writing & Review

### 3.1 Input

1. `topic_gap_idea.md`（frozen version，进入 Stage 3 时快照）
2. `related_work.md`
3. 实验 repo
4. `experiment_results.md`
5. `pre_review_checklist.md`

### 3.2 Output

1. 论文主图（方法框架图，drawio → PDF）
2. 论文表格（LaTeX 格式）
3. 可选图（问题描述图、主实验可视化、热力图等）
4. `paper_draft.md` — 论文完整 markdown 文稿
5. `review_responses.md` — 逐条 review 意见 + 修改说明（rebuttal 格式）

### 3.3 抽象功能

Agent 参考主 flow skill（`auto-research-s3-flow`），自主执行以下工作：

1. 参考 `auto-research-s3-figure-drawio` skill 绘制问题描述图（Fig.1）和方法主图（Fig.2）
2. 参考 `auto-research-s3-figure-plot` skill 生成数据可视化图
3. 参考 `auto-research-s3-table-generator` skill 从实验结果生成表格
4. 参考 `auto-research-s3-paper-writing` skill 编写完整论文文稿
5. 参考 `auto-research-s3-paper-review` skill 进行模拟审稿（切换为 reviewer 视角）
6. 参考 `auto-research-s3-paper-revision` skill 根据 review 意见修改 + 生成 `review_responses.md`
7. 维护 `stage3_progress.md`，判断 review loop 停止条件
8. **回退**：若 review 指出缺少实验 → 回退 Stage 2 定向补充

### 3.4 Skills 清单

| Skill | 类型 | 职责 |
|---|---|---|
| `auto-research-s3-flow` | **主 flow** | 控制 Stage 3 整体流程：图表 → 写作 → review loop，TODO 维护，停止条件判断 |
| `auto-research-s3-figure-drawio` | 子 skill | drawio 图绘制指南（主图、问题描述图的结构设计原则） |
| `auto-research-s3-figure-plot` | 子 skill | matplotlib/plotly 数据可视化指南（柱状图、折线图、热力图） |
| `auto-research-s3-table-generator` | 子 skill | 从实验结果生成 LaTeX/markdown 表格的指南 |
| `auto-research-s3-paper-writing` | 子 skill | 论文各章节撰写指南（结构、语气、引用规范） |
| `auto-research-s3-paper-review` | 子 skill | 模拟审稿指南（输出格式、评分维度、严重级别定义） |
| `auto-research-s3-paper-revision` | 子 skill | 根据 review 意见定位修改点并执行修改的指南 |

### 3.5 论文结构模板（通用 AI 顶会）

1. Abstract（问题 → 方法 → 结果，150-250 词）
2. Introduction（问题重要性 → 现有不足 → 我们的方法 → 贡献列表）
3. Related Work（2-3 小节，每节末尾 soft positioning）
4. Method（框架图 + 各模块描述 + 算法/公式）
5. Experiments（设置 → 主实验 → 消融 → 分析）
6. Conclusion（总结 + 局限性 + 未来工作）

### 3.6 Review Loop 规则

1. Agent 参考 `auto-research-s3-paper-review` skill 输出结构化意见，每条标注严重级别：`major` / `minor` / `suggestion`
2. 修改后生成 `review_responses.md`（逐条：原意见 → 修改内容 → 修改位置）
3. **停止条件**（满足任一）：
   - 无 `major` 级别意见（仅剩 minor / suggestion）
   - 已完成 3 轮 review-revise
4. 若 review 指出缺少实验 → 回退 Stage 2 补充实验（仅补指定实验，非全量重跑）

### 3.7 进度 TODO 文档：`stage3_progress.md`

```markdown
# Stage 3 Progress

## Figures & Tables
- [ ] Fig.1 问题描述图
- [ ] Fig.2 方法主图
- [ ] Table 1 主实验结果
- [ ] Table 2 消融实验
- [ ] 可选：主实验可视化图
- [ ] 可选：热力图 / 训练曲线

## Paper Sections
- [ ] Abstract
- [ ] Introduction
- [ ] Related Work
- [ ] Method
- [ ] Experiments
- [ ] Conclusion

## Review Loop
- [ ] Round 1: review 完成 → N major / N minor / N suggestion
- [ ] Round 1: revision 完成 → review_responses.md 更新
- [ ] Round 2: ...（如需要）
- [ ] Round 3: ...（如需要）
- [ ] 停止条件满足：_______________

## Final
- [ ] paper_draft.md 定稿
- [ ] review_responses.md 完成
- [ ] 所有图表嵌入/引用正确
```

---

## 4. 跨阶段规则

### 4.1 文档版本管理

1. `topic_gap_idea.md` 在 Stage 1-2 中为 living document，持续更新
2. 进入 Stage 3 时创建 frozen snapshot（`topic_gap_idea_frozen.md`）
3. Stage 3 中若需修改 idea/gap（因 review 反馈），更新 frozen version 并记录变更原因

### 4.2 回退机制

1. Stage 2 → Stage 2（idea pool 内切换）：当前 idea 实验失败，选下一个
2. Stage 2 → Stage 1（idea pool 耗尽）：所有候选 idea 均失败，重新搜索
3. Stage 3 → Stage 2（补实验）：review 指出缺少实验，定向补充
4. 每次回退在对应 progress 文档中记录原因

### 4.3 决策门（需用户确认）

1. Stage 1 → Stage 2：用户从 idea pool 中选择实现的 idea(s)
2. Stage 2 → Stage 3：用户确认实验结果足以支撑论文
3. Stage 2 回退 Stage 1：用户确认是否需要重新搜索或调整 topic

### 4.4 文件组织

所有 skills 扁平存放，层级关系通过 skill 内容中的引用表达：

```
nano-auto-research/
├── SPEC.md                                    ← 本文档
├── auto-research/SKILL.md                     ← 顶层入口（跨阶段调度）
├── auto-research-s1-flow/SKILL.md             ← Stage 1 flow
├── auto-research-s1-arxiv-search/SKILL.md
├── auto-research-s1-paper-analysis/SKILL.md
├── auto-research-s1-github-search/SKILL.md
├── auto-research-s1-modelscope-query/SKILL.md
├── auto-research-s1-huggingface-query/SKILL.md
├── auto-research-s2-flow/SKILL.md             ← Stage 2 flow
├── auto-research-s2-asset-download/SKILL.md
├── auto-research-s2-vllm-deploy/SKILL.md
├── auto-research-s2-model-training/SKILL.md
├── auto-research-s2-model-call/SKILL.md
├── auto-research-s2-eval-infrastructure/SKILL.md
├── auto-research-s2-experiment-runner/SKILL.md
├── auto-research-s2-result-analysis/SKILL.md
├── auto-research-s3-flow/SKILL.md             ← Stage 3 flow
├── auto-research-s3-figure-drawio/SKILL.md
├── auto-research-s3-figure-plot/SKILL.md
├── auto-research-s3-table-generator/SKILL.md
├── auto-research-s3-paper-writing/SKILL.md
├── auto-research-s3-paper-review/SKILL.md
└── auto-research-s3-paper-revision/SKILL.md

projects/<project_name>/             ← 每个研究项目独立目录
├── SPEC.md                          ← 用户需求文档（用户维护）
├── README.md                        ← 项目概况（agent 维护）
├── project_config.yaml           ← 项目配置（topic/method_sketch/target_venue/constraints/models，用户填写，agent 只读）
├── docs/                            ← 项目文档（agent 维护，S1 产物）
│   ├── project_status.md            ← 全局状态（auto-research 维护）
│   ├── stage1_progress.md
│   ├── stage2_progress.md
│   ├── stage3_progress.md
│   ├── topic_gap_idea.md
│   ├── topic_gap_idea_frozen.md
│   ├── related_work.md
│   ├── assets.md
│   ├── baselines.md
│   ├── experiment_results.md
│   └── pre_review_checklist.md
├── data/                            ← 数据集、benchmark
├── models/                          ← 下载的模型权重
├── baselines/                       ← 克隆的开源 baseline 项目
├── src/                             ← 源码（面向对象，核心方法实现）
├── scripts/                         ← 脚本（面向过程，训练/推理/评估脚本）
├── exp/                             ← 实验脚本（sh 文件，单文件 = 单完整实验）
├── output/                          ← 实验产物（S2 产物：checkpoints、日志、结果）
└── paper/                           ← 论文相关（S3 产物：文稿、图表、review）
    ├── paper_draft.md
    ├── review_responses.md
    ├── figures/
    └── tables/
```

### 4.5 项目目录职责说明

| 目录 | 维护者 | 内容 | 对应阶段 |
|---|---|---|---|
| `SPEC.md` | 用户 | 需求、约束、目标 | 全程 |
| `README.md` | agent | 项目概况、当前状态、快速上手 | 全程 |
| `project_config.yaml` | 用户 | 项目配置：topic, method_sketch, target_venue, constraints, models（本地: name/type/path/desc; API: name/type/url/api_key/max_concurrency/rpm/desc） | 项目启动前填写 |
| `docs/` | agent | 调研文档、进度、结果 | S1 产物为主 |
| `data/` | agent | 数据集下载 & 预处理 | S1 末下载 |
| `models/` | agent | 模型权重下载 | S1 末下载 |
| `baselines/` | agent | 克隆的 baseline repos | S1 末下载 |
| `src/` | agent | 核心方法代码（OOP） | S2 |
| `scripts/` | agent | 训练/推理/评估脚本 | S2 |
| `exp/` | agent | 实验 sh 脚本（单文件单实验） | S2 |
| `output/` | agent | checkpoints、日志、结果文件 | S2 |
| `paper/` | agent | 论文文稿、图表、review 回复 | S3 |

### 4.6 资产下载时机

资产下载（模型、数据集、baseline repos）在 **S1 决策门通过后** 执行：

1. 用户确认 idea → agent 根据 `assets.md` 和 `baselines.md` 确定下载清单
2. 下载模型到 `models/`、数据集到 `data/`、克隆 baseline repos 到 `baselines/`
3. 验证下载完整性（模型可加载、数据集可读取、baseline 可运行）
4. 完成后 S1 正式结束，进入 S2

这确保 S2 启动时所有资产已就绪，无需等待下载。

### 4.7 Skill 编写规范

所有 skill 遵循 [Agent Skills Spec](https://agentskills.io/llms.txt)：

1. 每个 skill 为一个目录，包含 `SKILL.md`（必须）+ 可选 `scripts/`、`references/`、`assets/`
2. `SKILL.md` 包含 YAML frontmatter（name, description）+ Markdown body（行动指南）
3. Skill 为纯文本指导文档，agent 读取后自行执行操作
4. **所有 skills 扁平存放**，不使用子目录分层。层级关系通过 skill body 中的引用表达（如 "参考 `auto-research-s1-arxiv-search` skill 执行搜索"）
5. Skill 层级：
   - **顶层**（`auto-research`）：跨阶段调度、决策门、回退、版本管理
   - **阶段 flow**（`auto-research-s1-flow` / `auto-research-s2-flow` / `auto-research-s3-flow`）：阶段目标、子任务顺序、TODO 维护规则、完成条件判断逻辑、阶段内回退规则
   - **子 skill**（其余）：具体操作步骤、命令示例、输出格式要求、常见错误处理
6. 单个 `SKILL.md` 控制在 500 行内，详细内容拆分到 `references/` 目录
7. `description` 字段应包含足够的关键词，使 agent 能通过语义匹配发现并加载正确的 skill
