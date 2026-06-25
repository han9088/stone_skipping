# Unity 学习路线 & PRD 实现指南

> 本文档基于 [PRD.md](./PRD.md)，将产品需求拆解为 **可执行的 Unity 学习阶段** 与 **对应实现方案**。  
> 目标：零基础到能独立完成 V1 MVP（iOS + Android 双端）。

---

## 0. 总览：PRD → Unity 模块映射

| PRD 模块 | Unity 实现要点 | 学习阶段 |
|----------|----------------|----------|
| 动作采集 | Input System 传感器 | 阶段 2 |
| 虚拟出手识别 | 自定义状态机 + 滑动窗口 | 阶段 3 ★ |
| 物理模拟 | 纯 C# 参数模型（非 Rigidbody 流体） | 阶段 4 |
| 震动调度 | `Handheld.Vibrate` + 原生插件 | 阶段 5 |
| 轨迹存档 | ScriptableObject / JSON | 阶段 4 |
| 结果 UI | UGUI / UI Toolkit | 阶段 6 |
| 3D 回放 | Cinemachine + 时间轴驱动 | 阶段 7 |
| 水面特效 | URP + Particle System | 阶段 8 |
| 双端发布 | Build Settings + IL2CPP | 阶段 9 |

### 推荐实现顺序

```
环境搭建 → 传感器读取 → 出手识别 → 物理+震动 → UI结果页 → 3D回放 → 特效 → 真机校准 → 打包
```

### 建议项目结构

```
Assets/
├── Scripts/
│   ├── Core/           # GameState、事件总线
│   ├── Sensors/        # MotionCapture、坐标归一化
│   ├── ThrowDetection/ # 状态机、滑动窗口
│   ├── Physics/        # StonePhysicsEngine、Trajectory
│   ├── Haptics/        # HapticsScheduler、平台适配
│   ├── Replay/         # ReplayController、CameraSwitch
│   ├── FX/             # Splash、Ripple 触发器
│   └── UI/             # 待机、结果、回放入口
├── Scenes/
│   ├── Boot.unity
│   ├── Gameplay.unity      # 极简 UI + 传感器逻辑
│   └── Replay.unity        # 3D 场景（可选同场景不同 State）
├── Prefabs/
│   ├── Stone.prefab
│   └── WaterSplash.prefab
└── Settings/           # URP Asset、Input Actions
```

---

## 阶段 0：环境与基础（1–3 天）

### 要学什么

- Unity Hub 安装 **Unity 6**（6000.x LTS 或当前稳定版）
- Editor 基本操作：Scene / Game / Hierarchy / Inspector / Project
- C# 基础：类、接口、协程（`Coroutine`）、事件（`Action` / `UnityEvent`）
- 移动平台模块：Android Build Support、iOS Build Support

### 要做什么

1. 新建 **3D (URP)** 项目，命名 `StoneSkip`
2. 安装 Package Manager 依赖：

| Package | 用途 |
|---------|------|
| Input System | 加速度计 / 陀螺仪 |
| Cinemachine | 回放镜头 |
| Universal RP | 渲染（项目模板已含） |

3. `Edit → Project Settings → Player`：
   - 勾选 **Accelerometer Frequency → 100Hz**（iOS 有效，Android 尽力）
   - Active Input Handling → **Input System Package**
4. 创建空场景 `Gameplay`，一个 Directional Light，一个 Canvas（Screen Space Overlay）

### 验收标准

- [ ] 项目能在 Editor Play 模式运行
- [ ] Package 无报错
- [ ] 了解 Scene 与 Prefab 概念

### 学习资源

- Unity Learn：[Get Started with Unity](https://learn.unity.com/pathway/unity-essentials)
- 官方文档：[Input System](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest)

---

## 阶段 1：游戏状态与流程骨架（1–2 天）

> 对应 PRD §2.1 两阶段体验、§4 架构

### 要学什么

- 简单 **状态机** 模式（`enum GameState` + `switch` 或 State 类）
- `DontDestroyOnLoad` 单例 / ScriptableObject 事件通道
- 场景内 UI 切换（不依赖复杂框架）

### 要做什么

定义状态流转，与 PRD 时序对齐：

```csharp
public enum GameState
{
    Idle,           // 待机，等待甩动
    Recording,      // 甩动采集中
    Simulating,     // 实时物理 + 震动（主玩法）
    Result,         // 展示 Skips / Score
    Replay          // 可选 3D 回放（无震动）
}
```

实现 `GameFlowController`：

```
Idle → (检测到甩动) → Recording
Recording → (虚拟出手) → Simulating
Simulating → (模拟结束) → Result
Result → (用户点击回放) → Replay
Replay → (回放结束) → Idle
```

### 验收标准

- [ ] Editor 中可用键盘模拟状态切换（后续替换为真实传感器）
- [ ] Result 页有「查看回放」「再玩一次」按钮
- [ ] **Replay 状态不调用 Haptics**

---

## 阶段 2：动作采集 — Motion Capture（2–3 天）

> 对应 PRD §3.1

### 要学什么

- Input System：**Sensors** 用法
- `Accelerometer.current`、`Gyroscope.current`
- 传感器启用 / 禁用、采样频率
- 坐标系转换（设备坐标 → Unity 世界坐标）

### 核心 API

```csharp
// Input Actions 或直接 API
Accelerometer.current?.Enable();
Gyroscope.current?.Enable();

var accel = Accelerometer.current.ReadValue();  // Vector3 m/s²
var gyro  = Gyroscope.current.ReadValue();      // Vector3 rad/s
```

### 要做什么

1. **`MotionSample`** 数据结构：

```csharp
struct MotionSample
{
    public float Time;
    public Vector3 Acceleration;
    public Vector3 AngularVelocity;
}
```

2. **`MotionCaptureService`**：
   - `FixedUpdate` 或 `InputSystem.onBeforeUpdate` 中以最高频率采样
   - 维护 **环形缓冲区**（容量 ≈ 2s × 100Hz = 200 条）
   - 输出 `IReadOnlyList<MotionSample> GetWindow()`

3. **`MotionNormalizer`**（跨平台）：
   - 统一使用 **用户坐标系**（Y 向上、Z 为投掷方向）
   - 减去重力分量：`linearAccel = rawAccel - Physics.gravity`
   - 记录设备型号用于后续校准（`SystemInfo.deviceModel`）

### Editor 调试技巧

- 无真机时用 **Input Debugger**（Window → Analysis → Input Debugger → Sensors）
- 或写 `MockMotionProvider`：按键模拟甩动曲线

### 验收标准

- [ ] 真机 Log 输出连续 accel / gyro 数值
- [ ] 环形缓冲区在 2s 窗口内正确滚动
- [ ] iOS / Android 同一甩动动作，归一化后量级接近

---

## 阶段 3：虚拟出手识别 — Throw Detection（3–5 天）★ 核心难点

> 对应 PRD §3.2

### 要学什么

- 信号处理基础：滑动窗口、峰值检测、阈值
- 有限状态机（FSM）实现
- 数据可视化调试（Editor 曲线 / 真机 Log）

### 状态机设计

```
Idle ──(|a| > startThreshold)──→ Recording
Recording ──(peak + sharpDrop)──→ ReleaseDetected
Recording ──(timeout 2s)────────→ Idle（无效）
ReleaseDetected ──(姿态校验通过)──→ TriggerSimulate
ReleaseDetected ──(姿态不符)──────→ Idle（无效，Toast 提示）
```

### 判定算法（V1）

```csharp
// 伪代码
float magnitude = sample.Acceleration.magnitude;

// 1. 甩动开始
if (state == Idle && magnitude > startThreshold)
    EnterRecording();

// 2. 追踪峰值
if (state == Recording)
{
    if (magnitude > peakValue) { peakValue = magnitude; peakIndex = i; }

    // 3. 峰值后急剧回落 → 虚拟出手
    if (peakValue > minPeak && magnitude < peakValue * dropRatio)
        EnterReleaseDetected(peakIndex);
}

// 4. 姿态确认：pitch 入水角、gyro 水平分量
if (state == ReleaseDetected && ValidatePose(peakIndex))
    ExtractThrowParams(peakIndex);  // velocity, angle, spin, sideSpin
```

### 参数提取

| PRD 参数 | 计算方式 |
|----------|----------|
| velocity | 峰值前 100ms 窗口内 `\|a\|` 积分或峰值映射到 0–1 再缩放 |
| angle | 出手时刻 `Quaternion` 的 pitch（设备前倾角） |
| spin | `gyro.y`（绕垂直轴） |
| sideSpin | `gyro.x` 或 `gyro.z` |

### 要做什么

1. `ThrowDetector` 订阅 `MotionCaptureService`
2. 所有阈值放 **`ThrowDetectionConfig` ScriptableObject**，便于真机调参
3. 无效动作 UI：`「再试一次」` 轻提示，回到 Idle
4. **练习模式**：降低 `startThreshold` / `minPeak`

### 验收标准

- [ ] 10 次真实甩动，≥ 8 次正确触发（真机迭代阈值）
- [ ] 走路 / 放下手机不会误触发
- [ ] 触发后输出 `ThrowParams` 结构体，供物理引擎消费

---

## 阶段 4：简化物理模拟 + 轨迹生成（2–4 天）

> 对应 PRD §3.3

### 要学什么

- 纯 C# 逻辑（**不用** Unity Physics 流体）
- 协程 / `async` 时间线调度
- 数据结构：`List<TrajectoryPoint>`

### 核心模型

```csharp
struct TrajectoryPoint
{
    public float Time;      // 从出手起算
    public Vector3 Position;
    public bool IsWaterContact;
}

class StonePhysicsEngine
{
    public SimulationResult Simulate(ThrowParams p)
    {
        // 入水角 < 20° 且 speed > minSpeed → 可水漂
        // 循环：触水 → skips++ → speed *= decay → 弹起
        // speed < sinkThreshold → 结束
    }
}
```

### 两种运行模式（PRD 要求）

| 模式 | 实现 |
|------|------|
| **实时模拟** | 预计算完整 `Trajectory`，用协程按 `Time` 依次 `yield return WaitForSeconds(delta)`，每个触水点发事件 |
| **回放渲染** | 读取已存 `Trajectory`，驱动石头 Transform 插值移动 |

```csharp
// 实时模拟驱动震动
IEnumerator RunRealtimeSimulation(SimulationResult result)
{
    foreach (var pt in result.Trajectory.Where(p => p.IsWaterContact))
    {
        yield return new WaitForSeconds(pt.Time - lastTime);
        OnWaterContact?.Invoke(pt);  // → HapticsScheduler
        lastTime = pt.Time;
    }
    OnSimulationComplete?.Invoke(result);
}
```

### 要做什么

1. `StonePhysicsEngine.Simulate()` 同步算出完整轨迹（毫秒级，可放主线程）
2. `SimulationRunner` 协程播放时间线
3. `TrajectoryStore` 缓存最近一次结果，供回放 Scene 读取
4. `ScoringSystem`：基于 skips + angle + velocity 算 0–100 分与 S/A/B/C

### 验收标准

- [ ] 不同参数产生不同 skips 次数
- [ ] 协程触水时间点与轨迹数据一致
- [ ] 模拟结束后 Result 页显示正确 Skips / Score / Rating

---

## 阶段 5：震动反馈 — Haptics Scheduler（1–2 天）

> 对应 PRD §3.6

### 要学什么

- Unity 基础：`Handheld.Vibrate()`（双端通用但较粗糙）
- 可选原生插件：iOS Core Haptics、Android `VibrationEffect`
- 事件订阅模式

### V1 实现（最快路径）

```csharp
class HapticsScheduler : MonoBehaviour
{
    public void OnReleaseConfirmed() => Pulse(40);  // 出手 1 次

    public void OnWaterContact(TrajectoryPoint pt) => Pulse(35);  // 每落点 1 次

    void Pulse(int durationMs)
    {
        if (GameState != Simulating) return;  // 回放绝不震动
#if UNITY_IOS || UNITY_ANDROID
        Handheld.Vibrate();  // MVP 先用内置 API
#endif
    }
}
```

### V1.1 增强（可选）

| 平台 | 方案 |
|------|------|
| Android | 写 Java 插件 `VibrationEffect.createOneShot(ms, amplitude)` |
| iOS | 写 Objective-C++ 插件 `UIImpactFeedbackGenerator` |

使用 **`.androidlib` / `.mm` 插件** 或 Asset Store 的 Nice Vibrations 等库。

### 要做什么

1. 仅 `GameState.Simulating` 时响应 `OnWaterContact`
2. `Replay` / `Result` / `Idle` 状态忽略震动
3. 最后一次落点震后 **延迟 300ms** 再切 Result（PRD：短暂停顿）

### 验收标准

- [ ] 真机：出手震 1 次 + N 次水漂各震 1 次
- [ ] 3D 回放全程无震动
- [ ] 15 连漂节奏可感知，无叠震

---

## 阶段 6：UI 与结果页（1–2 天）

> 对应 PRD §3.7、§2.1 阶段一结束

### 要学什么

- UGUI：Canvas、TextMeshPro、Button
- 简单动效：DOTween（可选）或 Animator

### 页面结构

| 页面 | 内容 |
|------|------|
| 待机 | 「甩动手机开始」+ 可选练习模式开关 |
| 模拟中 | 极简（黑屏或 logo），可不显示画面 |
| 结果 | Skips、Score、Rating(S/A/B/C)、「查看回放」「再玩一次」 |
| 回放 | 全屏 3D，角落「跳过 / 返回」 |

### 验收标准

- [ ] 完整走通 PRD 用户流程 1–5 步
- [ ] Rating 分级规则可配置

---

## 阶段 7：3D 回放系统（3–5 天）

> 对应 PRD §3.4

### 要学什么

- Cinemachine 3：Virtual Camera、Follow、Blend
- Timeline（可选）或手写插值驱动
- `Time.timeScale` 慢动作

### 场景搭建

1. **环境**：Plane 水面 + 简单 Skybox + URP 水体 Shader（或免费 Water 素材）
2. **石头 Prefab**：低面数 Mesh，Replay 时沿 Trajectory 移动
3. **Cinemachine 镜头**：
   - `CM Follow`：`CinemachineFollow` 跟随石头
   - `CM Side`：固定侧面远景
   - 回放开始默认 Follow，UI 可切换 Side

### ReplayController

```csharp
IEnumerator PlayReplay(Trajectory trajectory)
{
    Time.timeScale = 1f;
    foreach (var pt in trajectory)
    {
        stone.position = pt.Position;
        if (pt.IsWaterContact)
            splashFX.Play(pt.Position);  // 仅视觉，不调 Haptics
        yield return null;  // 或按 pt.Time 等待
    }
}
```

### 慢动作

- UI 按钮切换 `Time.timeScale = 0.3f`
- 或使用 Cinemachine **Speed Multiplier**

### 验收标准

- [ ] 回放轨迹与实时模拟数据一致
- [ ] 至少 2 种镜头可切换
- [ ] 回放无震动（再次确认）

---

## 阶段 8：水面特效 — Water FX（2–3 天）

> 对应 PRD §3.5（仅回放模式）

### 要学什么

- Particle System：Burst、Shape、Color over Lifetime
- URP Decal / 简单 Ripple 平面缩放动画
- Object Pooling（`UnityEngine.Pool`）

### 特效拆分

| 特效 | 实现 |
|------|------|
| Splash | 粒子 Burst，触水时 `Instantiate` 或 Pool Get |
| Ripple | 圆环 Mesh 或 Decal，`scale` 0→1 + alpha 淡出 |
| Foam | 可选，粒子 Sub Emitter |

### 要做什么

1. `WaterFxSpawner.OnWaterContact(Vector3 pos)` 仅 Replay 状态调用
2. 对象池避免 GC 卡顿

### 验收标准

- [ ] 每次触水有明显水花 + 波纹
- [ ] 连续 15 次触水帧率稳定（移动端 ≥ 30fps）

---

## 阶段 9：双端打包与真机校准（3–5 天）

> 对应 PRD §5、§6、§10

### 要学什么

- Android：Keystore、AAB、Gradle
- iOS：Apple Developer、Provisioning、Xcode 导出
- IL2CPP、ARM64
- 真机 Profiler / Logcat / Xcode Console

### 构建 checklist

**Android**

```
File → Build Settings → Android
Scripting Backend: IL2CPP
Target Architectures: ARM64
Player → Other → VIBRATE permission（Manifest）
Build AAB
```

**iOS**

```
File → Build Settings → iOS → Build
Xcode → Signing → Run on Device
Info.plist: NSMotionUsageDescription（若需要）
TestFlight 内测
```

### 校准流程

1. 准备 2–3 台 iOS + 2–3 台 Android 真机
2. 导出每次 `ThrowParams` + skips 到 CSV
3. 调整 `ThrowDetectionConfig` 阈值直到误判率可接受
4. 震动时长 / 间隔在不同机型试听微调

### 验收标准

- [ ] PRD §6 MVP 全部 checkbox 完成
- [ ] 双端各至少 2 台真机通过完整流程

---

## 阶段 10： polish 与 V2 预备（持续）

| PRD V2 项 | Unity 方向 |
|-----------|------------|
| 风力 | 物理引擎加横向力参数 |
| 水面波动 | Shader Graph 顶点动画 |
| 石头类型 | ScriptableObject 配置质量 / 形状 |
| 旋转系统 | 轨迹加 spin 影响弹跳角 |
| 评分优化 | 机器学习或更多特征权重 |

---

## 学习路线时间表（参考）

| 周次 | 阶段 | 里程碑 |
|------|------|--------|
| 第 1 周 | 0–1 | 项目骨架 + 状态机跑通 |
| 第 2 周 | 2–3 | 真机采传感器 + 出手识别初版 |
| 第 3 周 | 4–5 | 物理模拟 + 震动闭环（**核心玩法可玩**） |
| 第 4 周 | 6–7 | 结果 UI + 3D 回放 |
| 第 5 周 | 8–9 | 特效 + 双端打包 + 校准 |

> 有 Unity 基础可压缩至 2–3 周；完全零基础按 5–6 周规划。

---

## 关键原则（贯穿全程）

1. **先闭环，后美化**：阶段 4–5 完成即可在真机「甩 → 震 → 结果」，回放与特效后补。
2. **阈值外置**：所有检测参数放 ScriptableObject，不要硬编码。
3. **回放与震动分离**：任何 FX / Replay 代码不得调用 `HapticsScheduler`。
4. **Editor 可测**：Mock 传感器 + 键盘触发，减少真机依赖。
5. **不用真实流体**：PRD 明确参数模型即可，避免 Shader / Fluid 坑。

---

## 推荐 Unity 官方文档索引

| 主题 | 链接 |
|------|------|
| Input System Sensors | https://docs.unity3d.com/Packages/com.unity.inputsystem@latest/manual/Sensors.html |
| Cinemachine | https://docs.unity3d.com/Packages/com.unity.cinemachine@latest |
| URP | https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest |
| Particle System | https://docs.unity3d.com/Manual/ParticleSystems.html |
| Mobile optimization | https://docs.unity3d.com/Manual/MobileOptimizationPracticalGuide.html |
| Android build | https://docs.unity3d.com/Manual/android-BuildProcess.html |
| iOS build | https://docs.unity3d.com/Manual/iphone-GettingStarted.html |

---

## 与 PRD MVP Checklist 对照

| PRD MVP 项 | 对应阶段 |
|------------|----------|
| 虚拟出手识别 | 阶段 3 |
| 动作参数提取 | 阶段 2 + 3 |
| 实时物理模拟 + 落点震动 | 阶段 4 + 5 |
| 水漂次数计算 | 阶段 4 |
| 轨迹生成与存档 | 阶段 4 |
| 结果页展示 | 阶段 6 |
| 可选 3D 回放 | 阶段 7 |
| 回放水花效果 | 阶段 8 |
| 双端真机测试 | 阶段 9 |

---

*文档版本：与 PRD V1 同步 · 最后更新：2026-06*
