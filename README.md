<div align="center">

# Speculum

把视频链接粘进来，下载下来。macOS 原生应用。

**安装包里自带 yt-dlp 与 ffmpeg，新的 Mac 上不需要装任何东西**

**本仓库只放安装包，源代码不在这里。**

</div>

---

## 它是什么

一个瘦客户端：界面、队列、文件管理归它，真正的解析和下载交给 yt-dlp，音视频合并交给 ffmpeg。你只需要粘贴一个链接，剩下的它来编排。

支持最佳 MP4、仅音频 M4A、仅音频 MP3、指定分辨率 1080P / 720P / 480P；可选下载字幕与封面；队列和历史存在本地，重启不丢。

## 下载

| 文件 | 大小 | 说明 |
| --- | --- | --- |
| [Speculum-2.2.dmg](../../releases) | 60 MB | 打开后把 Speculum 拖进「应用程序」 |

> **仅支持 Apple Silicon（M1 及以后），需要 macOS 14.0 及以上。** Intel Mac 用不了。

体积的九成不是应用本身——是里面带的 yt-dlp（36 MB）和 ffmpeg（48 MB）。

## 安装

1. 下载上面的 DMG，双击打开
2. 把 Speculum 拖进「应用程序」
3. 首次打开若提示「无法验证开发者」，到「系统设置 → 隐私与安全性」点「仍要打开」
4. 如果提示文件已损坏，终端执行：

```bash
xattr -dr com.apple.quarantine /Applications/Speculum.app
```

本包是 ad-hoc 签名，没有做 Apple 公证，所以会有第 3、4 步的提示。这不是文件坏了。

## 用了哪些开源项目

Speculum 自己只负责界面、队列和调用编排。真正解析页面、下载文件、合并音视频的，是两个成熟的开源命令行工具：

| 组件 | 在这里干什么 | 许可 |
| --- | --- | --- |
| [**yt-dlp**](https://github.com/yt-dlp/yt-dlp) | 解析页面、枚举可用格式、下载视频/音频/字幕 | Unlicense（公共领域） |
| [**ffmpeg**](https://ffmpeg.org/) | 音视频合并、转封装、音频转码（m4a / mp3） | LGPL / GPL（取决于构建时的编译选项） |

两个都是**未经修改的官方二进制**，随包放在 `Speculum.app/Contents/Resources/bin/`，由应用以参数数组的方式启动子进程调用——不经过 shell，所以不存在命令注入。版权归各自作者所有，源码与完整许可条款见上表链接。

除这两个以外没有别的第三方运行时依赖：界面是 SwiftUI + AppKit，队列和历史用 UserDefaults + JSON 存在本地。

## 它是怎么工作的

```text
粘贴链接（或 ⌘⇧V 直接读剪贴板）
   ↓  URL 校验：仅 http/https，按规范去重
yt-dlp -J        解析元数据与可用格式（只解析，不下载）
   ↓  格式映射：最佳 MP4 / 仅音频 M4A / 仅音频 MP3 / 1080P / 720P / 480P
下载队列         并发 1–6，默认 1
   ↓  yt-dlp 下载；需要合并时再调 ffmpeg
完成             可打开文件所在目录
```

几个值得说明的点：

- **格式怎么来的**：选「最佳 MP4」时让 yt-dlp 分别取视频流和音频流，再由 ffmpeg 合并成一个文件；选「仅音频」时只取音频流，M4A 直接转封装、MP3 走 libmp3lame 转码。
- **不重复解析**：解析阶段用 `-J` 拿到完整 info JSON，下载阶段用 `--load-info-json` 复用它，同一个链接不会解析两次。
- **并行分片**：`--concurrent-fragments 8`，DASH/HLS 的分片下载会并行拉取。
- **为什么要先把二进制「释放」出来**：应用包有代码签名，在包内就地升级二进制会破坏签名，可能导致应用再也打不开；而且 `yt-dlp -U` 要求对自身文件有写权限，从挂载的只读 DMG 里运行必然失败。所以首次启动会把它们复制到 `~/Library/Application Support/Speculum/bin` 再用。换新版时，只有副本仍是「上次释放的原样」才会被覆盖，你自己 `-U` 升级过的副本不会被冲掉。
- **找不到依赖时怎么办**：按这个顺序找——设置里指定的路径 → 释放出的副本 → 应用包内 → `~/.local/bin`、`/opt/homebrew/bin`、`/usr/local/bin` 等常见目录 → pip 用户目录 → PATH。GUI 应用的 PATH 由 launchd 提供，默认不含 Homebrew 路径，所以必须显式枚举。
- **数据不出本机**：队列与历史在 `~/Library/Application Support/Speculum/tasks.json`，日志在 `~/Library/Logs/Speculum/speculum.log`，下载目录默认 `~/Downloads/Speculum`。不上传任何 URL、文件或统计信息。

## 校验值

| 文件 | SHA-256 |
| --- | --- |
| `Speculum-2.2.dmg` | `cab240732b5e1cdf9e3f2c3e51de62129ae102fc81b5a9e748b7dbb43883a202` |

对比方式：`shasum -a 256 文件名`。

## 已知局限

- **依赖上游页面结构**。解析靠 yt-dlp，各站点改版后旧版 yt-dlp 会失效——应用内可以「检查更新」升级它（升级的是释放到 Application Support 的副本，不动应用包本身）
- **不是官方客户端**。与任何视频平台无关，只是把公开网页交给 yt-dlp 处理
- **百度网盘、需要登录的私享内容**这类不适用：不做登录绕过，只处理公开可访问的内容
- **未公证**，如前所述，首次打开需要在系统设置里放行

## 合规

不实现、不协助任何 DRM 绕过、付费墙绕过或登录绕过。所有解析与下载都在本地通过 yt-dlp 完成。请仅对你有权下载的公开、非 DRM 内容使用本工具。

## 问题反馈

在 [Issues](../../issues) 里提交，请说明 macOS 版本、Speculum 版本，以及具体在做什么操作时出的问题。
