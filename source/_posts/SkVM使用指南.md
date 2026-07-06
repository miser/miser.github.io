---
title: SkVM 如何度量并优化 LLM Skill
date: 2026-07-06 10:00:00
category: 技术
tags:
  - Skill
  - LLM
keywords: SkVM,LLM,Skill
description: SkVM使用指南
---
## 前言：Skill 迭代中的两个核心焦虑

在沉淀了一套 Skill 体系：一个主 Skill 负责意图识别与能力路由，多个子 Skill 负责具体功能执行。随着迭代深入，我遇到了两个越来越突出的问题。

**第一，能力裂化难以感知。** Skill 里充斥着复杂的自然语言描述，每次迭代都会调整这些描述。改动后会不会影响其他模块？影响面有多大？仅凭人工 Review 很难给出确定答案。我们缺少一套可复现的评测机制，把「感觉没问题」变成「数据证明没问题」。

**第二，模型一致性难以保证。** Skill 开发者通常基于某个模型编写和验证，在这个模型上跑得通，不等于在其他模型上也能跑得通。推向市场后，用户会调用不同厂商、不同参数的模型；遇到能力较弱的模型，Skill 很可能无法完成预期能力。

因此，我们需要一套面向 LLM Agent Skill 的评测与优化框架，来防止能力裂化、保证跨模型一致性。

之前在 ATA 上看到这篇文章：[让“Skill优化”从“玄学调参”变成“工程流水线”——SkVM 原理分析及评测](https://ata.atatech.org/articles/11020630805)。研究并实践后发现非常有价值，于是写文记录，欢迎一起学习交流，寻找保证 Skill 迭代质量的更优方案。

> **SkVM** 是一个面向 LLM Agent Skill 的编译与运行时系统，目标是让 Skill 能够在不同模型与 Harness 之间迁移与复用。 GitHub：[SJTU-IPADS/SkVM](https://github.com/SJTU-IPADS/SkVM)

接下来，我先以我自己开发的一套 Skill 为例，直观展示 SkVM 带来的帮助；再深入到原理层面，拆解它是如何工作的。

<!-- more -->

## 一、我的实践：一个 Skill 的评测 → 诊断 → 迭代闭环

### 1.1 业务背景

我开发的这套 Skill 服务于中台体验平台，核心任务是对 T-1 的中台埋点数据进行分析。一次典型的分析任务涉及：分析对象、时间跨度、期望指标、结果解读等。实际场景中，系统还要根据用户问题的描述，判断哪些参数需要 LLM 推测、哪些需要向用户澄清、甚至在信息缺失时决定下一步操作。

「识别用户意图并解析出上下文所需参数」是一种非常常见且关键的能力。我们就以它为例，介绍 SkVM 如何帮助我们提升质量。

### 1.2 工程能力搭建

SkVM 对外是一个 SDK。为了方便对多个 Skill 进行测试与数据沉淀，我在它的基础上封装了一个简易工程：

```plaintext
experience-skvm
├── bench                         # 评测计划配置目录
│   ├── exp-main                  # 主SKILL
│   │   └── plans
│   │       ├── parameter-parsing-plan.yaml           # 参数解析测试任务
│   │       └── ab-v1-vs-jit-v1.yaml                  # 迭代优化测试任务
│   └── exp-data-***              # 其他skill
│       └── plans
│           └── ***.yaml          # 其他测试任务
│
├── scripts                       # 评分脚本目录
│   ├── grade_parameter.py        # 参数解析评分
│   ├── grade_quality.py          # 质量诊断评分
│   ├── grade_query.py            # 查询执行评分
│   ├── grade_report.py           # 报告查询评分
│   └── grade_sql.py              # SQL 生成评分
│
├── skvm-data                     # SkVM 运行数据与产物
│   ├── profiles                  # 性能 Profile 数据
│   ├── reports                   # 评测报告归档
│   │   └── exp-main              # Skill的测试报告
│   ├── skills                    # 各 Skill 多版本定义
│   │   ├── exp-main
│   │   │   ├── v1
│   │   │   └── jit-v1            # Skill 的不同版本
│   │   └── exp-data-***
│   │       ├── v1
│   │       └── jit-v1
│   └── tasks                     # 评测用例集
│       ├── exp-main
│       │   └── parameter-parsing
│       │       └── caseXX/       # 具体测试用例
│       │           ├── task.json
│       │           └── grade.py
│       └── exp-data-***
│           └── sql-generation
│
├── templates                     # 任务与评分模板
│   ├── task-template.json
│   ├── grade-config-template.json
│   └── grade-sql-config-template.json
│
├── .env                          # 本地环境变量
├── .env.example                  # 环境变量示例
├── .gitignore
├── .venv                         # Python 虚拟环境
├── README.md
├── bench.sh                      # 单版本评测脚本
├── bench-ab.sh                   # A/B 对比评测脚本
├── compile.sh                    # 编译/构建脚本
├── jit-optimize.sh               # JIT 优化执行脚本
├── profile.sh                    # 性能采集脚本
├── run-jit-log.sh                # JIT 优化日志脚本
├── run-jit-verify.sh             # JIT 优化验证脚本
```

`bench.sh` 是对 SkVM 核心命令的进一步封装，主要完成三件事：

1.  安装 `skvm` 命令；
    
2.  封装 SkVM 调用，使其符合该工程的目录约定；
    
3.  每次评测后，将需要沉淀的数据从 `~/.skvm` 复制到工程目录，作为迭代记录。
    

### 1.3 执行流程

> 下面会提前用到 SkVM 中的一些概念，例如能力级别（L1–L3）、JIT 优化等。后续在「原理」部分会详细介绍。

**Step 1：定义评测用例（13 个 task.json）**

每个用例覆盖 L1 基础到 L3 高级，包含边界与异常场景，并配有标准答案：

```shell
{
  // ═══ 元信息 ═══
  "id": "parameter-parsing-case01",       // 唯一标识，用于日志和报告关联
  "name": "L1 基础系统级查询...",          // 人类可读名称
  "category": "parameter-parsing",         // 分类标签（同目录下的 case 共享）
  "difficulty": "easy",                    // 难度等级（仅文档用途，不影响评分）

  // ═══ Agent 执行配置 ═══
  "prompt": "帮我看下昨天轩辕的整体PV数据\n\n---\n请将你解析出的...",
  //  ↑ 发给被测 skill 的用户消息。"---" 之后是评测框架追加的输出格式要求
  "timeoutMs": 60000,                      // 单 case 超时（60s）
  "maxSteps": 10,                          // Agent 最多执行 10 步 tool call

  // ═══ 期望输出（供评分参考）═══
  "expected_output": {
    "ds": "20260624",
    "date_range": { "type": "single" },
    "app_name": "轩辕",
    "role_names": [],
    "bu_names": [],
    "data_type": "system",
    "page_codes": [],
    "resolved_by": "experience-data"
  },
  // ↑ 标准答案，grade.py 读取后逐字段对比 Agent 实际输出

  // ═══ 评分流水线（按顺序执行）═══
  "eval": [
    {
      "id": "output-exists",
      "method": "file-check",              // SkVM 内置评估器
      "path": "output.json",               // 检查 Agent 工作目录下是否存在此文件
      "mode": "json-schema",               // 验证模式：JSON Schema 校验
      "expected": "{...required:[ds,app_name,data_type]}",  // 最低结构要求
      "weight": 0.1                        // 占总分 10%
    },
    {
      "id": "parameter-grading",
      "method": "custom",                  // 自定义评估器
      "evaluatorId": "python-grade",       // 告诉 SkVM 使用 python-grade 协议
      "weight": 0.9                        // 占总分 90%
    }
  ]
}
```

**Step 2：定义评分规则（**`**grade_parameter.py**` **本地评分）**

从四个维度打分：完整性、准确性、边界处理、格式合规。规则是确定性的，可复现，不依赖 LLM as Judge。

**Step 3：基线评测（**`**skvm bench**`**）**

跑 v1 版本的 `SKILL.md`，量化当前能力，暴露问题。

**Step 4：JIT 优化（**`**skvm jit-optimize**`**）**

SkVM 自动分析失败原因，生成 `SKILL.md` 修改建议，产出 `jit-v1` 候选版本。

**Step 5：A/B 验证并固化**

用同一套 case 集对比 `v1` 与 `jit-v1`，确认提升后固化为正式版本。

### 1.4 数据发生了什么变化

| 阶段 | 均分 | case通过率 |
| --- | --- | --- |
| 初始版本（未优化） | 0.617 | 33% |
| JIT 优化后（jit-v1） | 0.810 | 88% |
| 对比提升 | +0.193 | +55 个百分点 |

**诊断暴露的 4 类问题：**

| 问题 | 影响面 | 根因 |
| --- | --- | --- |
| `date_range` 格式错误 | 2 个 case | `SKILL.md` 未定义结构化 schema |
| `data_type` 默认值缺失 | 3–4 个 case | 决策规则未显式写入 |
| 枚举外值未校验 | 3 个 case | 缺少 validation → clarification 流程 |
| Agent 超时/未产出 | 3–4 个 case | prompt 过长 / 步骤数不足 |

### 1.5 所有 Skill 测试结果

| Skill | Baseline | JIT | 结论 |
| --- | --- | --- | --- |
| main（主 Skill：能力路由与核心参数解析） | 0.617 | 0.810 | 显著提升，JIT 版本已固化 |
| quality（质量诊断） | 0.528 | 0.961 | 提升 82%，JIT 版本已固化 |
| sql（SQL 生成） | 0.760 | 0.775 | 仅提升 1.5%，放弃 JIT 版本 |
| query（查询执行） | 0.780 | 退化 | 放弃 JIT 版本 |
| report（报告生成） | 0.960 | skip | 逻辑较简单，暂不投入测试成本 |

### 1.6 使用 SkVM 的感受

**对 Skill 的直接价值：**

*   从「凭感觉觉得还行」变成 **0.617 → 0.810 的量化证据**；
    
*   每次修改 `SKILL.md` 后都可回归验证，**防止迭代引发能力裂化**；
    
*   JIT 优化自动发现问题并生成修复建议，**人工只需 Review 和决策**。
    

**对团队的方法论价值：**

*   **可复现**：同一 case 集 + 确定性评分 → 任何人、任何时间都能跑出一致结果；
    
*   **可对比**：A/B 报告直接展示 v1 vs jit-v1，不再依赖主观判断；
    
*   **可积累**：case 库持续扩展，质量门槛只升不降。
    

SkVM 让 LLM Skill 的质量从**黑盒感知**变成**白盒度量**：定义标准 → 量化现状 → 自动优化 → 验证提升 → 固化成果，形成可持续的质量飞轮。

# 二、SkVM 原理解读

前文主要介绍了 `jit-optimize` 的使用效果，但 SkVM 的能力远不止于此。它包含四个核心子系统：**Profiler、Compiler、JIT-boost、JIT-optimize**。

### 2.1 整体架构

SkVM（Skill Virtual Machine）是一个 LLM Agent Skill 编译与优化系统，通过「分析模型能力 → 编译适配 → 运行时优化」的三层架构，让 Skill 能自动适配不同 LLM 模型。

```plaintext
┌─────────────────────────────────────────────────────────────────────┐
│                          SkVM 完整流水线                              │
└─────────────────────────────────────────────────────────────────────┘

① Profiler ─────► TCP (能力档案)
                    │
                    ▼
② Compiler ─────► 编译后的 Skill 变体 (适配弱模型)
                    │
                    ▼
③ Runtime ──────► Agent 执行 Skill
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
④ JIT-boost               ⑤ JIT-optimize
   (运行时拦截)               (多轮迭代)
   重复调用→直接执行          执行证据→改进Skill
   延迟：~10ms               周期：分钟/小时
   无需重编译                 产出新Proposal

```

| 子系统 | 运行时机 | 输入 | 输出 | 是否调LLM |
| --- | --- | --- | --- | --- |
| Profiler | 首次使用模型 | 模型+Adapter | TCP | 是（26×3次） |
| Compiler | 部署前 | Skill + TCP | 适配变体 | 是（3个Pass） |
| JIT-boost | 每次Agent调用 | LLM response | 直接执行结果 | 编译时是，运行时否 |
| JIT-optimize | 离线/后台 | Skill + 执行日志 | 改进后的Skill | 是（优化器Agent） |

**核心设计哲学**：借用传统编译器的 AOT + JIT 分层思想，但操作对象不是机器码，而是**自然语言 Skill 指令**。Profiler 相当于「目标平台探测」，Compiler 相当于「交叉编译」，JIT-boost 相当于「热点内联」，JIT-optimize 相当于「PGO（Profile-Guided Optimization）」。

> 说明：架构图里画了 5 个编号，是因为把 Runtime 也作为独立一层列出；日常讨论时常说「四个核心子系统」，是指 Profiler、Compiler、JIT-boost、JIT-optimize，两者并不矛盾。

## 2.2 Profiler：能力画像系统

#### 2.2.1 谁的能力画像

Profiler 评测的是 **LLM + Harness 组合** 在 26 个维度上的实际表现，最终生成 TCP（Target Capability Profile）。示例可参考：[openclaw + qwen3.6 的 TCP](https://github.com/SJTU-IPADS/SkVM-data/blob/d92112ae03aa6d50e5ab02c9db63d60058ac215d/profiles/openclaw/openrouter--qwen--qwen3.6-plus/latest.json)。

我直接使用了社区现成的 Profile，没有自己评测，以节省时间和 Token 成本。

```typescript
// 26个原语，每个分L1/L2/L3三级
type Level = "L0" | "L1" | "L2" | "L3"

// TCP = 最终产物：模型能力档案
interface TCP {
  model: string          // "anthropic/claude-sonnet-4.6"
  harness: string        // "bare-agent"
  capabilities: Record<string, Level>  // { "gen.code.python": "L2", ... }

  details: PrimitiveProfileDetail[]

}

// 每个Generator生成测试实例
interface MicrobenchmarkInstance {
  prompt: string                        // 给LLM的指令
  setupFiles?: Record<string, string>   // 预置文件
  eval: EvalCriterion                   // 评判标准
}
```
```markdown
generators/
├── 代码生成类 (5个)
│   ├─ gen-code-python.ts
│   ├─ gen-code-javascript.ts
│   ├─ gen-code-html.ts
│   ├─ gen-code-shell.ts
│   └─ gen-code-sql.ts
│
├── 文本生成类 (4个)
│   ├─ gen-text-long.ts
│   ├─ gen-text-prose.ts
│   ├─ gen-text-structured.ts
│   └─ gen-regex.ts
│
├── 推理类 (5个)
│   ├─ reason-analysis.ts
│   ├─ reason-arithmetic.ts
│   ├─ reason-logic.ts
│   ├─ reason-planning.ts
│   └─ reason-spatial.ts
│
├── 工具使用类 (8个)
│   ├─ tool-exec.ts
│   ├─ tool-file-read.ts
│   ├─ tool-file-write.ts
│   ├─ tool-web.ts
│   ├─ tool-browser.ts
│   ├─ tool-call-batch.ts
│   ├─ tool-call-format.ts
│   └─ tool-call-batch.ts
│
└── 指令遵循类 (4个)
    ├─ follow-constraint.ts
    ├─ follow-delegation.ts
    ├─ follow-format.ts
    ├─ follow-procedure.ts
    └─ follow-style.ts
```

#### 2.2.2 解决什么问题

Skill 运行在「LLM + Harness」环境中，不同组合的能力直接影响 Skill 的最终表现。因此需要先刻画运行环境的能力，再与 Skill 所需能力对比，才能有的放矢地优化。

如果 Skill 在不同 LLM + Harness 组合下都表现良好，那么放到市场中供任何人使用都不会有大的问题。这就是解决本文开头第二个问题——如何保证 Skill 描述的能力一致性。

#### 2.2.3 执行流程

```python
profile(model, harness):
  if 缓存存在 → return 缓存TCP
  
  for each primitive in 26_PRIMITIVES:    // gen.code.python, tool.exec, ...
    generator = getGenerator(primitive)
    highestLevel = "L0"
    
    for level in [L1, L2, L3]:           // 从易到难
      instances = generator.generate(level) × 3  // 生成3个随机实例
      
      for each instance:
        result = adapter.run(instance.prompt, workDir)  // Agent 执行
        score  = evaluate(instance.eval, result)        // 评分
      
      if 通过率 == 100%:
        highestLevel = level              // 升级！
      else:
        break                             // 停止，当前level不通过
    
    tcp.capabilities[primitive] = highestLevel
  
  save(tcp) → ~/.skvm/profiles/
  return tcp

```

**关键点**：L1→L2→L3 逐级递进，比如 `gen.code.python` 的 L1 可能是"写 Hello World"，L3 可能是"实现并发 HTTP 服务器"。

## 2.3 Compiler：AOT 编译器

#### 2.3.1 解决什么问题

Compiler 将 Skill 针对特定模型的能力缺口进行重写，产出适配后的 Skill 变体，主要面向**弱模型场景**。

> 个人建议：如果后续还要用 JIT-optimize 继续优化，可以考虑跳过 Compiler AOT 这一步。因为编译后的 Skill 变体往往会因为「能力降级」而比原始 Skill 更复杂，反而增加后续 JIT 优化的难度。

我开发的这套 Skill 目标是在 QoderWork 里使用，没有弱模型场景，所以直接跳过了 Compiler AOT。

#### 2.3.2 举例：把「代码审查」Skill 从强模型适配到弱模型

**原始 Skill**

```markdown
# Code Review Skill

## 任务
审查用户提交的代码变更，找出 bug、安全漏洞和性能问题。

## 步骤
1. 读取 git diff 内容
2. 分析每个变更文件的上下文（读取相关引用文件）
3. 综合推理潜在影响，生成结构化审查报告：
   - 按严重程度分级（Critical/Warning/Info）
   - 给出修复建议代码片段
   - 评估变更对系统整体架构的影响

## 输出格式
生成 `review.json`，包含 findings 数组

```

**Step 1：extractSCR —— LLM 分析 Skill 需要什么能力**

```log
SCR = {
  purposes: [{
    id: "code-review",
    primitives: [
      { id: "tool.file.read",    minLevel: "L2" },  // 读多文件+上下文
      { id: "reason.analysis",   minLevel: "L3" },  // 综合推理架构影响
      { id: "gen.code.python",   minLevel: "L2" },  // 生成修复代码
      { id: "follow.format",     minLevel: "L2" },  // 结构化 JSON 输出
      { id: "gen.text.structured", minLevel: "L2" } // 生成分级报告
    ]
  }]
}
```

**Step 2：Gap 分析 —— 对比模型实际能力**

目标模型 TCP（比如 Qwen3-7B）：

```markdown
tool.file.read:      L2 ✓
reason.analysis:     L1 ✗ (需要L3，只有L1)
gen.code.python:     L1 ✗ (需要L2，只有L1)
follow.format:       L2 ✓
gen.text.structured: L1 ✗ (需要L2，只有L1)
```

Gap 结果：

```markdown
reason.analysis:     需要L3 → 实际L1  （差2级，严重）
gen.code.python:     需要L2 → 实际L1  （差1级）
gen.text.structured: 需要L2 → 实际L1  （差1级）
```

**Step 3：Rewrite —— LLM 重写 Skill 补偿缺口**

编译后的变体：

```c
# Code Review Skill (Compiled for Qwen3-7B)

## 任务
审查用户提交的代码变更，找出明显的 bug 和问题。

## 步骤
1. 读取 git diff 内容
2. 对每个变更文件，**逐个**检查以下清单：
   □ 是否有未处理的 null/undefined？
   □ 是否有明显的逻辑错误（条件反转、死循环）？
   □ 是否有安全风险（SQL拼接、eval使用）？
   □ 是否有资源泄漏（未关闭的文件/连接）？
3. 对每个发现的问题，按以下模板输出：

## 输出格式
生成 `review.json`，严格按以下模板填写：

```json
{
  "findings": [
    {
      "file": "文件路径",
      "line": 行号,
      "severity": "critical 或 warning",
      "issue": "一句话描述问题",
      "suggestion": "一句话描述如何修复"
    }
  ]
}
```

**变体做了什么**

| 原始要求 | Gap | 编译后的降级策略 |
| --- | --- | --- |
| "综合推理架构影响" (reason.analysis L3) | 差2级 | **直接删除**这个要求，改为逐文件检查清单 |
| "生成修复代码片段" (gen.code.python L2) | 差1级 | **降级为**"一句话描述修复方向"（文本即可） |
| "生成分级结构化报告" (gen.text.structured L2) | 差1级 | **提供固定模板**，只需填空（follow.format L2 够用） |

**关键点**：把「需要高级推理能力」的开放式任务，改写成「只需要基础能力」的结构化 / 清单式任务。牺牲一部分深度，换取弱模型也能可靠完成。

## 2.4 JIT-boost：运行时代码固化

#### 2.4.1 解决什么问题

在对 Skill 进行评测的过程中，会触发大量 LLM 调用，带来两个问题：一是时间和 Token 成本巨大；二是短时间大量调用可能触发模型接口限流，造成工作流阻塞和费用激增。而实际上，很多调用返回的结果是可以被缓存或固化的。

JIT-boost 是一个运行时热点探测器：当发现 Agent 反复让 LLM 生成同类型的工具调用时，直接提取并固化调用模板，后续用预编译的脚本替代 LLM 调用，绕过不必要的 LLM 调用。

#### 2.4.2 举例：JSON 格式化助手

前 3 次 LLM 正常思考并返回命令，第 4 次起直接执行 `python -m json.tool`：结果一样，但延迟从 30 秒降到 10 毫秒。

```markdown
第 1 次 Agent 调用:
  用户: "把 data.json 格式化"
  LLM 返回: tool_call("bash", { command: "python3 -m json.tool data.json > formatted.json" })
  
  Solidifier 对比: "python3 -m json.tool" 匹配 codeSignature ✓
  → consecutive = 1

第 2 次 Agent 调用:
  用户: "把 config.json 美化一下"
  LLM 返回: tool_call("bash", { command: "python3 -m json.tool config.json > pretty.json" })
  
  → consecutive = 2

第 3 次 Agent 调用:
  用户: "format output.json"
  LLM 返回: tool_call("bash", { command: "python3 -m json.tool output.json > result.json" })
  
  → consecutive = 3 → ★ PROMOTED! ★
```

某次短路执行失败了

```markdown
第 7 次:
  用户: "format 一下那个 broken.json"（文件其实有语法错误）
  
  Solidifier 执行模板 → python3 报错!
  → fallbackCount = 1（记录失败）
  → 回退本次，让 LLM 正常处理

如果连续失败 3 次:
  → DEMOTED! 回退到监控状态
  → 重新观察，不再冒然拦截
```

**状态机全景**

```markdown
          ┌──────────────────────────────────────────┐
          │                                          │
          ▼                                          │
    ┌───────────┐   连续匹配3次      ┌───────────┐    │
    │ MONITORING │ ──────────────► │ PROMOTED  │    │
    │ (观察中)   │                  │ (已激活)   │    │
    └───────────┘                  └─────┬─────┘    │
          ▲                              │          │
          │          连续失败3次           │          │
          └──────────────────────────────┘          │
                                                    │
          不匹配时: consecutive 归零 ─────────────────┘
```

**对比效果**

|  | **无 JIT-Boost** | **有 JIT-Boost（已推升）** |
| --- | --- | --- |
| "format data.json" | 调 LLM → 思考 → 返回命令 → 执行 | 直接执行命令 |
| 延迟 | ~3-8 秒 | ~10-50 毫秒 |
| Token 消耗 | ~500 tokens | 0 tokens |
| 成本 | ~$0.003 | $0 |
| 质量 | LLM 可能输出略有不同 | 100% 确定性（同一模板） |

## 2.5 JIT-optimize：Skill 迭代优化

#### 2.5.1 解决什么问题

通过一个个 Test Case 对 Skill 进行多轮循环评测和优化，基于执行证据自动改进 Skill 内容，防止能力裂化，甚至提升迭代质量。

#### 2.5.2 执行流程

```plaintext
jitOptimize(config)
  │
  ▼
┌─ Round 0 (Baseline) ─────────────────────────────┐
│  1. 加载原始 Skill                                 │
│  2. 解析任务 → train集 + test集(held-out)           │
│  3. 对每个任务执行 runsPerTask 次                   │
│  4. 收集 Evidence (score + criteria + 对话日志)    │
│  5. 计算 baselineScore                            │
└──────────────────────────────────────────────────┘
  │
  ▼
┌─ Round 1..N (优化循环) ──────────────────────────┐
│                                                    │
│  ┌─① 优化器Agent─────────────────────────┐        │
│  │ 输入: 当前Skill + TRAIN证据 + 历史       │        │
│  │ (注意: TEST证据不给优化器看！)            │        │
│  │                                         │        │
│  │ 优化器读取证据，诊断根因，编辑Skill      │        │
│  │ 输出: submission.json                   │        │

│  │   { rootCause, changes[], confidence }  │        │

│  └─────────────────────────────────────────┘        │
│                    │                                 │
│                    ▼                                 │
│  ┌─② 应用变更─────────────────────────────┐        │
│  │ 将优化器的edits应用到Skill文件            │        │
│  │ 快照保存到 round-N/ 目录                 │        │
│  └─────────────────────────────────────────┘        │
│                    │                                 │
│                    ▼                                 │
│  ┌─③ 重新执行─────────────────────────────┐        │
│  │ 用新Skill跑 train+test 全部任务          │        │
│  │ 收集新一轮 Evidence                      │        │
│  │ 计算 trainScore + testScore              │        │
│  └─────────────────────────────────────────┘        │
│                    │                                 │
│                    ▼                                 │
│  收敛检查: testScore ≥ 0.95 ? → 提前退出           │
└──────────────────────────────────────────────────┘
  │
  ▼
┌─ Best Round 选择 ────────────────────────────────┐
│  for each round:                                   │
│    ① 改进 ≥ minImprovement (2%) ?                  │
│    ② 任何单个task回归 ≤ 20% ?                       │
│    ③ 分数相同时，成本更低 ≥ 15% ?                   │
│  → 选出最优round                                   │
└──────────────────────────────────────────────────┘
  │
  ▼
  输出: Proposal { bestRound, proposalDir, status: "pending" }

```

#### 2.5.3 评分机制（执行 → 打分 → 汇总）

**评估器类型**

每个任务都有一组评估标准，由不同类型的评估器执行：

| 评估器 | 工作方式 | 分数 | 典型场景 |
| --- | --- | --- | --- |
| `script` | 在工作目录执行命令，读取输出分数 | 0~1 | `python3 check.py` |
| `file-check` | 检查文件是否存在/内容匹配 | 0 或 1 | 检查 `output.json` |
| `llm-judge` | 调用 LLM 按 rubric 打分 | 0~1 | "代码是否处理了边界情况" |
| `custom` | 自定义 Python 脚本 | 0~1 | `grade.py` 返回分数 |

**评分公式**

```plaintext
单次运行分数 = Σ(criterion.weight × criterion.score) / Σ(criterion.weight)

例：
  script(weight=0.4):     score=1.0  →  0.40
  file-check(weight=0.3): score=1.0  →  0.30
  llm-judge(weight=0.3):  score=0.5  →  0.15
  ──────────────────────────────────────────
  本次分数 = 0.85

多次运行同一任务 → 取平均
多个任务 → 再取平均 → roundScore

```

**Tainted（污染）运行**

如果一次运行因**基础设施问题**失败（超时、崩溃、adapter 挂了），而非 Skill 质量问题，该运行被标记为 tainted：

*   Tainted 运行的分数**不计入** roundScore
    
*   如果一轮中所有运行都是 tainted → 整轮终止
    
*   优化器在证据中能看到 "TAINTED" 标记，但不会据此优化（因为不是 Skill 的锅）
    

**Train/Test 分组**

```plaintext
任务集
├── TRAIN（~70%）→ 证据提供给优化器，优化器据此改 Skill
└── TEST（~30%） → 优化器看不到！用来验证改进是否泛化

为什么分？防过拟合。
如果优化器只针对 TRAIN 硬编码答案，TEST 分数不会提高。

```

框架既支持指定集合，也支持让 LLM 自动生成。

**重要澄清：评分不感知 L1/L2/L3**

JIT-optimize 的评分与 Profiler 的 L1/L2/L3 能力分级**完全无关**：

|  | Profiler 评分 | JIT-optimize 评分 |
| --- | --- | --- |
| 关心什么 | 模型在 26 原语上的能力等级 | 任务执行的结果质量 |
| 感知 L1-L3？ | 是 | 否 |
| 评分方式 | 逐级测试通过率 | 文件对不对、脚本跑通吗、LLM 判官觉得好吗 |
| 输出 | TCP：`gen.code.python: "L2"` | 分数：`0.85` |

JIT-optimize 不管你用什么方式做到的，只看**做到了没有**。正是因为它的独立性，所以我可以直接用它来做测试，而不用让其他模块参与。

### 2.6 LLM 输出稳定性保障机制

SkVM 里很多地方都依赖 LLM 提取信息或产出结果，而 LLM 每次执行都有不确定性。SkVM 的设计**不追求单次 LLM 调用的确定性，而是通过系统级机制容忍和对冲不确定性**。

#### 2.6.1 六层稳定性策略

| 层次 | 策略 | 说明 |
| --- | --- | --- |
| 单次调用 | Schema 约束 + Zod 校验 | tool\_use 模式锁定输出格式，枚举值固定 |
| 多次调用 | 聚合统计 | Bench/Profiler 多次运行取平均值 |
| 多版本 | 竞争选优 | JIT-optimize 多轮产出，pickBestRound 选最好 |
| 系统级 | 确定性计算夹层 | Gap 分析等核心逻辑不涉及 LLM |
| 运行时 | 阈值容错 | JIT-boost 连续 3 次匹配才信任 |
| 最终兜底 | 实测评分说了算 | 不管中间过程，Bench 分数决定是否采纳 |

#### 2.6.2 具体机制

1.  **结构化输出约束**：extractSCR 使用 tool\_use + Zod，输出结构被锁死
    
2.  **多次运行聚合**：Profiler/Bench 对同一任务跑多次取平均，消除单次波动
    
3.  **竞争选优**：JIT-optimize 不信任单一结果，多轮竞争选最好
    
4.  **确定性计算隔离**：`analyzeGaps()` 等关键逻辑是纯计算，100% 确定性
    
5.  **阈值容错**：JIT-boost 需要连续 3 次匹配才晋升，过滤偶发模式
    
6.  **评测兜底**：所有优化产物最终都过 Bench 评测，分数不达标则丢弃
    

## 三、总结

在 Skill 的日常开发中，「如何保证复杂逻辑描述不出现能力裂化」和「如何保证不同模型下的推理一致性」是两大难题。不确定性会直接影响 Skill 的交付结果。

引入 SkVM 后，它通过四个模块提供了系统性的解法：

*   **Profiler** 刻画目标 LLM + Harness 的能力画像；
    
*   **Compiler** 根据能力差异重新编译 Skill，使其适配弱模型；
    
*   **JIT-boost** 在运行时固化热点调用，降低成本与延迟；
    
*   **JIT-optimize** 对沉淀的 Test Case 进行多轮测试，找出调优依据并自动改进 Skill。
    

在具体的实践中，我已经看到了 Skill 的优化结果：主 Skill 均分从 0.617 提升到 0.810，质量诊断 Skill 提升 82%，同时也节省了大量手工测试时间，开发质量和效率都有明显提升。

需要强调的是，**0.810 并不是终点，而是一个新的起点**。这个分数本身说明 Skill 仍有约 19% 的场景没有完全达标，还有继续优化的空间。借助 SkVM 的评测与 JIT-optimize 闭环，我可以继续扩展 Test Case、诊断剩余失败根因、生成新一轮优化版本，直到达到业务可接受的阈值。

后续我会继续迭代目前的工程能力，把更多 Skill 纳入这套评测体系，并探索 0.810 → 0.90+ 的路径，也欢迎一起交流。
