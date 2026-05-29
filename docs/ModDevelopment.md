# MGC Mod 开发文档

## 目录

1. [快速开始](#快速开始)
2. [Mod 基础结构](#mod-基础结构)
3. [生命周期](#生命周期)
4. [事件系统](#事件系统)
5. [Mod 间通信](#mod-间通信)
6. [配置系统](#配置系统)
7. [UI 系统](#ui-系统)
8. [更新循环](#更新循环)
9. [完整示例](#完整示例)
10. [常见问题](#常见问题)
11. [API 参考](#api-参考)
12. [相关文档](#相关文档)
---

## 快速开始

### 环境要求

| 项目 | 要求 |
|------|------|
| Unity 版本 | 2017.4.24f1 |
| .NET Framework | 3.5 及以下 |
| 开发工具 | dnSpy 或 Visual Studio |

### 第一个 Mod

```csharp
using MGC.ModSystem;

public class MyFirstMod : ModBase
{
    // === 必须实现 ===
    public override string name => "我的第一个Mod";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    public override bool RequiresRestart => false;
    
    // === 可选实现 ===
    public override string Description => "这是我的第一个Mod";
    
    public override void Load()
    {
        base.Load();
        MGCManager.DebugInfo.Add($"[{name}] 加载成功！");
    }
}
```

### Mod 放置位置

| 类型 | 位置 |
|------|------|
| 内部 Mod | 直接编译到游戏主程序 |
| 外部 Mod（预留） | `{持久化目录}/Mods/` 文件夹 |

---

## Mod 基础结构

### IMod 接口

所有 Mod 必须实现 `IMod`，推荐继承 `ModBase`。

| 成员 | 类型 | 说明 |
|------|------|------|
| `name` | get | Mod 唯一标识（显示在 UI） |
| `ModVersion` | get | 版本号 |
| `SupportedModpackVersion` | get | 支持的 MGC 版本 |
| `enabled` | get/set | 当前是否启用 |
| `willbeenabled` | get/set | 下次启动是否启用 |
| `Description` | get | 描述信息（显示在 UI） |
| `RequiresRestart` | get | 开关是否需要重启 |
| `Load()` | void | 加载时调用 |
| `Unload()` | void | 卸载时调用 |
| `Update()` | void | 每帧调用 |
| `FixedUpdate()` | void | 固定帧调用 |
| `OnGUI()` | void | IMGUI 绘制 |
| `OnSceneLoaded()` | void | 场景加载时 |
| `OnEnableChanged()` | void | 开关状态变化时 |
| `OnCreateUGUI()` | void | 创建配置界面时 |
| `SaveConfig()` / `LoadConfig()` | void | 配置读写 |

### ModBase 提供的便捷功能

- ✅ 配置读写（`GetConfig` / `SetConfig`）
- ✅ 事件监听（`Listen` / `Call`）
- ✅ Mod 间通信（`GetMod`）
- ✅ 自动清理监听器

---

## 生命周期

### 流程图

```
┌─────────────────────────────────────────────────────────┐
│                      游戏启动                            │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Load()                                                 │
│  ├── LoadConfig()      # 读取配置                        │
│  ├── enabled = willbeenabled                           │
│  └── if enabled → RegisterListeners()  # 注册事件       │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      游戏运行                            │
│  ├── Update()          # 每帧（需自己检查 enabled）      │
│  ├── FixedUpdate()     # 固定帧                         │
│  ├── OnGUI()           # IMGUI 绘制                     │
│  └── OnSceneLoaded()   # 场景切换时                     │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│  Mod 关闭 / 游戏退出                                     │
│  Unload()                                               │
│  ├── ClearAllListeners()  # 清理事件监听                 │
│  └── SaveConfig()         # 保存配置                     │
└─────────────────────────────────────────────────────────┘
```

### 生命周期方法详解

| 方法 | 调用时机 | 必须调 base |
|------|----------|-------------|
| `Load()` | Mod 加载时（游戏启动/重启后） | ✅ 是 |
| `Unload()` | Mod 卸载时（关闭/退出） | ✅ 是 |
| `RegisterListeners()` | Load 中自动调用（仅 enabled 时） | ❌ 否 |
| `Update()` | 每帧 | ❌ 否 |
| `FixedUpdate()` | 固定帧（物理相关） | ❌ 否 |
| `OnGUI()` | IMGUI 绘制时 | ❌ 否 |
| `OnSceneLoaded()` | 场景切换时 | ❌ 否 |
| `OnEnableChanged()` | UI 开关状态变化时 | ❌ 否 |
| `OnCreateUGUI()` | 创建配置界面时 | ❌ 否 |

> **注意**：`Update`、`FixedUpdate`、`OnGUI` 中需要自己检查 `enabled` 状态。

---

## 事件系统

### 监听事件

```csharp
public class MyEventMod : ModBase
{
    public override void RegisterListeners()
    {
        // 无参数
        Listen("Player.Jump", OnPlayerJump);
        
        // 一个参数
        Listen<int>("Player.TakeDamage", OnDamage);
        
        // 多个参数
        Listen<int, int>("Player.HealthChanged", OnHealthChanged);
        
        // Lambda 表达式
        Listen("Game.Start", () => MGCManager.DebugInfo.Add("游戏开始"));
        
        // Func 委托（有返回值）
        Listen<int, int, int>("Combat.CalculateDamage", (atk, def, crit) => (atk - def) * crit);
    }
    
    private void OnPlayerJump() { }
    private void OnDamage(int damage) { }
    private void OnHealthChanged(int oldHp, int newHp) { }
}
```

### 触发事件（游戏内部调用）

```csharp
// 无参数
ModEventSystem.Trigger("Player.Jump");

// 带参数
ModEventSystem.Trigger("Player.TakeDamage", 10);

// 带返回值
int result = ModEventSystem.TriggerWithReturn<int>("Combat.CalculateDamage", 50, 20, 2);
```

### 可用事件列表

> 📖 **完整事件列表请查看：[EventList.md](EventList.md)**

### 调试事件

```csharp
// 获取所有被监听的方法
List<string> methods = ModEventSystem.GetMonitoredMethods();

// 保存到文件
ModEventSystem.SaveMonitoredMethods();  // 保存到 Mods/MonitoredMethods.txt

// 清除所有事件（调试用）
ModEventSystem.ClearAllEvents();
```

---

## Mod 间通信

### 调用其他 Mod

```csharp
public class MyMod : ModBase
{
    public override void Load()
    {
        base.Load();
        
        // 方式一：事件调用（推荐，松耦合）
        Call("Economy.AddGold", 100);
        
        // 带返回值
        int gold = Call<int>("Economy.GetGold");
        
        // 方式二：直接获取实例
        var economy = GetMod<EconomyMod>("EconomyMod");
        economy?.AddGold(100);
    }
}
```

### 暴露方法给其他 Mod

```csharp
public class EconomyMod : ModBase
{
    private int _gold = 100;
    
    public override void RegisterListeners()
    {
        Listen<int>("Economy.AddGold", AddGold);
        Listen("Economy.GetGold", () => _gold);
        Listen<int, bool>("Economy.SpendGold", (amount, checkOnly) => {
            if (checkOnly) return _gold >= amount;
            if (_gold >= amount) { _gold -= amount; return true; }
            return false;
        });
    }
    
    public void AddGold(int amount) => _gold += amount;
}
```

---

## 配置系统

### 基础用法

```csharp
public class MyConfigMod : ModBase
{
    public override void Load()
    {
        base.Load();
        
        // 读取配置（带默认值）
        int volume = GetConfig("volume", 100);
        string name = GetConfig("username", "玩家");
        bool enabled = GetConfig("enabled", true);
        float speed = GetConfig("speed", 1.0f);
        
        // 保存配置
        SetConfig("volume", 80);
        SetConfig("username", "新名字");
        SetConfig("speed", 1.5f);
    }
    
    public override void Update(float deltaTime, GameObject player)
    {
        if (Input.GetKeyDown(KeyCode.F1))
        {
            SetConfig("speed", 2.0f);
            MGCManager.DebugInfo.Add("速度已改为 2.0");
        }
    }
}
```

### 配置方法

| 方法 | 说明 |
|------|------|
| `GetConfig(key, defaultValue)` | 读取配置 |
| `SetConfig(key, value)` | 保存配置 |
| `HasConfig(key)` | 检查是否存在 |
| `DeleteConfig(key)` | 删除单个配置 |
| `DeleteAllConfig()` | 删除所有配置 |

**支持的类型：** `int`、`float`、`bool`、`string`

> 配置自动保存在 `ModPlayerPrefs.xml` 中，无需手动管理文件。

---

## UI 系统

### 创建配置界面

```csharp
using MGC.UI.Controls;

public class MyUIMod : ModBase
{
    public override void OnCreateUGUI(Transform parent)
    {
        // 1. 标题
        UIControls.CreateCategoryTitle(parent, "我的模组设置", 40);
        
        // 2. 容器
        var container = UIControls.CreateValueContentVertical(parent).transform;
        
        // 3. 开关
        UIControls.CreateToggle(container, "启用功能", 
            GetConfig("enabled", true), 
            (isOn) => SetConfig("enabled", isOn), 
            false);
        
        // 4. 输入框
        UIControls.CreateInputField(container, "速度倍数", 
            GetConfig("speed", 1.0f).ToString(), 
            (val) => {
                if (float.TryParse(val, out float v)) 
                    SetConfig("speed", v);
            }, 
            false);
        
        // 5. 按钮
        UIControls.CreateButton(container, "重置", 
            () => {
                SetConfig("speed", 1.0f);
                SetConfig("enabled", true);
                MGCManager.DebugInfo.Add("已重置为默认值");
            }, 
            true);
        
        // 6. 说明文字
        UIControls.CreateContentText(container, "修改后即时生效", 
            30, new Color?(Color.yellow), FontStyle.Normal, false);
    }
}
```

### UI 控件一览

| 控件 | 方法 | 说明 |
|------|------|------|
| 标题 | `CreateCategoryTitle(parent, text, fontSize)` | 分组标题 |
| 开关 | `CreateToggle(parent, label, defaultValue, callback, useDefaultColors)` | 布尔值开关 |
| 输入框 | `CreateInputField(parent, label, defaultValue, callback, useDefaultColors)` | 文本/数字输入 |
| 按钮 | `CreateButton(parent, label, callback, useDefaultColors)` | 触发操作 |
| 文本 | `CreateContentText(parent, text, fontSize, color, fontStyle, useDefaultColors)` | 显示说明 |

> 所有控件都支持 `useDefaultColors` 参数，设为 `true` 时使用 Mod 面板的默认配色。

---

## 更新循环

### 每帧更新

```csharp
public class MyUpdateMod : ModBase
{
    private float _timer;
    
    public override void Update(float deltaTime, GameObject player)
    {
        if (!enabled) return;
        
        _timer += deltaTime;
        if (_timer >= 1f)
        {
            _timer = 0f;
            MGCManager.DebugInfo.Add("每秒执行一次");
        }
        
        if (player != null)
        {
            Vector3 pos = player.transform.position;
            // 处理逻辑...
        }
    }
}
```

### 固定帧更新（物理相关）

```csharp
public override void FixedUpdate(float deltaTime, GameObject player)
{
    if (!enabled) return;
    // 物理逻辑放在这里
}
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `deltaTime` | `float` | 上一帧耗时（秒） |
| `player` | `GameObject` | 玩家对象，可能为 `null` |

> ⚠️ **注意**：`Update`、`FixedUpdate`、`OnGUI` 中**不会自动检查** `enabled`，需要手动判断。

---

## 完整示例

### 示例一：简单计时器 Mod

```csharp
using MGC.ModSystem;
using UnityEngine;
using UnityEngine.UI;

public class SimpleTimerMod : ModBase
{
    public override string name => "简单计时器";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    public override string Description => "在屏幕上显示游戏时长";
    public override bool RequiresRestart => false;
    
    private Text _timerText;
    private float _currentTime;
    
    public override void Load()
    {
        base.Load();
        CreateUI();
        _currentTime = GetConfig("saved_time", 0f);
    }
    
    public override void Update(float deltaTime, GameObject player)
    {
        if (!enabled) return;
        
        _currentTime += deltaTime;
        SetConfig("saved_time", _currentTime);
        
        if (_timerText != null)
        {
            var time = System.TimeSpan.FromSeconds(_currentTime);
            _timerText.text = $"游玩时间: {time.Hours:D2}:{time.Minutes:D2}:{time.Seconds:D2}";
        }
    }
    
    private void CreateUI()
    {
        var canvas = GameObject.FindObjectOfType<Canvas>();
        if (canvas == null) return;
        
        var go = new GameObject("TimerText", typeof(Text));
        go.transform.SetParent(canvas.transform, false);
        _timerText = go.GetComponent<Text>();
        _timerText.font = Resources.GetBuiltinResource<Font>("Arial.ttf");
        _timerText.fontSize = 30;
        _timerText.color = Color.white;
        _timerText.alignment = TextAnchor.UpperLeft;
        
        var rect = _timerText.rectTransform;
        rect.anchorMin = new Vector2(0, 1);
        rect.anchorMax = new Vector2(0, 1);
        rect.anchoredPosition = new Vector2(10, -10);
    }
    
    public override void OnEnableChanged(bool newEnabled)
    {
        if (_timerText != null)
            _timerText.gameObject.SetActive(newEnabled);
    }
}
```

### 示例二：重力修改 Mod（需要重启）

```csharp
using MGC.ModSystem;
using MGC.UI.Controls;
using UnityEngine;

public class GravityMod : ModBase
{
    public override string name => "重力修改";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    public override bool RequiresRestart => true;
    
    public override void Load()
    {
        base.Load();
        float gravity = GetConfig("gravity", -9.81f);
        Physics2D.gravity = new Vector2(0, gravity);
        MGCManager.DebugInfo.Add($"[{name}] 重力已设置为 {gravity}");
    }
    
    public override void OnCreateUGUI(Transform parent)
    {
        var container = UIControls.CreateValueContentVertical(parent).transform;
        
        UIControls.CreateInputField(container, "重力值", 
            GetConfig("gravity", -9.81f).ToString(), 
            (val) => {
                if (float.TryParse(val, out float g))
                    SetConfig("gravity", g);
            }, false);
        
        UIControls.CreateContentText(container, "⚠️ 修改后需要重启游戏才能生效", 
            30, new Color?(Color.yellow), FontStyle.Normal, false);
    }
}
```

### 示例三：事件监听 Mod

```csharp
using MGC.ModSystem;
using UnityEngine;

public class EventDemoMod : ModBase
{
    public override string name => "事件演示";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    
    public override void RegisterListeners()
    {
        // 监听玩家受伤
        Listen<int>("Player.TakeDamage", (damage) => {
            MGCManager.DebugInfo.Add($"玩家受到 {damage} 点伤害");
        });
        
        // 监听场景切换
        Listen("ScreenFader.StartSceneRoutine", () => {
            MGCManager.DebugInfo.Add("场景正在切换...");
        });
        
        // 监听菜单开关
        Listen<bool>("MGC.UIEvents.ToggleMenu", (isActive) => {
            MGCManager.DebugInfo.Add($"菜单状态: {(isActive ? "开启" : "关闭")}");
        });
    }
}
```

### 示例四：Mod 间协作

```csharp
// EconomyMod.cs
public class EconomyMod : ModBase
{
    public override string name => "经济系统";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    
    private int _gold = 100;
    
    public override void RegisterListeners()
    {
        Listen<int>("Economy.AddGold", (amount) => {
            _gold += amount;
            MGCManager.DebugInfo.Add($"获得 {amount} 金币，现有 {_gold}");
        });
        
        Listen("Economy.GetGold", () => _gold);
    }
}

// ShopMod.cs
public class ShopMod : ModBase
{
    public override string name => "商店";
    public override string ModVersion => "1.0.0";
    public override string SupportedModpackVersion => "1.28.0";
    
    public override void Load()
    {
        base.Load();
        Call("Economy.AddGold", 50);
        int gold = Call<int>("Economy.GetGold");
        MGCManager.DebugInfo.Add($"当前金币: {gold}");
    }
}
```

---

## 常见问题

### Q1: Mod 加载后没效果？

| 检查项 | 说明 |
|--------|------|
| [ ] | `Load()` 中是否调用了 `base.Load()` |
| [ ] | `RequiresRestart = true` 的 Mod 是否重启了游戏 |
| [ ] | 查看 `MGCManager.DebugInfo` 中的错误信息 |
| [ ] | Mod 的 `SupportedModpackVersion` 与游戏版本匹配 |

### Q2: 事件监听不触发？

| 检查项 | 说明 |
|--------|------|
| [ ] | 事件名拼写是否正确（区分大小写） |
| [ ] | `enabled` 是否为 `true` |
| [ ] | 参数类型是否匹配（`int` 不能匹配 `float`） |
| [ ] | 是否在 `RegisterListeners()` 中注册了监听 |

### Q3: 配置没有保存？

| 检查项 | 说明 |
|--------|------|
| [ ] | 配置在 `Unload()` 时自动保存 |
| [ ] | 可手动调用 `SaveConfig()` 立即保存 |
| [ ] | 检查 `ModPlayerPrefs.xml` 文件是否存在 |

### Q4: 如何调试 Mod？

```csharp
// 输出调试信息
MGCManager.DebugInfo.Add("调试信息");

// 查看所有监听的事件
var methods = ModEventSystem.GetMonitoredMethods();
foreach (var m in methods)
    MGCManager.DebugInfo.Add($"监听: {m}");

// 保存到文件
ModEventSystem.SaveMonitoredMethods();
```

文件位置：`{持久化目录}/Mods/MonitoredMethods.txt`

### Q5: UI 控件不显示？

| 检查项 | 说明 |
|--------|------|
| [ ] | 是否在 `OnCreateUGUI` 中创建 |
| [ ] | `parent` 参数是否正确传递 |
| [ ] | 是否使用了正确的容器（`CreateValueContentVertical` 或 `CreateValueContentHorizontal`） |

### Q6: 如何让 Mod 支持热更新？

设置 `RequiresRestart = false`，然后在 `OnEnableChanged` 中处理 UI 显隐。

```csharp
public override void OnEnableChanged(bool newEnabled)
{
    if (_myUI != null)
        _myUI.SetActive(newEnabled);
}
```

### Q7: Update 中 player 参数为 null？

玩家对象可能还未生成或已被销毁，需要提前返回。

```csharp
public override void Update(float deltaTime, GameObject player)
{
    if (!enabled) return;
    if (player == null) return;
    // 安全使用 player
}
```

### Q8: 如何获取游戏中的其他组件？

```csharp
// 获取玩家控制器
var playerControl = GameAPI.playerControl;
if (playerControl != null)
{
    playerControl.TakeDamage(10);
}

// 获取设置管理器
var settings = GameAPI.settingsManager;
if (settings != null)
{
    // 使用 settings
}
```

---

## API 参考

### ModBase

| 方法 | 说明 |
|------|------|
| `Listen(methodName, callback)` | 监听事件 |
| `Call(methodName, args)` | 调用事件 |
| `Call<T>(methodName, args)` | 调用事件并获取返回值 |
| `GetMod<T>(modName)` | 获取其他 Mod 实例 |
| `GetConfig<T>(key, defaultValue)` | 读取配置 |
| `SetConfig(key, value)` | 保存配置 |
| `HasConfig(key)` | 检查配置是否存在 |
| `DeleteConfig(key)` | 删除配置 |
| `DeleteAllConfig()` | 删除所有配置 |
| `SaveConfig()` | 手动保存配置 |
| `LoadConfig()` | 手动加载配置 |

### ModEventSystem

| 方法 | 说明 |
|------|------|
| `Trigger(methodName, args)` | 触发事件（游戏内部） |
| `TriggerWithReturn<T>(methodName, args)` | 触发事件并获取返回值 |
| `GetMonitoredMethods()` | 获取所有被监听的方法 |
| `SaveMonitoredMethods()` | 保存监听方法列表到文件 |
| `LoadMonitoredMethods()` | 从文件加载监听方法列表 |
| `ClearAllEvents()` | 清除所有事件（调试用） |
| `HasListener(methodName)` | 检查是否有监听器 |

### GameAPI

| 属性 | 类型 | 说明 |
|------|------|------|
| `player` | `GameObject` | 玩家对象 |
| `canvas` | `GameObject` | 画布对象 |
| `settingsManager` | `SettingsManager` | 设置管理器 |
| `playerControl` | `PlayerControl` | 玩家控制器 |
| `narrator` | `Narrator` | 旁白控制器 |
| `screenFader` | `ScreenFader` | 屏幕淡入淡出 |
| `saviour` | `Saviour` | 存档管理器 |
| `cameraControl` | `CameraControl` | 摄像机控制器 |

### UIControls

| 方法 | 说明 |
|------|------|
| `CreateCategoryTitle(parent, text, fontSize)` | 创建分类标题 |
| `CreateValueContentVertical(parent)` | 创建垂直布局容器 |
| `CreateValueContentHorizontal(parent)` | 创建水平布局容器 |
| `CreateToggle(parent, label, defaultValue, callback, useDefaultColors)` | 创建开关 |
| `CreateInputField(parent, label, defaultValue, callback, useDefaultColors)` | 创建输入框 |
| `CreateButton(parent, label, callback, useDefaultColors)` | 创建按钮 |
| `CreateContentText(parent, text, fontSize, color, fontStyle, useDefaultColors)` | 创建文本 |

---

## 相关文档

- 📋 [更新日志](Changelog.md)
- 📖 [完整事件列表](EventList.md)
