## 战点制度：怪物 AI 与指挥官子系统文档

> **适用版本**：Unreal Engine 5 + PaperZD + Gameplay Ability System
> **核心原则**：封闭战点内，怪物自动锁定玩家，无需 AI 感知组件；所有战术协调由「战点指挥官」集中管理，怪物行为树仅负责个体状态机。

---

### 一、架构概述

游戏战斗发生在**封闭战点**内，不存在“潜行”或“丢失视野”概念。一旦进入战点，所有怪物立刻将玩家视为唯一目标。这种行为被**硬编码**在战点激活流程中，不再依赖 `AIPerceptionComponent`（视觉、听觉等）。

整体分为两个协作子系统：

- **战点指挥官**：全局资源管理（围攻席位、攻击互斥）、战术指令（集结、冲锋、撤退）、动态目标生成。
- **怪物 AI**：个体行为状态机（巡逻、游走、围攻、攻击、逃离、失控、死亡），具体动作执行。

两者均通过 **Gameplay Tag** 和 **Gameplay Event** 解耦，完全融入 GAS 体系。

---

### 二、关键资产清单

#### 2.1 蓝图类

| 蓝图类 | 说明 |
|--------|------|
| `BP_CombatZoneManager` | 战点指挥官 Actor，挂载 ASC，拥有战术 GA / GE |
| `BP_Grunt_Character` | 怪物角色基类，挂载 ASC，实现 `IAbilitySystemInterface` |
| `AIC_Grunt` | 怪物 AI 控制器，运行行为树 / 黑板 |
| `BT_Grunt` | 行为树资产（状态机） |
| `BB_Grunt` | 黑板资产（共享数据） |

#### 2.2 枚举

**`EMonsterCombatState`**（黑板 Enum 键类型）

| 状态 | 说明 |
|------|------|
| `Patrol` | 未进入战点时的默认巡逻（或待机） |
| `Roaming` | 已发现玩家但围攻满员，在禁区外游走 |
| `Sieging` | 已分配围攻席位，保持距离等待攻击 |
| `Attacking` | 正在执行攻击动作，占用攻击令牌 |
| `Fleeing` | 低血量逃离，跑向安全位置 |
| `LostControl` | 受击硬直 / 击飞，任何动作暂停 |
| `Dead` | 死亡，行为树停止执行 |

#### 2.3 Gameplay Tag

| 标签 | 用途 |
|------|------|
| `Faction.Player` | 玩家阵营标识（松散标签） |
| `Faction.Enemy` | 怪物阵营标识（松散标签） |
| `State.LostControl` | 失控标签（GE 添加，起身动画移除） |
| `State.Dead` | 死亡标签（GE 添加） |
| `Command.Regroup` | 集结指令（指挥官 GA 发送，怪物监听） |
| `Command.Charge` | 冲锋指令 |
| `Command.Retreat` | 撤退指令 |
| `Cooldown.Command.Regroup` | 集结冷却标签（GE 添加给指挥官） |
| `Cooldown.Command.Charge` | 冲锋冷却标签 |
| `Cooldown.Command.Retreat` | 撤退冷却标签 |

#### 2.4 黑板键（`BB_Grunt`）

| 键名 | 类型 | 说明 |
|------|------|------|
| `TargetActor` | Actor | 当前目标（玩家），由战点激活时写入 |
| `CombatState` | Enum `EMonsterCombatState` | 当前行为状态 |
| `bIsLostControl` | Bool | 失控标志（由 Tag 事件同步） |
| `SiegeTargetLocation` | Vector | 围攻保持点（由指挥官计算） |
| `RoamingTargetLocation` | Vector | 游走目标点（由指挥官或 Service 计算） |
| `HomeLocation` | Vector | 巡逻原点 / 出生点 |
| `AssignedAngleIndex` | Int | 分配的围攻角度索引 |
| `bHasSiegeSlot` | Bool | 是否持有围攻席位（用于快速判断） |
| `ActiveCommandTag` | GameplayTag | 当前正在执行的指挥官命令标签 |
| `AttackCooldownRemaining` | Float | 攻击冷却剩余（可选，用于 Service） |

---

### 三、战点指挥官子系统

#### 3.1 核心职责

- 管理**围攻席位**（上限 5 个），向怪物分配围绕玩家的均匀角度和保持点。
- 管理**攻击互斥令牌**，同一时间仅允许一只怪物攻击。
- 提供**随机游走目标点**，避免候补怪物闯入围攻禁区。
- 作为**战术指令中枢**，通过 GA 发动集结、冲锋、撤退等，并管理冷却。

#### 3.2 关键配置属性（蓝图变量）

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `MaxSiegeSlots` | int | 5 | 最大围攻席位数量 |
| `SiegeRadius` | float | 100.0 | 围攻时与玩家的保持距离（厘米） |
| `RoamingRadius` | float | 600.0 | 游走禁区半径，必须 ≥ `SiegeRadius` |
| `AttackTokenTimeout` | float | 2.0 | 攻击令牌超时（秒），防止死锁 |
| `SiegeAngles` | TArray<float> | 0,72,144,216,288 | 围攻角度（均匀分布） |

#### 3.3 内部数据（运行时维护）

- `ActiveSiegeHolders`：`TArray<APawn*>`，当前围攻者列表。
- `UnusedAngles`：`TArray<int32>`，可用角度索引池。
- `MonsterAngleMap`：`TMap<APawn*, int32>`，每个怪物的分配角度索引。
- `bAttackTokenInUse`：bool，攻击令牌状态。
- `TokenHolder`：`APawn*`，当前持有令牌的怪物。
- `TokenStartTime`：float，令牌授予时刻。

#### 3.4 蓝图公开函数（供怪物 Service 调用）

| 函数 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `RequestSiegeSlot` | Monster | (bool 成功, int 角度索引, Vector 目标位置) | 尝试分配席位，成功则怪物加入围攻列表，计算并返回世界坐标 |
| `ReleaseSiegeSlot` | Monster | void | 释放席位，从列表移除，回收角度 |
| `RequestAttackPermission` | Monster | bool 成功 | 若令牌空闲且该怪物无冷却，则授予令牌，记录时间 |
| `ReleaseAttackPermission` | Monster | void | 释放令牌，记录该怪物攻击冷却 |
| `GetRandomRoamingTarget` | Monster | Vector | 在玩家周围大于 `RoamingRadius` 但小于 1500 的区域内，随机生成一个导航可到达的点 |
| `ActivateCombatZone` | PlayerActor | void | **战点激活入口**：设置目标玩家，通知所有怪物进入战斗状态 |
| `DeactivateCombatZone` | – | void | 战斗结束，重置所有状态，怪物返回巡逻 |

#### 3.5 战术能力（GA）与冷却 GE

**GA（所有都继承 `GameplayAbility`，并在指挥官 ASC 上授予）**

| GA | 事件标签 | 冷却 GE | 说明 |
|----|----------|---------|------|
| `GA_Regroup` | `Command.Regroup` | `GE_Regroup_Cooldown`（持续 10s） | 命令所有怪物向指挥官位置集结 |
| `GA_Charge` | `Command.Charge` | `GE_Charge_Cooldown`（持续 15s） | 命令所有怪物无视席位直接冲锋玩家 |
| `GA_Retreat` | `Command.Retreat` | `GE_Retreat_Cooldown`（持续 20s） | 命令所有怪物释放席位并撤退到游走区域 |

**GA 内部流程**：
1. 检查冷却（通过 `CommitAbility` 的 Cooldown 规则）。
2. 对范围内所有怪物发送 `Gameplay Event`（事件标签 = 对应的命令标签），附带可选参数（如集结位置）。
3. 应用冷却 GE 到自身。

**怪物端**：在 `BeginPlay` 中通过 ASC 注册这些命令标签的事件，接收到后立即更新黑板 `ActiveCommandTag`，行为树命令分支被触发。

#### 3.6 需要的外部辅助函数

| 函数 | 位置 | 说明 |
|------|------|------|
| `IsPointReachable` | `ACTBlueprintFunctionLibrary` | 检查某点是否在导航网格上且可达 |
| `GetRandomReachablePointInRange` | 同上 | 在环形区域内生成随机可达点，用于 `GetRandomRoamingTarget` |
| `BroadcastEventToMonsters` | 指挥官蓝图或函数库 | 将 Gameplay Event 广播给所有活动怪物 |

---

### 四、怪物 AI 子系统

#### 4.1 战点激活：硬编码锁定玩家

- **不再依赖 AI 感知**。`AIPerceptionComponent` 可以从 `AIC_Grunt` 中移除。
- 当战点触发时（例如玩家走进触发盒），调用 `BP_CombatZoneManager::ActivateCombatZone(Player)`。
- 该函数内部遍历战点内所有怪物（可通过预设数组或 Tag 查询），为每个怪物执行：
  - 设置黑板 `TargetActor = Player`。
  - 可选：设置 `CombatState = Roaming`（或直接由状态评估 Service 处理）。

#### 4.2 行为树顶层结构

```
Root (Sequence, Decorator: State != Dead)
 └── Selector
      ├─ 失控分支: CombatState == LostControl → Wait (无限)
      ├─ 死亡分支: CombatState == Dead → （由根装饰器阻止，实际留空）
      ├─ 命令分支: ActiveCommandTag 有效 → 对应命令序列
      ├─ 逃离分支: CombatState == Fleeing → 逃离逻辑
      ├─ 攻击分支: CombatState == Attacking → 攻击逻辑
      ├─ 围攻分支: CombatState == Sieging → 站位保持
      ├─ 游走分支: CombatState == Roaming → 游走移动
      └─ 巡逻分支: CombatState == Patrol → 原地待命或局部巡逻
```

**命令序列**（例如集结）：
```
Sequence (ActiveCommandTag == Command.Regroup)
  ├─ Move To: 目标点（从指挥官事件数据获取，或预设的集结位置）
  ├─ Wait: 1s（稳定）
  └─ Clear ActiveCommandTag
```

#### 4.3 核心 Service

| Service | 挂载位置 | 间隔 | 职责 |
|---------|----------|------|------|
| `BTS_StateEvaluator` | 根节点 | 0.2s | **状态决策心脏**：根据血量、目标存在、席位状态、命令标签，切换 `CombatState`。也负责向指挥官请求席位、处理 `bIsLostControl`→`LostControl` 状态。 |
| `BTS_AttackMonitor` | 围攻序列 | 0.1s | 检查与玩家距离，向指挥官请求攻击令牌；若获取成功，则设置 `CombatState = Attacking` 并记录。 |
| `BTS_RoamingNavigator` | 游走序列 | 3.0s | 定期从指挥官获取新的随机游走点，写入 `RoamingTargetLocation`。 |
| `BTS_CommandListener` | 根节点 | 事件驱动 | 不靠 Tick，仅绑定 Gameplay Event 回调，更新 `ActiveCommandTag`。 |

**`BTS_StateEvaluator` 核心决策逻辑**（伪代码）：

```
IF NOT TargetActor: CombatState = Patrol; RETURN

IF HAS bIsLostControl: CombatState = LostControl; RETURN

IF Health < fleeThreshold AND Random chance:
    CombatState = Fleeing
    ReleaseSiegeSlot()
    RETURN

IF ActiveCommandTag is valid:
    // 保持当前状态，由命令分支覆盖行动
    RETURN

IF NOT bHasSiegeSlot:
    RequestSiegeSlot() -> if success: CombatState = Sieging, bHasSiegeSlot = true
                          else: CombatState = Roaming
ELSE:
    CombatState = Sieging (保持)
```

#### 4.4 核心 Task

| Task | 用途 | 备注 |
|------|------|------|
| `BTTask_PerformAttack` | 执行一次攻击：播放动画、触发 GA 造成伤害，结束后释放攻击令牌，设 `CombatState = Sieging`。 | 应该等待攻击动画结束或 GA 完成才返回成功。 |
| `BTTask_FleeToSafety` | 计算远离玩家的位置，移动过去，然后释放围攻席位，设 `CombatState = Roaming` 或 `Patrol`。 | |
| `BTTask_ClearCommand` | 将 `ActiveCommandTag` 清空，并释放指挥官给的强制状态（如冲锋后的自动回归）。 | |
| `BTTask_SetBlackboardEnum` | 通用节点：设置 `CombatState`。 | |

#### 4.5 状态转换图示

```
Patrol ───(获得 TargetActor)───→ 请求席位
                                    ├─ 成功 → Sieging
                                    └─ 失败 → Roaming
Roaming ──(定期检查席位有空)──→ Sieging
Sieging ──(获取攻击令牌)──→ Attacking
Attacking ──(攻击结束)──→ Sieging
任何非失控状态 ──(受击)──→ LostControl
LostControl ──(硬直结束)──→ 重新评估（通常回到之前的围攻/游走）
Sieging/Roaming ──(血量低且概率)──→ Fleeing
Fleeing ──(到达安全点)──→ Roaming or Patrol
目标丢失 → Patrol
```

---

### 五、辅助函数库（`ACTBlueprintFunctionLibrary`）

| 函数 | 签名 | 状态 |
|------|------|------|
| `CountNearbyPaperZDFactionActors` | `(Center, Radius, FactionTag) -> int32` | ✅ 已实现 |
| `GetRandomReachablePointInRing` | `(Origin, InnerRadius, OuterRadius) -> FVector` | 🚧 待实现（需集成导航查询） |
| `IsPointReachable` | `(Origin, Point) -> bool` | 🚧 待实现 |
| `SendGameplayEventToActorArray` | `(TArray<AActor*>, EventTag, EventData)` | 🚧 待实现（批量发送 Gameplay Event） |
| `SetTeamIdForController` | `(Controller, TeamId)` | ⚠️ 已注释（弃用 TeamID 方案） |

---

### 六、典型流程

#### 6.1 怪物进入战点

- 玩家进入触发区域 → 关卡脚本调用 `ActivateCombatZone(Player)`。
- 指挥官遍历怪物列表，为每个怪物的黑板写入 `TargetActor`。
- 怪物 `BTS_StateEvaluator` 检测到目标存在，尝试请求围攻席位：
  - 若成功 → `CombatState = Sieging`，`SiegeTargetLocation` 被设定。
  - 若失败 → `CombatState = Roaming`，`BTS_RoamingNavigator` 开始生成游走点。

#### 6.2 围攻与攻击循环

- 怪物在 `Sieging` 状态：行为树 `Move To (SiegeTargetLocation)`，到位后等待。
- `BTS_AttackMonitor` 检测距离满足、攻击冷却已过，调用 `RequestAttackPermission`。
- 获得令牌 → `CombatState = Attacking`。
- `BTTask_PerformAttack` 执行攻击动作，结束后调用 `ReleaseAttackPermission`，重置状态为 `Sieging`。

#### 6.3 指挥官指令

- 条件触发（如玩家使用特定技能或按某个测试键）→ 指挥官激活 `GA_Regroup`。
- 自身添加冷却 GE，向所有怪物发送 `Command.Regroup` 事件。
- 怪物收到事件 → 写入 `ActiveCommandTag`。
- 行为树命令分支优先执行，怪物移动至集结位置，完成后清除命令标签，返回之前的状态。

---

### 七、开发状态与下一步

#### 已完成项
- GameplayTag 阵营区分（`Faction.Enemy` 松散标签）
- `CountNearbyPaperZDFactionActors` 统计函数
- 基础失控（`State.LostControl`）和死亡逻辑
- 怪物受伤、硬直、转身等底层反

#### 待实现项（按优先级）
1. **创建 `BP_CombatZoneManager` 蓝图及核心函数**（席位管理、游走点生成）
2. **重构怪物行为树**：引入 `CombatState` 枚举，搭建状态分支结构
3. **实现 `BTS_StateEvaluator`**（状态机核心）
4. **实现 `BTTask_PerformAttack`**（攻击流程）
5. **实现指挥官 GA 及冷却 GE**（集结等）
6. **实现逃离和游走路径生成**
7. **移除 AI 感知组件**，改为硬编码目标设置
8. 测试完整战斗流程并调节数值

---

*最后更新：2026-06-09*
