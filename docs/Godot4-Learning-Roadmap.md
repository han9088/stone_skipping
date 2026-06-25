# Godot 4 学习路线 & PRD 实现指南

> 本文档基于 [PRD-Godot4.md](./PRD-Godot4.md)，将产品需求拆解为 **可执行的 Godot 4 学习阶段** 与 **对应实现方案**。  
> 目标：零基础到能独立完成 V1 MVP（iOS + Android 双端）。

---

## 0. 总览：PRD → Godot 4 模块映射

| PRD 模块 | Godot 4 实现要点 | 学习阶段 |
|----------|------------------|----------|
| 动作采集 | `Input.get_accelerometer()` / `get_gyroscope()` | 阶段 2 |
| 虚拟出手识别 | 自定义 FSM + 环形缓冲区 | 阶段 3 ★ |
| 物理模拟 | 纯 GDScript（不用 RigidBody 流体） | 阶段 4 |
| 震动调度 | `Input.vibrate_handheld()` | 阶段 5 |
| 轨迹存档 | Resource / JSON | 阶段 4 |
| 结果 UI | Control + Label/Button | 阶段 6 |
| 3D 回放 | Camera3D + 协程插值 | 阶段 7 |
| 水面特效 | GPUParticles3D / CPUParticles3D | 阶段 8 |
| 双端发布 | Export Presets | 阶段 9 |

### 推荐实现顺序

```
环境搭建 → 传感器读取 → 出手识别 → 物理+震动 → UI结果页 → 3D回放 → 特效 → 真机校准 → 打包
```

### 建议项目结构

```
stone_skip/
├── project.godot
├── scenes/
│   ├── gameplay.tscn       # 主流程：UI + 传感器 Autoload
│   └── replay.tscn         # 3D 回放场景
├── scripts/
│   ├── core/
│   │   ├── game_flow.gd    # 状态机
│   │   └── game_state.gd   # enum
│   ├── sensors/
│   │   ├── motion_capture.gd
│   │   └── motion_normalizer.gd
│   ├── throw/
│   │   ├── throw_detector.gd
│   │   └── throw_detection_config.gd  # Resource
│   ├── physics/
│   │   ├── stone_physics_engine.gd
│   │   ├── simulation_runner.gd
│   │   └── trajectory_store.gd
│   ├── haptics/
│   │   └── haptics_scheduler.gd
│   ├── replay/
│   │   ├── replay_controller.gd
│   │   └── camera_rig.gd
│   ├── fx/
│   │   └── water_fx_spawner.gd
│   └── ui/
│       ├── idle_panel.gd
│       ├── result_panel.gd
│       └── replay_hud.gd
├── resources/
│   └── throw_detection_default.tres
├── prefabs/
│   ├── stone.tscn
│   └── water_splash.tscn
└── autoload/
    └── game_bus.gd         # 信号总线（可选）
```

---

## 阶段 0：环境与基础（1–3 天）

### 要学什么

- 安装 **Godot 4.4+**（[godotengine.org](https://godotengine.org)）
- Editor：Scene 树、Inspector、FileSystem、运行场景
- GDScript 基础：信号（`signal`）、`_ready`、`_process`、`_physics_process`
- 节点与场景（`.tscn`）概念

### 要做什么

1. 新建 **3D** 项目，命名 `stone_skip`
2. **Project → Project Settings**：

```
Application → Config → Name: Stone Skip

Input Devices → Sensors:
  enable_accelerometer = true
  enable_gyroscope = true
  enable_gravity = true

Rendering → Renderer → Rendering Method (Mobile): mobile
```

3. **Editor → Manage Export Templates** 下载导出模板
4. 创建 `scenes/gameplay.tscn`：根节点 `Control` 全屏 + 一个 `Label`

### 验收标准

- [ ] 项目 F5 运行无报错
- [ ] 了解 Scene / Node / Script 关系
- [ ] Export Templates 已安装

### 学习资源

- [Godot 4 官方文档](https://docs.godotengine.org/en/stable/)
- [Your first 2D game](https://docs.godotengine.org/en/stable/getting_started/first_2d_game/index.html)（熟悉 GDScript 即可，本项目是 3D）

---

## 阶段 1：游戏状态与流程骨架（1–2 天）

> 对应 PRD §2.1 两阶段体验

### 要学什么

- 枚举 + 状态机
- Autoload 单例（`Project → Project Settings → Autoload`）
- 信号驱动解耦（`game_flow.state_changed`）

### 要做什么

**`scripts/core/game_state.gd`**

```gdscript
enum State {
    IDLE,        # 待机
    RECORDING,   # 采集中
    SIMULATING,  # 实时模拟 + 震动
    RESULT,      # 结果页
    REPLAY       # 3D 回放（无震动）
}
```

**`scripts/core/game_flow.gd`**（Autoload）：

```
IDLE → RECORDING → SIMULATING → RESULT → REPLAY → IDLE
```

用键盘调试：

- `Space`：模拟进入 SIMULATING
- `R`：进入 REPLAY
- `Esc`：回 IDLE

### 验收标准

- [ ] 状态切换有 Log 输出
- [ ] Result 面板有「查看回放」「再玩一次」
- [ ] **REPLAY 状态不调用震动**

---

## 阶段 2：动作采集 — Motion Capture（2–3 天）

> 对应 PRD §3.1

### 要学什么

- `Input` 传感器 API
- `_physics_process` 固定帧率采样
- 环形缓冲区（`Array` + 固定容量）

### 核心 API

```gdscript
func _physics_process(_delta: float) -> void:
    var accel := Input.get_accelerometer()   # m/s²，桌面为 ZERO
    var gyro  := Input.get_gyroscope()      # rad/s
    var gravity := Input.get_gravity()
    _buffer.push(MotionSample.new(Time.get_ticks_msec() / 1000.0, accel, gyro))
```

### 要做什么

1. **`MotionSample`** 类：time, acceleration, angular_velocity
2. **`MotionCapture`**（Node）：
   - 环形缓冲容量 ≈ 120（2s × 60Hz）
   - `get_window() -> Array[MotionSample]`
3. **`MotionNormalizer`**：
   - 线性加速度 ≈ `accel - gravity`（或直接用 magnitude）
   - 记录 `OS.get_model_name()` 用于校准日志

### Editor 调试

**`MockMotionProvider`**：按键生成假甩动曲线，无真机也能测后续模块

```gdscript
# 按住 W 模拟甩动峰值
if Input.is_key_pressed(KEY_W):
    return Vector3(15, 2, 8)
```

### 验收标准

- [ ] 真机 `print(accel)` 有连续输出
- [ ] 缓冲区 2s 内正确滚动
- [ ] iOS / Android 归一化后量级接近

---

## 阶段 3：虚拟出手识别 — Throw Detection（3–5 天）★

> 对应 PRD §3.2

### 要学什么

- 有限状态机
- 峰值检测 + 阈值
- Custom Resource（`.tres`）存配置

### 状态机

```
Idle ──(|a| > start)──→ Recording
Recording ──(peak + drop)──→ ReleaseDetected
Recording ──(timeout)──→ Idle
ReleaseDetected ──(pose ok)──→ emit throw_detected
ReleaseDetected ──(pose fail)──→ Idle + Toast
```

### Resource 配置

**`throw_detection_config.gd`** extends Resource：

```gdscript
@export var start_threshold: float = 12.0
@export var min_peak: float = 18.0
@export var drop_ratio: float = 0.55
@export var window_timeout: float = 2.0
@export var practice_mode: bool = false
```

### 判定伪代码

```gdscript
var mag := sample.acceleration.length()

if state == IDLE and mag > config.start_threshold:
    _enter_recording()

if state == RECORDING:
    if mag > _peak: _peak = mag; _peak_idx = i
    if _peak > config.min_peak and mag < _peak * config.drop_ratio:
        _enter_release(_peak_idx)
```

### 参数提取 → `ThrowParams`

| 字段 | 来源 |
|------|------|
| velocity | 峰值窗口积分 |
| angle | 设备倾斜（需结合 gravity 算 pitch） |
| spin | gyro.y |
| side_spin | gyro.x |

### 验收标准

- [ ] 10 次甩动 ≥ 8 次正确触发
- [ ] 走路 / 放下手机不误触发
- [ ] 发出 `throw_detected(params)` 信号

---

## 阶段 4：简化物理模拟 + 轨迹（2–4 天）

> 对应 PRD §3.3

### 要学什么

- 纯 GDScript 算法
- `await get_tree().create_timer(delta).timeout` 时间线
- `RefCounted` / 自定义数据类

### 核心类

```gdscript
class_name TrajectoryPoint
var time: float
var position: Vector3
var is_water_contact: bool

class_name SimulationResult
var skips: int
var trajectory: Array[TrajectoryPoint]
var score: int
var rating: String
```

**`StonePhysicsEngine.simulate(params) -> SimulationResult`**：

- 同步计算完整轨迹（毫秒级）
- 入水角 < 20° && speed > min → 水漂循环
- 每次触水 `skips += 1`，`speed *= decay`

### 实时模拟（驱动震动）

**`SimulationRunner.gd`**：

```gdscript
func run_realtime(result: SimulationResult) -> void:
    var last_time := 0.0
    for pt in result.trajectory:
        if not pt.is_water_contact:
            continue
        var wait := pt.time - last_time
        await get_tree().create_timer(wait).timeout
        GameFlow.ensure_state(GameState.SIMULATING)
        HapticsScheduler.on_water_contact()
        last_time = pt.time
    await get_tree().create_timer(0.3).timeout  # 最后一次震后停顿
    TrajectoryStore.save(result)
    GameFlow.change_state(GameState.RESULT)
```

### 验收标准

- [ ] 不同参数 → 不同 skips
- [ ] 触水时间点与震动一致
- [ ] Result 显示 Skips / Score / Rating

---

## 阶段 5：震动反馈 — Haptics（1 天）

> 对应 PRD §3.6

### 要学什么

- `Input.vibrate_handheld(duration_ms, amplitude)`
- 导出预设权限

### 实现

**`haptics_scheduler.gd`**

```gdscript
func on_release_confirmed() -> void:
    _pulse(45)

func on_water_contact() -> void:
    if GameFlow.state != GameState.SIMULATING:
        return
    _pulse(35)

func _pulse(ms: int) -> void:
    Input.vibrate_handheld(ms, 0.5)
```

### 导出配置

**Android Export Preset** → Permissions → ✅ `VIBRATE`

> Godot 4.4+ 未开 VIBRATE 调用震动可能崩溃，务必勾选。

### 验收标准

- [ ] 出手 1 震 + N 落点各 1 震
- [ ] REPLAY / RESULT 全程无震动
- [ ] 15 连漂节奏清晰

---

## 阶段 6：UI 与结果页（1–2 天）

> 对应 PRD §3.7

### 要学什么

- Control 布局：`VBoxContainer`、`MarginContainer`
- Theme / Label 样式
- Button 信号连接

### 页面

| 节点 | 内容 |
|------|------|
| `IdlePanel` | 「甩动手机开始」+ 练习模式 CheckBox |
| `SimulatingPanel` | 极简 Logo / 隐藏 |
| `ResultPanel` | Skips、Score、Rating、「查看回放」「再玩一次」 |

**「查看回放」**：

```gdscript
func _on_replay_pressed() -> void:
    get_tree().change_scene_to_file("res://scenes/replay.tscn")
```

### 验收标准

- [ ] PRD 流程 1–5 步 UI 完整
- [ ] Rating 规则可配置

---

## 阶段 7：3D 回放系统（3–5 天）

> 对应 PRD §3.4

### 要学什么

- 3D 场景：`MeshInstance3D`、`DirectionalLight3D`、`WorldEnvironment`
- `Camera3D`、`SpringArm3D`
- `Engine.time_scale` 慢动作

### 场景搭建

```
Replay (Node3D)
├── Water (MeshInstance3D, Plane)
├── Stone (MeshInstance3D)
├── CameraRig (SpringArm3D → Camera3D)   # Follow
├── SideCamera (Camera3D)                # Side，默认 current=false
└── ReplayController (Script)
```

### 镜头切换

```gdscript
func set_follow_mode() -> void:
    follow_cam.current = true
    side_cam.current = false

func set_side_mode() -> void:
    follow_cam.current = false
    side_cam.current = true
```

### ReplayController

```gdscript
func play() -> void:
    var traj := TrajectoryStore.get_last().trajectory
    var t := 0.0
    for pt in traj:
        var dt := pt.time - t
        await get_tree().create_timer(dt / Engine.time_scale).timeout
        stone.global_position = pt.position
        if pt.is_water_contact:
            water_fx.spawn(pt.position)  # 不调 Haptics
        t = pt.time
```

### 验收标准

- [ ] 轨迹与模拟数据一致
- [ ] Follow / Side 可切换
- [ ] 回放零震动

---

## 阶段 8：水面特效 — Water FX（2–3 天）

> 对应 PRD §3.5（仅回放）

### 要学什么

- [GPUParticles3D](https://docs.godotengine.org/en/stable/classes/class_gpuparticles3d.html)
- [CPUParticles3D](https://docs.godotengine.org/en/stable/classes/class_cpuparticles3d.html)（兼容降级）
- 对象池：`Array` 复用或简单 `instantiate` + `queue_free`

### 特效

| 特效 | 节点 |
|------|------|
| Splash | GPUParticles3D，`one_shot = true`，触水 `emitting = true` |
| Ripple | 圆环 Mesh + `Tween`：`scale` 0→3，`modulate.a` 1→0 |
| Foam | 可选 CPUParticles3D |

### 移动端

- Export → 使用 **Mobile** 渲染器
- 老设备：Particles → Convert to CPUParticles3D

### 验收标准

- [ ] 每次触水有水花 + 波纹
- [ ] 15 次触水移动端 ≥ 30fps

---

## 阶段 9：双端打包与真机校准（3–5 天）

> 对应 PRD §5、§6

### Android

1. 安装 OpenJDK 17、Android SDK
2. **Editor Settings** → Android SDK Path
3. **Project → Export → Add Android**
4. 勾选 VIBRATE、配置 Keystore
5. Export APK/AAB → 真机安装

文档：[Exporting for Android](https://docs.godotengine.org/en/stable/tutorials/export/exporting_for_android.html)

### iOS

1. **必须在 macOS** 上操作
2. Export → iOS → 生成 Xcode 工程
3. Xcode 配置 Team ID、Bundle ID、签名
4. Run on Device / TestFlight

文档：[Exporting for iOS](https://docs.godotengine.org/en/stable/tutorials/export/exporting_for_ios.html)

### 校准流程

1. 2–3 台 iOS + 2–3 台 Android
2. 日志导出 `ThrowParams` + skips → CSV
3. 调整 `throw_detection_default.tres`
4. 微调 `vibrate_handheld` 时长（35–50ms）

### 验收标准

- [ ] PRD §6 MVP 全部完成
- [ ] 双端各 2 台真机通过全流程

---

## 阶段 10：Polish 与 V2 预备

| PRD V2 | Godot 方向 |
|--------|------------|
| 风力 | 物理引擎加 `wind_bias` |
| 水面波动 | Shader 顶点位移 |
| 石头类型 | Resource 配置 mass / shape |
| 精细震动 | GDExtension 原生插件 |

---

## 学习路线时间表（参考）

| 周次 | 阶段 | 里程碑 |
|------|------|--------|
| 第 1 周 | 0–1 | 项目骨架 + 状态机 |
| 第 2 周 | 2–3 | 真机传感器 + 出手识别 |
| 第 3 周 | 4–5 | 物理 + 震动闭环（**核心可玩**） |
| 第 4 周 | 6–7 | UI + 3D 回放 |
| 第 5 周 | 8–9 | 特效 + 打包 + 校准 |

> 有 Godot 基础可压至 2–3 周；零基础按 5–6 周。

---

## 关键原则

1. **先闭环，后美化**：阶段 4–5 完成即可真机「甩 → 震 → 结果」。
2. **阈值外置 Resource**：不要硬编码在脚本里。
3. **回放与震动分离**：`replay_controller.gd` / `water_fx_spawner.gd` 禁止 import `haptics_scheduler`。
4. **Mock 优先**：桌面用键盘模拟传感器，减少真机往返。
5. **不用真实流体**：参数模型足够，别陷进 Shader 深水区。

---

## 推荐官方文档索引

| 主题 | 链接 |
|------|------|
| Input 传感器 | https://docs.godotengine.org/en/stable/classes/class_input.html |
| GPUParticles3D | https://docs.godotengine.org/en/stable/classes/class_gpuparticles3d.html |
| Camera3D | https://docs.godotengine.org/en/stable/classes/class_camera3d.html |
| GDScript | https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/index.html |
| Mobile 渲染 | https://docs.godotengine.org/en/stable/tutorials/3d/3d_rendering_limitations.html |
| Android 导出 | https://docs.godotengine.org/en/stable/tutorials/export/exporting_for_android.html |
| iOS 导出 | https://docs.godotengine.org/en/stable/tutorials/export/exporting_for_ios.html |

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

*文档版本：与 PRD-Godot4 V1 同步 · 最后更新：2026-06*
