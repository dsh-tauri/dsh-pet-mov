# dsh-pet-mov

[DeepSeek Harness Desktop](https://github.com/dsh-tauri-desk/deepseek-harness-desktop) 预设桌宠在 **macOS** 上的素材仓库。

## 为什么需要这个仓库

macOS 的 WKWebView（WebKit）在解码 **VP9-alpha WebM** 时会丢弃 alpha 平面，把透明区域渲染成不透明的黑色矩形，而且**不触发任何 error 事件**（排查极难）。
见 issue [#434](https://github.com/dsh-tauri-desk/deepseek-harness-desktop/issues/434)。

Safari / WKWebView 原生支持的透明视频格式是 **HEVC-with-Alpha**（`AVVideoCodecType.hevcWithAlpha`，`hvc1` tag），而 `hevcWithAlpha` 编码器**只有 macOS 有**。
因此本仓库用 GitHub Actions 的 macOS runner 把上游 [PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet) 的透明 VP9 WebM 批量转码为 `.mov`，
桌面端在 macOS 上只下载本仓库、只播放 `mov/` 里的资源；Windows / Linux 继续使用上游 WebM。

## 内容

| 路径 | 说明 |
| --- | --- |
| `config.jsonc` | 上游 `dsh-pet/assets/config.jsonc` 原样拷贝（动画池协议） |
| `mov/*.mov` | HEVC-with-Alpha 动画，文件名主名与上游 `assets/webm/*.webm` 一一对应 |
| `.github/workflows/build-mov.yml` | 自动转码流水线 |

除 `config.jsonc` 与 `mov/` 之外**不保留任何素材**（上游的 `fonts/`、`pic/`、`webm/`、`preview/` 都不落盘）。
桌面端在 macOS 上以本仓库**根目录**作为 assets 前缀下载，安装到 `~/.dsh/pets/<id>/`，形如：

```
~/.dsh/pets/maid-deepseek-whale/
├── config.jsonc
└── mov/
    ├── 待机呼吸休闲.mov
    └── ...
```

## 流水线

`build-mov.yml` 的三条触发路径：

- `workflow_dispatch`：手动重编，可指定上游 ref；
- `schedule`：每周一 03:17 UTC 跟随上游 `main` 重编；
- `repository_dispatch: dsh-pet-updated`：上游素材更新时外部触发。

编码方案（沿用上游 `scripts/encode_hevc_alpha.sh`，即 issue 贡献者在 Safari 实测通过的方案）：

```
上游 webm → ffmpeg -c:v libvpx-vp9 -pix_fmt bgra（保留 alpha 的解码）
          → swift hevc_alpha_encoder.swift（AVAssetWriter + hevcWithAlpha）
          → mov/*.mov
```

产物会被 `scripts/check_alpha.py` 抽检（首帧 alpha 必须有真实层次），随后由 workflow 机器人提交回本仓库的 `main`。

## 依赖与出处

- 素材来源：[PC2005-cloud/dsh-pet](https://github.com/PC2005-cloud/dsh-pet)（MIT）。
- 编码脚本：上游 `scripts/encode_hevc_alpha.sh`、`scripts/hevc_alpha_encoder.swift`、`scripts/check_alpha.py`，CI 直接调用被检出的上游副本，不在本仓库重复维护。
- 消费者：[dsh-tauri-desk/deepseek-harness-desktop](https://github.com/dsh-tauri-desk/deepseek-harness-desktop) 的 `src-tauri/resources/preset-pets.json`（`macos` 平台覆盖）。
