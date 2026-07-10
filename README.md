# 3C Combat Design Workflow（战斗 3C 设计工作流）

一个面向**动作/MOBA/射击等实时战斗游戏**的 3C（Character / Control / Camera）设计方法论知识库，设计为可被任意 AI agent（Claude Code、Cursor、GPT 等）加载使用的 skill。

核心理念：**项目无关**。本库不预设任何一套 3C 体系——镜头类型（锁定/自由/固定俯角）、输入设备、网络模型、引擎，全部由使用者在 `conventions/` 中自行定义。知识库只提供方法论、参数化框架、质量标尺与避坑规则。

## 这个库能做什么

对 AI agent 说一句：

> "为 XXX 英雄的位移技能出一套 3C 方案"
> "设计一套第三人称战斗镜头的跟随与动效规则"
> "帮我审计当前移动手感，产出调参建议"

agent 会按 `SKILL.md` 中的 6 阶段流程走完：**读项目约定 → 定体验目标 → 拆解三轴机制 → 参数化（预设+可调范围） → 边界与冲突审计 → 产出审阅级设计稿**。

最终产出物是 **3C 方案设计稿**：一份含体验目标、机制说明、完整参数表（默认值/预设/可调范围）、生命周期定义、验收标准与程序/美术需求清单的文档，可直接进评审。

## 目录结构

```
3C/
├─ README.md                  # 本文件
├─ SKILL.md                   # 方法论入口：6 阶段流程 + 知识库索引（agent 主要读这个）
├─ USAGE.md                   # 如何在各 AI 工具中加载
├─ LICENSE                    # MIT
├─ conventions/               # 项目约定（使用者填写）
│  ├─ TEMPLATE.md             #   留空模板：定义你的项目 3C 约定
│  └─ example-conventions.md  #   示例：虚构 MOBA 项目《环山之战》
├─ references/                # 知识库主体
│  ├─ movement-feel.md        #   移动与手感：速度模型、转向、位移技能
│  ├─ camera-design.md        #   镜头：构图、跟随、动效层、优先级仲裁、防晕
│  ├─ control-input.md        #   操控：输入缓冲、取消窗口、目标选择、预览/执行
│  ├─ hit-feel.md             #   打击感：时序分解、顿帧/震屏/受击反馈
│  ├─ netcode-3c.md           #   网络预测约束下的 3C 设计规则
│  ├─ meta-rules.md           #   元规则：跨系统通用的设计铁律
│  ├─ anti-patterns.md        #   反模式自检清单
│  ├─ quality-rubric.md       #   5 维质量标尺（方案评分用）
│  └─ case-library.md         #   公开游戏 3C 案例库
├─ templates/
│  ├─ blank-3c-design-doc.md  #   3C 方案设计稿模板（8 章）
│  └─ blank-param-sheet.md    #   参数表模板（预设 + 可调范围格式）
└─ output/                    # 产出区
   ├─ README.md               #   产出规范与命名规则
   └─ examples/
      └─ blink-dash-3c.md     #   样板：位移技能 3C 方案完整稿
```

## 快速开始

1. 读 `USAGE.md`，把本库接入你的 AI 工具。
2. 复制 `conventions/TEMPLATE.md` 为 `conventions/my-project.md`，填写你的项目约定（参考 `example-conventions.md`）。
3. 对 agent 说"为 XXX 出 3C 方案 / 审计 XXX 手感"。
4. 产出物对照 `output/examples/` 中的样板检查规格。

## 设计立场（为什么这个库长这样）

本库由一线战斗 3C 策划维护，方法论以**策划工作流**为第一评判标准：

- **参数必须带预设**。给策划的配置体验应该是"拿预设改 1~2 个数"，而不是面对 40 个原子参数从零调起。所有参数表都要求"默认值 + 预设档位 + 可调范围"三列齐全。
- **每个持续性效果必须定义完整生命周期**。施加、叠加、刷新、移除、死亡、重生——缺一条就是未来的 bug。
- **冲突必须显式仲裁**。多个系统同时想控制镜头/速度/输入时，用优先级带（band）仲裁，禁止 last-wins。
- **表现与逻辑分层**。伤害、位移等结算逻辑放权威层，表现层（Cue/特效/镜头）只做表现。

这些立场的完整展开见 `references/meta-rules.md`。

## 版权与引用说明

本库为个人方法论沉淀，MIT 许可。文中引用的商业游戏（见 `references/case-library.md`）均属研究/评论性引用，仅作设计参考，不包含任何受版权保护的资产或未公开信息。
