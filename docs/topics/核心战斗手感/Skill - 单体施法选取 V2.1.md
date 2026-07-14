# Skill — 单体施法选取 V2.1

> **本版相对 V2 的关键变化**:从"按下锁存"的 α 模式翻案为 **β 模式(Armed 跟手 + 候选评分实时驱动)**。
> 多个"红线/禁止行为"被软化或删除。ResolvePolicy 从 5 个缩成 2 个。UI 极简化为 3 态 + F2 L 括号。
>
> 关联决策记录:
> - [ADR-0001 — 单体技能默认不允许自动 fallback 自施法](../../../../docs/adr/0001-no-auto-self-fallback-by-default.md)
> - [ADR-0002 — Armed Target 由候选评分实时驱动(β 模式)](../../../../docs/adr/0002-armed-target-follows-candidate-scoring.md)
> - [ADR-0003 — 服务器目标验证采用 300ms 时间窗容忍](../../../../docs/adr/0003-server-validation-tolerance-window.md)
>
> 术语权威定义见 [`CONTEXT.md`](../../../../CONTEXT.md)

---

## 1. 背景与目标

面向第三人称 MOBA / Hero Shooter 的单体目标技能。系统持续按候选评分维护 Soft / Armed Target;
玩家通过瞄准(屏幕方向)和按键(技能键、Alt 修饰键、右键)显式表达意图,系统**不擅自决定**
自施法,但**允许**在 Holding 中按候选评分自然跟手切换 Armed。

### 核心规则
系统按候选评分**实时**维护 Soft/Armed Target;原 Armed 失效时按评分规则自然让位给次佳候选。
**不能自动切成自施法**(`AllowSelfFallback` 必须技能级显式 opt-in)。

---

## 2. 本版范围

### 本版要做
- 候选实时维护:空闲(且有驱动源时)和 Holding 期间持续按评分跑候选选取
- 武装目标 β 模式:技能按下进入 Holding 时初始化 Armed Target;Holding 期间随评分跟手跳转
- 释放结算:Release 时打到**当时**的 Armed Target;候选池空走 ResolvePolicy
- 评分稳定:LockRetainRadius / LockProtectTime / SwitchCooldown / Grace 加分等节流机制
- 技能级配置:不同技能通过 Profile 与 ResolvePolicy 表达差异
- UI 三态(Soft / Armed / SelfCast)+ F2 L 括号视觉
- 服务器 300ms 时间窗容忍验证

### 本版不做
- 不引入连续光束 / channeling 类技能(本版仅覆盖按住-蓄力-确认范式)
- 不引入目标循环按键(`Tab` 切换)—— 纯瞄准驱动
- 不覆盖范围技能、地面点选、锁敌相机、辅助瞄准完整方案
- 不引入 full lag compensation(rewind)—— 见 ADR-0003

---

## 3. 术语

| 术语 | 定义 | 使用边界 |
|------|------|----------|
| 候选目标 | 通过硬过滤的可参与评分 Actor | 进入评分前必须先过硬过滤 |
| 硬过滤 | 阵营、死亡、世界距离、ScreenCone、LOS、技能目标类型、可选中性 | 不满足直接出局,不参与评分 |
| **Soft Target** | **空闲(非 Holding)阶段**的当前评分胜出者 | UI 弱(灰白 L 括号);仅在有驱动源时才显示 |
| **Armed Target** | **Holding 阶段**的当前评分胜出者(随评分跟手,**不锁死**) | UI 强(队伍色 L 括号);Release 时打它 |
| 驱动源 | 英雄装备的单体技能中,被标记为"空闲驱动 Soft UI"的那一个;每英雄至多一个 | Editor 校验唯一;无驱动源 → 空闲不显示 |
| Grace 加分 | 候选短暂失效(出 LOS / 出范围 / 出 ScreenCone)时,在评分中保留较高加分一段时间 | 防单帧抖动;**不**用于"保留原 Armed 不变";死亡 / 阵营反转 / NotTargetable **不享受** |
| LockRetainRadius | 已锁定目标维持的评分加成半径(屏幕空间) | 防边界抖动 |
| LockProtectTime | 新锁定目标的短暂分数加成 | 防止刚锁上就被相邻目标抢走 |
| SwitchCooldown | 评分胜出者改变后的强制冷却 | 防 A/B 抖动;玩家显式甩枪(超 LockRetainRadius)可覆盖 |
| ResolvePolicy | Release 时候选池空的收口规则 | 仅 2 个:`NoCastOnEmpty` / `AllowSelfFallback` |
| SelfCast | Alt+键 触发的显式自施法,或 `AllowSelfFallback` 触发的回落自施法 | UI:自蓝 L 括号挂自己 |

---

## 4. 行为规则

### 阶段行为表

| 阶段 | 系统行为 |
|------|----------|
| 空闲 + 有驱动源 + 技能可用 | 20Hz 持续候选采集 + 评分;Soft Target = 评分胜出者;F2 灰白 L 括号挂在 Soft Target |
| 空闲 + 无驱动源 | **完全不算 Soft**;屏幕 0 框 |
| 空闲 + 驱动源 CD / 沉默 / 无蓝 | **不显示 Soft 框**(HUD 技能图标已表达不可用);Invalid 状态无独立 UI |
| 点按快放(InputMode=Tap) | 按下瞬间取当前候选胜出者 = Armed,立即 Resolve;不进入 Holding |
| 按下进 Holding | 用该技能 Profile 接管候选采集;Armed = 当前评分胜出者;UI 升级为队色 L 括号 |
| Holding 中目标失效 / 评分变化 | **Armed 自然跟手跳转到次佳候选**(死亡 / 失 LOS / 失范围 / 评分被甩超 LockRetainRadius)。F2 框瞬移到新 actor 身上(纯视觉,无音效) |
| Holding 中玩家**主动甩枪** | 等同于上一条;评分变化驱动,无特殊"玩家意图检测"分支 |
| Holding 中候选池**完全为空** | Armed 暂时无,F2 框不显示;Release 走 ResolvePolicy |
| Release(松开技能键) | 取**当时**的 Armed Target(或候选池空 → ResolvePolicy)。客户端打包成 EventData RPC 上报服务器 |
| Cancel(右键) | Holding 中止;**无消耗 / 无 CD / 无动画 / 无 UI 反馈**;F2 框消失 |
| Alt+技能键(SelfCast) | 跳过 Soft/Armed 评分;直接 cast 在自己身上;走 SelfCast UI |
| Alt+技能键 但该技能 `AllowedTargetTeam` 不包含 Self | **静默吞掉**;不响应、不报错、不给 HUD 提示 |

### 红线(收窄后)
- 系统**不得**自动切成自施法(必须 `AllowSelfFallback` 显式 opt-in,且 UI 必须显示"将自施法")
- 玩家**显式输入**(Alt+键、右键取消)优先级永远高于评分自动决策
- 评分自动决策**仅在当前评分变化时触发**,服务器不擅自替换玩家上报的 ArmedTarget

### 已删除的旧禁止行为(V2 → V2.1)
- ~~"目标短暂丢失时偷偷换成另一个目标"~~ —— β 下就是该跳
- ~~"Grace 结束后自动跳到下一个目标"~~ —— β 下 Grace 仅是评分加分,期满次佳自然上位
- ~~"用'最近友军/最近敌人'替代已锁存目标"~~ —— β 下次佳上位是正常机制
- ~~"目标失效时自动发明新目标"~~ —— β 下自然换次佳就是发明新目标

---

## 5. 运行流程

```
[采集候选(屏幕 + 范围内全部 Actor)]
        ↓
[硬过滤(阵营/死亡/距离/ScreenCone/LOS/Targetable/技能目标类型)]
        ↓
[分桶(英雄 > 召唤物 > 小兵)]
        ↓
[评分(屏幕中心 + 朝向 + 距离 + LockRetain 加分 + Grace 加分 + 粘滞 + 节流)]
        ↓
[胜出者 = Soft Target (空闲) 或 Armed Target (Holding)]
        ↓
[UI 框跟随胜出者 / Release 时打胜出者 / 候选池空 → ResolvePolicy]
```

整个流程在 `UTargetSelectionComponent`(挂 `AMatchPlayerController`)上以 **20Hz** 跑;
玩家按下技能时该技能的 Profile **替换**驱动源 Profile,Release/Cancel 后恢复。

---

## 6. 配置需求

### 6.1 TargetSelectionProfile

| 字段 | 用途 | 备注 |
|------|------|------|
| `AllowedTargetTeam` | 允许友军 / 敌人 / 自身 / 召唤物 / 建筑 / 中立 | 不满足硬过滤 |
| `AllowedUnitTypeTags` | 允许的单位类型(`TAG_UNIT_HERO` / `_MINION` / 等) | 不满足硬过滤 |
| `Range` | 世界距离上限(米) | 硬过滤 |
| `ScreenCone` | 屏幕半径百分比(0.0-1.0,典型 0.25) | **2D 屏幕空间**(NDC),非 3D 锥角 |
| `LineOfSightRule` | None / Required / RequiredWithGrace | 默认 RequiredWithGrace(`LostGraceTime` 内允许) |
| `PriorityBuckets` | 单位类型分桶 + 桶间优先级 | 英雄桶非空时,小兵桶即使分数高也不胜出 |
| `ScoreWeights` | 屏幕中心距离 / 朝向 / 世界距离 / 范围余量 / 当前目标粘滞 | 实际数值需 playtest 调 |
| `LockRetainRadius` | 已锁定目标的评分加成半径(屏幕%) | 默认 0.15 |
| `LockProtectTime` | 新锁定后的短暂加成时长 | 默认 0.15-0.30s |
| `LostGraceTime` | 候选短暂失效的 Grace 加分时长 | 默认 0.10-0.30s |
| `SwitchCooldown` | 评分胜出者改变后的强制冷却 | 默认 0.10-0.15s |
| `ResolvePolicy` | `NoCastOnEmpty` / `AllowSelfFallback` | 默认 `NoCastOnEmpty` |

### 6.2 技能卡新增字段(`UHeroAbilityDataAsset`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `TargetSelectionProfile` | `TObjectPtr<UTargetSelectionProfileDataAsset>` | 引用 Profile(空 = 不走单体管线) |
| `bDrivesIdleSoftTarget` | bool | 是否驱动空闲 Soft UI(每英雄至多一个 true,editor 校验) |
| `ResolvePolicy` | enum | `NoCastOnEmpty` / `AllowSelfFallback` |
| `bSelfCastEnabled` | bool | Alt+键 是否对此技能生效(默认 true,纯敌方技能可关) |

`SelfCastInput` 全局机制(Alt+键)写在输入数据资产,**不**在每个技能卡重复配置。

---

## 7. ResolvePolicy

仅 2 个,β 模式下"目标失效"实际只剩**候选池完全为空**这一种情况。

### `NoCastOnEmpty`(默认)
候选池空 + Release → **啥也不发生**:
- 无资源消耗
- 不进入冷却
- 不播施法动画
- 不触发 SkillModule
- 无 UI / 无音效反馈
- 等价于"按键失效,系统当你没按过"

### `AllowSelfFallback`(opt-in,标记在技能卡)
候选池空 + Release → cast 在自己身上:
- 走 SelfCast UI(自蓝 L 括号挂自己)
- 正常消耗资源 / 进 CD / 播动画 / 触发 SkillModule
- 适用:防御性资源型友军技能(治疗 / 护盾)的明确设计意图
- 必须配合 ADR-0001 的约束(自施法不能默认,opt-in 必须显式)

### V2 → V2.1 删除的 policy
- `FailOnInvalid` → 合并到 `NoCastOnEmpty`(消耗 / CD / 动画 / 反馈全去掉)
- `CancelCast` → 跟 `NoCastOnEmpty` 等价
- `RevalidateSameTarget` → β 下"原 Armed"概念在 Holding 中不持续
- `ExplicitSelfCastOnly` → Alt+键全局机制,不需要 policy 字段表达

---

## 8. UI 与网络

### 8.1 UI 三态 + F2 L 括号

**统一形态**:四角 L 形括号(粗 2px),贴 actor 在屏幕上的投影包围盒(远小近大)。
**同一时刻只有一个 L 括号显示**(挂在当前 Soft/Armed/SelfCast 对应的 actor 身上)。

| 态 | 触发 | 颜色 | 几何 |
|---|---|---|---|
| Soft | 空闲 + 有驱动源 + 技能可用 | 灰白 | 四角 L 括号 + 描边 |
| Armed | Holding 中 | 队伍色(友绿 / 敌红 / 自蓝) | 四角 L 括号 + 描边 |
| SelfCast | Alt+键触发 或 `AllowSelfFallback` 触发 | 自蓝 | L 括号挂自己角色身上(复用 F2 形态) |

**Invalid 不存在独立 UI 态**:技能 CD / 沉默 / 无蓝时,Soft 框直接不显示。
HUD 技能图标已表达不可用状态。

**β 跳转视觉信号**:L 括号瞬移(从 A actor 消失,在 B actor 出现) —— 框的位置变化即 motion event,
玩家眼睛即使在战场中央也能感知。**不加跳转音效**,纯靠视觉。

**屏幕中心准星**:跟本系统**完全无关**。准星只服务普攻/枪械,不随 Soft/Armed 状态变色或变形;
单体目标状态完全靠 F2 L 括号传达,准星是独立信道。

**ScreenCone 边界**:**不画出**(V1 模式)。如未来 playtest 反馈"新人不懂 Armed 跳"再启用
V3 模式(仅 Holding 时画淡边界)。

**Release 失败 / 右键取消 / 跳转 / 技能名 label**:**全部无 UI 反馈**(极简风格)。

### 8.2 网络与权威

| 事项 | 要求 |
|------|------|
| 客户端 | 20Hz 跑候选评分,预测 Soft/Armed UI;Release/Confirm 时打包 `Armed Target` 进 `FGameplayEventData` 走现有 `ServerConfirmHoldingWithTimeStamp` RPC |
| 服务器 | 验证 Armed Target 合法性,采用 **300ms 时间窗容忍模式**(见 ADR-0003) |
| 容忍项 | Dead / OutOfRange / NoLOS / OutOfScreenCone(300ms 内合法过即放行) |
| 不容忍项 | InvalidTeam / NotTargetable / 技能 CD / 沉默 / 蓝量(永远按服务器当前判定) |
| 容忍允许但执行时已失效 | **静默 fail**;客户端不显示失败反馈;玩家感知"我以为打到了"但不被打脸 |
| 反作弊 | `ClientTime` 须做 sanity check(`abs(ClientTime - ServerTime) < ping * 2`),否则拒收 |
| 回放 / 观战 | 记录最终决议目标、ResolvePolicy 触发(若有)、自施法标记 |

**不做** full lag compensation(rewind 世界快照),原因见 ADR-0003。

---

## 9. 技能示例:Muriel Shield

| 项 | 配置 |
|---|---|
| `TargetSelectionProfile.AllowedTargetTeam` | FriendlyHero (+ FriendlySummon 可选) |
| `TargetSelectionProfile.Range` | 12 米 |
| `TargetSelectionProfile.ScreenCone` | 0.30(屏幕中心半径 30%) |
| `TargetSelectionProfile.LineOfSightRule` | RequiredWithGrace |
| `TargetSelectionProfile.LostGraceTime` | 0.20s |
| `TargetSelectionProfile.SwitchCooldown` | 0.12s |
| `bDrivesIdleSoftTarget` | true(每英雄至多一个) |
| `InputMode` | Hold(按住蓄力) |
| `ResolvePolicy` | `AllowSelfFallback`(空旷场地自动给自己) |
| `bSelfCastEnabled` | true(Alt+Q 强制给自己) |

**典型流程**:
1. Muriel 走动时,屏幕中央友军(Carry)身上自动出现灰白 L 括号(Soft)
2. 按下 Q,框升级为绿色 L 括号(Armed = Carry)
3. Carry 拐过墙,Armed 自然跳到旁边友军 B(框瞬移到 B 身上)
4. 松开 Q,护盾打到 B 身上
5. 或:按下 Q 后玩家发现都没人,右键取消(无消耗)
6. 或:空旷场地按下 Q 后松手 → `AllowSelfFallback` 触发,框自蓝色出现在自己脚下 + 自上盾

---

## 10. 验收

| 优先级 | 验收项 | 通过标准 |
|--------|--------|----------|
| **P0** | β 跟手 | Holding 期间 Armed 跟随当前评分胜出者跳转;原目标死/出 LOS/出范围 → 自然让位次佳 |
| **P0** | 候选池空收口 | 候选池完全为空时 Release → 走 ResolvePolicy(`NoCastOnEmpty` 默认无任何反馈;`AllowSelfFallback` 走 SelfCast UI) |
| **P0** | 自施法不擅自 | 没标 `AllowSelfFallback` 的技能,候选池空时 Release 绝不自动 cast 自己 |
| **P0** | UI 三态正确 | Soft(灰白) / Armed(队色) / SelfCast(自蓝);Invalid 不显示框 |
| **P0** | 服务器容忍 | 跨服 ping 150 玩家可正常释放刚死亡 < 300ms 的目标;超 300ms 失败;静默 fail |
| **P0** | 输入优先级 | Alt+键 / 右键取消优先于评分自动决策 |
| **P1** | Profile 数据化 | 至少两个技能复用同一 Profile,仅改技能卡引用 |
| **P1** | Editor 校验 | 同英雄 2 个 `bDrivesIdleSoftTarget=true` 报错;选 `AreaPreviewTargetOrSelf` 不可见 |
| **P1** | Debug 可解释 | Debug 命令能看到候选列表、过滤原因、评分明细、当前 Soft/Armed |

---

## 11. 程序备注

### 实现层次(策划视角的 5 个功能模块)

1. **M1 — 目标自动框选 + 显示**:空闲时框跟着我看的人走
2. **M2 — 完整单体施法闭环**:按下蓄力 + 跟手 + Release + 右键取消
3. **M3 — 自施法 + 防御性技能 fallback**:Alt+键 + `AllowSelfFallback`
4. **M4 — 联机 + 高 ping 容忍**:服务器 300ms 容忍窗
5. **M5 — Editor 规范 + 遗留清理**:驱动源唯一性 + `AreaPreviewTargetOrSelf` 标 Hidden

### 数据/状态拆分(初版)

```
UTargetSelectionProfileDataAsset
  AllowedTargetTeam / AllowedUnitTypeTags
  Range / ScreenCone / LineOfSightRule
  PriorityBuckets / ScoreWeights
  LockRetainRadius / LockProtectTime / LostGraceTime / SwitchCooldown
  ResolvePolicy

UTargetSelectionComponent(挂 AMatchPlayerController)
  ActiveProfile(当前驱动 Profile,空闲=驱动源,Holding=该技能 Profile)
  CandidateTargets(20Hz 更新)
  CurrentTarget(Soft 或 Armed,根据状态机阶段区分)
  StateMachine(Idle / Holding / SelfCast)
  LastSwitchTime / LastLockTime(节流)
  GraceBuffer(每候选最近 300ms 的失效时刻)
```

### 约束

| 约束 | 说明 |
|------|------|
| 统一入口 | 目标搜索、评分、Holding 状态机、Release 决议都走 `UTargetSelectionComponent`;技能蓝图不重复实现 |
| 技能只给配置 | 通过 `TargetSelectionProfile` + 技能卡字段表达差异;不写 C++ 找目标 |
| 失败可追踪 | Debug 命令至少包含:候选列表 / 各候选过滤通过情况 / 各候选评分明细 / 当前 Soft/Armed / 最近一次跳转事件 |
| 手动优先 | Alt+键 / 右键取消 / 玩家显式甩枪(超 LockRetainRadius)优先级高于自动评分 |
| 反作弊 | ClientTime sanity check;不容忍项严格 server-now 判定 |

---

**版本**:V2.1
**日期**:2026/05/21
**变更原因**:V2 grilling 后的 β 翻案 + UI 极简化 + 服务器策略明确
**关联 ADR**:0001 / 0002 / 0003

---

## 12. 实现状态(V2.1 落地后)

整套 M1-M5 已在项目里实现,以下是 **设计稿 → 实际代码 / 资产** 的索引,供后续 designer / 工程师 查阅或对其他英雄复制扩展。

### 12.1 代码位置

| 模块 | 类 / 文件 |
|---|---|
| Profile 数据资产 | `Source/GameMoba/Targeting/TargetSelectionProfileDataAsset.h` |
| 主组件(挂 PC) | `Source/GameMoba/Targeting/TargetSelectionComponent.h/.cpp` |
| 服务器验证 | `Source/GameMoba/Targeting/TargetingValidator.h/.cpp`(挂在 `MobaUnitInterface::MobaUnitDead` 记录死亡时刻) |
| 单体目标读取入口 | `EApplyBuffTargetType::ArmedTarget`(SkillModule 配置时选这个) |
| 输入字段 | `GameInputDataAsset::CancelHoldingInputAction` + `SelfCastModifierInputAction` |
| 自施法/取消 handler | `AMatchPlayerController::Input_CancelHolding/Input_SelfCastModifier` |
| Ability 钩子 | `UHeroAbility::ActivateAbility/EndAbility` 中 BeginHolding/BeginSelfCast/EndHolding/EndSelfCast |
| Editor 校验 | `UHeroDataAsset::IsDataValid` 检查驱动源唯一性 |

### 12.2 配置入口(designer 在哪改)

| 改什么 | 在哪 |
|---|---|
| 给某技能加单体管线 | `UHeroAbilityDataAsset` 上设 `TargetSelectionProfile`(选一个 `UTargetSelectionProfileDataAsset` 资产) |
| 让某技能作为英雄空闲驱动源 | 同上 + 勾 `bDrivesIdleSoftTarget`(**每英雄至多一个**,Editor 自动校验) |
| 让 SkillModule 实际打到 ArmedTarget | `SkillModuleApplyBuffToTarget.TargetType = ArmedTarget` |
| 防御性技能开启自动给自己 | Profile 的 `ResolvePolicy = AllowSelfFallback` |
| 单体目标 UI widget(L 括号样式) | Profile 的 `TargetIndicatorWidgetClass`(可指任意 UMG widget) |
| 自施法 modifier 键位 | `GameInputData_Global.SelfCastModifierInputAction`(默认 Alt) |
| 取消 Holding 键位 | `GameInputData_Global.CancelHoldingInputAction`(默认右键) |

### 12.3 已知简化 / 后续 polish

| 项 | 现状 | 后续 |
|---|---|---|
| Production widget 样式 | 复用 `W_MarkTargetIndicator`(原 SkillAreaPreview 用的标记)同款,挂自蓝/友绿/敌红色按 Component State 不同(实际颜色由 widget 蓝图内部读 owner 自定) | 美术做 V2.1 设计稿描述的 F2 L 括号 + 跳转动画 |
| Widget DrawSize | 当前硬编码 256x256(Component cpp 内) | 后续应做成 Profile 字段,designer 调 |
| 服务器 ring buffer | 只追踪"最近死亡时刻"单点 timestamp,不是完整 N-state ring buffer | 如果未来出现其他需要回溯的失效原因(如 NotTargetable 期窗口),扩展 `FTargetingValidator` |
| Debug 可视化 | `Targeting.DebugDraw 1` 控制台开关,画 wireframe box(非 Shipping 才可用) | Production 不需要,内部 debug 用 |
| 驱动源 discovery 时序 | Tick 轮询 owner pawn 变化 + GAS abilities;abilities 未 grant 时持续 retry 直到找到 | 可优化为 GAS 事件订阅(OnAbilityGiven) |

### 12.4 Muriel Shield 测试配置(可作他英雄模板)

| 资产 | 用途 |
|---|---|
| `/Game/Heroes/Muriel/Shield/TSP_MurielShield` | Profile 实例(Friendly + 12m + 0.30 cone + AllowSelfFallback) |
| `/Game/Heroes/Muriel/Shield/HAD_MurielShield_V21Test` | 技能 DataAsset(从原 HAD_MurielShield 复制 + 加 V2.1 字段;`ConfirmHoldingSkillModules[1].TargetType = ArmedTarget`) |
| `/Game/Heroes/Muriel/Shield/GA_MurielShield_V21Test` | Ability 蓝图(从原 GA_MurielShield 复制,`AbilityDataAsset` 指向 HAD_V21Test) |
| `/Game/Heroes/Muriel/HeroData_Muriel` | Abilities[2] 替换为 GA_MurielShield_V21Test_C |
| `/Game/Heroes/Kallari/Mark/W_SoftTargetIndicator` | 单体目标指示 widget(复制自 W_MarkTargetIndicator) |
| `/Game/Input/Actions/IA_CancelHolding` + `IA_SelfCastModifier` | 输入资产(右键 / Alt) |
| `/Game/GlobalSettings/GameInputData_Global` | 全局输入配置,已填这两个 IA |

### 12.5 给别的英雄做单体技能的最少工作量

1. 新建 Profile 资产 `TSP_<HeroName>_<SkillName>`(可直接复制 Muriel 的改差异字段)
2. 在英雄技能的 HAD 上:`TargetSelectionProfile = TSP_...`,需要空闲驱动就 `bDrivesIdleSoftTarget = true`
3. 在该技能的 `ConfirmHolding/InputPress/InputRelease SkillModules` 中,把 `SkillModuleApplyBuffToTarget.TargetType` 改成 `ArmedTarget`(原本可能是 `AreaPreviewTarget`)
4. 完成。**不需要碰 C++ 代码**

---
