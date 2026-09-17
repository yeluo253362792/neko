# Neko 项目功能与实现原理分析

> **分析对象**：`m1k1o/neko` v3
> **代码版本**：`ddf2787c`（分支 `seaslug`，与 `master` 无差异）
> **分析日期**：2026-09-17
> **代码规模**：Go 服务端 ~23k 行 / 141 文件，前端 ~11k 行 / 181 文件

---

## 目录

- [一、项目定位](#一项目定位)
- [二、整体架构与数据流](#二整体架构与数据流)
- [三、服务端实现原理](#三服务端实现原理)
  - [3.1 启动与依赖注入](#31-启动与依赖注入)
  - [3.2 桌面控制层](#32-桌面控制层如何操作-x)
  - [3.3 采集与编码](#33-采集与编码gstreamer)
  - [3.4 WebRTC 传输](#34-webrtc传输pion)
  - [3.5 会话成员权限模型](#35-会话--成员--权限模型)
  - [3.6 WebSocket 层](#36-websocket-层)
  - [3.7 HTTP API 与 Legacy 兼容层](#37-http-api-与-legacy-兼容层)
  - [3.8 插件系统](#38-插件系统)
- [四、前端客户端](#四前端客户端vue-2)
- [五、部署与镜像构建](#五部署与镜像构建)
- [六、设计要点与风险提示](#六设计要点与风险提示)

---

## 一、项目定位

Neko 是一个**自托管的虚拟浏览器 / 远程桌面流媒体服务器**。核心思路是：在容器里运行一个 X11 桌面（通常是浏览器），用 GStreamer 抓屏编码，通过 WebRTC 分发给多个用户，并把用户的鼠标键盘输入反向注入回 X 服务器，实现多人实时协作。

| 能力 | 说明 |
|---|---|
| 多人共享一个浏览器 | 多个用户看到同一块屏幕，是真正的"共同操作"而非屏幕共享 |
| 实时控制权移交 | 同一时刻只有一个人能操作（host 模型），支持隐式获取、请求、管理员强制授予 |
| 音视频低延迟 | 使用 WebRTC 而非 WebSocket 传图，内置音频支持 |
| 聊天 / 表情 | 聊天栏、emoji、飘屏 emote 动画 |
| 文件传输 | 上传到容器、下载、删除，支持拖拽投放到桌面 |
| 剪贴板双向同步 | 浏览器 ↔ 容器 X11 剪贴板（文本 + HTML + 图片） |
| RTMP 直播 | 把房间内容推流到 Twitch / YouTube |
| 硬件加速 | VAAPI (Intel) / NVENC (NVIDIA) 编码 |
| 会话录制 | 通过 RTMP + nginx-rtmp 落盘 |
| 插件系统 | Go `plugin` 动态库扩展 |

**典型使用场景**：看片同步（watch party）、互动演示、协同调试、远程支持教学、内嵌到自有 Web 应用、一次性/持久化浏览器、堡垒机、自动化浏览器（配合 playwright/puppeteer）。

---

## 二、整体架构与数据流

```
┌──────────────────────── 容器内部 ────────────────────────┐
│  Xorg :99.0 (dummy 驱动)  ← runtime/xorg.conf            │
│      ↑ XTest / XWarpPointer / Xkb     ↓ XGetImage(XSHM)  │
│  ┌──────────────┐              ┌────────────────────┐    │
│  │ desktop 模块 │              │ capture (GStreamer)│    │
│  │              │              │ gst → 编码器 → appsink│  │
│  └──────────────┘              └─────────┬──────────┘    │
│         ↑                                ↓ Sample{}       │
│  ┌──────┴───────┐              ┌─────────┴──────────┐    │
│  │ websocket    │              │ webrtc (pion)      │    │
│  │ 信令+控制     │◄────────────►│ TrackLocalStatic   │    │
│  └──────┬───────┘              └─────────┬──────────┘    │
└─────────┼────────────────────────────────┼───────────────┘
          │ WS (信令/控制/聊天)             │ WebRTC (音视频)
          │                                │ + DataChannel (输入)
          ↓                                ↓
                    浏览器 (Vue 2 客户端)
```

### 三条通道各司其职

| 通道 | 方向 | 承载内容 |
|---|---|---|
| **WebSocket** | 双向 | 信令（SDP/ICE 交换）、控制权、聊天、剪贴板、屏幕配置 |
| **WebRTC 媒体流** | 服务端 → 浏览器 | 视频与音频（两个独立 track，不做 RTP 复用） |
| **WebRTC DataChannel** (`"data"`) | 浏览器 → 服务端 | 输入事件，7/11 字节定长二进制帧 |

---

## 三、服务端实现原理

### 3.1 启动与依赖注入

**入口链路**：

```
server/cmd/neko/main.go:13   → 打印 banner，调用 cmd.Execute()
server/cmd/root.go:34        → cobra root（v3 唯一子命令 serve）
server/cmd/serve.go:138-206  → Start()：手写的 DI 容器
```

`Start()` 严格按依赖顺序构造各模块：

```
session → member → desktop → capture → webrtc → websocket → api → plugins → http
```

`Shutdown()` (`serve.go:208`) 逆序拆除。所有模块通过 `server/pkg/types/*.go` 的接口解耦 —— 构造显式、运行时松耦合。

**配置系统**：基于 Viper，合并顺序为

```
默认值  →  配置文件  →  环境变量 (NEKO_*)  →  命令行参数
```

配置文件加载路径优先级：`-config=` 参数 / `NEKO_CONFIG` 环境变量 / 可执行文件同目录 `neko.yaml` / `/etc/neko/neko.yaml`。支持 YAML、JSON、TOML、HCL、envfile、Java properties 六种格式。

日志基于 zerolog，支持 JSON 输出、文件轮转、级别控制（`NEKO_DEBUG=true` 快捷开启调试）。

### 3.2 桌面控制层（如何操作 X）

cgo 绑定实现在 `server/pkg/xorg/xorg.c`，链接 `-lX11 -lXrandr -lXtst -lXfixes -lXi -lxcvt`。

#### 鼠标

- **移动**：使用 `XWarpPointer` 而非 `XTestFakeMotionEvent`（`xorg.c:62-66`）
- **按键**：`XTestFakeButtonEvent`（`xorg.c:107-114`）
- **滚轮**：合成 button 4/5/6/7 配对事件（`xorg.c:77-105`）

#### 键盘（关键设计：keysym 而非 keycode）

客户端发送的是 **X11 keysym**，服务端在注入时才翻译成 keycode：

- 翻译用 `XkbKeysymToKeycode`（TigerVNC 移植版，`xorg.c:158-194`）
- 未注册的 keysym 通过 `XkbAddKeyKeysym` 动态添加（`xorg.c:197-249`）
- **缓存了 "Virtual core XTEST keyboard" 这个 XInput1 设备**（`xorg.c:18-40`），通过 `XTestFakeDeviceKeyEvent` 派发 —— 这是为了让 GDK3/XI2 应用（如 Firefox）能真正收到按键；否则退回 core `XTestFakeKeyEvent` 会被忽略（`xorg.c:272-281`）

> 这样设计的好处：客户端不需要知道服务端的键盘布局，布局切换由 XKB 统一处理。

- **键盘布局 / 变体**：非原生实现，通过 `exec setxkbmap -layout/-variant` 和 `setxkbmap -query`（`desktop/xorg.go:110-140`）
- **修饰键状态**：`XkbLockModifiers` 设置，`XkbGetState` / `XkbBuildCoreState` 读取（`xorg.c:411-424`）
- **去抖**：`ButtonUp` / `KeyUp` 必须有配对的 Down，否则返回 debounced 错误；`ResetKeys` 兜底释放所有键（`pkg/xorg/xorg.go:104-173`），配合 1s tick 的 debouncer（10s 窗口）

#### `keysymdef.go`（2498 行）

**脚本生成**，非手写。`server/pkg/xorg/keysymdef.sh` 下载 X.Org 的 `keysymdef.h`，把 `#define XK_* 0x…` 正则转换成 Go 常量并加上 `package xorg`。即完整的 X11 keysym 常量表。

#### 光标与截图

| 功能 | 实现 |
|---|---|
| 光标位置 | `XQueryPointer`（`xorg.c:68-75`） |
| 光标图像 | `XFixesGetCursorImage`，解码为 RGBA（`pkg/xorg/xorg.go:273-309`） |
| 屏幕截图 | 根窗口 `XGetImage`，24-bit RGB（`xorg.c:431-459`） |

#### 剪贴板同步

完全基于 `xclip` 子进程（`desktop/clipboard.go`）：

- 读：`xclip -selection clipboard -out -target <mime>`
- 写：`xclip -in -target <mime>`
- 文本用 `UTF8_STRING`，HTML 用 `text/html`

> **关键点**：X 剪贴板要求持有者进程存活，所以 `ClipboardSetBinary` 会**保持 `xclip -in` 进程常驻**，切换时杀掉上一个（`clipboard.go:64-79`）。这正是很多简陋实现会踩的坑。

变更检测通过 XFixes 的 `XFixesSetSelectionOwnerNotifyMask` 监听 `CLIPBOARD` atom（`pkg/xevent/xevent.c:33-36`）。

#### 触摸

XInput2 + 自定义驱动 `xf86-input-neko`，通过 unix socket `/tmp/xf86-input-neko.sock` 传递固定 12 字节结构体消息，操作码 `XI_TouchBegin/Update/End = 18/19/20`（`pkg/xinput/types.go`）。

#### 拖拽投放

`pkg/drop/drop.c` 用 GTK3 开一个 100×100 的 `GTK_WINDOW_POPUP` 假窗口作为 drag source（URI + text target），Go 侧的 `desktop/drop.go:20-69` 编排 XTest 假输入（按下 → 移动到目标 → 释放），把文件"拖"进桌面。HTTP 入口在 `api/room/upload.go:18-91`。

#### 文件选择对话框

`desktop/filechooserdialog.go` 用 `xdotool` 做 UI 自动化：按窗口名 "Open File" 查找、输入 URI、清除自动补全、Alt+F4 关闭。窗口隐藏通过 `XFileChooserHide` 移除 `WM_TRANSIENT_FOR` 并设为 `_NET_WM_STATE_BELOW`（`xevent.c:176-192`）。

### 3.3 采集与编码（GStreamer）

**Go 不做 XSHM，而是拼 GStreamer pipeline 字符串**，用 cgo 包 `gst_parse_launch`，从 `appsink` 取出编码后的 sample。

#### 默认 pipeline

`capture/manager.go:52-55`：

```
ximagesrc display-name=:99.0 show-pointer=true use-damage=false
  ! capsfilter caps=video/x-raw,framerate=25/1                          ← fps
  ! videoscale method=0 ! capsfilter caps=video/x-raw,width=1920,height=1080  ← 分辨率
  ! video/x-raw ! videoconvert ! queue
  ! vp8enc name=encoder target-bitrate=... cpu-used=4 end-usage=cbr ...
  ! appsink name=appsink
```

- `use-damage=false` 表示**放弃 X damage 扩展，强制全帧轮询**
- 编码器参数由 `types.VideoConfig.GetPipeline`（`pkg/types/capture.go:165-248`）用 **gval 表达式**求值拼装，所以 `GstParams` 里可以写 `target-bitrate=3072*650` 这类算式
- 自定义 pipeline 可用 `{display}` 占位符替换

#### cgo 桥接层

`server/pkg/gst/gst.go` + `gst.c`：

- `CreatePipeline` → `gst_parse_launch`
- `AttachAppsink` 设 `emit-signals=TRUE, sync=FALSE`，钩 `new-sample` 回调（`gst.c:109-113`）
- 回调把 buffer 拷贝后交给 Go 的 `goHandlePipelineBuffer`，压入带缓冲 channel，产出 `types.Sample{Data, Length, Timestamp, Duration, DeltaUnit}`（`gst.go:213-236`），其中 `DeltaUnit` 标记是否为非关键帧

#### 内置编解码器

`pkg/types/codec/codecs.go`：

| 编码 | Payload Type | 默认编码器 |
|---|---|---|
| VP8 | 96 | `vp8enc cpu-used=4 end-usage=cbr` |
| VP9 | 98 | `vp9enc` |
| H264 | 102 | `x264enc tune=zerolatency speed-preset=veryfast` |
| H265 | 116 | — |
| AV1 | — | — |
| Opus | 111 | `opusenc inband-fec=true bitrate=128000` |
| G722 / PCMU / PCMA | 9 / 0 / 8 | — |

RTCP feedback 统一为 TransportCC、GoogREMB、CCM/fir、NACK/pli。

#### 关键帧 lobby（避免花屏）

新观众加入时不会立刻收到画面，而是被放进 **keyframe lobby**（`capture/streamsink.go:379-386`），等到下一个非 delta 帧才放行。切流时还会主动发 `force_key_unit` 事件注入关键帧（`gst.c:207-217`）。

#### 流选择器

`StreamSelectorManagerCtx`（`capture/streamselector.go`）支持配置多套 pipeline（不同码率/分辨率），`GetStream(selector)` 按 `exact` / `nearest` / `lower` / `higher` 选择，配合带宽估计做自适应码率切换。

#### 屏幕尺寸变更协调

`desktop.SetScreenSize` 发出 `before_screen_size_change` / `after_screen_size_change` 事件，capture manager 在此期间销毁并重建所有 video/broadcast/screencast pipeline（`capture/manager.go:196-235`）。

#### 其他采集用途

- **Screencast**：JPEG 静帧 pipeline，按需启动 + 5s 空闲自动停止
- **Broadcast**：RTMP 推流，`flvmux ! rtmpsink`，支持 `{hostname}/{display}/{device}/{url}` 占位符

### 3.4 WebRTC 传输（pion）

#### Peer Connection 构建

`webrtc/manager.go:174-269`：

- 编码器注册到新的 `webrtc.MediaEngine`
- `SetICETimeouts(4s/6s/2s)`
- **`SetAnsweringDTLSRole(DTLSRoleServer)`** —— 服务端固定为 DTLS server
- 共享 ICE mux：`ice.NewTCPMuxDefault` + `ice.NewMultiUDPMuxFromPort`
- UDP 端口范围默认 **59000-59100**（`config/webrtc.go:283-292`）
- `SetNAT1To1IPs` 用于 NAT 穿透配置
- 启用带宽估计时注册 `cc.NewInterceptor` + `gcc.NewSendSideBWE`（Google Congestion Control）与 TWCC 扩展头

#### 服务端是 offerer（非常规）

大多数 WebRTC 应用是客户端 offer，这里是**服务端主动 offer**：连接建立后调 `peer.CreateOffer(false)`，通过 `signal/provide` 把 SDP + ICE servers 发给浏览器（`websocket/handler/signal.go:13-75`）。浏览器回 `signal/answer`。后续 renegotiation 由 `OnNegotiationNeeded` 触发。

#### 自适应码率

`webrtc/peer.go:203-311`：

1. 每 2s（`ReadInterval`）轮询 `estimator.GetTargetBitrate()`
2. 喂给 `TrendDetector` —— 基于 **Kendall's Tau 秩相关**的滑动窗口趋势检测（`pkg/utils/trenddetector.go`，移植自 livekit）
3. 结合退避策略决定切换 stream：
   - 稳定 12s（`StableDuration`）才允许升级
   - 不稳定 6s（`UnstableDuration`）触发降级
   - 卡顿 24s（`StalledDuration`）强制降级
   - 阈值 `DiffThreshold = 0.15`

#### 反向媒体（浏览器 → 容器）

`StreamSrcManager` 处理 `OnTrack`，把收到的 RTP 解码后写回内核设备：

- **摄像头**：`appsrc ! rtpvp8depay ! decodebin ! videoconvert ! v4l2sink device=/dev/video0`
- **麦克风**：`appsrc ! rtpopusdepay ! decodebin ! pulsesink device=audio_input`

需 `CanShareMedia` 权限。每 3s 发一次 PLI 请求关键帧。

#### Metrics

`webrtc/metrics.go` 暴露 Prometheus 指标：连接状态、ICE 候选数、REMB/GCC 估计带宽、丢包/抖动/NACK、ICE/SCTP 字节数等。采集侧另有 `streamsink_bytes`、`streamsink_listeners`、`pipelines_active` 等。

### 3.5 会话 / 成员 / 权限模型

这是整个项目**最精巧的部分**。

#### Session（会话）

`session.SessionCtx` 持有一个持久身份，可挂 0 或 1 个 WebSocket peer + 0 或 1 个 WebRTC peer。**会话比连接活得久**（可持久化到文件），所以断线重连能保留身份。

- `wsDelayedDuration = 5s` 延迟断开（`session/session.go:15`），应对瞬时抖动
- `MercifulReconnect` 配置允许新登录踢掉旧连接
- `profileChanged`（`session.go:44-70`）响应权限被回收：掉线 host、踢掉 watcher/连接

#### Member（用户）—— 4 种 Provider

通过 `member.provider` 配置切换（`member/manager.go:18-39`）：

| Provider | 存储 | 密码处理 | 适用场景 |
|---|---|---|---|
| `noauth` | 无 | 无密码，**每次登录都是管理员** | 默认值，单用户 / 完全隔离内网 |
| `multiuser` | 配置里两个密码 | **明文比较**（默认 `admin` / `neko`） | 最常见的多人场景 |
| `file` | JSON 文件 | **SHA-256 → base64（无盐）** | 持久化用户管理 |
| `object` | 配置内存 | 明文（代码里有 `TODO: hash`） | 配置驱动 |

`noauth` 和 `multiuser` 的 session id 格式为 `<username>-<5位随机>`，每次登录都是新会话（用户是临时的）。

#### 权限位

`pkg/types/member.go:11-27` 的 `MemberProfile`：

```
IsAdmin, CanLogin, CanConnect, CanWatch, CanHost,
CanShareMedia, CanAccessClipboard,
SendsInactiveCursor, CanSeeInactiveCursors
+ Plugins PluginSettings（per-plugin 的键值对）
```

中间件封装在 `pkg/auth/auth.go`：`AdminsOnly`、`HostsOnly`、`HostsOrAdminsOnly`、`CanWatchOnly`、`CanHostOnly`、`CanAccessClipboardOnly`、泛型 `PluginsGenericOnly[V]`。

#### 房间级设置（管理员可实时修改）

`types.SessionSettings`：

| 设置 | 含义 |
|---|---|
| `PrivateMode` | 私有模式，用户收不到房间音视频 |
| `LockedLogins` | 锁定登录（管理员仍可登录） |
| `LockedControls` | 锁定控制（管理员仍可控制） |
| `ControlProtection` | 只有当房间里至少有一个管理员时用户才能获得控制 |
| `ImplicitHosting` | 用户点击屏幕即自动获得控制 |
| `InactiveCursors` | 是否显示非活跃用户光标 |
| `MercifulReconnect` | 是否允许新登录踢掉未正常关闭的旧连接 |
| `HeartbeatInterval` | 心跳间隔（秒） |

#### Host 模型（控制权仲裁）

manager 里维护一个 atomic 的 `hostId` —— **同一时刻只有一个人能操作**。

所有输入事件在 handler 层先做 `controlRequest` 确保自己是 host，再调用 `desktop.*`（`websocket/handler/control.go:34-70`）。这是把"多人协作"退化为"轮流控制"的仲裁点。

实现上，`hostId atomic.Value` 里存的是 **session ID 字符串**而非会话指针（`session/manager.go:77,251`），`GetHost()` 再做一次 `Get(hostId)` 查表 —— 这样避免持有会话对象引用，host 离开后不会留下悬挂指针。

### 3.6 WebSocket 层

- 事件名常量集中在 `pkg/types/event/events.go`
- 消息信封：`{event: string, payload: json.RawMessage}`
- 派发是一个大 `switch`（`websocket/handler/handler.go:36-207`）：先给核心 handler，再依次给每个插件 handler（`:350-357`）—— 插件由此接入消息流
- **心跳**：10s WS ping + 应用层 `client/heartbeat`
- 非正常关闭 → `delayedDisconnect = true`（走 5s 延迟断开）
- **非活跃光标 ticker**：750ms 轮询一次鼠标位置，批量广播 `session/cursors`（`manager.go:375-431`）
- 会话事件通过 `github.com/kataras/go-events` 发布，websocket manager / api / plugins 各自订阅，完全解耦

#### 事件分类

`system` / `client` / `signal` / `session` / `control` / `screen` / `clipboard` / `keyboard` / `broadcast` / `send` / `file-chooser-dialog`。

其中 `send/unicast` 和 `send/broadcast` 是插件使用的通用通道（如 emote 飘屏）。

### 3.7 HTTP API 与 Legacy 兼容层

#### 新 API

基于 chi router。OpenAPI 文档位于 `webpage/docs/api/`（由 `server/openapi.yaml` 自动生成）。

| 路径 | 说明 |
|---|---|
| `POST /api/login` | 公开 |
| `/api/logout`、`/api/whoami`、`/api/profile`、`/api/stats` | 需认证 |
| `/api/sessions` | 列出会话；管理员可查看/删除/断开指定会话 |
| `/api/members` + `/api/members_bulk` | 用户 CRUD 与批量更新/删除 |
| `/api/room/settings` | 房间设置（管理员） |
| `/api/room/broadcast` | RTMP 推流启停（管理员） |
| `/api/room/clipboard` | 剪贴板读写 + `image.png` |
| `/api/room/keyboard` | 键盘布局映射、修饰键状态 |
| `/api/room/control` | 控制权请求/释放/授予/强制接管 |
| `/api/room/screen` | 屏幕分辨率配置、`cast.jpg`、`shot.jpg` |
| `/api/room/upload` | `drop` / `dialog`（文件投放与对话框） |
| `/api/chat`、`/api/filetransfer`、`/api/openinapp` | 插件贡献的路由 |
| `/api/ws` | WebSocket 升级 |
| `/api/batch` | 批量请求（排除自身与 `/api/ws`） |
| `/health`、`/metrics`、`/debug/pprof` | 运维端点 |

#### Legacy 兼容层

`internal/http/legacy/` 是 v2 兼容垫片，暴露老端点 `/ws`、`/file`、`/screenshot.jpg`、`/stats`、`/health`。

其 `/ws` handler（`legacy/handler.go:68-220`）的工作方式很特别：

1. 升级客户端连接
2. 内部 `POST /api/login` 创建会话
3. `dial` 新后端的 `/api/ws?token=...`
4. 跑两个 goroutine 做**双向事件翻译**
   - `wstobackend.go`：旧事件 → 新事件/API 调用
   - `wstoclient.go`：新事件 → 旧消息结构

当检测到任何 v2 配置项时自动启用，且会额外起一个 localhost 代理监听器。

### 3.8 插件系统

> **注意**：不是 hashicorp/go-plugin，用的是 Go 标准库 `plugin` 包（**进程内 `.so` 加载，无 gRPC**）。`go.mod` 中没有相关依赖。

#### 接口

`pkg/types/plugins.go`：

```go
Plugin            // Name() / Config() / Start(PluginManagers) / Shutdown()
DependablePlugin  // + DependsOn() []string
ExposablePlugin   // + ExposeService() any
```

`PluginManagers` 向插件注入 `SessionManager`、`WebSocketManager`、`ApiManager` 以及 `LoadServiceFromPlugin`（拿其他插件的服务实例）。

#### 依赖管理

`plugins/dependency.go` 做**环形依赖检测**（`addPlugin`） + 深度优先启动（`startPlugin`）。

#### 插件能力

- 注册 HTTP 路由（`ApiManager.AddRouter`）
- 注册 WebSocket 消息 handler（`WebSocketManager.AddHandler`）
- 订阅会话事件（如 `sessions.OnConnected`）
- 发送消息（`session.Send`）
- 读取 per-session / 全局设置（`session.Profile().Plugins` / `sessions.Settings().Plugins`）

#### 内置插件

| 插件 | 功能 | 关键实现 |
|---|---|---|
| `chat` | 聊天消息 | `can_send`/`can_receive` 权限解析，广播给所有 CanReceive 会话 |
| `filetransfer` | 文件浏览/上传/下载/删除 | `fsnotify` 监听 + 周期刷新，文件名净化（`..` 正则 + `filepath.Clean` + `Base`） |
| `openinapp` | 在容器内打开链接 | host-only，校验 `http`/`https` scheme 后 `xdg-open` |

`server/plugins/` 目录默认为空（`/etc/neko/plugins`），供运维放置自定义插件。

---

## 四、前端客户端（Vue 2）

### 技术栈

| 项 | 选型 |
|---|---|
| 框架 | Vue 2.7.13 + TypeScript 4.8（Class Component / `vue-property-decorator`） |
| 构建 | Vue CLI 5 (Webpack)，`publicPath: './'` |
| 状态 | Vuex 3.5 + `typed-vuex`（类型安全的 `this.$accessor`） |
| UI | Font Awesome 6、v-tooltip、vue-notification、vue-context、sweetalert2、animejs |
| i18n | vue-i18n（15 种语言） |
| 通信 | 原生 WebSocket + 原生 RTCPeerConnection + axios |

### 分层结构

```
client/src/
├── neko/        协议层
│   ├── base.ts      抽象 BaseClient（WS + RTCPeerConnection + DataChannel）
│   ├── index.ts     NekoClient，把服务端事件桥接到 Vuex
│   ├── events.ts    EVENT 常量表
│   ├── messages.ts  WS 消息类型定义
│   └── data.ts      输入二进制 OPCODE
├── store/       Vuex 模块（client/video/remote/settings/user/chat/files/emoji/openinapp）
├── components/  UI 组件
├── plugins/     全局插件（$client / $http / $swal / $anime / $log / i18n）
├── utils/       guacamole-keyboard、localstorage、全屏与键盘锁
└── locale/      15 种语言文件
```

### 输入捕获

一块覆盖在 video 上的**透明 `<textarea class="overlay">`** 是唯一的捕获面（`video.vue:14-32`）。

**二进制协议**（`neko/data.ts`）：

| Opcode | 长度 | 内容 |
|---|---|---|
| `MOVE` 0x01 | 7 B | uint16 x, y |
| `SCROLL` 0x02 | 7 B | int16 x, y |
| `KEY_DOWN` 0x03 | 11 B | BigUint64 keysym |
| `KEY_UP` 0x04 | 11 B | BigUint64 keysym |

**坐标换算**：

```js
x = round((serverWidth / rect.width) * (clientX - rect.left))
```

**键盘**：使用 **Apache Guacamole 的 Keyboard 实现**（`utils/guacamole-keyboard.js`）生成 X11 keysym，含 macOS 修饰键重映射。IME 合成事件通过保留 textarea 值处理。

**触摸**：`onTouchHandler` 把 touch 事件合成为 `mousedown`/`mousemove`/`mouseup` 再派发。

**滚轮**：归一化 deltaMode 到像素（`WHEEL_LINE_HEIGHT = 19`），应用反转与钳制，100ms 节流。

**剪贴板**：入向用 `navigator.clipboard.writeText`；出向在窗口聚焦/鼠标进入时读取并比对后发送。Firefox 的读取因已知挂起问题被禁用。

### 视频渲染与播放

- `<video ref="video" playsinline>`，`srcObject` 绑定
- **叠加层**：emote 动画层、透明输入 textarea、播放/静音图标、保持宽高比的 `.player-aspect`
- **播放解锁**：`canplaythrough` → `playable = true`；`play()` 失败时**降级为静音重试**，再失败才暂停（`video.vue:430-459`）—— 应对浏览器自动播放策略
- SDP 注入 `stereo=1`（Chromium 立体声兼容 workaround，`base.ts:366-368`）
- 支持画中画、全屏 + `navigator.keyboard.lock()`

### 重连策略

**没有自动重连**。`EVENT.RECONNECTING` 只弹一个提示条，用户需手动重新登录。

`base.ts:281-310` 的状态机中明确注释：`disconnected` 不是终态（瞬时网络抖动可能自愈），只有 `failed` / `closed` 才走 `onDisconnected()`。

### UI 功能

登录（支持 `?pwd=` / `?usr=` 一键加入）、聊天（markdown 渲染 + emoji 选择器 + 成员右键菜单）、文件传输（拖拽上传、进度、断点取消、多选删除）、管理面板（锁定开关、踢人/封禁/静音、分辨率切换、RTMP 推流）、emote 飘屏、成员头像条、设置持久化（localStorage）、关于页、`?cast` / `?embed` 视频模式。

---

## 五、部署与镜像构建

### runtime/ 基础层

| 文件 | 作用 |
|---|---|
| `Dockerfile` | Debian trixie-slim + Xorg(dummy) + PulseAudio + GStreamer + Noto 字体；创建 `neko` 用户；`DISPLAY=:99.0` |
| `Dockerfile.intel` | 追加 VAAPI 驱动与 `gstreamer1.0-vaapi`，设 `NEKO_HWENC=VAAPI` |
| `Dockerfile.nvidia` | Ubuntu 24.04 + CUDA 12.5 + VirtualGL，有 GPU 时用 `vglrun` 包裹 |
| `supervisord.conf` | 三个进程：`x-server`、`pulseaudio`、`neko serve --server.static /var/www` |
| `xorg.conf` | headless dummy 驱动 + 巨大 modeline 表（到 3840×2160@25） |
| `default.pa` | PulseAudio 虚拟设备拓扑（`audio_output` / `audio_input` 两个 null sink） |
| `widevine-installer/` | arm64 上安装 Widevine CDM（Asahi Linux 脚本） |
| `fontconfig/` `fonts/` `icon-theme/` | 字体与图标主题定制点 |

### 镜像构建流程

`Dockerfile.tmpl` 只是模板，真正的多阶段 Dockerfile 由 **Go 预处理器生成**：

```
build (bash)
  └─> utils/docker/main.go
        └─> 把 `FROM ./server/ AS server` 这类相对引用内联展开
              └─> docker build
```

`apps/*` 下每个应用（firefox / chromium / google-chrome / microsoft-edge / brave / waterfox / vlc / xfce / kde / remmina / opera / vivaldi / tor-browser / ungoogled-chromium）都有自己的 `Dockerfile` + `supervisord.conf` 片段，多数还会装 openbox 并注入窗口配置。

### 参考部署

`docker-compose.yaml` 关键参数：

```yaml
shm_size: 2gb                    # Chromium 必需
ports:
  - "8080:8080"
  - "52000-52100:52000-52100/udp"  # WebRTC
environment:
  NEKO_WEBRTC_EPR: 52000-52100
  NEKO_DESKTOP_SCREEN: 1920x1080@30
  NEKO_WEBRTC_NAT1TO1: <公网IP>
```

---

## 六、设计要点与风险提示

### 设计上值得称道的地方

1. **输入用 keysym 而非 keycode** —— 客户端不需要知道服务端键盘布局，布局切换由 XKB 统一处理，客户端与服务端彻底解耦。
2. **keyframe lobby** —— 新观众等到关键帧才放行，避免花屏；切流时主动注入关键帧。这是流媒体里的经典技巧。
3. **`xclip` 常驻进程** —— 正确处理了 X 剪贴板所有权语义，是很多简陋实现会踩的坑。
4. **服务端作为 offerer** —— 简化客户端逻辑，也让"先建好 peer 再选流"变得自然。
5. **Host 单点仲裁** —— 用 atomic `hostId` 把"多人协作"变成"轮流控制"，从根上避免输入竞争。
6. **事件驱动解耦** —— 会话事件通过 event emitter 发布，各模块订阅而非直接调用。
7. **Session 独立于连接** —— 会话持久化，断线重连保留身份，配合 5s 延迟断开应对抖动。

### 安全上需要注意的地方

| 风险 | 位置 | 说明 |
|---|---|---|
| 明文密码 | `multiuser` / `object` provider | 直接字符串比较 |
| 弱哈希 | `file` provider（`provider.go:23-33`） | 无盐 SHA-256 → base64，非 bcrypt |
| Token 明文落盘 | `session/serialize.go` | 会话持久化文件含明文 token |
| **zip-slip** | `pkg/utils/zip.go` 的 `Unzip` | `filepath.Join(target, file.Name)` 无 `../` 校验（对比：filetransfer 插件做了净化） |
| 凭据入日志 | 文件传输 `?pwd=` 查询参数 | 会进入访问日志 |
| 默认全管理员 | `noauth` provider | 仅适合完全隔离环境 |
| 默认弱口令 | `multiuser` 默认 `admin` / `neko` | 必须修改 |

**生产部署建议**：至少使用 `file` provider 并配合反向代理做额外鉴权；确保 `server.proxy` 仅在确实位于反向代理之后时启用；`server.pprof` 在生产环境应关闭。

### 项目活跃度

从近期 commit 可见项目正在持续加固：

- `ba098db6` markdown 渲染禁用 Vue 模板编译并转义属性（修 XSS）
- `371f5c2c` 修复 RTCP PLI ticker 的 goroutine 泄漏
- `347418c3` 修复 CString / GFile 内存泄漏
- `ddf2787c` 更新 Tor Browser 下载源

---

## 附录：核心文件索引

| 模块 | 关键文件 |
|---|---|
| 启动 / DI | `server/cmd/serve.go:138-206` |
| 会话 | `server/internal/session/{session,manager,serialize,auth}.go` |
| 成员 / 权限 | `server/internal/member/*`、`server/pkg/auth/auth.go`、`server/pkg/types/member.go` |
| WebSocket | `server/internal/websocket/{manager,peer}.go`、`server/internal/websocket/handler/*` |
| WebRTC | `server/internal/webrtc/{manager,peer,track,metrics}.go` |
| 采集 | `server/internal/capture/*`、`server/pkg/gst/*` |
| 桌面控制 | `server/internal/desktop/*`、`server/pkg/{xorg,xinput,xevent,drop}/*` |
| HTTP API | `server/internal/api/*`、`server/internal/http/*`（含 `legacy/`） |
| 插件 | `server/internal/plugins/*`、`server/pkg/types/plugins.go` |
| 前端协议层 | `client/src/neko/{base,index,data,events,messages}.ts` |
| 前端输入 | `client/src/components/video.vue` |
| 镜像构建 | `build`、`utils/docker/main.go`、`runtime/*`、`apps/*` |
