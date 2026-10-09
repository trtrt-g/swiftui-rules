---
name: swiftui-app-foundation
description: >-
  Bootstrap native SwiftUI social/live iOS apps: {Project} prefix, Modules
  Page/Logic split, CocoaPods workspace, FactoryKit singletons, Launch→Main
  shell. Use when scaffolding Application/Config/Core/Models/Modules, Podfile,
  Info.plist, or starting a SwiftUI stack project.
---

# SwiftUI App Foundation

**P0 Rules**：`swiftui-page-logic`、`swiftui-scaffold-navbar`、`swiftui-minimal-diff`（`~/.cursor/rules/swiftui/`）

`{Project}` = 类型/文件名前缀（PascalCase）；`{project}` = 扩展方法与磁盘键小写前缀。  
从 **SwiftUI 社交模板仓**复制后全局替换前缀，不要从其它已上线仓整仓拷贝。

SQL / Edge 与 Flutter 社交 kit 共用：`flutter-supabase-kit`。本 Skill 只管 iOS 工程骨架。

## 栈边界（相对 Flutter）

本栈是 **原生 iOS 社交 App**（SwiftUI 或 UIKit 宿主），不是 Flutter Stack A/B：

- **默认 A 面**：无 `Opi/`、无 H5、无 Adjust / Facebook。需要 B 面时追加 `{Project}/Opi/`，见 **`swiftui-opi-shell`**
- **禁止 ATT**（`other/no-att.mdc`）；B 面 Facebook / Adjust 按 `swiftui-opi-shell/third-event.md`
- **无** GetX、ScreenUtil、EasyRefresh、`{project}_*_page.dart`
- 列表同步靠 Manager 拉取 + `{Project}Notification`，**不要**默认接 Realtime
- 编译 `{Project}.xcworkspace`；不要往工程里塞 Flutter Runner 约定；**不要**从 Flutter `lib/opi/` 拷 Dart

从 Flutter 仓拷文件时只允许对 SQL / Edge 思路，禁止拷 `lib/` 业务代码。

## 目录

```
{Project}/
├── {Project}.xcodeproj
├── {Project}.xcworkspace          # 必须用此编译
├── Podfile / Podfile.lock
├── Supabase/                      # bootstrap_all.sql + functions/
└── {Project}/
    ├── Application/               # {Project}App、{Project}Root
    ├── Config/                    # {Project}Config、{Project}Global
    ├── Core/
    │   ├── Navigation/            # {Project}Router、{Project}Container
    │   ├── Network/               # Manager、IAP、Error
    │   ├── Persistence/           # Storage、JSON、Notification、VideoCache
    │   └── UI/                    # Scaffold、Sheet、Hud、Paging、WebImage…
    ├── Models/                    # {Project}User、Live、Moment…
    ├── Modules/                   # 按功能：Launch / LogIn / Main / Home…
    ├── Resources/                 # Assets.xcassets、Audio、Text、字体
    └── SupportFiles/              # Info.plist、{Project}.entitlements
```

**禁止**改 `Pods/`。

## 命名

| 目录 | 文件 | 类 |
|------|------|-----|
| Config | `{Project}Config.swift` | `{Project}Config`、`{Project}Colors`、`{Project}Fonts`、`{Project}ClassName` |
| Application | `{Project}App.swift` | `@main`、`{Project}AppDelegate` |
| Navigation | `{Project}Router.swift` | `{Project}Route`、`{Project}Router` |
| Modules/{Feature} | `{Project}{Feature}Page.swift` / `Logic.swift` | `{Project}{Feature}Page` / `Logic` |
| Core/UI | `{Project}Scaffold.swift` 等 | `{Project}*` |
| Models | `{Project}{Entity}.swift` | `{Project}User`、`{Project}Live`… |

扩展：`Color.{project}Hex`、`Font.{project}Anton`、`View.{project}DismissKeyboardOnTap`。  
Asset 名：snake_case（`tab_home_nor`）。DB 列：snake_case CodingKeys。

禁止短文件名（`Config.swift`、`Router.swift`）。

## 新建项目脚手架

只生成空白 Page + Logic（无业务）：

| 域 | 文件 |
|----|------|
| launch | `{Project}LaunchPage` / `Logic` |
| login | `Start` / `SignIn` / `SignUp` / `Register` / `Social` / `Protocol` |
| main | `{Project}MainPage` / `Logic` |
| home | `{Project}HomePage` / `Logic` |
| news | `{Project}NewsPage` / `Logic` |
| conversation | `{Project}ConversationPage` / `Logic` |
| mine | `{Project}MinePage` / `Logic` |

切图按设计后续加；**不**复制模板 `images` 业务素材。空白页 = `{Project}Scaffold` + 绑 Logic。

## 启动顺序

`{Project}App.init`：

1. `{Project}Global.load()`
2. `{Project}Storage.shared.load()`
3. `{Project}User.shared.load()`
4. 触发 `{Project}SupabaseManager.client`（读本地 session）
5. `{Project}IapCoordinator.shared.ensureStarted()`
6. `{Project}NotificationManager.shared.start()`（请求通知权限；同意后注册 APNs）

`{Project}RootView`：`isMain == false` → Launch。`bootstrap` **循环** `requestServerTime()`，失败 sleep 1s 再试，成功才 `showMain()`。已登录则 `fetchUserWith()`，失败则清 session / `setLoggedIn(false)`。已登录且 `needsSocialProfile` → `push(.social)`。

磁盘：Documents `Storage.dat`（agree / login / pushToken）、`User.dat`（资料单例）。不要改成 UserDefaults 散键。

日志：`{Project}PrintLog`（网络 `🚀`）；不要 `print`。

APNs：启动请求权限，同意才 `registerForRemoteNotifications`。前台 `willPresent` → banner/sound/badge；点击只打日志。模拟器无 token。

## 依赖

**SPM：** supabase-swift、FactoryKit  

**CocoaPods（iOS 16）：** Kingfisher、JXPhotoBrowser、TZImagePickerController、JXSegmentedView、JXPagingView、SkeletonView、IQKeyboardManagerSwift、MJRefresh

## 编译

```bash
pod install
xcodebuild -workspace {Project}.xcworkspace -scheme {Project} \
  -destination 'generic/platform=iOS Simulator' -skipPackageUpdates build
```

必须 **workspace**，不要用 `.xcodeproj`。

## Info.plist / entitlements

权限文案（英文、举使用场景）：相机、麦克风、相册读/写。  
`UIBackgroundModes`: `remote-notification`。  
entitlements：`aps-environment`、`com.apple.developer.applesignin`。

**禁止 ATT** → `other/no-att.mdc`。不要加 `NSUserTrackingUsageDescription`、`AppTrackingTransparency`。

定位：本栈社交 App **不**申请定位。

## `{Project}ClassName`

```swift
enum {Project}ClassName: String {
    case file = "files"
    case apikey = "apikeys"
    case profile = "profiles"
    case room = "rooms"
    case live = "lives"
    case store = "stores"
    case message = "messages"
    case moment = "moments"
    case assistant = "assistants"
    case conversation = "conversations"
    case statistics = "statistics"
}
```

`.file` 是 Storage bucket 名，不是 REST 表。

## FactoryKit 单例

`Core/Navigation/{Project}Container.swift` 注册 tab 级：`router`、`root`、`mainLogic`、`homeLogic`、`newsLogic`、`conversationLogic`、`mineLogic`。  
暴露 `{Project}Router.shared` 等。构造须在 MainActor（`{project}MainActor` helper）。

## 模块扩展（按需，非脚手架）

Live / Moment / Post / Store / Profile / Chat / ChatAi / Video（假视频通话 UI）— 见 `swiftui-page-logic`。
