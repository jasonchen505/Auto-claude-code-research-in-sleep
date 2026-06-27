# ARIS 项目面试准备指南
> 针对 LLM 算法实习岗位，深度挖掘 Auto-claude-code-research-in-sleep (ARIS) 项目中的核心技术点

---

## 目录
1. [项目概述与核心价值](#1-项目概述与核心价值)
2. [关键技术架构深度解析](#2-关键技术架构深度解析)
3. [LLM & Agent 核心考察点](#3-llm--agent-核心考察点)
4. [应用 & 后训练相关问题](#4-应用--后训练相关问题)
5. [Auto-Research 领域深入探讨](#5-auto-research-领域深入探讨)
6. [系统设计与工程实践](#6-系统设计与工程实践)
7. [高频面试问题与参考答案](#7-高频面试问题与参考答案)
8. [项目亮点总结与个人陈述建议](#8-项目亮点总结与个人陈述建议)

---

## 1. 项目概述与核心价值

### 1.1 一句话定义
ARIS (Auto Research in Sleep) 是一个**基于 Claude Code 的自主 ML 科研工作流框架**，通过跨模型对抗协作实现从 idea 生成到论文投稿的全流程自动化。

### 1.2 核心创新点
```
传统科研流程: 人做实验 → 人写论文 → 人审稿 → 人修改
ARIS 流程:    AI 做实验 → AI 写论文 → 跨模型 AI 审稿 → AI 修改
              ↑ 人类只需要在关键节点做决策
```

**面试关键表述**：
> "ARIS 的核心不是让一个 AI 做所有事，而是让**不同模型家族**互相审查——就像学术界的 peer review，但用不同架构的 LLM 来打破 self-play 的盲区。"

### 1.3 项目规模与影响力
- **79 个 Skills**：覆盖科研全流程（文献调研、idea 生成、实验、论文写作、rebuttal）
- **跨平台支持**：Claude Code、Codex CLI、Cursor、Trae、Antigravity、Copilot CLI
- **Hugging Face Daily Paper #1**（2026.05）
- **技术报告**：arXiv:2605.03042

---

## 2. 关键技术架构深度解析

### 2.1 Skill-as-Markdown 设计哲学

**核心理念**：整个 skill 层就是纯 Markdown 文件，没有框架、没有数据库、没有 Docker。

```markdown
# 每个 Skill 的结构
---
name: auto-review-loop
description: "Autonomous multi-round research review loop..."
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Skill, mcp__codex__codex
---

# 具体的工作流指令...
```

**面试深挖点**：
- **Q**: 为什么选择 Markdown 而不是代码框架？
- **A**: 
  1. **零锁定**：任何 LLM 都能读懂，可以无缝切换 Claude/Codex/Cursor
  2. **可组合性**：Skills 可以像函数一样调用其他 Skills
  3. **版本控制友好**：纯文本，Git 友好
  4. **人类可读**：非工程师也能理解和修改

### 2.2 跨模型对抗协作架构

这是 ARIS 最核心的设计，也是面试中最值得深挖的点：

```
┌─────────────────────────────────────────────────────────┐
│                    Executor (Claude)                      │
│  - 写代码、跑实验、写论文                                   │
│  - 速度快、执行流畅                                        │
└─────────────────────┬───────────────────────────────────┘
                      │ 文件路径（不含摘要）
                      ▼
┌─────────────────────────────────────────────────────────┐
│                  Reviewer (GPT-5.5)                      │
│  - 打分、找弱点、建议修改                                   │
│  - 严谨、深入、批判性                                      │
│  - 每次调用都是 fresh thread（防止 narrative accumulation）│
└─────────────────────────────────────────────────────────┘
```

**关键设计原则**：

1. **Reviewer Independence（审稿者独立性）**
   ```markdown
   ✅ 可以传递：文件路径、审稿目标、venue 约束
   ❌ 不能传递：执行者的摘要、解释、建议、结论
   ```
   - **原因**：如果执行者"指导"审稿者，就失去了跨模型审查的意义

2. **Thread Freshness（线程新鲜度）**
   - 每次审稿都用新的 `mcp__codex__codex` 线程
   - 不用 `codex-reply`（会导致 narrative accumulation，分数虚高）
   - **类比**：就像每次审稿都找一个新的 reviewer，而不是让同一个 reviewer 看多轮

3. **Cross-Model Invariant（跨模型不变量）**
   ```
   规则：executor 和 reviewer 必须是不同的模型家族
   原因：同家族模型共享训练先验和盲区
   类比：stochastic bandit vs adversarial bandit
   ```

**面试深挖点**：
- **Q**: 为什么不用同一个模型的不同 temperature 或者不同 prompt？
- **A**: 这是 stochastic bandit（噪声可预测）vs adversarial bandit（对手会主动找弱点）的区别。同家族模型的 agreement 是 correlated blindness，不是 consensus。

### 2.3 Type-A / Type-B 门控系统

这是 ARIS 防止"自我赦免"的核心机制：

```python
# Type-A 门控：执行/客观信号（可以自判）
✅ exit code == 0
✅ 文件存在 / PDF 编译成功
✅ N/N jobs 完成
✅ 测试通过

# Type-B 门控：质量/正确性/接受度（必须跨模型判）
❌ "论文够好了"
❌ "证明有效"
❌ "claim 被结果支持"
❌ "idea 是新颖的"
```

**核心原则**：
> "A loop can DRIVE; it cannot ACQUIT."
> 循环可以驱动自己前进，但不能赦免自己的工作。

**面试深挖点**：
- **Q**: 如何判断一个门控是 Type-A 还是 Type-B？
- **A**: 问自己："一个没有品味的脚本能回答这个问题吗？"
  - 能 → Type-A（执行记账）
  - 不能，需要品味/正确性判断 → Type-B（需要不同模型家族）

### 2.4 Fan-Out Pattern（扇出模式）

当需要广度时（多个候选 idea、多个攻击角度），ARIS 使用扇出模式：

```
┌─────────────────────────────────────────────────────────┐
│                    Fan-Out (火力)                         │
│  - 同家族子代理生成 N 个候选                                │
│  - 只生成，不评分                                         │
│  - 3 层降级：并行 → Agent 工具 → 顺序执行                  │
└─────────────────────┬───────────────────────────────────┘
                      │ 合并 + 机械去重
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    Jury (裁判席)                          │
│  - 不同模型家族                                           │
│  - 只评分，不生成                                         │
│  - 始终是同一个跨模型步骤                                  │
└─────────────────────────────────────────────────────────┘
```

**关键规则**：
- ✅ Fan out 搜索候选
- ❌ 永远不要 fan out 裁判席
- **原因**：N 个 Claude 的 agreement 是 correlated blindness，不是 consensus

---

## 3. LLM & Agent 核心考察点

### 3.1 Agent 架构设计

**ARIS 的 Agent 架构**：
```
Orchestrator (Claude Code)
├── Skills (Markdown 指令)
├── Tools (Python/Bash 脚本)
├── MCP Servers (Codex MCP, Gemini MCP)
└── State Management (JSON 状态文件)
```

**面试问题**：
1. **Q**: ARIS 如何处理长时间运行的任务（如通宵实验）？
   - **A**: 
     - **Resumable Runs**：通过 `run_state.py` 记录每个阶段的状态
     - **Watchdog**：`watchdog.py` 监控状态文件的 mtime，检测静默死亡
     - **Iteration Log**：`iteration_log.py` 计数新发现，强制结构性 pivot
     - **Heartbeat**：`/loop` 或 `CronCreate` 定期唤醒，但只能检测，不能裁决

2. **Q**: 如何防止 Agent 陷入死循环或局部最优？
   - **A**:
     - **MAX_ROUNDS**：硬性上限（如 auto-review-loop 最多 4 轮）
     - **Stale Count**：连续无新发现时强制 pivot
     - **Cross-Model Jury**：质量判断必须由不同模型家族做出
     - **Human Escalation**：stale_count ≥ 4 时升级给人类

### 3.2 Prompt Engineering 实践

**ARIS 的 Prompt 设计特点**：

1. **结构化指令**：
   ```markdown
   ## Constants
   - MAX_ROUNDS = 4
   - POSITIVE_THRESHOLD: score >= 6/10 AND verdict ∈ {"ready", "almost"}
   
   ## Workflow
   ### Phase A: Review
   [具体指令...]
   ```

2. **防御性 Prompt**：
   ```markdown
   > ⚠️ **Nightmare + Manual incompatibility**: 
   > If REVIEWER_BACKEND = manual and REVIEWER_DIFFICULTY = nightmare, STOP with:
   > "difficulty: nightmare requires Codex CLI..."
   ```

3. **Anti-Hallucination 指令**：
   ```markdown
   - NEVER fabricate BibTeX. Use DBLP → CrossRef → [VERIFY] chain
   - Do NOT generate BibTeX from memory
   ```

**面试问题**：
- **Q**: 如何设计一个可靠的 Agent 系统的 Prompt？
- **A**:
  1. **明确边界**：什么可以做，什么不能做
  2. **状态感知**：Agent 需要知道自己在哪个阶段
  3. **失败处理**：明确失败时的行为
  4. **可验证性**：输出应该是结构化的，便于验证

### 3.3 Tool Use & MCP 协议

**ARIS 的工具使用**：

1. **MCP (Model Context Protocol)**：
   ```json
   {
     "mcpServers": {
       "codex": {
         "command": "codex",
         "args": ["mcp-server"],
         "trust": true
       }
     }
   }
   ```

2. **工具调度**：
   - `mcp__codex__codex`：启动新的审稿线程
   - `mcp__codex__codex-reply`：在同一线程中继续对话
   - `Bash(*)`：执行任意 shell 命令
   - `Read/Write/Edit`：文件操作

3. **安全机制**：
   ```rust
   // PermissionMode::Prompt 的 derived-Ord bug
   // v0.4.6 修复：之前静默放过所有 tool
   ```

**面试问题**：
- **Q**: MCP 协议的核心设计是什么？
- **A**:
  - **stdio 传输**：进程间通信
  - **JSON-RPC**：消息格式
  - **工具发现**：`tools/list` 返回可用工具
  - **工具调用**：`tools/call` 执行工具
  - **版本协商**：客户端和服务端协商协议版本

### 3.4 Multi-Agent 协作

**ARIS 的多代理模式**：

1. **Executor-Reviewer 模式**：
   - Claude 做执行
   - GPT-5.5 做审查
   - 两者不互相评价自己的工作

2. **Fan-Out 模式**：
   - 多个 Claude 子代理并行生成候选
   - 单一跨模型 jury 做裁决

3. **Debate Protocol**（hard/nightmare 模式）：
   ```
   Executor 写 rebuttal → Reviewer 做 ruling → 更新分数
   ```

**面试问题**：
- **Q**: 多代理系统中的通信机制有哪些？
- **A**:
  1. **共享文件系统**：Skills 通过文件传递状态
  2. **MCP 调用**：跨模型通信
  3. **Thread ID**：保持对话上下文
  4. **State JSON**：持久化状态，支持恢复

---

## 4. 应用 & 后训练相关问题

### 4.1 RLHF 与 ARIS 的关系

**面试问题**：
- **Q**: ARIS 的跨模型审查和 RLHF 有什么相似之处？

**参考答案**：
```
RLHF: 人类反馈 → 训练模型 → 模型生成更好输出
ARIS: 跨模型反馈 → 迭代改进 → 论文质量提升

相似点：
1. 都需要外部信号来改进（人类/不同模型）
2. 都是迭代过程
3. 都需要防止 reward hacking（过度优化指标/自我赦免）

不同点：
1. RLHF 改变模型权重，ARIS 改变输出内容
2. RLHF 是训练时，ARIS 是推理时
3. RLHF 的 reward model 可能被 hack，ARIS 的跨模型审查更难被 hack
```

### 4.2 后训练中的对齐问题

**面试问题**：
- **Q**: ARIS 如何确保 Agent 的行为符合人类意图？

**参考答案**：
1. **Human Checkpoint**：
   ```markdown
   — human checkpoint: true | false  # 是否暂停等待人类批准
   — AUTO_PROCEED: true | false      # 是否自动继续
   ```

2. **Assurance Levels**：
   - `draft`：快速迭代，审计可选
   - `submission`：严格审计，必须通过

3. **Cross-Model Jury**：
   - 质量判断必须由不同模型家族做出
   - 防止"自己给自己打高分"

4. **Anti-Autoresearch**：
   - 检测 39 种 autoresearch hack-patterns
   - 7 个 family 的欺诈模式

### 4.3 Inference-Time Scaling

**面试问题**：
- **Q**: ARIS 如何在推理时扩展计算？

**参考答案**：
1. **Fan-Out Pattern**：
   - 并行生成多个候选
   - 增加搜索广度

2. **Multi-Round Review**：
   - 最多 4 轮审稿-修改循环
   - 增加迭代深度

3. **Effort Levels**：
   ```markdown
   — effort: lite | balanced | max | beast
   ```
   - 控制 papers/rounds/ideation 的数量

4. **Debate Protocol**：
   - Executor rebuttal → Reviewer ruling
   - 增加对抗深度

### 4.4 模型选择与组合

**面试问题**：
- **Q**: 为什么选择 Claude + GPT-5.5 的组合？

**参考答案**：
1. **互补性**：
   - Claude：快速、流畅的执行
   - GPT-5.5：慢但严谨、深入的批判

2. **打破盲区**：
   - 同家族模型共享训练先验
   - 不同架构的模型有不同的 failure modes

3. **最小配置**：
   - 2 个模型是打破 self-play 盲区的最小配置
   - 2-player games 收敛到 Nash equilibrium 更高效

4. **成本考虑**：
   - 1→2 的提升最大
   - 2→4 的边际收益递减

---

## 5. Auto-Research 领域深入探讨

### 5.1 Auto-Research 的挑战

**面试问题**：
- **Q**: Auto-Research 面临哪些核心挑战？

**参考答案**：

1. **幻觉问题**：
   - LLM 可能编造不存在的论文
   - 解决方案：`verify_papers.py` 3 层交叉验证（arXiv/CrossRef/S2）

2. **自我赦免**：
   - 模型可能给自己的工作打高分
   - 解决方案：跨模型审查 + Type-A/Type-B 门控

3. **实验诚信**：
   - 可能使用假的 ground truth
   - 解决方案：`/experiment-audit` 跨模型完整性检查

4. **长程记忆**：
   - 通宵运行时可能忘记之前的发现
   - 解决方案：Research Wiki 持久化知识库

5. **认知打转**：
   - 卡住时可能无限重试近似变体
   - 解决方案：`iteration_log.py` 强制结构性 pivot

### 5.2 Research Wiki 设计

**面试问题**：
- **Q**: Research Wiki 如何解决长程记忆问题？

**参考答案**：

```python
# Wiki 的 4 层节点
papers/     # 论文摘要和元数据
ideas/      # 研究想法（成功/失败）
experiments/ # 实验记录
claims/     # 研究声明（verified/refuted/unproven）

# 关系图
graph/edges.jsonl  # 节点之间的关系

# 关键功能
ingest_paper()   # 论文摄入
add_claim()      # 声明记录
upsert_idea()    # idea 记录
add_experiment() # 实验记录

# Anti-Repetition
query_pack.md    # 压缩的知识包，包含 failed ideas
```

**设计亮点**：
1. **确定性写入器**：每个节点层都有专属的 Python 写入器
2. **声明状态机**：`drafted → unproven → sound-modulo-imports → verified/refuted`
3. **注入扫描**：防止恶意 prompt injection
4. **Anti-Repetition**：失败的 idea 会被记录，防止重复尝试

### 5.3 实验自动化

**面试问题**：
- **Q**: ARIS 如何自动化实验流程？

**参考答案**：

```
实验计划 → 代码实现 → 跨模型代码审查 → Sanity Check → 部署 → 收集结果
    │           │            │              │           │         │
    ▼           ▼            ▼              ▼           ▼         ▼
EXPERIMENT  Claude      GPT-5.5        最小实验     GPU 集群   JSON/CSV
  _PLAN.md   写代码      审查代码        验证环境      并行执行    结构化
```

**关键设计**：
1. **Cross-Model Code Review**：
   - GPT-5.5 在部署前审查代码
   - 捕获逻辑错误，避免浪费 GPU 时间

2. **Sanity First**：
   - 先跑最小实验验证环境
   - 失败时自动 debug（最多 3 次尝试）

3. **Queue Routing**：
   - ≤5 jobs → `/run-experiment`
   - ≥10 jobs → `/experiment-queue`（OOM retry, wave gating）

4. **结果收集**：
   - 自动解析 JSON/CSV
   - 更新 EXPERIMENT_TRACKER.md

---

## 6. 系统设计与工程实践

### 6.1 错误处理与恢复

**面试问题**：
- **Q**: ARIS 如何处理长时间运行中的失败？

**参考答案**：

1. **Resumable Runs**：
   ```python
   # .aris/runs/<run_id>.json
   {
     "phases": {
       "idea-discovery": {"status": "accepted", "verdict_id": "..."},
       "experiment-bridge": {"status": "done", "artifact": "..."},
       "auto-review-loop": {"status": "running"}
     }
   }
   ```

2. **Watchdog**：
   ```python
   # watchdog.py 监控状态文件的 mtime
   # STALE / MISSING / COMPLETED 写入 alerts.log
   # 仅检测，不重启
   ```

3. **Iteration Log**：
   ```python
   # iteration_log.py 计数新发现
   # stale_count ≥ 2 → 强制结构性 pivot
   # stale_count ≥ 4 → 升级给人类
   ```

4. **Stream Retry**：
   ```rust
   // ARIS_STREAM_RETRY: chunk decode 失败时整段重启
   // 只在尚未输出任何内容时触发
   ```

### 6.2 安全设计

**面试问题**：
- **Q**: ARIS 有哪些安全机制？

**参考答案**：

1. **System Prompt 配置脱敏**（v0.4.14）：
   ```rust
   // 敏感 key 替换为 [REDACTED]
   // apikey/token/secret/password/authorization
   // MCP command 替换为 <configured>
   // MCP url 仅保留 scheme://host[:port]
   ```

2. **Sandbox StrictMode**：
   ```json
   {
     "sandbox": {
       "strictMode": true  // 忽略所有 LLM-supplied override
     }
   }
   ```

3. **Injection Hygiene**：
   ```python
   # threat_scan.py 扫描注入 payload
   # quarantine() 隔离可疑内容
   # 保留原始文本供人类审查
   ```

4. **MCP 协议版本协商**：
   ```rust
   // 验证协商版本是否在支持集合中
   // 不支持则终止子进程 + 清槽 + 报错
   ```

### 6.3 性能优化

**面试问题**：
- **Q**: ARIS 如何优化性能？

**参考答案**：

1. **并行执行**：
   - Fan-Out Pattern：并行生成多个候选
   - Tier 1/2/3 降级：ultracode → Agent 工具 → 顺序执行

2. **流式处理**：
   - SSE (Server-Sent Events) 流式响应
   - `stream_options.include_usage: true` 解析 cached_tokens

3. **缓存机制**：
   - Research Wiki 持久化知识
   - query_pack.md 压缩知识包
   - REVIEW_STATE.json 恢复上下文

4. **成本控制**：
   - Mechanical dedup 在 jury 前减少输入
   - MAX_ROUNDS 硬性上限
   - Effort levels 控制深度

---

## 7. 高频面试问题与参考答案

### 7.1 基础概念题

**Q1: 什么是 Agent？ARIS 中的 Agent 有什么特点？**

**A**: Agent 是能够自主执行任务的 LLM 系统。ARIS 中的 Agent 有以下特点：
1. **Skill-Based**：每个任务是一个 Markdown 文件，定义工作流
2. **Cross-Model**：不同模型家族协作，打破 self-play 盲区
3. **Resumable**：支持长时间运行和故障恢复
4. **Auditable**：所有审稿都有 trace，可追溯

**Q2: 什么是 MCP？在 ARIS 中起什么作用？**

**A**: MCP (Model Context Protocol) 是 Anthropic 提出的协议，用于 LLM 和工具之间的通信。
- **作用**：ARIS 通过 MCP 调用 Codex（GPT-5.5）作为 reviewer
- **关键点**：
  - stdio 传输 + JSON-RPC
  - 工具发现和调用
  - 版本协商和安全机制

**Q3: 解释 ARIS 的跨模型对抗协作**

**A**: 
```
传统 self-play: 同一个模型给自己打分 → 容易 game
ARIS: 不同模型家族互相审查 → 更难被 hack

类比：
- Stochastic bandit: 单模型自审（噪声可预测）
- Adversarial bandit: 跨模型审查（对手会主动找弱点）

最小配置：2 个模型（1→2 提升最大，2→4 边际收益递减）
```

### 7.2 技术深挖题

**Q4: ARIS 如何防止"自我赦免"（self-acquittal）？**

**A**: 通过 Type-A/Type-B 门控系统：
```python
# Type-A：执行/客观信号（可以自判）
✅ exit code == 0, 文件存在, jobs 完成

# Type-B：质量/正确性（必须跨模型判）
❌ "论文够好了", "证明有效", "idea 新颖"

判断标准："一个没有品味的脚本能回答这个问题吗？"
- 能 → Type-A
- 不能 → Type-B（需要不同模型家族）
```

**Q5: Research Wiki 如何解决长程记忆问题？**

**A**: 
```python
# 4 层节点 + 关系图
papers/ ideas/ experiments/ claims/ graph/

# 确定性写入器（非 LLM freehand）
ingest_paper() add_claim() upsert_idea() add_experiment()

# Anti-Repetition
failed ideas → query_pack.md → /idea-creator 读取 → 避免重复

# 声明状态机
drafted → unproven → verified/refuted
```

**Q6: ARIS 的 Fan-Out Pattern 是什么？**

**A**: 
```
Fan-Out (火力)          Jury (裁判席)
├─ 同家族子代理          ├─ 不同模型家族
├─ 生成 N 个候选         ├─ 只评分，不生成
├─ 只生成，不评分        ├─ 始终是同一个跨模型步骤
└─ 3 层降级              └─ 不随 tier 变化

关键规则：
✅ Fan out 搜索候选
❌ 永远不要 fan out 裁判席

原因：N 个 Claude 的 agreement 是 correlated blindness
```

### 7.3 系统设计题

**Q7: 如何设计一个可靠的长时间运行的 Agent 系统？**

**A**: 
```
1. 状态持久化
   - run_state.py 记录每个阶段的状态
   - 支持 resume 而不是 restart

2. 故障检测
   - watchdog.py 监控 mtime
   - iteration_log.py 计数新发现

3. 强制 pivot
   - stale_count ≥ 2 → 结构性 pivot
   - stale_count ≥ 4 → 升级给人类

4. 质量保证
   - 跨模型审查
   - Type-A/Type-B 门控
   - MAX_ROUNDS 硬性上限
```

**Q8: ARIS 如何处理实验失败？**

**A**: 
```
Sanity Check → 失败 → 自动 Debug（最多 3 次）
                       │
                       ├─ 读取错误（traceback, stderr）
                       ├─ 分类失败（OOM, ImportError, CUDA error）
                       ├─ 应用修复
                       └─ 仍然失败 → Codex rescue（第二意见）
                                      │
                                      └─ 仍然失败 → 停止，报告所有尝试

原则："Never give up on the first failure"
```

**Q9: 如何确保实验的诚信？**

**A**: 
```markdown
1. Ground Truth 检查
   - 必须使用数据集提供的 ground truth
   - 禁止使用模型输出作为 ground truth

2. Score Normalization 检查
   - 禁止除以模型自己的 max/min
   - 必须包含所有方法（包括 baseline）

3. Phantom Results 检查
   - 每个数字必须追溯到实际输出文件
   - 禁止引用不存在的函数

4. 跨模型审计
   - /experiment-audit 由不同模型家族执行
   - Executor 不能 judge 自己的实验诚信
```

### 7.4 开放讨论题

**Q10: Auto-Research 的未来发展方向是什么？**

**A**: 
```
1. 更强的自主性
   - 减少人类干预
   - 更好的故障恢复

2. 更广的适用性
   - 从学术研究扩展到其他领域
   - ARIS-Anything: 投资尽调、法律研究、市场研究

3. 更深的集成
   - 与实验基础设施深度集成
   - 自动 GPU 调度和优化

4. 更好的评估
   - Anti-Autoresearch: 检测欺诈模式
   - 更严格的诚信检查

5. 多模态扩展
   - ARIS-Movie-Director: 图像电影生成
   - 跨模态的对抗审查
```

**Q11: 如果让你改进 ARIS，你会怎么做？**

**A**: 
```
1. 更智能的 idea 生成
   - 结合 knowledge graph
   - 自动发现研究空白

2. 更高效的实验
   - 自动超参数优化
   - 多保真度优化

3. 更好的写作
   - 自动图表生成
   - 风格迁移

4. 更强的安全性
   - 更严格的注入防护
   - 更好的审计 trail

5. 更广的集成
   - 更多 LLM 支持
   - 更多实验平台
```

---

## 8. 项目亮点总结与个人陈述建议

### 8.1 项目亮点

1. **创新的跨模型对抗协作**
   - 打破 self-play 盲区
   - 类型-A/Type-B 门控系统
   - Fan-Out Pattern

2. **完整的科研工作流**
   - 79 个 Skills 覆盖全流程
   - 从 idea 到论文到 rebuttal

3. **可靠的工程实践**
   - Resumable Runs
   - Watchdog + Iteration Log
   - 安全机制（injection hygiene, sandbox strictMode）

4. **开源影响力**
   - Hugging Face Daily Paper #1
   - 多平台支持
   - 活跃的社区

### 8.2 个人陈述建议

**开头**：
> "我在研究 ARIS（Auto Research in Sleep）项目，这是一个基于 Claude Code 的自主科研工作流框架。它的核心创新是跨模型对抗协作——让不同架构的 LLM 互相审查，打破 self-play 的盲区。"

**技术深挖**：
> "我深入研究了它的 Type-A/Type-B 门控系统。Type-A 是执行信号（如 exit code == 0），可以自判；Type-B 是质量信号（如'论文够好了'），必须由不同模型家族判断。这防止了'自我赦免'的问题。"

**工程实践**：
> "在工程方面，ARIS 使用 Research Wiki 解决长程记忆问题，用 Watchdog 和 Iteration Log 处理通宵运行中的故障。它的 Fan-Out Pattern 很巧妙——同家族子代理生成候选（火力），不同模型家族做裁决（裁判席），两者严格分离。"

**未来展望**：
> "我认为 Auto-Research 的未来在于更智能的 idea 生成、更高效的实验、更好的安全机制。ARIS-Anything 已经开始将这套方法论扩展到非学术领域，如投资尽调和法律研究。"

### 8.3 可能的追问

**Q: 你在这个项目中做了什么贡献？**

**A**: 
```
1. 研究和理解
   - 深入分析了 79 个 Skills 的设计
   - 理解了跨模型对抗协作的原理

2. 技术总结
   - 提炼了 Type-A/Type-B 门控系统
   - 总结了 Fan-Out Pattern

3. 面试准备
   - 整理了高频面试问题
   - 准备了技术深挖的答案
```

**Q: 这个项目最大的挑战是什么？**

**A**: 
```
1. 防止自我赦免
   - 如何确保模型不给自己打高分
   - 解决方案：跨模型审查 + Type-A/Type-B 门控

2. 长程记忆
   - 通宵运行时如何记住之前的发现
   - 解决方案：Research Wiki + 确定性写入器

3. 实验诚信
   - 如何防止假的 ground truth
   - 解决方案：/experiment-audit 跨模型检查

4. 故障恢复
   - 如何处理长时间运行中的失败
   - 解决方案：Resumable Runs + Watchdog
```

---

## 附录：关键术语表

| 术语 | 解释 |
|------|------|
| ARIS | Auto Research in Sleep |
| MCP | Model Context Protocol |
| Fan-Out | 同家族子代理并行生成候选 |
| Type-A | 执行/客观信号，可以自判 |
| Type-B | 质量/正确性信号，必须跨模型判 |
| Self-Acquittal | 自我赦免，模型给自己的工作打高分 |
| Research Wiki | 持久化研究知识库 |
| Watchdog | 静默死亡检测器 |
| Iteration Log | 新发现计数器，强制结构性 pivot |
| Cross-Model Jury | 不同模型家族的裁决席 |
| Thread Freshness | 每次审稿都用新线程 |
| Narrative Accumulation | 多轮对话导致分数虚高 |

---

*最后更新：2026-06-27*
*基于 ARIS 项目版本：v0.4.20*
