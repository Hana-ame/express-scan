# express-scan · 快递单号扫码识别（M0 纯前端 + OCR 兜底扩展）

单文件网页：浏览器打开即用，拍照/选图上传 → 原生 BarcodeDetector 扫码（ZXing-js CDN 兜底）→ 单号正则判定 → 页面展示可复制 JSON（`barcodes.{1d,2d}` + `ocr` + `tracking_number`）。零安装、零后端、零训练、零 GPU。

## 扩展功能：OCR 兜底（M1）

条码**未出有效单号**时，自动尝试纯前端 OCR（Tesseract.js，`chi_sim` 中文模型）从面单文字里提取单号，结果填入 JSON 的 `ocr` 字段（`full_text` / `lines` / `tracking_number.method='ocr'`）。

**保证 fallback（核心原则）**：
- OCR 只在条码失败时触发，条码命中绝不跑 OCR；
- 引擎脚本、模型、CDN 任一加载失败 → **静默跳过**，条码结果与页面完全不受影响（`ocr` 返回 `null`）；
- 可手动关闭（页面「🔤 OCR 兜底：开/关」按钮），关闭后不触发；
- OCR 识别未命中单号 → `status=partial/failed`，只展示文字不报假号。

**注意**：
- 首次 OCR 需联网加载引擎（jsdelivr，小）与**中文模型 `models/chi_sim.traineddata.gz`（20MB，同源托管**，之后浏览器缓存）；
- OCR 仅作用于**拍照/选图**路径，实时扫码不做（实时帧太重）；
- 升级位：PaddleOCR.js（官方浏览器 SDK）待核实 API 后前置接入 `loadOcrEngine()`；
- **坑（已修）**：`chi_sim` 模型最初走 `cdn.jsdelivr.net/gh/naptha/tessdata@main/...`——**404**（该仓库默认分支是 `gh-pages`，且文件 20.16MB 超 jsdelivr 文件上限）→ 已改为同源 `./models/` 托管，此路必通；
- **待真机验证**：Tesseract.js 中文面单命中率。

## 基准：本地 OCR 有多慢（手机实测）

页面「⏱ OCR 基准」按钮：用一张**合成面单样本**（1240×1754，白底黑字含 `SF123456789012`）跑 Tesseract.js `chi_sim`，输出三段时延（模型加载 / 首次识别 / 稳态识别）并入 JSON（`bench` 字段）。

**本机参考数（2026-09-30 实测，无头 Chromium 151 · WASM · CPU，Ryzen 5 3600）**：

| 项目 | 时延 |
|---|---|
| Tesseract.js 模型加载（20MB 同源） | **2.8s** |
| 首次识别（含 WASM 预热） | **1.2s** |
| 稳态识别 | **0.9s**（且样本单号识别命中 ✓） |
| YOLOv8n 推理（ort-web WASM，640 输入） | **0.64s**（模型加载 1.7s） |

**手机预期**：移动 SoC 的 WASM 通常比桌面 CPU 慢 **2–5×**，即稳态识别约 **2–4s**、首次（预热）约 **3–6s**、20MB 模型按网络加载 5–20s；WebGPU 可用时 YOLO 可显著提速，OCR（Tesseract 无 WebGPU 后端）不受益。**真实数字请在手机上点「⏱ OCR 基准」报数**——参考口径就是上面那张合成样本。

## 扩展功能：YOLO 检测（管线已通，权重待微调）

选图后点「🔍 YOLO 检测」：**预训练 YOLOv8n（COCO 80 类）** 在浏览器里跑 onnxruntime-web（WebGPU → WASM 兜底）推理，检测框叠加在图上、结果并入 JSON（`yolo.detections`）。模型 `models/yolov8n.onnx`（约 13MB）随仓库同源托管在 GitHub Pages。

**保证 fallback**：ort 引擎 / 模型 / 推理任一失败 → 跳过并提示，不影响扫码与 OCR。

**现状与下一步**：COCO 预训练权重对面单无实际类别——它是**管线验证**；真正价值是**微调"单号区域"检测器**（用你的面单照片标注单号框，几十张即可，CPU 微调可行）→ 导出 ONNX 替换 `models/yolov8n.onnx`（文件名不变、代码零改动）→ 检测到单号区域后裁剪放大再走扫码/OCR，提升"角度不定、面单占比小"场景的命中率。**待真机验证**：WebGPU 可用性、WASM 推理时延、模型同源加载。

## 在线访问（GitHub Pages）

<https://Hana-ame.github.io/express-scan/> —— 手机浏览器直接打开即用（HTTPS，实时扫码可用）；代码在浏览器本地运行，**照片不出设备**。

## 用法

1. 把 `index.html` 发到手机（微信文件传输/数据线拷贝），用浏览器打开；或本机起个静态服务后访问。
   ```bash
   cd ~/express-scan && python3 -m http.server 8080   # 可选：起静态服务（不算部署）
   ```
2. 两条输入路径：
   - **🖼 拍照/选图**：任何环境可用（普通 HTTP 也能拉起相机），推荐主用。
   - **📷 实时扫码**：需 HTTPS 或 localhost（file:// 可能被浏览器拒绝相机）。
3. 扫码后页面大字显示单号 + JSON；点「复制 JSON」拿结果。

## 浏览器兼容

| 能力 | 要求 | 兜底 |
|---|---|---|
| BarcodeDetector（一维+二维） | Chrome 83+ / Android、Safari 17+ / iOS 17+ | ZXing-js（CDN，需联网加载） |
| 实时相机 getUserMedia | HTTPS 或 localhost | 拍照上传路径 |
| 复制 JSON clipboard | 安全上下文 | 手动复制 prompt |

Firefox 不支持 BarcodeDetector → 自动走 ZXing CDN；CDN 也失败则页面提示转路线 B。

## 退出条件（转路线 B）

用 ≥30 张真实面单（覆盖角度/亮度）实测，若**条码清晰面单命中率 <100% 或误读 >0**，纯前端不达标 → 退回后端 API 方案，规格见知识库 `notes/proj-express-tracking-ocr-api.md`（FastAPI + pyzbar/OpenCV + RapidOCR，CPU 运行）。

## 约束与口径

- 免训练、不装东西（运行时只依赖浏览器 + 兜底时的 CDN）、不用 YOLO、无 GPU。
- 单号规则（`index.html` 内 `RULES`）为**示意**，随快递公司版本变化，以官方为准；核心纪律：**宁可报"未识别"，不出假号**。
- OCR 兜底已启用（见上文扩展功能）；其加载失败/离线时自动跳过，`ocr` 为 `null`。
- 二维码内容可能是平台密文，`barcodes.2d[].data` 原样展示、不强行解析。

## 项目笔记

- 探索：`notes/proj-express-tracking-ocr.md`（方案全景、条码明文/加密、准确率口径）
- API 规格：`notes/proj-express-tracking-ocr-api.md`（路线 B）
- 方案：`notes/proj-express-tracking-ocr-plan.md`（里程碑与验收）
