# Agent OpenAPI 接入 MCP 指南

> 本文档面向通过 MCP 协议接入逐曜AI 的智能体开发者。MCP（Model Context Protocol）是 Agent OpenAPI 的**协议化接入方式**：
> 通过 `/mcp` 端点，把现有的 Agent OpenAPI 能力封装为 **65 个 MCP 工具**，智能体无需手拼 REST 请求，
> 直接用 `tools/call` 调用，配合 `get_workflow` 蓝图引导操作顺序。
>
> 走 HTTP REST 的调用方请查看《Agent OpenAPI 调用指南》。
> 版本：2026-09-06（MCP v1.3.2，65 工具；新增案例库语义匹配 case_match——embedding 余弦相似度自动选案例，存量案例首次匹配现场补索引）

---

## 一、接入方式（必读）

MCP 端点为公网 HTTPS 端点：

```
https://www.zhuyaoai.com/mcp/
```

> - 正式域名端点（Let's Encrypt 证书，客户端无需证书特殊处理），**必须带尾斜杠**。
> - 接入侧建议把端点设为环境变量（如 `ZHUYAO_MCP_URL`），换环境改变量，不碰硬编码。
> - 智能体经此端点直接接入，无需内网/代理。认证/授权/限流与 REST 一致，由系统统一保障。

---

## 二、协议说明

MCP Streamable HTTP，JSON-RPC 2.0。核心方法：

| 方法 | 用途 |
|------|------|
| `initialize` | 握手，返回服务信息 `zhuyaoai-agent v1.0.0`、协议版本 |
| `tools/list` | 列出全部工具（含完整 inputSchema） |
| `tools/call` | 调用工具（`name` + `arguments`） |

调用约定：

- 两种鉴权方式二选一：①每个工具参数带 `apiKey`（Agent API Key，形如 `agk_` 开头）；②**连接级注入**——连接 MCP 端点时带 `X-Agent-API-Key` 请求头（WorkBuddy 连接器等自填 Token 模式用这种），此后工具参数 apiKey 可省略。两条路径最终都转发为 `X-Agent-API-Key` 头。
- 需要指定公司的写操作必须带 `companyId`（须在 Key 授权范围内，越权 403）。
- 工具返回统一结构 `content[{ type: 'text', text }]`，业务结果在 `text` 的 JSON 字符串里。
- **授权/scope/限流完全复用后端 guard 链**：scope 不足返回 403，超限返回 429。
- **新能力默认全量开放（2026-09-06 产品决策）**：新增能力的 scope（如案例库 `case:read`/`case:write`/`case:review`）自动对所有有效 Key 生效，**无需重建或补配 Key**；GET /me 返回的 scopes 已含默认授权项。唯一例外：scopes 为空数组的异常 Key 仍 fail-closed 拒绝一切。

---

## 三、65 个工具清单

### 编排工具（先调 get_workflow 确定顺序）

| 工具 | 说明 |
|------|------|
| `get_workflow` | 返回标准工作流 SOP（步骤：工具/说明/必填参数/依赖）。**不确定操作顺序时最先调用** |
| `task_status` | 查询异步任务终态（pending/running/success/failed） |
| `task_list` | 分页查询当前 Key 可见的异步任务列表 |

### 自查询（BOOTSTRAP 自检，无需 scope）

| 工具 | 说明 |
|------|------|
| `get_me` | 返回当前 Key 自描述（keyId/scopes/companyIds/isActive/expiresAt）。**BOOTSTRAP 第一步：确认 Key 有效 + 已授权权限/公司** |
| `get_companies` | 返回当前 Key 授权公司列表（id/name/shortName/industry + allCompanies 通配标记）。**确认 companyId 的唯一正路，写操作前必调，禁止猜/硬编码门牌号** |

### 知识库 / RAG

| 工具 | 说明 |
|------|------|
| `knowledge_fill` | 写入知识库（带 moduleCode + companyId = 更新式写入）。**写入后自动生成 embedding** |
| `knowledge_query` | 查询知识库（支持按 category / search 搜索，含 content 字段） |
| `knowledge_delete` | 删除知识库条目（软删） |
| `rag_query` | RAG 语义检索（按 query 命中知识库，自动过滤空内容模板） |

### 文章

| 工具 | 说明 |
|------|------|
| `article_create` / `article_get` / `article_list` / `article_update` / `article_delete` | 文章 CRUD（article_create 注意：`tags`=逗号字符串非数组；无 `category` 字段，挂官网栏目用 `categorySlug`+`websiteId`） |
| `article_publish` | 发布文章到官网（需 websiteId）。**草稿/发布失败状态自动提升为已发布再同步**，生成中/审核不通过仍拒绝 |
| `article_generate` | 批量 AI 生成文章（异步，返回 taskId） |
| `article_generate_status` | 查询生成任务状态 |

### 官网 / 博客

| 工具 | 说明 |
|------|------|
| `website_sync` | 同步官网内容（知识库模块 + 托管站） |
| `blog_upload` | 上传博客内容（需逐曜授权开通） |

### GEO 监测

| 工具 | 说明 |
|------|------|
| `monitor_create_task` | 创建 GEO 监测任务（只创建不执行） |
| `monitor_execute` | 触发监测采集 |
| `monitor_list` / `monitor_get` | 任务列表 / 详情 |
| `monitor_analysis_list` / `monitor_analysis_get` | 监测分析报告列表 / 详情 |
| `monitor_results` | 读取监测任务原始采集结果（分页） |

### 图集

| 工具 | 说明 |
|------|------|
| `gallery_list` / `gallery_get` | 图集列表 / 详情 |
| `gallery_match` | 按关键词评分匹配选图（写文章配图用） |
| `gallery_category_list` / `gallery_category_create` | 图集分类树查询 / 创建分类（上传前先查，无合适再建） |
| `gallery_upload` | 上传图片/视频到图集（base64 文件，multipart 转发，复用 sharp 管线与配额）。**上传前先查/建分类，name 与 tags 必写**（配图按 tags→name→分类→AI描述评分） |

### 案例库

| 工具 | 说明 |
|------|------|
| `case_list` | 分页查询公司案例素材（case:read，分页 ≤100；industry/tag/keyword 过滤，关键词扫标题/摘要/客户背景/痛点）。**写文章引用案例先查列表拿 id，再填进 article_generate 的 caseIds** |
| `case_match` | 按关键词语义匹配案例（case:read，embedding 余弦相似度，minScore 缺省 0.15）。**写文章引用案例优先用本工具自动选**；存量案例首次匹配现场补索引可能略慢 |
| `case_get` | 查询案例完整详情（case:read，含实施过程与量化指标 keyMetrics；越权 404） |
| `case_create` | 创建案例（case:write，幂等；五要素必填：title/clientProfile/painPoints/solution/outcome）。images 可传图集图片 URL（先 gallery_match 选图）；**published 缺省 false 不上站**；直发需 case:review scope（未持有写 true 返回 403），发布成功自动触发官网同步 |
| `case_update` | 更新案例（case:write，幂等；仅更新传入项，无需 companyId）。直写 published=true 需 case:review scope；常用：补 images/keyMetrics、改标题摘要 |
| `case_delete` | 删除案例（case:write；不可恢复，级联清理与文章任务的关联，删除前确认 id） |

### DSH 沙箱

| 工具 | 说明 |
|------|------|
| `dsh_task_submit` | 提交 DSH 代码执行/分析任务（异步，返回 taskId） |
| `dsh_task_get` | 查询 DSH 任务状态与结果 |
| `dsh_task_cancel` | 取消 DSH 任务 |

### 蒸馏词

| 工具 | 说明 |
|------|------|
| `keywords_group_create` | 创建蒸馏词组（一键表格导入：可同时传 `keywords` 数组建组+导词，source 自动标记 `agent_api`；coreWord 缺省取首词） |
| `keywords_group_import` | 批量导入关键词到词组（source 自动标记 `agent_api`）。**groupId 先用 keywords_group_list 查** |
| `keywords_group_list` / `keywords_group_get` | 蒸馏词组列表（分页 ≤100，含 `keywordCount`）/ 分组详情（含词根维度与组内关键词分页），均只读 |

### 视频

| 工具 | 说明 |
|------|------|
| `video_list` / `video_templates` / `video_get` | 视频项目列表 / 公司模板列表（含 `isCompanyDefault` 默认标记，公共模板 companyId=0 不返回）/ 视频详情（均只读，不落任务） |
| `video_create` | 创建视频项目（写操作，返回 taskId）。**不传 templateId 自动套用公司「默认模板」配置（推荐）**；传 templateId 显式引用公司模板（仅限自身公司模板，系统内置模板缺品牌素材不可引用会 403），`videoConfig`/`voiceConfig` 可叠加覆盖模板项；图集轮播节奏 `videoConfig.background.switchMode`：缺省按口播段切换（图用完钉末图）；`"cycle"` 图少段多时循环轮播；`"duration"` 按 media[].duration 独立切换 |
| `video_analyze_article` | 从文章 AI 生成口播稿。short=同步返回（每段已拆到 ≤3.75s）；**long=异步任务**，返回 taskId 用 task_status 轮询至 success 取 result.script（两三分钟口播 50-90 段，每段 ≤3.75s）；`localize` 缺省 false=忠实原文不注入地域/公司词；结果可作 video_create 的 `script` |
| `video_submit_build` | 提交视频出片（异步任务，返回 taskId + videoId；出片需数分钟，用 video_status 或 task_status 轮询） |
| `video_status` | 查询视频构建状态与进度（scripting/tts/rendering/done/failed） |
| `video_play_url` | 视频播放签名 URL（对象存储 24h 有效，降级本地 Nginx 路径；仅 done 后可访问） |
> 视频能力 = 口播短视频（文字 → TTS 人声 + 字幕 + 背景 + BGM 渲染 MP4），非文生视频。完整链路 = video_create →（可选 video_analyze_article）→ video_submit_build → video_status 轮询 → video_play_url 取播放签名 URL 交付用户。

### SOP 运营进度

| 工具 | 说明 |
|------|------|
| `sop_step_list` | 查询 SOP 模板步骤（只读，scope=sop:read）。**companyId 可选**：不传返回全局默认模板；传返回该公司生效模板（全局打底 + 公司定制合并，每步带 `source: global/custom`）。steps[].id 即 stepId |
| `sop_progress_update` | 标记某客户某 SOP 步骤进度（scope=sop:write，幂等）。`companyId`/`stepId`/`status` 必填，companyId 归属校验（防越权）；status ∈ pending/doing/done/skipped，重复调用幂等覆盖 |

> SOP 支持**公司级定制**：每账号下每公司可独立改模板（全局默认模板打底 + 公司专属覆盖）。stepId 要取**该公司生效模板**里的 id——先调 `sop_step_list`（带 companyId）拿到该公司生效步骤再标记。

### 产品库

| 工具 | 说明 |
|------|------|
| `product_upload` | 上传产品图片组建产品草稿（scope=product:write，multipart files[] 多图）。首图=封面、余图=详情图；**categoryId 先用 product_category_list 查（无合适则建）**，name 必写、tags/summary 建议填写；published 强制 false 不上站 |
| `product_category_list` / `product_category_create` | 产品分类分页查询 / 创建分类（重名 400 当已存在处理） |
| `product_list` | 分页查询产品草稿列表（scope=product:read，companyId 必填归属校验） |
| `product_get` | 查询单个产品草稿详情（scope=product:read，越权 404） |

### 资料下载

| 工具 | 说明 |
|------|------|
| `download_upload` | 上传单个文档建资料草稿（scope=download:write，multipart file 单文件）。category 为自由文本（缺省"资料"），**先 download_list 查现有分类名并复用**；title/description 必写表意清晰；published 强制 false 不上站 |
| `download_list` | 分页查询资料草稿列表（scope=download:read，companyId 必填归属校验） |
| `download_get` | 查询单个资料草稿详情（scope=download:read，越权 404） |

> **分派指引**：单图素材 → `gallery_upload`；文档 → `download_upload`；产品图组 → `product_upload` 一次传 files[] 多图（首图=封面、余图=详情图）。

---

## 四、get_workflow 内置工作流

内置 5 个蓝图 SOP（`get_workflow` 返回步骤清单，智能体按步骤执行）：

1. **article_publish_flow 文章发布全流程**：knowledge_fill → article_create → article_publish →（可选）monitor_create_task
2. **geo_monitor_flow GEO 监测流程**：monitor_create_task → monitor_execute → task_status（等 success）→ monitor_get
3. **website_sync_flow 官网内容流程**：website_sync →（可选）blog_upload
4. **video_flow 视频制作全流程**：video_templates（查公司模板，确认默认模板 `isCompanyDefault=true`）→ video_create（不传 templateId 自动套默认模板，或传 templateId 固定风格）→（可选 video_analyze_article 从文章生成口播稿）→ video_submit_build → video_status 轮询至 done → video_play_url 取播放签名 URL 交付用户
5. **asset_reference_flow 公司资产引用流程**：gallery_match 选图 + case_match/case_list 选案例 → article_generate（带 galleryIds/caseIds 生成）；（可选）case_create 把对话中沉淀的客户案例入库

> 通用顺序提示：写操作返回 `taskId` 后，用 `task_status` 轮询至 `success` 再进行下一步；不确定顺序先调 `get_workflow`。

---

## 五、幂等

以下 23 个写工具支持幂等（重复调用同一 `idempotencyKey` 时后端幂等表去重，返回 `idempotentReplay: true`）：

- 
- website_sync、blog_upload、monitor_create_task、monitor_execute
- gallery_upload、gallery_category_create、dsh_task_submit、dsh_task_cancel、keywords_group_create、keywords_group_import、product_category_create
- video_create、video_analyze_article、video_submit_build
- case_create、case_update

**调用方式**：工具参数里**可选传 `idempotencyKey`**（自定义字符串，如 `order-20260822-01`）。
同一逻辑操作重试时传同一个 key，即可避免重复创建。不传则每次都是新操作。

---

## 六、调用示例（curl JSON-RPC）

```bash
# 1. 握手（{MCP_URL} 替换为实际端点）
curl -X POST {MCP_URL} \
  -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"my-agent","version":"1.0"}}}'

# 2. 列出工具
curl -X POST {MCP_URL} \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'

# 3. 获取工作流蓝图
curl -X POST {MCP_URL} \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_workflow","arguments":{}}}'

# 4. RAG 检索（apiKey / companyId 为真实值，{companyId} 经 GET /companies 确认）
curl -X POST {MCP_URL} \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"rag_query","arguments":{"apiKey":"agk_xxx","companyId":{companyId},"query":"营业时间","topK":5}}}'
```

> 智能体接入建议直接用 MCP SDK（如 `@modelcontextprotocol/sdk`）连接，无需手写 JSON-RPC。

---

## 七、与 REST 的关系与选择建议

| 维度 | MCP（/mcp） | REST（/api/open/agent） |
|------|------------|------------------------|
| 接口形态 | 65 个语义化工具 + 蓝图 | 原生 HTTP 端点 |
| 上手成本 | 低（get_workflow 引导） | 中（需读接口清单） |
| 认证 | 工具参数 `apiKey` | `X-Agent-API-Key` 头 |
| 适用 | 自主编排的智能体 | 简单脚本 / 对拍调试 |

两者底层同一套 Agent OpenAPI 安全边界（认证/scope/限流），能力等价，按调用方形态选择。

---

## 八、故障排查

| 现象 | 处理 |
|------|------|
| 连接失败 / 拒绝 | 确认网络可达 `https://www.zhuyaoai.com/mcp/`（正式域名证书），检查防火墙/代理拦截 |
| 403 | apiKey 无效 / 公司不在授权范围 / scope 不足 |
| 429 | 触发限流（每 Key 每分钟 60 次），稍后重试 |
| 400 上传文件缺失 | gallery_upload 的 fileBase64 解码为空或未传 fileName，检查 base64 有效性 |
| 结果为空 / 命中旧模板 | 知识库内容为空模板（已自动过滤）；确认内容已写入且非空 |
| tools/call 返回 isError | 业务错误在 content[0].text 里，按文本排查 |

---
