# EduTrace 开发路线图 V0.1

## P0：可行性验证（当前阶段）

- [ ] 固定统一 Payload V1、随机短码、认证校验和分发身份映射。
- [ ] 定义统一 Engine 的 `inspect/embed/extract/validate` 接口。
- [ ] 编制不含隐私信息的 PDF/PPTX/图片/视频基准样本与评估脚本。
- [ ] PDF：完成结构指纹的最小嵌入与读取，并验证可搜索、可复制。
- [ ] PPTX：完成 OOXML 结构指纹，**保持可编辑**，不触碰无关对象。
- [ ] 通过 PowerPoint / WPS 测试 PPTX 修改、保存、再次提取、导出。
- [ ] 对比 PixelSeal / TrustMark 在教材页面的表现。
- [ ] 验证 VideoSeal 视频帧嵌入、转码、短片段抽帧与认证。
- [ ] 完成实验报告并决定进入 P1 或专项调整。

**P0 完成定义：** 两种原生文档格式可可靠写入/读取结构层分发 ID；PPTX 可编辑兼容性通过；至少一条图像及视频候选链路可运行且有量化指标及负样本数据。截图/印刷识别不在 P0 中提前视为“已实现”。

## P1：统一服务

- [ ] FastAPI + CLI，分发任务 / 检测任务 / 证据查询接口。
- [ ] Engine Router、分发登记、认证校验与密钥版本管理。
- [ ] 文件安全隔离、异步任务、幂等、失败重试和审计日志。
- [ ] PDF 原生结构、图片水印能力服务化。
- [ ] PPTX 对现有图片对象按白名单实施可回退视觉水印，不破坏可编辑文字/图形。
- [ ] Docker Compose、离线模型权重管理、部署文档。
- [ ] 自动化轻量单测和样本回归。

## P2：鲁棒性提升

- [ ] 中文字形/排版微扰在 PDF、PPTX 上的可行性研究。
- [ ] PPTX 文字和图形在截图、导出 PDF 后的识别与兼容性研究。
- [ ] 打印、复印、斜拍、对齐与多区域证据融合。
- [ ] 教学录屏、转码及截取视频片段的识别优化。
- [ ] 多副本差分、篡改和误归属风险评估。
- [ ] Vue 3 控制台及教学平台分发接入。

## 技术基线（候选）

Python 3.11+；PoC 首期 CLI；PyMuPDF；OOXML(`zipfile+lxml`)；Pillow/OpenCV；FFmpeg；VideoSeal/PixelSeal、TrustMark 仅作为**待基准验证**候选；国密封装独立可替换，禁止自研密码原语。

## 推荐仓库结构

```text
README.md
docs/
  01-solution.md
  02-validation-plan.md
  03-roadmap.md
src/edutrace/
  core/               # 协议与加解码
  engines/            # PDF/PPTX/图像/视频
  registry/           # 分发记录
  evaluation/         # 对照实验
  cli.py
tests/fixtures/       # 合成无隐私测试素材
benchmarks/           # 实验脚本与去标识数据
pyproject.toml
```

先以可重复测试取代复杂后台；大型 GPU 视频任务不放在 GitHub Actions 的常规 CI 流程中。
