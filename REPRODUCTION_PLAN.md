# ARIS 项目完整复现 Plan
> 基于 8卡4090 (24GB) 算力的全流程复现指南

---

## 目录
1. [算力评估与可行性分析](#1-算力评估与可行性分析)
2. [环境准备清单](#2-环境准备清单)
3. [分阶段复现计划](#3-分阶段复现计划)
4. [学习路线图](#4-学习路线图)
5. [风险与应对策略](#5-风险与应对策略)
6. [预期成果与验收标准](#6-预期成果与验收标准)

---

## 1. 算力评估与可行性分析

### 1.1 硬件资源清单

```
┌─────────────────────────────────────────────────────────┐
│                    8卡 RTX 4090                          │
├─────────────────────────────────────────────────────────┤
│ GPU:        8 × RTX 4090                                │
│ 显存:       24GB × 8 = 192GB 总计                       │
│ 适用场景:   中等规模 ML 实验、模型微调、推理             │
│ 不适用:     LLM Pre-training (需要 A100/H100)           │
└─────────────────────────────────────────────────────────┘
```

### 1.2 ARIS 项目的 GPU 需求分析

**关键认知**：ARIS 本身是一个 **workflow orchestration 框架**，不是模型训练框架。

```
ARIS 的组成：
├── Skills (Markdown 指令)          → 不需要 GPU
├── Tools (Python/Bash 脚本)        → 不需要 GPU
├── MCP Servers (Claude/GPT 调用)   → 不需要 GPU（API 调用）
└── 实验执行 (ML Training)          → 需要 GPU ✓
```

**GPU 使用场景**：

| 场景 | GPU 需求 | 4090 适用性 |
|------|----------|-------------|
| Idea 验证 (Pilot) | 1-2 卡，几小时 | ✅ 完全适用 |
| 小规模实验 | 2-4 卡，几天 | ✅ 完全适用 |
| 中等规模实验 | 4-8 卡，一周 | ✅ 适用 |
| 大规模实验 | 8+ 卡，多周 | ⚠️ 需要优化 |
| LLM Pre-training | 64+ 卡 A100 | ❌ 不适用 |
| LLM Fine-tuning | 1-4 卡 4090 | ✅ 适用 (QLoRA) |

### 1.3 可复现范围

**完全可以复现的部分**（约 80% 的项目价值）：

```
✅ Workflow 1: Idea Discovery
   - /research-lit: 文献调研（不需要 GPU）
   - /idea-creator: Idea 生成（不需要 GPU）
   - /novelty-check: 新颖性检查（不需要 GPU）
   - /research-review: 跨模型审查（不需要 GPU）

✅ Workflow 1.5: Experiment Bridge
   - 代码实现（不需要 GPU）
   - 跨模型代码审查（不需要 GPU）
   - Pilot 实验（1-2 卡 4090）
   - 全量实验（4-8 卡 4090，取决于任务）

✅ Workflow 2: Auto Review Loop
   - 跨模型审稿（不需要 GPU）
   - 代码修改（不需要 GPU）
   - 实验部署（4-8 卡 4090）

✅ Workflow 3: Paper Writing
   - 论文规划（不需要 GPU）
   - LaTeX 生成（不需要 GPU）
   - 编译（不需要 GPU）

✅ 其他功能
   - Research Wiki（不需要 GPU）
   - Meta-optimize（不需要 GPU）
   - Rebuttal（不需要 GPU）
```

**需要调整的部分**：

```
⚠️ 大规模实验
   - 原设计：A100 80GB × 8+
   - 调整方案：
     1. 使用 QLoRA 微调（4090 24GB 够用）
     2. 减小 batch size（梯度累积补偿）
     3. 使用 gradient checkpointing
     4. 分阶段训练（teacher → student）

⚠️ 长时间训练
   - 原设计：连续训练数天
   - 调整方案：
     1. 使用 checkpoint 恢复
     2. 分段训练
     3. 监控 GPU 温度和功耗
```

### 1.4 成本估算

```
API 成本（主要成本）：
├── Claude (Executor): ~$3-15/1M tokens
├── GPT-5.5 (Reviewer): ~$5-25/1M tokens
└── 一篇论文完整流程: ~$100-300

GPU 成本（本地 4090，电费）：
├── 4090 功耗: ~450W × 8 = 3.6KW
├── 24 小时运行: ~86.4 度电
├── 电费: ~0.6 元/度 = ~52 元/天
└── 一篇论文实验周期: ~7-14 天 = ~364-728 元

总成本（一篇论文）：
├── API: ~$100-300 (约 700-2100 元)
├── 电费: ~400-700 元
└── 总计: ~1100-2800 元
```

---

## 2. 环境准备清单

### 2.1 软件环境

```bash
# 1. 基础环境
操作系统: Ubuntu 22.04 LTS (推荐)
Python: 3.10+
CUDA: 12.1+
cuDNN: 8.9+

# 2. 包管理
conda: Miniconda3 或 Anaconda
pip: 最新版本

# 3. 必装工具
git: 2.30+
tmux/screen: 用于后台运行
ssh: 用于远程连接
rsync: 用于文件同步
latexmk: 用于论文编译（可选）
```

### 2.2 Python 环境

```yaml
# environment.yml
name: aris
channels:
  - pytorch
  - nvidia
  - conda-forge
dependencies:
  - python=3.10
  - pytorch=2.1.0
  - torchvision=0.16.0
  - torchaudio=2.1.0
  - pytorch-cuda=12.1
  - transformers=4.35.0
  - datasets=2.14.0
  - accelerate=0.24.0
  - peft=0.6.0  # for QLoRA
  - bitsandbytes=0.41.0  # for 4-bit quantization
  - wandb=0.16.0
  - scipy=1.11.0
  - numpy=1.24.0
  - pandas=2.0.0
  - matplotlib=3.7.0
  - seaborn=0.12.0
  - tqdm=4.65.0
  - pyyaml=6.0
  - requests=2.31.0
  - beautifulsoup4=4.12.0
```

### 2.3 API Keys 准备

```
必需：
├── Anthropic API Key (Claude)
│   - 用途：Executor（执行者）
│   - 获取：https://console.anthropic.com/
│   - 预算：~$50-100/月
│
├── OpenAI API Key (GPT-5.5)
│   - 用途：Reviewer（审查者）
│   - 获取：https://platform.openai.com/
│   - 预算：~$50-100/月
│   - 替代：ChatGPT Plus 订阅（通过 Codex MCP）
│
└── 可选：
    ├── Google API Key (Gemini)
    │   - 用途：文献调研
    │   - 获取：https://makersuite.google.com/
    │
    ├── Semantic Scholar API Key
    │   - 用途：论文搜索
    │   - 获取：https://www.semanticscholar.org/product/api
    │
    └── Exa API Key
        - 用途：Web 搜索
        - 获取：https://exa.ai/
```

### 2.4 Claude Code + Codex CLI 安装

```bash
# 1. 安装 Claude Code
# 参考：https://docs.anthropic.com/en/docs/claude-code
npm install -g @anthropic-ai/claude-code

# 2. 安装 Codex CLI
# 参考：https://developers.openai.com/codex
npm install -g @openai/codex

# 3. 认证 Codex
codex login

# 4. 注册 Codex 为 MCP Server
claude mcp add codex -s user -- codex mcp-server

# 5. 验证
claude mcp list | grep codex
# 应该显示：codex: codex mcp-server - ✓ Connected
```

### 2.5 项目初始化

```bash
# 1. 创建研究项目
mkdir ~/aris-research-project
cd ~/aris-research-project
git init
touch CLAUDE.md

# 2. 克隆 ARIS 仓库
git clone https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep.git ~/aris_repo

# 3. 安装 ARIS Skills
cd ~/aris-research-project
bash ~/aris_repo/tools/install_aris.sh

# 4. 初始化 Research Wiki
# 在 Claude Code 中运行：
/research-wiki init

# 5. 验证安装
# 在 Claude Code 中运行：
/alphaxiv https://arxiv.org/abs/1706.03762
```

### 2.6 GPU 服务器配置

```markdown
# 在 CLAUDE.md 中添加：

## Remote Server
- gpu: local
- This machine has direct GPU access (no SSH needed)
- GPU: 8x RTX 4090 (24GB)
- Conda env: `aris` (Python 3.10 + PyTorch 2.1)
- Activate: `eval "$(/path/to/miniconda3/bin/conda shell.bash hook)" && conda activate aris`
- Code directory: `/home/username/aris-research-project/`
- Use `tmux` for background jobs: `tmux new -d -s exp0 'bash -c "..."'`
- wandb: true
- wandb_project: aris-research
```

---

## 3. 分阶段复现计划

### Phase 0: 环境搭建与验证（1-2 天）

**目标**：确保所有工具和环境正常工作

```
Day 1: 基础环境
├── 上午：安装 Python 环境、CUDA、PyTorch
├── 下午：安装 Claude Code、Codex CLI
└── 晚上：API Keys 配置、验证连接

Day 2: ARIS 安装
├── 上午：克隆 ARIS、安装 Skills
├── 下午：初始化 Research Wiki
└── 晚上：运行简单测试（/alphaxiv）
```

**验收标准**：
- [ ] `claude mcp list` 显示 codex 已连接
- [ ] `/alphaxiv` 成功获取论文信息
- [ ] `research-wiki/` 目录结构正确
- [ ] `nvidia-smi` 显示 8 张 4090

### Phase 1: 单模块学习与验证（3-5 天）

**目标**：理解每个核心模块的工作原理

```
Day 3: Literature & Search 模块
├── 上午：学习 /research-lit 的实现
├── 下午：运行 /arxiv、/semantic-scholar
└── 晚上：理解多源聚合和去重逻辑

Day 4: Ideation 模块
├── 上午：学习 /idea-creator 的实现
├── 下午：运行 /idea-discovery（lite 模式）
└── 晚上：理解 Fan-Out Pattern

Day 5: Review 模块
├── 上午：学习 /auto-review-loop 的实现
├── 下午：理解跨模型对抗协作
└── 晚上：理解 Type-A/Type-B 门控

Day 6: Experiment 模块
├── 上午：学习 /experiment-bridge 的实现
├── 下午：理解实验部署流程
└── 晚上：理解 /experiment-queue

Day 7: Writing 模块
├── 上午：学习 /paper-writing 的实现
├── 下午：理解 LaTeX 生成流程
└── 晚上：理解审计门控
```

**验收标准**：
- [ ] 能解释每个模块的核心功能
- [ ] 能解释跨模型对抗协作的原理
- [ ] 能解释 Type-A/Type-B 门控系统
- [ ] 能解释 Fan-Out Pattern

### Phase 2: 小规模全流程验证（5-7 天）

**目标**：用一个简单课题跑通完整流程

**选题建议**（适合 4090 的小规模实验）：

```
选项 1: 文本分类改进
- 数据集：IMDB/SST-2
- 模型：BERT-base (110M)
- 实验：几小时/实验
- GPU：1-2 卡 4090

选项 2: 图像分类改进
- 数据集：CIFAR-10/ImageNet-subset
- 模型：ResNet-18/50
- 实验：几小时/实验
- GPU：1-2 卡 4090

选项 3: 扩散模型微调
- 数据集：MNIST/CIFAR-10
- 模型：DDPM (小规模)
- 实验：1-2 天/实验
- GPU：2-4 卡 4090

选项 4: NLP 任务微调
- 数据集：GLUE 子任务
- 模型：RoBERTa-base
- 实验：几小时/实验
- GPU：1-2 卡 4090
```

**执行流程**：

```
Day 8-9: Workflow 1 - Idea Discovery
├── 运行 /research-lit "改进方向"
├── 运行 /idea-creator
├── 运行 /novelty-check
└── 产出：IDEA_REPORT.md

Day 10-11: Workflow 1.5 - Experiment Bridge
├── 运行 /experiment-bridge
├── 代码实现
├── Pilot 实验（1-2 卡 4090）
└── 全量实验（4-8 卡 4090）

Day 12-13: Workflow 2 - Auto Review Loop
├── 运行 /auto-review-loop
├── 跨模型审稿（2-3 轮）
├── 代码修改
└── 重新实验

Day 14: Workflow 3 - Paper Writing
├── 运行 /paper-writing
├── LaTeX 生成
└── 编译 PDF
```

**验收标准**：
- [ ] 成功生成 IDEA_REPORT.md
- [ ] 成功运行实验并收集结果
- [ ] 成功完成 2-3 轮审稿-修改循环
- [ ] 成功生成论文 PDF

### Phase 3: 中等规模实验验证（7-10 天）

**目标**：验证 ARIS 在更复杂场景下的能力

**选题建议**：

```
选项 1: 模型压缩/蒸馏
- Teacher: BERT-large (340M)
- Student: BERT-base (110M)
- 实验：多组配置
- GPU：4-8 卡 4090

选项 2: 多任务学习
- 数据集：GLUE 全任务
- 模型：RoBERTa-large
- 实验：多任务组合
- GPU：4-8 卡 4090

选项 3: 扩散模型改进
- 数据集：CelebA/FFHQ-subset
- 模型：DDPM/LDM
- 实验：消融实验
- GPU：4-8 卡 4090

选项 4: RLHF/DPO 微调
- 数据集：Anthropic HH
- 模型：LLaMA-7B (QLoRA)
- 实验：多组超参数
- GPU：4-8 卡 4090
```

**执行流程**：

```
Day 15-17: 深度 Idea Discovery
├── 更广泛的文献调研
├── 更多 Idea 候选（8-12 个）
├── 更严格的 Novelty Check
└── 更详细的 Experiment Plan

Day 18-22: 大规模实验
├── Pilot 实验（验证可行性）
├── 主实验（多配置并行）
├── 消融实验（验证每个组件）
└── Baseline 对比

Day 23-26: 深度 Review Loop
├── 4 轮审稿-修改循环
├── Nightmare 模式（reviewer 直接读代码）
├── Debate Protocol（rebuttal）
└── 最终定稿

Day 27: 论文撰写
├── 完整论文生成
├── 图表制作
└── 编译和校对
```

**验收标准**：
- [ ] 成功运行 10+ 组实验
- [ ] 成功完成 4 轮审稿-修改循环
- [ ] 论文分数达到 6+/10
- [ ] 生成完整的论文 PDF

### Phase 4: 高级功能验证（5-7 天）

**目标**：验证 ARIS 的高级功能

```
Day 28-29: Research Wiki 深度使用
├── 学习 Wiki 的 4 层节点结构
├── 验证 Anti-Repetition 功能
├── 验证确定性写入器
└── 分析 Wiki 对 Idea 生成的影响

Day 30-31: Meta-Optimize
├── 运行 /meta-optimize
├── 分析 SKILL.md 改进建议
├── 应用改进
└── 验证改进效果

Day 32-34: Rebuttal 练习
├── 模拟审稿意见
├── 运行 /rebuttal
├── 分析 Rebuttal 质量
└── 学习如何回应审稿
```

**验收标准**：
- [ ] Wiki 成功记录了完整的研究过程
- [ ] Meta-optimize 生成了有价值的改进建议
- [ ] Rebuttal 质量达到可提交水平

### Phase 5: 总结与文档（3-5 天）

**目标**：总结学习成果，形成可复用的知识

```
Day 35-37: 技术总结
├── 整理关键技术点
├── 总结问题与解决方案
├── 编写技术文档
└── 准备面试材料

Day 38-39: 经验分享
├── 编写复现指南
├── 记录踩坑经验
├── 分享最佳实践
└── 反馈给 ARIS 社区
```

---

## 4. 学习路线图

### 4.1 第一周：基础理解

```
目标：理解 ARIS 的核心概念

学习内容：
├── 1. 项目结构和设计理念
│   ├── Skill-as-Markdown
│   ├── 跨模型对抗协作
│   └── Type-A/Type-B 门控
│
├── 2. 核心模块
│   ├── /research-lit
│   ├── /idea-creator
│   ├── /auto-review-loop
│   └── /experiment-bridge
│
└── 3. 工具链
    ├── Claude Code
    ├── Codex CLI
    └── MCP 协议

实践任务：
├── 安装和配置环境
├── 运行简单测试
└── 阅读核心 SKILL.md
```

### 4.2 第二周：流程验证

```
目标：跑通完整的研究流程

学习内容：
├── 1. Workflow 1: Idea Discovery
│   ├── 文献调研
│   ├── Idea 生成
│   └── Novelty Check
│
├── 2. Workflow 1.5: Experiment Bridge
│   ├── 代码实现
│   ├── 实验部署
│   └── 结果收集
│
├── 3. Workflow 2: Auto Review Loop
│   ├── 跨模型审稿
│   ├── 代码修改
│   └── 迭代改进
│
└── 4. Workflow 3: Paper Writing
    ├── 论文规划
    ├── LaTeX 生成
    └── 编译 PDF

实践任务：
├── 选择一个小课题
├── 跑通完整流程
└── 记录遇到的问题
```

### 4.3 第三周：深度理解

```
目标：理解 ARIS 的设计细节和权衡

学习内容：
├── 1. 跨模型对抗协作
│   ├── 理论基础（Adversarial Bandit）
│   ├── 实现细节（MCP 调用）
│   └── 效果验证
│
├── 2. Type-A/Type-B 门控
│   ├── 设计动机
│   ├── 分类标准
│   └── 实际应用
│
├── 3. Fan-Out Pattern
│   ├── 3 层降级
│   ├── 去重逻辑
│   └── 与 Jury 的关系
│
└── 4. Research Wiki
    ├── 4 层节点结构
    ├── 确定性写入器
    └── Anti-Repetition

实践任务：
├── 分析核心模块的源码
├── 设计对比实验
└── 验证设计决策
```

### 4.4 第四周：高级应用

```
目标：掌握 ARIS 的高级功能

学习内容：
├── 1. Nightmare 模式
│   ├── Reviewer 直接读代码
│   ├── 更严格的审查
│   └── Debate Protocol
│
├── 2. Meta-Optimize
│   ├── 日志分析
│   ├── SKILL.md 改进
│   └── 自我进化
│
├── 3. Rebuttal
│   ├── 审稿意见解析
│   ├── 策略制定
│   └── 反驳撰写
│
└── 4. 多平台适配
    ├── Cursor
    ├── Trae
    └── Copilot CLI

实践任务：
├── 运行高级功能
├── 分析效果
└── 总结最佳实践
```

---

## 5. 风险与应对策略

### 5.1 技术风险

```
风险 1: API 调用失败
├── 原因：网络问题、API 限流、Key 失效
├── 影响：审稿-修改循环中断
└── 应对：
    ├── 配置 API Key 备份
    ├── 实现重试机制
    └── 使用本地缓存

风险 2: GPU 内存不足
├── 原因：模型太大、batch size 太大
├── 影响：实验失败
└── 应对：
    ├── 使用 QLoRA (4-bit 量化)
    ├── 减小 batch size
    ├── 使用 gradient checkpointing
    └── 分阶段训练

风险 3: 实验时间过长
├── 原因：模型复杂、数据量大
├── 影响：复现周期延长
└── 应对：
    ├── 使用小规模数据集
    ├── 减少实验配置
    ├── 并行运行实验
    └── 使用 checkpoint 恢复

风险 4: 跨模型审查质量不稳定
├── 原因：模型版本变化、prompt 不一致
├── 影响：审稿质量波动
└── 应对：
    ├── 固定模型版本
    ├── 记录审稿历史
    └── 多审稿者投票
```

### 5.2 资源风险

```
风险 1: API 成本超预算
├── 原因：审稿轮数多、token 消耗大
├── 影响：成本超支
└── 应对：
    ├── 使用 lite effort level
    ├── 减少审稿轮数
    ├── 缓存审稿结果
    └── 监控 API 使用量

风险 2: GPU 故障
├── 原因：硬件问题、过热
├── 影响：实验中断
└── 应对：
    ├── 监控 GPU 温度
    ├── 定期检查硬件
    ├── 使用 checkpoint 恢复
    └── 备份实验结果

风险 3: 存储空间不足
├── 原因：实验结果、日志文件
├── 影响：无法保存结果
└── 应对：
    ├── 定期清理日志
    ├── 压缩实验结果
    ├── 使用外部存储
    └── 监控存储使用量
```

### 5.3 学习风险

```
风险 1: 概念理解困难
├── 原因：跨模型协作、Type-A/Type-B 门控等概念复杂
├── 影响：无法正确使用 ARIS
└── 应对：
    ├── 阅读技术报告
    ├── 分析源码
    ├── 参与社区讨论
    └── 实践验证

风险 2: 工具使用困难
├── 原因：Claude Code、Codex CLI 等工具不熟悉
├── 影响：效率低下
└── 应对：
    ├── 阅读官方文档
    ├── 运行示例
    ├── 参与社区讨论
    └── 积累经验

风险 3: 实验设计困难
├── 原因：不知道如何设计有效的实验
├── 影响：无法验证 ARIS 的效果
└── 应对：
    ├── 参考 ARIS 的示例
    ├── 阅读相关论文
    ├── 请教有经验的人
    └── 从简单开始
```

---

## 6. 预期成果与验收标准

### 6.1 学习成果

```
理论理解：
├── ✅ 理解跨模型对抗协作的原理和价值
├── ✅ 理解 Type-A/Type-B 门控系统
├── ✅ 理解 Fan-Out Pattern
├── ✅ 理解 Research Wiki 的设计
└── ✅ 理解 ARIS 的整体架构

实践能力：
├── ✅ 能够安装和配置 ARIS
├── ✅ 能够运行完整的研究流程
├── ✅ 能够分析和调试问题
├── ✅ 能够优化实验配置
└── ✅ 能够撰写技术文档

面试准备：
├── ✅ 能够清晰介绍 ARIS 项目
├── ✅ 能够回答技术深挖问题
├── ✅ 能够分析问题和解决方案
├── ✅ 能够讨论业务价值
└── ✅ 能够展示工程能力
```

### 6.2 复现成果

```
文档成果：
├── ✅ 复现指南（本文档）
├── ✅ 学习笔记
├── ✅ 问题与解决方案
├── ✅ 面试准备材料
└── ✅ 技术总结

代码成果：
├── ✅ 完整的实验代码
├── ✅ 实验结果和日志
├── ✅ 论文草稿
└── ✅ 改进建议

经验成果：
├── ✅ 问题定位能力
├── ✅ 实验设计能力
├── ✅ 工程落地能力
└── ✅ 业务理解能力
```

### 6.3 验收标准

```
Phase 0 验收（环境搭建）：
├── [ ] 所有软件安装成功
├── [ ] API Keys 配置正确
├── [ ] GPU 环境正常
├── [ ] ARIS Skills 安装成功
└── [ ] 简单测试通过

Phase 1 验收（单模块学习）：
├── [ ] 能解释每个模块的功能
├── [ ] 能解释核心设计决策
├── [ ] 能分析源码实现
└── [ ] 能回答技术问题

Phase 2 验收（小规模全流程）：
├── [ ] 成功生成 IDEA_REPORT.md
├── [ ] 成功运行实验
├── [ ] 成功完成审稿循环
├── [ ] 成功生成论文 PDF
└── [ ] 论文分数达到 5+/10

Phase 3 验收（中等规模实验）：
├── [ ] 成功运行 10+ 组实验
├── [ ] 成功完成 4 轮审稿循环
├── [ ] 论文分数达到 6+/10
├── [ ] 生成完整的论文
└── [ ] 消融实验验证每个组件

Phase 4 验收（高级功能）：
├── [ ] Wiki 成功记录研究过程
├── [ ] Meta-optimize 生成改进建议
├── [ ] Rebuttal 质量达标
└── [ ] 高级功能正常工作

Phase 5 验收（总结文档）：
├── [ ] 技术文档完整
├── [ ] 面试准备充分
├── [ ] 经验总结清晰
└── [ ] 可复现性强
```

---

## 附录：快速参考

### A. 常用命令

```bash
# 环境激活
eval "$(/path/to/miniconda3/bin/conda shell.bash hook)" && conda activate aris

# GPU 监控
nvidia-smi
watch -n 1 nvidia-smi

# 进程管理
tmux ls
tmux attach -t <session>
tmux kill-session -t <session>

# 文件同步
rsync -avz --include='*.py' --exclude='*' ./ user@server:/path/

# 实验监控
tail -f logs/experiment.log
watch -n 10 "cat results/latest.json"
```

### B. 故障排查

```bash
# API 连接测试
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model":"claude-3-opus-20240229","max_tokens":10,"messages":[{"role":"user","content":"test"}]}'

# GPU 状态检查
nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total --format=csv

# 磁盘空间检查
df -h
du -sh /path/to/experiments/*

# 网络连接测试
ping -c 3 api.anthropic.com
curl -I https://api.openai.com
```

### C. 参考资源

```
官方文档：
├── ARIS README: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep
├── Claude Code: https://docs.anthropic.com/en/docs/claude-code
├── Codex CLI: https://developers.openai.com/codex
└── MCP 协议: https://modelcontextprotocol.io/

技术报告：
├── ARIS 技术报告: https://arxiv.org/abs/2605.03042
└── Anti-Autoresearch: https://github.com/wanshuiyin/Anti-Autoresearch

社区资源：
├── GitHub Issues: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/issues
├── 讨论区: https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep/discussions
└── 微信群: 见 README 中的二维码
```

---

*最后更新：2026-06-27*
*基于 ARIS 项目版本：v0.4.20*
*硬件配置：8卡 RTX 4090 (24GB)*
