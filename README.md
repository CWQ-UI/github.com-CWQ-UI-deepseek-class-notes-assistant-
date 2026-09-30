# 课堂纪要助手 · DeepSeek Class Notes Assistant

> 一个基于 **[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)** 框架搭建的 AI 智能体，把课堂录音的语音识别（ASR）转写纯文本，整理成结构化、可复习的课堂纪要。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Framework: DeepSeek Harness](https://img.shields.io/badge/Framework-DeepSeek%20Harness-blue.svg)](#技术架构)
[![Preset: class-notes-organizer](https://img.shields.io/badge/Preset-class--notes--organizer-green.svg)](#如何运行)

---

## 目录

- [项目简介](#项目简介)
- [功能说明](#功能说明)
- [效果示例](#效果示例)
- [技术架构](#技术架构)
- [如何运行](#如何运行)
- [项目结构](#项目结构)
- [设计取舍与已知限制](#设计取舍与已知限制)
- [测试与验证](#测试与验证)
- [关于安全](#关于安全)
- [License](#license)

---

## 项目简介

课堂录音转成文字之后，得到的通常是一大段充满「嗯、那个、就是说」、同音字错认、语序颠倒、师生对话交错的原始文本，几乎无法直接复习。

**课堂纪要助手**是一个会话级 AI 智能体，专门解决这一步：它接收 ASR 原始转写，输出一份结构固定的 Markdown 课堂纪要——课堂主题、核心内容、重点与考点、待办与作业——并且在信息不足时**明确拒绝编造**，而不是硬凑出一份看似完整的纪要。

它不是一个独立的应用程序，而是 **DeepSeek Harness 的一个 Agent Preset（智能体预设）**，通过一份声明式 YAML 配置挂载到 DSH 会话中。

## 功能说明

| 能力 | 说明 |
|---|---|
| **纠错清洗** | 修正同音字、专业术语、人名、数字的识别错误；删除口语填充词、无意义重复与寒暄 |
| **结构重组** | 按讲解推进顺序重新组织，而不是照抄句子顺序；把同一知识点的分散表述合并到一处 |
| **固定四段输出** | 课堂主题 / 核心内容 / 重点与考点 / 待办与作业，全部 Markdown |
| **考点信号提取** | 识别「这个要考」「考试会考」「必考」「记住这个」「容易错」「划重点」等 10 类强调信号，单独成段，并按 `- **<知识点>** —— 教师原话信号：「……」` 格式保留教师原话 |
| **师生互动压缩** | 问答不逐句复述，合并为知识点并标注「提问（生）→ 解答（师）」；点名、答到、维持纪律等一律省略 |
| **忠实转写** | 不确定的词保留原文并加「（疑似：…）」，不做猜测式改写；定义、公式、数值、步骤、举例、反例都保留，不做概括性丢弃 |
| **信息不足保护** | 内容不足以支撑至少两个实质知识点时，明确声明无法整理，逐条列出缺失项，**禁止编造**知识点、数值、人名、作业要求 |

## 效果示例

**输入**（ASR 原始转写，节选）：

> 呃 今天我们讲那个二叉搜索数 嗯 就是说 它首先是一棵二叉树 那个 对于任意一个节点 它的左子树的值都比它小 右子树都比它大 这个就叫 嗯 二叉搜索树性质……

**输出**（节选，完整版见 [`docs/tests/case-1-normal-notes.md`](docs/tests/case-1-normal-notes.md)）：

```markdown
### 课堂主题

本节课讲解二叉搜索树（BST）的定义、查找与时间复杂度，以及因退化问题引出的
平衡二叉树（AVL 树、红黑树）；下节课安排讲树的遍历（前序、中序、后序）。

### 核心内容

- **一、二叉搜索树（BST）的定义与性质**
  - BST 首先是一棵二叉树。
  - 对于任意一个节点：其左子树中所有节点的值都小于该节点，右子树中所有节点的值都大于该节点。

- **三、查找的时间复杂度**
  - 平均情况下为 O(log n)，即 log 以 2 为底 n。
  - 最坏情况下会退化成链表，复杂度为 O(n)。

### 重点与考点

转写中未出现明确的考点信号。      ← 无信号时如实声明，绝不臆造考点

### 待办与作业

- 课后自行阅读教材第六章。
- 下节课讲树的遍历：前序、中序、后序。
```

> 注意上面「重点与考点」这一段：转写里**没有**任何强调信号，智能体就如实写「未出现明确的考点信号」。
> 这是本项目最看重的行为——**宁可少写，不可编造**。

含考点信号、含师生互动、以及信息不足（边界情形）的输出示例，分别见
[`case-3`](docs/tests/case-3-exam-signal.md)、[`case-2`](docs/tests/case-2-dialogue.md)、[`case-4`](docs/tests/case-4-too-short.md)。

## 技术架构

本项目基于 **DeepSeek Harness（DSH）** 框架搭建。DSH 是一个以 **Cordis 插件容器**为核心的智能体运行时：所有能力——提示词（persona）、文件读写工具、shell、子 Agent、workflow、上下文压缩——都是可插拔的**插件行（plugin row）**，通过分层 YAML 声明组装成一个会话。

课堂纪要助手的落地方式是一个 **Agent Preset**——也就是「一组预先组装好的插件行」，用户在新建 DSH 会话时选择它，该会话就只加载这套能力。

```
                        DeepSeek Harness (DSH)
                                │
                ┌───────────────┴────────────────┐
                │   profile patch 层（用户可写）   │   ← 本项目改动点，唯一
                │   ~/.dsh/profiles/<profile>/     │
                │        cordis.patch.yml          │
                └───────────────┬────────────────┘
                                │  insert 一行 preset 声明
                                ▼
              @deepseek-ai/dsh-agent-preset  →  preset-class-notes-organizer
                                │
                ┌───────────────┴────────────────┐
                │     该 Preset 的插件行（2 项）    │
                ├────────────────────────────────┤
                │ ① @deepseek-ai/dsh-persona      │  本项目的核心 Prompt
                │      prefix            = 课堂纪要助手系统提示词
                │      complete          = true   （成为唯一 system prompt）
                │      includeRuntimeContext = false
                │                                │
                │ ② @deepseek-ai/dsh-tool-fs      │  基础文件读写
                │      read / write / edit / glob │
                └────────────────────────────────┘
```

**为什么工具集只有 2 项？** DSH 的 `standard` 基线预设包含 24 项插件行（浏览器、子 Agent、workflow、goal、plan、shell、上下文压缩等）。本项目采用「最小能力集」思路，只保留完成任务所必需的两项，从而：

- 把 system prompt 完全交给 persona（`complete: true` 抑制其余所有提示词段落）；
- 消除智能体跑偏去联网、开子 Agent、执行命令的可能；
- 让提示词行为更容易预测和测试。

净结果：**24 项 → 2 项**。

**为什么不直接修改内置预设？** 内置 preset 属于 DSH 发行包，升级会被覆盖；而且 `@deepseek-ai/dsh-agent-preset-registry` 的设计是 *"A user override replaces the complete child list and does not automatically merge future changes to the builtin list"*。因此本项目以**新增一个独立 preset**的方式落地——内置 `standard` / `ptc` / `minimal` 全部保持原样，回滚只需删除配置里的 insert 块，无残留状态。

完整的架构论证、基线 24 项逐条处置表、以及一次「加载失败」故障的排查记录，见 [`docs/design.md`](docs/design.md)。

## 如何运行

### 前置条件

1. 已安装 **DeepSeek Orb**（内置 DeepSeek Harness 运行时）。
2. 已登录并配置好模型凭据（在 GUI 的「设置 → 账号」中完成，**不需要**修改本项目任何文件）。

### 安装步骤

**第 1 步：定位你的 profile patch 文件**

| 平台 | 路径 |
|---|---|
| Windows | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` |
| macOS / Linux | `~/.dsh/profiles/<profile>/cordis.patch.yml` |

`<profile>` 是你的 profile 名。常见的两个是 `web` 与 `desktop`；**当前 GUI 实际使用哪一个，以环境变量 `DSH_PROFILE` 为准**（也可以两个都写，互不影响）。

若文件不存在，新建一个，内容为一个 YAML 数组。

**第 2 步：粘贴 preset 声明**

打开 [`config/preset.example.yml`](config/preset.example.yml)，把其中第 1 步的 `- insert:` 块粘贴进你的 `cordis.patch.yml`。

**第 3 步：填入完整 Prompt**

把 [`prompts/persona-prefix.md`](prompts/persona-prefix.md) 的**完整正文**（不含该文件顶部的引用说明）逐字粘贴到 `prefix: |-` 之下，注意保持 YAML 块标量的缩进一致。

> ⚠️ `prefix` 用的是 YAML 块标量 `|-`，正文每一行都必须比 `prefix:` 这一行**多缩进**，否则会解析失败。

**第 4 步（可选）：设为默认预设**

把示例文件中第 2 步的 `agent-preset-registry` 段一并粘贴，`selectedDefault: class-notes-organizer` 会让新会话默认选中它。省略这一段时，预设仍会出现在选择器中，只是不会自动选中。

**第 5 步：重启并新建会话**

- preset 定义在**启动时**读取，因此**必须重启 DeepSeek Orb** 才能在 GUI 中看到新预设。
- 重启后新建会话，在 Preset 选择器中选「课堂纪要助手」。
- 注意：**已存在的会话不会切换 Preset**（registry 设计如此），必须新建会话。

### 使用方式

新建会话后，直接把课堂录音的 ASR 转写文本整段粘贴进去即可。例如：

```
帮我把这段课堂转写整理成纪要：

呃 今天我们讲那个二叉搜索数 嗯 就是说 它首先是一棵二叉树 那个 对于任意一个节点...
```

### 回滚

删除 `cordis.patch.yml` 中的 `- insert:` 整块（以及可选的 `agent-preset-registry` 段），重启 DeepSeek Orb 即可。无残留状态，内置预设从未被改动。

## 项目结构

```
class-notes-agent/
├── README.md                          # 本文件
├── LICENSE                            # MIT 协议
├── .gitignore                         # 排除 node_modules / .env / 密钥 / Office 临时文件
├── .gitattributes                     # 统一换行符为 LF，并标记二进制文件类型
│
├── prompts/
│   └── persona-prefix.md              # ★ 核心 Prompt：完整的系统提示词正文
│
├── config/
│   └── preset.example.yml             # ★ Preset 配置示例（脱敏，含逐段注释与设计说明）
│
└── docs/
    ├── design.md                      # 架构设计稿：Preset 格式判定、基线 24 项处置表、
    │                                  #   影响面与回滚、故障排查记录
    ├── test-report.md                 # 测试报告：L1 运行时装载 / L2 提示词行为 / L3 端到端
    └── tests/
        ├── case-1-normal-notes.md     # 正例：无强调信号的普通课堂笔记转写
        ├── case-2-dialogue.md         # 正例：含师生互动 + 考点信号
        ├── case-3-exam-signal.md      # 正例：密集强调信号（6 类）
        └── case-4-too-short.md        # 边界例：仅约 20 字 → 触发「信息不足」保护
```

## 设计取舍与已知限制

诚实声明如下：

1. **基线副本不自动合并**——本 preset 是 `standard` 基线的一份独立副本。DSH 升级若改动内置 `standard`，本 preset 不会自动跟随，需手工同步。这是「不覆盖内置预设」这一选择的代价。
2. **无上下文压缩**——已移除 `compaction` 组与 `tool-result-pruner`，**超长转写可能触及上下文上限**，需要分段投喂。
3. **无 shell**——已移除 `tool-bash` / `tool-pwsh`，本 preset 不能执行命令。
4. **无检索与提问工具**——最小重建版也移除了 `tool-fs-search`（grep）、`tool-ask-user`、`tool-todo`。副作用之一是 persona 中「使用 `ask_user_question` 工具」这一条缺少对应工具挂载，边界情形下依赖运行时容错（模型会改为输出等价的问题清单，行为已在测试中验证）。如需真实工具调用，可在 `plugins` 列表中补一行 `tool-ask-user`。
5. **`complete: true` 的副作用**——system prompt 只剩 persona 文本，工具清单仍通过 tool schema 通道提供给模型，但不再有额外的工具使用指引段落。
6. **L3 端到端未验证**——参见下方测试与验证。

## 测试与验证

完整报告见 [`docs/test-report.md`](docs/test-report.md)，分三层：

| 层级 | 内容 | 结论 |
|---|---|---|
| **L1 运行时装载** | 新 Preset 声明是否被运行中的 DSH host 接受 | ✅ 通过（`Config.listConfigs` 中新增 `include:preset-class-notes-organizer`） |
| **L2 提示词行为** | persona 是否产生符合规格的输出 | ✅ 4/4 通过（3 正例 + 1 边界例，真实模型推理） |
| **L3 端到端会话** | 用该 Preset 新建会话并核对工具集 | ⚠️ **未验证**，需人工确认 |

**L2 四个测试用例**：无强调信号时是否臆造考点（不臆造 ✅）、师生互动是否被逐句复述（不 ✅）、密集考点信号格式是否合规（4/4 ✅）、信息不足时是否编造（不 ✅）。

**L2 验证方法**：把配置中的 persona 文本**逐字复制**作为系统提示词，对 4 个样例运行真实模型推理，逐项核对输出。

**L3 为什么未验证**：测试时的运行进程本身就是该 profile 的 host，从既有会话内部无法注入新会话，而 Preset 的挂载只在新会话创建时发生。人工确认步骤见测试报告 §4。

## 关于安全

- 本仓库**不包含任何 API Key、Token 或凭据**。模型凭据由 DeepSeek Orb 自身的账号设置管理，不写进 preset 配置。
- 若你 fork 本项目并想加入自己的密钥：**请勿**提交 `.env` 或任何密钥文件，[`.gitignore`](.gitignore) 已预置相应规则。
- 仓库中不含任何真实课堂录音或学生个人信息；所有测试样例均为合成的教学素材（二叉搜索树 / TCP 三次握手 / 线性回归），不涉及真实个人数据。

## License

[MIT](LICENSE) © 2025 [CWQ-UI](https://github.com/CWQ-UI)

---

**框架**：[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（[开发者预览版介绍](https://www.deepseek.com/harness/) · [文档](https://deepseek-harness.github.io/deepseek-harness/)）
· **预设 ID**：`class-notes-organizer` · **预设显示名**：课堂纪要助手
