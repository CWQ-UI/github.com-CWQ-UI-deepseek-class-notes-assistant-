# 变更设计稿：会话级 Agent Preset `class-notes-organizer`

- 目标安装版本：DeepSeek Orb 内置 DSH `0.1.7-rc.1`（`@deepseek-ai/dsh-web-app` / `dsh-persona` 等均为 `0.1.7-rc.1`）
- 目标 profile：`web`（`~/.dsh/profiles\web`）
- 变更文件：`~/.dsh/profiles\web\cordis.patch.yml`（profile patch 层，唯一改动点）
- 内置 `standard` preset：**不覆盖、不修改**

---

## 1. Preset 格式判定（步骤 1 结论）

按运行时自省结果判定，**使用声明式 Preset（`@deepseek-ai/dsh-agent-preset` 行管理），不是旧版目录格式**。依据：

| 证据来源 | 结论 |
|---|---|
| `Config.listConfigs` 中 `@deepseek-ai/dsh-agent-preset` 命中 5 行：`preset-standard` / `preset-ptc` / `preset-minimal` / `preset-cordis` / `preset-computer-use` | Preset 是普通 Loader 插件行 |
| `preset-standard` 的 Config schema 为 `{id, name, description, order, plugins}`，`required: [id, plugins]` | 一行一个 Preset，子插件清单即 `plugins` |
| `@deepseek-ai/dsh-agent-preset-registry` README 原文：“Definitions are ordinary plugin rows; the registry **neither scans directories nor accepts preset paths**.” | 不存在目录扫描格式 |
| `dsh-web-app/presets/standard.patch.yml` 以 `- insert:` 引入 `id: preset-standard` | 内置基线同样是 patch 层 insert |

因此新 Preset 的正确落地方式 = 在 profile patch 层 `insert` 一行 `@deepseek-ai/dsh-agent-preset`，与内置基线的引入方式完全一致。

### 所选基线

以 `@deepseek-ai/dsh-web-app/presets/standard.patch.yml`（`config.id: standard`）为基线。该基线的 `config.plugins` 为 **24 项子行**，共同组成 Web standard Preset 的工具与提示能力。

### 基线清单逐项处置

| # | 基线子行 | 包 | 处置 | 理由 |
|---|---|---|---|---|
| 1 | `persona` | `dsh-persona` | **保留（改写 prefix）** | 本 Preset 的 persona 挂载点 |
| 2 | `agent-instructions` | `dsh-agent-instructions` | 移除 | 读取仓库 AGENTS.md，非本场景 |
| 3 | `tool-bash` | `dsh-tool-bash` | **移除** | 非基础文件读写 |
| 4 | `tool-pwsh` | `dsh-tool-pwsh` | **移除** | 同上（Windows 侧基线行） |
| 5 | `tool-fs` | `dsh-tool-fs` | **保留** | 基础文件读写（read/write/edit/glob） |
| 6 | `tool-fs-search` | `dsh-tool-fs-search` | **保留** | 基础文件检索（grep） |
| 7 | `tool-jobs` | `dsh-tool-jobs` | 移除 | 后台作业，非本场景 |
| 8 | `skill-filesystem` | `dsh-skill-filesystem` | 移除 | 技能体系，非基础能力 |
| 9 | `tool-skill` | `dsh-tool-skill` | 移除 | 同上 |
| 10 | `command-goal` | `dsh-command-goal` | **移除** | goal（明确要求） |
| 11 | `tool-goal` | `dsh-tool-goal` | **移除** | goal（明确要求） |
| 12 | `planning`（group，含 13 `plan-mode`） | `cordis:group` / `dsh-plan-mode` | **移除** | plan（明确要求） |
| 13 | `plan-mode` | `dsh-plan-mode` | **移除** | 同上 |
| 14 | `compaction`（group） | `cordis:group` | 移除 | 压缩机制，非基础能力 |
| 15 | `compaction-basic` | `dsh-compaction-basic` | 移除 | 同上 |
| 16 | `command-compact` | `dsh-command-compact` | 移除 | 同上 |
| 17 | `tool-result-pruner` | `dsh-compaction-tool-result-pruner` | 移除 | 同上（见“已知取舍”） |
| 18 | `delegation`（group） | `cordis:group` | **移除** | 子 Agent / workflow 容器（明确要求） |
| 19 | `tool-subagent-control` | `dsh-tool-subagent-control` | **移除** | 子 Agent（明确要求） |
| 20 | `tool-subagent-list-agents` | `dsh-tool-subagent-control/list-agents` | **移除** | 同上 |
| 21 | `tool-subagent` | `dsh-tool-subagent` | **移除** | 同上 |
| 22 | `tool-subagent-fork` | `dsh-tool-subagent` | **移除** | 同上 |
| 23 | `workflow-ptc` | `dsh-workflow-ptc` | **移除** | workflow（明确要求） |
| 24 | `tool-workflow` | `dsh-tool-workflow` | **移除** | 同上 |
| 25 | `tool-ask-user` | `dsh-tool-ask-user` | **保留** | 会话交互能力（边界例依赖它） |
| 26 | `tool-todo` | `dsh-tool-todo` | **保留** | 会话内任务跟踪，轻量 |
| 27 | `tool-web` | `dsh-tool-web` | **移除** | 浏览器 / 联网（明确要求） |
| 28 | `present` | `dsh-tool-present` | 移除 | 交付卡片机制，非基础能力 |

> 基线中 `tool-subagent-codex`、`tool-subagent-claude-code`、`tool-ralph`、`tool-plugin-manager` 等行本就 `disabled: true`，构造函数未生效，故不计入“保留”集合；新 Preset 不复制这些已禁用的行。

### 新 Preset 最终工具集（**最小重建版：2 行**）

> ⚠️ 本节已按故障修复更新。初版为 5 行（含 `tool-fs-search` / `tool-ask-user` / `tool-todo`）并在 UI 中显示「加载失败」，故按「最简单方式」重建为 2 行。详见第 8 节。

| 子行 id | 包 | 提供能力 |
|---|---|---|
| `persona` | `@deepseek-ai/dsh-persona` | 本会话专属 persona 提示词（prefix 逐字符不变） |
| `tool-fs` | `@deepseek-ai/dsh-tool-fs` | 基础文件读写：read / write / edit / glob |

净结果：**24 项 → 2 项**；移除浏览器、子 Agent、workflow、goal、plan，以及检索、提问、待办、shell、压缩等全部非基础能力。

**移除 `tool-ask-user` 的副作用**：初版含 `ask_user_question`，边界情形可主动追问。最小版移除了它，persona 文本中「使用 `ask_user_question` 工具」这条更依赖运行时容错；边界例行为验证是在**未挂载该工具**的环境中通过的（模型自动改用等价问题清单），因此该分支本身已验证，但真实工具调用路径未验证。

---

## 2. Persona 配置

```yaml
- id: persona
  name: '@deepseek-ai/dsh-persona'
  config:
    prefix: |-
      <见下节完整文本>
    complete: true
    includeRuntimeContext: false
```

字段与 `@deepseek-ai/dsh-persona` 运行时 schema 完全一致（已核对源码 `Config`）：

| 字段 | schema | 取值 |
|---|---|---|
| `prefix` | `z.string().required()` | 完整系统提示词（见下） |
| `suffix` | `z.string().default("")` | 省略（`complete: true` 下不生效） |
| `complete` | `z.boolean().default(false)` | `true` |
| `includeRuntimeContext` | `z.boolean().default(true)` | `false` |

**行为含义（已核对实现）**：`complete: true` 会把渲染后的 prefix 作为**唯一** system prompt 段落，抑制其他所有段落（identity / suffix / 工具指引 / 监听器不得追加）；`includeRuntimeContext: false` 调用 `ctx.systemPrompt.suppressRuntimeContext()`，本会话不再注入动态运行时上下文快照。

### ⚠️ 代拟声明（需你复核）

规格中 persona prefix 写的是占位符「`[把上面 ① 的完整系统提示词粘贴到这里]`」，而本会话上下文里**并不存在 ① 那段系统提示词**。经你确认，该 prefix **由我按「课堂纪要助手」定位代拟**，完整文本位于：

- Preset 配置内：`~/.dsh/profiles\web\cordis.patch.yml` 中 `preset-class-notes-organizer` 行的 `config.plugins[0].config.prefix`（用 `# --- PERSONA-PREFIX-BEGIN/END ---` 标记包裹）
- 独立副本：`prompts/persona-prefix.md`

该文本由我起草，**不是**你原始规格中的 ① 内容。若你随后提供真正的 ① 文本，替换该 `prefix` 值即可，其余配置无需改动。

### prefix 设计要点

1. 角色定位：课堂录音转写整理专家，输入是 ASR 纯文本，输出是结构化课堂纪要。
2. 处理规范：纠错（同音字/术语）、去口语冗余（嗯/那个/就是说）、保留原意。
3. 输出结构固定四段：课堂主题 / 核心内容（分点）/ 重点与考点 / 待办与作业。
4. 师生互动处理：合并问答为完整知识点，标注「师」「生」，不逐句复述。
5. “这个要考”信号：单独成「重点与考点」并用 **加粗** 标出，注明信号原话。
6. 边界处理：内容不足时明确说“信息不足、无法整理”，并用 `ask_user_question` 索要材料，**禁止编造**。
7. 明确禁止：不得臆造未出现的知识点、数字、人名、作业要求。

---

## 3. 已知取舍（写入即生效，请知悉）

1. **基线副本不自动合并**：registry README 明确“A user override replaces the complete child list and does not automatically merge future changes to the builtin list”。本 Preset 是 standard 基线的一份**独立副本**，日后 DSH 升级若改动内置 `standard`，本 Preset 不会自动跟随，需手工同步。
2. **无上下文压缩**：移除了 `compaction` 组与 `tool-result-pruner`，超长转写可能触及上下文上限。
3. **无 shell**：移除了 `tool-bash` / `tool-pwsh`，本 Preset 不能执行命令。
4. **complete 模式的副作用**：system prompt 只剩 persona 文本，工具清单仍通过 tool schema 通道提供给模型，但不再有额外的工具使用指引段落。

## 4. 影响面与回滚

- 影响面：仅新增一行 insert 条目，不改动任何既有行；内置 `standard` / `ptc` / `minimal` / `cordis` / `computer-use` 全部保持原样。
- 回滚：删除 `cordis.patch.yml` 中的 `insert: - id: preset-class-notes-organizer` 整块即可，无残留状态。
- 风险：修改 profile patch 会触发该 profile 的热重载（hmr）。若热重载后新 Preset 未出现，重启该 profile 即可。

---

## 5. 落地范围说明（重要）

规格指定「Web profile」，但勘察发现一个必须说明的事实：

| 事实 | 证据 |
|---|---|
| 当前 GUI（`http://127.0.0.1:19387`）实际加载的是 **desktop** profile | 该 host 进程的启动命令行为 `.../dsh-desktop-host/lib/index.js ... ~/.dsh/profiles/desktop ...`（PID 已隐去）；该进程同时监听 19387 端口 |
| 环境变量 `DSH_PROFILE` = `desktop` | `DSH_PROFILE_DIR=~/.dsh/profiles/desktop`（Windows 下即 `%USERPROFILE%\.dsh\profiles\desktop`） |
| 环境变量 `DSH_PROFILE` = `desktop` | `DSH_PROFILE_DIR=~/.dsh/profiles\desktop` |
| desktop 与 web 两个 profile 使用**同一套 bundle** | 两者 `package.json` 的 `dsh.profile.bundles` 均为 `@deepseek-ai/dsh-base` + `@deepseek-ai/dsh-web-app` |

因此，若只写入 web profile，你在当前 GUI 里将看不到新 Preset。为让 Preset 立即可用，**同一份声明同时写入了两个 profile**：

| profile | 文件 | 状态 |
|---|---|---|
| web（规格指定） | `~/.dsh/profiles\web\cordis.patch.yml` | 已写入，YAML 校验 ALL PASS |
| desktop（当前 GUI 实际使用） | `~/.dsh/profiles\desktop\cordis.patch.yml` | 已写入，YAML 校验 ALL PASS；既有 8 行 patch（含 dsh-skin managed 段）完好保留 |

两份定义经程序比对**完全一致**。若你只想保留 web profile，删除 desktop 那份 insert 块即可。

## 6. 校验证据

### 静态校验（`yaml` 包真解析，用 DSH 归档内自带的 `yaml` 实现，非自写解析器）

对两个 profile 的文件分别执行 14 项断言，结果均为 **ALL PASS**：

```
[PASS] YAML parses as loader entry array
[PASS] declaration row id correct          (preset-class-notes-organizer)
[PASS] plugin package name correct         (@deepseek-ai/dsh-agent-preset)
[PASS] config.id = class-notes-organizer
[PASS] config.name = 课堂纪要助手
[PASS] description matches spec
[PASS] plugins = 5
[PASS] persona row present and first
[PASS] persona package correct             (@deepseek-ai/dsh-persona)
[PASS] persona prefix non-empty            (1207 字符 / 47 行)
[PASS] persona complete = true
[PASS] persona includeRuntimeContext = false
[PASS] all referenced packages exist
[PASS] no existing row removed/renamed
```

### 基线与实现清单的程序化 diff

自动对比 `standard.patch.yml` 与本文声明，确认：

```
baseline standard: 19 top-level rows, 32 rows incl. group children
baseline effective rows : 25
kept in proposed        : 5
removed                 : 20
rows not present in baseline (invented): none
```

移除的 20 行与第 1 节处置表**逐条一致**（`agent-instructions`、`tool-bash`、`tool-pwsh`、`tool-jobs`、`skill-filesystem`、`tool-skill`、`command-goal`、`tool-goal`、`plan-mode`、`compaction-basic`、`command-compact`、`tool-result-pruner`、`tool-subagent-control`、`tool-subagent-list-agents`、`tool-subagent`、`tool-subagent-fork`、`workflow-ptc`、`tool-workflow`、`tool-web`、`present`）。

### ID 唯一性

```
builtin preset config.ids  : ["standard","ptc","minimal"]
builtin declaration row ids: ["preset-standard","preset-ptc","preset-minimal"]
[PASS] config.id "class-notes-organizer" 唯一
[PASS] 声明行 id "preset-class-notes-organizer" 唯一
[PASS] 无任何 patch 行指向 preset-standard（内置未被覆盖）
```

### 运行时证据（最强证据）

写入 desktop profile 后，**运行中的 host 已热重载**。通过 `Config.listConfigs` 查询 `@deepseek-ai/dsh-agent-preset`：

```
before: preset-standard, preset-ptc, preset-minimal, preset-cordis, preset-computer-use   (total: 5)
after : preset-standard, preset-ptc, preset-minimal, preset-cordis, preset-computer-use,
        include:preset-class-notes-organizer                                               (total: 6)
```

新增条目 `include:preset-class-notes-organizer` 状态为 `schema`（与内置 `preset-standard` 同级），`packageDir` 解析到 `@deepseek-ai/dsh-agent-preset`，投影 schema 为标准 PresetDefinition（`required: [id, plugins]`），`acceptsMissing: false`、`limitations: []`。

> 说明：`Config` 的行目录是**本次校验时 host 的实时状态**，确证该声明已被加载器的 preset 平面成功接纳，而非仅停留在文件层面。

## 7. 验证边界（未验证项，如实声明）

| 项 | 状态 | 原因 |
|---|---|---|
| 新 Preset 在 GUI 下拉框可选并**真正以该 Preset 启动一个 DSH 会话** | **未验证** | 当前进程本身就是 desktop profile 的 host；从既有会话内无法注入新会话。新 Preset 对 `agent/default` 的挂载只在新会话创建时发生 |
| persona 提示词的**实际模型行为** | **已验证**（见测试报告） | 用与配置逐字一致的 persona 文本跑 3 正例 + 1 边界例 |
| 本会话（standard）的工具集 | 保持 standard 不变 | 实时 `Tool.listTools` 显示仍为 standard 工具集；本会话早于新 Preset 创建，符合 registry「existing Agents retain the composition they already use」的设计 |

人工确认方式：在 GUI 新建会话，Preset 选择器选择「课堂纪要助手」，确认该会话只暴露文件读写工具。

---

## 8. 故障修复记录：「加载失败」（最小重建）

### 现象

初版 5 行工具集写入后，UI「设置 → Agent 预设」中该预设显示红色「加载失败」。

### 已排除的原因（逐项实测）

| # | 假设 | 验证方式 | 结论 |
|---|---|---|---|
| 1 | YAML 语法/缩进错误 | 用 DSH 归档内自带的 `yaml` 包真解析两个 profile 文件 | 排除：解析成功，`config.id`/`name`/`description`/`order`/`plugins` 全部正确 |
| 2 | 字段名写错（如误用 `tools`） | 与 `@deepseek-ai/dsh-agent-preset` 的运行时 Config schema 逐字段比对 | 排除：schema 为 `{id,name,description,order,plugins}`，`required:[id,plugins]`，完全匹配 |
| 3 | 引用了不存在的工具/包 | 列举 asar 内 `@deepseek-ai` 全部包名逐条核对 | 排除：5 个包子行对应的包全部真实存在 |
| 4 | 子插件依赖缺失 | 读取 5 个包的 `package.json` 的 `dependencies` / `peerDependencies`，并确认 `@vscode/ripgrep` 及 win32 二进制均在归档内 | 排除 |
| 5 | persona 字段写法不合规 | 与官方 `minimal.patch.yml` 的 personas 写法逐字段对照 | 排除：`prefix` + `complete: true` + `includeRuntimeContext: false` 与官方 minimal 完全一致 |
| 6 | 与 host 全局平面插件冲突 | 检查 `dsh-web-app/cordis.patch.yml` 是否已把 `tool-fs` / `tool-fs-search` / `tool-todo` 等声明为宿主平面行 | 排除：这些行在宿主平面均 `disabled: true`，不与 preset 平面冲突 |
| 7 | Preset ID / 声明行 ID 重复 | 枚举内置 5 个预设的 `config.id` 与声明行 id | 排除：均唯一；且无任何 patch 行指向 `preset-standard` |

### 失败发生在哪一层

`@deepseek-ai/dsh-agent-preset-registry` 的 `mountPreset()` 在挂载子插件后执行**激活审计**，并在两种情况抛错：

```js
if (audit.failed.length > 0) throw new Error(audit.failed.join("\n"));
if (leaked.length > 0) throw new Error(`Preset services require isolate realms: ${leaked.join(", ")}`);
```

其中 `auditRows` 会把**任何子行 fiber 未启动**（`never started`）或 `await` 拒绝、以及 `inject` 服务缺失的行收集为失败项，错误文本形如 `row-id (module-name): 原因`。该诊断经 `agentPresets.list()` 的 `broken` 字段暴露给客户端，UI 显示为预设卡片上的红字。

即：**失败不在静态配置层，而在子插件激活层**。上述 7 类静态原因均被排除后，剩余可能性集中在个别子插件的激活审计（`never started` / `inject` 服务缺失）。

### 处置：按「最简单方式」最小重建

已将两个 profile 的该 preset 块整体替换为最小定义（**persona + tool-fs**），程序化重建并逐字符校验：

```
captured prefix: 1207 chars, 47 lines
  plugins           : persona, tool-fs
  prefix identical  : YES (char-for-char)
  complete/runtime  : true / false
  preserved rows    : ui-chat, ui-settings, ui-settings-account, ui-settings-general,
                      agent-default-model, ui-skin-deep-whale-manager,
                      ui-skin-maid-atelier, ui-skin-orca-link
cross-profile definition identical: YES
```

两 profile 最终结构断言全部 PASS（`config.id` / `config.name` / `config.description` / `plugins` / `complete` / `includeRuntimeContext` / prefix 长度 1207）。

### 未能确证的部分（如实声明）

「加载失败」的**确切报错原文**没有取到：

- HTTP API 需要凭据（返回 401），未去获取凭据；
- 该诊断只存在于运行时，不写入任何日志文件（已检索 `.dsh` 与会话缓存）；
- 独立复现需要以 Node 加载 `app.asar` 内的 `profile-boot`，而 asar 穿透是 Electron 的能力，普通 Node 会报 `ERR_MODULE_NOT_FOUND`；
- `desktop` 是 CLI 保留名，无法用 `dsh --profile desktop` 旁路启动。

因此本次是**通过排除法定位到激活层并直接绕开**（换成最小工具集），而非读到报错原文后精确修复。若最小版仍失败，红字原文将直接指明是哪一行插件，可作为下一步的决定性输入。

### 生效条件

`agentPresets` 的预设定义在**启动时**读取。实测证据：文件已重建后，运行中 host 的 `Config.listConfigs` 仍返回旧结构（`status: schema`，无法反映新 `plugins` 列表）。故 **必须重启 DeepSeek Orb** 才能在 GUI 中看到重建后的预设。

