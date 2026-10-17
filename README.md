# EduTrace

跨媒介数字隐水印与内容溯源系统（Cross-media Invisible Watermarking & Content Tracing）。

> **状态：V0.1 方案设计 / PoC 验证阶段。** 当前仓库以技术设计为主；PDF/PPTX 截图、打印拍照、视频转码后的实际识别率仍需基于真实样本测量，不视为已实现功能。

## 核心目标

对同一教学资源的不同接收者/分发批次生成独立指纹，尽量保留原始使用体验，在出现流出的文档、照片、截图或视频片段时恢复对应的分发记录与技术证据。

| 媒介 | 兼容性承诺 / 设计边界 | 水印路线 |
|---|---|---|
| PDF | 优先保留正文搜索、选择与复制 | 结构指纹；视觉水印待验证 |
| **PPTX** | **分发 PPT 必须保持文字、图表和对象可编辑，不栅格化整页** | OOXML 结构标记；图片/文字原生微扰研究 |
| MP4 | 正常播放、音画同步；支持短片段实验 | 视频帧隐水印与多帧融合 |
| PNG / JPEG | 尽量保持图像质量与文字清晰 | PixelSeal / TrustMark 等对比基线 |

采用 **一个分发身份协议 + 多个可替换嵌入引擎**，而不是强行用同一算法处理所有媒介。视觉水印提取、可编辑 PPT 的结构指纹分别声明不同能力等级。

## 文档入口

- [技术方案 V0.1](docs/01-solution.md)：目标、整体架构、Payload/密钥、引擎设计、接口、数据模型与风险。
- [PoC 验证与验收](docs/02-validation-plan.md)：测试素材、场景矩阵、识别率/误报率与编辑兼容性。
- [开发路线图](docs/03-roadmap.md)：P0 技术验证、P1 服务化、P2 鲁棒性研究。

## P0 研发重点

1. **Payload V1 与分发登记**：唯一随机指纹、认证校验，避免在水印内写个人身份。
2. **PDF/PPTX 原生指纹**：最小侵入、写入与提取；PPTX 保留 PowerPoint 和 WPS 的编辑能力。
3. **图片与视频算法验证**：建立可复现实验，先实测 PixelSeal/TrustMark/VideoSeal 再决定技术选型。
4. **鲁棒性基准**：截图、打印/扫描、斜拍、导出、视频重编码及局部片段。
5. **Go/No-Go 决策**：统计失败和负样本；不以一个成功演示替代产品能力结论。

## 拟采用技术

Python 3.11+ / CLI（PoC）→ FastAPI；PyMuPDF；OOXML(`zipfile + lxml`)；Pillow/OpenCV；FFmpeg；可选 GPU 水印模型；国密认证与签名按用途以成熟实现接入。

## 安全与仓库约定

**本仓库目前公开可见（Public）。** 严禁提交真实学生信息、教辅商业母版、客户资料、分发映射、认证密钥、模型权重或原始泄露样本。样本应使用合成或获得授权的资料，测试产物放置在 Git 忽略目录中。若计划存放保密设计或正式教辅，请由仓库所有者在 GitHub Settings 将其调整为 Private。

本项目定位为溯源而非 DRM；检测出某份文件最初分发给某位接收者，**不等于证明该接收者实施了传播**。对 OCR 重制、全部重排、多副本差分和彻底重绘不保证可追溯。

## 参考项目

- [Meta VideoSeal / PixelSeal](https://github.com/facebookresearch/videoseal)
- [Adobe TrustMark](https://github.com/adobe/trustmark)
- [Microsoft PresentationML](https://learn.microsoft.com/en-us/office/open-xml/presentation/overview)
