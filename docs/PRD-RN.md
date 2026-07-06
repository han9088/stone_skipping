# 📄 PRD：打水漂体感模拟器（React Native 版）

## 1. 产品概述

### 1.1 产品名称

Stone Skip / 打水漂模拟器

### 1.2 产品定位

基于手机体感（加速度计 + 陀螺仪）的投掷模拟 App。  
用户手持手机模拟打水漂甩动动作，系统识别「虚拟出手」后运行 **Rapier 3D 刚体物理模拟**，通过 **震动节奏** 感知水漂次数，无需盯屏。

### 1.3 支持平台

| 平台 | 最低版本 | 说明 |
|------|----------|------|
| **iOS** | iOS 14+ | iPhone，需加速度计 + 陀螺仪 |
| **Android** | Android 8.0 (API 26)+ | 需加速度计 + 陀螺仪 |

V1 同步支持 iOS / Android 双端，采用 **Expo Managed Workflow** 单代码库。

### 1.4 核心体验

用户全程 **握持手机**（不可真的扔出手机），核心流程：

```
甩动手机 → 识别虚拟出手 → 物理模拟 → 落点震动 × N → 展示结果（次数 / 评分）
```

主反馈是 **震动**，不是画面。模拟阶段可显示极简 UI（计数、进度点），也可黑屏待机。

### 1.5 V1 范围说明

| 包含 | 不包含 |
|------|--------|
| 传感器采集、出手识别、**Rapier 3D 物理模拟** | 3D / 2D 轨迹回放渲染 |
| 落点震动反馈 | CFD / SPH 真实流体模拟 |
| 结果页（次数、评分、等级） | 在线多人、排行榜 |
| | 本地历史记录（推迟至 V1.1） |
| | AI 教练、ML 识别 |

---

## 2. 核心玩法

### 2.1 用户流程

1. 打开 App，进入投掷页
2. 手持手机，做打水漂甩动动作
3. 系统识别「虚拟出手瞬间」，提取速度 / 角度 / 旋转
4. 同步运行物理模拟，生成落点时间线
5. 进入 **模拟中** 状态：按落点时间线依次震动（用户等待，不可操作）
6. 全部震动结束 + 300 ms 停顿后，展示结果页
7. 点击「再玩一次」返回投掷页

### 2.2 体验时序

```
[待机] → [甩动] → [识别出手] → [模拟中 + 落点震动 × N] → [300ms 停顿] → [结果页]
         ↑ 用户操作      ↑ 核心难点        ↑ 主反馈（阻塞至震动播完）
```

#### 模拟阶段 UI 规则

| 状态 | UI | 用户操作 |
|------|-----|----------|
| 待机 | 「准备甩动」 | 可离开页面 |
| 甩动中（tracking） | 「甩动中…」 | 不可点击其他按钮 |
| 模拟中（simulating） | 「模拟中…」+ 可选落点计数 | **不可操作**，直至震动序列结束 |
| 结果页 | Skips / Score / Rating | 可「再玩一次」 |

> 物理 `simulate()` 在 Rapier 3D 中步进求解（通常 50–200 ms），期间 UI 保持 **simulating**；**结果页仍须等 `HapticsScheduler.play()` resolve 后** 再展示。

### 2.3 设计约束

- 识别的是 **「模拟出手」**，不是检测手机脱手
- 一个水漂落点 → **震动一次**，不叠加、不连震
- 虚拟出手确认时额外震 **1 次**，表示开始模拟
- 无效动作提示「再试一次」，不进入模拟

---

## 3. 技术选型

| 类别 | 选型 |
|------|------|
| 框架 | **Expo SDK 52+** |
| 语言 | **TypeScript 5.x** |
| 传感器 V1 | `expo-sensors`（优先 `DeviceMotion`，备选 Acc+Gyro 分订阅） |
| 传感器 V1.1 | 自研 **Expo Native Module**（真机不达标时） |
| 震动 | `expo-haptics` + `react-native` `Vibration`（Android fallback） |
| 物理引擎 | **Rapier 3D**（`@dimforge/rapier3d-compat`）无头刚体模拟 |
| 状态管理 | `zustand` |
| 本地存储 | V1.1：`expo-file-system` + JSON |
| 测试 | **jest-expo** + Jest + 传感器 CSV fixture 回放 |

### 平台权限与 Expo 配置

| 平台 | 配置 | 用途 |
|------|------|------|
| Android | `VIBRATE` | 落点震动 |
| Android 12+ | `HIGH_SAMPLING_RATE_SENSORS` | 采样间隔 < 200 ms（如 10 ms / 100 Hz） |
| iOS | 无运行时权限 | 传感器默认可用 |
| iOS | `NSMotionUsageDescription` | App Store 审核说明（推荐） |

`app.json` 示例：

```json
{
  "expo": {
    "orientation": "portrait",
    "plugins": [
      [
        "expo-sensors",
        {
          "motionPermission": "用于检测甩动动作，模拟打水漂投掷。"
        }
      ]
    ],
    "ios": {
      "infoPlist": {
        "NSMotionUsageDescription": "用于检测甩动动作，模拟打水漂投掷。"
      }
    },
    "android": {
      "permissions": [
        "VIBRATE",
        "android.permission.HIGH_SAMPLING_RATE_SENSORS"
      ]
    }
  }
}
```

Jest 配置（`package.json` 或 `jest.config.js`）：

```json
{
  "jest": {
    "preset": "jest-expo",
    "transformIgnorePatterns": [
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg)"
    ]
  }
}
```

Rapier WASM（`metro.config.js` 片段）：

```javascript
const { getDefaultConfig } = require('expo/metro-config');
const config = getDefaultConfig(__dirname);
config.resolver.assetExts.push('wasm');
module.exports = config;
```

---

## 4. 系统架构

```
┌─────────────────────────────────────────────────────────┐
│                  React Native App (Expo)                 │
├─────────────────────────────────────────────────────────┤
│  expo-sensors                                            │
│    Accelerometer ──┐                                     │
│    Gyroscope     ──┼──► MotionCaptureService             │
│                    │         │                           │
│                    │         ▼ SensorSample[]            │
│                    │    ThrowDetector                    │
│                    │         │                           │
│                    │         ▼ ThrowParams                 │
│                    │    StonePhysicsEngine (Rapier 3D)    │
│                    │         │                           │
│                    │         ▼ SimulationResult           │
│                    └──► HapticsScheduler → 结果页 UI     │
└─────────────────────────────────────────────────────────┘
```

**核心原则**：采集、识别为 **纯 TypeScript**；物理步进由 **Rapier 3D WASM** 驱动，触水逻辑（SkipContactModel）为 TS。均不依赖 React 组件，可在 Jest 中 mock Rapier 单测。

---

## 5. 核心功能模块

### 5.1 动作采集模块（Motion Capture）

#### 职责

- 高频读取加速度计与陀螺仪
- 维护最近 1~2 秒的 **环形缓冲区**
- 输出统一坐标系下的 `SensorSample`
- 屏蔽 iOS / Android 采样率与坐标系差异

#### 依赖（V1 默认：expo-sensors）

**优先方案**：`DeviceMotion`（单路回调，加速度 + 旋转速率同步性更好）

```typescript
import { DeviceMotion } from 'expo-sensors';

DeviceMotion.setUpdateInterval(10);
DeviceMotion.addListener((m) => {
  // m.acceleration — 含重力，m/s²
  // m.rotationRate — deg/s，需转为 rad/s
  // m.interval
});
```

**备选方案**：`Accelerometer` + `Gyroscope` 分订阅（两路独立回调，需时间戳配对）

```typescript
import { Accelerometer, Gyroscope } from 'expo-sensors';
```

| API | 用途 |
|-----|------|
| `DeviceMotion.addListener(cb)` | **推荐**：融合数据，单路回调 |
| `Accelerometer.isAvailableAsync()` | 备选：启动前检测 |
| `Gyroscope.isAvailableAsync()` | 备选：启动前检测 |
| `*.setUpdateInterval(ms)` | 设置期望采样间隔 |
| `Accelerometer.addListener(cb)` | 备选：订阅加速度（单位 **g**） |
| `Gyroscope.addListener(cb)` | 备选：订阅角速度（**rad/s**） |

#### 采样策略

| 参数 | 目标值 | 说明 |
|------|--------|------|
| 更新间隔 | **10 ms**（100 Hz） | `setUpdateInterval(10)` |
| 实际频率 | 60–100 Hz | 受 OS 与机型限制，运行时统计 |
| 缓冲区时长 | **2000 ms** | 环形缓冲，超出覆盖最旧样本 |
| 最大样本数 | 200 | `2000ms / 10ms` |

> Android 12+ 需 `HIGH_SAMPLING_RATE_SENSORS` 权限才能将采样间隔设为 10 ms（100 Hz）；无此权限时系统最低间隔为 200 ms（5 Hz）。

#### expo-sensors 已知限制

| 限制 | 说明 | 对项目的影响 |
|------|------|--------------|
| 采样率为期望值 | `setUpdateInterval(10)` 不保证 100 Hz，实际常见 50–100 Hz | 运行时统计 `actualHz`，低于 50 告警 |
| JS 线程事件调度 | 传感器为 **EventEmitter 推送**：原生侧 C++/JSI 发事件，listener 仍在 **JS 线程**执行；高频率下受 event loop 拥堵影响，可能抖动 / 丢帧 | 峰值+回落判定可能偶发漏检 |
| 双传感器不同步 | Acc + Gyro 分订阅时为两路独立流 | 优先 `DeviceMotion`；或 ±16 ms 时间戳配对 |
| 单位不统一 | Acc 为 **g**，Gyro 为 **rad/s**；DeviceMotion 加速度为 **m/s²**、旋转为 **deg/s** | 归一化层统一为 m/s² + rad/s |
| 无 UserAcceleration | `Accelerometer` 不含去重力；DeviceMotion 提供含/不含重力 | V1 用含重力模长；V1.1 可用 DeviceMotion 去重力 |
| 时间戳非硬件级 | 回调 `timestamp` 多为到达 JS 的时间 | 短窗口（50 ms）检测精度略差 |
| 后台降频 | App 进后台后传感器停更或大幅降频 | 仅前台投掷页采集（已约束） |
| Android 碎片化 | 低端机实际 Hz 更低、JS 线程调度更慢 | 真机矩阵测试 + 视觉计数 fallback |

> **结论**：对 MVP「50 Hz+、峰值+回落」**多数机型够用**；不应 Day 1 自写 Native，须 **真机测 `actualHz` 与识别稳定性** 后再决定。

> **架构说明**：`expo-sensors`（SDK 52+）基于 **Expo Modules API + JSI** 加载原生模块，**不是**旧版 RN Bridge（异步 JSON 序列化）。`setUpdateInterval` 等调用可走 JSI 同步路径；传感器数据通过 **C++ EventEmitter** 推送到 JS 线程的 `addListener` 回调。剩余瓶颈主要在 **OS 采样率** 与 **JS 线程 listener 执行**，而非 Bridge 本身。

#### Native Module 降级路径（V1.1，条件触发）

当 V1 真机验证出现以下 **任一** 情况时，启动自研 Native Module：

1. `actualHz` 在目标机型上 **持续 < 50 Hz**（≥ 3 s）
2. 同一甩动动作识别结果 **方差过大**（JS 线程 listener 调度抖动导致漏检峰值 / 回落）
3. 需要 **> 100 Hz 稳定采样** 或 **硬件单调时间戳**

**实现要点**（Expo Modules API + Dev Client）：

```
iOS     CMMotionManager.deviceMotion（100–200 Hz）
Android SensorManager SENSOR_DELAY_FASTEST + 原生 RingBuffer
        ↓ 批量推送（如每 10 ms 一帧），减少跨线程事件次数
        ↓
TS      MotionCaptureService（接口不变，ThrowDetector 无感切换）
```

| 项 | 说明 |
|----|------|
| 包名建议 | `expo-motion-capture`（本地 Expo Module） |
| 输出 | 与 `SensorSample` 相同（m/s² + rad/s + 单调 timestamp） |
| 前置条件 | `expo prebuild` + Dev Client；非纯 Managed Workflow |
| 工期估计 | 1–2 周（含 iOS / Android 双端 + 真机） |
| 原则 | **上层接口不变**，仅替换 `MotionCaptureService` 底层实现 |

**Phase 1（MVP）**：expo-sensors + `actualHz` 日志 + 波形调试  
**Phase 2（按需）**：Native Module，由 Week 1 真机数据决定是否启动

#### 原始单位与归一化

`expo-sensors` 各 API 原始单位如下，**必须在归一化层统一**：

| 来源 | 加速度 | 角速度 |
|------|--------|--------|
| `Accelerometer` | **g** | — |
| `Gyroscope` | — | **rad/s** |
| `DeviceMotion` | **m/s²**（含重力） | **deg/s** → 需 × `π/180` 转 rad/s |

归一化后 `SensorSample` 统一为 **m/s² + rad/s**：

```typescript
const G = 9.80665;

function toMs2(raw: RawSensorReading): { ax: number; ay: number; az: number } {
  return { ax: raw.x * G, ay: raw.y * G, az: raw.z * G };
}
```

> 判定逻辑与 `ThrowDetector` **只消费归一化后的 m/s²**，不直接读 Expo 原始 g 值。CSV fixture 也应存 m/s²，便于桌面回放与单测。

#### 数据模型

```typescript
interface RawSensorReading {
  x: number; y: number; z: number;  // 单位：g（Expo 原始）
  timestamp: number;
}

interface SensorSample {
  t: number;
  ax: number; ay: number; az: number;   // 单位：m/s²（含重力，已归一化）
  gx: number; gy: number; gz: number;   // 单位：rad/s
  axUser?: number; ayUser?: number; azUser?: number;  // m/s²，V1.1 去重力
}

interface MotionWindow {
  samples: SensorSample[];
  startT: number;
  endT: number;
  sampleCount: number;
  actualHz: number;
}
```

#### 坐标系归一化

设备坐标系转换为 **投掷坐标系**：

| 轴 | 含义 |
|----|------|
| **+X** | 出手方向（水平向前） |
| **+Y** | 垂直向上 |
| **+Z** | 横向（侧旋） |

V1 固定 **竖屏 portrait** 握持，App 锁定竖屏。归一化矩阵写入 `config/orientation.ts`，真机校准后冻结。

#### 去重力（V1.1 可选）

- `Accelerometer` 不提供 UserAcceleration，可在 TS 中低通分离，或
- 使用 `DeviceMotion` 的含/不含重力字段（推荐）

#### MotionCaptureService 接口

```typescript
interface MotionCaptureConfig {
  bufferDurationMs: number;   // default 2000
  updateIntervalMs: number;   // default 10
}

interface MotionCaptureService {
  start(config?: Partial<MotionCaptureConfig>): Promise<void>;
  stop(): void;
  getWindow(): MotionWindow;
  onSample(cb: (s: SensorSample) => void): () => void;
  getStats(): { actualHz: number; droppedFrames: number };
}
```

#### 时间与对齐

- 加速度计与陀螺仪独立回调，按最近时间戳合并（允许 ±16 ms 偏差）
- 无法配对时以加速度时间戳为主
- 丢弃乱序样本（`t <= lastT`）

#### 生命周期与错误处理

| App 状态 | 行为 |
|----------|------|
| 投掷页 active | `start()` |
| 识别完成 / 离开页面 | `stop()` |
| 进入后台 | `stop()`；若正在播放震动序列则 `hapticsScheduler.cancel()` |

| 错误 | 处理 |
|------|------|
| 传感器不可用 | 投掷页展示「此设备不支持体感」，禁用投掷 |
| `actualHz` < 50 持续 3 s | 顶部警告「采样率偏低，识别可能不准」，不阻断；**记录日志供 Phase 2 决策** |
| 权限被拒（Android 高采样率） | 降级为 OS 默认间隔，同上警告 |
| 样本乱序 | 丢弃该帧，计入 `droppedFrames` |

#### 输出

| 输出 | 类型 | 消费者 |
|------|------|--------|
| 实时样本流 | `SensorSample` | ThrowDetector |
| 窗口快照 | `MotionWindow` | 调试 / 参数提取 |

---

### 5.2 动作识别模块（Throw Detection）

#### 职责

- 从握持状态下的传感器曲线推断 **虚拟出手瞬间**
- 提取投掷参数 `ThrowParams`
- 过滤无效动作

> **核心难点**：用户全程握持手机，必须从运动曲线的峰值与突变推断出手，不能依赖「设备离手」检测。

#### 状态机

```
                    ┌─────────┐
         启动       │  IDLE   │◄──────────────────┐
        ──────────► │  待机   │                   │
                    └────┬────┘                   │
                         │ |a| > startThreshold   │
                         ▼                        │
                    ┌─────────┐                   │
                    │ TRACKING│─── 超时 2s ────────┤
                    │ 跟踪甩动 │                   │
                    └────┬────┘                   │
           峰值后急剧回落 │                        │
                         ▼                        │
                    ┌─────────┐    姿态不符       │
                    │ RELEASE │───────────────────┤
                    │ 锁定出手 │                   │
                    └────┬────┘                   │
                         │ 参数校验通过            │
                         ▼                        │
                    ┌─────────┐                   │
                    │  DONE   │──► 物理模拟       │
                    └─────────┘                   │
                         │                        │
                    无效  └──────────────────────►│ INVALID → 提示重试
                                                  └──────────
```

#### ThrowPhase 与状态映射

| `ThrowPhase` | 状态机节点 | 进入条件 | 退出条件 |
|--------------|------------|----------|----------|
| `idle` | IDLE | 初始 / `reset()` | `\|a\| > startThreshold` |
| `tracking` | TRACKING | 甩动开始 | 峰值后回落 → `released`；或超时 → `invalid` |
| `released` | RELEASE | 锁定 `releaseT` | 参数校验通过 → `done`；否则 → `invalid` |
| `done` | DONE | 输出 `ThrowParams` | 外部 `reset()` |
| `invalid` | INVALID | 超时 / 姿态不符 / 速度不足 | 外部 `reset()` |

#### 判定逻辑（V1）

| 阶段 | 传感器特征 | 动作 |
|------|------------|------|
| 待机（`idle`） | `\|a\| ≈ 9.8 m/s²`，方差低 | 等待甩动 |
| 甩动开始（→ `tracking`） | `\|a\| > startThreshold`（默认 **15 m/s²**，≈ 1.53g） | 开始记录窗口 |
| 虚拟出手（→ `released`） | `\|a\|` 峰值后单帧回落 > dropRatio（默认 40%） | 锁定 `releaseT` |
| 确认（→ `done`） | pitch 合理 + `velocity >= minVelocity` | 输出 `ThrowParams` |
| 无效（→ `invalid`） | 未达阈值 / 超时 / 姿态不符 | 提示重试 |

```typescript
function magnitude(a: { ax: number; ay: number; az: number }): number {
  return Math.sqrt(a.ax ** 2 + a.ay ** 2 + a.az ** 2);
}
```

#### 出手参数提取

在 `releaseT` 及其前 **50 ms** 窗口内计算：

| 参数 | 字段 | 算法 |
|------|------|------|
| 速度 | `velocity` | 窗口内 `\|a\|` 峰值（m/s²）× `kVel`（默认 **3.5**）→ m/s |
| 入水角 | `angleDeg` | `atan2(ay, ax)` → 度，[0°, 45°] |
| 自旋 | `spin` | `\|gy\|` 在 release 时刻（rad/s） |
| 侧旋 | `sideSpin` | `\|gz\|` 在 release 时刻（rad/s） |

> **速度标定**：`kVel` 为经验系数，将加速度峰值映射为投掷速度（典型甩动峰值 30–50 m/s² → 速度 10–17 m/s）。真机校准流程：3 人 × 3 力度，调整 `kVel` 使「大力甩 → skips 15+、轻甩 → skips 3–5」。

#### confidence 计算（V1）

```typescript
function computeConfidence(
  peakMag: number,      // m/s²
  velocity: number,
  angleDeg: number,
  config: ThrowConfig
): number {
  const peakScore = Math.min(peakMag / config.startThreshold, 2) / 2;
  const velScore = Math.min(velocity / config.minVelocity, 2) / 2;
  const angleOk =
    angleDeg >= config.minAngleDeg && angleDeg <= config.maxAngleDeg ? 1 : 0;
  return Math.min(
    1,
    Math.max(0, 0.4 * peakScore + 0.3 * velScore + 0.3 * angleOk)
  );
}
```

- `confidence < 0.5`：UI 提示「动作不够清晰，再试一次」，不进入模拟（phase → `invalid`）
- `confidence >= 0.5`：正常进入物理模拟

```typescript
interface ThrowParams {
  releaseT: number;
  velocity: number;
  angleDeg: number;
  spin: number;
  sideSpin: number;
  rawWindow: MotionWindow;
  confidence: number;   // 0–1，低于 0.5 建议重试
}
```

#### 阈值配置

外置 `config/throw-detection.json`（V1 仅本地文件，不做远程热更新）：

```json
{
  "startThreshold": 15.0,
  "dropRatio": 0.4,
  "minVelocity": 3.0,
  "maxAngleDeg": 45,
  "minAngleDeg": 5,
  "trackingTimeoutMs": 2000,
  "kVel": 3.5,
  "practiceMode": {
    "startThreshold": 12.0,
    "minVelocity": 2.0
  }
}
```

> 所有阈值基于 **归一化后的 m/s²**；`startThreshold: 15.0` 等价于约 **1.53g**。

#### ThrowDetector 接口

```typescript
type ThrowPhase = 'idle' | 'tracking' | 'released' | 'invalid' | 'done';

interface ThrowDetector {
  feed(sample: SensorSample): void;
  getPhase(): ThrowPhase;
  getResult(): ThrowParams | null;
  reset(): void;
}
```

#### 集成示例

```typescript
const capture = createMotionCaptureService();
const detector = createThrowDetector(config);
const physicsEngine = createStonePhysicsEngine();
const haptics = createHapticsScheduler();

await capture.start();
capture.onSample(async (sample) => {
  detector.feed(sample);
  const phase = detector.getPhase();
  if (phase === 'done') {
    const params = detector.getResult()!;
    capture.stop();
    setUiPhase('simulating');
    const result = await physicsEngine.simulate(params);
    await haptics.play(result);  // resolve = 全部震动 + 300ms 停顿结束
    navigate('Result', { result });
  }
  if (phase === 'invalid') {
    detector.reset();
    showToast('再试一次');
  }
});
```

#### 校准与容错

| 机制 | 说明 |
|------|------|
| 首次引导 | 轻甩 → 正常甩 → 大力甩，建立基线 |
| 练习模式 | 降低阈值，不计入正式记录 |
| 桌面调试 | `fixtures/throw-*.csv` 回放，无需真机 |

---

### 5.3 物理模拟（Stone Physics Engine）

> 采用 **Rapier 3D 刚体引擎** 做无头（headless）物理步进，不含 3D 渲染。  
> 水面交互使用 **基于入射角 / 速度 / 自旋的触水模型**（经验公式，非 CFD）。  
> **不使用 Rapier 2D**，也不使用纯参数 if/else 假物理。

#### 职责

- 接收 `ThrowParams`，在 3D 世界中初始化石头刚体
- 逐步进重力、空气阻力、水面碰撞，自然产生弹跳或下沉
- 从碰撞事件提取落点时间线（驱动震动）与轨迹（侧视投影）

#### 坐标系（Rapier 3D 世界）

| 轴 | 含义 |
|----|------|
| **+X** | 出手水平方向 |
| **+Y** | 垂直向上（水面法线） |
| **+Z** | 横向（侧旋） |

侧视输出时将 `(position.x, position.y)` 投影为 2D 轨迹；`position.z` 保留供 V2 回放扩展。

#### 刚体与场景 setup

```
World
├── stone（Dynamic RigidBody）
│   ├── Collider: 扁圆盘（Cylinder，height=0.02m，radius=0.04m）
│   ├── mass: 0.15 kg（可配置）
│   ├── 初速: 由 ThrowParams.velocity + angleDeg 分解为 vx, vy
│   └── 初角速: (spin, sideSpin, 0) 来自陀螺仪
├── water（Static half-space / Cuboid，y = 0）
│   └── 触水回调 → SkipContactModel
└── gravity: (0, -9.81, 0)
```

| 对象 | 参数 |
|------|------|
| 石头 | 密度 ~3000 kg/m³（近似岩石），摩擦 0.3，恢复系数由触水模型动态计算 |
| 空气阻力 | 每步对 `linvel` 施加 `-k * \|v\| * v`（k 可配置，默认 0.02） |
| 水面 | 静态 collider，厚度忽略；碰撞时调用 `SkipContactModel` |
| 时间步 | 固定 `dt = 1/120 s`，最多模拟 15 s 或 `maxSkips` 达上限 |

#### 触水模型（SkipContactModel）

Rapier 给出碰撞法线、相对速度、接触点后，由 TS 层计算是否水漂：

```typescript
interface WaterContact {
  speed: number;           // 入水速度标量 m/s
  incidentAngleDeg: number; // 速度方向与水面法线夹角（越小越平）
  spin: number;            // 绕 X 轴角速度 rad/s
}

function resolveWaterContact(c: WaterContact, cfg: PhysicsConfig): 'skip' | 'sink' {
  // 基于 Clanet 等打水漂经验关系式的简化版
  const canSkip =
    c.incidentAngleDeg < cfg.maxSkipAngleDeg &&   // 默认 20°
    c.speed > cfg.minSkipSpeed;                    // 默认 2.5 m/s
  return canSkip ? 'skip' : 'sink';
}

function applySkipImpulse(body: RigidBody, c: WaterContact, cfg: PhysicsConfig): void {
  // 按入射角衰减法向 / 切向速度，保留部分水平动能
  const e = cfg.restitution * (1 - c.incidentAngleDeg / 90);  // 动态恢复系数
  // 修改 body.linvel() / angvel()，由 Rapier 下一步继续积分
}
```

- **skip**：施加 `applySkipImpulse`，`skipCount++`，记录 `skipEvents`
- **sink**：将 body 设为 kinematic 或施加大阻尼下沉，模拟结束

> 这是 **刚体 + 经验触水力学**，不是网格流体。比参数表 if/else 更真实（速度 / 角度 / 自旋连续影响结果），但仍是可玩性校准模型。

#### 模拟流程

```typescript
async function simulate(params: ThrowParams): Promise<SimulationResult> {
  await Rapier.init();  // WASM 一次性加载
  const world = createWorld(params);
  const trajectory: TrajectoryPoint[] = [];
  const skipEvents: Array<{ t: number; x: number }> = [];
  let t = 0;
  let skipCount = 0;

  while (t < maxSimTime && skipCount < maxSkips && !sunk) {
    world.step();
    t += dt;
    recordTrajectory(world.stone, t, trajectory);
    // EventQueue 处理触水 → skip / sink
  }
  return { skips: skipCount, skipEvents, trajectory, score, durationMs: t * 1000 };
}
```

#### 依赖与 Expo 集成

| 项 | 说明 |
|----|------|
| 包 | `@dimforge/rapier3d-compat`（WASM，Hermes 兼容） |
| Metro | WASM 作为 asset 打包；`metro.config.js` 增加 `wasm` 支持 |
| 初始化 | App 启动或首次投掷前 `await RAPIER.init()`，约 20–50 ms |
| 线程 | 模拟 1000–2000 步约 50–200 ms；V1 主线程可接受，超出迁 Worker |
| 降级 | WASM 加载失败时 **报错提示**，不 silently 降级为假物理 |

#### 输出

```typescript
interface TrajectoryPoint {
  t: number;    // 秒，零点 = 虚拟出手（t=0）
  x: number;    // 侧视水平 m（= world.x）
  y: number;    // 高度 m（= world.y）
  z: number;    // 横向 m（= world.z，V1 仅存档）
  event: 'release' | 'skip' | 'splash' | 'sink';
}

interface SimulationResult {
  skips: number;
  score: number;
  trajectory: TrajectoryPoint[];
  /** t：秒，零点 = 虚拟出手；供 HapticsScheduler 做 setTimeout */
  skipEvents: Array<{ t: number; x: number }>;
  durationMs: number;
}
```

> **时间基准**：`trajectory[].t` 与 `skipEvents[].t` 均为相对虚拟出手的秒数。

#### 接口

```typescript
interface PhysicsConfig {
  gravity: number;           // 9.81
  releaseHeight: number;     // 1.2（石头初始 y）
  releasePositionX: number;  // 0
  stoneMass: number;         // 0.15
  stoneRadius: number;       // 0.04
  stoneHeight: number;       // 0.02
  airDragK: number;          // 0.02
  restitution: number;       // 0.72（触水基准恢复系数）
  maxSkipAngleDeg: number;   // 20
  minSkipSpeed: number;      // 2.5
  maxSkips: number;          // 50
  maxSimTime: number;        // 15（秒）
  dt: number;                // 1/120
}

interface StonePhysicsEngine {
  simulate(params: ThrowParams, config?: Partial<PhysicsConfig>): Promise<SimulationResult>;
}
```

> `simulate()` 改为 **async**（WASM 初始化 + 多步求解）；调用方 `await physicsEngine.simulate(params)`。

#### 与 §5.2 集成（更新）

```typescript
if (phase === 'done') {
  const params = detector.getResult()!;
  capture.stop();
  setUiPhase('simulating');
  const result = await physicsEngine.simulate(params);
  await haptics.play(result);
  navigate('Result', { result });
}
```

#### 评分

```typescript
function computeScore(skips: number, params: ThrowParams): number {
  const base = Math.min(skips * 6, 90);
  const angleBonus = params.angleDeg >= 8 && params.angleDeg <= 18 ? 10 : 0;
  return Math.min(100, base + angleBonus);
}
```

| 等级 | 分数 |
|------|------|
| S | ≥ 95 |
| A | ≥ 80 |
| B | ≥ 60 |
| C | < 60 |

---

### 5.4 震动反馈（Haptics）

主玩法反馈，发生在 **模拟中（simulating）** 阶段；结果页展示前必须播完。

#### 播放时序

```
t=0        识别成功 → impactAsync（出手确认，1 次）
t=skipEvents[0].t   → 落点 1
t=skipEvents[1].t   → 落点 2
...
t=skipEvents[n-1].t → 落点 n
全部完成 + 300ms    → Promise resolve → 导航至结果页
```

| 时机 | 次数 | 说明 |
|------|------|------|
| 虚拟出手确认 | 1 次 | `t=0`，进入 simulating |
| 每个水漂落点 | 各 1 次 | 与 `skipEvents[i].t` 一一对应 |
| 序列结束 | 0 次 | 300 ms 停顿后 resolve |

#### 实现策略

| 平台 | 主路径 | Fallback |
|------|--------|----------|
| iOS | `expo-haptics` `impactAsync(Medium)` | — |
| Android | `expo-haptics` `impactAsync(Medium)` | `Haptics.impactAsync` 无感时改用 `Vibration.vibrate(40)` |

```typescript
import * as Haptics from 'expo-haptics';
import { Platform, Vibration } from 'react-native';

async function pulse(): Promise<void> {
  try {
    await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
  } catch {
    if (Platform.OS === 'android') Vibration.vibrate(40);
  }
}

interface HapticsScheduler {
  /**
   * 按 skipEvents 时间线 scheduling；t=0 触发出手震。
   * resolve 时机：末次震动后 300ms。
   * 进入后台或 cancel() 时 reject / 提前结束。
   */
  play(result: SimulationResult): Promise<void>;
  cancel(): void;
}
```

#### 视觉 Fallback（Android 弱震动机型）

模拟中 UI 同步显示 **落点计数**（如「● ● ● ○ ○」），确保无震感时用户仍能感知 skips 进度。§8 验收：**震感可感知 OR 视觉计数正确** 二者满足其一即可。

> 不使用 `VibrationEffect.createOneShot` 原生 API（需 dev client）；V1 通过 `react-native` 内置 `Vibration` 覆盖。

---

### 5.5 UI 与数据（简要）

| 页面 | 内容 |
|------|------|
| 首页 | 开始投掷 |
| 投掷页 | 状态指示（待机 / 甩动中 / 模拟中 + 落点计数） |
| 结果页 | Skips、Score、Rating、「再玩一次」 |

本地历史记录（最近 50 条 JSON）推迟至 **V1.1**，MVP 不持久化。

---

## 6. 目录结构

```
src/
├── core/
│   ├── motion/
│   │   ├── MotionCaptureService.ts
│   │   ├── normalize.ts
│   │   └── RingBuffer.ts
│   ├── detection/
│   │   ├── ThrowDetector.ts
│   │   └── thresholds.ts
│   ├── physics/
│   │   ├── StonePhysicsEngine.ts   # 入口，async simulate()
│   │   ├── rapier3d.ts             # Rapier 3D World 步进
│   │   ├── skipContactModel.ts     # 触水力学（入射角/速度/自旋）
│   │   └── airDrag.ts              # 空气阻力
│   ├── haptics/
│   │   └── HapticsScheduler.ts
│   └── types.ts
├── config/
│   ├── throw-detection.json
│   └── physics.json
├── fixtures/
│   ├── throw-good.csv
│   └── throw-weak.csv
└── screens/
    ├── HomeScreen.tsx
    ├── ThrowScreen.tsx
    └── ResultScreen.tsx
```

---

## 7. 测试策略

| 层级 | 覆盖 |
|------|------|
| 单元测试 | `normalize`、ThrowDetector 状态机、`skipContactModel`、固定初值 Rapier 快照 |
| Fixture 回放 | CSV（m/s²）→ `feed()` → 断言 `ThrowParams` |
| 快照测试 | 固定 `ThrowParams` → `simulate()` → `skipEvents` / 轨迹 JSON 快照 |
| Rapier | WASM 初始化 + 10 次 simulate 不崩溃；skips 随 velocity 单调趋势正确 |
| 真机 | 3 种力度 × 2 平台，校准 `kVel` 与阈值 |
| 震动 | 落点次数与 `skips` 一致；或视觉计数 fallback 正确 |
| 工具 | **jest-expo** preset，core 模块在 Node 环境运行 |

---

## 8. MVP 验收标准

### 必须实现

- [ ] 加速度 g → m/s² 归一化正确（静止 ≈ 9.8 m/s²）
- [ ] `expo-sensors` 双端采样，实际频率 ≥ 50 Hz
- [ ] 2 s 环形缓冲，样本时间单调
- [ ] 虚拟出手识别（峰值 + 回落）
- [ ] `confidence < 0.5` 时不进入模拟
- [ ] 无效动作提示重试，不进入模拟
- [ ] Rapier 3D 无头模拟：刚体飞行 + 触水 skip/sink
- [ ] `simulate()` async，输出 skips / skipEvents / score / trajectory
- [ ] 落点震动与 skips 一致，或模拟中视觉计数正确
- [ ] 结果页在 `HapticsScheduler.play()` resolve **之后** 展示
- [ ] 结果页展示次数、评分、等级
- [ ] 同参数双端 skips 一致
- [ ] 核心逻辑 Jest 覆盖率 ≥ 80%

### V1.1 可选

- [ ] 自研 Native Module 传感器采集（`expo-motion-capture`，条件触发见 §5.1）
- [ ] 物理模拟迁 Worker 线程（降低主线程阻塞）
- [ ] 去重力用户加速度
- [ ] 首次校准引导
- [ ] 本地历史记录

### 暂不实现

- 轨迹可视化 / 3D 或 2D 回放
- CFD / SPH 真实流体模拟
- Rapier 2D（侧视简化版，不采用）
- 在线多人、排行榜
- ML 动作识别

---

## 9. 风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| 虚拟出手误识别 | 核心体验 | 练习模式、confidence、真机调参 |
| 传感器采样率不足 | 漏检峰值 | 高采样率权限、`DeviceMotion`、actualHz 监控；不达标则 V1.1 Native Module |
| JS 线程事件调度抖动 | 峰值/回落误判 | 优先 DeviceMotion；Native 侧批量推送 |
| 双端手感不一致 | 评分偏差 | 固定竖屏、分平台 kVel 校准 |
| Android 震动弱 | 无法感知节奏 | `Vibration` fallback + 模拟中视觉计数 |
| 后台打断震动序列 | 结果页异常 | 后台 `cancel()`，回前台提示重试 |
| Rapier 3D WASM 兼容性 | 无法模拟 | Metro 配置 + 启动预加载；失败时明确报错 |
| 物理步进耗时 50–200ms | 模拟阶段卡顿感 | simulating UI + V1.1 迁 Worker |
| 仅靠震动反馈 | 新用户不理解 | 首次引导 + 模拟中落点计数 |

---

## 10. 实施顺序建议

| 阶段 | 交付 |
|------|------|
| Week 1 | MotionCaptureService + 归一化 + **actualHz 真机验证**（决定是否需要 Native Module） |
| Week 2 | ThrowDetector + confidence + CSV fixture 单测 |
| Week 3 | Rapier 3D 集成 + SkipContactModel + HapticsScheduler |
| Week 4 | 投掷页 / 结果页 + simulating 时序 + 双端真机校准（kVel + 物理参数） |

---

## 11. 一句话总结

React Native + expo-sensors 采集甩动数据，ThrowDetector 锁定出手参数，**Rapier 3D** 无头刚体模拟计算水漂落点，expo-haptics 震出节奏——震动播完后展示结果，全程无需画面回放。
