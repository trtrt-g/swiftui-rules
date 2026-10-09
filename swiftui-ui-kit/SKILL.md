---
name: swiftui-ui-kit
description: >-
  SwiftUI UI 手册：{Project}Scaffold、自定义 Tab、375 布局、Sheet/HUD、切图按钮、
  DiamondBadge、Paging。P0 见 swiftui-scaffold-navbar、swiftui-colors。
---

# SwiftUI UI Kit

**P0 Rules**：`swiftui-scaffold-navbar`、`swiftui-colors`（`~/.cursor/rules/swiftui/`）

组件在 `Core/UI/{Project}*.swift`。尺寸**只**写在各文件 `private enum *Assets`，Page 只引用常量。

## 375 设计尺

无运行时缩放库。Figma **375×667** 标注 1:1 落到 `CGFloat` 字面量。

| 常量 | 值 | 用途 |
|------|-----|------|
| 全宽 | `375` | banner、通栏 |
| 水平 inset | `20` | 页边 |
| 内容/面板宽 | `335` | 按钮区、Sheet |
| 表单宽 | `305` | 输入框、主按钮 |
| 导航栏 | `44` | 自绘 header（不含状态栏） |
| Tab 栏 | `49` | 自定义底栏（不含 Home Indicator） |

长屏只垂直延伸（`maxHeight: .infinity`、`ScrollView`），**不要**按屏宽比例换算。

## `{Project}Scaffold`

```swift
{Project}Scaffold(                    // 默认 backgroundImage: "base_back"
    contentSafeArea: .all,            // Tab 根页传 []
    dismissKeyboardOnTap: true
) { /* 内容 */ }
```

- 底层品牌色 `{Project}Colors` + 可选全屏切图
- 把系统 inset 写入 `@Environment(\.{project}SafeAreaInsets)`
- `.ignoresSafeArea(.keyboard)`

Tab 根页手动 `padding(.top, insets.top)`；push 页用默认 `.all`。

## Main 壳

**一个** `NavigationStack(path: $router.path)`：

```
{Project}Scaffold(contentSafeArea: []) {
    VStack(spacing: 0) {
        tabContent          // switch MainLogic.currentIndex
        {Project}MainTabBar // 切图 nor/pre，高度 49
    }
}
.toolbar(.hidden, for: .navigationBar)
.navigationDestination(for: {Project}Route.self) { ... }
```

Tab 切图：`tab_home_nor` / `tab_home_pre`（news、chat、mine 同理）。  
子页加 `{Project}SwipeBack()` 启用侧滑返回。  
**禁止** `TabView`、`IndexedStack` 保活四页、子页再建栈。视频通话 Sheet 用 `fromBottom: true`，不进 `Route`。

## 自绘 Nav

1. `padding.top = safeArea.top`
2. `frame(height: 44)` 包返回 / 标题 / 右侧
3. 滚动区在下方，**不再**叠状态栏高度
4. 返回：`{Project}AuthBackButton` → `logic.actionExit()` / `Router.shared.pop()`

右侧槽可放 `{Project}DiamondBadge(count:)`（按数字撑开，最小宽度约 97）。点钻石走 Logic `actionStore`（先登录）。

## 气泡

Chat / Live 评论文案由内容撑开，**不要**固定宽高：

- 最大宽 Chat/Live **265**，Moment **335**
- `.frame(maxWidth:)` + 自适应高度
- 标题旁 `.frame(minHeight:)` 必须 `alignment: .top`（默认垂直居中会拉开间距）

## 切图

```swift
{Project}AssetImage(name: "home_chat_now", width: assets.btnW, height: assets.btnH)
```

Figma 已导出完整按钮时：**切图即按钮**，禁止再套圆底 `Color` 代替 `*_pre`。  
选中/未选中用两张切图。

## Sheet / HUD / Toast

| API | 用途 |
|-----|------|
| `{Project}SheetView.show(_:)` | 居中或底部面板，宽 335；`dismiss()` |
| `{Project}MoreView.show(actions:onAction:)` | 更多动作 |
| `{Project}ReportView` / `{Project}TipsView` / `{Project}EulaView` | 举报、确认、协议 |
| `{Project}Hud.show()` / `.dismiss()` | 全屏 loading |
| `{Project}Toast.error(_:)` / `.show(_:)` | 2s toast；过滤 cancellation |

Sheet 展示时 IQKeyboard `pushDisabled`；聊天 / 发帖页 `.frostIgnoresIQKeyboard()`。  
状态栏：`{Project}StatusBar.shared.applyMainTab(index)`（Main 壳 onAppear / onChange），不要各页自己改 `preferredStatusBarStyle`。

HUD 何时强制 → **`swiftui-hud.mdc`**。Tab 首屏用 `isShowSkeleton`，不用全屏 HUD。

## 列表

`{Project}PagingView`：JXPagingView + JXSegmentedView + MJRefresh + SkeletonView。  
Logic `actionReload()`；过期响应用 `reloadGeneration` 丢弃。  
空态：`{Project}NoDataPlaceholder`。

## 其它组件

| 组件 | 用途 |
|------|------|
| `{Project}WebImage` | 网络图（见 `swiftui-media-cache`） |
| `{Project}AuthField` / `{Project}AuthButton` | 登录表单 |
| `{Project}GiftsView` / `{Project}GiftsShowView` | 直播礼物 |
| `{Project}PhotoPicker` | TZImagePickerController |
| `{Project}PhotoBrowser` | JXPhotoBrowser |
| `{Project}SettingView` | 设置 + 清缓存 |
| `{Project}VideoPage` | 假视频通话 UI（铃声，非运营商通话） |

## 字体

- 展示：`Font.{project}Anton(size)`（Info.plist `UIAppFonts`）
- 正文：系统 `Avenir-Medium` / `Heavy` / `Black`（不打包）
