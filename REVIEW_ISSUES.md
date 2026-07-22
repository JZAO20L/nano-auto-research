# REVIEW_ISSUES — nano-auto-research Skills 审查

Phase 1 静态审查结果（2026-07-22），4 个并行 agent 审查 22 个 SKILL.md + 遗留目录。

状态标记：`[ ]` 待修复 / `[x]` 已修复 / `[-]` 不修复

---

## 🔴 CRITICAL（2 项）

### C1. arxiv-search frontmatter name 与目录名不匹配
- **文件**: `auto-research-s1-arxiv-search/SKILL.md`
- **问题**: frontmatter `name: arxiv-search`，但目录名为 `auto-research-s1-arxiv-search`。s1-flow 和 paper-analysis 均用目录名引用。无论 skill 加载机制用哪种名字，都有一端解析失败。
- **修复**: 将 frontmatter name 改为 `auto-research-s1-arxiv-search`，同时将 `source`/`extracted_at` 替换为 `metadata.version: "0.1"` 以统一格式。
- [x] 已修复

### C2. model-training 中硬编码 SwanLab API Key
- **文件**: `auto-research-s2-model-training/SKILL.md`
- **问题**: SwanLab 配置段包含明文 API Key `lIrr4ZEyrTCWsWGQE9szs`，属于凭据泄露。
- **修复**: 删除 key，替换为 "paste your API key from https://swanlab.cn/settings"。
- [x] 已修复

---

## 🟡 MODERATE（按主题分组）

### 交叉引用与调用机制

#### M1. s1-flow 未说明如何调用 sub-skill
- **文件**: `auto-research-s1-flow/SKILL.md`
- **问题**: 写 "invoke `auto-research-s1-arxiv-search`" 但未解释调用机制（Skill tool？读 SKILL.md 后执行？）。fresh agent 不知道 "invoke" 意味着什么。
- **修复**: 加一句说明，如 "Read the target skill's SKILL.md and follow its instructions" 或 "Use the Skill tool with the skill name"。
- [x] 已修复

#### M2. s2-flow 从未引用 vllm-deploy
- **文件**: `auto-research-s2-flow/SKILL.md`
- **问题**: SPEC §2.3 step 5 要求 flow 引用 vllm-deploy，但实际 SKILL.md 只引用了 model-call 和 model-training。vllm-deploy 是 model-call 的前置依赖，却从未被任何 skill 引用（孤儿 skill）。
- **修复**: 在 s2-flow step 1 或 2 加入 "Reference **auto-research-s2-vllm-deploy** for local model serving"。
- [x] 已修复

#### M3. s3-flow Step 1 源文件未定义
- **文件**: `auto-research-s3-flow/SKILL.md`
- **问题**: "Copy the current frozen idea to `docs/topic_gap_idea_frozen.md` if not already present" — 从哪里 copy？源文档未指明。
- **修复**: 明确源为 `docs/topic_gap_idea.md`（S1 产出）。
- [x] 已修复

#### M4. s3-flow 伪代码调用语法歧义
- **文件**: `auto-research-s3-flow/SKILL.md`
- **问题**: `run(auto-research-s3-paper-review)` 看起来像函数调用但不是真实命令。agent 可能尝试在 shell 中执行。
- **修复**: 改为自然语言描述或明确标注为伪代码。
- [x] 已修复

### 一致性

#### M5. frontmatter metadata.version 缺失
- **文件**: `auto-research-s2-vllm-deploy/SKILL.md`, `auto-research-s2-model-training/SKILL.md`
- **问题**: 其他 20 个 skill 均有 `metadata.version: "0.1"`，这两个缺失。
- [x] 已修复

#### M6. vLLM 启动命令不一致
- **文件**: `auto-research-s2-model-call/SKILL.md` vs `auto-research-s2-vllm-deploy/SKILL.md`
- **问题**: model-call 用 `python -m vllm.entrypoints.openai.api_server`（已废弃），vllm-deploy 用 `vllm serve`（新版）。agent 看到两种写法会困惑。
- **修复**: 统一为 `vllm serve`，旧写法仅作注释提及。
- [x] 已修复

#### M7. curl --max-time 使用不一致
- **文件**: `auto-research-s1-modelscope-query/SKILL.md`, `auto-research-s1-huggingface-query/SKILL.md`
- **问题**: arxiv-search 所有 curl 均带 `--max-time 20`，但 modelscope 和 huggingface 完全没有。挂起的 curl 会阻塞 agent。
- **修复**: 所有 curl 命令加 `--max-time 20`。
- [x] 已修复

#### M8. web_fetch 指导矛盾
- **文件**: `auto-research-s1-arxiv-search/SKILL.md` vs `auto-research-s1-github-search/SKILL.md`
- **问题**: arxiv-search 明确说 web_fetch 不可靠（safety classifier、proxy 504），github-search 却建议用 web_fetch 作 fallback。
- **修复**: 从 github-search 移除 web_fetch 建议，或注明 "web_fetch 对 GitHub 可能可用但对 arxiv 不可用"。
- [x] 已修复

#### M9. --enforce_eager vs --enforce-eager
- **文件**: `auto-research-s2-model-training/SKILL.md` vs `auto-research-s2-vllm-deploy/SKILL.md`
- **问题**: model-training 用下划线 `--enforce_eager`，vllm-deploy 用连字符 `--enforce-eager`。vLLM CLI 用连字符，下划线形式可能失败。
- [x] 已修复

### 命令与 API 问题

#### M10. drawio XML 模板不完整
- **文件**: `auto-research-s3-figure-drawio/SKILL.md`
- **问题**: edge cell 引用 `target="4"` 但模板中无 `id="4"` 的节点。agent 照搬会生成无效 XML。
- [x] 已修复

#### M11. 无 drawio→PDF 转换指引
- **文件**: `auto-research-s3-figure-drawio/SKILL.md`
- **问题**: 输出要求 `fig1_problem.pdf`，但 skill 只生成 XML。未说明如何转换（drawio CLI / LibreOffice / 手动导出）。
- [x] 已修复

#### M12. HuggingFace papers API 端点可能有误
- **文件**: `auto-research-s1-huggingface-query/SKILL.md`
- **问题**: 使用 `https://huggingface.co/api/papers?search=KEYWORD`，但 HF papers API 历史上用 `?q=` 而非 `?search=`。可能返回空结果。
- **修复**: 实际测试确认正确参数。
- [x] 已修复

#### M13. GitHub HTML fallback 无用
- **文件**: `auto-research-s1-github-search/SKILL.md`
- **问题**: `curl -s "https://github.com/search?q=..."` 返回 JS 渲染页面，curl 无法解析。此 fallback 实际无效。
- **修复**: 移除或替换为 GitHub API 搜索。
- [x] 已修复

#### M14. eval-infrastructure 模板缺少 import
- **文件**: `auto-research-s2-eval-infrastructure/SKILL.md`
- **问题**: 脚本模板中使用 `judge_refusal(r)` 但未 import。agent 照搬会 NameError。
- [x] 已修复

#### M15. experiment-runner tee 管道捕获错误退出码
- **文件**: `auto-research-s2-experiment-runner/SKILL.md`
- **问题**: `bash "$exp" | tee ...` 后 `$?` 是 tee 的退出码而非脚本的。应使用 `set -o pipefail` 或 `${PIPESTATUS[0]}`。
- [x] 已修复

### 缺失信息

#### M16. orchestrator 无 mkdir -p 指令
- **文件**: `auto-research/SKILL.md`
- **问题**: Section 1 说 "create the project directory structure" 但未给出 shell 命令。agent 可能向不存在的目录写文件。
- [x] 已修复

#### M17. table-generator 无 LaTeX 包依赖说明
- **文件**: `auto-research-s3-table-generator/SKILL.md`
- **问题**: 模板使用 `\toprule`/`\midrule`/`\bottomrule`（booktabs）但未说明需要 `\usepackage{booktabs}`。
- [x] 已修复

#### M18. paper-writing 引用格式矛盾
- **文件**: `auto-research-s3-paper-writing/SKILL.md`
- **问题**: 写 "Citation format: [Author et al., Year]"（author-year），但 AAAI 用编号引用 `[1]`。"AAAI/ACL format" 含糊（AAAI 7+1 页，ACL 8+1 页）。
- **修复**: 明确选择编号引用，区分 AAAI/ACL 页数。
- [x] 已修复

#### M19. figure-plot 未声明 seaborn 依赖
- **文件**: `auto-research-s3-figure-plot/SKILL.md`
- **问题**: heatmap 示例用 `import seaborn as sns`，但 Global Style 只 import matplotlib/numpy。
- [x] 已修复

#### M20. figure-plot 相对路径无工作目录说明
- **文件**: `auto-research-s3-figure-plot/SKILL.md`
- **问题**: `plt.savefig('paper/figures/...')` 用相对路径但未指定 cwd。
- [x] 已修复

#### M21. asset-download 代理地址硬编码
- **文件**: `auto-research-s2-asset-download/SKILL.md`
- **问题**: `http://127.0.0.1:7890` 出现 3 次。应注明 "replace with your proxy address"。
- [x] 已修复

#### M22. model-training 引用不明路径
- **文件**: `auto-research-s2-model-training/SKILL.md`
- **问题**: 引用 `docs/docs/trl/` 作为文档路径，不清楚相对于什么。fresh agent 无法解析。
- [x] 已修复

#### M23. model-training 实验性模块缺版本标注
- **文件**: `auto-research-s2-model-training/SKILL.md`
- **问题**: `trl.experimental.distillation`、`trl.experimental.gspo_token` 等可能不存在于所有 TRL >= 0.19 版本。`environment_factory` 需 `transformers>=5.2.0` 但未在版本表中列出。
- [x] 已修复

#### M24. paper-revision 无 scope creep 指导
- **文件**: `auto-research-s3-paper-revision/SKILL.md`
- **问题**: minor revision 级联成多节重写时，无指导何时停下请求人工审查。
- [x] 已修复

#### M25. github-search Stars>50 阈值过刚性
- **文件**: `auto-research-s1-github-search/SKILL.md`
- **问题**: 小众领域唯一可用 repo 可能只有 30 stars。无放宽阈值的指导。
- [x] 已修复

#### M26. modelscope 对 HuggingFace 的硬依赖
- **文件**: `auto-research-s1-modelscope-query/SKILL.md`
- **问题**: "search on HuggingFace first, then verify on ModelScope" — 若 HF 不可用或模型不在 HF，无替代发现路径。
- [x] 已修复

#### M27. result-analysis compute_delta 除零
- **文件**: `auto-research-s2-result-analysis/SKILL.md`
- **问题**: baseline 为 0 时返回 `float("inf")`，在 markdown 表格中渲染为 `inf%`。
- [x] 已修复

#### M28. arxiv-search 全文截取仅 5000 字符
- **文件**: `auto-research-s1-arxiv-search/SKILL.md`
- **问题**: `print(text[:5000])` 约 800 词，不足以提取方法/实验细节。无分页或写文件的指导。
- [x] 已修复

#### M29. paper-analysis 无 arxiv 外论文处理
- **文件**: `auto-research-s1-paper-analysis/SKILL.md`
- **问题**: 通过 GitHub/HuggingFace 发现的论文若不在 arxiv 上，无 fallback 策略。
- [x] 已修复

---

## 🟢 MINOR（精选，共 30+ 项）

| # | 文件 | 问题 |
|---|------|------|
| m1 | orchestrator | `{user_topic}`/`{date}` 占位符未说明来源 |
| m2 | orchestrator | S2/S3 无 phase 词汇表（S1 有） |
| m3 | s1-flow | 完成标准 "3/4 criteria met" 无优先级 |
| m4 | s1-arxiv-search | Semantic Scholar API 提及但无端点/示例 |
| m5 | s1-paper-analysis | "Published within the last 2 years" 未说明相对日期 |
| m6 | s1-github-search | 无命令验证 README/scripts 是否存在 |
| m7 | s1-modelscope-query | namespace 映射仅 2 例（Qwen, Llama） |
| m8 | s1-modelscope-query | 无安全测试下载的方法（避免下载数十 GB） |
| m9 | s1-huggingface-query | 无 rate-limiting 指导 |
| m10 | s2-asset-download | 未提及 git-lfs |
| m11 | s2-vllm-deploy | `vllm bench serve` 版本依赖未注明 |
| m12 | s2-vllm-deploy | Qwen3.5-397B 高级 flag 无版本标注 |
| m13 | s2-model-call | `parse_response` 对畸形 think 标签脆弱 |
| m14 | s2-model-call | sync 版每次创建新 OpenAI client |
| m15 | s2-eval-infrastructure | JSONL glob 可能匹配非预期文件 |
| m16 | s2-experiment-runner | `fuser -k /dev/nvidia0` 在共享机器上危险 |
| m17 | s2-experiment-runner | collect_results.py 缺表头分隔行 |
| m18 | s2-result-analysis | 示例数据为领域特定（jailbreak ASR） |
| m19 | s3-flow | "auto-research orchestrator" 未显式命名 |
| m20 | s3-flow | "score ≥ 7/10" 仅在 review skill 定义 |
| m21 | s3-figure-drawio | 无 XML vs structured text 选择指导 |
| m22 | s3-figure-plot | 无数据不可用时的 fallback |
| m23 | s3-table-generator | 无非标准表格结构的 fallback |
| m24 | s3-paper-writing | `[METHOD NAME]` 占位符未说明来源 |
| m25 | s3-paper-review | 未说明模拟几个 reviewer |
| m26 | s3-paper-review | 与项目级 academic-paper-review skill 潜在重叠 |
| m27 | s3-paper-revision | "Report to auto-research orchestrator" 未显式命名 |
| m28 | s2-flow | 中英文混用（做题型/出题型） |
| m29 | s3-figure-drawio | Steps 在 Output 之后，逻辑流不顺 |
| m30 | s3-paper-writing | Related Work 示例为领域特定 |

---

## 结构性问题

### S1. 遗留空目录 experimenting/ 和 writing/
- **状态**: 两个目录完全为空，无任何文件。无任何 SKILL.md、SPEC.md、README.md 引用。
- **结论**: 早期设计遗留，已被 `auto-research-s2-*` 和 `auto-research-s3-*` 完全取代。
- **建议**: 删除。
- [ ] 待处理

---

## 统计

| 严重度 | 数量 |
|--------|------|
| 🔴 CRITICAL | 2 |
| 🟡 MODERATE | 29 |
| 🟢 MINOR | 30 |
| 结构性 | 1 |
| **合计** | **62** |

## 修复优先级建议

1. **立即**: C1（name 不匹配）、C2（API key 泄露）
2. **高优**: M1-M4（调用机制/交叉引用）、M5-M9（一致性）、M10-M11（drawio）
3. **中优**: M12-M15（命令修正）、M16-M23（缺失信息）
4. **低优**: M24-M29（边界情况）、所有 MINOR
