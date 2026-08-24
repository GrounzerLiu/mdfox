# MDFox Compose 迁移追踪

> 目标：将 MDFox（固定版 mozilla-central `092b4be38e4fa` / Firefox 156.0a1）Fenix 的**库/历史界面**从传统 View 架构迁移到 Jetpack Compose。
> 起点界面：`mobile/android/fenix/app/src/main/java/org/mozilla/fenix/library`（Compose 渗透率 3%，全库最低）。
> 每次改动后必须 `./mach gradle :fenix:assembleDebug` 验证编译通过。

---

## 1. 现状（2026-08-24 实测）

Fenix app 全局：1389 个 Kotlin 文件，356 个用 Compose（25.6%），293 个含 `@Composable`，131 个 XML 布局。

| 界面 | Compose/总文件 | 状态 |
| --- | --- | --- |
| 标签页 TabsTray | 39/88 | 已全面 Compose（material3 + Redux） |
| 主页 Home | 48/128 | 大量 Compose（HomeScreenViewModel） |
| 设置 Settings | 54/269 | 混合（28 个 PreferenceFragmentCompat） |
| 主浏览 chrome | 9/41 | 混合（EngineView + ac View 工具栏 + 自研 Compose 工具栏） |
| **库/历史 Library** | **1/30** | **几乎全 View（本迁移对象）** |
| 通用 components | 33/191 | 混合 |

### 库/历史界面结构（`library/`）

- `history/`：HistoryFragment / HistoryView / HistoryAdapter / HistoryDataSource（Paging3）/ HistoryFragmentStore / viewholders / middleware
  - ⚠️ 搜索体验（awesomebar、toolbar）**已是 Compose**（`mozilla.components.compose.browser.awesomebar.AwesomeBar`、`BrowserToolbarStore`）
  - 列表主体仍是 View（RecyclerView.Adapter）
- `historymetadata/`：HistoryMetadataGroupFragment / Adapter / ViewHolder / View
- `recentlyclosed/`：RecentlyClosedFragment / View / Adapter / ItemViewHolder / Interactor / Controller / Store
- 根：LibraryPageFragment（基类）/ LibraryPageView / LibrarySiteItemView（自定义 View）

主题系统：**AcornTheme**（Mozilla 新设计系统），入口 `FirefoxTheme { }`，提供 `colors/typography/layout/gradients/windowSize`。

## 2. 迁移原则

1. **保留 Redux/MVI 架构**：Store / State / Action / Controller / Interactor 不动，只替换 View 层（Fragment 渲染 + Adapter + ViewHolder + XML）
2. **用 `FirefoxTheme { }` 包裹所有 Compose 界面**，颜色/字体走 Acorn tokens（不写死颜色）
3. **状态通过 `consumeFrom` / `collectAsState` 订阅 Store**，不引入新状态管理
4. **菜单（MenuProvider）、对话框暂用 View 实现**（后续单独 Compose 化），优先迁移列表主体
5. 参考已 Compose 化界面：`tabstray/ui/tabitems/BasicTabListItem.kt`（列表项）、`tabstray/ui/tabpage/EmptyTabPage.kt`（空状态）、`tabstray/ui/tabsearch/TabSearchScreen.kt`（Screen）、`home/topsites/ui/ShortcutsScreen.kt`
6. 迁移后删除不再使用的 View 层文件（Adapter / ViewHolder / FragmentView / XML），保持代码库干净

## 3. 工作流（编辑 → 同步 → 验证）

源码真身在 WSL（`/home/grounzer/mdfox-src`，构建环境）。Windows 侧沙箱只能写 `D:\any\`，故用镜像工作区：

1. `D:\any\mdfox-work\` 镜像待改文件（从 WSL 拷出）
2. 在 `D:\any\mdfox-work\` 编辑（write/edit）
3. `cp` 同步回 WSL（`/mnt/d/any/mdfox-work/...` → `/home/grounzer/mdfox-src/...`）
4. WSL 构建验证：`./mach gradle :fenix:assembleDebug`（增量，约 5-10 分钟）
5. 更新本文档 + commit 到 mdfox 仓库

> ⚠️ 不要覆盖 WSL 中已适配的文件：`mobile/android/fenix/app/build.gradle`、`.../values/static_strings.xml`（MDFox 定制版）。

## 4. 迁移清单与状态

| # | 界面 | 文件 | 状态 | 里程碑 |
| --- | --- | --- | --- | --- |
| 1 | **RecentlyClosed（最近关闭）** | `library/recentlyclosed/*` + `component_recently_closed.xml` | ✅ 已完成（编译通过） | M1 |
| 2 | History 列表主体 | `library/history/*`（Adapter/ViewHolder/View） | 🔴 未开始 | M2 |
| 3 | HistoryMetadataGroup | `library/historymetadata/*` | 🔴 未开始 | M3 |
| 4 | Library 入口页 | `library/LibraryPageFragment.kt` 等 | 🔴 未开始 | M4 |

## 5. 里程碑记录

### M1：RecentlyClosed Compose 化（进行中）

**目标**：把"最近关闭"页从 Fragment + RecyclerView 迁移为 Compose Screen（保留 Store/Controller/Interactor）。

**涉及文件**：

| 文件 | 动作 |
| --- | --- |
| `recentlyclosed/RecentlyClosedScreen.kt` | 新增：Compose Screen（列表 + 空状态 + viewMoreHistory） |
| `recentlyclosed/RecentlyClosedFragment.kt` | 修改：onCreateView 改为 ComposeView + setContent；保留 store/controller/interactor/menu |
| `recentlyclosed/RecentlyClosedFragmentView.kt` | 删除（View 层被 Compose 取代） |
| `recentlyclosed/RecentlyClosedAdapter.kt` | 删除 |
| `recentlyclosed/RecentlyClosedItemViewHolder.kt` | 删除 |
| `res/layout/component_recently_closed.xml` | 删除（不再使用） |
| `res/layout/fragment_recently_closed_tabs.xml` | 删除（不再使用） |
| `res/layout/history_list_item.xml` | 暂保留（History 仍用） |

**Compose 结构**：

```kotlin
FirefoxTheme {
  RecentlyClosedScreen(
    state = store.state.collectAsState(),
    interactor = interactor,
  )
}
```

- 列表：`LazyColumn`，首项为 "显示完整历史"（原 viewMoreHistory），其后每项为 RecentlyClosedItem
- 空状态：居中文本 `recently_closed_empty_message`
- 列表项：title（空则 url）+ url + favicon（`loadFavicon` 的 Compose 等价：`mozilla.components.compose.engine`? 待定）+ 删除按钮（`mozac_ic_cross_24`）
- 多选：store.selectedTabs 驱动选中态（参考 `TabsTrayItemSelectionState.kt`）

**迁移日志**：

| 日期 | 改动 | 验证 |
| --- | --- | --- |
| 2026-08-24 | 新增 `RecentlyClosedScreen.kt`（Compose：LazyColumn + 空状态 + 显示完整历史项，基于 `FaviconListItem`/`IconListItem`）；`RecentlyClosedFragment.kt` 改为 ComposeView 宿主（`observeAsComposableState` 订阅 Store，保留 Store/Controller/Interactor/菜单）；`RecentlyClosedInteractor` 接口移入 `RecentlyClosedFragmentInteractor.kt`；删除 `RecentlyClosedFragmentView.kt`/`RecentlyClosedAdapter.kt`/`RecentlyClosedItemViewHolder.kt`/`component_recently_closed.xml`/`fragment_recently_closed_tabs.xml` | `:fenix:assembleDebug` BUILD SUCCESSFUL（3m33s） |

---

## 6. 待办 / 风险

- [ ] M1 RecentlyClosed 编译通过
- [ ] M1 真机/模拟器验证（可选）
- [ ] M2 History 列表 LazyColumn 化（含 Paging3 适配 `collectAsLazyPagingItems`）
- [ ] 迁移期间保持固定版本 `092b4be38e4fa`，不跟随上游
- [ ] 风险：favicon 加载（`loadFavicon` 在 View 上是扩展函数，Compose 侧需找等价或桥接）
- [ ] 风险：多选手势/长按（原 setSelectionInteractor）需用 Compose 手势重写
