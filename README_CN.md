# autoresearch-creator

一个 Claude Code 插件，为你的项目生成专属的自治优化 skill。灵感来自 [karpathy/autoresearch](https://github.com/karpathy/autoresearch)。

> **[English](./README.md)**

---

## 这是什么？

Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch) 展示了一种强大的工程哲学：让自治 agent 通过迭代实验循环来优化 ML 模型训练——修改、评估、保留或回滚，周而复始。

**本项目将这一哲学泛化到任何领域。** CPU 性能、测试覆盖率、API 延迟、构建体积、游戏帧率、转化率——只要你能度量，就能 autoresearch。

`/create-autoresearch` 是一个工厂 skill，它分析你的代码库，然后为你的优化问题生成一个量身定制的 autoresearch skill。

## 核心思路

```
任何优化问题 = 状态(代码) × 度量(标量) × 变异(修改) × 评估(运行) × 选择(保留/回滚)
```

Karpathy 的 autoresearch 针对 ML 训练实现了这个模式。我们把它抽象成了 **5 个支柱 (Pillars)**，适用于任何可度量的领域：

| 支柱 | 定义 |
|------|------|
| **Metric (度量)** | 单一标量指标，方向明确（越高越好或越低越好） |
| **Surface (表面)** | agent 可以修改的文件范围，范围外的一切都被冻结 |
| **Budget (预算)** | 每次实验的固定资源上限，使结果可比较 |
| **Harness (评估)** | 冻结的评估命令，agent 无法修改评估方式（防作弊） |
| **Protocol (协议)** | 保留/回滚规则、变异策略、循环模式 |

## 工作流程

```
/create-autoresearch
      │
      ▼
Phase 1: 4 个专家 agent 并行分析代码库
         (架构分析师、度量侦察兵、运行分析师、领域分析师)
      │
      ▼
Phase 2: 与用户逐步对话，确认 5 个支柱
         (每次只问一个问题，基于专家分析给出推荐)
      │
      ▼
Phase 3: 生成项目专属 skill
         (实验循环 + 三层判定 + 自我优化 + 知识库)
      │
      ▼
Phase 4: 用户审批 → 调用 /autoresearch-<domain> 开始优化
```

## 关键特性

### 三层判定系统

| 层级 | 类型 | 作用 |
|------|------|------|
| L1 | 指标检查 | 主指标是否改善？护栏指标是否在安全范围内？ |
| L2 | 合规检查 | 测试通过？Lint 通过？冻结文件未被修改？ |
| L3 | 专家审查 | 基于默会知识 (Michael Polanyi) 的直觉判断——超越规则的经验 |

**L2 守护显性知识**（规则、测试、lint），**L3 守护默会知识**（直觉、经验、模式识别）。两者互补，不重叠。

### 三层领域分级

| 分级 | 反馈周期 | 循环模式 | 示例 |
|------|---------|---------|------|
| T1 | 秒级，确定性 | 完全自治 | ML 训练、编译器优化、数据库查询 |
| T2 | 秒级，有噪声 | 自治 + 噪声处理 | Web 性能、API 延迟、游戏 FPS |
| T3 | 天/周级 | 交互式，用户参与 | 转化率、SEO 排名、A/B 测试 |

### 自我优化 (Meta-Review)

生成的 skill 能优化自身。当遇到瓶颈时（连续 5 次丢弃、检测到收敛、或每 20 次实验），它会：

1. 分析命中率、收益递减、crash 模式、变异方向
2. 对可变协议规则提出具体修改建议
3. 用户逐条审批
4. 更新后继续循环

安全边界：**度量定义和评估命令是冻结的**——skill 无法改变成功的衡量标准。

### 知识库系统

防止重复踩坑，沉淀有效经验：

- **错题集 (Anti-Patterns)** — 失败的尝试，附带原因分析
- **经典案例 (Proven Patterns)** — 成功的方法，附带适用条件
- **领域启发规则 (Domain Heuristics)** — 从多次实验中提炼的泛化规则

知识生命周期：`候选 → 正式 → 启发规则 → (可选) 废弃`

被废弃的知识不会删除，而是标记废弃原因——知道一个规则为什么不再成立，本身也是知识。

### 护栏指标 (Guard Metrics)

创建 skill 时，专家会模拟"如果没有这个护栏会发生什么"，将具体的风险场景展示给用户选择。护栏指标作为硬约束，防止 agent 为了主指标而牺牲其他质量维度。

## 适用领域

| 领域 | 指标示例 | 分级 |
|------|---------|------|
| ML 模型训练 | val_loss, val_bpb | T1 |
| 编译器优化 | benchmark 执行时间 | T1 |
| 数据库查询优化 | 平均查询时间 | T1 |
| Web 前端性能 | Lighthouse 分数 | T2 |
| API 延迟优化 | p95 响应时间 | T2 |
| 测试覆盖率 | 变异测试分数 | T2 |
| 游戏渲染优化 | 基准场景 FPS | T2 |
| 构建体积优化 | gzip 后 KB | T2 |
| 电商转化率 | 点击率 / 购买率 | T3 |
| 内容 SEO | 搜索排名 | T3 |

有可度量的指标 + 可修改的代码，就能用。

## 安装

```bash
# 通过 Claude Code 插件市场（推荐）
/plugin install autoresearch-creator

# 或从源码加载（开发调试用）
claude --plugin-dir /path/to/autoresearch
```

## 快速开始

```bash
# 在你的项目目录下，调用工厂 skill
/create-autoresearch

# 或者带上领域提示，加速分析
/create-autoresearch web-performance
```

Skill 会：
1. 用 4 个专家 agent 分析你的代码库
2. 引导你逐步定义优化问题
3. 生成完整的 autoresearch skill

然后调用生成的 skill 开始优化：

```bash
/autoresearch-web-performance
```

## 生成的文件

运行 `/create-autoresearch` 后，你的项目中会创建：

```
.claude/skills/autoresearch-<domain>/
  SKILL.md              # 实验循环入口
  program.md            # 冻结规则 + 可变规则
  meta-review.md        # 自我优化协议
  knowledge-seed.md     # 初始领域知识

autoresearch/
  results.tsv           # 实验记录 (commit, metric, status, description)
  knowledge.md          # 持续增长的知识库
```

## 致谢

本项目的灵感直接来自 Andrej Karpathy 的 [autoresearch](https://github.com/karpathy/autoresearch)。autoresearch 展示了自治实验循环在 ML 模型优化中的强大能力——约束搜索空间、固定资源预算、使用单一不可篡改的指标、让棘轮机制确保只会前进。

我们将这一工程哲学从 ML 训练泛化为领域无关的框架，让任何人都能把它应用到自己的优化问题上。感谢 Karpathy 的开创性工作和开源精神。

## 许可证

MIT

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=vibe-agi/autoresearch&type=Date)](https://star-history.com/#vibe-agi/autoresearch&Date)
