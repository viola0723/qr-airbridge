# QR-AirBridge · 项目锚点

> 给新会话的快速定位文件。技术调研与迭代路线见同目录 `RESEARCH.md`。
> 最后更新：2026-08-05（①②已落地并真机复测通过；v1.4.0 滑块上限 FPS 50/单帧 1300 + 接收完成提示音；**已开源 <https://github.com/viola0723/qr-airbridge>**；**v1.5.0 已发布：#7 BarcodeDetector 原生第零档 + UI 修复两项；v1.5.1 探测可诊断（3s 超时 + envCheck 显示跳过原因），待安卓真机复测**；下一步 = 真机复测 → decode 侧 #9→#8→#10（安卓）/ #5 光学时序（iPhone 优先），讨论结论已固化于 RESEARCH §5/§8）

## 这是什么

单文件 HTML 应用 `index.html`（约 3450 行 / ~1.9MB，无构建工具、无外部依赖、所有库内嵌含 WASM base64）。
物理隔离（air-gap）文件传输：发送端屏幕轮播二维码，接收端手机摄像头扫码，
喷泉码（LT 风格）重组数据，全程不经网络。已实测可用，小文件体验好；
当前瓶颈是大文件的速率与稳定性。

吞吐状态：v0.9 基线 450B@3FPS ≈ 1.3 KB/s（真机实测）；
**v1.1 默认 450B@8FPS，用户设备实测 140KB @16~20FPS ≈ 7.5~8 KB/s（≈6× 基线）**；
**v1.3 wasm+Worker：900B@24FPS 解码 ~10/s（≈9 KB/s 供给率），密度路线解锁，参数可自由搭配**；
**v1.3.2 上限放宽后，iPhone 实测 1200B@30FPS 解码 ~50/s（36 KB/s 供给接近打满）——瓶颈回到发送端 FPS/光学**。
协议余量：base45 使同尺寸 QR 载荷 +53%（单帧加大需接收端成像升级解锁）；Soliton ε≈0.05。

## 真机实测数据（2026-08-03，用户设备，v1.1）

**小文件（20KB，450B）**：解码 ~50/s（每次采帧几乎都成功）→ 解码管线远未饱和。
3 FPS 重复 ~700（每码 ~16 次扫描）/ 8 FPS 重复 ~200（每码 ~6 次）；拒收 0；8 FPS 明显快（~6s）。

**140KB（450B，全屏发射）**：319 块 → 需 ~335 帧。
- 12 FPS：重复 500~900；16 FPS：重复 200~500；16 快于 12（≈28s→21s，5~6.7 KB/s）。解码 20~60 徘徊（手持/对焦/尺度死区）。
- **FPS 膝点 ≈ 20**：20 FPS 解码 ~20/s（成功解码≈新帧供给，刚好打平）；24 FPS 解码 <20/s（开始整帧错过，重复 <100）。
  光学极限机制 = 曝光横跨两帧切换瞬间拍花（表现为解码率掉，非拒收）；拒收全程为 0。
- **单帧膝点 ≤ 650（接收端成像限制，非协议）**：650/900B 解码 <10/s、900B 甚至 1/s——
  根因是快速通道固定 ≤640px：≥185 模块的密码被降到 2~3px/模块，ZXing 无码可找（非算力烧穿）。
- **甜点位判断**：450B@16~20FPS ≈ 8 KB/s 是该设备「单色 QR + JS 解码」的实用甜点——
  单帧加大 → 单帧解码变慢 → FPS 必须降，三个因子互相打架，密度杠杆被解码预算吃掉。
- 结论：解码算力在 450B 下过剩（~50/s），但在 650B+ 密码下是真瓶颈。
- 用户手机微距差（主摄最近对焦距离限制），「拿近扫密码」不成立；全屏发射已是全部测试的基准姿势。
- 喷泉码特性确认：发送快不压垮接收端（跟不上只是普通丢帧），故 FPS 默认值提高对慢设备无害。

**v1.2 分辨率阶梯复测（2026-08-03，用户设备）**：
- 650/900B：档位自动升满 1280，解码率从 ~1/s 回到个位数/s——「无码可找」的成像限制解除，
  但 1280 分辨率下 ZXing-js 逐帧解码的算力成为新瓶颈，密度路线吞吐仍不及 450B 甜点。
- 450B@16FPS：大部分稳 640 档、少量 960（迟滞回降正常），450B 路径无回归。
- 结论：①买到「找得到码」的鲁棒性/距离容忍（符合预期管理）；解码算力瓶颈坐实 → ②的翻案依据成立。

**v1.3 wasm+Worker 复测（2026-08-04，用户设备）**：900B@24FPS 解码 ~10/s、全程可用
（对比：v1.1 900B ~1/s 无码可找；v1.2 升满 1280 档后个位数/s）。解码管线瓶颈解除，
帧率/单帧可按文件大小自由搭配；用户判定当前性能达到可用状态。

**v1.3.1 苹果复测（2026-08-04，用户 iPhone，主线程-wasm 档）**：900B@24FPS 解码 **50+/s**，
远超安卓机（worker 档 ~10/s）。解读：wasm 解码是单核 CPU 活，iPhone 单核性能强是主因
（相机对焦/曝光更稳也抬单帧命中率）；iPhone 侧解码已超发送供给，限制回到发送端 FPS 与光学换帧混叠（膝点 ≈20~24）。

**v1.3.2 苹果复测（2026-08-04，用户 iPhone，主线程-wasm 档）**：**1200B@30FPS 解码 ~50/s**，
档位在 640↔960 徘徊（1200B=133 模块，iPhone 成像好到 640 档 4.8px/模块 即可解，无需升 1280）；
迟滞升降按设计工作。供给 30×1200B=36KB/s，解码 50/s ≈ 每帧平均解 ~1.7 次，供给接近打满——
当前上限受发送端 FPS/光学，而非解码或成像。

**v1.4.0 拉满复测（2026-08-04，用户 iPhone + 安卓）**：完成提示音/震动双端出声正常（iOS 手势解锁生效）。
**1300B@50FPS（双拉满）：iPhone 解码 ~30/s、安卓 ~10/s**。
解读：iPhone 解码率相对 v1.3.2（1200B@30FPS 的 ~50/s）不升反降——FPS 过膝点（≈20~24）后
光学换帧混叠主导，且 137 模块单帧解码算力也略增；FPS 拉满不划算，甜点位回到 24~30FPS 区间
换单帧 density（解码/s 口径含重复解码，精确捕获带宽以实际收文件耗时为准）。
安卓在拉满档仍 ~10/s，与 900B@24FPS 持平——解码算力受限，密度/帧率红利吃不到。

**定案的默认值**：FPS 3→**8**（实测扎实；慢设备上无害收敛）；单帧保持 **450**；滑块 FPS 1–50 / 单帧 300–1300 供折腾（v1.3.2→v1.4.0 两度上调上限）。

## 硬约束（迭代时必须遵守）

- 保持单文件、双击能开（`file://` 直接可用），不引入构建步骤。
- 第三方库一律以内嵌 `<script>` 形式加入，离线可用（WASM 须 base64 内嵌，禁 fetch）。
- UI 文案中文，代码注释中文。
- 目标场景：PC 屏幕 → 手机摄像头；现代 Chrome/Edge/Safari。
- 视觉通道单向无反馈（目前没有反向 ACK 通道，见 RESEARCH.md §7）。
  ⇒ 发送端参数（FPS/单帧）不存在「自动配置」，只能靠人看统计行调；接收端只能单方面自适应。
- **版本规约（2026-08-04 定）**：小调整不升号——bug 修复、参数/上限微调先记在「本轮已完成的改动」
  末尾的「未发布」段攒着；攒够一批或涉及以下任一情形才 bump：修复批次 → patch（v1.4.1），
  新能力（如提示音）→ minor，线协议变更 → major。发布 = push 即上线（Pages 自动重建），
  升号与发布同步做，不预支版本号。

## 备份策略

- `backups/` 存放只读基线副本：每轮较大改动前先复制 `index.html` 进去，命名 `index-v<版本>-<日期>.html`，并 `chmod 444` 防误改。覆盖只读备份前先 `chmod 644`，改完再 `chmod 444`。
- 2026-08-04 起项目已开源：<https://github.com/viola0723/qr-airbridge>（本地 git 仓库在 `main`，提交身份 viola0723 + noreply 邮箱；`backups/` 经 `.gitignore` 不入库，仍作本地只读基线）。
  **GitHub Pages 在线版已开通**：<https://viola0723.github.io/qr-airbridge/>（push 后自动重建，1~3 分钟生效）——https 环境是手机端最佳姿势（摄像头安全上下文 + iPhone worker 档解锁），推荐日常使用。
- 当前基线：`backups/index-v1.5.1-20260805.html`（#7 原生第零档 + 探测可诊断 + UI 修复两项）。改崩了直接拷回覆盖。
- 历史基线：`index-v1.5.0-20260805.html`（#7 BarcodeDetector 原生第零档 + UI 修复两项）、`index-v1.4.0-20260804.html`（滑块上限 FPS 50 / 单帧 1300 + 完成提示音）、`index-v1.3.2-20260804.html`（三级解码链 + 滑块上限 FPS 30 / 单帧 1200）、`index-v1.3.1-20260804.html`（三级解码链）、`index-v1.3-20260803.html`（zxing-wasm+Worker 首版）、`index-v1.2-20260803.html`（接收端分辨率阶梯）、`index-v1.1-20260803.html`（Robust Soliton 纯喷泉）、`index-v1.0-20260803.html`（协议 v2 首版）、`index-v0.9-20260803.html`（协议 v1）。

## index.html 结构地图（行号会漂移，以区段注释为准）

| 区段 | 大约位置 | 内容 |
|---|---|---|
| `<style>` | 7–74 | 全部 CSS；`.btnrow .full` = 全宽按钮 |
| 发送端 HTML | 86–139 | 卡片1选择内容 / 卡片2参数与发射 / 卡片3播放二维码流 |
| 接收端 HTML | 141–176 | 扫描接收卡 / 进度卡 recvProg / 完成卡 okCard |
| qrcode-generator (MIT) | 187–2486 | 二维码生成库，勿动；`addData(data,'Alphanumeric')` 支持低密模式 |
| jsQR (MIT) | 2487–2496 | 备用解码器 |
| zxing-js (Apache-2.0) | 2497–2499 | 降级链主解码器（HybridBinarizer 局部自适应二值化） |
| zxing-wasm reader (MIT) | 2502–2503 | `text/plain` 惰性块（不执行）：IIFE glue + WASM base64，作 Blob Worker 源码/字节仓 |
| AirBridge 引擎 | 2504–2971 | CRC32 / SHA-256 / base45 / deflate / Soliton / Fountain / Sender / Receiver / Store |
| 应用层 | 2973–末尾 | 全局状态、微信引导、发送端、接收端、bd 第零档 + wasm worker 管线、帧处理、Tab 切换 |

## 线协议（v2 帧格式 + v1.1 度分布/种子语义，v1.2/v1.3 未动协议、v1.1+ 可混发收；v0.9/v1.0/v1.1 两两互不兼容，同文件收发同步升级）

QR **alphanumeric** 模式文本，定长头、无分隔符：
`TID(8) + SEED(6) + TOTAL(4) + CRC(6) + PAYLOAD(base45)`

- `TID`：8 字符 base36 大写随机任务 ID，接收端靠它识别新任务并重置（**先验 CRC 再认任务**）。
- `SEED`/`TOTAL`：base36 大写补零定长。SEED 空间 36^6≈21 亿；TOTAL 上限 36^4≈168 万块。
  PRNG(16807 线性同余) 由 SEED 复现度分布与参与块索引，**免传索引表**；
  索引生成统一在 `Fountain.idxsOf`（makePacket/processPacket 两处共用）。
  **注意：PRNG 首输出对小种子非均匀**（s·16807 不取模缠绕），随机种子务必用大值（生产用 [L, 36^6) 均匀随机，测试勿用步进小种子探针）。
- **seed < L 约定**：seed 值即原始块索引的度1帧（不走 PRNG），用于每 32 帧重发清单块（seed 0）。
- `CRC`：uint32 的 6 字符定长 base45（低位在前）。CRC32 输入 = **头部 18 字符 ASCII + 载荷字节**。
- `PAYLOAD`：base45（RFC 9285，字母表 = QR alphanumeric 45 字符）编码的 XOR 块。
  块长 B → ceil(B/2)*3 字符；450B 块整帧 699 字符，比特数约为 v1 的 65%。
- **清单块（替代 v1 每帧 META）**：喷泉块 0 = `[lenHi, lenLo, ...JSON(UTF-8), 零填充]`，
  JSON = `{"n":文件名,"s":原始大小,"h":sha256hex,"c":0/1,"t":transmitLen}`（文件名超长时砍半截短）。
  清单就绪前 UI 显示「任务 \<TID\>（等待清单块…）」。
- **发送策略（Sender.nextSeed）**：纯 Robust Soliton 喷泉，随机种子 [L, 36^6)；
  每 32 帧插一帧清单块（seed 0）。**无系统码第一轮**（实测否决，见下）。
- **度分布**：Robust Soliton（c=0.1, δ=0.5，CDF 按 L 缓存于 `solitonCDF`），
  替换 v1.0 的 40/30/20/10 经验值；贪心消元（peeling）不变。
- 接收端：**SEED 去重**（'dup' 计入统计）；帧处理顺序：定长解析 → base45 → CRC → 认新任务 → 去重 → 消元 → 块 0 就绪则解析清单。
- 默认参数：单帧 450B（滑块 300–1300）、**8 FPS（滑块 1–50）**、QR 纠错 M、deflate 可选、文件级 SHA-256 兜底。

## 本轮已完成的改动（2026-08-03）

**v1.0 = RESEARCH §8 第 1 项 + 死代码清理**
- 协议 v2：base64→base45、每帧 META → 清单入喷泉块 0、CRC 覆盖头部、定长无分隔符。
- 接收端 SEED 去重 + 先验 CRC 再认新任务；扫描统计加「重复」计数。
- 清理：删死代码 `initSenderQRs()`、删 `u8ToB64/b64ToU8`；喷泉索引逻辑合并为 `Fountain.idxsOf`。

**v1.1 = RESEARCH §8 第 3 项（按实测修正后落地）+ 真机调参**
- Robust Soliton（c=0.1, δ=0.5）替换经验度分布；`solitonCDF` 按 L 缓存。
- **系统码第一轮：实测后否决，未采用**。无头矩阵（交付帧，6 种子均值，字节核验）：
  纯 Soliton 在 L∈{9,21,70} × loss∈{0,0.15,0.30} 九场景中赢七场，且对丢帧免疫
  （L=70 时 F≈73~76 恒定，ε≈0.05）；系统轮仅在 0% 丢帧恰 L 帧（省 ~5 帧），
  有损时丢失的系统帧成「洞」，尾部回补代价更高（L=70@15% 丢：100 vs 75 帧）。
  经验分布从头解码 ε 实测 0.2~0.5，证实 RESEARCH 的担心；Soliton 从头解码 ε≈0.03~0.07。
- 版本号 v1.1（envCheck + footer）。
- 验证：node 引擎回环 22 断言全绿（含中途加入/篡改拒收/去重/空文件 L=1/ε 抽查）；
  headless Chrome QR 编解真回环 PASS；界面截图无破版。
- **真机参数扫描定默认值**：FPS 3→8、滑块上限 FPS 24 / 单帧 900（膝点数据见「真机实测数据」）。
- 注意：某子 agent 曾把丢帧测试的「总发送帧（含丢失）」当 ε 分子，得出偏差的对比数字；
  正确口径是**交付帧**（丢帧不计）。复核 harness 已删除，方法记录在本节备查。

**v1.2 = ANCHOR 任务书①（接收端分辨率阶梯）**
- `MS_SCALES` 固定尺度表 → `CAPS = [640, 960, 1280]` 档位 + 档内比例轮换 `CAP_SCALES = [1, 0.6, 0.45]`。
- `capLadder` 状态机：连败 12 次升一档、连胜 10 次降一档（迟滞），只统计快速通道；
  独立小函数、直接操作传入状态对象，便于无头单测。
- 统计行加「档位」；`startStats` 复位阶梯（每次开扫从 640 档起步）；深度通道（每 20 帧全帧 ≤1280）不动；
  0 档 = 历史固定 640 行为（450B 路径零回归）。
- 版本号 v1.2（envCheck + footer）；线协议未动，v1.1/v1.2 可混发收。
- 无头验证：语法双段 `node --check` OK；capLadder 12 断言全绿（升/降/到顶/到底/互清/打断重计）；
  QR 真回环回归 PASS（450B 块、L=12、decoded=15/22、字节比对 5400/5400 一致，含 0.6×/0.45× 重采样路径）；
  界面截图无破版。
- **真机复测已完成**（2026-08-03，见「真机实测数据」）：650/900 升满 1280 档、解码个位数/s
  （成像限制解除、算力成新瓶颈）；450B@16FPS 稳 640 档为主、少量 960，无回归。

**v1.3 = ANCHOR 任务书②（zxing-wasm + Worker 解码管线）**
- 内嵌：zxing-wasm v3.1.2 reader（MIT，Sec-ant/zxing-wasm；WASM 的 SHA256 与包内公布值核对一致）
  以两个 `text/plain` 惰性块入文件（IIFE glue 38KB + WASM base64 1.42MB），单文件涨至 ~1.9MB（预算内）。
- 管线：主线程采帧/裁剪/阶梯；像素 transferable 零拷贝发 Blob Worker；worker 内 [原生,0.6×,0.45×]
  一次试完（`rescaleImg`/`zxwWorkerMain` 经 `.toString()` 注入，免两份维护——二者必须保持自包含）；
  忙则丢帧（pull 模型：忙时不采帧，统计口径与同步链一致）。全程无 fetch：`wasmBinary` override
  直喂字节；Blob URL 建 worker（file:// 均可用）。输入走鸭子类型 `{data,width,height}`（免 ImageData 依赖）。
- 降级：初始化失败 / 8s 超时 / onerror → `failWasm()` 退回 v1.2 同步链（zxing-js→jsQR，代码路径未动）；
  envCheck 解码器栏随就绪/降级刷新（`renderEnvCheck`）。
- 无头验证：语法三段 `node --check` OK；node 引擎回环 PASS；capLadder 12 断言全绿；
  **worker 真回环 PASS**（450B 块、L=12、18/18 全解码 miss=0、字节比对 5400/5400 一致，含 0.6×/0.45× 路径）；
  同步链抽检 7/8（zxing-js 对 699 字符密集码的天然 miss，v1.2 起即如此，非回归）；截图无破版。
- **踩坑记录**：嵌入二进制时 `String.replace(anchor, 含$的大字符串)` 会把替换串里的 `` $` ``/`$'` 当模式
  解释，将锚点前/后全文复制进文件 → 损坏。对策：一律用替换函数 `replace(anchor, () => block)`。
- **真机复测通过**（2026-08-04，用户设备）：900B@24FPS 解码 ~10/s，全程可用；详见「真机实测数据」。

**v1.3.1 = 苹果设备降级修复（三级解码链 + 失败原因可见）**
- 用户报告苹果系统走不到 worker-wasm（降级）。根因未定（blob worker / wasm 编译受限皆有可能），
  故补**二级降级**：worker 失败 → 主线程实例化 wasm（解码时短暂卡 UI，但仍远快于 zxing-js）→ 三级同步链不变。
  glue 经 `new Function` 执行返回 ZXingWASM 对象（不污染全局），同样 wasmBinary 免 fetch，8s 超时判败。
- `failWasm(原因)` 记录失败原因（worker报错/就绪超时/创建异常/wasm初始化err），envCheck 降级时显示
  （截 30 字），便于真机回报定位；二级失败原因在 `zxw.mainErr`。
- 解码回填统一为 `onDecodeDone`（三条链共用）；scanLoop 忙判扩展为 `(zxw.busy || zxw.mainBusy)`。
- 无头验证：语法 OK；19 断言全绿（一级回环 17/17 字节一致；**模拟一级失败 → 二级接管**、
  二级解码文本正确、envCheck 显示「主线程」；三级抽检 5/8 符合 zxing-js 天然 miss 率）；截图无破版。
- **苹果复测结果（2026-08-04，用户 iPhone）**：显示 `ZXing-wasm✓(主线程)`——worker 路径失败、二级主线程 wasm
  接管成功；900B@24FPS 解码 50+/s，反超安卓机（worker 档 ~10/s）。
  worker 失败根因已回报：`worker报错: Script error`——WebKit 在不透明源页面（file:// / App 内文件预览）拦截
  blob worker 且脱敏错误内容（http(s) 页面下 Safari 的 blob worker 正常；如需 worker 档可经本地 http 服务打开页面）。
  **GitHub Pages（https）实测闭环（2026-08-04，同机）**：envCheck 显示 `ZXing-wasm✓`，一级 worker 档复活——根因推断确认。

**v1.3.2 = 滑块上限上调（给强机余量）**
- FPS 上限 24→30、单帧上限 900→1200（默认值 8/450 不变；弱机自行调低即可）。
- 依据：iPhone 主线程档 900B@24FPS 解码 50+/s，远超供给；容量核算 1200B → 1824 字符 → 133 模块，
  1280 档 9.6px/模块（QR-M 上限 3391 字符，余量充足）。
- 无头验证：1200B worker 真回环 PASS（L=6、9/9 解码 miss=0、字节比对 7200/7200 一致）。
- 注意：FPS>24 的收益受光学换帧混叠限制（膝点 ≈20~24，见「真机实测数据」），强机可试 24–30 寻优。

**v1.4.0 = 滑块上限再上调（FPS 50 / 单帧 1300）+ 接收完成提示音**
- FPS 上限 30→50、单帧上限 1200→1300（默认值 8/450 不变）。
- 依据：v1.3.2 iPhone 实测 36KB/s 供给接近打满、瓶颈回到发送端 FPS/光学，继续给强机留寻优余量；
  容量核算 1300B → 1974 字符（24 头 + 1950 载荷）→ QR-M 自动选版 v30 / 137 模块，
  1280 档 9.3px/模块（QR-M 上限 3391 字符，余量充足）。
- 完成提示音：Web Audio 即时合成，免音频资源、保单文件离线。校验通过 = 上行三音 chime
  （659/880/1318Hz），失败 = 低双音告警；附带 `navigator.vibrate`（支持设备上随震动）。
  iOS 只在用户手势窗口解锁 AudioContext——`ensureAudio()` 同步跑在「启动摄像头」点击栈里预热，
  出声在 `finalizeRecv` SHA 校验后。
- 无头验证：node 引擎回环 10 断言全绿（1300B 上限帧 20KB 随机数据 L=17 字节一致 + SHA 通过 +
  帧长 1974 全 alphanumeric + QR-M v30/137 模块可容纳 + 450B 默认回归字节一致）；app 块 `node --check` OK。
  **真机复测已完成**（2026-08-04，见「真机实测数据」）：提示音/震动双端正常；拉满档 iPhone ~30/s、安卓 ~10/s。

**v1.5.0 = #7 BarcodeDetector 原生第零档 + UI 修复两项（2026-08-05 发布，push 即上线）**

**#7 BarcodeDetector 原生第零档（2026-08-05）**
- 探测两步：`'BarcodeDetector' in window` + `getSupportedFormats()` 含 `qr_code`（部分平台构造函数在但无 QR 后端）。
  就绪即最优先：detect 走系统/硬件检测服务不卡主线程，直接吃 `scanCv` canvas（省 getImageData 一次全帧拷贝），
  多尺度由检测器内部处理（不再轮换 CAP_SCALES）；忙则丢帧/pull 模型、onDecodeDone 回填口径与 wasm 路径一致。
- 降级：不支持（iOS/桌面 Safari 未原生支持，WebKit 仅旗标实验）探测静默跳过；detect 连续 3 次异常判半坏退回
  wasm 链（`bd.failed`，envCheck 标「原生✗」）。支持面：安卓 Chrome、Android WebView 83+、Edge、桌面 Chrome（macOS/Windows/ChromeOS）。
- 改动点：scanLoop 派发加 bd 分支（getImageData 下沉到各 wasm/同步分支内）、busy 门加 bd 位、
  `initBarcodeDetector`/`bdDetectText` 新函数、envCheck 第零档显示；引擎与线协议未动。
- 无头验证：app 块 `node --check` OK；node 接线 harness **15 断言全绿**（mock BD/Worker/DOM 边界 + 真引擎喷泉回环）：
  探测 → envCheck 原生✓ → 忙则丢帧（挂起期间 rAF 不再采帧）→ bd 独占 52 帧完成 20KB 字节一致、wasm 零调用 →
  连续 3 次异常降级 → wasm 二级接管完成字节一致 → 无 BD 环境 wasm 链独立字节一致（harness 用后已删）。
  **真 BarcodeDetector QR 真回环已验**（2026-08-05，Electron Chromium 148 / macOS Vision 后端：
  20KB L=47，50/50 帧全解码 miss=0，assemble 字节逐位一致；方法见「无头验证方法」末节）。
  剩余未验 = 安卓硬件解码速率与摄像头光学路径，留待安卓真机复测（对比 wasm worker 档 ~10/s 基线）。
- 备注：README 解码链描述已随发布同步为「原生第零档 + wasm 三级链」（中英文两节）。

**UI 修复两项（2026-08-05，Electron 截图复验）**
- **压缩徽标残留（bug）**：`resetSend` 未复位 `compressBadge`（在卡片2 checkbox 旁，sendPanel 之外），
  重置后界面仍显示上次任务的「已压缩 x→y」——已修为复位「自动」，截图前后对照确认。
- **接收端「重置所有状态」整页刷新 → 原地复位**：原为 `location.reload()`，弱网（Pages https）要重载
  ~1.9MB 页面 + 重编译 WASM + 丢掉解码器预热；接收状态本就收在 `recv`/`recv.receiver` 两处，原地清理即等价。
  现行为：停扫描 + 弃喷泉状态 + 计数/阶梯/进度卡/完成卡/统计行清零，页面不刷新（`window` 标记存活证实无 reload）。
  附带行为变化：旧 reload 会连带清空发送端状态，新 resetAll 只管接收端（发送端有自己的「重置状态」按钮）。
  tab 记忆 hack 保留（应对手动刷新/后台回收），switchTab 注释已同步。
- 验证：app 块 `node --check` OK；Electron 运行时断言全过（徽标回「自动」；resetAll 后 receiver=null /
  TID 清空 / 计数清零 / 双卡隐藏 / 扫描按钮文案复原）；两tab 界面截图无破版。

**v1.5.1 = #7 探测可诊断（2026-08-05，用户安卓机未见「原生✓」后定位用）**
- 背景：v1.5.0 用户安卓机 envCheck 显示 `ZXing-wasm✓` 无「原生✗」= 探测阶段静默跳过，原因不可见
  （候选：旧版缓存 / 非 Chrome 浏览器无 API / 真 Chrome 无 GMS 后端导致 formats 无 qr_code）。
- 改动：`initBarcodeDetector` 加 **3s 探测超时**（formats 查询挂起不能永远等）；
  跳过原因写入 `bd.skip`（无API/无qr_code/探测超时/异常:*）并显示在 envCheck（`(原生✗:原因)` 后缀，
  与 failWasm 诊断同风格）；版本号 v1.5.1。
- 验证：node 探测矩阵 harness **11 断言全绿**（正常/无API/无qr_code/挂起超时/异常 五路的 ready、skip、
  envCheck 文案全对，harness 用后已删）；Electron 注入 `delete window.BarcodeDetector` 截图确认
  envCheck 显示「ZXing-wasm✓（原生✗:无API） · v1.5.1」。
- **待真机回报**：用户安卓机 envCheck 的「原生✗」后缀原因 = 定位依据（无API→换浏览器；无qr_code→设备无 GMS 后端，安心用 wasm 档；探测超时→后端模块加载不来）。

## 已知遗留

- 引擎里有 IndexedDB 断点续传 `Store`，但接收流程未接入（仅在接收完成时 `remove`）。
- 纯喷泉进度条非线性（peeling 雪崩效应：前段慢、尾段陡），大文件 UX 待真机观察。
- Soliton c/δ 参数留待 §8 #4 按真实信道丢帧率调优（当前 c=0.1, δ=0.5）。
- Chrome 截图相对路径会写失败，需用 Windows 绝对路径（`--screenshot="C:\\..."`）。

## 无头验证方法（本会话验证过，可复用）

- 语法检查：抽出目标 `<script>` 段 → `node --check`（引擎段约 2500 起、应用层段约 2969 起）。
- **引擎免浏览器全回环**：引擎是 UMD（`module.exports`），抽出后 `require` 进 node harness，
  Sender 喂 `{name, arrayBuffer}` 假文件即可驱动 makeFrame→Receiver.handleFrame→assemble 全流程
  （node ≥18 有 CompressionStream/crypto.subtle）。可复现随机用 LCG，勿用 Math.random。
- **ε/参数对比**：同法抽两份引擎（当前版 + backups 里的旧版），同一 LCG 丢帧流驱动，
  统计**交付帧** F 与 ε=(F−L)/L；每轮必须 assemble 后字节比对，防"假完成"。
- **QR 真回环**：临时副本注入脚本：drawQR 到画布 → getImageData → 应用层 decodeQR（ZXing 真解码）
  → Receiver，配 `--virtual-time-budget` 跑 headless Chrome，结果写 `<div id="testResult">` 后 `--dump-dom | grep` 读取。
  **注意：virtual-time 在 worker/离线程异步下会早退**（虚拟时钟快过真实异步），v1.3 起 worker 链路须用 CDP：
  `chrome --headless=new --remote-debugging-port=9223` + node（≥22 全局 WebSocket）连 page target，
  `Page.navigate` 后每 500ms `Runtime.evaluate` 轮询 `#testResult`（本轮 harness `_zw/cdp_run.js` 用后已删，要点即此）。
- 界面截图：`chrome --headless=new --window-size=600,1500 --screenshot=out.png "file:///.../index.html"`；
  截图用 ReadMediaFile 回看一眼再下结论；临时副本与截图用完删除。
- **本机无 Chrome 时的截图/JS 驱动（2026-08-05 起，macOS）**：在 /tmp 下 `npm install electron`（勿入仓），
  offscreen BrowserWindow + `executeJavaScript` 可驱动页面内任意函数并回读 JSON，`capturePage` 出 PNG。
  要点：① 本机全局环境变量 `ELECTRON_RUN_AS_NODE=1` 会把 Electron 退化成纯 node，调用必须 `env -u ELECTRON_RUN_AS_NODE`；
  ② VS Code/Kimi 等魔改 Electron 二进制优先跑自家 app（传入的 app 路径被当 CLI 参数忽略），不能当通用 runtime，
  要用 npm 装的原件；③ qlmanage 缩略图不执行 JS，只能看静态版式；④ safaridriver 需在 Safari 设置手动开
  「允许远程自动化」，未开不可用。harness 在 `/tmp/eleshot`（main.js 通用：`<html> <png> [pre.js] [result.json]`）。
- Chrome 路径：`C:/Program Files/Google/Chrome/Application/chrome.exe`。

## 下一步（#7 已实现未发布；真机复测后按 #9→#8→#10 推进）

### ① 接收端分辨率阶梯 —— ✅ v1.2 已落地（真机复测完成）

- 改法与验收记录见「本轮已完成的改动」v1.2 节；复测数据见「真机实测数据」v1.2 节。
- 结论：成像限制解除（650/900 升满 1280 档后找得到码），但解码算力成为密度路线新瓶颈 → ②解锁依据成立。

### ② zxing-wasm + Worker 解码管线 —— ✅ v1.3 已落地（真机复测通过）

- 改法与验收记录见「本轮已完成的改动」v1.3 节；复测数据见「真机实测数据」v1.3 节。
- 结论：900B@24FPS 解码 ~10/s（v1.2 同期个位数/s），解码瓶颈解除，用户判定可用。

### ③ decode 侧 #7~#10（安卓 ROI 序）/ #5 光学（iPhone 主菜）

- **#7 BarcodeDetector 原生第零档 —— ✅ 已实现（未发布攒批，待安卓真机复测）**。改法与验证见「本轮已完成的改动·未发布」节。
- 复测通过后按 #9 ROI 跟踪 → #8 Worker 池 ×2 → #10 换帧检测门控推进；#5（相机参数锁定/帧同步信标）是 iPhone 抬供给的主菜。

### 远期（知道即可）

- #5 光学时序收尾：帧同步信标（抬 FPS 膝点）、相机参数锁定（曝光/对焦，用户手机对焦会飘）。
- #4 自动扫描工装（Soliton c/δ 一并调优）；#6 彩色网格（密度 ×10，②的解码底座已就绪）/ 反向 ACK。
