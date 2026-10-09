---
name: swiftui-media-cache
description: >-
  SwiftUI 网络图片与视频磁盘缓存：{Project}WebImage（Kingfisher）、视频 URL 首帧、
  {Project}LoopPlayer、{Project}VideoCache（Range 续传 / 256KB 可播）。Use when
  implementing network images, video playback, prefetch, or cache clear.
---

# SwiftUI Media Cache

组件：`Core/UI/{Project}WebImage.swift`、`Core/Persistence/{Project}VideoCache.swift`。  
上传走 `{Project}ObjectManager`（见 `swiftui-data-network`）。

## 网络图 `{Project}WebImage`

- Kingfisher；memory ~300s / disk 7d
- 普通 URL：KF 加载
- **视频 URL**（`.mp4` 等）：`AVAssetImageDataProvider` 抽首帧当封面，不要把 mp4 当图片解
- 占位：切图或 `{Project}Colors` 底，不要系统灰

## 循环播放 `{Project}LoopPlayer`

`UIViewRepresentable` + `AVPlayer`：静音、循环。  
先 `{Project}VideoCache.playableFile(for:)`，命中则播本地；未命中播远程并 `prefetch`。

直播挂载：列表可见时播放，划走暂停；不要每个 cell 同时抢播放器。

## `{Project}VideoCache`

| 项 | 约定 |
|----|------|
| 目录 | Caches/`{project}_video` |
| 文件名 | URL 的 SHA256 |
| 可播阈值 | 256KB（`minPlayableBytes`） |
| 续传 | `.part` + `Range: bytes=existing-`；206 append；416 当完整 |
| 上限 | 80 文件 / 30 天 prune |
| 跳过 | 非 http(s)、路径含 `.m3u8` |

```swift
{Project}VideoCache.playableFile(for: url)   // 完整命中 → 本地 URL
{Project}VideoCache.prefetch(url)            // 后台下
{Project}VideoCache.clear()                  // 设置页清缓存
```

设置：`{Project}SettingView` 同时清 Kingfisher + `{Project}VideoCache.clear()`。

## 上传

- 图：`jpegData(compressionQuality: 0.8)` → `{userId}/{uuid}.jpg`
- 视频/音频：`uploadData(_:uploadType:)` → mp4 / mp3
- 多图 `withThrowingTaskGroup`；失败则 `storage.remove` 已上传 path
- upsert：`FileOptions(contentType:upsert: true)`（Storage 需 INSERT+SELECT+UPDATE）

Public URL 拼 `{Project}Config.supabaseURL` + `supabasePath` + `/files/` + path。

## 禁止

- 业务页自己 `URLSession` 下视频（必须走 `{Project}VideoCache`）
- 把视频 URL 丢给 Kingfisher 当静图（须走视频首帧分支）
- 清缓存只清内存不删 `{project}_video` 目录
