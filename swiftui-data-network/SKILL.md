---
name: swiftui-data-network
description: >-
  SwiftUI Supabase 手册：UserManager、ObjectManager、SupabaseManager、
  CursorManager、IAP StoreKit1、Storage、AuthGuard、Error、Apple Sign In。
  P0 见 swiftui-data-layer、swiftui-supabase-schema。
---

# SwiftUI Data & Network

**P0 Rules**：`swiftui-data-layer`、`swiftui-supabase-schema`（`~/.cursor/rules/swiftui/`）  
SQL / RLS / Edge 步骤：`flutter-supabase-kit`（列约定以 Swift `CodingKeys` 为准）。

## Secrets

只写在 `Config/{Project}Config.swift`：

- `{Project}Config.supabaseURL`
- `{Project}Config.supabasePublishableKey`（publishable / anon）
- `{Project}Config.supabasePath` = `/storage/v1/object/public`

Logic **不**硬编码 URL / key。不要把 service_role 放进 App。

## 栈

| 类 | 文件 | 职责 |
|----|------|------|
| `{Project}SupabaseManager` | `Core/Network/` | `SupabaseClient` 单例、`get_server_time`、`delete-account` |
| `{Project}UserManager` | 同上 | Auth（email、Apple）、profile |
| `{Project}ObjectManager` | 同上 | 表 CRUD、filter、上传、statistics |
| `{Project}CursorManager` | 同上 | Edge `cursor-generate` |
| `{Project}IapCoordinator` | 同上 | StoreKit 1 消耗型 |
| `{Project}User` | `Models/` | 单例 + `rxUser`；磁盘 `User.dat` |
| `{Project}Storage` | `Core/Persistence/` | `isFirst` / `isAgree` / `isLogIn` / `pushToken` |
| `{Project}AuthGuard` | `Core/UI/` | `ensureLoggedIn()` → `.start`；`ensureAgreed()` → EULA |
| `{Project}Error` | `Core/Network/` | `message(for:)`、`isCancellation` |
| `{Project}JSON` | `Core/Persistence/` | 日期；`{project}String` / `{project}Int` / `{project}Date` |

无独立 Dio/`NetworkManager`。业务表只走 ObjectManager。

## Client

```swift
static let client = SupabaseClient(
    supabaseURL: {Project}Config.supabaseURL,
    supabaseKey: {Project}Config.supabasePublishableKey,
    options: SupabaseClientOptions(
        auth: .init(emitLocalSessionAsInitialSession: true),
        global: .init(logger: {Project}SupabaseLogger())
    )
)
```

## `{Project}UserManager`

```swift
static func logInWith(email:password:) async throws
static func signUpWith(email:password:) async throws
static func appleWith(idToken:accessToken:) async throws
static func fetchUserWith() async throws
static func updateUserWith(params: [String: AnyJSON]) async throws
static func deleteUserWith() async throws          // Edge delete-account
static func syncAppleProfileIfNeeded(credentialEmail:givenName:familyName:) async throws
static func findUsersWith(filters:orders:) async throws -> [{Project}User]
```

登录后 `syncUserSingleton` → `{Project}User.setup` + `{Project}Storage.setLoggedIn(true)`。  
`needsSocialProfile`：`avatarUrl` 空 → 走 Social 页。

## `{Project}ObjectManager`

```swift
static func addObjectWith<T>(_ table: {Project}ClassName, model: T, select: String?) async throws -> T?
static func updateObjectWith<T>(…) async throws -> T?
static func deleteObjectWith(_ table: {Project}ClassName, objectId: String) async throws
static func findObjectsWith<T>(_ table: {Project}ClassName, filters:orders:select:limit:) async throws -> [T]
static func uploadImages(_ images: [UIImage]) async throws -> [String]
static func uploadData(_ data: Data, uploadType: {Project}UploadType) async throws -> String
static func recordStatisticsWith(_ statisticsType: {Project}StatisticsType) async
// device_type = {Project}Global.deviceName；device_num = identifierForVendor
```

Filter：`{Project}SupabaseFilter`（`.eq` `.inList` `.or` `.cs` …）、`{Project}SupabaseOrder`。  
列名一律 snake_case：`"objectid"`、`"send_userid"`。

jsonb 数组：`.cs("focus_list", id)` 表示 contains。关注/拉黑/点赞 **整列覆写**：

```swift
try await {Project}UserManager.updateUserWith(params: [
    "focus_list": .array(list.map { .string($0) }),
])
try await {Project}UserManager.fetchUserWith()
{Project}Notification.notifyFocus()
```

禁止为此建关联表。Fans 反查：别人的 `focus_list` contains 我。

Join select 示例：

```swift
"*, send_user:profiles!lives_send_userid_fkey(*)"
```

上传路径：`{userId}/{uuid}.jpg|mp4` → bucket `files`。  
Public URL：`{supabaseURL}/storage/v1/object/public/files/{path}`。  
新建 `objectid`：`UUID().uuidString.lowercased()`。

私聊会话：先 `.or` 查双向 userid；没有再 insert `conversation_type = User`。AI 会话 `Assistant` + `assistantid`。

## Model 解码

`KeyedDecodingContainer`：`{project}String` / `{project}Int`（Int 或 Double）/ `{project}Strings`（数组或 JSON 字符串）/ `{project}Date` / `{project}Raw`。  
Encoding：`{project}EncodeObjectId`（空则省略）、`{project}EncodeDate`（epoch 省略）。**不要**把 join 出来的 `send_user` 再 encode 回去。  
缺省用 `""` / `0` / `[]`，避免 decode 失败整页空白。  
枚举 rawValue **PascalCase**（`Male`、`Live`、`Text`、`iOS`、`Image`/`Video`）。  
`{Project}MomentStyle.photo` 的 rawValue 是 **`Image`**，不是 `Photo`。  
Cursor `messages[].role` 用**小写** `user`/`assistant`/`system`。

## 登录漏斗

1. 任何登录/注册前：`ensureAgreed()`（EULA sheet）。未同意直接 return。
2. **邮箱**：Start → SignUp（email 小写、密码 ≥6、name）→ Register（**必须头像**）→ `signUpWith` + 上传头像 + `updateUserWith`（`name_str`/`about_str`/`avatar_url`/`device_type`）→ `recordStatisticsWith(.signUp)` → `popToRoot`。两步草稿用同一个 SignUp Logic 单例。
3. **Apple**：Start 上点 Apple → `ensureAgreed` → Sign in with Apple。用户取消**不 Toast**。`authorizationCode` 当 accessToken。成功 `recordStatisticsWith(.social)` → `completeLogin()`；`needsSocialProfile` 则 Social 页只补 **头像 + about**（不改 name）。
4. **邮箱 SignIn**：成功 `recordStatisticsWith(.signIn)`。

## Apple Sign In

1. `{Project}AppleSignIn.shared.signIn()` → `idToken` + `authorizationCode`（作 accessToken）
2. `{Project}UserManager.appleWith(idToken:accessToken:)`
3. `syncAppleProfileIfNeeded` 补 email / name

Supabase Auth → Apple：Client IDs = Bundle ID。JWT / `.p8` **不进仓库、不写进审核回复**。

## IAP（StoreKit 1）

`{Project}IapCoordinator.shared`：

```swift
func ensureStarted()                          // App 启动；补单
func buyConsumable(productId: String) async -> Bool
```

流程：

1. `{Project}StoreLogic`：`findObjectsWith(.store, filters: [.eq("store_type", "iOS")])`
2. `buyConsumable(productId: store.productId)`
3. 成交：`fetchUserWith()` → `diamond_num = 当前余额 + stores.diamond` → `updateUserWith`
4. **DB 成功后**才 `finishTransaction`，并把 `transactionIdentifier` 写入 `{project}_iap_delivered_purchase_ids`
5. 加钻失败：transaction **不** finish，写入 `{project}_iap_recovery_queue`，启动时 `processMissedOrders`

消耗型金币；不能提现。余额不足（礼物/AI）：Toast `"Not enough diamonds"` + `push(.store)`，**不要发消息**。  
**禁止**在客户端写死 productId 列表。

登出：本地 `User.signOut` + `auth.signOut`，**不**调删号。  
删号：`deleteUserWith()`（Edge `delete-account`）再 `signOut`。两者都 `popToRoot` 且 Main tab 回 Home。

## AuthGuard / Error

```swift
await {Project}AuthGuard.ensureLoggedIn()   // false 时已 push .start
await {Project}AuthGuard.ensureAgreed()     // EULA sheet
{Project}Toast.error({Project}Error.message(for: error))
```

`{Project}Error`：`notLoggedIn`、`loginFailed`、`appleLoginFailed`、`uploadFailed`、`message(String)`。

## Cursor AI

`{Project}CursorManager.generateReply(messages:…)` → Edge `cursor-generate`。  
body：`messages: [{ role, content }]`，role **小写**。token 只在 `apikeys`。  
`systemPrompt` 由 `assistants` 的 name/about/tags/`first_str` 拼，回复语言由产品定。每条扣 `price_num`（先扣钻，失败则进商店）。
