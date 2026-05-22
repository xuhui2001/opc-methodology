# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在此仓库中工作时提供指引。

## 仓库简介

本仓库包含两个相关内容：

1. **一本 mdBook**（`src/`）——约六万字的中文书籍《一人企业方法论》v2.1，使用 [mdBook](https://rust-lang.github.io/mdBook/) 构建。

2. **一套 AI Agent 技能集**（`skills/`）——基于 Codex/OpenAI 的 Agent 技能定义，将同一套方法论实现为可交互的引导式工作流，每个技能对应 OPC 流程的一个阶段。

## 构建命令

```bash
# 构建 mdBook（HTML 输出 → book/）
mdbook build

# 构建 EPUB
mdbook-epub --standalone true   # 输出到 book/ 目录

# 统计 src/*.md 文件的总字符数
./words.sh
```

本仓库没有测试套件，也没有 linter。唯一的"构建产物"是书籍输出文件。

## 仓库结构

```
book.toml              # mdBook 配置；源目录为 src/，语言为 zh-cn
src/                   # 书籍章节（中文 markdown）
  SUMMARY.md           # mdBook 目录——定义章节顺序
  *.md                 # 书籍章节
  images/              # 章节中引用的图片
skills/                # Agent 技能定义（每个技能一个目录）
  opc-orchestrator/    # 总编排——整套流程的入口
  opc-resource-audit/  # 阶段 01
  opc-niche-positioning/  # 阶段 02
  opc-value-proposition/  # 阶段 03
  opc-business-model-design/  # 阶段 04
  opc-mvp-designer/    # 阶段 06
  opc-conversion-loop/ # 阶段 07
  opc-asset-ops/       # 阶段 08（可重复触发）
  opc-dashboard-review/# 阶段 09（可重复触发）
words.sh               # 字符统计工具
```

每个技能目录的结构如下：
```
skills/<skill-name>/
  SKILL.md             # 完整技能说明（权威规范）
  agents/openai.yaml   # 接口元数据：display_name、short_description、default_prompt
  references/          # 技能运行时读取的参考文档
```

## 技能架构

### 各技能的协作方式

技能集实现了两阶段生命周期：

**建盘期（一次性、线性，阶段 01–07）：**
```
01 opc-resource-audit
  → 02 opc-niche-positioning
    → 03 opc-value-proposition
      → 04 opc-business-model-design
        → 06 opc-mvp-designer
          → 07 opc-conversion-loop
```
每个阶段必须完成并写入产物后，下一阶段才能开始。阶段 05（机会评分）已合并到阶段 02 中完成。

**运营循环（按需触发，阶段 08–09）：**
- `opc-asset-ops`（08）：当出现可复用的重复产物、需要系统化沉淀时触发
- `opc-dashboard-review`（09）：当运营遇到瓶颈或需要周期性复盘时触发

两个循环阶段均可多次触发；其输出文件使用带日期的文件名（格式 `YYYYMMDD`），以保留历史记录。

### 共享文件契约（`opc-doc/`）

所有技能均从**用户当前工作目录**下的 `opc-doc/` 读写数据（不在本仓库内）。目录结构如下：

```
opc-doc/
  inputs/                    # 用户可选提供的上下文文件
  state/
    current-stage.json       # 当前所在阶段及状态
    decisions.json           # 已确认的关键决策记录
    assumptions.json         # 待验证假设
    user-preferences.json    # 交互模式、术语偏好
  outputs/
    00-orchestrator/         # 会话摘要
    01-resource-audit/       # inventory.md, scorecard.json
    02-niche-positioning/    # three-ring-analysis.md, candidates.md, target-segment.json, positioning-statement.md
    03-value-proposition/    # value-proposition-canvas.md, segment-vp-matrix.md, messaging.md
    04-business-model/       # lean-canvas.md, business-model-canvas-lite.md, pricing-notes.md, risky-assumptions.md
    06-mvp-design/           # mvp-spec.md, experiment-plan.md, human-ai-split.md
    07-conversion-loop/      # channel-strategy.md, content-plan.md, conversion-path.md
    08-asset-ops/            # asset-inventory-[date].md, action-plan-[date].md
    09-dashboard-review/     # review-[date].md
  reviews/
    dashboard.json           # 只追加写入的历史复盘数据
```

`opc-doc/` 不需要预先创建。技能会在需要时按需创建。若目录不存在，技能视为全新开始。

### 交互协议（适用于所有技能）

每个技能均遵循 `skills/opc-orchestrator/references/interaction-protocol.md` 中定义的交互规则：

- **对话先行，文件其后**：在写入任何文件之前，先在对话中呈现选项并获得用户确认。在对话中描述结论不等于落盘。
- **默认一次只问一个问题**；仅当几个问题都很轻且紧密相关时，可合并为 2–3 个。
- **关键决策始终提供 3 个选项 + "4. 我有自己的方案"**。只给分析，不给推荐结论。
- **阶段边界管控**：每个技能只处理本阶段的内容。若话题属于后续阶段，须明确说明并延后处理。
- **轮次限制**：每个阶段应在第 10 轮前收口。第 5–8 轮进入收尾阶段；第 9 轮强制总结；第 10 轮默认完成。
- **会话恢复**：每次新会话开始时，在提问之前先读取 `opc-doc/state/current-stage.json` 和 `decisions.json`，向用户展示进度摘要并询问是否继续。

### 新增或修改技能

1. 技能行为的权威规范是其 `SKILL.md`。
2. `agents/openai.yaml` 只提供展示元数据（`display_name`、`short_description`、`default_prompt`），不包含逻辑。
3. `references/` 中的参考文档由技能在运行时读取——保持内容准确、稳定。
4. 新增阶段时，分配一个未使用的阶段编号，并将其添加到 `skills/opc-orchestrator/references/stage-map.md`。
5. 若新阶段影响编排顺序或循环触发条件，须更新 `skills/opc-orchestrator/SKILL.md`。

### 新增书籍内容

书籍章节必须在 `src/SUMMARY.md` 中注册才会出现在构建输出中。`SUMMARY.md` 中的标题和章节层级决定 mdBook 的导航结构。封面图片在 `book.toml` 的 `[output.epub]` 部分设置。
