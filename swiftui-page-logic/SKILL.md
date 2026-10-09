---
name: swiftui-page-logic
description: >-
  SwiftUI Page/Logic 手册：FactoryKit 单例、实体 registry、scheduleApply*、
  列表刷新模板、Logic 原型表（logic-archetypes.md）。P0 见 swiftui-page-logic、swiftui-hud。
---

# SwiftUI Page + Logic

**P0 Rules**：`swiftui-page-logic`、`swiftui-hud`（`~/.cursor/rules/swiftui/`）

`{Project}` = 类型前缀。一页一 Logic；弹层只调 `{Project}SheetView.show`。

## 新建页面

```
Modules/{Feature}/
  {Project}{Feature}Page.swift
  {Project}{Feature}Logic.swift
```

Page：`@ObservedObject`（tab 单例 / registry）或 `@StateObject`（一次性页）。  
`.task { await logic.actionAppear() }`；按钮 `logic.action*()`。

## Logic 两类

**A. Tab 级单例（FactoryKit）**

`{Project}HomeLogic.shared` 等，注册见 `swiftui-app-foundation` § FactoryKit。  
Page：`@ObservedObject private var logic = {Project}HomeLogic.shared`

**B. 实体级 registry**

Live / Moment / Message / Profile / Chat / ChatAi / Video：

```swift
static func put(_ model: {Project}Live) -> {Project}LiveLogic
static func find(tag: String) -> {Project}LiveLogic?
static func retainOnly(_ items: [{Project}Live])   // 列表刷新 prune
```

卡片：`{Project}LiveCard(logic: {Project}LiveLogic.put(live))`。  
详情用同一 tag，避免第二份状态。

**卡片 vs 详情（同一 Logic）**

- 卡片 `onAppear` 只 `prepareCardSurface()`（刷新关注态等），**禁止**拉评论/弹幕/会话消息
- 详情 Page `.task { logic.prepareDetailSurface() }` 才加载详情；用 `didPrepare` 避免每次 appear 全屏 HUD
- 列表刷新后 `{Project}LiveLogic.retainOnly` / `{Project}MessageLogic.sync(messages, previous:)` 修剪 registry

会话列表预览：ConversationLogic 另查 messages 填 `previews[conversationId]`，卡片读 `listPreview`，不要在 `conversations` select 里嵌套最后一条消息。  
拉黑过滤会话时 **保留** `conversation_type == Assistant`。

一次性页（Launch / Post / Store / Edit）：`@StateObject private var logic = {Project}LaunchLogic()`。

## `scheduleApply*`

视图更新中禁止直接写 `@Published`。合并同 runloop 推送：

```swift
private func scheduleApplyMessage(_ message: {Project}Message) {
    let enqueue = pendingMessage == nil
    pendingMessage = message
    guard enqueue else { return }
    DispatchQueue.main.async { [weak self] in
        MainActor.assumeIsolated {
            guard let self else { return }
            let next = self.pendingMessage
            self.pendingMessage = nil
            if let next { self.applyMessage(next) }
        }
    }
}
```

Live 用 `scheduleApplyLive` 同理。`apply*` 里才赋值 `@Published`。

## 列表刷新

```swift
reloadGeneration += 1
let generation = reloadGeneration
// await ObjectManager.find…
guard generation == reloadGeneration else { return }
items = next
```

HUD：写操作包 `{Project}Hud.show/dismiss`。Tab Feed 用 `isShowSkeleton`，见 P0 `swiftui-hud.mdc`。  
`actionAppear`：`guard !hasLoaded` 只拉一次。

## 路由

```swift
enum {Project}Route: Hashable {
    case start, signIn, signUp, register, social, agreement, privacy
    case live(String), moment(String), post, edit, store
    case focus, fans, block, profile(String), chat(String), chatAi(String)
}

{Project}Router.shared.push(.live(id))
{Project}Router.shared.pop()
{Project}Router.shared.popToRoot()
{Project}Router.shared.replaceAll([.social])
```

登录完成：`{Project}MainLogic.shared.completeLogin()` — 需补资料则 `replaceAll([.social])`，否则 `popToRoot()`。  
Chat / Mine tab：未登录 `push(.start)`。

**禁止**子页再建 `NavigationStack`。

## `{Project}Notification`

Combine `PassthroughSubject<Void, Never>`（不是 UNNotification）：

| Subject | 触发 |
|---------|------|
| `focus` | 关注/取关 |
| `block` | 拉黑 |
| `likes` | 点赞 |
| `moments` | 发帖/删帖/资料 |
| `conversations` | 会话变更 |
| `messages` | 评论/弹幕/私信 |

Logic：`{Project}Notification.observe(_:store:_:)` 订阅；写成功后 `{Project}Notification.notifyFocus()` 等（**不要** `focus.send()`）。

原型表 → [logic-archetypes.md](logic-archetypes.md)

## 打开实体（先 put 再 push）

```swift
{Project}ProfileLogic.open(user)
await {Project}ChatLogic.open(user, sayHi: true)   // 进页后发 "Hi~"
{Project}ChatLogic.open(conversation)
await {Project}VideoLogic.show(peer)               // 不进 Route
```

Home **ChatNow**：拉 `assistants` 第一条 → `{Project}ChatAiLogic.open`（不是私聊）。  
私聊 `open(user)` 内部 `loadOrCreateConversation`：`.or` 双向 `send_userid`/`receive_userid`，没有则 `addObjectWith`（`objectid` 小写 UUID）。

禁止只 `push(.chat(userId))`。

## 写操作模板

```swift
func actionFollow() {
    Task {
        guard await {Project}AuthGuard.ensureLoggedIn() else { return }
        {Project}Hud.show()
        defer { {Project}Hud.dismiss() }
        do {
            try await {Project}UserManager.updateUserWith(params: …)
            {Project}Notification.notifyFocus()
        } catch {
            {Project}Toast.error({Project}Error.message(for: error))
        }
    }
}
```

钻石不足：Toast + `push(.store)`。送礼/AI 扣 `diamond_num` 走 `updateUserWith`。

Tab 钻石：Logic 订 `rxUser` 只改 `diamondNum`；Page 用 `{Project}DiamondBadge`。`actionStore` 先 `AuthGuard`。

礼物：`{Project}GiftCatalog` 本地写死价格，**不**进 `stores` 表。

拉黑后 Feed：订 `notifyBlock` → `applyFilter()`（剔除 `blockList`），不要为此全量 `actionReload`。

## More 动作矩阵

`{Project}MoreView.show(actions:)` 按场景传数组，不要一套全开：

| 场景 | actions |
|------|---------|
| Chat / ChatAI 点气泡 | `[.deleteMessage]` |
| 自己的 Moment | `[.deletePost]` |
| 他人资料 / 直播卡 / 他人 Moment | `[.report, .block]` |

删会话：会话列表 `deleteObjectWith(.conversation)` + `deleteIfRegistered`，再 `notifyConversations`。

## 发帖

图最多 **3** 张（每次 `{Project}PhotoPicker.pick()` 只选 1 张再 append）**或** 一条视频，互斥。  
`moment_style`：`Image` / `Video`；`moment_type` 默认 `Other`。图写入 `photo_list`，视频只 `path_url`。

## Video 页

`{Project}VideoPage` 是应用内等待 UI + 循环铃声，**不是**运营商通话 / FaceTime。不要申请额外通话权限。

APNs：`{Project}NotificationManager`（前台 banner/sound/badge；点击只打日志，不跳转）。token 存 `{Project}Storage.pushToken`。
