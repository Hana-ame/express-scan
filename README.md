# express-scan · 快递单号扫码识别（M0 纯前端）

单文件网页：浏览器打开即用，拍照/选图上传 → 原生 BarcodeDetector 扫码（ZXing-js CDN 兜底）→ 单号正则判定 → 页面展示可复制 JSON（`barcodes.{1d,2d}` + `tracking_number`）。零安装、零后端、零训练、零 GPU。

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
- OCR 兜底（无条码/破损照片）属 M1，本期未启用，JSON 中 `ocr.*` 留空。
- 二维码内容可能是平台密文，`barcodes.2d[].data` 原样展示、不强行解析。

## 项目笔记

- 探索：`notes/proj-express-tracking-ocr.md`（方案全景、条码明文/加密、准确率口径）
- API 规格：`notes/proj-express-tracking-ocr-api.md`（路线 B）
- 方案：`notes/proj-express-tracking-ocr-plan.md`（里程碑与验收）
