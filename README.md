# DailyLog Workspace

> 跨平台「每日任务记录」应用的工作区编排仓（多仓模式，Google `repo` 工具管理）。
> 架构路线：**共享逻辑 + 原生 UI**（KMP Shared Logic + Jetpack Compose / SwiftUI）。

---

## 仓库拓扑

```
android-seraphim-framework/        # 本仓 = 工作区编排（清单 + 文档）
├── manifests/default.xml          # repo 清单（唯一事实源）
├── docs/                          # 架构文档
└── <repo sync 拉取的子仓>
    ├── build/                     # dailylog-build.git   · 约定插件 + 版本目录
    ├── shared/                    # dailylog-shared.git  · KMP 共享逻辑层（MVI）
    └── apps/
        ├── android/               # dailylog-android.git · 原生 Android (Compose)
        └── ios/                   # dailylog-ios.git     · 原生 iOS (SwiftUI)
```

| 子仓 | 路径 | 职责 |
|---|---|---|
| dailylog-build | `build/` | Gradle 约定插件、`libs.versions.toml` 版本目录 |
| dailylog-shared | `shared/` | 全部业务逻辑：MVI ViewModel / UseCase / Repository / SQLDelight / Koin |
| dailylog-android | `apps/android/` | 原生 Android App：Jetpack Compose + Navigation3，`includeBuild` 消费 shared |
| dailylog-ios | `apps/ios/` | 原生 iOS App：SwiftUI + NavigationStack，XCFramework 消费 shared |

## 技术选型

| 层 | 选型 |
|---|---|
| 架构模式 | MVI + 单向数据流 + Shared ViewModel |
| 本地存储 | SQLDelight（本地优先，软删除为云同步铺路） |
| 依赖注入 | Koin |
| 异步 | Kotlin Coroutines / Flow（iOS 经 SKIE 桥接） |
| 序列化 / 时间 | Kotlinx Serialization / kotlinx-datetime |
| 导航 | 平台原生：Android Navigation3 · iOS NavigationStack |

详细设计见 [docs/kmp-daily-task-app-architecture.md](docs/kmp-daily-task-app-architecture.md)。

## 快速开始

```bash
# 1. 安装 repo 工具（Windows Git Bash 已配置于 ~/bin/repo）
# 2. 同步全部子仓
repo init -u git@github.com:li-lance/android-seraphim-framework.git -b developer -m manifests/default.xml
repo sync --no-repo-verify   # Windows Git Bash 下需 --no-repo-verify

# 3. 构建 Android App
cd apps/android && ./gradlew assembleDebug
```

> 注意：iOS 产物需在 macOS 上编译（本机或 CI 的 macOS runner）。

## 历史

- `master` 分支：旧 Seraphim Framework（多应用 KMP 框架，已归档）
- `developer` 分支：DailyLog 重构（2026-07，推翻旧设计从零开始）

## 许可证

Apache License 2.0，见 [LICENSE](LICENSE)。
