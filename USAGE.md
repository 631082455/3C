# 使用方式

本库是一个纯文档知识库，任何能读文件的 AI agent 都能加载。

## Claude Code（推荐）

### 方式 A：作为 skill 安装（自动触发）

把仓库 clone 到本地，然后在你的项目或全局 skill 目录建立链接/拷贝：

```
# 全局安装（所有项目可用）
%USERPROFILE%\.claude\skills\combat-3c-design\   ← 把仓库内容放进来（或 junction 链接）

# Windows junction 示例
mklink /J "%USERPROFILE%\.claude\skills\combat-3c-design" "C:\path\to\3C"
```

`SKILL.md` 顶部已含 frontmatter（name/description），Claude Code 会在对话涉及 3C 设计时自动触发。

### 方式 B：项目内直接引用

把仓库放在项目旁边，在项目的 `CLAUDE.md` 中加一行：

```markdown
涉及 3C（移动手感/镜头/操控/打击感）设计任务时，先读 <路径>/3C/SKILL.md 并按其流程执行。
```

## Cursor / 其他支持规则文件的工具

在规则文件（如 `.cursorrules`）中加入：

```
处理 3C 设计任务时，读取 3C/SKILL.md 并严格按其 6 阶段流程执行，
项目约定读 3C/conventions/ 下的非模板文件。
```

## 无文件系统访问的对话式工具（网页版 GPT / Claude 等）

按此顺序把文件内容粘贴进对话：

1. `SKILL.md`（流程）
2. 你填好的 `conventions/my-project.md`（项目约定）
3. 按 SKILL.md 索引表，把本次任务相关的 `references/*.md` 粘贴进去（不必全贴）

然后描述你的设计需求。

## 首次使用前：填写项目约定

**这一步不可跳过。** 复制 `conventions/TEMPLATE.md` 为 `conventions/<你的项目名>.md`，逐项填写。填写质量直接决定产出方案的落地程度——尤其是"网络模型"和"现有系统形态"两节。

不确定怎么填时参考 `conventions/example-conventions.md`（虚构 MOBA 项目《环山之战》的完整示例）。

## 私有知识（不进 git）

如果你想让 agent 参考公司内部资料（内部文档、竞品拆解、项目参数表），建 `knowledge-local/` 目录放进去——该目录已在 `.gitignore` 中，不会被提交。**切勿把内部资料放进 `conventions/` 或 `references/` 后 push 到公开仓库。**
