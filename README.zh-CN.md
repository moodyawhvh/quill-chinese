# quill

一个极简的、完全本地化的 macOS 会议录音 + 转录工具。只需点击菜单栏一下，即可录制麦克风和所有系统音频为两个独立音轨；停止录制后，quill 会在设备本地进行转录，并生成带说话人标记的转录文本。所有数据永远不会离开你的电脑。

以羽毛命名。与 [parrot](https://github.com/digimata/parrot) 是姊妹项目，共享相同架构：单个 Swift 二进制文件，菜单栏托盘运行，无需应用包。

> 📌 **原项目地址：** [digimata/quill](https://github.com/digimata/quill)
> 
> 💬 **代部署/定制服务请联系微信：uaycar**

## 安装

```sh
cd quill
swift build -c release
sudo cp .build/release/quill /usr/local/bin/quill
quill install --launch-at-login   # 可选 — 登录时后台自动启动
```

**系统要求：** macOS 15+（需要 Core Audio 进程音频捕获功能，无需虚拟设备或内核扩展）。推荐 Apple Silicon 以获得更快的转录速度。

## 使用方法

1. **运行**（在终端输入 `quill`，或通过 LaunchAgent 启动）。
2. **点击菜单栏中的羽毛图标 → 开始录音。** 首次使用时会提示授权麦克风和系统音频录制权限。录音过程中，图标变为红色并显示已录制时长，macOS 会显示紫色录音指示器。
3. **点击 → 停止录音**，会议结束后停止。转录自动开始（菜单显示进度）；转录完成时会发送通知。

每次录音会话保存在 `~/Recordings/<yyyy.MM.dd-HHmm>/` 目录下：

| 文件 | 内容 |
|---|---|
| `mic.caf` | 你这边的声音（默认输入设备，AAC 格式） |
| `system.caf` | Mac 播放的所有声音 — 通话对方的声音（AAC 格式） |
| `meta.json` | 开始/结束时间戳、时长、每个音轨的起始偏移量 |
| `transcript.json` | 标准转录文本 — 包含引擎来源、带时间戳和说话人标记的片段 |
| `transcript.md` | 同一转录文本的可读格式 |
| `transcribe.log` | 本次会话的转录进度/错误日志 |

使用双音轨是有意设计：语音模型在干净的单源音频上表现更好，而麦克风与系统音频的分离天然实现了双方分离——"我"和"对方"无需说话人识别模型。使用 CAF 格式也是有意设计：与 m4a 不同，它不需要最终化处理——如果进程在会议中途崩溃，已写入的数据仍然可读。

## 转录功能

内置、本地运行、自动执行。默认引擎是 **Parakeet TDT 0.6B v2**（英文），通过 [FluidAudio](https://github.com/FluidInference/FluidAudio) 的 Core ML 移植版本运行——在 Apple Silicon 上每小时音频大约只需 20 秒。模型（约 600 MB）在首次转录时下载一次；`quill doctor` 可以检查模型是否已缓存，避免在重要会议前才下载。

每个音轨单独转录，通过起始偏移量对齐到同一时间线，然后按时间戳合并。任务在串行队列中运行——你可以在上一个录音转录时开始新的录音。未完成的任务在下次启动时恢复（文件系统就是队列：有 `meta.json` 但没有 `transcript.json` 的会话就是待处理的）。失败信息会追加到会话的 `transcribe.log` 中，不会阻塞后续任务。

引擎通过一个小协议层封装；计划添加 Whisper 引擎（WhisperKit large-v3-turbo）作为备选/重新转录选项。

## 配置

可选配置文件位于 `~/.config/quill/config.json`：

```json
{
  "recordings_dir": "~/Recordings",
  "transcription": { "enabled": true, "engine": "parakeet" },
  "on_stop": "my-hook"
}
```

- `recordings_dir` — 录音保存位置。优先级：`--out` 参数 > 配置文件 > `~/Recordings`。
- `transcription.enabled` — 设为 `false` 则只录音不转录。
- `mic_voice_processing` — Apple 的麦克风回声消除（默认关闭）。当通过扬声器录制会议时设为 `true`，这样播放声音不会泄漏到麦克风音轨中被重复转录。代价是：语音单元运行时，macOS 会略微降低其他播放音量。使用耳机时没有回声需要消除，因此原始录音是更好的默认选择。
- `on_stop` — 转录完成后（或录音禁用时录音结束后）以会话目录为参数执行的 shell 命令。可以连接到任何后续流程：摘要、归档、索引。

## 命令行

```sh
quill                        # 运行菜单栏守护进程（^C 退出）
quill run --out <dir>        # 自定义录音根目录（默认 ~/Recordings）
quill doctor                 # 检查权限、录音文件夹、模型状态
quill install --launch-at-login
quill install --uninstall
```

## 技术栈

- **Swift** — 单个 SPM 可执行目标
- **Core Audio 进程音频捕获**（`AudioHardwareCreateProcessTap`，macOS 14.2+）— 通过私有聚合设备捕获系统音频
- **AVAudioEngine** — 麦克风录音
- **AVAudioFile** — 流式 AAC 编码写入 CAF
- **FluidAudio / Parakeet** — 本地 Core ML 转录
- **NSStatusItem** — 整个 UI 就是菜单栏图标

## 注意事项

- 全局音频捕获会录制 Mac 播放的*所有*声音——通知提示音、音乐等等。开会时不要播放 Spotify（或者如果你在意的话，可以请求按进程选择的功能）。
- 如果录音出来是静音的，请检查 系统设置 → 隐私与安全性 → 屏幕与系统音频录制。
- Parakeet v2 仅支持英文。其他语言将在 Whisper 引擎中支持。
- 二进制文件内嵌了 Info.plist（`__TEXT,__info_plist`），这样 TCC 可以在作为 LaunchAgent 运行时将权限归属于 quill 本身。
