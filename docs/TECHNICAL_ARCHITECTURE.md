# 技术架构与数据设计

## 1. 架构目标

以可审计、可替换、可逐步扩展为目标。先做单城只读信息服务，不先做复杂评分、全国爬虫或自动化举报。

## 2. 逻辑架构

```text
合法数据源
  ├─ 政府公开网页 / 开放数据
  ├─ 获授权数据
  └─ 用户主动提交的线索
        ↓
采集与来源登记
        ↓
原始材料归档（URL、时间、快照、哈希）
        ↓
解析与结构化（HTML / PDF / OCR）
        ↓
主体识别与实体消歧
        ↓
事件归一、冲突检测、证据分级
        ↓
检索 API / 地理索引 / 时间线
        ↓
Web / 小程序 / App / 内容生产工具
        ↓
纠错、申诉、更新与审计
```

## 3. 推荐技术栈（可替换）

个人 MVP 可选：
- 后端：Python FastAPI 或 Node.js。
- 数据库：PostgreSQL；小规模原型可用 SQLite。
- 搜索：PostgreSQL 全文检索，后续按需引入 OpenSearch/Meilisearch。
- 前端：Next.js / Vue。
- 任务调度：系统 Cron、Celery、RQ 或轻量队列。
- 文件存储：本地对象存储或合规云存储。
- OCR：PaddleOCR 等可本地部署方案。
- 地理能力：PostGIS 或地理编码服务（遵守服务条款）。
- 部署：Docker Compose 起步。

技术选择不是强制标准。优先采用开发者熟悉、能够长期维护的方案。

## 4. 核心数据对象

### source
- source_id
- publisher_name
- source_type
- canonical_url
- access_method
- license_or_terms
- checked_at
- update_frequency
- reliability_notes

### establishment
- establishment_id
- display_name
- legal_entity_name（仅在必要且合法时）
- address
- city / district
- geo_point（可选）
- identity_confidence
- identity_evidence

### event
- event_id
- establishment_id（允许为空或待确认）
- event_type
- event_date
- published_at
- summary
- official_status
- source_id
- source_record_id
- evidence_level
- valid_from / valid_to
- correction_status

### user_submission
- submission_id
- category
- user_statement
- submitted_at
- consent_state
- review_state
- retention_deadline

用户提交内容默认不公开。公开前需要必要审核、隐私处理和法律依据。

## 5. 证据状态设计

建议至少区分：
- OFFICIAL_PUBLISHED：官方公开发布。
- MEDIA_REPORTED：媒体报道，需保留原文出处。
- USER_REPORTED：用户陈述，未独立核实。
- MODEL_EXTRACTED：模型从来源材料抽取，待校验。
- CONFLICTED：来源之间存在冲突。
- OUTDATED：信息可能过期。
- CORRECTED：已更正，保留更正历史。
- UNKNOWN：无法确认。

不能把模型抽取状态自动升级为官方事实。模型置信度不是事实证明。

## 6. 采集与更新

- 为每个数据源登记合法性、访问方式、条款和频率。
- 优先使用官方 API、开放数据和明确允许的下载方式。
- 遵守 robots、访问频率、身份验证、版权和网站使用条款。
- 对每条原始记录保存采集时间、来源 URL、内容哈希和处理版本。
- 采用增量更新和变更检测，避免重复采集。
- 对页面结构变化、验证码、访问限制和来源下线设置告警，不绕过访问控制。

## 7. 实体消歧

同名门店不能仅靠名称合并。建议组合：
- 地址及门牌信息；
- 经营主体公开信息（仅在合法必要范围）；
- 品牌与分店关系；
- 原始公告中的主体名称；
- 时间与历史地址；
- 人工纠错结果。

匹配结果分为 VERIFIED / PARTIAL / UNKNOWN，低置信度不强行归并。

## 8. 安全与隐私

- 不收集与功能无关的个人信息。
- 用户举报材料可能包含手机号、面部、支付信息等敏感内容，应默认私密、最小化保存并设置删除机制。
- 对外展示前进行个人信息检测和必要脱敏。
- 管理员操作、纠错、删除和状态变化保留审计日志。
- 保护密钥、限制权限、定期备份并测试恢复。
- 不公开用户举报人的身份和联系方式。

## 9. 可观测性与质量指标

- 来源可访问率、更新延迟、解析成功率。
- 主体匹配准确率、误合并率、漏匹配率。
- 原始出处可回溯率。
- 冲突记录比例、过期记录比例。
- 纠错响应时间和更正完成率。
- 搜索无结果率、用户查询成功率。

测试数据与真实经营数据必须分开标识。不得将演示数据伪装成真实监管记录。
