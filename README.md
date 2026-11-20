# express-scan · 快递单号扫码识别（纯前端四级管线 + 可选后端兜底）

单文件网页 `index.html`，浏览器打开即用：拍照/选图（或实时相机抓拍）后，按 **条码 → 前端 OCR → YOLO ROI → LLM → Fallback API** 逐级降级，页面直接给出「单号 + 快递公司 + 可复制/可下载的结构化 JSON」。默认零后端、零安装、零训练、零 GPU——后两级只有你显式配置并打开开关才会走网络。

在线版：<https://Hana-ame.github.io/express-scan/>

## 识别管线（逐级降级）

| 级 | 手段 | 默认 | 触发条件 | 关闭/失败时 |
|---|---|---|---|---|
| 1 | 原生 `BarcodeDetector` → ZXing-js，三轮解码（原图 / 灰度增强 / 旋转 90°） | 开 | 总是先跑 | 继续下一级 |
| 2 | Tesseract.js `chi_sim` 前端 OCR | 开 | 第 1 级没出有效单号 | 静默跳过，`ocr=null` |
| 3 | YOLOv8n 检测 + ROI 裁剪二次条码 | **关** | 开关打开即跑（前两级已命中时只做展示） | 静默跳过，`yolo=null` |
| 4 | OpenAI 兼容 LLM 辅助 | **关（未配置）** | `on_no_tracking` / `always_assist` / `manual_only` | 错误写进 `json.llm.error`，结果不变 |
| 5 | 自定义 Fallback API | **关（endpoint 为空）** | `on_no_tracking` / `on_no_barcode` / `manual_only` | 错误写进 `json.fallback`，结果不变 |

**核心纪律：命中即停、绝不造号。** 第 1 级出了有效单号就不再跑 OCR；`status=ok` 之后 LLM / Fallback 不再自动触发（`always_assist` 是唯一例外，用于校验与结构化）。任何一级失败都**不影响**前面已经拿到的结果。

## 主结果 JSON

```json
{
  "request_id": "9f2c…",
  "status": "ok | partial | failed",
  "source": "photo | camera_capture | drop | paste | sample_barcode | sample_text_only",
  "decode_pass": "original | contrast | rotate90 | enhanced_all | direct",
  "latency_ms": 1650,
  "barcodes": {
    "1d": [{ "format": "CODE_128", "data": "SF123456789012", "points": [[120, 480], [620, 486]], "confidence": 0.99 }],
    "2d": []
  },
  "ocr": {
    "ran": true,
    "full_text": "…面单全文…",
    "lines": ["…"],
    "tracking_number": { "value": "SF123456789012", "company": "sf", "company_name": "顺丰速运", "method": "ocr", "confidence": 0.9, "status": "ok" },
    "latency_ms": 900
  },
  "tracking_number": { "value": "SF123456789012", "company": "sf", "company_name": "顺丰速运", "method": "barcode_1d", "confidence": 1.0, "status": "ok" },
  "yolo": { "engine": "onnxruntime-web", "ep": "webgpu|wasm", "imgW": 3024, "imgH": 4032, "detections": [], "roi_tracking": null, "latency_ms": 640 },
  "llm": { "invoked": true, "ok": true, "model": "gpt-4o-mini", "mode": "vision_and_text", "latency_ms": 3200, "tracking_number": null, "parsed": {}, "usage": {}, "error": null },
  "fallback": { "invoked": true, "ok": true, "endpoint": "https://…", "status_code": 200, "latency_ms": 800, "tracking_number": null, "raw_response": {} }
}
```

- `status`：`ok` 出单号；`partial` 扫到条码但都不符合单号规则（页面明确显示"扫到条码，但非快递单号"）；`failed` 什么都没读到。
- 顶层 `tracking_number` 是**最终采信**结果，各级自己的候选结果保留在 `ocr.*` / `yolo.roi_tracking` / `llm.*` / `fallback.*` 里，便于回溯是哪一级命中。
- `tracking_number.method` 取值：`barcode_1d`、`barcode_2d`、`ocr`、`yolo_roi_barcode`、`llm_oai`、`fallback_api`、`manual_ocr_pick`。

## 前端条码解码

- 长边先压到 **1600px** 再解码，控制移动端内存。
- **Pass 1** 原图 → 命中有效单号直接返回；**Pass 2** 灰度线性对比度拉伸（以 128 为中心 ×1.65）；**Pass 3** 旋转 90°（应对竖拍/倒置条码）。坐标会按变换矩阵反算回原图。
- 三轮结果按 `format|data` 去重合并，`decode_pass` 记录实际命中的那一轮。
- 工具栏「图像增强」关闭后只跑原图一轮（省时）。
- 引擎优先 `BarcodeDetector`（自动探测支持的格式表），不可用时加载 ZXing-js（jsdelivr → unpkg 两路 CDN）。

## 前端 OCR 兜底

- 只在第 1 级未命中、且「前端 OCR」开关为开时触发；**实时扫码路径不跑 OCR**（逐帧太重）。
- 模型加载三级尝试：配置的 `langPath`（默认同源 `./models/` 的 20MB `chi_sim.traineddata.gz`）→ 官方 `4.0.0_fast` 的 `chi_sim` → `eng`；任一成功即用，全失败静默跳过（`ocr=null`）。
- 先逐行匹配，再整段文本匹配；**11 位手机号一律排除**，避免把收件人电话当单号。
- 结构化详情里 OCR 的每一行都可点击**手动采信**（`method=manual_ocr_pick`），用于自动规则没命中但人眼能确认的场景。

## 前端 YOLO + ROI（默认关）

- 模型 `models/yolov8n.onnx`（12.8MB，同源托管），onnxruntime-web 1.20 以 `webgpu` → `wasm` 顺序建会话。
- letterbox 到 640 → 推理 → 输出两种布局都兼容（`[1,84,N]` 与 `[1,N,84]`，自动判通道序）→ 置信度阈值 0.25、NMS IoU 0.45。
- 取置信度 top3 的框、外扩 12px、短边 ≥24px 才裁，裁剪图再跑一次条码解码，命中即 `method=yolo_roi_barcode`（置信度 0.99）。
- 现状：COCO 预训练权重对面单没有对应类别，它验证的是**管线**；真价值是微调"单号区域"检测器（几十张标注即可）→ 导出 ONNX 覆盖同名文件，代码零改动。**待真机验证**：WebGPU 可用性、移动端推理时延。

## LLM 辅助（可选，默认关闭）

- OpenAI 兼容接口：Base URL 自动补 `/chat/completions`（已带则不重复），需 `model` + `API Key`；超时默认 25s。设置里有「测试 LLM 连通性」。
- 两种模式：
  - `vision_and_text`（默认）——图片压到长边 1280px、JPEG q0.82 转 dataURL，连同前端条码/OCR 文本一起发，适合视觉模型；
  - `text_only`——只发文本，适合纯文本模型或省 token。
- 三种触发：`on_no_tracking`（默认，前端都没出单号时才问）、`always_assist`（每次识别后都校验并结构化）、`manual_only`（工具栏「LLM 辅助」按钮手动触发）。
- 内置提示词要求返回**严格 JSON**（单号 / 公司 / 收件人 / 电话 / 地址 / 置信度 / 依据）；解析失败会退回文本正则兜底。命中后 `method` 统一为 `llm_oai`。
- 失败只记录到 `json.llm.error`，前端结果照旧展示。

## Fallback API（可选，默认不存在）

默认 `enabled=false`、`endpoint` 为空，**页面不会发起任何外部请求**。设置弹窗「Fallback API」页签里的可配置项：

| 配置 | 默认 | 说明 |
|---|---|---|
| Endpoint URL | 空 | 填了且开关打开才生效 |
| HTTP 方法 | `POST` | 可选 `PUT` |
| 触发时机 | `on_no_tracking` | 另有 `on_no_barcode`（前端没扫到码就发，跳过 OCR）、`manual_only`（工具栏按钮） |
| 请求体格式 | `multipart` | 或 `json`（图片转 Base64） |
| 图片字段名 | `file` | multipart 时的表单字段 |
| 附带前端结果上下文 | 开 | 把条码/OCR 文本一起送过去 |
| 鉴权方式 | `none` | `bearer` / `header`（自定义 Header 名） |
| 单号响应字段路径 | 空 | 留空自动在 JSON 里找，如 `data.tracking_number` |
| 额外请求头 | 空 | JSON 字符串，如 `{"X-Client":"express-scan"}` |
| 超时 | 15000ms | 1–120s |

**注意**：浏览器直连，目标服务必须开 **CORS**；页面是 HTTPS 时不能调 HTTP 接口（混合内容会被拦）。API Key / Token 明文存在本地 localStorage，仅适合个人设备。

## 界面与工具

- **四步状态条**：条码 / OCR / LLM / Fallback 各自显示待机、运行中、命中耗时、未配置、已跳过。
- **结果卡**：大字单号 + 公司徽标 + 来源徽标；支持复制单号、复制 JSON、下载 JSON、清空当前。
- **预览画布**：条码框绿色、YOLO 框红色，直接看定位准不准。
- **四个页签**：结构化详情（条码列表 / OCR 行 / YOLO 目标 / LLM 解析 / Fallback 响应）、原始 JSON、历史记录、单号规则。
- **历史记录**：`localStorage` 最多 50 条，可按单号/公司搜索、点条目回填结果、导出 CSV（时间/单号/公司/识别方式/耗时）。
- **单号规则页**：列出内置 11 条规则（顺丰、极兔、圆通、京东、EMS、德邦、申通、中通、韵达、邮政包裹、通用 12–15 位），带输入框可现场测试匹配。
- **测试工具**（默认折叠）：一键生成**合成条码单**（真 Code 128B，可被真机扫出 `SF123456789012`）、**无条码单**（只测 OCR/LLM 链路）、**OCR 基准**。

## 基准：本地 OCR / 推理有多慢

页面「测试工具 → OCR 基准」用一张合成面单样本（1240×1754，白底黑字含 `SF123456789012`）跑 Tesseract.js `chi_sim`，输出三段时延（模型加载 / 首次识别 / 稳态识别）并入 JSON。

**本机参考数（2026-09-30 实测：无头 Chromium 151 · WASM · CPU，Ryzen 5 3600）**：

| 项目 | 时延 |
|---|---|
| Tesseract.js 模型加载（20MB 同源） | **2.8s** |
| 首次识别（含 WASM 预热） | **1.2s** |
| 稳态识别 | **0.9s**（样本单号识别命中 ✓） |
| YOLOv8n 推理（ort-web WASM，640 输入） | **0.64s**（模型加载 1.7s） |

**手机预期**：移动 SoC 的 WASM 通常比桌面 CPU 慢 **2–5×**，即稳态识别约 **2–4s**、首次（预热）约 **3–6s**、20MB 模型按网络加载 5–20s；WebGPU 可用时 YOLO 可显著提速，OCR（Tesseract 无 WebGPU 后端）不受益。上表是旧版口径，新版引擎版本已升级，请以页面「OCR 基准」现场复测为准。

## 数据与隐私

- 配置和历史只存在浏览器 `localStorage`（`express_scan_config_v2` / `express_scan_history_v2`），清缓存即丢，不落盘、不上传。
- 未配置 LLM / Fallback 时，**照片全程不出设备**（解码、OCR、YOLO 都在浏览器里跑）。
- 一旦打开 LLM 或 Fallback，图片/文本会发往你自己填的地址——这部分由你的服务负责。

## 依赖与离线

- 无构建、无 npm、无打包：`index.html` 单文件 + `models/` 两个模型。
- 运行时 CDN 依赖：Tailwind（样式）、ZXing-js（BarcodeDetector 缺位时）、Tesseract.js（OCR）、onnxruntime-web（YOLO）。
- 断网时：内联逻辑和原生条码引擎仍可用；Tailwind 拉不到会导致**样式退化**，ZXing/OCR/YOLO 需先联网缓存过一次。

## 用法

1. 打开在线版，或把 `index.html` + `models/` 一起拷到手机用浏览器打开；本机预览可起静态服务：
   ```bash
   cd ~/express-scan && python3 -m http.server 8080   # 同源模型需要走 http，file:// 下模型会拉不到
   ```
2. 两条输入路径：
   - **拍照 / 选图 / 拖拽 / 粘贴**：任何环境可用，推荐主用；
   - **实时扫码**：需 HTTPS 或 localhost（`file://` 可能被拒绝相机），扫描到有效单号自动抓拍识别。
3. 结果出来后：复制单号 → 复制/下载 JSON → 切页签看各级明细。

## 浏览器兼容

| 能力 | 要求 | 兜底 |
|---|---|---|
| BarcodeDetector（一维+二维） | Chrome 83+ / Android、Safari 17+ / iOS 17+ | ZXing-js（CDN，需联网） |
| 实时相机 getUserMedia | HTTPS 或 localhost | 拍照上传路径 |
| WebGPU（YOLO 加速） | Chrome/Edge 113+ | 自动回落 WASM |
| 复制到剪贴板 | 安全上下文（HTTPS/localhost） | 弹窗手动复制 |
| LLM / Fallback 请求 | 目标服务开 CORS，HTTPS 页面调 HTTPS 接口 | 关闭开关，纯前端运行 |

Firefox 不支持 BarcodeDetector → 自动走 ZXing；CDN 也失败则页面提示改用后端路线。

## 退出条件（转路线 B）

用 ≥30 张真实面单（覆盖角度/亮度）实测，若**条码清晰面单命中率 <100% 或误读 >0**，且 OCR / YOLO ROI / LLM 也补不回来 → 纯前端不达标，退回后端 API（FastAPI + pyzbar/OpenCV + RapidOCR，CPU 运行），规格见 `notes/proj-express-tracking-ocr-api.md`。

## 约束与口径

- 单号规则（`index.html` 内 `RULES`）为**示意**，随快递公司版本变化，以官方为准；核心纪律：**宁可报"未识别"，不出假号**。
- 二维码内容可能是平台密文，`barcodes.2d[].data` 原样展示、不强行解析。
- 前端模型只接受"同名替换"升级（`models/yolov8n.onnx` 换权重即可，代码零改动）。

## 项目笔记

- 探索：`notes/proj-express-tracking-ocr.md`（方案全景、条码明文/加密、准确率口径）
- API 规格：`notes/proj-express-tracking-ocr-api.md`（路线 B）
- 方案：`notes/proj-express-tracking-ocr-plan.md`（里程碑与验收）
