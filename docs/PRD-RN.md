# 📄 PRD：打水漂体感模拟器（React Native 版）

## 1. 产品概述

### 1.1 产品名称

Stone Skip / 打水漂模拟器

### 1.2 产品定位

基于手机体感（加速度计 + 陀螺仪）的投掷模拟 App。  
用户手持手机模拟打水漂甩动动作，系统识别「虚拟出手」后运行 **Rapier 3D 刚体物理模拟**（真实飞行 + 经验水面接触模型），通过 **震动节奏** 感知水漂次数，无需盯屏。

### 1.3 支持平台

| 平台 | 最低版本 | 说明 |
|------|----------|------|
| **iOS** | iOS 14+ | iPhone，需加速度计 + 陀螺仪 |
| **Android** | Android 8.0 (API 26)+ | 需加速度计 + 陀螺仪 |

V1 同步支持 iOS / Android 双端，采用 **Expo Managed Workflow** 单代码库。

### 1.4 核心体验

用户全程 **握持手机**（不可真的扔出手机），核心流程：

```
甩动手机 → 识别虚拟出手 → 物理模拟 → 落点震动 × N → 展示结果（次数）
```

主反馈是 **震动**，不是画面。模拟阶段可显示极简 UI（计数、进度点），也可黑屏待机。

### 1.5 V1 范围说明

| 包含 | 不包含 |
|------|--------|
| 传感器采集、出手识别、**Rapier 3D 物理模拟**、**运动轨迹数据** | 3D / 2D 轨迹回放渲染 |
| **Reanimated Worklet 采集**（真机验证通过后） | CFD / SPH 真实流体模拟 |
| 落点震动反馈 | 评分 / 等级、在线多人、排行榜 |
| 结果页（水漂次数） | 本地历史记录（推迟至 V1.1） |
| | AI 教练、ML 识别 |

---

## 2. 核心玩法

### 2.0 握持与坐标（不锁物理方向）

**横拿 / 竖拿有没有影响？**  
有——但影响的是 **设备坐标系里的 raw 读数**，不是用户「会不会打水漂」。若算法写死「竖屏 + gy = spin」，用户随便握就会偏。  
V1 **不规定** 手机在手中的横竖朝向；只要求：**单手握住，做一次水平方向的甩腕**（像甩石子，不是把手机扔出去）。

| 区分 | 说明 |
|------|------|
| **App 竖屏** | 仅 UI 布局锁定 portrait，**≠** 假设手机在手中也是竖着的 |
| **握持自由度** | 横拿、竖拿、略歪均可；算法用 **重力对齐坐标系**，不依赖固定 gx/gy/gz 映射 |
| **用户引导** | 首次弹窗一句：「单手握住，朝水面方向水平甩出去即可。」**不展示**「必须竖拿/横拿」示意图 |

**重力对齐投掷系**（每个样本或至少在 `releaseT` 重算）：

```
ĝ = normalize(重力向量)          // DeviceMotion 重力字段，或静止时加速度低通
↑ = −ĝ                          // 世界「上」
a_horiz = a_user − (a_user·↑)^   // 用户加速度在水平面投影
ω_horiz = ω − (ω·↑)^             // 角速度在水平面分量
```

| 参数 | 提取（与握持无关） |
|------|-------------------|
| 峰值 / 速度 | `\|a_user\|` 或 `\|a_horiz\|` 的峰值 + §2.4 持续时间（标量，旋转不变） |
| `angleDeg` | `atan2(|a_horiz|, |a_up|)`：出手方向相对水平面的仰角 |
| `spin` | `\|ω_horiz\|` @ release：水平面内总自旋（不绑定某一设备轴） |
| `sideSpin` | V1 可省略或取 `\|ω·↑\|`（绕「竖直轴」分量，辅助 confidence，不进物理主参） |

> **核心原则**：识别层尽量用 **标量 + 重力对齐向量**；只有进 Rapier 前才把 `(velocity, angleDeg, spin)` 投到固定的模拟世界坐标（+X 出手、+Y 上）。握持乱 → 影响的是 confidence，不应让物理直接发疯——仍靠 `maxVelocity` clamp + 形态校验。

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
| 结果页 | Skips、一句随机评语、「再玩一次」 |

> 物理 `simulateChunked()` 在 Rapier 3D 中分片步进（wall time 50–200 ms，分散多帧），期间 UI 保持 **simulating** 且动画流畅；**结果页仍须等 `HapticsScheduler.play()` resolve 后** 再展示。

### 2.3 设计约束

- 识别的是 **「模拟出手」**，不是检测手机脱手
- 一个水漂落点 → **震动一次**，不叠加、不连震
- **震动力度跟随该次水漂**：入水越猛 → 震感越强；后续水漂能量衰减 → 震感自然变轻
- 虚拟出手确认时额外震 **1 次**，表示开始模拟
- 无效动作提示 **具体原因**（见 §5.2），不进入模拟
- 连续 2 次 `confidence < 0.5` 自动切入 **轻松模式**（降低阈值 + UI 提示）

---

## 2.4 开发与逻辑对齐（启动会必读）

以下在编码前对齐，避免返工。

### 速度提取：峰值 + 持续时间（V1 一步到位）

> 批注：**不要拆 V2/V3**，V1 即采用「峰值 + 形态校验」完整方案，不单靠 `peakMag × kVel`。

人类甩动峰值与持续时间差异大：慢而长（峰值低、冲量大）vs 快而短（峰值极高）。V1 算法：

| 观测量 | 字段 | 算法 |
|--------|------|------|
| 峰值加速度 | `peakMag` | tracking 窗口内 `\|a\|` 最大（m/s²） |
| 峰值持续时间 | `peakDurationMs` | `\|a\| > 0.5 × peakMag` 的连续时长（ms） |
| 上升时间 | `riseTimeMs` | 首次超过 `startThreshold` → 峰值（ms） |
| 峰值上升率 | `peakRiseRate` | `peakMag / (riseTimeMs / 1000)`（m/s³ 量级） |

```typescript
function estimateVelocity(m: ThrowMotionMetrics, cfg: ThrowConfig): number {
  // 慢长甩动：durationFactor 补偿峰值偏低；快短甩动：durationFactor 接近 1
  const durationFactor = Math.min(
    cfg.maxDurationFactor, // 默认 1.25
    cfg.minDurationFactor + m.peakDurationMs / cfg.refPeakDurationMs // ref 默认 180ms
  );
  const raw = cfg.kVel * m.peakMag * durationFactor;
  return clamp(raw, cfg.minVelocity, cfg.maxVelocity); // maxVelocity 默认 22 m/s
}
```

**形态异常 → 降 confidence，不硬算离谱初速**：

| 异常 | 条件（默认） | 处理 |
|------|--------------|------|
| 过尖脉冲 | `riseTimeMs < 30` 且 `peakMag > 1.5 × startThreshold` | confidence −0.3；`invalidReason: 'too_snappy'` |
| 上升率过激 | `peakRiseRate > maxPeakRiseRate`（默认 800） | confidence −0.25 |
| 速度顶格 | `raw > maxVelocity` | confidence −0.2；**仍 clamp 后进入模拟**（若 confidence 仍 ≥ 0.5） |

`ThrowParams` 增加 `motionMetrics` 与可选 `invalidReason`（供 UI 文案）。

### Rapier 碰撞事件时序

`world.step()` 顺序：**积分 → 碰撞检测 → 填充 EventQueue**。在 EventQueue 回调里 `applyWaterContact()` 改 `linvel`/`angvel`，改动在 **下一次 `step()`** 生效——这是正确用法。

**实现约束**：

1. 每轮 `step()` 后 **排空当帧 EventQueue**（同一帧可能多个接触点，如溅水）  
2. 处理 EventQueue 时 **禁止** 嵌套调用 `world.step()`  
3. 单帧内对同一 body 的多次触水：按事件顺序依次 `applyWaterContact()`，或合并为一次冲量（V1 用顺序处理即可）

```typescript
world.step();
eventQueue.drainCollisionEvents((handle1, handle2, started) => {
  if (!started) return;
  // ... applyWaterContact for water collider
});
// 然后再进入下一帧 / 下一批 step
```

### `simulateChunked` 与 App 生命周期

仅用 `requestAnimationFrame` 让出主线程时，**App 切后台后 rAF 不触发**，Promise 永不 resolve，UI 卡死「模拟中…」。

**V1 必须实现**：

```typescript
async function yieldFrame(signal: AbortSignal): Promise<void> {
  if (signal.aborted) throw new SimulationAbortedError('background');
  await Promise.race([
    new Promise<void>((r) => requestAnimationFrame(() => r())),
    delay(500), // 兜底：后台 / 低帧率时仍推进
  ]);
}
```

- 监听 `AppState`：`background` → `abortController.abort()` → `simulateChunked` reject  
- 外层 `catch`：`hapticsScheduler.cancel()` + `setUiPhase('idle')` + Toast「模拟已中断，请重试」  
- `HapticsScheduler.play()` 同样监听 `AppState`，后台即 `cancel()`

### 传感器时间 vs 物理时间 vs 墙钟

| 时间 | 含义 | 用途 |
|------|------|------|
| `releaseT` | 传感器单调时间戳 | 仅采集 / 调试 |
| `skipEvents[].t` | 物理模拟时间（秒，出手 = 0） | 轨迹、**震动排期基准** |
| `Date.now()` | 墙钟 | `play()` 内换算 `setTimeout` |

`simulateChunked` 下物理时间与墙钟 **非线性**。`HapticsScheduler.play()` **必须**：

```typescript
const t0 = Date.now(); // play() 开始时刻 = 物理 t=0
for (const ev of skipEvents) {
  const delayMs = Math.max(0, ev.t * 1000 - (Date.now() - t0));
  await sleep(delayMs);
  await pulse();
}
```

**禁止**在模拟尚未结束时，用「物理总时长」一次性预排所有 `setTimeout(ev.t * 1000)`（chunked 下 wall time 与 physics time 不一致）。

### Rapier WASM 预加载

**强制**：`App.tsx` 挂载时 `Rapier.init()`（首页静默预加载），**禁止**等到首次投掷才 init。投掷页 `simulateChunked` 内仅 `await ensureRapierReady()`（已 init 则立即返回）。

---

## 3. 技术选型

| 类别 | 选型 |
|------|------|
| 框架 | **Expo SDK 52+** |
| 语言 | **TypeScript 5.x** |
| 传感器 + 识别 V1 | **`expo-sensors` DeviceMotion**（JS 线程，先跑通算法） |
| 传感器 V1 升级 | **Reanimated `useAnimatedSensor` + UI Worklet**（Week 1 真机 A/B 后决定） |
| 传感器 V1.1 | 自研 Native Module（Worklet + expo-sensors 均不达标时） |
| 动画 | **react-native-reanimated**（模拟中 UI；传感器 Worklet 为可选升级） |
| 物理引擎 | **Rapier 3D**（`@dimforge/rapier3d-compat`）无头刚体模拟 |
| 物理线程 | **JS 主线程 + 时间分片**（chunking）；WASM **只能**在 RN JS Runtime 跑 |
| 震动 | `expo-haptics` + `react-native` `Vibration`（Android fallback） |
| 状态管理 | `zustand` |
| 本地存储 | V1.1：`expo-file-system` + JSON |
| 测试 | **jest-expo** + Jest + 传感器 CSV fixture 回放 |

### 多线程架构：术语澄清（必读）

RN 生态里「Worklet / Worker」有多套实现，**名称极易误导**。本项目用到的与 **不存在的** 如下：

| 名称 | 是否存在 | 实际是什么 | 本项目 |
|------|----------|------------|--------|
| **Reanimated UI Worklet** | ✅ | `react-native-reanimated` 在 **UI 线程**跑的 `'worklet'` 函数；`useAnimatedSensor` 把读数写入 SharedValue | 传感器识别 **可选升级** |
| **`react-native-worklets`** | ✅ | Software Mansion 独立多线程库（Reanimated 4 依赖）；`createWorkletRuntime` 可开 **后台 JS 线程** | 物理 **不适用**（见下） |
| **`react-native-worklets-core`** | ✅ | Margelo 的 Worklet 运行时，多为 VisionCamera / Skia 等库的 **peer dependency** | **非首选**；不单独引入 |
| **`expo-workers`** | ❌ **不存在**（客户端） | 网上同名的是 **Cloudflare Workers 服务端部署**（`expo-server` / EAS Hosting），与 RN App 后台线程无关 | **禁止写入 PRD/代码** |
| **Rapier WASM** | ✅ | 仅在 RN **主 JS Runtime**（Hermes）通过 `@dimforge/rapier3d-compat` 加载 | V1 分片模拟 |

**关键结论**：

1. **UI Worklet ≠ 后台 Worker**：Reanimated Worklet 跑 UI 线程，不能跑 Rapier WASM。  
2. **`react-native-worklets` 后台 Runtime 也不支持 WASM**（精简 JS 上下文，无 `WebAssembly.instantiate`）→ **Rapier 无法迁到 Worklet/Worker Runtime**。  
3. 物理模拟 realistic 选项只有：**JS 主线程分片**（V1），或未来 **Native Module 绑 Rapier C++**（V2，不在 V1 范围）。  
4. 传感器 Worklet **有价值但需真机验证**，不是「Day 1 降维打击」；算法先用 expo-sensors 跑通。

```
┌──────────── UI 线程 · Reanimated Worklet（可选升级）────────────┐
│  useAnimatedSensor(ACCELEROMETER + GYROSCOPE, interval: 10)       │
│       ↓ merge → throwDetectorCore（'worklet'）                    │
│       ↓ runOnJS(onThrowDetected) ───────────────────────────────┼──► JS 主线程
└───────────────────────────────────────────────────────────────────┘      │
                                                                         ▼
┌──────────── JS 主线程（Hermes + WASM）─────────────────────────────────┐
│  expo-sensors 路径：DeviceMotion.addListener → throwDetectorCore       │
│  simulateChunked(params)  ← Rapier WASM，rAF 每帧 N 步让出             │
│       ↓ SimulationResult → HapticsScheduler                            │
└────────────────────────────────────────────────────────────────────────┘
```

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
      "react-native-reanimated/plugin",
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
      "node_modules/(?!((jest-)?react-native|@react-native(-community)?)|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg|react-native-reanimated)"
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
│  V1 基线：expo-sensors DeviceMotion → JS 线程            │
│       ↓ throwDetectorCore                               │
│                                                          │
│  可选升级：UI Worklet（useAnimatedSensor + 同上 core）    │
│       ↓ runOnJS(onThrowDetected)                        │
│                          │                               │
│  JS 主线程：simulateChunked() · Rapier WASM              │
│       ↓ HapticsScheduler → 结果页                       │
└─────────────────────────────────────────────────────────┘
```

**核心原则**：

- **算法唯一**：`throwDetectorCore.ts` 一份逻辑，expo-sensors / Worklet / Jest 共用
- **先基线后 Worklet**：Week 1 用 expo-sensors 验证识别；Worklet 仅在有 measurable 增益时切换主路径
- **Rapier 只在 JS Runtime**：禁止 UI Worklet / `createWorkletRuntime` 跑 WASM
- **Rapier 决定运动**；水模型只改碰撞后状态；skip 从轨迹派生

---

## 5. 核心功能模块

### 5.1 动作采集模块（Motion Capture）

#### 职责

- 高频读取加速度计与陀螺仪
- 维护最近 1~2 秒的 **环形缓冲区**
- 输出统一坐标系下的 `SensorSample`
- 屏蔽 iOS / Android 采样率与坐标系差异

#### V1 基线路径：expo-sensors（JS 线程）

**先跑通，再优化。** 识别算法、阈值、fixture 单测均基于此路径；Worklet 是性能升级而非前提。

```typescript
import { DeviceMotion } from 'expo-sensors';

DeviceMotion.setUpdateInterval(10);
DeviceMotion.addListener((m) => {
  const sample = normalizeDeviceMotion(m); // m/s² + rad/s
  const outcome = feedThrowDetector(state, sample, config);
  // phase === 'done' → 触发 simulateChunked
});
```

| API | 用途 |
|-----|------|
| `DeviceMotion.addListener(cb)` | **V1 基线**：融合加速度 + 旋转速率，单路回调 |
| `Accelerometer` + `Gyroscope` | 备选（两路流，需时间戳配对） |

| 已知限制 | 影响 | 缓解 |
|----------|------|------|
| listener 在 JS 线程 | 100 Hz 下受 React 渲染 / 物理分片影响 | Week 1 测 actualHz；不达标再切 Worklet |
| 采样率为期望值 | 实际常见 50–100 Hz | 运行时统计，< 50 告警 |
| Android 12+ 高采样率 | 需 `HIGH_SAMPLING_RATE_SENSORS` | 已在 app.json 配置 |

#### 可选升级：Reanimated Worklet（UI 线程）

> **Review 结论**：Worklet **值得做，但不是无脑替换**。Reanimated 官方文档写明：传感器最高约 **100 Hz**；数据写入 SharedValue，Worklet 在 **UI 线程**读取——确实绕过 JS event loop，但 **不消除 OS 采样上限**，也 **没有 DeviceMotion 融合 API**。

**何时切换为主路径**（Week 1 真机 A/B，需同时满足）：

1. expo-sensors 路径 `actualHz < 50` 或识别方差显著更大  
2. Worklet 路径同动作识别更稳定（≥ 3 人 × 10 次）  
3. `useAnimatedSensor` 双端 `isAvailable === true`

**Reanimated 与 expo-sensors 的关键差异**（实现时必须处理）：

| 项 | expo-sensors DeviceMotion | Reanimated `useAnimatedSensor` |
|----|---------------------------|--------------------------------|
| 加速度语义 | 含重力（≈ 9.8 m/s² 静止） | `ACCELEROMETER` = **不含重力**的用户加速度 |
| 融合 | 单路 DeviceMotion | ACC + GYRO **两路独立** SharedValue，需 Worklet 内 merge |
| 接入方式 | 任意 TS 模块 | **必须是 React Hook**（`useThrowCapture`） |
| 角速度单位 | deg/s → 需转 rad/s | GYROSCOPE 已是 **rad/s** |
| iOS | — | 部分传感器需系统 **定位服务** 开启（官方 Remarks） |

**推荐算法适配**（Worklet 路径）：直接使用 **用户加速度** 做峰值检测（静止 ≈ 0，甩动峰值更干净），阈值与含重力模型 **分开校准**，不可复用 `startThreshold: 15` 同一数值。

```typescript
import {
  useAnimatedSensor,
  SensorType,
  useAnimatedReaction,
  runOnJS,
  useSharedValue,
} from 'react-native-reanimated';
import { feedThrowDetector } from './throwDetectorCore';

export function useThrowCapture(onDetected: (p: ThrowParams) => void) {
  const acc = useAnimatedSensor(SensorType.ACCELEROMETER, { interval: 10 });
  const gyro = useAnimatedSensor(SensorType.GYROSCOPE, { interval: 10 });
  const state = useSharedValue(createDetectorState());

  useAnimatedReaction(
    () => ({ a: acc.sensor.value, g: gyro.sensor.value }),
    (s) => {
      'worklet';
      if (!acc.isAvailable || !gyro.isAvailable) return;
      const sample = mergeAccGyro(s.a, s.g); // 用户加速度 + rad/s
      const outcome = feedThrowDetector(state.value, sample, WORKLET_CONFIG);
      state.value = outcome.nextState;
      if (outcome.phase === 'done') runOnJS(onDetected)(outcome.params!);
    }
  );
}
```

| 真实优势 | 不应过度承诺 |
|----------|--------------|
| 识别逻辑不与 JS event loop 抢时间 | ≠ 保证 100 Hz（OS + Reanimated 仍限 ~100 Hz） |
| 物理分片期间仍可采样（UI 线程独立） | ≠ 完全不受 UI 负载影响（同线程仍有动画竞争） |
| 零 listener 跨线程延迟 | ≠ 替代 Native Module（极端场景仍可能需要） |

**Worklet 约束**：`throwDetectorCore` 必须 worklet-safe；环形缓冲用 SharedValue 固定数组；识别成功只 `runOnJS` 回调。

#### Jest / fixture 路径

Node 环境无 Reanimated → **始终**用 expo-sensors 风格的 `feedThrowDetector` + CSV 回放，与运行时路径无关。

#### 采样策略

| 参数 | 目标值 | 说明 |
|------|--------|------|
| 更新间隔 | **10 ms**（100 Hz） | expo-sensors：`setUpdateInterval(10)`；Worklet：`{ interval: 10 }` |
| 实际频率 | 目标 **≥ 50 Hz**（基线）；Worklet 目标 **≥ 80 Hz** | 运行时统计 `actualHz` |
| 缓冲区时长 | **2000 ms** | 环形缓冲，超出覆盖最旧样本 |
| 最大样本数 | 200 | `2000ms / 10ms` |

> Android 12+ 需 `HIGH_SAMPLING_RATE_SENSORS` 权限（expo-sensors 与 Reanimated 底层均受 OS 限制）。

#### Native Module（V1.1，低优先级）

仅当 **expo-sensors 与 Worklet 两条路径** 在目标机型上均不达标时考虑（1–2 周，需 Dev Client）。

**Phase 1（MVP）**：expo-sensors 基线 + actualHz 日志  
**Phase 1b（同 Week 1）**：Worklet A/B，有增益则切换  
**Phase 2（低概率）**：Native Module

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

**两层坐标**：

| 层 | 名称 | 做法 |
|----|------|------|
| 1 | 设备原始 | expo `DeviceMotion` → m/s² + rad/s |
| 2 | **重力对齐投掷系** | 用重力向量建水平面，投影 `a_user`、`ω`（见 §2.0） |
| 3 | Rapier 世界系 | 模拟器固定 +X 出手、+Y 上；仅 `ThrowParams` 进入物理时映射 |

`config/orientation.ts` 只存 **重力对齐算法参数**（低通系数等），**不存**「竖拿固定旋转矩阵」。App UI 锁定 portrait 与传感器解耦。

```typescript
interface GravityAlignedSample {
  t: number;
  aUser: { x: number; y: number; z: number };   // m/s²，去重力
  aHoriz: { x: number; y: number; z: number }; // 水平面投影
  omegaHoriz: { x: number; y: number; z: number }; // rad/s
  magUser: number;
  magHoriz: number;
  spinMag: number;   // |omegaHoriz|
}
```

#### 重力 / 用户加速度（V1 必做）

`DeviceMotion` 在 iOS / 多数 Android 提供 **含重力** 与 **用户加速度**（或可用 `gravity` + `acceleration` 组合）。V1 识别 **必须**用 `a_user` 做峰值与速度，避免握持改变静止读数。

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
> **V1 运行位置**：默认可在 **JS 线程**（expo-sensors listener）或 **UI Worklet**（升级后）执行，均调用同一 `throwDetectorCore`。

#### 代码分层

```
throwDetectorCore.ts     ← 纯函数 feedThrowDetector()，Worklet + Jest 共用
throwDetectorWorklet.ts  ← 'worklet' 包装 + SharedValue 状态
ThrowDetector.ts         ← expo-sensors 降级路径的 class 封装（feed() 调 core）
```

```typescript
// throwDetectorCore.ts — 无 'worklet' 标记，但函数体必须 worklet-safe（无闭包捕获、无 async）
export function feedThrowDetector(
  state: DetectorState,
  sample: SensorSample,
  config: ThrowConfig
): { nextState: DetectorState; phase: ThrowPhase; params?: ThrowParams } {
  // 状态机：idle → tracking → released → done | invalid
}
```

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

在 `releaseT` 及其前 **50 ms** 窗口内计算（速度见 §2.4「峰值 + 持续时间」）：

| 参数 | 字段 | 算法 |
|------|------|------|
| 速度 | `velocity` | `estimateVelocity(motionMetrics)`；`peakMag` 取自 **\|a_horiz\|** 或 **\|a_user\|** 峰值 |
| 形态指标 | `motionMetrics` | `peakMag`、`peakDurationMs`、`riseTimeMs`、`peakRiseRate` |
| 入水角 | `angleDeg` | 重力对齐系下出手仰角：`atan2(|a_horiz|, |a_up|)` → [0°, 45°] |
| 自旋 | `spin` | `\|ω_horiz\|` @ release（rad/s），**不绑定 gy/gz** |
| 侧旋 | `sideSpin` | V1 可选：`\|ω·û\|`；仅参与 confidence，默认物理初值可置 0 |

> **标定**：`kVel`、`refPeakDurationMs`、`maxPeakRiseRate` 外置 `throw-detection.json`；真机 3 人 × 3 力度校准，使轻甩 skips 3–5、大力 15+，且 clamp 后无「飞出天际」轨迹。

#### confidence 计算（V1）

```typescript
type InvalidReason =
  | 'too_weak'
  | 'angle_bad'
  | 'too_snappy'      // 上升时间过短、尖峰
  | 'rise_rate_high'  // peakRiseRate 异常
  | 'unclear';        // 综合过低

function computeConfidence(
  metrics: ThrowMotionMetrics,
  velocity: number,
  angleDeg: number,
  config: ThrowConfig
): { confidence: number; invalidReason?: InvalidReason } {
  const peakScore = Math.min(metrics.peakMag / config.startThreshold, 2) / 2;
  const velScore = Math.min(velocity / config.minVelocity, 2) / 2;
  const angleOk =
    angleDeg >= config.minAngleDeg && angleDeg <= config.maxAngleDeg ? 1 : 0;
  const durationScore = Math.min(metrics.peakDurationMs / config.refPeakDurationMs, 1);

  let confidence =
    0.3 * peakScore + 0.25 * velScore + 0.25 * angleOk + 0.2 * durationScore;

  let invalidReason: InvalidReason | undefined;
  if (metrics.riseTimeMs < config.minRiseTimeMs && metrics.peakMag > config.startThreshold * 1.5) {
    confidence -= 0.3;
    invalidReason = 'too_snappy';
  }
  if (metrics.peakRiseRate > config.maxPeakRiseRate) {
    confidence -= 0.25;
    invalidReason = 'rise_rate_high';
  }
  if (velocity <= config.minVelocity) invalidReason = 'too_weak';
  if (!angleOk) invalidReason = 'angle_bad';

  return { confidence: clamp01(confidence), invalidReason };
}
```

| confidence | UI |
|------------|-----|
| `< 0.5` | 不进入模拟；展示 **具体原因**（见下表） |
| `≥ 0.5` | 进入物理模拟 |

**无效动作文案**（`invalidReason` → 用户可见）：

| `invalidReason` | 文案 |
|-----------------|------|
| `too_weak` | 速度太慢，再用力甩一下 |
| `angle_bad` | 出手角度不对，试着更平一点 |
| `too_snappy` | 动作太急促，顺势带长一点 |
| `rise_rate_high` | 动作不够顺畅，再试一次 |
| `unclear` / 其他 | 动作不够清晰，再试一次 |

**轻松模式（挫败感缓解）**：会话内连续 **2 次** `confidence < 0.5` → 自动应用 `practiceMode` 阈值 + 顶部条「已切换轻松模式」；成功一次投掷后恢复默认阈值。

```typescript
interface ThrowParams {
  releaseT: number;
  velocity: number;
  angleDeg: number;
  spin: number;
  sideSpin: number;
  motionMetrics: ThrowMotionMetrics;
  rawWindow: MotionWindow;
  confidence: number;
  invalidReason?: InvalidReason;
}
```

#### 阈值配置

外置 `config/throw-detection.json`（V1 仅本地文件，不做远程热更新）：

```json
{
  "startThreshold": 15.0,
  "dropRatio": 0.4,
  "minVelocity": 3.0,
  "maxVelocity": 22.0,
  "maxAngleDeg": 45,
  "minAngleDeg": 5,
  "trackingTimeoutMs": 2000,
  "kVel": 3.5,
  "refPeakDurationMs": 180,
  "minDurationFactor": 0.6,
  "maxDurationFactor": 1.25,
  "minRiseTimeMs": 30,
  "maxPeakRiseRate": 800,
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

#### 集成示例（V1 基线 · expo-sensors）

```typescript
const capture = createMotionCaptureService(); // DeviceMotion 封装
const detector = createThrowDetector(config);  // 内部调 throwDetectorCore

await capture.start();
capture.onSample(async (sample) => {
  const outcome = detector.feed(sample);
  if (outcome.phase === 'done') {
    capture.stop();
    setUiPhase('simulating');
    const result = await physicsEngine.simulateChunked(outcome.params!);
    await haptics.play(result);
    navigate('Result', { result });
  }
});
```

#### 集成示例（Worklet 升级 · ThrowScreen）

```typescript
// ThrowScreen.tsx
const physicsEngine = useMemo(() => createStonePhysicsEngine(), []);
const haptics = useMemo(() => createHapticsScheduler(), []);

const onThrowDetected = useCallback(async (params: ThrowParams) => {
  setUiPhase('simulating');
  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium); // 出手确认震
  const result = await physicsEngine.simulateChunked(params); // JS 线程分片，非 Worklet
  await haptics.play(result);
  navigate('Result', { result });
}, []);

useThrowCapture(onThrowDetected); // §5.1 Worklet 订阅
```

#### 集成示例（expo-sensors 降级）

同 V1 基线示例；与「降级」一词无关——**这就是默认路径**。

#### 校准与容错

| 机制 | 说明 |
|------|------|
| 握持引导 | 首次一句「水平甩出即可」，不锁横竖 |
| 首次引导 | 轻甩 → 正常甩 → 大力甩，建立基线 |
| 轻松模式 | 连续 2 次失败自动降低阈值；成功一次恢复 |
| 具体失败文案 | `invalidReason` 映射（见上表） |
| 桌面调试 | `fixtures/throw-*.csv` 回放，无需真机 |

---

### 5.3 物理模拟（Stone Physics Engine）

> 采用 **Rapier 3D 刚体引擎** 做无头（headless）物理步进，不含 3D 渲染。  
> **不使用 Rapier 2D**，也不使用纯参数 if/else 假物理。

#### 物理定位（必读）

本系统的物理本质是：

> **真实刚体飞行 + 经验水动力接触模型**

而不是「真实水体模拟」。Rapier 擅长刚体运动与接触事件；水面行为由 TS 层经验公式近似，**不**做 CFD / SPH / 流体求解。

| 类别 | Rapier / 本系统能做 | 本系统不做（需流体求解器） |
|------|---------------------|---------------------------|
| 刚体飞行 | 重力、空气阻力（简化）、spin 轨迹 | — |
| 接触事件 | 入射角、速度、接触点、法线 | — |
| 水面响应 | 每次接触的能量损失 + 自旋耦合 → 浅反弹 → 自然再接触 | 表面张力分布、局部波动反馈、水膜形变、非对称水动力升力、湍流 / 涡结构 |

**一句话**：石头在重力 + 阻力场中运动，在水面发生 **多次非弹性碰撞**，每次碰撞损失部分能量，直到无法再离开水面——**skip 是这个连续过程的自然结果**，不是一次 `if/else` 判定。

设计原则：

| 原则 | 说明 |
|------|------|
| **物理是 truth** | Rapier 步进 + 触水后修改刚体状态，轨迹是唯一运动真相 |
| **skip 是 observation** | `skipCount` / `skipEvents` 从轨迹或接触日志 **事后检测**，不由触水模型直接返回 |
| **连续动力学，非离散规则** | 不做 `canSkip ? skip++ : sink`；做 `applyEnergyLoss()` → Rapier 继续积分 → 自然再碰撞或沉没 |
| **Clanet 等公式作系数参考** | 入射角 / 速度 / 自旋影响 **恢复系数 e**，而非 hard pass/fail 阈值 |

#### 职责

- 接收 `ThrowParams`，在 3D 世界中初始化石头刚体
- 逐步进重力、空气阻力、水面碰撞；每次触水修改刚体状态，由 Rapier 自然产生连续弹跳直至沉没
- 记录完整轨迹与触水接触日志
- **事后**从轨迹派生 skip 次数与落点时间线（驱动震动）

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
│   └── 触水回调 → WaterContactModel.apply()
└── gravity: (0, -9.81, 0)
```

| 对象 | 参数 |
|------|------|
| 石头 | 密度 ~3000 kg/m³（近似岩石），摩擦 0.3，恢复系数由触水模型 **每帧动态计算** |
| 空气阻力 | 每步对 `linvel` 施加 `-k * \|v\| * v`（k 可配置，默认 0.02） |
| 水面 | 静态 collider，厚度忽略；碰撞时调用 `WaterContactModel.apply()` |
| 时间步 | 固定 `dt = 1/120 s`，最多模拟 15 s 或满足 **终止条件**（见下） |

#### 水面接触模型（WaterContactModel）

Rapier 在每次触水时提供碰撞法线、相对速度、接触点、当前角速度。TS 层 **只计算并施加状态修改**，不返回 skip / sink：

```
触水瞬间（Rapier EventQueue）
    │
    ├─ Step 1：读取 contact → speed, incidentAngleDeg, spin, sideSpin
    │
    ├─ Step 2：计算能量损失系数
    │         e = f(angle, speed, spin)     // Clanet 启发的连续函数，非阈值
    │         spinRetention = f(angle, spin) // 自旋保留系数
    │
    ├─ Step 3：修改刚体状态
    │         v' = reflect(v, normal) * e + tangential retention
    │         ω' = ω * spinRetention
    │         （直接写 body.linvel() / body.angvel()）
    │
    └─ Step 4：记录 contactLog 条目，交还 Rapier 继续步进
              → 自然再次飞行 → 自然再次触水 … 直到终止
```

```typescript
interface WaterContactInput {
  speed: number;            // 入水速度标量 m/s
  incidentAngleDeg: number; // 速度方向与水面法线夹角（越小越平）
  spin: number;             // 绕 X 轴角速度 rad/s
  sideSpin: number;         // 绕 Z 轴 rad/s
  normal: { x: number; y: number; z: number };
  velocity: { x: number; y: number; z: number };
}

interface WaterContactRecord {
  t: number;
  x: number;
  speedBefore: number;
  speedAfter: number;
  incidentAngleDeg: number;
  restitution: number;      // 本次实际应用的 e
}

/** 只算系数，不做 skip/sink 判定 */
function computeRestitution(c: WaterContactInput, cfg: PhysicsConfig): number {
  const angleRad = (c.incidentAngleDeg * Math.PI) / 180;
  const angleFactor = Math.cos(angleRad);                          // 越平 e 越高
  const speedFactor = Math.min(c.speed / cfg.referenceSpeed, 1.5); // 参考速度归一化
  const spinLift = 1 + cfg.spinLiftK * Math.min(Math.abs(c.spin), cfg.maxSpinRad);
  const e = cfg.baseRestitution * Math.max(0, angleFactor) * speedFactor * spinLift;
  return Math.max(cfg.minRestitution, Math.min(e, cfg.maxRestitution));
}

function computeSpinCoupling(c: WaterContactInput, cfg: PhysicsConfig): number {
  const preserve = 1 - cfg.spinDampK * (c.incidentAngleDeg / 90);
  return Math.max(cfg.minSpinRetention, preserve);
}

/** 修改刚体状态；不返回 skip / sink，不递增 skipCount */
function applyWaterContact(
  body: RigidBody,
  input: WaterContactInput,
  cfg: PhysicsConfig
): { restitution: number; speedAfter: number } {
  const e = computeRestitution(input, cfg);
  const spinRetention = computeSpinCoupling(input, cfg);
  // 按法线分解速度，法向 * e，切向保留 cfg.tangentialRetention
  // 写回 body.linvel() / body.angvel() * spinRetention
  // ...
  return { restitution: e, speedAfter: /* ... */ };
}
```

**与旧设计的区别**：

| 旧（不推荐） | 新（本 PRD） |
|--------------|--------------|
| `resolveWaterContact()` 返回 `'skip' \| 'sink'` | `applyWaterContact()` 只改刚体状态 |
| `angle < 20° && speed > 2.5` → skip | `e = f(angle, speed, spin)` 连续映射 |
| 触水时 `skipCount++` | skip 事后从轨迹检测 |
| sink 时立刻 kinematic / 大阻尼 | 能量不足时 **自然** 无法离开水面，见终止条件 |

> Clanet 等打水漂经验关系用于 **校准 e 的曲线形状**，使「平角 + 高速 + 适当 spin → 更多次弹跳」，而非硬阈值门控。

#### Rapier EventQueue 处理（实现约束）

见 §2.4。每轮 `world.step()` 后 **排空当帧队列**再进入下一 step；禁止在碰撞回调内嵌套 `step()`。

```typescript
function stepWorld(world: World, eventQueue: EventQueue, ctx: SimContext): void {
  world.step();
  ctx.t += cfg.dt;
  recordTrajectory(ctx);

  eventQueue.drainCollisionEvents((h1, h2, started) => {
    if (!started) return;
    const contact = extractWaterContact(h1, h2, ctx);
    if (!contact) return;
    applyWaterContact(ctx.stoneBody, contact, cfg);
    ctx.contactLog.push(toRecord(contact, ctx.t));
  });

  ctx.terminated = checkTermination(ctx);
}
```

#### 模拟终止条件

模拟在以下 **任一** 条件满足时结束（均为连续物理的自然结果，非 skip 判定）：

| 条件 | 说明 |
|------|------|
| `t >= maxSimTime` | 默认 15 s 上限 |
| 触水后 `speedAfter < minBounceSpeed` | 默认 0.8 m/s；无法再产生有效反弹 |
| 石头 `y < sinkDepth` 且 `vy < 0` 持续 N 步 | 默认 sinkDepth = -0.05 m，N = 6；已没入水面 |
| 检测到 `maxContacts` 次触水 | 默认 50；防极端参数死循环 |

**不设** `maxSkips` 作为模拟循环退出条件；`maxContacts`（默认 50）仅防极端参数死循环。

#### Skip 检测（TrajectoryAnalyzer）

`skipEvents` 与 `skips` 为 **派生数据**，在 `simulate()` 结束后从 `trajectory` + `contactLog` 计算：

```typescript
interface SkipEvent {
  t: number;           // 秒，相对虚拟出手
  x: number;           // 侧视水平位置 m
  index: number;       // 第几次有效水漂（0-based）
  intensity: number;   // 0–1，供 Haptics 映射力度
  speedBefore: number; // m/s，触水前速度（debug / 校准）
}
```

`intensity` 由对应 `contactLog` 条目计算（**随物理，非固定 Medium**）：

```typescript
function computeSkipIntensity(contact: WaterContactRecord, cfg: PhysicsConfig): number {
  // 入水越猛、反弹后剩余动能越大 → intensity 越高
  const impact = contact.speedBefore / cfg.referenceSpeed;       // 默认 referenceSpeed 8 m/s
  const rebound = contact.speedAfter / cfg.referenceSpeed;
  const raw = 0.65 * impact + 0.35 * rebound;
  return clamp(raw, cfg.minSkipIntensity, 1); // minSkipIntensity 默认 0.25
}
```

> 同一趟水漂序列里，因 `speedBefore` 逐次下降，`intensity` 会 **自然递减**——无需人为「第 N 下打八折」。

function detectSkipEvents(
  trajectory: TrajectoryPoint[],
  contactLog: WaterContactRecord[],
  cfg: PhysicsConfig
): SkipEvent[] {
  // 策略 A（推荐）：从 contactLog 筛选「有效反弹」
  //   speedAfter >= minBounceSpeed 且 距上次 skip 间隔 >= minSkipIntervalMs
  // 策略 B（备选）：从 trajectory 检测 y≈0 处的 downward→upward 穿越
  // 两者应一致；不一致时以 contactLog 为准并打 debug 日志
}
```

检测规则（V1）：

| 规则 | 默认值 | 说明 |
|------|--------|------|
| `minBounceSpeed` | 0.8 m/s | 与终止条件共用；低于此速度的触水不计为 skip |
| `minSkipIntervalMs` | 80 ms | 过滤同一触水事件的重复采样 |
| `maxIncidentAngleDeg` | 35° | 过陡的触水（溅水）不计 skip，但仍参与物理 |

> **震动时序只读 `skipEvents`**，不读触水模型的返回值。轨迹上 `event: 'skip'` 字段由 `detectSkipEvents` 回填，供调试与 V2 回放。

#### 模拟流程

```typescript
async function simulate(params: ThrowParams): Promise<SimulationResult> {
  await Rapier.init();
  const world = createWorld(params);
  const trajectory: TrajectoryPoint[] = [];
  const contactLog: WaterContactRecord[] = [];
  let t = 0;
  let terminated = false;

  while (t < maxSimTime && !terminated) {
    stepWorld(world, eventQueue, ctx); // 见 EventQueue 小节
  }

  const skipEvents = detectSkipEvents(trajectory, contactLog, cfg);
  annotateTrajectoryEvents(trajectory, skipEvents, contactLog);

  return {
    skips: skipEvents.length,
    skipEvents,
    trajectory,
    contactLog,
    durationMs: t * 1000,
  };
}
```

#### 依赖与 Expo 集成

| 项 | 说明 |
|----|------|
| 包 | `@dimforge/rapier3d-compat`（WASM，Hermes 兼容） |
| Metro | WASM 作为 asset 打包；`metro.config.js` 增加 `wasm` 支持 |
| 初始化 | **`App.tsx` 挂载时 `Rapier.init()`**；`simulateChunked` 仅 `ensureRapierReady()` |
| 线程 | **JS 主线程 + 时间分片**（唯一 realistic 方案） |
| 降级 | WASM 加载失败时 **报错提示**，不 silently 降级为假物理 |

#### 线程策略：禁止 UI Worklet；Rapier 只能跑 JS Runtime

> **绝对禁止** 把 Rapier 放进 Reanimated UI Worklet 或 `react-native-worklets` 的 `createWorkletRuntime`。  
> 原因：① UI Worklet 阻塞会卡死画面；② **Worklet Runtime 无 WASM**，Rapier 无法加载。  
> **`expo-workers` 不存在于 RN 客户端**，勿采用。

**V1 唯一方案：JS 主线程时间分片（Chunking）**

不引入 Worker，把 `while` 循环拆成每帧一批：

```typescript
async function simulateChunked(
  params: ThrowParams,
  cfg: PhysicsConfig,
  opts?: { stepsPerFrame?: number; signal?: AbortSignal; onProgress?: (t: number) => void }
): Promise<SimulationResult> {
  const stepsPerFrame = opts?.stepsPerFrame ?? 100;
  await ensureRapierReady();
  const ctx = createSimulationContext(params, cfg);
  const signal = opts?.signal ?? createAppStateAbortSignal(); // background → abort

  while (!ctx.terminated) {
    const batch = Math.min(stepsPerFrame, ctx.remainingSteps);
    for (let i = 0; i < batch; i++) {
      if (signal.aborted) throw new SimulationAbortedError();
      stepWorld(ctx.world, ctx.eventQueue, ctx);
    }
    opts?.onProgress?.(ctx.t);
    await yieldFrame(signal); // rAF + 500ms 兜底，见 §2.4
  }
  return ctx.buildResult();
}
```

| 项 | 说明 |
|----|------|
| 每帧步数 | 默认 **100 步**（可调；低端机降至 50） |
| 让出策略 | `Promise.race([rAF, delay(500)])` + `AppState` abort |
| 中断处理 | reject → `haptics.cancel()` + UI 回 idle + Toast |
| UI 表现 | 「模拟中…」动画、落点计数保持流畅 |
| 接口 | `StonePhysicsEngine.simulate()` 内部调用 `simulateChunked()` |

**V1.1 可选优化（非 Worker 迁 WASM）**

| 方向 | 可行性 | 说明 |
|------|--------|------|
| 继续调优分片 | ✅ 推荐 | 增大 `stepsPerFrame`、预加载 WASM、InteractionManager |
| `react-native-worklets` 后台 Runtime | ⚠️ 不适用 Rapier | 可跑纯 JS，**无法** `WebAssembly.instantiate` |
| Native Module 绑 Rapier C++ | ✅ 长期 | 真正后台物理，工程量 V2 级 |
| `expo-workers` | ❌ | 服务端 Cloudflare Workers，与 App 无关 |

#### 输出

```typescript
interface TrajectoryPoint {
  t: number;    // 秒，零点 = 虚拟出手（t=0）
  x: number;    // 侧视水平 m（= world.x）
  y: number;    // 高度 m（= world.y）
  z: number;    // 横向 m（= world.z，V1 存档，侧视可忽略）
  /** 由 TrajectoryAnalyzer 事后标注，非物理步进时写入 */
  event?: 'release' | 'skip' | 'splash' | 'sink';
}

interface SimulationResult {
  skips: number;              // = skipEvents.length
  trajectory: TrajectoryPoint[];
  /** 派生自 trajectory + contactLog，供 HapticsScheduler */
  skipEvents: Array<{ t: number; x: number; intensity: number }>;
  contactLog: WaterContactRecord[];  // 每次触水明细，供调试 / 校准
  durationMs: number;
}
```

> **时间基准**：`trajectory[].t` 与 `skipEvents[].t` 均为相对虚拟出手的秒数。

#### 运动轨迹数据（V1 可获取，不渲染）

**可以获取。** 每次 `simulate()` 都会在 Rapier 步进中记录完整运动轨迹，作为 `SimulationResult.trajectory` 返回。V1 **不做画面回放**，但数据层完整保留，供调试、单测快照与后续 V2 可视化。

| 项 | 说明 |
|----|------|
| 采样频率 | 每物理步 1 点，`dt = 1/120 s` → 约 **120 Hz** |
| 典型点数 | 一次模拟约 **500–2000** 点（取决于飞行时长） |
| 坐标系 | 世界坐标 m；侧视 2D 取 `(x, y)`，`z` 保留供 V2 |
| 事件标注 | 模拟结束后 `annotateTrajectoryEvents()` 回填 `event` 字段 |
| 触水明细 | `contactLog` 记录每次触水的 `t / x / speedBefore / speedAfter / restitution` |

侧视 2D 轨迹提取（无需渲染即可使用）：

```typescript
function toSideView(trajectory: TrajectoryPoint[]): Array<{ t: number; x: number; y: number }> {
  return trajectory.map(({ t, x, y }) => ({ t, x, y }));
}
```

V1 获取方式：

| 场景 | 做法 |
|------|------|
| 开发调试 | `__DEV__` 下 `console.log(JSON.stringify(result.trajectory))` 或写入 `expo-file-system` |
| 单测 / 快照 | Jest 断言 `trajectory` JSON 与 golden file 一致 |
| 结果页传参 | `navigate('Result', { result })` 已携带 `trajectory`，V1 UI 不展示 |
| V2 回放 | 直接消费 `trajectory` + `skipEvents` 画 Canvas / Skia 曲线 |

> V1 不持久化轨迹到本地；需要历史轨迹时随 V1.1 本地记录一并存储。

#### 接口

```typescript
interface PhysicsConfig {
  gravity: number;              // 9.81
  releaseHeight: number;        // 1.2（石头初始 y）
  releasePositionX: number;     // 0
  stoneMass: number;            // 0.15
  stoneRadius: number;          // 0.04
  stoneHeight: number;          // 0.02
  airDragK: number;             // 0.02
  baseRestitution: number;      // 0.72（触水基准恢复系数）
  minRestitution: number;       // 0.05
  maxRestitution: number;       // 0.85
  referenceSpeed: number;       // 8.0 m/s（e 归一化参考速度）
  tangentialRetention: number;  // 0.92
  spinLiftK: number;            // 0.08
  spinDampK: number;            // 0.3
  minSpinRetention: number;     // 0.4
  maxSpinRad: number;           // 40
  minBounceSpeed: number;       // 0.8（终止 + skip 检测共用）
  minSkipIntensity: number;     // 0.25，末几下单震感仍可读
  minSkipIntervalMs: number;    // 80
  maxIncidentAngleDeg: number;  // 35（过陡触水不计 skip）
  sinkDepth: number;            // -0.05 m
  maxContacts: number;          // 50
  maxSimTime: number;           // 15（秒）
  dt: number;                   // 1/120
}

interface StonePhysicsEngine {
  /** V1：内部分片；V1.1 可切 Worker */
  simulate(params: ThrowParams, config?: Partial<PhysicsConfig>): Promise<SimulationResult>;
  simulateChunked(params: ThrowParams, config?: Partial<PhysicsConfig>): Promise<SimulationResult>;
}
```

> `simulate()` / `simulateChunked()` 均为 **async**；V1 二者等价（`simulate` 委托给 `simulateChunked`）。

#### 与 §5.2 集成

```typescript
const onThrowDetected = async (params: ThrowParams) => {
  setUiPhase('simulating');
  const result = await physicsEngine.simulateChunked(params);
  await haptics.play(result);
  navigate('Result', { result });
};
```

---

### 5.4 震动反馈（Haptics）

主玩法反馈，发生在 **模拟中（simulating）** 阶段；结果页展示前必须播完。  
**力度跟随水漂**：每次落点震感强度由 `skipEvents[i].intensity` 决定，模拟「越打越轻、第一下最脆」的真实节奏。

#### 播放时序

```
墙钟 t0 = play() 开始 = 物理 t=0
t0 + 0ms              → 出手确认震（1 次，固定 Medium）
t0 + skipEvents[0].t  → 落点 1，强度 intensity[0]（通常最强）
t0 + skipEvents[1].t  → 落点 2，强度 intensity[1]
...
末次震动后 + 300ms    → resolve → 结果页
```

| 时机 | 次数 | 力度 |
|------|------|------|
| 虚拟出手确认 | 1 次 | 固定 **Medium**（与物理无关，表「开演」） |
| 每个水漂落点 | 各 1 次 | **`mapIntensity(skipEvents[i].intensity)`** |
| 序列结束 | 0 次 | — |

#### 力度映射

| `intensity` | iOS `ImpactFeedbackStyle` | Android fallback `Vibration` |
|-------------|---------------------------|------------------------------|
| 0.00 – 0.35 | Light | `vibrate(25)` |
| 0.35 – 0.65 | Medium | `vibrate(45)` |
| 0.65 – 1.00 | Heavy | `vibrate(70)` |

```typescript
function mapImpactStyle(intensity: number): Haptics.ImpactFeedbackStyle {
  if (intensity < 0.35) return Haptics.ImpactFeedbackStyle.Light;
  if (intensity < 0.65) return Haptics.ImpactFeedbackStyle.Medium;
  return Haptics.ImpactFeedbackStyle.Heavy;
}

async function pulseSkip(intensity: number): Promise<void> {
  try {
    await Haptics.impactAsync(mapImpactStyle(intensity));
  } catch {
    if (Platform.OS === 'android') {
      const ms = Math.round(25 + intensity * 45); // 25–70ms
      Vibration.vibrate(ms);
    }
  }
}
```

> 阈值外置 `config/haptics.json`，真机校准：大力甩第 1 下应为 Heavy，末几下单 Light 仍可感知。

#### 实现策略

| 平台 | 主路径 | Fallback |
|------|--------|----------|
| iOS | `impactAsync(Light \| Medium \| Heavy)` | — |
| Android | 同上 | 按 intensity 映射震动时长 |

```typescript
interface HapticsScheduler {
  /**
   * 出手震：固定 Medium。
   * 落点震：按 skipEvents[i].intensity 映射 Light/Medium/Heavy（Android 映射时长）。
   * 墙钟排期见 §2.4；resolve 末次震动后 300ms。
   */
  play(result: SimulationResult): Promise<void>;
  cancel(): void;
}
```

#### 视觉 Fallback（Android 弱震动机型）

模拟中 UI 同步显示 **落点计数**；圆点大小或填充深度可随 `intensity` 变化（大点 = 强震），确保无震感时仍能感知 **次数 + 强弱节奏**。

> 不使用 `VibrationEffect.createOneShot` 原生 API（需 dev client）；V1 通过 `react-native` 内置 `Vibration` 覆盖。

---

### 5.5 UI 与数据（简要）

| 页面 | 内容 |
|------|------|
| 首页 | 开始投掷；后台已完成 Rapier 预加载 |
| 投掷页 | 首次轻提示 + 状态指示（待机 / 甩动中 / 模拟中 + 落点计数）+ 轻松模式条 |
| 结果页 | **Skips: N** + 一句随机评语 +「再玩一次」 |

**结果页评语**（零成本正反馈，非评分系统）：按 `skips` 区间从 `config/flavor-lines.json` 随机抽一句，例如：

| skips | 示例评语 |
|-------|----------|
| 0–2 | 「扑通。」 |
| 3–8 | 「水花不错。」 |
| 9–15 | 「水花消失术！」 |
| 16+ | 「湖面收割机。」 |

```typescript
function pickFlavorLine(skips: number): string {
  const bucket = FLAVOR_LINES.find((b) => skips >= b.min && skips <= b.max);
  return randomPick(bucket?.lines ?? FLAVOR_LINES[0].lines);
}
```

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
│   │   ├── throwDetectorCore.ts    # 纯函数状态机，Worklet + Jest 共用
│   │   ├── useThrowCapture.ts      # Worklet 升级（可选）
│   │   ├── ThrowDetector.ts        # expo-sensors 降级封装
│   │   └── thresholds.ts
│   ├── physics/
│   │   ├── ensureRapierReady.ts    # App 启动预加载 + simulate 内复用
│   │   ├── StonePhysicsEngine.ts   # simulateChunked 入口
│   │   ├── simulateChunked.ts      # rAF 时间分片
│   │   ├── rapier3d.ts             # Rapier 3D World 步进
│   │   ├── waterContactModel.ts    # 触水能量损失 + 自旋耦合（不返回 skip）
│   │   ├── trajectoryAnalyzer.ts   # 从轨迹 / contactLog 派生 skipEvents
│   │   └── airDrag.ts              # 空气阻力
│   ├── haptics/
│   │   └── HapticsScheduler.ts
│   └── types.ts
├── config/
│   ├── throw-detection.json
│   ├── physics.json
│   ├── flavor-lines.json
│   └── haptics.json              # intensity 分档阈值
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
| 单元测试 | `throwDetectorCore`（Node）、`waterContactModel`、`trajectoryAnalyzer`、Rapier 快照 |
| Worklet A/B | Week 1 真机：expo-sensors vs useAnimatedSensor，对比 actualHz + 识别稳定性 |
| Fixture 回放 | CSV（m/s²）→ `feed()` → 断言 `ThrowParams` |
| 快照测试 | 固定 `ThrowParams` → `simulate()` → `skipEvents` / 轨迹 JSON 快照 |
| Rapier | WASM 初始化 + 10 次 simulate 不崩溃；velocity ↑ → skips 单调趋势；**同参数双跑 skipEvents 一致** |
| 物理回归 | 平角高速 > 陡角低速 skips；触水次数 = contactLog.length；skipEvents ⊆ contactLog |
| 真机 | 3 种力度 × 2 平台，校准 `kVel` 与阈值 |
| 震动 | 落点次数与 skips 一致；**同序列 intensity 单调趋势不增**（随能量衰减）；或视觉计数 + 点大小 fallback |
| 工具 | **jest-expo** preset，core 模块在 Node 环境运行 |

---

## 8. MVP 验收标准

### 必须实现

- [ ] 加速度归一化正确（DeviceMotion 含重力路径：静止 ≈ 9.8 m/s²）
- [ ] `expo-sensors` DeviceMotion 双端采样，actualHz ≥ 50 Hz
- [ ] 2 s 环形缓冲，样本时间单调
- [ ] 虚拟出手识别（峰值 + 回落）
- [ ] `throwDetectorCore` + CSV fixture 单测通过
- [ ] 速度：`estimateVelocity`（峰值 + 持续时间 + clamp）+ 形态异常降 confidence
- [ ] 连续 2 次失败自动轻松模式 + `invalidReason` 具体文案
- [ ] Rapier EventQueue 排空后步进；`simulateChunked` 含 rAF 兜底 + AppState abort
- [ ] `HapticsScheduler` 墙钟排期（非预排物理 t）
- [ ] App 启动时 `Rapier.init()` 预加载
- [ ] 结果页随机评语（`flavor-lines.json`）
- [ ] 重力对齐：`a_user` / `a_horiz` / `|ω_horiz|` 提取，不依赖固定握姿
- [ ] 首次引导：水平甩出文案（无横竖示意图）
- [ ] 无效动作提示重试，不进入模拟
- [ ] Rapier 3D + **simulateChunked** 时间分片（模拟中 UI 不卡死）
- [ ] `WaterContactModel` 只修改刚体状态；`skipEvents` 由 `TrajectoryAnalyzer` 派生
- [ ] `simulate()` async，输出 skips / skipEvents / trajectory / contactLog
- [ ] 落点震动与 skips 一致，**力度随 `skipEvents[].intensity` 变化**
- [ ] 结果页在 `HapticsScheduler.play()` resolve **之后** 展示
- [ ] 结果页展示水漂次数
- [ ] 同参数双端 skips 一致
- [ ] 核心逻辑 Jest 覆盖率 ≥ 80%

### V1 可选（Worklet 升级）

- [ ] Week 1 A/B 验证通过：`useAnimatedSensor` + Worklet 识别更稳或 actualHz 更高
- [ ] Worklet 路径单独阈值配置（用户加速度模型）

### V1.1 可选

- [ ] 自研 Native Module（expo-sensors + Worklet 均不达标）
- [ ] Native Rapier C++ 后台物理（替代 WASM 分片）
- [ ] 去重力用户加速度
- [ ] 首次校准引导
- [ ] 本地历史记录（含 trajectory JSON）
- [ ] 评分 / 等级系统

### 暂不实现

- 轨迹可视化 / 3D 或 2D 回放（**数据已在 `SimulationResult.trajectory`，仅 UI 不做**）
- CFD / SPH 真实流体模拟
- Rapier 2D（侧视简化版，不采用）
- 在线多人、排行榜
- ML 动作识别

---

## 9. 风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| 虚拟出手误识别 | 核心体验 | 形态 confidence、轻松模式、握持引导 |
| simulateChunked 切后台死锁 | UI 永驻 simulating | rAF 500ms 兜底 + AppState abort |
| 尖峰加速度 → 离谱初速 | 轨迹飞出天际 | durationFactor + maxVelocity + confidence 形态校验 |
| 传感器采样率不足 | 漏检峰值 | expo-sensors 基线；Week 1 A/B 后切 Worklet；仍不行则 Native |
| Worklet 加速度语义不同 | 阈值失效 | 用户加速度模型单独校准 |
| Worklet 与 expo-sensors 识别不一致 | 测试盲区 | core 单测 + 真机 A/B |
| 物理分片 wall time 过长 | 模拟等待感 | 调 `stepsPerFrame`、WASM 预加载 |
| 握持方向随意 | 轴映射漂移 | 重力对齐 + 标量峰值；形态 confidence 兜底 |
| 双端手感不一致 | skips 偏差 | 分平台 kVel 校准；同参数双端快照回归 |
| Android 震动弱 | 无法感知节奏 | `Vibration` fallback + 模拟中视觉计数 |
| 后台打断震动序列 | 结果页异常 | 后台 `cancel()`，回前台提示重试 |
| Rapier 3D WASM 兼容性 | 无法模拟 | Metro 配置 + 启动预加载；失败时明确报错 |
| 物理模型可玩性 vs 真实感 | skips 过少 / 过多 | Clanet 曲线校准 `baseRestitution` / `referenceSpeed`；真机 kVel 联动 |
| 物理步进阻塞 UI | 画面卡死 | 禁止 UI Worklet 跑 Rapier；用 simulateChunked 分片 |
| 触水检测与 skip 派生不一致 | 震动次数异常 | contactLog 为权威源；trajectory 穿越作交叉校验 |
| 仅靠震动反馈 | 新用户不理解 | 首次引导 + 模拟中落点计数 |

---

## 10. 实施顺序建议

| 阶段 | 交付 |
|------|------|
| Week 1 | expo-sensors 基线 + `throwDetectorCore` + actualHz 真机 + **Worklet A/B** |
| Week 2 | 选定采集路径 + `estimateVelocity` / `motionMetrics` 单测 + fixture |
| Week 3 | Rapier 3D + `simulateChunked` + WaterContactModel + TrajectoryAnalyzer + Haptics |
| Week 4 | 投掷页 / 结果页 + simulating 时序 + 双端真机校准（kVel + 物理参数） |

---

## 11. 一句话总结

React Native + expo-sensors（基线）/ Reanimated Worklet（可选升级）识别甩动，**Rapier 3D** 在 JS 线程分片模拟，TrajectoryAnalyzer 派生水漂次数，expo-haptics 震出节奏。

> **Review 备忘**：不存在 `expo-workers` 客户端包；Rapier WASM 只能跑 JS Runtime，不能跑 Worklet Runtime。

> **物理定位**：真实刚体飞行 + 经验水动力接触模型；skip 是连续 rebounce 的观测结果，不是触水模型的返回值。
