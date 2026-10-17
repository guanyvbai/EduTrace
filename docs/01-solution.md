# EduTrace 跨媒介数字隐水印与溯源系统 — 技术方案 V0.1

- 文档状态：技术设计草案
- 日期：2026-10-17
- 项目形态：独立服务；预留向教学管理平台提供 API
- 首期重点：可行性、兼容性、抗变换性能的实证验证

## 1. 需求背景与目标

教材通常同时以 PDF、PPTX、教学视频和图片格式分发。分发对象可能转发原文件，也可能导出 PDF、截图、打印、复印、拍照、录屏、压缩或转码。目标是在尽量不影响内容使用体验的前提下，为每个分发副本植入可追踪指纹，并在发现外流副本时提取证据、关联到初始分发记录。

### 1.1 目标

1. 提供统一的水印编码、分发登记、密钥管理、检测和溯源接口。
2. 支持 PDF、PPTX、MP4、PNG/JPEG；适配算法可插拔。
3. PPTX 分发文件保留原生文字编辑、图形、图片、表格等正常编辑能力。
4. PDF 优先保留文本提取与搜索；视频保持音画、播放兼容性。
5. 能从未破坏的原始文件准确提取结构指纹；视觉载体争取支持截图、拍照、转码等变换。
6. 检测结果具有可核查的证据、算法版本、校验结果与失败原因。

### 1.2 非目标与边界

- 不是 DRM，不阻止复制、截图、打印、录屏。
- 不承诺所有水印经 OCR 重建、彻底重绘、裁去所有水印区域或多副本共谋后仍能恢复。
- 不能把“文件最初发给某人”推断成“某人必然主动泄露”。
- 不用栅格化幻灯片替代原生可编辑 PPTX。
- V1 不训练通用深度模型，不投入复杂管理后台。

## 2. 核心思路：一个协议，多种载体

```text
原始文件 + 分发对象 + 分发批次
              │
       Distribution Service
       产生唯一分发 ID
              │
      Payload Codec / Keyring
    载荷映射 + 校验 + 纠错参数
              │
      Engine Router（按格式路由）
         ├─ PDF Engine
         ├─ PPTX Engine
         ├─ Image Engine
         └─ Video Engine
              │
      带水印文件 + 分发登记 + 验证结果

待识别材料（文件 / 截图 / 扫描 / 片段）
              │
       Normalize / Detect Router
              │
       各引擎提取 + 纠错/校验
              │
    Evidence Aggregator + Registry
              │
       溯源报告 / 无法确定
```

### 2.1 身份模型

- `content_id`：逻辑教材/课件/课程资源。
- `revision_id`：内容版本，独立于文件二进制哈希。
- `recipient_id`：接收者内部标识，仅在受权限保护的服务端保存。
- `distribution_id`：每次分发的不可预测随机唯一 ID。
- `bundle_id`：可选，关联同一课程包下的多个文件；多个文件仍各有自己的 `distribution_id`。
- `watermark_payload_id`：分发编号在特定算法容量下的短码映射。
- `algorithm_id`/`algorithm_version`：嵌入与提取算法版本。
- `key_version`：密钥版本，支持轮换。

### 2.2 载荷与密码学

推荐 **先生成短随机码并映射后端，再按算法能力编码**；不同算法容量不同，不能强行规定所有引擎采用同一字节数。

- 载荷示意：`schema_version | short_distribution_token | truncated_auth_tag`。
- 校验：使用经正式设计的 HMAC-SM3（或可信现成密码库实现）；认证标签长度及伪造概率需通过威胁模型确认。
- 错误恢复：依算法提供的 BCH/RS/软判决机制，外部纠错不要与模型内部编码简单堆叠而不测试。
- SM4 可用于数据库字段/敏感文件加密，而不是默认把大段密文放入水印。
- SM2 用于签名分发清单与报告，按需实施；并非 V1 阻断项。
- 数据库中保存哈希与映射，解码必须通过 MAC/登记记录/算法证据联合验证；不能仅凭随机命中就肯定溯源。
- 密钥用 KMS/Secret 管理，不写入 Git 仓库、日志或导出报告。

## 3. 分媒介策略

### 3.1 PDF

**基线层（确定性）**：写入文档属性/自定义元数据作为结构指纹，原生文件直接读取方便；可能被重新导出、去元数据移除。

**视觉层（研发）**：对可嵌入图片的局部区域使用图像水印；文字密集区域探索字形、字距、微结构或页面坐标编码。尽量原位修改文档绘制对象并保留可搜索文本，不把整页渲染成单张图作为正式分发版。截图、打印识别属于 PoC 结果，不提前承诺。

**提取**：读取元数据；再对 PDF 页面渲染图像开展视觉检测；对拍照样本先透视校正、版面定位，汇聚多区域证据。

### 3.2 可编辑 PPTX（关键约束）

**结构层（V1）**：对 OOXML 包写入关联信息/自定义属性，并尽可能不变更未涉及的部件、关系、动画、母版和主题。测试 PowerPoint/WPS 打开、编辑、保存和导出行为。

**视觉层 A（研发）**：对原有照片、插图等图片对象替换为内容带水印的等尺寸素材，保持对象可移动、可删除、可替换。图片被替换后指纹可能消失。

**视觉层 B（高风险探索）**：在 OOXML 文本/形状对象上使用字符样式、字距、坐标的微量差异构造编码；可能改变换行、字体替换、对齐或被“统一格式”操作清除。必须受白名单、布局差分与回退机制约束，不能直接作为可靠鲁棒水印承诺。

**原则**：对动画、SmartArt、特殊图表、嵌入对象、公式、宏等复杂内容优先保持原样；对无法安全嵌入的幻灯片返回能力说明而非破坏文件。

**提取**：结构指纹解析；文件导出为 PDF/截图时仅能依赖视觉层；验证需区分 `native_file` 和 `rendered_capture` 两类路径。

### 3.3 图像

比较 Meta PixelSeal、Adobe TrustMark 等模型，测试不同分辨率与文字/插图/纯白背景。评估 JPEG 压缩、缩放、裁剪、拍照、旋转、颜色空间与亮度调整。可根据测试结果对纹理区域/图片块加权嵌入，纯白区域降低或放弃嵌入。

### 3.4 视频

使用 VideoSeal 作为优先候选；FFmpeg 负责解封装、解码、抽帧与编码。可对同一个短码在时间上冗余嵌入，检测时多帧投票并输出时间戳证据。

不得假设每帧独立处理就自动具有良好视觉时序一致性；应使用模型支持的视频流程并评估闪烁与画质损失。保留音轨、时长、FPS/时间戳与音画同步，处理失败保留原件且不返回已成功状态。

## 4. 统一接口（概念级）

```http
POST /api/v1/contents                  # 注册逻辑内容/版本
POST /api/v1/distributions             # 创建分发记录并发起嵌入任务
GET  /api/v1/jobs/{job_id}             # 任务进度与结果
POST /api/v1/detections                # 上传待检测文件并发起提取
GET  /api/v1/detections/{detection_id}  # 校验/证据/分发结果
GET  /api/v1/algorithms                # 查询各算法能力和已验证场景
```

嵌入引擎最小接口：`inspect(file) -> capability_report`、`embed(file,payload,options) -> output+evidence`、`extract(file,options) -> candidates+evidence`、`validate(output) -> checks`。

### 4.1 结果类型

- `VERIFIED`：密码学/登记校验通过，证据满足阈值。
- `CANDIDATE`：视觉检测存在候选，但证据不足以确定。
- `NOT_DETECTED`：未检出；不得视为“文件没有水印”。
- `UNSUPPORTED`：不支持格式或检测路径。
- `FAILED`：处理错误。

检测结果应保存变换场景、页码/幻灯片/视频时间戳、算法版本、数据质量、提取候选与检测阈值。`confidence` 若提供必须注明是模型分数还是经标定的概率，不应冒充可信概率。

## 5. 最小数据模型

- `content`: id, title, media_type, revision, source_hash, created_at
- `distribution`: id, content_id, recipient_ref, bundle_id, issued_at, status, output_hash
- `watermark_issuance`: id, distribution_id, algorithm_id, version, payload_ref, key_version, embed_options, validation_json
- `job`: id, kind, input_ref, status, progress, error_code, created_at
- `detection`: id, sample_hash, media_type, status, candidates_json, evidence_ref, created_at
- `audit_event`: id, actor, action, object_ref, timestamp, integrity_tag

最小部署 PoC 可用 SQLite/JSONL；API 化以后可迁移 PostgreSQL + S3/MinIO；原文件/输出文件须权限隔离，并按策略保留/删除。日志不打印个人信息、令牌、原始密钥。

## 6. 推荐实现与部署

- Backend：Python 3.11+ / FastAPI；PoC 首先提供 Python CLI。
- PDF：PyMuPDF。
- PPTX：`zipfile` + `lxml` 做最小 OOXML 修补，`python-pptx` 协助检查。
- 图像：Pillow、OpenCV、Meta PixelSeal、Adobe TrustMark。
- 视频：FFmpeg、Meta VideoSeal；支持专用 GPU Worker，但不依赖 GPU 运行管理 API。
- 加密：成熟国密库，单独封装 `crypto_adapter`，禁止自行实现原语。
- 任务：本地 CLI → 后续 Celery/Redis 按性能需要添加。
- 文件存储：本地隔离目录 → S3/MinIO（按后续服务化计划）。
- 前端：V2 引入 Vue 3 管理台，初期只保留 REST API / CLI。
- 部署：Docker Compose；视频/GPU worker 与 API 进程隔离。

## 7. 安全、兼容性与伦理

- 每次分发不同 ID；避免在载荷中嵌入 PII；身份映射要权限控制并保留访问审计。
- 提取仅对有合法权限的资料进行操作；使用目的限内容溯源，不将推断性结果包装为责任认定。
- 不执行 PPTX 中任何宏、外部链接或嵌入脚本；处理 XML/ZIP 防压缩炸弹、路径穿越、实体扩展、恶意文件。
- 上传限制大小、页数、视频时长；隔离 FFmpeg/Office 转换进程；限制 CPU/GPU/内存。
- 模型权重不可默默下载到生产机；记录模型版本和校验和，私有化部署使用可复现离线包。
- PPTX 编辑应保留未修改元素和动画；输出校验与可编辑测试失败则回滚，不发布损坏文件。
- 检测与解码必须防止随机碰撞/伪造导致错误归属。

## 8. 主要风险与对策

| 风险 | 影响 | 应对 |
|---|---|---|
| PPT 编辑导致水印消失 | 原生 PPT 可以正常编辑但无法追溯 | 结构+视觉双层，定义可保留的编辑边界，实测 |
| 文字页缺少图像纹理 | 视觉模型难以可靠嵌入 | 文字微结构算法探索、必要时诚实降级为结构指纹 |
| 打印复印损伤 | 视觉水印解码失败 | 对齐/纠错/多处冗余与专项实验 |
| 视频重编码和片段截取 | 时间段水印丢失 | 多帧冗余、时序融合、短片段基准测试 |
| 对比多份个性化副本 | 可能消除差异 | 不承诺抗共谋，后续考虑专门抗合谋编码 |
| 二次 OCR 重制 | 水印被去除 | 内容指纹匹配只能定位母版，不能凭此确认接收者 |
| 错误溯源 | 影响个人权益 | 身份认证标签、独立复核、明确证据等级 |

## 9. V1 验收边界

V1 必须能把结构层身份写入/提取 PDF 和 PPTX，并对原生文件可靠验证；PPTX 仍可编辑。至少有一个图像与一个视频候选算法完成端到端实验，形成性能数据。

**V1 不预先承诺**可编辑 PPTX 经任意截图或打印照片仍可识别。只有在 `docs/02-validation-plan.md` 的真实数据集达到目标后，相关能力才可以标记为已支持。

## 10. 技术参考

- Meta VideoSeal / PixelSeal：https://github.com/facebookresearch/videoseal
- Adobe TrustMark：https://github.com/adobe/trustmark
- Microsoft Open XML / PresentationML：https://learn.microsoft.com/en-us/office/open-xml/presentation/overview
- PyMuPDF：https://pymupdf.readthedocs.io/en/latest/