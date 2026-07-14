# V2.1 单体施法选取 — 代码交接(给主程 review)

> 配套 spec:`Skill - 单体施法选取 V2.1.md/.html`
> 关联 ADR:`docs/adr/0001` `0002` `0003`
> 术语索引:仓库根 `CONTEXT.md`

---

## TL;DR

引入**单体目标管线**(独立于已有 `SkillAreaPreview` 范围管线,**并列共存,不替代**),
覆盖空闲软锁定 → Holding 跟手 → Release 决议 → 服务器验证 → Alt 自施法 → 右键取消的完整闭环。
已实现 M1-M5 全部功能,通过本地 PIE 验证。**联机环境未测**(H6)。

**Submit 前 review 重点**:`HeroAbility.cpp` 里的 SelfCast 路径 + GAS 时序、`UWorldSubsystem` 的 lifecycle、`MobaUnitInterface.cpp` 加的 hook 是否符合项目惯例。

---

## 风险清单(按 review 优先级)

| 等级 | 项 | 文件 / 行 | 处理状态 |
|---|---|---|---|
| 🔴 Blocker (已 fix) | server-side PC 不该跑评分 | `TargetSelectionComponent.cpp` TickComponent 开头加 `IsLocalController` guard | ✅ |
| 🔴 Blocker (已 fix) | TargetingValidator 全局 static map → PIE 多 world 串台 | 重构为 `UTargetingValidatorSubsystem`(UWorldSubsystem) | ✅ |
| 🔴 Blocker (已 fix) | `HeroAbility::ActivateAbility` 内部直接调 `ConfirmHoldingAbility` 跟 GAS 时序打架 | 推到 `SetTimerForNextTick` | ✅ |
| 🟠 High | `MobaUnitInterface::MobaUnitDead` 加 `RecordDeath` 调用 — 侵入核心 interface | 等后续抽 OnUnitDied delegate 重构 | 接受 |
| 🟠 High | `SkillModuleApplyBuffToTarget::Execute` 服务器验证特殊路径埋在通用模块 | 等加新 enum 值时一并抽出 | 接受 |
| 🟠 High | `WidgetComponent.Owner = PreviewHelperActor`,若它 destroy widget 会 dangling | TWeakObjectPtr 替代 + IsValid 防御 | 待跟进 |
| 🟠 High | `CanActivateAbility` 每 tick 调用(20-60Hz) | 监听 OnAbilityCooldownChanged 事件缓存 | 待跟进 |
| 🟠 High | `GetAllMobaUnits` per LOS trace 重复查 | 在 GatherCandidates 入口缓存一次 | 待跟进 |
| 🟠 High | 联机场景未测(M4 服务器 300ms 容忍) | DS + 模拟 ping 验证 race 场景 | 待测 |
| 🟡 Medium | `SetDrawSize(256,256)` 硬编码 | Profile 加 DrawSize 字段 | M5.2 polish |
| 🟡 Medium | `IdleDriveAbility` discovery 每 tick 跑 | OnGiveAbility 事件订阅 | 待跟进 |
| 🟡 Medium | Profile 15+ 字段平铺 | 拆嵌套 struct(FScoreWeights/FStickinessConfig)| 待跟进 |
| 🟡 Medium | 命名混乱:`bDrivesIdleSoftTarget` / `IdleDriveProfile` / `驱动源` 三个名一概念 | 统一 IdleDriver 命名 | 待跟进 |

---

## 改动总览

| 类型 | 数量 |
|---|---|
| 新增 .h/.cpp(GameMoba 模块) | 5 |
| 修改 .h/.cpp(GameMoba 模块) | 9 |
| 新增 Content 资产(测试配置) | 6 |
| 修改 Content 资产 | 4 |
| Design 仓新增文档 | 6(本文 + CONTEXT.md + 3 ADR + V2.1 .md/.html) |

---

## 文件清单(diff 索引)

### 新增 Source(`p4 add`)

| 文件 | 行数 | 职责 |
|---|---:|---|
| `Source/GameMoba/Targeting/TargetSelectionProfileDataAsset.h` | ~120 | `UDataAsset` 子类。所有候选采集/评分/节流字段配置。15+ UPROPERTY |
| `Source/GameMoba/Targeting/TargetSelectionComponent.h` | ~110 | `UActorComponent`,挂在 `AMatchPlayerController`(本地 PC),20Hz/60Hz 自适应 tick |
| `Source/GameMoba/Targeting/TargetSelectionComponent.cpp` | ~470 | 状态机(Idle/Holding/SelfCast)+ GatherCandidates + ScoreAndSelectBest + UpdateTargetIndicatorWidget |
| `Source/GameMoba/Targeting/TargetingValidator.h` | ~55 | `UWorldSubsystem`,服务器侧 Dead/Team 验证 + 300ms 容忍 |
| `Source/GameMoba/Targeting/TargetingValidator.cpp` | ~100 | RecentDeathMap + ValidateTargetOnServer 实现 |

### 修改 Source(`p4 edit`)

| 文件 | 关键改动 | 影响半径 |
|---|---|---|
| `AbilitySystem/Abilities/HeroAbility.cpp` | `ActivateAbility`/`EndAbility` 钩 BeginHolding/EndHolding/EndSelfCast;SelfCast 路径用 `SetTimerForNextTick` 延迟 ConfirmHoldingAbility | **中** — 改了关键的 ability lifecycle hooks |
| `AbilitySystem/Abilities/HeroAbilityDataAsset.h` | `+TargetSelectionProfile` `+bDrivesIdleSoftTarget` 2 个 UPROPERTY | 小 — 纯加字段,旧资产 backwards compat |
| `AbilitySystem/Abilities/SkillModuleApplyBuffToTarget.h` | 加 `ArmedTarget` enum 值;`AreaPreviewTargetOrSelf` 标 `UMETA(Hidden)` | 小 — enum 增加 + 老值不可见 |
| `AbilitySystem/Abilities/SkillModuleApplyBuffToTarget.cpp` | UpdateTargetData 加 ArmedTarget 分支(读 Component CurrentTarget + AllowSelfFallback 兜底);Execute 加服务器验证 | **中** — 通用 module 加了 V2.1 专属逻辑 |
| `Character/HeroDataAsset.h/.cpp` | 加 `IsDataValid` 重写,校验驱动源唯一性 | 小 |
| `Input/GameInputDataAsset.h` | `+CancelHoldingInputAction` `+SelfCastModifierInputAction` 2 个 UPROPERTY | 小 |
| `MobaUnit/MobaUnitInterface.cpp` | `MobaUnitDead` 末尾加 `Validator->RecordDeath` 调用 + include | **中** — 改了核心 interface |
| `Player/MatchPlayerController.h` | `+UTargetSelectionComponent*` 成员 + `Input_CancelHolding/Input_SelfCastModifier/IsSelfCastModifierActive` 3 个方法 | 中 — PC 加了新组件持有 |
| `Player/MatchPlayerController.cpp` | 构造 CreateDefaultSubobject + SetupInputComponent 加 2 个 IA 绑定 + handler | 中 |

### 新增 Content(`p4 add`,测试用,可独立回滚)

| 资产 | 类型 | 用途 |
|---|---|---|
| `/Game/Heroes/Muriel/Shield/TSP_MurielShield.uasset` | TargetSelectionProfileDataAsset | Muriel 单体 Profile 实例 |
| `/Game/Heroes/Muriel/Shield/HAD_MurielShield_V21Test.uasset` | HeroAbilityDataAsset | 复制自 HAD_MurielShield + 加 V2.1 字段 + ConfirmHoldingSkillModules[1].TargetType=ArmedTarget |
| `/Game/Heroes/Muriel/Shield/GA_MurielShield_V21Test.uasset` | Blueprint(UHeroAbility) | 复制自 GA_MurielShield + AbilityDataAsset 指向 V21Test |
| `/Game/Input/Actions/IA_CancelHolding.uasset` | InputAction | 右键取消 |
| `/Game/Input/Actions/IA_SelfCastModifier.uasset` | InputAction | Alt 自施法 modifier |
| `/Game/Heroes/Kallari/Mark/W_SoftTargetIndicator.uasset` | WidgetBlueprint | 复制自 `W_MarkTargetIndicator`(目前样式相同,后续美术 polish) |

### 修改 Content(`p4 edit`)

| 资产 | 改动 |
|---|---|
| `/Game/Heroes/Muriel/HeroData_Muriel.uasset` | `Abilities[2]` GA_MurielShield_C → **GA_MurielShield_V21Test_C** |
| `/Game/Input/Mappings/IMC_Default.uasset` | 加 2 条映射(右键 → IA_CancelHolding,LeftAlt → IA_SelfCastModifier)|
| `/Game/Input/Mappings/IMC_Default_AltKeys.uasset` | 同上(双 IMC 共存,各加一份)|
| `/Game/GlobalSettings/GameInputData_Global.uasset` | 填 `CancelHoldingInputAction` + `SelfCastModifierInputAction` 两个字段 |

---

## 架构关系

```
                     ┌─ TargetSelectionProfileDataAsset (Designer 配置)
                     │
       Designer ─┐   │   ┌── 15+ 字段(Range/ScreenCone/ScoreWeights/Stickiness/ResolvePolicy/WidgetClass)
                ↓   ↓   ↓
HeroAbilityDataAsset.TargetSelectionProfile ─┐
HeroAbilityDataAsset.bDrivesIdleSoftTarget ──┤
                                              ↓
                  ┌── UTargetSelectionComponent (挂 AMatchPlayerController, 本地玩家)
                  │   │
                  │   ├── State: Idle/Holding/SelfCast
                  │   ├── ActiveProfile + CurrentTarget
                  │   ├── 20Hz tick (Idle) / 60Hz tick (Holding)
                  │   ├── GatherCandidates → ScoreAndSelectBest
                  │   ├── UpdateTargetIndicatorWidget(挂 WidgetComponent 到 CurrentTarget)
                  │   └── IdleDriveAbility / Handle(查 CanActivateAbility 决定是否显示)
                  │
                  ├── UHeroAbility::ActivateAbility (M2.1 hook)
                  │   ├── 普通路径 → BeginHolding(profile)
                  │   └── Alt+键   → BeginSelfCast + SetTimerForNextTick(ConfirmHoldingAbility)
                  │
                  ├── UHeroAbility::EndAbility (M2.1 hook)
                  │   └── 检测 Component.State → EndSelfCast / EndHolding
                  │
                  ├── SkillModuleApplyBuffToTarget.UpdateTargetData
                  │   └── TargetType=ArmedTarget → 读 Component.CurrentTarget(+ AllowSelfFallback 兜底)
                  │
                  └── SkillModuleApplyBuffToTarget.Execute (server side only)
                      └── UTargetingValidatorSubsystem.ValidateTargetOnServer
                          ├── Dead with 300ms tolerance
                          ├── Team strict
                          └── (Range/LOS 信任 client)

                  独立外部 hook:
                  IMobaUnitInterface::MobaUnitDead → Validator->RecordDeath (server only)
                  UHeroDataAsset::IsDataValid → 驱动源唯一性 editor 校验
```

---

## 关键设计决策(链接到 ADR)

| ADR | 主题 | 一句话 |
|---|---|---|
| **0001** | 不默认 fallback 自施法 | 防御性技能 opt-in `AllowSelfFallback`,系统不擅自决定 |
| **0002** | β 模式(Armed 跟手) | Holding 期间 Armed 随评分跟手,不锁死 — 推翻了 V2 doc 的 α 设计 |
| **0003** | 服务器 300ms 容忍窗 | 不做 full lag-comp rewind;Dead 容忍 300ms,Team/CD/Mana 严格判定 |

---

## Test 状态(诚实)

### ✅ 已验证(本地 PIE)
- 空闲走动,L 括号跟着友军跳
- 按 Shield 蓄力,变绿,跟手跳转
- 友军被另一队员挡住 — 仍能正确选(LOS fix)
- 松手护盾打到当前 Armed 上
- 右键蓄力中取消,无消耗
- Alt+Shield 立即上自己
- Shield CD / 沉默 / 未学习时,框消失
- 空旷场地按 Shield → AllowSelfFallback 给自己

### ❌ 未验证
- **联机场景**(2+ client + DS) — M4 服务器 300ms 容忍窗的实际行为,需用 `net pktlag 200` 模拟跨服 ping 跑 race 测试
- **Editor 校验** — 没真创建过"两个 ability 都标 `bDrivesIdleSoftTarget=true`"的英雄数据看是否报错
- **多英雄回归** — 其他英雄换装 V21Test 后 SkillModule 行为是否仍正确
- **战斗压测** — 团战 10v10 高密度场景下 LOS trace 性能(H5)

---

## Submit 建议

**分 2 个 P4 changelist**,降低回滚成本:

### CL-1 — "V2.1 单体施法选取 M1-M5 代码实现"
所有 Source/ 改动 + 新增。Block 在以下 3 个 review 点:
- HeroAbility.cpp SelfCast 路径(B3 fix 后的 SetTimerForNextTick 写法是否符合项目时序惯例)
- MobaUnitInterface.cpp 加 hook(主程是否接受这种侵入,还是建议改 OnUnitDied delegate)
- UWorldSubsystem 模式(项目其他系统是否也用,以保持一致)

### CL-2 — "V2.1 Muriel Shield 测试配置(V21Test 系列)"
所有 Content/ 资产。**可独立先 submit**(纯测试数据,不影响别人)。

### Design 仓(Git)
- `CONTEXT.md`(新)
- `docs/adr/0001 0002 0003.md`(新)
- `docs/fdd/.../Skill - 单体施法选取 V2.1.md/.html`(新)
- 本文(`代码交接.md`)

---

## 后续 follow-up 优先级排序

1. **H6 联机测试**(必须做)— 上线前必须真实 DS 环境跑过
2. **H3 Widget owner lifecycle**(应做)— 边界 case 易 dangling
3. **H4 + H5 性能优化**(profile 后决定)— 团战压测看是热点再优化
4. **H1 + H2 Hook 抽象**(可选)— 等下一次重构窗口
5. **M1-M6** — 各种 nit / polish,慢慢清

---

**版本**:V2.1 代码交接
**日期**:2026/05/22
**作者**:战斗策划 + AI 协作产出
**改动 commit author**:likeke@xd.com
