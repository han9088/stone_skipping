# Stone Skip Simulator / 打水漂体感模拟器

基于手机体感（加速度 + 陀螺仪）的打水漂模拟应用。甩动手机识别虚拟出手，实时震动反馈水漂落点，结束后可选 3D 回放。

## 文档

| 文档 | 说明 |
|------|------|
| [docs/PRD.md](docs/PRD.md) | 产品需求（引擎无关） |
| [docs/PRD-Godot4.md](docs/PRD-Godot4.md) | Godot 4 技术版 PRD |
| [docs/Godot4-Learning-Roadmap.md](docs/Godot4-Learning-Roadmap.md) | Godot 4 学习路线与实现指南 |
| [docs/Unity-Learning-Roadmap.md](docs/Unity-Learning-Roadmap.md) | Unity 学习路线（备选） |

## 技术栈（规划）

- **主选**：Godot 4 + GDScript
- **平台**：iOS 14+ / Android 8.0+
- **备选**：Unity 6

## 核心体验

```
甩动 → 识别虚拟出手 → 实时模拟 + 每落点震一次 → 结果页 → [可选] 3D 回放
```

## Cursor Cloud Agents

本仓库已配置 `.cursor/environment.json`，可在 [Cursor Cloud Agents](https://cursor.com/agents) 中连接 GitHub 仓库后使用云端 Agent 继续开发。
