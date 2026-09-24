# WoW simplesba 插件产品文档

## 1. 产品背景

World of Warcraft 正式服已经提供 Assisted Combat 系统，可以根据玩家当前职业、专精、资源、目标状态和战斗环境，给出当前推荐释放的下一个技能。

Blizzard 同时提供 Single-Button Assistant，使玩家能够通过一个固定按键持续执行 Assisted Combat 推荐的技能。

但 Single-Button Assistant 存在明显代价：

* 通过 Single-Button Assistant 释放技能时，会增加约 25% 的 Global Cooldown。
* 对熟悉职业技能、但不想完整记忆输出循环的玩家而言，这个惩罚比较明显。
* Blizzard Assisted Combat 已经完成了“下一技能判断”，但默认界面对推荐结果的呈现较为简单。

因此，可以开发一个第三方插件：

> 使用 Blizzard Assisted Combat API 获取当前推荐技能，但不通过 Single-Button Assistant 释放技能。

插件只负责：

* 判断当前推荐技能；
* 显示技能；
* 显示对应键位；
* 高亮 Action Bar；
* 辅助玩家自己完成按键。

实际技能仍由玩家通过普通 Action Bar 按键手动施放。

这样可以保留 Assisted Combat 的推荐能力，同时避免 Single-Button Assistant 自身的 GCD penalty。

---

# 2. 产品目标

插件的核心目标是：

> 将 Blizzard Assisted Combat 从“单按钮自动施法工具”变成一个高可读性的实时 Rotation Assistant。

插件不主动施法。

插件负责回答一个问题：

> “我现在下一招应该按哪个键？”

理想体验：

```text
             NEXT

        Hammer of Wrath
             [ E ]

          [技能图标]

────────────────────────

资源 / CD / Target 信息
```

玩家看到提示后直接按：

```text
E
```

技能通过普通 Action Bar 执行。

---

# 3. 核心设计原则

## 3.1 推荐和执行完全分离

系统结构：

```text
Blizzard Assisted Combat
          │
          ▼
C_AssistedCombat.GetNextCastSpell()
          │
          ▼
       spellID
          │
          ▼
      插件 UI
          │
          ▼
技能名称 / 图标 / 键位 / ActionBar 高亮
          │
          ▼
       玩家按键
          │
          ▼
普通 ActionBar Secure Action
          │
          ▼
       技能施放
```

插件只参与：

```text
Recommendation
```

不参与：

```text
Execution
```

这样可以尽量避开 Blizzard 对战斗自动化的限制。

---

# 4. 为什么不用 Single-Button Assistant

Blizzard 原生模式：

```text
一个固定键
   ↓
Assisted Combat 判断下一技能
   ↓
Single-Button Assistant
   ↓
技能释放
   ↓
附带 GCD penalty
```

本插件模式：

```text
Assisted Combat 判断下一技能
   ↓
插件显示推荐
   ↓
告诉玩家对应键位
   ↓
玩家按普通技能键
   ↓
正常技能施放
```

区别在于：

```text
Blizzard 模式

Recommendation + Execution
都由 Assistant 完成
```

而插件：

```text
Recommendation
由 Assisted Combat 完成

Execution
仍然由玩家完成
```

---

# 5. 非目标

插件第一阶段明确不尝试实现以下功能。

## 5.1 不做真正的一键 Rotation

不尝试实现：

```text
永远按 F

F → Judgment

下一 GCD：

F → Crusader Strike

下一 GCD：

F → Hammer of Wrath
```

这种机制要求插件在战斗中动态决定一个 SecureActionButton 当前到底执行什么技能。

这正是 WoW Combat Lockdown / Protected Action 系统重点限制的行为。

---

## 5.2 不绕过 Blizzard Combat Security

不尝试：

* 战斗中动态修改 protected frame 的 spell attribute；
* 普通 Lua 自动执行 CastSpell；
* 自动 Target；
* 自动根据 combat state 选择 SecureAction；
* 模拟玩家输入；
* Hardware Event 绕过；
* 自动连续执行多个 GCD。

插件应当保持：

```text
每个实际技能释放
=
一次真实玩家输入
```

---

## 5.3 不自行开发完整 Rotation Engine

第一阶段不自己计算：

```text
Debuff
Proc
Resource
Cooldown
Target Count
Execute Phase
Buff Window
Trinket Window
```

来决定技能。

因为 Blizzard 已经提供：

```lua
C_AssistedCombat.GetNextCastSpell()
```

插件优先直接消费 Blizzard 的计算结果。

---

# 6. 目标用户

主要目标玩家：

### A. 不想完整学习 Rotation 的玩家

希望：

```text
看到什么
按什么
```

但又不想使用 Single-Button Assistant 的额外 GCD。

---

### B. 治疗职业

尤其适合：

* Holy Paladin
* Discipline Priest
* Holy Priest
* Restoration Shaman
* Restoration Druid
* Mistweaver Monk
* Preservation Evoker

治疗技能仍然由玩家判断。

输出技能则由 Assisted Combat 辅助。

例如奶骑：

```text
治疗部分

Holy Shock
Word of Glory
Flash of Light
Beacon
Cooldown
Utility

玩家自己处理
```

同时：

```text
输出部分

Judgment
Crusader Strike
Hammer of Wrath
Consecration
其他输出技能

由 Assisted Combat 推荐
```

这样可以降低治疗职业“补 DPS”的操作负担。

---

### C. Alt 玩家

玩家可能有大量小号。

不需要每个职业重新记完整 Priority List。

只需要知道：

```text
技能在哪个键
```

---

# 7. MVP 功能

第一版尽量保持简单。

## 7.1 当前推荐技能

读取：

```lua
C_AssistedCombat.GetNextCastSpell()
```

获取：

```text
spellID
```

进一步取得：

```text
Spell Name
Spell Icon
```

UI：

```text
┌─────────────────────┐
│        NEXT         │
│                     │
│   [技能 Icon]       │
│                     │
│ Judgment            │
│                     │
│       [ Q ]         │
└─────────────────────┘
```

---

# 8. Action Bar 键位识别

插件需要自动判断推荐技能当前在哪一个 Action Bar Slot。

例如：

```text
Judgment
```

存在于：

```text
ActionButton5
```

对应 Binding：

```text
Q
```

最终显示：

```text
Judgment

Q
```

---

# 9. Action Bar 高亮

当 Blizzard 推荐：

```text
Judgment
```

插件找到 Judgment 所在 Action Bar Button。

然后：

```text
添加 Glow
```

效果类似：

```text
┌──────────┐
│ Judgment │
│    Q     │
└──────────┘
   ↑
 Glow
```

建议支持：

* Blizzard Action Bar
* Bartender
* Dominos

第三方 ActionBar 支持可以放在后续版本。

---

# 10. 中央技能提示

提供独立 HUD。

默认位置：

```text
屏幕中央
角色下方
```

类似：

```text
          NEXT

      [ Judgment ]

           Q
```

支持：

* 拖动；
* Scale；
* Alpha；
* Icon Size；
* Text Size；
* 锁定位置。

---

# 11. 推荐技能变化刷新

插件需要监听可能导致 Assisted Combat 推荐变化的事件。

可以采用：

```text
Event Driven
+
短间隔刷新
```

推荐刷新逻辑：

```text
PLAYER_ENTERING_WORLD
PLAYER_REGEN_DISABLED
PLAYER_REGEN_ENABLED
PLAYER_TARGET_CHANGED
UNIT_POWER_UPDATE
SPELL_UPDATE_COOLDOWN
UNIT_AURA
ACTIONBAR_UPDATE_STATE
ACTIONBAR_SLOT_CHANGED
```

同时可以做一个低频：

```text
OnUpdate
```

例如：

```text
0.05～0.10 秒
```

查询当前：

```lua
C_AssistedCombat.GetNextCastSpell()
```

只有 spellID 发生变化时更新 UI。

避免每帧重新绘制。

---

# 12. 技能 → Keybind Mapping

需要建立：

```text
spellID
      ↓
ActionBar Slot
      ↓
Button
      ↓
Binding
```

例如：

```text
spellID:

20271
Judgment
```

找到：

```text
ACTIONBUTTON5
```

对应：

```text
Q
```

最终：

```text
20271
→ ACTIONBUTTON5
→ Q
```

---

# 13. 一个技能存在多个键位时

例如 Judgment 同时放在：

```text
Q

Mouse Button 4
```

默认选择：

```text
Primary Binding
```

设置里允许：

```text
Prefer Keyboard

Prefer Mouse

First Binding
```

---

# 14. 无绑定技能

如果推荐技能没有放 ActionBar：

```text
Hammer of Wrath

No Keybind
```

UI 显示：

```text
Hammer of Wrath

UNBOUND
```

并可选择：

```text
红色提示
```

方便玩家完善 Action Bar。

---

# 15. 下一技能 HUD

MVP UI：

```text
       NEXT

   ┌──────────┐
   │          │
   │  ICON    │
   │          │
   └──────────┘

   Judgment

      Q
```

后续可以增加：

```text
Cooldown
Resource
Target
Range
```

例如：

```text
Judgment
Q

Ready
30 yd
```

---

# 16. Combat Only Mode

默认：

```text
仅战斗中显示
```

设置：

```text
Show:

Combat Only
Always
Target Exists
Training Dummy Only
```

---

# 17. Assisted Combat 状态检测

如果当前职业 / Spec：

```text
不支持 Assisted Combat
```

或者：

```text
GetNextCastSpell()
返回 nil
```

显示：

```text
No Recommendation
```

或者自动隐藏。

---

# 18. 治疗职业模式

这是插件比较有价值的使用场景。

例如 Holy Paladin。

插件只辅助：

```text
Damage Rotation
```

治疗继续由玩家自己处理。

UI可以设置：

```text
Hide Recommendation
While Mouseover Friendly Unit
```

或者：

```text
Only Show Damage Recommendations
```

最终操作模型：

```text
队友掉血

→ Mouseover Heal

血线稳定

→ 看 HUD

→ Q / E / R 输出
```

---

# 19. 可选模式：ActionBar Only

有些玩家不想看额外 HUD。

模式：

```text
HUD OFF

ActionBar Glow ON
```

此时 Blizzard 推荐 Judgment：

```text
Judgment 按钮发光
```

玩家直接按。

---

# 20. 可选模式：HUD Only

相反：

```text
HUD ON

ActionBar Glow OFF
```

适合已经熟悉键位的人。

---

# 21. 推荐技能历史

后续版本可以增加 Debug 模式：

```text
14:30:02 Judgment
14:30:04 Crusader Strike
14:30:06 Hammer of Wrath
14:30:07 Judgment
```

用于分析 Blizzard Assisted Rotation。

可以记录：

```text
Timestamp
SpellID
Spell Name
Target
Combat State
```

---

# 22. Rotation 分析模式

Debug 信息可以统计：

```text
一场战斗 Assisted Combat 推荐：

Judgment            38
Crusader Strike     32
Hammer of Wrath     18
Consecration        11
```

用途：

* 分析 Blizzard 内置 rotation；
* 比较职业设计；
* 验证 Assisted Combat 是否出现明显异常；
* 发现某些 talent 是否完全没有进入 rotation。

---

# 23. Training Dummy 模式

可以增加一个测试模式。

玩家攻击木桩后：

```text
记录 Blizzard 推荐序列
```

例如：

```text
0.0 Judgment
1.5 Crusader Strike
3.0 Judgment
4.5 Hammer of Wrath
6.0 Consecration
```

这样可以研究不同：

```text
Talent
Gear
Haste
Target Count
```

下 Blizzard 推荐行为。

---

# 24. 潜在增强功能：推荐确认

可以记录：

```text
Recommended Spell
```

与：

```text
Player Cast Spell
```

进行比较。

例如：

```text
Recommended:
Judgment

Player Cast:
Crusader Strike
```

统计：

```text
Follow Rate: 82%
```

注意：

该功能用于分析，不应评价玩家操作“正确/错误”。

---

# 25. 潜在增强功能：Missed Recommendation

例如：

```text
推荐 Hammer of Wrath
```

玩家 3 秒内没有使用。

插件可以记录：

```text
Skipped
```

用于训练 rotation。

默认建议关闭声音提醒，避免 UI 过于干扰。

---

# 26. Potential Experimental Feature

可以研究：

```text
SecureActionButton
```

但只作为实验功能。

例如战斗外预先建立：

```text
ButtonJudgment
ButtonCrusaderStrike
ButtonHammerOfWrath
```

每个 Button 固定绑定一个技能。

但不尝试在战斗中通过普通 Lua 动态修改：

```text
spell
type
macrotext
click target
```

核心研究问题：

```text
是否存在 Blizzard 官方允许的 secure execution path，
可以根据 Assisted Combat recommendation
选择预创建的 Secure Action。
```

如果只能依靠受保护、未公开、明显规避设计限制的机制，则不进入正式版本。

---

# 27. 为什么“一键无 penalty”不是 MVP

理想中的功能：

```text
F
↓
读取 GetNextCastSpell()
↓
Judgment
↓
F 释放 Judgment

下一 GCD

F
↓
Crusader Strike
```

技术上的关键问题不是：

```text
如何知道下一个技能
```

这个 Blizzard 已经提供：

```lua
C_AssistedCombat.GetNextCastSpell()
```

真正的问题是：

```text
如何在 Combat Lockdown 中，
让同一个 SecureActionButton
根据普通 Lua 得到的 spellID
执行不同 protected action。
```

这正属于 Blizzard Secure Execution Model 的核心限制范围。

因此：

```text
Recommendation

容易
```

```text
Dynamic Protected Execution

困难 / 很可能不允许
```

---

# 28. 产品安全边界

插件正式版本应遵守以下原则：

```text
插件：
可以读取
可以计算
可以显示
可以高亮
可以提醒
```

但：

```text
插件：
不自动施法
不模拟输入
不动态替玩家决定 Secure Action
```

最终行为始终：

```text
Player Input
→ Action
```

---

# 29. 插件架构

建议模块：

```text
AssistedCombatHelper/
│
├── AssistedCombatHelper.toc
│
├── Core.lua
│
├── AssistedCombat.lua
│
├── ActionBar.lua
│
├── Keybind.lua
│
├── HUD.lua
│
├── Glow.lua
│
├── Events.lua
│
├── Settings.lua
│
└── Debug.lua
```

职责：

### Core.lua

插件初始化。

### AssistedCombat.lua

调用：

```lua
C_AssistedCombat.GetNextCastSpell()
```

维护：

```text
CurrentRecommendedSpell
```

### ActionBar.lua

扫描 ActionBar。

建立：

```text
spellID → action slot
```

### Keybind.lua

读取：

```text
Button → Keybind
```

### HUD.lua

中央技能提示。

### Glow.lua

ActionBar Highlight。

### Events.lua

监听战斗 / 技能 / ActionBar 相关事件。

### Settings.lua

配置。

### Debug.lua

日志与 Rotation 分析。

---

# 30. SavedVariables

建议：

```lua
AssistedCombatHelperDB = {
    enabled = true,

    hud = {
        enabled = true,
        scale = 1.0,
        alpha = 1.0,
        x = 0,
        y = -150,
    },

    glow = {
        enabled = true,
    },

    display = {
        combatOnly = true,
        showSpellName = true,
        showKeybind = true,
    },

    debug = {
        enabled = false,
        logRecommendations = false,
    }
}
```

---

# 31. Slash Commands

建议：

```text
/ach
```

打开设置。

其他：

```text
/ach lock
/ach unlock
/ach reset
/ach debug
/ach test
```

---

# 32. MVP 开发阶段

第一阶段只实现：

```text
GetNextCastSpell
↓
显示 Icon
↓
显示 Spell Name
↓
识别 ActionBar
↓
显示 Keybind
```

完成后已经具有实际使用价值。

---

# 33. 第二阶段

增加：

```text
ActionBar Glow
Combat Only
HUD customization
Third-party ActionBar support
```

---

# 34. 第三阶段

增加：

```text
Recommendation History
Combat Logging
Training Dummy Analysis
Recommendation Follow Rate
```

---

# 35. 第四阶段：Secure API Research

独立进行实验：

```text
Assisted Combat
+
SecureActionButton
```

目标仅为确认：

```text
Blizzard 当前允许做到哪里
```

而不是把绕过限制作为产品设计基础。

重点研究：

```text
SecureActionButtonTemplate
SecureHandler
StateDriver
Click Binding
Macro Conditionals
Combat Lockdown
Attribute Changes
```

---

# 36. 产品最终定位

插件不应该定位为：

> 一键输出插件。

更准确的定位是：

> Blizzard Assisted Combat 的增强型 HUD 与键位助手。

一句话描述：

> Shows Blizzard's next recommended ability and the key you should press.

中文：

> 实时显示 Blizzard Assisted Combat 推荐的下一技能以及对应快捷键。

---

# 37. 最终体验目标

理想状态下，玩家进入战斗以后只需要关注一个很小的区域：

```text
          NEXT

        [ICON]

      Judgment

          Q
```

按下：

```text
Q
```

随后：

```text
          NEXT

        [ICON]

   Crusader Strike

          E
```

玩家：

```text
E
```

整个流程仍然是：

```text
一个技能
=
一次真实按键
```

但玩家不需要自己记完整 Priority List。

这样可以同时获得：

* Blizzard Assisted Combat 的实时推荐；
* 普通技能施放路径；
* 不依赖 Single-Button Assistant；
* 更低的 Rotation 学习成本；
* 更清晰的键位提示；
* 较好的职业切换体验。

这应当作为插件 V1 的核心产品方向。
