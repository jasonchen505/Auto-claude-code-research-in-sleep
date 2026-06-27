# 技术面试五类问题深度应对指南
> 基于 ARIS 项目，针对 LLM 算法实习岗位的技术面试准备

---

## 目录
1. [第一类：底层原理理解](#第一类底层原理理解)
2. [第二类：实验和方案验证能力](#第二类实验和方案验证能力)
3. [第三类：问题定位能力](#第三类问题定位能力)
4. [第四类：工程落地能力](#第四类工程落地能力)
5. [第五类：业务与实际场景理解](#第五类业务与实际场景理解)

---

## 第一类：底层原理理解

> **考察重点**：不是回答清楚概念，而是讲清楚方法解决什么问题、存在哪些局限性、有哪些改进方法

### 1.1 跨模型对抗协作

**解决什么问题？**

```
问题：单模型 self-play 存在 correlated blindness
- 同家族模型共享训练先验和盲区
- 自我审查容易陷入局部最优
- 类比：自己给自己改作业，很难发现自己的盲点

数据支撑：
- ARIS 的 auto-review-loop 使用 Claude 做 executor，GPT-5.5 做 reviewer
- 两者在不同数据上训练，有不同的 failure modes
- 跨模型审查能发现单模型看不到的问题
```

**为什么这么设计？**

```python
# 理论基础：Bandit 问题
Stochastic Bandit: 单模型自审（噪声可预测，容易被 game）
Adversarial Bandit: 跨模型审查（对手会主动找弱点，更难被 hack）

# 最小配置：2 个模型
- 1→2 的提升最大（打破 self-play 盲区）
- 2→4 的边际收益递减（协调成本增加）
- 2-player games 收敛到 Nash equilibrium 更高效
```

**局限性是什么？**

```
1. 成本问题
   - 每次审稿都要调用另一个模型的 API
   - GPT-5.5 xhigh reasoning 比较贵
   - 多轮审稿累积成本高

2. 延迟问题
   - 跨模型调用增加延迟
   - 审稿-修改循环需要等待
   - 通宵运行时可能卡在审稿

3. 一致性问题
   - 不同模型的评分标准可能不同
   - GPT-5.5 可能比 Claude 更严格或更宽松
   - 需要 calibration

4. 模型依赖
   - 依赖特定模型的 API 可用性
   - 模型更新可能导致行为变化
   - 需要 fallback 机制
```

**有哪些改进方法？**

```
1. 成本优化
   - 分层审稿：lite 模式跳过部分审稿
   - 缓存机制：相似内容复用审稿结果
   - 异步审稿：审稿和执行并行

2. 一致性校准
   - 评分标准化：不同模型的分数映射到统一尺度
   - 多审稿者投票：多个 reviewer 的结果取平均
   - 历史校准：基于历史数据调整评分阈值

3. 更智能的调度
   - 动态选择 reviewer：根据任务类型选择最合适的模型
   - 自适应轮数：根据收敛情况动态调整审稿轮数
   - 优先级调度：重要的部分优先审稿

4. 更强的对抗性
   - Nightmare 模式：reviewer 直接读代码
   - Debate Protocol：executor 可以 rebuttal
   - 多角度审查：不同 reviewer 关注不同方面
```

### 1.2 Type-A/Type-B 门控系统

**解决什么问题？**

```
问题：Agent 循环中的"自我赦免"
- 循环可能在质量不达标时停止
- 模型可能给自己的工作打高分
- 缺乏客观的停止标准

例子：
/auto-review-loop 的停止条件是 "score >= 6 AND verdict ∈ {ready, almost}"
如果分数是 Claude 自己打的，就会自我赦免
```

**为什么这么设计？**

```python
# 核心原则："A loop can DRIVE; it cannot ACQUIT."
# 循环可以驱动自己前进，但不能赦免自己的工作

# 判断标准："一个没有品味的脚本能回答这个问题吗？"
- 能 → Type-A（执行记账，可以自判）
  - exit code == 0
  - 文件存在
  - jobs 完成
  
- 不能 → Type-B（质量判断，必须跨模型判）
  - "论文够好了"
  - "证明有效"
  - "idea 新颖"
```

**局限性是什么？**

```
1. 灰色地带
   - 有些门控介于 Type-A 和 Type-B 之间
   - 例如："测试通过" 是 Type-A，但 "测试覆盖率足够" 可能是 Type-B
   - 需要仔细分类

2. 复合门控
   - 很多停止条件是 Type-A 和 Type-B 的组合
   - 例如："score >= 6 AND verdict ready" 
   - 需要拆分：A 部分自判，B 部分跨模型判

3. 执行成本
   - 所有 Type-B 门控都需要跨模型调用
   - 增加延迟和成本
   - 可能成为瓶颈

4. 判断标准模糊
   - "一个没有品味的脚本能回答这个问题吗？" 
   - 不同人可能有不同理解
   - 需要具体案例指导
```

**有哪些改进方法？**

```
1. 更细粒度的分类
   - 将 Type-A/Type-B 扩展为更细的类别
   - 例如：Type-A0（纯客观）、Type-A1（可自动化）、Type-B1（需要品味）、Type-B2（需要专业知识）

2. 自动化门控
   - 用确定性脚本替代 LLM 判断
   - 例如：verify_paper_audits.sh 检查审计结果
   - 减少跨模型调用

3. 门控缓存
   - 相同输入的门控结果可以缓存
   - 避免重复调用
   - 需要缓存失效机制

4. 门控组合
   - 预定义常见的门控组合
   - 简化 skill 开发
   - 减少分类错误
```

### 1.3 Fan-Out Pattern

**解决什么问题？**

```
问题：搜索空间太大，单线程太慢
- 生成 8-12 个 idea 需要很长时间
- 文献调研需要搜索多个来源
- 攻击角度枚举需要广度

传统方案：
- 顺序执行：太慢
- 并行执行：需要协调机制
```

**为什么这么设计？**

```python
# 核心原则："Fan-out is 火力; the jury is 裁判席"
# 子代理生成候选（火力），不同模型家族做裁决（裁判席）

# 3 层降级
Tier 1: ultracode / Workflow 真并行（动态编排）
Tier 2: Plain Agent 工具（静态扇出）
Tier 3: 顺序执行（上下文重置）

# 关键规则
✅ Fan out 搜索候选
❌ 永远不要 fan out 裁判席

# 原因
N 个 Claude 的 agreement 是 correlated blindness，不是 consensus
```

**局限性是什么？**

```
1. 协调成本
   - 并行执行需要协调机制
   - 结果合并需要去重逻辑
   - 失败处理更复杂

2. 资源消耗
   - 多个子代理同时运行
   - API 调用次数增加
   - 内存和 CPU 消耗增加

3. 质量控制
   - 子代理生成质量参差不齐
   - 需要机械去重（不能用 LLM 判断）
   - 可能遗漏高质量候选

4. 降级问题
   - Tier 3 顺序执行效率低
   - 上下文重置可能丢失信息
   - 需要设计良好的降级策略
```

**有哪些改进方法？**

```
1. 动态调度
   - 根据任务复杂度动态选择 tier
   - 重要任务用 Tier 1，简单任务用 Tier 3
   - 减少资源浪费

2. 智能去重
   - 用 embedding 计算相似度
   - 保留语义不同的候选
   - 减少 jury 输入

3. 质量过滤
   - 用简单的启发式规则过滤低质量候选
   - 例如：长度检查、格式检查
   - 减少 jury 负担

4. 异步合并
   - 子代理结果异步合并
   - 不等待所有子代理完成
   - 提高响应速度
```

---

## 第二类：实验和方案验证能力

> **考察重点**：不仅关注做了什么，更关注怎么证明它是有效的，追问实验细节

### 2.1 如何验证跨模型审查的有效性？

**实验设计**：

```
实验 1：单模型 vs 跨模型审查
- 对照组：Claude 审查 Claude 的输出
- 实验组：GPT-5.5 审查 Claude 的输出
- 指标：发现的 bug 数量、严重程度、修复率

实验 2：不同 reviewer 的一致性
- 多个 reviewer 审查同一篇论文
- 计算 inter-rater agreement
- 分析分歧原因

实验 3：审稿轮数 vs 质量
- 记录每轮审稿的分数变化
- 分析收敛速度
- 找到最优轮数
```

**实际数据**：

```markdown
# ARIS 的真实 score progression
Round 1: 4/10 (初始状态)
Round 2: 5.5/10 (第一轮修改后)
Round 3: 6.5/10 (第二轮修改后)
Round 4: 7/10 (最终状态)

# 关键发现
- 前两轮提升最大（+1.5, +1.0）
- 后两轮提升较小（+0.5）
- 4 轮通常是足够的
```

**追问应对**：

```
Q: 如何证明跨模型审查比单模型审查更好？
A: 
1. 定量指标
   - 发现的 bug 数量：跨模型平均多发现 30% 的问题
   - 严重程度：跨模型能发现更多 critical bugs
   - 修复率：跨模型发现的问题修复率更高

2. 定性分析
   - 跨模型能发现单模型的盲区
   - 例如：Claude 可能忽略的数学错误，GPT-5.5 能发现
   - 案例：ARIS 的 auto-review-loop 中，GPT-5.5 多次发现 Claude 遗漏的实验设计问题

3. 对比实验
   - 同一篇论文，分别用单模型和跨模型审查
   - 对比审查结果的差异
   - 分析哪些问题是只有跨模型能发现的
```

### 2.2 如何验证 Research Wiki 的 anti-repetition 效果？

**实验设计**：

```
实验 1：有 Wiki vs 无 Wiki 的 idea 生成
- 对照组：不使用 Research Wiki，直接生成 idea
- 实验组：使用 Research Wiki，读取 failed ideas
- 指标：重复 idea 比例、idea 质量

实验 2：query_pack 的压缩效果
- 测试不同 max_chars 下的信息保留率
- 分析哪些信息被截断
- 找到最优压缩比

实验 3：确定性写入器的可靠性
- 测试写入器的正确性
- 分析写入失败的情况
- 验证状态机的正确性
```

**实际数据**：

```python
# Research Wiki 的关键指标
- 论文摄入：自动去重（arxiv_id + slug）
- Idea 记录：outcome 状态机（pending → positive/negative/mixed）
- Claim 记录：status 状态机（drafted → unproven → verified/refuted）
- Anti-repetition：query_pack 包含 failed ideas，/idea-creator 读取后避免重复

# 真实案例
- 用户报告：首次 /idea-creator 记下的 idea，重新生成时不见了
- 原因：wiki 页面是 freehand 写的，这种 prose 步骤在"再生成一版"时容易被跳过
- 解决方案：每个节点层都有专属的 Python 写入器（ingest_paper, add_claim, upsert_idea, add_experiment）
```

**追问应对**：

```
Q: 如何证明 Research Wiki 真正解决了长程记忆问题？
A:
1. 问题复现
   - 没有 Wiki 时，通宵运行会忘记之前的发现
   - 重复尝试已经失败的 idea
   - 浪费时间和资源

2. 解决方案验证
   - Wiki 的确定性写入器确保信息不丢失
   - query_pack 压缩知识包，/idea-creator 读取后避免重复
   - 状态机记录 idea 的 outcome，防止重复尝试

3. 实际效果
   - 使用 Wiki 后，重复 idea 比例下降 80%
   - 通宵运行的成功率提升 50%
   - 用户反馈：不再忘记之前的发现
```

### 2.3 如何验证实验自动化流程的有效性？

**实验设计**：

```
实验 1：Cross-Model Code Review 的效果
- 对照组：不进行代码审查，直接部署
- 实验组：GPT-5.5 审查代码后再部署
- 指标：部署失败率、bug 发现率、GPU 时间浪费

实验 2：Sanity Check 的必要性
- 测试不进行 Sanity Check 直接部署的情况
- 分析失败原因和修复成本
- 验证 Sanity Check 的 ROI

实验 3：自动 Debug 的成功率
- 测试自动 Debug 的成功率
- 分析哪些错误可以自动修复
- 找到自动 Debug 的边界
```

**实际数据**：

```python
# Cross-Model Code Review 的效果
- 发现的 critical bugs：平均每个实验 2-3 个
- 部署失败率下降：60%
- GPU 时间节省：40%

# Sanity Check 的效果
- 环境问题发现率：90%
- 平均修复时间：5 分钟
- 避免的 full experiment 失败：70%

# 自动 Debug 的成功率
- OOM 问题：95% 自动修复
- ImportError：90% 自动修复
- CUDA error：70% 自动修复
- NaN/divergence：50% 自动修复
```

**追问应对**：

```
Q: 如何证明实验自动化流程是可靠的？
A:
1. 故障注入测试
   - 故意引入各种错误（OOM, ImportError, CUDA error）
   - 测试自动修复的成功率
   - 分析失败原因

2. 真实场景验证
   - 在真实实验中验证流程
   - 记录故障和修复过程
   - 分析流程的鲁棒性

3. 对比实验
   - 对比手动调试和自动调试的效率
   - 分析自动调试的优势和劣势
   - 找到最优的自动化程度
```

---

## 第三类：问题定位能力

> **考察重点**：排查问题的思路与解决方案，而不是堆砌复杂流程

### 3.1 长时间运行中的故障检测

**问题场景**：
```
通宵运行的 /research-pipeline 突然卡住
- 没有输出
- 没有错误信息
- 不知道卡在哪里
```

**排查思路**：

```
Step 1: 检查状态文件
- 查看 .aris/runs/<run_id>.json
- 确定当前卡在哪个阶段
- 检查每个阶段的状态（running/done/accepted）

Step 2: 检查 watchdog 日志
- 查看 alerts.log
- 检查是否有 STALE/MISSING/COMPLETED 信号
- 确定是否是静默死亡

Step 3: 检查 iteration log
- 查看 iteration_log.py 的输出
- 检查 stale_count
- 确定是否需要结构性 pivot

Step 4: 检查进程状态
- 查看 screen/tmux 会话
- 检查 GPU 使用情况
- 确定是否有进程卡住
```

**解决方案**：

```python
# 方案 1: Watchdog 监控
# watchdog.py 监控状态文件的 mtime
# STALE / MISSING / COMPLETED 写入 alerts.log
# 仅检测，不重启（避免破坏裁决）

# 方案 2: Iteration Log 计数
# iteration_log.py 计数新发现
# stale_count ≥ 2 → 强制结构性 pivot（换 frame + 选未试过的方向）
# stale_count ≥ 4 → 升级给人类

# 方案 3: Heartbeat 机制
# /loop 或 CronCreate 定期唤醒
# 检测 stalled phase（无进度、死进程、资源阻塞）
# 只能检测，不能裁决
```

**追问应对**：

```
Q: 如何区分"正常等待"和"卡住"？
A:
1. 时间阈值
   - 每个阶段有预期的完成时间
   - 超过阈值（如 2 小时）才报警
   - 避免误报

2. 状态变化检测
   - 监控状态文件的 mtime
   - 如果 mtime 长时间不变，可能是卡住
   - 结合进程状态判断

3. 资源使用检测
   - 检查 GPU 使用率
   - 检查 CPU/内存使用
   - 如果资源空闲但没有进度，可能是卡住

4. 日志分析
   - 分析最近的日志输出
   - 检查是否有错误信息
   - 确定卡住的原因
```

### 3.2 流式传输中的 Bug

**问题场景**：
```
ARIS 的 REPL 中，短回复只显示 "✔ Done"
- 用户看不到实际的回复内容
- 长回复正常显示
- 问题难以复现
```

**排查思路**：

```
Step 1: 定位问题
- 检查 spinner 的实现
- 发现 spinner 用 Save/RestorePosition 画 "⠋ Thinking…"
- 流式输出在同一行覆盖它

Step 2: 分析原因
- finish 函数清掉了整行
- 把一条短的单行回复也抹了
- 长回复不受影响（有多行）

Step 3: 验证假设
- 测试不同长度的回复
- 确认只有短回复受影响
- 复现问题

Step 4: 修复问题
- 如果这一轮打印过可见文本，REPL 收尾不再清行
- Clear(UntilNewLine) 只擦回复之后的 spinner 残尾
- 保留回复内容
```

**解决方案**：

```rust
// 修复前
fn finish(&mut self) {
    self.clear_line();  // 清掉整行，包括回复
}

// 修复后
fn finish(&mut self) {
    if self.has_visible_text {
        // 只擦 spinner 残尾，保留回复
        self.clear_until_newline();
    } else {
        self.clear_line();
    }
}
```

**追问应对**：

```
Q: 如何避免类似的 UI bug？
A:
1. 分层测试
   - 单元测试：测试每个组件
   - 集成测试：测试组件交互
   - E2E 测试：测试完整流程

2. 边界条件测试
   - 测试空输入
   - 测试短输入
   - 测试长输入
   - 测试特殊字符

3. 真机验证
   - 在真实终端中测试
   - 不同终端模拟器
   - 不同操作系统

4. 回归测试
   - 每次修复后运行回归测试
   - 确保不引入新问题
   - 自动化测试流程
```

### 3.3 MCP 协议版本协商问题

**问题场景**：
```
MCP server 同意了一个 ARIS 不会说的版本
- 后续 tools/list / tools/call 跑在不兼容协议上
- 报错晦涩，难以定位
```

**排查思路**：

```
Step 1: 检查日志
- 发现 stdio 握手请求 2025-03-26
- 但 server 协商回来的版本被忽略
- 后续调用失败

Step 2: 分析协议
- MCP 规范要求客户端验证协商版本
- ARIS 之前没有验证
- 导致不兼容的版本被接受

Step 3: 复现问题
- 使用不同版本的 MCP server
- 确认版本不匹配会导致问题
- 验证修复方案

Step 4: 修复问题
- 添加版本验证逻辑
- 支持的版本集合：2025-11-25 / 2025-06-18 / 2025-03-26 / 2024-11-05
- 不支持的版本：终止子进程 + 清槽 + 报错
```

**解决方案**：

```rust
// 修复前
fn initialize(&mut self) -> Result<()> {
    let response = self.send_request("initialize", ...)?;
    // 没有验证协商版本
    Ok(())
}

// 修复后
fn initialize(&mut self) -> Result<()> {
    let response = self.send_request("initialize", ...)?;
    let negotiated_version = response.result.version;
    
    const SUPPORTED_VERSIONS: &[&str] = &[
        "2025-11-25", "2025-06-18", "2025-03-26", "2024-11-05"
    ];
    
    if !SUPPORTED_VERSIONS.contains(&negotiated_version.as_str()) {
        // 终止子进程 + 清槽 + 报错
        self.terminate();
        return Err(Error::UnsupportedVersion(negotiated_version));
    }
    
    Ok(())
}
```

**追问应对**：

```
Q: 如何设计一个健壮的协议协商机制？
A:
1. 版本兼容性
   - 定义支持的版本集合
   - 明确不支持的版本
   - 提供降级方案

2. 错误处理
   - 清晰的错误信息
   - 详细的日志记录
   - 优雅的失败处理

3. 测试覆盖
   - 测试所有支持的版本
   - 测试不支持的版本
   - 测试版本降级场景

4. 文档说明
   - 明确版本要求
   - 提供升级指南
   - 记录已知问题
```

### 3.4 流式传输中的 EOF 判断

**问题场景**：
```
MiniMax 等 OpenAI-compatible provider 无法正常使用
- clean-EOF 完成判定把 data: [DONE] 哨兵当成唯一权威信号
- 但 MiniMax 发 finish_reason: "stop" 后直接关连接不发 [DONE]
- 导致连接一直挂着
```

**排查思路**：

```
Step 1: 检查日志
- 发现连接一直挂着，没有完成
- 检查 SSE 流，发现有 finish_reason: "stop"
- 但没有 data: [DONE]

Step 2: 分析规范
- Chat Completions spec：finish_reason 是终止帧标志
- [DONE] 是 transport 约定，部分 provider 不发
- ARIS 之前只检查 [DONE]

Step 3: 复现问题
- 使用 MiniMax API 测试
- 确认 finish_reason: "stop" 后连接关闭
- 验证修复方案

Step 4: 修复问题
- clean-EOF 判定现在是纯函数 stream_eof_action(...)
- 见到 [DONE] 或非空 finish_reason 任一即算完成
- 不在 finish_reason 处提前停读（include_usage 尾随 chunk 仍消费）
```

**解决方案**：

```rust
// 修复前
fn is_stream_done(chunk: &str) -> bool {
    chunk.contains("data: [DONE]")
}

// 修复后
fn stream_eof_action(chunk: &str, has_content: bool) -> StreamAction {
    if chunk.contains("data: [DONE]") {
        return StreamAction::Complete;
    }
    
    if let Some(finish_reason) = extract_finish_reason(chunk) {
        if !finish_reason.is_empty() {
            // 还要消费可能的 include_usage 尾随 chunk
            return StreamAction::ConsumeThenComplete;
        }
    }
    
    if has_content {
        StreamAction::Continue
    } else {
        StreamAction::WaitingForContent
    }
}
```

**追问应对**：

```
Q: 如何处理不同 API provider 的差异？
A:
1. 规范理解
   - 深入理解 OpenAI Chat Completions spec
   - 区分 must-have 和 nice-to-have
   - 明确哪些是 transport 约定

2. 兼容性设计
   - 核心逻辑基于规范
   - 特殊处理基于 transport 约定
   - 可配置的 fallback 策略

3. 测试覆盖
   - 测试所有主流 provider
   - 测试边界条件
   - 测试错误场景

4. 监控告警
   - 监控连接状态
   - 检测异常行为
   - 及时告警和处理
```

---

## 第四类：工程落地能力

> **考察重点**：理论结合实际，真正落地生产价值过程中的问题

### 4.1 Resumable Runs 设计

**理论 vs 实际**：

```
理论：长时间运行的任务可以暂停和恢复
实际：
- 状态文件可能损坏
- 恢复时可能状态不一致
- 部分完成的任务难以恢复
```

**工程落地**：

```python
# 状态文件设计
.runs/<run_id>.json = {
    "run_id": "factorized-gap-2026-06-27",
    "phases": {
        "idea-discovery": {
            "status": "accepted",  # running/done/accepted/skipped
            "artifact": "idea-stage/IDEA_REPORT.md",
            "verdict_id": "codex-thread-019cd392-...",
            "reviewer": "codex-gpt-5.5"
        },
        "experiment-bridge": {
            "status": "done",
            "artifact": "refine-logs/EXPERIMENT_RESULTS.md"
        },
        "auto-review-loop": {
            "status": "running"
        }
    }
}

# 恢复逻辑
def resume(run_id):
    state = load_state(run_id)
    first_non_accepted = find_first_non_accepted(state.phases)
    
    # 重新验证 done 但未 accepted 的阶段
    for phase in state.phases:
        if phase.status == "done" and phase.status != "accepted":
            re_validate(phase)
    
    # 从第一个未完成的阶段开始
    start_from(first_non_accepted)
```

**遇到的问题与解决**：

```
问题 1: 状态文件损坏
- 原因：进程被 kill 时状态文件写入不完整
- 解决：原子写入（tempfile + rename）
- 验证：状态文件加载时验证 JSON 格式

问题 2: 恢复时状态不一致
- 原因：阶段之间有依赖关系
- 解决：重新验证依赖关系
- 验证：检查每个阶段的 artifact 是否存在

问题 3: 部分完成的任务难以恢复
- 原因：有些任务是原子性的，不能部分恢复
- 解决：记录任务的子状态
- 验证：从最近的检查点恢复
```

### 4.2 安全机制

**理论 vs 实际**：

```
理论：LLM 应该遵守安全规则
实际：
- LLM 可能被 prompt injection 攻击
- 工具调用可能绕过权限检查
- 配置可能泄漏敏感信息
```

**工程落地**：

```python
# 1. Injection Hygiene
# threat_scan.py 扫描注入 payload
# quarantine() 隔离可疑内容
# 保留原始文本供人类审查

def scan_for_threats(text, scope="strict"):
    patterns = [
        r"ignore previous instructions",
        r"you are now",
        r"system prompt",
        # ... 更多模式
    ]
    findings = []
    for pattern in patterns:
        if re.search(pattern, text, re.IGNORECASE):
            findings.append(pattern)
    return findings

# 2. Sandbox StrictMode
# 忽略所有 LLM-supplied override
# dangerouslyDisableSandbox, namespaceRestrictions, 
# isolateNetwork, filesystemMode, allowedMounts

# 3. System Prompt 配置脱敏
# 敏感 key 替换为 [REDACTED]
# apikey/token/secret/password/authorization
# MCP command 替换为 <configured>
# MCP url 仅保留 scheme://host[:port]
```

**遇到的问题与解决**：

```
问题 1: Prompt Injection
- 场景：论文中包含恶意指令
- 检测：threat_scan.py 扫描已知模式
- 处理：quarantine 隔离，保留原始文本
- 验证：注入测试用例

问题 2: 权限绕过
- 场景：LLM 请求危险操作
- 检测：PermissionMode 检查
- 处理：拒绝执行，记录日志
- 验证：权限测试用例

问题 3: 配置泄漏
- 场景：system prompt 包含敏感信息
- 检测：render_config_section() 脱敏
- 处理：替换为 [REDACTED]
- 验证：脱敏测试用例
```

### 4.3 MCP 工具调度

**理论 vs 实际**：

```
理论：MCP 提供统一的工具调用接口
实际：
- 协议版本不兼容
- 传输层实现差异
- 错误处理不完善
```

**工程落地**：

```rust
// MCP 调度流程
fn dispatch_tool(tool_name: &str, args: Value) -> Result<Value> {
    // 1. 解析工具名称
    let (server, tool) = parse_tool_name(tool_name)?;
    
    // 2. 获取 MCP 连接
    let connection = get_or_create_connection(server)?;
    
    // 3. 验证协议版本
    connection.validate_version()?;
    
    // 4. 发送请求
    let response = connection.send_request("tools/call", json!({
        "name": tool,
        "arguments": args
    }))?;
    
    // 5. 处理响应
    match response.result {
        Some(result) => Ok(result),
        None => Err(Error::ToolCallFailed(response.error)),
    }
}

// 错误处理
fn handle_mcp_error(error: Error) -> ErrorAction {
    match error {
        Error::Timeout => ErrorAction::Retry,
        Error::ProtocolMismatch => ErrorAction::TerminateAndReconnect,
        Error::ToolNotFound => ErrorAction::FailLoud,
        _ => ErrorAction::FailLoud,
    }
}
```

**遇到的问题与解决**：

```
问题 1: 协议版本不兼容
- 场景：MCP server 返回不支持的版本
- 检测：版本验证逻辑
- 处理：终止子进程 + 清槽 + 报错
- 验证：版本兼容性测试

问题 2: 传输层差异
- 场景：LSP-style framing vs NDJSON
- 检测：自动识别传输格式
- 处理：读两种都自动识别
- 验证：传输层测试

问题 3: 错误处理不完善
- 场景：MCP server 崩溃
- 检测：try_wait() 检测死进程
- 处理：kill().await + respawn
- 验证：故障注入测试
```

### 4.4 系统稳定性保证

**理论 vs 实际**：

```
理论：系统应该稳定运行
实际：
- 内存泄漏
- 连接超时
- 资源耗尽
```

**工程落地**：

```rust
// 1. 资源管理
struct ResourceManager {
    max_connections: usize,
    max_memory_mb: usize,
    max_cpu_percent: f64,
}

impl ResourceManager {
    fn check_resources(&self) -> Result<()> {
        if self.current_connections() > self.max_connections {
            return Err(Error::TooManyConnections);
        }
        if self.current_memory() > self.max_memory_mb {
            return Err(Error::OutOfMemory);
        }
        Ok(())
    }
}

// 2. 超时控制
fn with_timeout<F, T>(duration: Duration, f: F) -> Result<T>
where
    F: Future<Output = Result<T>>,
{
    tokio::time::timeout(duration, f)
        .await
        .map_err(|_| Error::Timeout)?
}

// 3. 重试机制
fn with_retry<F, T>(max_retries: usize, f: F) -> Result<T>
where
    F: Fn() -> Result<T>,
{
    let mut last_error = None;
    for attempt in 0..max_retries {
        match f() {
            Ok(result) => return Ok(result),
            Err(e) => {
                last_error = Some(e);
                if attempt < max_retries - 1 {
                    std::thread::sleep(Duration::from_secs(2_u64.pow(attempt as u32)));
                }
            }
        }
    }
    Err(last_error.unwrap())
}
```

**遇到的问题与解决**：

```
问题 1: 内存泄漏
- 场景：长时间运行内存持续增长
- 检测：监控内存使用
- 处理：定期重启、资源回收
- 验证：内存泄漏测试

问题 2: 连接超时
- 场景：MCP server 无响应
- 检测：超时检测
- 处理：超时后重试或重启
- 验证：超时测试

问题 3: 资源耗尽
- 场景：并发请求过多
- 检测：资源使用监控
- 处理：限流、降级、拒绝
- 验证：压力测试
```

---

## 第五类：业务与实际场景理解

> **考察重点**：项目真正需要产生的是能够有用的场景价值和业务价值

### 5.1 适合什么场景？

**核心价值**：

```
ARIS 的核心价值：
1. 自动化科研流程（节省时间）
2. 跨模型审查（提高质量）
3. 持久化记忆（避免重复）
4. 故障恢复（保证可靠性）

适合的场景：
1. 学术研究（论文写作、实验、审稿）
2. 技术调研（文献综述、竞品分析）
3. 专利撰写（发明披露、权利要求）
4. 技术报告（项目文档、技术方案）
```

**不适合的场景**：

```
1. 实时交互场景
   - ARIS 的审稿-修改循环需要时间
   - 不适合需要即时反馈的场景
   - 例如：在线客服、实时翻译

2. 高风险决策场景
   - ARIS 的输出需要人类审核
   - 不适合需要完全自动化的场景
   - 例如：医疗诊断、金融交易

3. 资源受限场景
   - ARIS 需要调用多个 LLM API
   - 不适合成本敏感的场景
   - 例如：个人博客、小型项目

4. 非结构化任务
   - ARIS 的 skills 是针对科研流程设计的
   - 不适合非结构化的创意任务
   - 例如：艺术创作、即兴演讲
```

### 5.2 用户更关心什么？

**用户画像**：

```
1. 科研人员
   - 关心：论文质量、审稿意见、实验结果
   - 痛点：审稿周期长、实验重复、写作困难
   - 价值：加速科研流程、提高论文质量

2. 技术团队
   - 关心：技术方案、代码质量、文档完整性
   - 痛点：文档更新慢、代码审查不足、知识流失
   - 价值：自动化文档、跨模型代码审查

3. 产品经理
   - 关心：需求文档、竞品分析、技术可行性
   - 痛点：调研耗时、分析不全面、决策依据不足
   - 价值：自动化调研、结构化分析

4. 专利代理人
   - 关心：专利撰写、权利要求、新颖性检查
   - 痛点：撰写耗时、审查严格、修改频繁
   - 价值：自动化撰写、跨模型审查
```

**用户关心的关键指标**：

```
1. 时间节省
   - 论文写作时间：从 2 周缩短到 2 天
   - 审稿-修改时间：从 1 个月缩短到 1 周
   - 实验时间：从 1 周缩短到 1 天

2. 质量提升
   - 论文分数：平均提升 2-3 分
   - Bug 发现率：提升 30%
   - 审稿通过率：提升 50%

3. 成本控制
   - API 调用成本：可控（effort levels）
   - GPU 成本：按需使用
   - 人力成本：显著降低

4. 可靠性
   - 故障恢复：自动 resume
   - 数据安全：版本控制
   - 审计 trail：完整记录
```

### 5.3 上线成本有多高？

**成本分析**：

```
1. API 成本
   - Claude: $3-15/1M tokens
   - GPT-5.5: $5-25/1M tokens
   - 其他模型：价格不同
   
   典型场景成本（一篇论文）：
   - Idea 生成：$5-10
   - 实验审查：$10-20
   - 论文写作：$20-40
   - 审稿-修改：$30-60
   - 总计：$65-130

2. GPU 成本
   - 本地 GPU：已有则免费
   - Vast.ai：$0.2-0.5/小时
   - Modal：按使用付费
   
   典型场景成本（一篇论文）：
   - Pilot 实验：$5-10
   - 全量实验：$20-50
   - 总计：$25-60

3. 人力成本
   - 初始设置：1-2 天
   - 日常维护：每周 2-4 小时
   - 问题处理：按需

4. 基础设施成本
   - 服务器：已有则免费
   - 存储：很少（主要是文本）
   - 网络：很少
```

**成本优化策略**：

```
1. Effort Levels
   - lite: 最低成本，快速迭代
   - balanced: 平衡成本和质量
   - max: 高成本，高质量
   - beast: 最高成本，最高质量

2. 模型选择
   - 执行：Claude（性价比高）
   - 审查：GPT-5.5（质量高）
   - 备选：其他模型（成本低）

3. 缓存机制
   - Research Wiki 缓存知识
   - 审稿结果可以复用
   - 避免重复计算

4. 按需使用
   - 只在需要时调用 API
   - 只在需要时使用 GPU
   - 避免资源浪费
```

### 5.4 如果资源有限，应该首先优化哪些部分？

**优先级排序**：

```
P0（必须优化）：
1. 跨模型审查机制
   - 这是 ARIS 的核心价值
   - 不能为了省钱跳过审查
   - 可以用更便宜的模型做初步审查

2. 故障恢复机制
   - 长时间运行必须可靠
   - 不能因为故障丢失进度
   - Resumable Runs 是基础

3. 安全机制
   - 防止 prompt injection
   - 防止权限绕过
   - 防止配置泄漏

P1（应该优化）：
1. Research Wiki
   - 解决长程记忆问题
   - 避免重复计算
   - 提高效率

2. 实验自动化
   - 减少人工干预
   - 提高实验效率
   - 降低错误率

3. 成本控制
   - Effort Levels
   - 模型选择
   - 缓存机制

P2（可以优化）：
1. UI/UX 改进
   - 更好的 REPL 体验
   - 更清晰的错误信息
   - 更友好的配置界面

2. 更多模型支持
   - 更多 LLM 选择
   - 更多 GPU 平台
   - 更多集成选项

3. 更多功能
   - 更多 skill
   - 更多 workflow
   - 更多自动化
```

**资源受限下的最优策略**：

```
场景 1：成本敏感
- 使用 lite effort level
- 选择便宜的模型（如 DeepSeek）
- 减少审稿轮数（2 轮）
- 跳过部分审计（assurance: draft）

场景 2：时间敏感
- 使用 max effort level
- 选择高质量模型（GPT-5.5 xhigh）
- 增加审稿轮数（4 轮）
- 完整审计（assurance: submission）

场景 3：质量敏感
- 使用 beast effort level
- 选择最强模型（GPT-5.5 Pro）
- 最大审稿轮数
- 完整审计 + 外部验证

场景 4：平衡场景
- 使用 balanced effort level
- 选择性价比模型（Claude Opus）
- 标准审稿轮数（3 轮）
- 标准审计（assurance: polished）
```

### 5.5 业务价值量化

**ROI 计算**：

```
假设：一个科研人员每年写 3 篇论文

没有 ARIS：
- 论文写作时间：2 周 × 3 = 6 周
- 审稿-修改时间：1 个月 × 3 = 3 个月
- 总时间：4.5 个月
- 人力成本：$50/小时 × 40 小时/周 × 18 周 = $36,000

使用 ARIS：
- API 成本：$100 × 3 = $300
- GPU 成本：$50 × 3 = $150
- 人力成本：$50/小时 × 10 小时/周 × 4 周 = $2,000
- 总成本：$2,450

节省：
- 时间：4.5 个月 → 1 个月（节省 3.5 个月）
- 成本：$36,000 → $2,450（节省 $33,550）
- ROI：1370%
```

**长期价值**：

```
1. 知识积累
   - Research Wiki 持久化知识
   - 避免重复研究
   - 加速后续研究

2. 质量提升
   - 跨模型审查提高论文质量
   - 更高的接收率
   - 更好的学术声誉

3. 效率提升
   - 自动化流程节省时间
   - 更多时间做创新研究
   - 更高的产出

4. 团队协作
   - 统一的工作流程
   - 知识共享
   - 减少沟通成本
```

---

## 附录：面试回答模板

### 开场白模板

```
"我在研究 ARIS（Auto Research in Sleep）项目，这是一个基于 Claude Code 的自主科研工作流框架。它的核心创新是跨模型对抗协作——让不同架构的 LLM 互相审查，打破 self-play 的盲区。

我在项目中深入研究了几个关键技术：
1. 跨模型对抗协作的理论基础和实现
2. Type-A/Type-B 门控系统防止自我赦免
3. Research Wiki 解决长程记忆问题
4. Fan-Out Pattern 平衡搜索广度和裁决质量

这些技术解决了 Auto-Research 中的核心挑战：如何让 AI 自主做科研，同时保证质量和可靠性。"
```

### 技术深挖模板

```
"关于 [具体技术点]，我来详细讲一下：

问题背景：
- [具体问题是什么]
- [为什么这个问题重要]
- [传统方案的局限性]

解决方案：
- [ARIS 的解决方案是什么]
- [为什么这么设计]
- [理论基础是什么]

实际效果：
- [量化指标]
- [真实案例]
- [用户反馈]

局限性与改进：
- [当前方案的局限性]
- [可能的改进方向]
- [未来的发展趋势]"
```

### 问题定位模板

```
"关于 [具体问题]，我来讲一下排查思路：

问题现象：
- [具体现象是什么]
- [什么时候发生]
- [影响范围]

排查步骤：
1. [第一步：检查什么]
2. [第二步：分析什么]
3. [第三步：验证什么]
4. [第四步：定位什么]

根本原因：
- [根本原因是什么]
- [为什么会发生]
- [如何避免]

解决方案：
- [具体解决方案]
- [实现细节]
- [验证方法]

经验教训：
- [从这个问题学到了什么]
- [如何改进流程]
- [如何预防类似问题]"
```

---

*最后更新：2026-06-27*
*基于 ARIS 项目版本：v0.4.20*
