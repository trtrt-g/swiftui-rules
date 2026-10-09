# Logic 原型表

脚手架只生成 launch / login / main / home / news / conversation / mine 空白页。下表供按需实现时对照。

`tag` = registry 键，一般是 `objectid`（trim，空则不入表）。

## 总览

| Logic | 类型 | Tag | 状态 | 通知 | 典型方法 |
|-------|------|-----|------|------|----------|
| Launch | 一次性 | 无 | — | — | `bootstrap` 循环校时 → `showMain` |
| Main | Tab 单例 | 无 | `currentIndex` | — | `onTabTap`（Chat/Mine 需登录）；`completeLogin` |
| Start / SignIn / SignUp / Register | 表单 | 无 | 两步草稿单例 | statistics | `ensureAgreed`；邮箱 Register 必须头像；Apple 取消静默 |
| Social | 补资料 | 无 | 头像、简介 | — | 只写 `avatar_url` + `about_str`；已有头像则 pop |
| Protocol | 静态 | 无 | 文案 | — | 读 `Resources/Text/*.txt` |
| Home | Feed 单例 | 无 | category、lives、skeleton、diamondNum | `block` → `applyFilter` | `actionReload`、`actionChatNow`、`actionStore` |
| News | Feed 单例 | 无 | category、moments | `block` filter；`moments` 静默 reload | `actionEditPost` → `.post` |
| Conversation | Tab 单例 | 无 | 会话列表、previews | `block`、`conversations`、`messages` | 用户会话 + AI 入口 |
| Message | 实体短 | `message.objectid` | `message` | — | `put` / `sync` / `excludingBlocked` |
| Chat | 实体 | `conversation.objectid` | user、messages、input | `block` 剔消息 | `open(user, sayHi:)`、`loadOrCreate`、`actionSend` |
| ChatAi | 实体 | conversation 或 assistant | assistant、messages | — | 扣 `price_num`；无视频拨打 |
| Live | 实体 | `live.objectid` | live、messages、gifts | `block`、`messages` | 卡片 `prepareCardSurface`；页 `prepareDetailSurface` |
| Moment | 实体 | `moment.objectid` | moment、评论 | `block`、`messages` | 同上；赞 `like_list` |
| Post | 一次性 | 无 | 媒体草稿 | 发 `notifyMoments` | 图≤3 XOR 视频；`moment_style` Image/Video |
| Profile | 实体 | `user.objectid` | profile、moments | `block`、`focus`、`moments` | `open(user)`；Chat / Video / More |
| Mine | Tab 单例 | 无 | 自己的 moments | `moments` | Edit / Settings / Store |
| Edit | 一次性 | 无 | 草稿 | 发 `moments` | `actionSave` |
| Focus / Fans / Block | 列表 | 无 | users | `focus` / `block` | `.cs` 反查；Block 可移除 |
| Store | 一次性 | 无 | `stores` 行、diamondNum | `rxUser` | `buyConsumable` |
| Video | 通话 UI | `user.objectid` | 铃声 | — | `show(peer)`；挂断停音频 |

## 导航入口

| 入口 | 调用 |
|------|------|
| 顶栏钻石 / 礼物页钻石 | `actionStore` → `AuthGuard` → `push(.store)` |
| Home ChatNow | 拉 `assistants` 第一条 → `{Project}ChatAiLogic.open` |
| Live / Moment SayHi | `{Project}ChatLogic.open(user, sayHi: true)` → 进页发 `"Hi~"` |
| 卡片头像 | `{Project}ProfileLogic.open(user)` |
| 会话行 | `{Project}ChatLogic.open(conversation)` |
| 视频 | `{Project}VideoLogic.show(peer)`，**不**进 `Route` |

## 并发

- 列表：`reloadGeneration`
- 发送：`isSending` 防重
- `actionAppear`：`hasLoaded` 只跑一次
- `rxUser` 只更新 `diamondNum` / 关注态，不要整页重建
