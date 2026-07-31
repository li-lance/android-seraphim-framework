# KMP 每日任务应用 —— 主流架构评估与从零设计

> 分支：`developer` · 日期：2026-07-31 · 状态：设计基线 v1
>
> 目标：从 0 设计一个跨平台（Android / iOS / Desktop）项目，用于记录个人每天的任务。

---

## 1. 2026 年 KMP 生态现状（调研结论）

| 维度 | 现状 |
|---|---|
| KMP 核心 | 2023-11 起官方 Stable，Google 官方推荐用于共享业务逻辑 |
| Compose Multiplatform (CMP) | iOS 端 2025-05（1.8.0）起 Stable；Desktop 早已 Stable；Web (Wasm) 接近成熟 |
| 生产验证 | Netflix、McDonald's、Forbes、Cash App、Duolingo 等在生产环境使用；Forbes 共享逻辑超 80% |
| 标准技术栈 | Ktor（网络）+ SQLDelight/Room KMP（存储）+ Koin（DI）+ Coroutines/Flow（异步）+ Kotlinx Serialization |
| Swift 互操作 | SKIE（Touchlab）或 KMP-NativeCoroutines，自动把 Flow 桥接为 Swift async/await |
| IDE | IntelliJ IDEA（2025 统一版）/ Android Studio + KMP 插件；Fleet 已停止 |
| 关键限制 | iOS 产物必须在 macOS 上编译；CI 需 macOS runner |

**行业共识**：MVI + 单向数据流（UDF）是 2026 年 KMP 表现层的首选模式；Shared ViewModel 可共享约 85% 的表现层逻辑。

---

## 2. 主流 KMP 架构方案评估

### 2.1 总体路线：三种形态

| 方案 | 共享范围 | 代码复用 | 优点 | 缺点 | 适用 |
|---|---|---|---|---|---|
| **A. 共享逻辑 + 原生 UI** | commonMain 放网络/存储/用例/ViewModel；Android 用 Compose，iOS 用 SwiftUI | ~60–70% | 100% 原生体验、无障碍、平台集成最深；风险最低 | UI 写两遍 | 重平台体验、长期维护的产品 |
| **B. 全共享 CMP UI** | UI 也用 Compose Multiplatform 写一份 | ~85–95% | 复用最高、单团队即可 | iOS 上是 Skia 渲染而非 UIKit 组件，视觉/无障碍与原生有差异；三方 SDK 接入略摩擦 | 表单/列表类工具应用、个人项目、MVP |
| **C. 混合（CMP 为主 + 原生补缺）** | CMP 写大部分页面，个别页面（如系统组件密集的）用原生 | ~80% | 灵活 | 架构边界要自律 | 大多数现实项目 |

### 2.2 表现层模式

| 模式 | 评估 |
|---|---|
| **MVVM**（多 observable 属性） | 熟悉但多个状态源（isLoading / data / error）在 Compose 与 SwiftUI 双端消费时易出现竞态与不一致 |
| **MVI / UDF** ✅ | 单一不可变 `ViewState` + Intent 驱动，天然契合声明式 UI；调试可回放、测试容易；2026 行业标准 |
| **Decompose 组件化** | 把导航+生命周期+状态全下沉到共享层，iOS 用 SwiftUI 包一层；强大但学习曲线陡，个人项目偏重 |

### 2.3 关键库选型对比

| 层 | 候选 | 结论 |
|---|---|---|
| 导航 | Voyager / Decompose / **Navigation3 (androidx, 多平台)** | 新项目用 CMP 选 **Navigation3**（官方、类型安全、生命周期对齐）；Decompose 适合重度组件化 |
| DI | **Koin** / kotlin-inject / Metro | **Koin**：KMP 生态最成熟、无 KSP 重生成、iOS 配置简单；Metro/kotlin-inject 编译期更快但生态较新 |
| 本地存储 | **SQLDelight 2.x** / Room KMP 3.0 | 两者皆生产可用。**SQLDelight**：SQL 即源码、编译期校验、全平台一致；Room KMP 适合从 Android 迁移的团队。本项目从 0 开始 → **SQLDelight** |
| 键值/偏好 | **DataStore (multiplatform)** | 官方、支持 KMP，存设置/标记 |
| 网络 | **Ktor Client** | OkHttp(Android) / Darwin(iOS) 引擎自动切换；插件化（重试/日志/序列化） |
| 序列化 | **Kotlinx Serialization** | 标准答案 |
| 协程桥接 | **SKIE** | 若 iOS 用 SwiftUI：Flow → async/await 自动桥接，免胶水代码 |
| 图片 | Coil 3（全 KMP） | 需要时引入 |
| 时间 | **kotlinx-datetime** | 任务日期/提醒的核心依赖 |

### 2.4 评估结论（本项目决策）

这是一个**个人任务记录工具**：页面以列表/表单/日历为主，无复杂平台专属 UI，单人或小团队维护。因此：

> **路线 A（共享逻辑 + 原生 UI），MVI + Shared ViewModel，SQLDelight 本地优先存储，Koin DI；多仓模式（repo 工具）管理。**
>
> 决策记录（v2，2026-07-31）：放弃初稿的路线 B。理由：① 100% 原生体验与无障碍，iOS 用 SwiftUI 无 Skia 渲染差异；② 团队现有工作流就是 repo 工具多仓模式，子仓独立演进；③ STRV 等生产实践验证「共享逻辑 + 原生 UI + monorepo/多仓」是风险最低的组合。Navigation3 不再适用（UI 不共享），导航归各平台原生（Android: Navigation3/Jetpack Navigation，iOS: SwiftUI NavigationStack）。

---

## 3. 从零设计：每日任务应用「DailyLog」（多仓模式）

### 3.0 仓库拓扑（repo 工具）

```
android-seraphim-framework (顶层仓 = 工作区编排)
├── manifests/default.xml   # repo 清单（唯一事实源）
├── docs/                   # 架构文档
└── <以下子仓由 repo sync 拉取>
    ├── build/              # dailylog-build.git     · 约定插件 + 版本目录
    ├── shared/             # dailylog-shared.git    · KMP 共享逻辑层
    └── apps/
        ├── android/        # dailylog-android.git   · 原生 Android (Compose)
        └── ios/            # dailylog-ios.git       · 原生 iOS (SwiftUI + Xcode)
```

规则：子仓各自独立构建、独立发版；`shared` 以源码方式被 Android 端复合构建（`includeBuild`）消费，iOS 端以 XCFramework 消费；跨仓契约（Shared 的公开 API）变更需在顶层仓 docs 记录。

### 3.1 产品范围（MVP → 进阶）

- **MVP**：今日任务列表（增删改查、勾选完成）、按日期浏览、任务备注、本地持久化
- **V1.1**：重复任务（每日/每周）、提醒通知、标签分类
- **V1.2**：统计视图（完成率、连续打卡 streak）、深色模式、数据导出
- **V2.0**：可选云同步（端到端加密）、桌面端、小组件

### 3.2 架构总览（共享逻辑 + 原生 UI）

```
┌───────────────────────────┐   ┌───────────────────────────┐
│  apps/android (Compose)   │   │  apps/ios (SwiftUI)       │
│  · Screens (dumb)         │   │  · Views (dumb)           │
│  · Navigation3            │   │  · NavigationStack        │
│  · 观察 StateFlow          │   │  · 经 SKIE 观察 Flow       │
└─────────────┬─────────────┘   └─────────────┬─────────────┘
              │  includeBuild 复合构建          │  XCFramework
┌─────────────┴───────────────────────────────┴─────────────┐
│  shared (dailylog-shared.git · commonMain 逻辑层 — 全 MVI) │
│   ├─ presentation   Shared ViewModel:                     │
│   │                 Intent → State (StateFlow)            │
│   │                 + Effect (一次性事件)                  │
│   ├─ domain         UseCase / 业务规则 / 模型              │
│   ├─ data           Repository 实现                        │
│   │   ├─ local      SQLDelight (.sq)                      │
│   │   └─ prefs      DataStore                            │
│   └─ di             Koin modules                          │
└───────────────────────────────────────────────────────────┘
```

**单向数据流**：`UI → Intent → ViewModel(reduce) → ViewState(StateFlow) → UI`；
副作用（toast、导航、提醒调度）走独立 `Effect` 通道（SharedFlow），避免状态里塞一次性事件。

### 3.3 各仓内部结构

```
shared/                    # KMP 库仓（独立 Gradle 构建，产出 Android AAR + iOS XCFramework）
└── src/
    ├── commonMain/        # ViewModel / domain / data / di（逻辑能放这就放这）
    ├── commonTest/        # 共享单元测试（一次编写全端跑）
    ├── androidMain/       # expect/actual: DriverFactory、通知调度入口
    └── iosMain/           # expect/actual 实现（nativeMain）

apps/android/              # 原生 Android 仓（Jetpack Compose + Navigation3）
└── app/src/main/          # Screens / theme / platform 集成；ViewModel 来自 shared

apps/ios/                  # 原生 iOS 仓（SwiftUI + Xcode 工程）
└── DailyLog/              # Views / ObservableObject 包装 / 平台集成；逻辑来自 shared XCFramework

build/                     # 构建设施仓（约定插件 + gradle/libs.versions.toml 版本目录）
```

原则：逻辑能放 `commonMain` 就放 `commonMain`，`expect/actual` 只用于真正平台相关的窄接口（通知调度、数据库驱动、文件导出）；UI、导航、生命周期全部留在平台侧，共享层只暴露状态流与 Intent 入口。

### 3.4 数据模型（SQLDelight `.sq` 草案）

```sql
CREATE TABLE task (
    id          TEXT PRIMARY KEY NOT NULL,          -- UUID
    title       TEXT NOT NULL,
    note        TEXT,
    date        TEXT NOT NULL,                      -- ISO LocalDate, 归属日期
    status      INTEGER NOT NULL DEFAULT 0,         -- 0=todo 1=done
    priority    INTEGER NOT NULL DEFAULT 0,         -- 0=none 1=low 2=mid 3=high
    tags        TEXT NOT NULL DEFAULT '',           -- 逗号分隔（MVP）
    recurrence  TEXT,                               -- RRULE（V1.1）
    reminder_at TEXT,                               -- ISO Instant（V1.1）
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL,
    deleted     INTEGER NOT NULL DEFAULT 0          -- 软删除，为将来同步铺路
);

CREATE INDEX idx_task_date ON task(date, deleted);
```

配套查询：`selectByDate` / `insert` / `toggleStatus` / `update` / `softDelete` / `statsCompletionRate`，全部经 SQLDelight 生成类型安全 API；Repository 暴露 `Flow<List<Task>>`（`.asFlow().mapToList()`），数据变化自动驱动 UI。

### 3.5 表现层契约（MVI 示例）

```kotlin
data class TaskListState(
    val date: LocalDate,
    val tasks: List<Task> = emptyList(),
    val loading: Boolean = true,
    val error: String? = null,
)

sealed interface TaskListIntent {
    data class SelectDate(val date: LocalDate) : TaskListIntent
    data class Toggle(val id: String) : TaskListIntent
    data class Delete(val id: String) : TaskListIntent
    data object AddRequested : TaskListIntent
}

sealed interface TaskListEffect {
    data class ShowUndoDelete(val id: String) : TaskListEffect
    data object NavigateToEditor : TaskListEffect
}

class TaskListViewModel(
    private val observeTasks: ObserveTasksByDate,
    private val toggleTask: ToggleTaskStatus,
    ...
) : ViewModel() {
    val state: StateFlow<TaskListState>
    val effect: SharedFlow<TaskListEffect>
    fun onIntent(intent: TaskListIntent)
}
```

### 3.6 平台相关窄接口（expect/actual）

| 接口 | Android | iOS | Desktop |
|---|---|---|---|
| `DatabaseDriverFactory` | AndroidSqliteDriver | NativeSqliteDriver | JdbcSqliteDriver |
| `NotificationScheduler`（V1.1） | AlarmManager + WorkManager | UNUserNotificationCenter | 系统托盘/无 |
| `FileExporter`（V1.2） | SAF | UIDocumentPicker | JFileChooser |

### 3.7 测试与 CI

- **commonTest**：UseCase、reducer、Repository（内存 Driver）——一次编写，Android/iOS 双端同跑
- **CI（GitHub Actions）**：每仓独立流水线；`shared` 在 Linux runner 跑 Android 编译与测试、macOS runner 编译 XCFramework；`apps/ios` 必须 macOS runner
- 本机注意：Windows 无法编译 iOS 产物，iOS 端验证需在 macOS 或 CI 上进行

### 3.8 实施路线图

| 阶段 | 内容 | 验收 |
|---|---|---|
| P0 仓库就位 | 创建 4 个 GitHub 空仓 → `repo sync` → 各仓最小可构建骨架（build 仓约定插件 + 版本目录先行） | 四仓同步成功、shared 空库编译通过 |
| P1 数据层 | .sq 建模、Repository、commonTest 覆盖 | 单测通过 |
| P2 今日任务 | shared 侧 TaskList MVI + Android 页面（列表/勾选/删除撤销） | Android 可用 |
| P3 编辑与日历 + iOS | 编辑器、日期切换；iOS SwiftUI 壳接入 XCFramework（需 macOS） | MVP 完成 |
| P4 打磨 | 统计、深色模式、导出、重复任务与提醒 | V1.x |

---

## 4. 决策记录（ADR 摘要）

| # | 决策 | 理由 |
|---|---|---|
| 1 | ~~路线 B：CMP 全共享 UI~~ → **v2 改为路线 A：共享逻辑 + 原生 UI**（2026-07-31） | 原生体验/无障碍最佳；与团队多仓工作流匹配；STRV 生产实践验证 |
| 2 | MVI + Shared ViewModel | UDF 契合声明式 UI；行业标准；可共享 85% 表现逻辑 |
| 3 | SQLDelight 而非 Room | 从 0 开始、编译期 SQL 校验、全平台一致；软删除为同步铺路 |
| 4 | Koin DI | KMP 最成熟、iOS 配置简单、无重代码生成 |
| 5 | ~~Navigation3 共享导航~~ → **导航归平台原生**（Android Navigation3 / iOS NavigationStack） | UI 不共享后共享导航失去意义（v2） |
| 6 | 本地优先、可选云同步 | 个人数据工具先做可靠本地存储，同步作为 V2 增量 |
| 7 | 保留 repo 工具多仓模式（v2） | 子仓独立演进；顶层仓只做编排（清单 + 文档）；build 仓统一约定插件与版本目录 |
