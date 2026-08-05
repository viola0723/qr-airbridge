# QR-AirBridge

**Air-gap file transfer through the screen: a PC plays an animated QR stream, a phone camera reads it — no network, no cable, no app install. The whole app is one `index.html`.**

**物理隔空的文件传输：电脑屏幕轮播二维码，手机摄像头扫码重组——不经网络、不装 App、不插线。整个应用就是一个 `index.html`（无构建、无外部依赖、所有库内嵌，双击即用）。**

---

## English

### What it is

QR-AirBridge moves files across an air gap over a one-way visual channel:

- **Sender**: a PC browser opens `index.html`, picks a file, and plays an animated QR stream (fullscreen mode recommended).
- **Receiver**: a phone browser opens the same file (or the same HTTPS URL), starts the camera, and watches the screen. A fountain code (Robust Soliton, LT-style) rebuilds the file from whatever frames make it through — dropped frames don't matter, and the receiver can join mid-stream.

### Features

- **Truly offline**: runs from `file://`, zero network access, single ~1.9 MB file.
- **Fountain coding**: no ACK channel needed; robust to frame loss; mid-stream join supported (manifest rides fountain block 0).
- **Fast decode pipeline**: native BarcodeDetector (Shape Detection API) as tier 0 where available (Android Chrome / WebView 83+ / desktop Chromium), then zxing-wasm (ZXing-C++ compiled to WASM) in a Blob Web Worker, with graceful fallback tiers — main-thread WASM → zxing-js → jsQR — so it keeps working on iOS WebKit quirks.
- **Adaptive optics**: receiver-side resolution ladder (640/960/1280) with hysteresis + multi-scale decode to survive screen moiré and high-density codes.
- **Integrity**: per-frame CRC32, whole-file SHA-256, optional deflate.

### Measured throughput (real devices, 2026-08)

| Device | Settings | Result |
|---|---|---|
| Android phone | 450 B @ 16–20 FPS | ≈ 8 KB/s effective |
| Android phone | 900 B @ 24 FPS | ~10 decodes/s |
| iPhone | 1200 B @ 30 FPS | ~50 decodes/s — 36 KB/s supply near-saturated |
| Android phone | 1300 B @ 50 FPS | ~10 decodes/s — decode-limited, no gain from maxing out |
| iPhone | 1300 B @ 50 FPS | ~30 decodes/s — past the FPS knee, rate drops; sweet spot back at 24–30 FPS |

Your mileage will vary — the visual channel depends on the camera and the screen. Tune the FPS / frame-size sliders while watching the live stats line (扫描/解码 per second).

### Quick start

1. **Sender (PC)**: open `index.html` in Chrome/Edge — double-click works.
2. **Receiver (phone)**: open the same file in the phone browser. Best practice: serve it over **HTTPS** — `getUserMedia` needs a secure context, and on iOS HTTPS also unlocks the Worker decoder tier. (Opening inside WeChat's built-in browser can't use the camera; use "Open in browser".)
3. Sender: choose file → 开始发送 → 全屏发射. Receiver: 启动摄像头, aim at the on-screen alignment box.
4. Tune: raise FPS first, then frame size, until 解码/s stops keeping up with the frame supply.

### Wire protocol (v2, one QR alphanumeric frame)

```
TID(8) + SEED(6) + TOTAL(4) + CRC(6) + PAYLOAD(base45)
```

Fixed-length header, no separators. A seed-driven PRNG (LCG-16807) reproduces the fountain degree and participating block indexes on both sides, so no index table is transmitted. Block 0 carries a JSON manifest (name / size / SHA-256 / compression flag). Full spec and iteration log: `ANCHOR.md`; research notes and roadmap: `RESEARCH.md`.

---

## 中文

### 这是什么

单文件 HTML 应用（约 3450 行 / ~1.9MB，无构建工具、无外部依赖、所有库内嵌含 WASM base64）。
物理隔离（air-gap）文件传输：发送端屏幕轮播二维码，接收端手机摄像头扫码，
喷泉码（LT 风格，Robust Soliton）重组数据，**全程不经网络**。适合断网/涉密/临时救急场景。

### 特性

- **真离线**：`file://` 双击即用，零网络请求，单文件拷走就能跑。
- **喷泉码**：无反向 ACK 也不怕丢帧；接收端可中途加入（清单块搭喷泉块 0 每 32 帧重发）。
- **四档解码链**：BarcodeDetector 原生（安卓 Chrome / WebView 83+ / 桌面 Chromium，系统级检测）→ zxing-wasm（Worker，不卡 UI）→ 主线程 wasm → zxing-js/jsQR；
  不支持原生 API 的设备（iOS/Safari）静默落到 wasm 档，苹果 WebKit 拦 blob worker 的机型再落主线程档，速度仍在。
- **自适应光学**：接收端分辨率阶梯（640/960/1280，连败升档/连胜降档）+ 多尺度轮换抗屏幕晶格混叠。
- **完整性**：帧级 CRC32 + 文件级 SHA-256 + 可选 deflate。

### 真机实测吞吐（2026-08，用户设备）

| 设备 | 参数 | 结果 |
|---|---|---|
| 安卓手机 | 450B @ 16–20 FPS | ≈ 8 KB/s 有效吞吐 |
| 安卓手机 | 900B @ 24 FPS | 解码 ~10/s |
| iPhone | 1200B @ 30 FPS | 解码 ~50/s，36 KB/s 供给接近打满 |
| 安卓手机 | 1300B @ 50 FPS | 解码 ~10/s——算力受限，拉满无收益 |
| iPhone | 1300B @ 50 FPS | 解码 ~30/s——FPS 过膝点回落，甜点位回 24~30 FPS 区间 |

视觉通道看设备——相机和屏幕都影响命中率。盯着统计行（扫描/解码每秒）调滑块：
先抬 FPS，再抬单帧，解码跟不上了就回退一格。

### 快速上手

1. **发送端（PC）**：Chrome/Edge 打开 `index.html`（双击即可）。
2. **接收端（手机）**：同一文件用手机浏览器打开。推荐经 **HTTPS** 提供——
   摄像头要求安全上下文，且 iOS 在 HTTPS 下才能启用 Worker 解码档。
   （微信内置浏览器调不了摄像头，请点右上角「在浏览器打开」。）
3. 发送端：选文件 → 开始发送 → 全屏发射；接收端：启动摄像头，对准屏幕中央虚线框。
4. 完成自动校验 SHA-256，点保存。

### 线协议（v2，QR alphanumeric 单帧）

```
TID(8) + SEED(6) + TOTAL(4) + CRC(6) + PAYLOAD(base45)
```

定长头、无分隔符；种子 PRNG（LCG-16807）双端复现度分布与参与块索引，免传索引表。
完整协议说明与迭代记录见 `ANCHOR.md`，调研与路线图见 `RESEARCH.md`。

---

## Embedded libraries / 内嵌库

| 库 | 许可证 | 用途 |
|---|---|---|
| qrcode-generator (Kazuhiko Arase) | MIT | QR 生成 |
| zxing-wasm (Sec-ant/zxing-wasm, ZXing-C++) | MIT | 主解码器（WASM） |
| zxing-js (@zxing/library) | Apache-2.0 | 降级解码器 |
| jsQR | Apache-2.0 | 降级解码器 |

各库许可证文本见其源码/官网；内嵌区段在 `index.html` 内均有注释标注。

## License / 开源许可

本项目代码以 [MIT](LICENSE) 发布 © 2026 QR-AirBridge contributors。
