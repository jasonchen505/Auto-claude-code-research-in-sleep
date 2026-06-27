# ARIS 项目增量学习笔记
> 对比前两轮面试准备，记录复现过程中新学习到的点

---

## 目录
1. [学习进度追踪](#1-学习进度追踪)
2. [Phase 0: 环境搭建新发现](#2-phase-0-环境搭建新发现)
3. [Phase 1: 单模块学习新发现](#3-phase-1-单模块学习新发现)
4. [Phase 2: 全流程验证新发现](#4-phase-2-全流程验证新发现)
5. [Phase 3: 中等规模实验新发现](#5-phase-3-中等规模实验新发现)
6. [Phase 4: 高级功能新发现](#6-phase-4-高级功能新发现)
7. [关键认知升级](#7-关键认知升级)
8. [面试回答优化](#8-面试回答优化)
9. [实践心得与最佳实践](#9-实践心得与最佳实践)

---

## 1. 学习进度追踪

### 1.1 前两轮学习回顾

**第一轮：项目概览与核心概念**
- ✅ 理解了 ARIS 的整体架构
- ✅ 理解了跨模型对抗协作的基本概念
- ✅ 理解了 Type-A/Type-B 门控系统
- ✅ 理解了 Fan-Out Pattern
- ⚠️ 对实现细节理解不够深入
- ⚠️ 缺乏实际操作经验

**第二轮：技术深挖与面试准备**
- ✅ 深入理解了核心设计决策
- ✅ 准备了面试回答模板
- ✅ 分析了问题与解决方案
- ⚠️ 对工程落地细节理解不足
- ⚠️ 缺乏实际调试经验

### 1.2 第三轮学习目标

```
目标：从"理解"到"实践"
├── 理解实现细节
├── 掌握调试技巧
├── 积累工程经验
├── 形成最佳实践
└── 优化面试回答
```

---

## 2. Phase 0: 环境搭建新发现

### 2.1 MCP 协议的实际实现

**前两轮理解**：
```
MCP 是 Anthropic 提出的协议，用于 LLM 和工具之间的通信。
- stdio 传输 + JSON-RPC
- 工具发现和调用
- 版本协商
```

**新发现**：

```python
# 1. 版本协商的实现细节
# 前两轮只知道"需要版本协商"
# 现在理解了具体的实现

# MCP 握手流程
Client → Server: initialize (protocolVersion: "2025-03-26")
Server → Client: initialize (protocolVersion: "2025-03-26")  # 协商的版本
Client → Server: notifications/initialized
Client → Server: tools/list

# 版本验证逻辑（v0.4.19 修复）
SUPPORTED_VERSIONS = [
    "2025-11-25", "2025-06-18", "2025-03-26", "2024-11-05"
]

if negotiated_version not in SUPPORTED_VERSIONS:
    terminate_child_process()
    clear_slot()
    raise Error("Unsupported MCP protocol version")
```

**关键认知升级**：
- MCP 协议的版本协商不是简单的版本号比较
- 需要验证 stdio 分帧格式是否兼容
- 不同版本的 stdio 分帧格式可能不同

### 2.2 Claude Code 的 Skill 加载机制

**前两轮理解**：
```
Skills 是 Markdown 文件，Claude Code 读取并执行。
- 放在 .claude/skills/ 目录
- 通过 symlink 链接到 ARIS 仓库
```

**新发现**：

```python
# 1. Skill 的加载顺序
# Claude Code 会按以下顺序查找 Skill：
# 1. .claude/skills/<name>/SKILL.md
# 2. ~/.claude/skills/<name>/SKILL.md
# 3. 全局 skills 目录

# 2. Skill 的 frontmatter 解析
---
name: auto-review-loop           # Skill 名称
description: "..."               # 描述（显示在自动补全）
argument-hint: [topic-or-scope]  # 参数提示
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Skill, mcp__codex__codex
---

# 3. allowed-tools 的权限控制
# - Bash(*): 允许执行任意 bash 命令
# - Read/Write/Edit: 文件操作
# - mcp__codex__codex: 调用 Codex MCP
# - Skill: 调用其他 Skill
```

**关键认知升级**：
- Skill 的 allowed-tools 是细粒度的权限控制
- 不是所有 Skill 都需要所有工具
- 最小权限原则可以防止意外操作

### 2.3 Research Wiki 的初始化流程

**前两轮理解**：
```
Research Wiki 是持久化的研究知识库。
- 4 层节点：papers, ideas, experiments, claims
- 关系图：graph/edges.jsonl
- 确定性写入器
```

**新发现**：

```python
# 1. 初始化命令
/research-wiki init

# 2. 创建的目录结构
research-wiki/
├── index.md               # 分类索引（自动生成）
├── log.md                 # 追加式时间线
├── gap_map.md             # 领域空白图
├── query_pack.md          # 压缩摘要（供 /idea-creator 使用）
├── papers/                # 论文节点
├── ideas/                 # Idea 节点
├── experiments/           # 实验节点
├── claims/                # 声明节点
└── graph/
    └── edges.jsonl        # 关系图

# 3. query_pack.md 的压缩逻辑
# - 项目方向：从 RESEARCH_BRIEF.md 提取
# - 空白图：gap_map.md 的前 1200 字符
# - 失败 Idea：ideas/ 中 outcome=negative/mixed 的记录
# - 论文摘要：papers/ 中的前 12 篇
# - 最近关系：graph/edges.jsonl 的前 20 条
# - 总计：最多 8000 字符
```

**关键认知升级**：
- query_pack.md 是 /idea-creator 的关键输入
- 它包含了"避免重复"的信息（失败 Idea）
- 压缩逻辑是有优先级的（问题 > 约束 > 方向 > 背景）

---

## 3. Phase 1: 单模块学习新发现

### 3.1 /research-lit 的多源聚合

**前两轮理解**：
```
/research-lit 从多个来源搜索论文。
- arXiv, Semantic Scholar, OpenAlex, Exa, DeepXiv
- 去重和验证
```

**新发现**：

```python
# 1. 多源聚合的策略
# 使用 integration-contract.md 的 Policy D2：
# - 调用所有可用的来源
# - 单个来源失败时 warn-and-continue
# - 只要 ≥1 个来源成功就继续

# 2. 去重逻辑
# - 优先使用 arxiv_id 去重
# - 其次使用 DOI 去重
# - 最后使用标题相似度去重

# 3. 验证逻辑
# 使用 verify_papers.py 进行 3 层交叉验证：
# Layer 1: arXiv API 验证
# Layer 2: CrossRef API 验证
# Layer 3: Semantic Scholar API 验证

# 4. Gemini 的特殊处理
# - 自动包含 gemini 作为来源（除非用户指定 sources）
# - gemini-cli 必须安装
# - 失败时 graceful degradation
```

**关键认知升级**：
- 多源聚合不是简单的"搜索所有来源"
- 有优先级、去重、验证的复杂逻辑
- Gemini 在文献调研中有特殊价值（AI 驱动的广泛覆盖）

### 3.2 /idea-creator 的 Fan-Out 实现

**前两轮理解**：
```
/idea-creator 使用 Fan-Out Pattern 生成多个 Idea。
- 同家族子代理生成候选
- 跨模型 Jury 做裁决
```

**新发现**：

```python
# 1. 7 个分析 Lenses（视角）
lenses = [
    "structural_gaps",      # 结构空白：A 方法有但 B 方法没有
    "contradictory_findings",  # 矛盾发现：不同论文结论相反
    "untested_assumptions",    # 未测试假设：公认但未验证
    "unexplored_scaling",      # 未探索缩放：没人试过的规模
    "cross_domain_transfer",   # 跨域迁移：从其他领域借鉴
    "negative_results",        # 负面结果：失败的尝试
    "combination_gaps",        # 组合空白：没人组合过的方法
]

# 2. Phase 3 的问题（已知 gap）
# /idea-creator Phase 3 会进行 same-family 快速新颖性检查
# 这违反了 fan-out-pattern.md 的规则：
# "executor pre-filters the jury's input with same-family quality judgment"
# 应该把所有非重复 idea 送到 Phase 4 的跨模型 Jury

# 3. 去重逻辑
# - dedup_key: 规范化后的标题/claim-stem
# - 机械聚类：字符串匹配/近似匹配
# - 不使用 LLM 判断"是否相同"
```

**关键认知升级**：
- Fan-Out 不是简单的"并行生成"
- 有严格的"生成"和"裁决"分离
- 已知的 gap：Phase 3 的 same-family 过滤

### 3.3 /auto-review-loop 的审稿流程

**前两轮理解**：
```
/auto-review-loop 进行多轮审稿-修改循环。
- 跨模型审稿
- 最多 4 轮
- 停止条件：score >= 6 AND verdict ∈ {ready, almost}
```

**新发现**：

```python
# 1. 三种难度模式
# medium: 标准 MCP 审稿
# hard: + Reviewer Memory + Debate Protocol
# nightmare: + Codex exec（reviewer 直接读代码）

# 2. Reviewer Memory 的实现
# - REVIEWER_MEMORY.md 记录每轮的怀疑
# - 下一轮审稿时传给 reviewer
# - reviewer 检查之前的怀疑是否被解决

# 3. Debate Protocol 的实现
# Step 1: Executor 写 rebuttal（最多 3 个）
# Step 2: Reviewer 做 ruling（SUSTAINED/OVERRULED/PARTIALLY SUSTAINED）
# Step 3: 根据 ruling 更新分数

# 4. Nightmare 模式的特殊处理
# - 不使用 MCP，而是使用 codex exec
# - GPT 直接读代码，Claude 无法过滤
# - 更接近真实的 hostile reviewer

# 5. 状态恢复机制
# - REVIEW_STATE.json 记录当前状态
# - 支持 context compaction 后恢复
# - 24 小时内的中断可以 resume
```

**关键认知升级**：
- 三种难度模式的设计很巧妙
- Reviewer Memory 解决了"多轮审稿记忆丢失"问题
- Nightmare 模式是最接近真实审稿的

### 3.4 /experiment-bridge 的实验部署

**前两轮理解**：
```
/experiment-bridge 负责实验部署。
- 代码实现
- 跨模型代码审查
- Sanity Check
- 全量实验
```

**新发现**：

```python
# 1. Queue Routing 的自动决策
# ≤5 jobs: 使用 /run-experiment
# ≥10 jobs: 使用 /experiment-queue
# 有 phase dependencies: 使用 /experiment-queue

# 2. Sanity Check 的自动 Debug
# 最多 3 次尝试
# 第 2 次尝试前调用 /codex:rescue（第二意见）
# 仍然失败则停止并报告

# 3. Cross-Model Code Review 的检查点
# - 代码是否正确实现了方法
# - 超参数是否与计划一致
# - 是否有逻辑错误
# - 是否使用了正确的 ground truth
# - 是否有 OOM 风险

# 4. 结果收集的格式
# - JSON/CSV 格式（便于后续分析）
# - 更新 EXPERIMENT_TRACKER.md
# - 检查成功标准
```

**关键认知升级**：
- Queue Routing 的自动决策很智能
- Sanity Check 的自动 Debug 可以节省大量时间
- Cross-Model Code Review 可以在部署前捕获错误

---

## 4. Phase 2: 全流程验证新发现

### 4.1 Idea Discovery 的实际运行

**前两轮理解**：
```
/idea-discovery 运行完整的 Idea 发现流程。
- 文献调研 → Idea 生成 → Novelty Check → Review → Refine
```

**新发现**：

```python
# 1. composed mode 的作用
# - /research-lit 在 composed mode 下不写独立文件
# - 而是把结果折叠到 IDEA_REPORT.md
# - 避免了"散落的 MD 文件"

# 2. AUTO_PROCEED 的行为
# - true: 展示结果后等待 10 秒，自动继续
# - false: 必须等待用户确认
# - 可以在每个 checkpoint 单独覆盖

# 3. REF_PAPER 的处理
# - 先生成 REF_PAPER_SUMMARY.md
# - 作为后续所有阶段的上下文
# - Idea 会基于参考论文的局限性生成

# 4. Research Wiki 的自动摄入
# - /research-lit 会自动把论文写入 Wiki
# - /idea-creator 会读取 Wiki 中的失败 Idea
# - 避免重复尝试已经失败的方向
```

**关键认知升级**：
- composed mode 是避免文件散落的关键设计
- AUTO_PROCEED 的 10 秒等待是一个巧妙的 UX 设计
- Research Wiki 的自动摄入是无缝集成的

### 4.2 Experiment Bridge 的实际运行

**前两轮理解**：
```
/experiment-bridge 负责实验部署。
- 解析实验计划
- 实现代码
- 部署实验
- 收集结果
```

**新发现**：

```python
# 1. BASE_REPO 的处理
# - 如果设置了 BASE_REPO，先 clone 仓库
# - 在现有代码基础上实现实验
# - 避免从零开始

# 2. 实验脚本的生成
# - argparse 配置所有超参数
# - 固定随机种子
# - 结果保存为 JSON/CSV
# - 使用 wandb 记录（如果配置）

# 3. GPU 分配策略
# - 每个实验分配独立的 GPU
# - 使用 CUDA_VISIBLE_DEVICES
# - 并行运行多个实验

# 4. 结果收集的自动化
# - 解析输出文件
# - 更新 EXPERIMENT_TRACKER.md
# - 检查成功标准
# - 运行 /ablation-planner（如果主结果正面）
```

**关键认知升级**：
- BASE_REPO 可以复用现有代码
- GPU 分配是自动的
- 结果收集是结构化的

### 4.3 Auto Review Loop 的实际运行

**前两轮理解**：
```
/auto-review-loop 进行多轮审稿-修改循环。
- 跨模型审稿
- 代码修改
- 重新实验
```

**新发现**：

```python
# 1. 审稿 Prompt 的结构
# - 文件路径（不是摘要）
# - 审稿目标（NeurIPS/ICML 级别）
# - 评分标准（1-10）
# - Verdict（ready/almost/not ready）

# 2. 代码修改的优先级
# - 优先修改 metric additions（便宜、高影响）
# - 跳过需要大量计算的修复
# - 跳过需要外部数据/模型的修复
# - 优先 reframing/analysis over new experiments

# 3. 结果验证的逻辑
# - 检查是否满足停止条件
# - 如果 score >= 6 AND verdict ∈ {ready, almost}，停止
# - 否则继续下一轮

# 4. 状态持久化
# - REVIEW_STATE.json 记录当前状态
# - 支持 context compaction 后恢复
# - 24 小时内的中断可以 resume
```

**关键认知升级**：
- 审稿 Prompt 的结构是精心设计的
- 代码修改有优先级
- 状态持久化是长时间运行的关键

---

## 5. Phase 3: 中等规模实验新发现

### 5.1 /experiment-queue 的队列管理

**前两轮理解**：
```
/experiment-queue 管理大规模实验队列。
- OOM retry
- Stale-screen cleanup
- Wave transitions
```

**新发现**：

```python
# 1. Job Manifest 的结构
project: my_grid_experiment
cwd: /home/user/your_project
conda: my_env
ssh: gpu-server
default_cmd: python run_distill.py --backbone softmax --lam 0.5

preconditions:
  - type: checkpoint_exists
    path: checkpoints/transformer/teacher_L96_K500_N{N}.pt

gpus: [0, 1, 2, 3, 4, 5, 6, 7]
max_parallel: 8
gpu_free_threshold_mib: 500
oom_retry:
  delay: 120
  max_attempts: 3

jobs:
  - id: s200_N64_n50K
    args: {seed: 200, n_hidden: 64, n_train_subset: 50000, subset_seed: 2024}

# 2. Job 状态机
pending → running → completed
                 ↘ failed_oom → pending (after delay) [retry up to N]
                 ↘ failed_other → stuck (needs manual inspection)
stale_screen_detected → cleaned → pending

# 3. Wave Orchestration
# - 一个 wave 是一批适合可用 GPU 的 jobs
# - 下一个 wave 只在当前 wave 完全结束后启动
# - 检查：所有 python 进程退出 + 没有 stale screens

# 4. 依赖管理
# - depends_on: 前置 job 完成后才启动
# - 用于 teacher → student 的链式训练
```

**关键认知升级**：
- Job Manifest 是声明式的
- 状态机是明确的
- Wave Orchestration 解决了"wave race"问题

### 5.2 跨模型代码审查的实际效果

**前两轮理解**：
```
跨模型代码审查可以在部署前捕获错误。
- GPT-5.5 审查 Claude 的代码
- 发现逻辑错误、超参数错误等
```

**新发现**：

```python
# 1. 审查的检查点
# - 代码是否正确实现了方法
# - 超参数是否与计划一致
# - 是否有逻辑错误
# - 是否使用了正确的 ground truth
# - 是否有 OOM 风险
# - 是否有数值不稳定风险

# 2. 实际发现的问题
# - 错误的 ground truth（使用模型输出作为 GT）
# - 错误的 metric 计算
# - 缺少随机种子固定
# - OOM 风险（batch size 太大）
# - 数值不稳定（loss 爆炸）

# 3. 审查的局限性
# - 不能发现所有错误
# - 有些错误只有运行时才能发现
# - 需要结合 Sanity Check

# 4. 审查的成本
# - 每次审查消耗 API tokens
# - 增加部署延迟
# - 需要平衡成本和收益
```

**关键认知升级**：
- 跨模型代码审查是有价值的
- 但不能替代 Sanity Check
- 需要平衡成本和收益

### 5.3 实验结果的结构化收集

**前两轮理解**：
```
实验结果需要结构化收集。
- JSON/CSV 格式
- 更新 EXPERIMENT_TRACKER.md
```

**新发现**：

```python
# 1. 结果文件的结构
{
    "run_id": "s200_N64_n50K",
    "config": {
        "seed": 200,
        "n_hidden": 64,
        "n_train_subset": 50000,
        "subset_seed": 2024
    },
    "metrics": {
        "train_loss": 0.123,
        "eval_loss": 0.456,
        "accuracy": 0.892,
        "f1": 0.876
    },
    "timing": {
        "start_time": "2026-06-27T10:00:00Z",
        "end_time": "2026-06-27T12:00:00Z",
        "duration_seconds": 7200
    },
    "hardware": {
        "gpu": "RTX 4090",
        "gpu_memory_used_mb": 18000,
        "gpu_utilization_percent": 95
    }
}

# 2. EXPERIMENT_TRACKER.md 的结构
| Run ID | System | Key Metric | Status | Notes |
|--------|--------|-----------|--------|-------|
| R001 | baseline_1 | 0.850 | DONE | |
| R002 | our_method | 0.892 | DONE | +4.2% |
| R003 | ablation_1 | 0.870 | DONE | -2.2% |

# 3. 成功标准的检查
# - 每个实验有明确的成功标准
# - 例如：accuracy > 0.85, loss < 0.5
# - 自动检查并记录
```

**关键认知升级**：
- 结果文件的结构是精心设计的
- EXPERIMENT_TRACKER.md 是人类可读的
- 成功标准是自动检查的

---

## 6. Phase 4: 高级功能新发现

### 6.1 Research Wiki 的 Anti-Repetition

**前两轮理解**：
```
Research Wiki 有 Anti-Repetition 功能。
- 记录失败的 Idea
- /idea-creator 读取后避免重复
```

**新发现**：

```python
# 1. 失败 Idea 的记录
# - outcome: negative/mixed 的 Idea 会被记录
# - 包含失败原因和教训
# - 存储在 ideas/ 目录

# 2. query_pack.md 的压缩逻辑
# - 失败 Idea 是高优先级信息
# - 最多 1400 字符
# - 格式："- **title**: failure reason..."

# 3. /idea-creator 的读取逻辑
# - 读取 query_pack.md
# - 解析失败 Idea 列表
# - 生成新 Idea 时避免类似方向
# - 但 Phase 3 的 same-family 过滤可能跳过这一步

# 4. 确定性写入器的作用
# - ingest_paper(): 论文摄入
# - add_claim(): 声明记录
# - upsert_idea(): Idea 记录
# - add_experiment(): 实验记录
# - 每个写入器都有 dedup 逻辑
```

**关键认知升级**：
- Anti-Repetition 是通过 query_pack.md 实现的
- 确定性写入器避免了 LLM freehand 的不稳定性
- 但 Phase 3 的 same-family 过滤可能绕过 Anti-Repetition

### 6.2 Meta-Optimize 的自我进化

**前两轮理解**：
```
/meta-optimize 分析 ARIS 使用日志，提出改进建议。
- 分析 SKILL.md
- 提出改进
- 应用改进
```

**新发现**：

```python
# 1. 日志收集
# - .aris/meta/events.jsonl 记录所有事件
# - 包括 Skill 调用、审稿结果、错误等
# - 通过 hooks 自动收集

# 2. 分析逻辑
# - 识别失败模式
# - 识别效率瓶颈
# - 识别质量问题
# - 生成改进建议

# 3. 改进的应用
# - /meta-apply 是唯一可以修改 Skill 的 Skill
# - 需要人类确认
# - 需要跨模型 Jury PASS

# 4. 安全机制
# - 只能修改 SKILL.md，不能修改核心代码
# - 所有修改都有审计 trail
# - 可以回滚
```

**关键认知升级**：
- Meta-Optimize 是 ARIS 的"自我进化"机制
- 有严格的安全机制
- 需要人类确认和跨模型审查

### 6.3 Rebuttal 的安全门控

**前两轮理解**：
```
/rebuttal 生成 Rebuttal。
- 三道安全门
- 两版输出
```

**新发现**：

```python
# 1. 三道安全门
# - 不编造：每句话有出处
# - 不过度承诺：没批准的不承诺
# - 全覆盖：每个审稿意见都追踪

# 2. 两版输出
# - PASTE_READY.txt：精确字数，直接粘贴
# - REBUTTAL_DRAFT_rich.md：详细版，自己改

# 3. Stress Test
# - GPT-5.5 对 Rebuttal 进行压力测试
# - 检查是否有漏洞
# - 检查是否过度承诺

# 4. Follow-up Rounds
# - 最多 3 轮 follow-up
# - 每轮都可以修改 Rebuttal
# - 最终版本是经过多轮审查的
```

**关键认知升级**：
- 三道安全门是精心设计的
- Stress Test 可以发现 Rebuttal 的漏洞
- Follow-up Rounds 可以迭代改进

---

## 7. 关键认知升级

### 7.1 从"概念"到"实现"

**前两轮**：
```
理解概念：
- 跨模型对抗协作
- Type-A/Type-B 门控
- Fan-Out Pattern
```

**第三轮**：
```
理解实现：
- MCP 协议的具体流程
- Skill 的加载和执行机制
- 审稿 Prompt 的结构
- 状态持久化的实现
```

### 7.2 从"功能"到"权衡"

**前两轮**：
```
理解功能：
- ARIS 可以做什么
- 每个模块的作用
```

**第三轮**：
```
理解权衡：
- 成本 vs 质量
- 速度 vs 深度
- 自动化 vs 人类控制
- 安全性 vs 灵活性
```

### 7.3 从"理想"到"现实"

**前两轮**：
```
理想情况：
- ARIS 完全自动化
- 所有功能正常工作
- 没有任何问题
```

**第三轮**：
```
现实情况：
- API 可能失败
- GPU 可能不足
- 时间可能不够
- 成本可能超支
- 需要降级和 fallback
```

### 7.4 从"技术"到"工程"

**前两轮**：
```
技术视角：
- 算法原理
- 设计决策
```

**第三轮**：
```
工程视角：
- 可靠性
- 可维护性
- 可扩展性
- 成本效益
```

---

## 8. 面试回答优化

### 8.1 底层原理回答优化

**前两轮回答**：
```
"ARIS 使用跨模型对抗协作，让不同架构的 LLM 互相审查，打破 self-play 的盲区。"
```

**优化后回答**：
```
"ARIS 的核心是跨模型对抗协作。具体实现是：
1. Executor（Claude）负责执行：写代码、跑实验、写论文
2. Reviewer（GPT-5.5）负责审查：打分、找弱点、建议修改
3. 两者通过 MCP 协议通信，每次审稿都是 fresh thread

关键设计决策：
- 传递文件路径，不传递摘要（保持审稿独立性）
- 每次审稿都是新线程（防止 narrative accumulation）
- 停止条件是 score >= 6 AND verdict ∈ {ready, almost}（两个条件都必须满足）

这个设计的理论基础是 Adversarial Bandit vs Stochastic Bandit：
- 单模型自审是 stochastic bandit（噪声可预测）
- 跨模型审查是 adversarial bandit（对手会主动找弱点）
- 2-player games 收敛到 Nash equilibrium 更高效"
```

### 8.2 实验验证回答优化

**前两轮回答**：
```
"ARIS 通过跨模型审查来验证实验的有效性。"
```

**优化后回答**：
```
"ARIS 通过多层验证确保实验的有效性：

1. Cross-Model Code Review（部署前）
   - GPT-5.5 审查 Claude 的代码
   - 检查逻辑错误、超参数、ground truth
   - 避免浪费 GPU 时间

2. Sanity Check（部署后）
   - 运行最小实验验证环境
   - 最多 3 次自动 debug
   - 失败时可以调用 /codex:rescue（第二意见）

3. Auto Review Loop（结果后）
   - 跨模型审稿（2-4 轮）
   - 每轮都有 score 和 verdict
   - 停止条件：score >= 6 AND verdict ∈ {ready, almost}

4. Assurance Audits（论文前）
   - /experiment-audit: 实验完整性
   - /paper-claim-audit: 数字准确性
   - /citation-audit: 引用正确性
   - /kill-argument: 最强反驳

关键点：Executor 不能 judge 自己的实验 integrity。
所有质量判断都必须由不同模型家族做出。"
```

### 8.3 问题定位回答优化

**前两轮回答**：
```
"ARIS 有 watchdog 和 iteration log 来检测问题。"
```

**优化后回答**：
```
"ARIS 有完整的故障检测和恢复机制：

1. Watchdog（静默死亡检测）
   - 监控状态文件的 mtime
   - STALE/MISSING/COMPLETED 写入 alerts.log
   - 仅检测，不重启（避免破坏裁决）

2. Iteration Log（认知打转检测）
   - 计数每个 tick 的新发现
   - stale_count ≥ 2 → 强制结构性 pivot（换 frame + 选未试过的方向）
   - stale_count ≥ 4 → 升级给人类

3. Resumable Runs（故障恢复）
   - .aris/runs/<run_id>.json 记录每个阶段的状态
   - 支持 resume 而不是 restart
   - 重新验证 done 但未 accepted 的阶段

4. Stream Retry（流式传输故障）
   - ARIS_STREAM_RETRY: chunk decode 失败时整段重启
   - 只在尚未输出任何内容时触发
   - 避免"error decoding response body"循环

关键点：这些机制都是 Type-A 信号——只说'继续/换方向'，从不说'够好了'。
质量裁决仍然终结于跨模型裁判席。"
```

### 8.4 工程落地回答优化

**前两轮回答**：
```
"ARIS 有 Resumable Runs 和安全机制。"
```

**优化后回答**：
```
"ARIS 在工程落地方面有几个关键设计：

1. Resumable Runs（可恢复运行）
   - 状态文件：.aris/runs/<run_id>.json
   - 原子写入：tempfile + rename
   - 恢复逻辑：重新验证依赖关系
   - 支持 context compaction 后恢复

2. 安全机制
   - Injection Hygiene: threat_scan.py 扫描注入 payload
   - Sandbox StrictMode: 忽略所有 LLM-supplied override
   - System Prompt 脱敏: 敏感 key 替换为 [REDACTED]
   - MCP 版本验证: 不支持的版本终止子进程

3. MCP 工具调度
   - 协议版本验证
   - 传输层自动识别（LSP-style vs NDJSON）
   - 错误处理和重试
   - 连接池管理

4. 系统稳定性
   - 资源管理：max_connections, max_memory_mb
   - 超时控制：tokio::time::timeout
   - 重试机制：指数退避
   - 监控告警：GPU 温度、内存使用

关键点：这些设计确保了 ARIS 可以长时间稳定运行。
例如，通宵运行的 /research-pipeline 可以在中断后 resume，不会丢失进度。"
```

### 8.5 业务场景回答优化

**前两轮回答**：
```
"ARIS 适合学术研究，可以节省时间、提高质量。"
```

**优化后回答**：
```
"ARIS 的业务价值体现在几个方面：

1. 适合场景
   - 学术研究：论文写作、实验、审稿（核心场景）
   - 技术调研：文献综述、竞品分析
   - 专利撰写：发明披露、权利要求
   - 技术报告：项目文档、技术方案

2. 用户关心指标
   - 时间节省：论文写作从 2 周缩短到 2 天
   - 质量提升：论文分数平均提升 2-3 分
   - 成本控制：API 成本 ~$100-300/论文
   - 可靠性：自动 resume、审计 trail

3. 上线成本
   - API 成本：~$100-300/论文
   - GPU 成本：~$400-700/论文（电费）
   - 人力成本：显著降低
   - 总成本：~$1100-2800/论文

4. 资源受限优化
   - P0：跨模型审查、故障恢复、安全机制
   - P1：Research Wiki、实验自动化、成本控制
   - P2：UI/UX 改进、更多模型支持、更多功能

5. ROI 计算
   - 时间：4.5 个月 → 1 个月（节省 3.5 个月）
   - 成本：$36,000 → $2,450（节省 $33,550）
   - ROI：1370%

关键点：ARIS 的核心价值是'跨模型对抗协作'。
这不是简单的'让 AI 做科研'，而是'让不同架构的 AI 互相审查'。"
```

---

## 9. 实践心得与最佳实践

### 9.1 环境搭建心得

```
1. API Keys 管理
   - 使用环境变量，不要硬编码
   - 定期检查额度
   - 配置备份 Key

2. GPU 环境配置
   - 使用 conda 管理环境
   - 固定 PyTorch 和 CUDA 版本
   - 测试 GPU 可用性

3. 网络配置
   - 确保可以访问 API
   - 配置代理（如果需要）
   - 测试连接稳定性
```

### 9.2 实验设计心得

```
1. 从小开始
   - 先用 lite effort level
   - 先用小数据集
   - 先用简单模型

2. 迭代改进
   - 每轮审稿后分析问题
   - 优先修复高影响问题
   - 记录每次改进

3. 资源管理
   - 监控 API 使用量
   - 监控 GPU 使用率
   - 定期清理日志
```

### 9.3 问题调试心得

```
1. 日志分析
   - 保存所有日志
   - 使用 grep 搜索关键信息
   - 分析错误模式

2. 状态检查
   - 检查 .aris/runs/ 目录
   - 检查 REVIEW_STATE.json
   - 检查 EXPERIMENT_TRACKER.md

3. 逐步排查
   - 先检查环境
   - 再检查配置
   - 最后检查代码
```

### 9.4 面试准备心得

```
1. 理解原理
   - 不要只记概念
   - 要理解设计决策
   - 要理解权衡

2. 积累经验
   - 实际操作比理论更重要
   - 记录遇到的问题
   - 总结解决方案

3. 优化表达
   - 用具体例子说明
   - 用数据支撑观点
   - 用类比解释概念
```

---

## 附录：学习资源

### A. 官方文档

```
ARIS 项目：
├── README.md: 项目概述
├── AGENT_GUIDE.md: Agent 指南
├── SETUP_GUIDE.md: 安装指南
├── docs/SKILLS_CATALOG.md: Skill 目录
└── skills/shared-references/: 共享参考文档

Claude Code：
├── 官方文档：https://docs.anthropic.com/en/docs/claude-code
└── MCP 协议：https://modelcontextprotocol.io/

Codex CLI：
├── 官方文档：https://developers.openai.com/codex
└── API 参考：https://platform.openai.com/docs
```

### B. 技术报告

```
ARIS 技术报告：
├── arXiv: https://arxiv.org/abs/2605.03042
└── Hugging Face: https://huggingface.co/papers/2605.03042

Anti-Autoresearch：
├── GitHub: https://github.com/wanshuiyin/Anti-Autoresearch
└── 39 种 hack-patterns，7 个 families
```

### C. 社区资源

```
GitHub：
├── Issues: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/issues
├── Discussions: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/discussions
└── Pull Requests: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/pulls

社区工具：
├── Claude Fleet: https://github.com/tianyilt/claude-fleet
├── posterly: https://github.com/Chenruishuo/posterly
└── ARIS-Anything: https://github.com/wanshuiyin/ARIS-Anything
```

---

*最后更新：2026-06-27*
*基于 ARIS 项目版本：v0.4.20*
*学习阶段：Phase 0-1 完成，Phase 2 进行中*
