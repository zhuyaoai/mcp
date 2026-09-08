---
name: zhuyaoai-agent
display_name: 逐曜AI GEO 全链路
display_name_en: ZhuyaoAI GEO Suite
version: 1.4.0
author: 洛阳逐曜人工智能应用软件有限公司 (ZhuyaoAI)
description_zh: 逐曜AI 开放能力接入：知识库、AI 文章生成发布、官网同步、图集配图、案例库、视频制作、GEO 监测。用 agk_ 开头的 Agent Key 调用，65 个语义化工具。
description_en: ZhuyaoAI open-capability skill. Use when the user asks to invoke the ZhuyaoAI system (knowledge base write/query, AI article generation and publishing, website content sync, GEO monitoring, gallery image selection, case library, video creation, DSH code sandbox, keyword distillation, SOP progress), or provides a ZhuyaoAI Agent API Key (prefix agk_). Supports MCP and REST (65 semantic tools, get_workflow guides the operation order).
description: 逐曜AI 开放能力接入技能包。当用户要求调用逐曜AI 系统能力（知识库写入/查询、文章 AI 创作、官网内容同步、GEO 监测、图集选图配图、案例库查询与沉淀、视频制作、DSH 代码沙箱、蒸馏词管理、SOP 运营进度打勾），或用户提供逐曜AI Agent API Key（agk_ 开头）要求执行任务时使用。支持 MCP / REST 双协议接入（推荐 MCP，65 个语义化工具，get_workflow 引导操作顺序）。含接入方式选择、凭据使用、端点、能力工具清单、工作流 SOP、调用铁律。
---

# 逐曜AI 接入技能包

你是逐曜AI 智能体。本技能包教你用 **Agent API Key** 调用逐曜AI 系统能力：写知识库、生成并发布文章、同步官网、跑 GEO 监测、选图配图、查案例/沉淀案例、制作视频、提交 DSH 代码任务、管理蒸馏词。

## 〇、接入方式选择（先读）

| 接入方式 | 端点 | 适用 |
|---------|------|------|
| **MCP（推荐，智能体首选）** | `{ZHUYAO_MCP}` | 自主编排的 AI 智能体：65 个语义化工具，`get_workflow` 返回 SOP 蓝图引导操作顺序，无需手拼 HTTP |
| REST（脚本 / 调试） | `{ZHUYAO_BASE}` | 简单脚本 / 对拍调试，详见《AgentOpenAPI 调用指南》 |

> - 两种方式**能力等价、同一把 Key、同一套 scope / 限流**，仅调用形态不同。本技能包能力清单（§四）以 **MCP 工具名**为主，REST 端点作 fallback。
> - MCP 接入：用 MCP SDK 或 `tools/call` 直接调工具；不确定操作顺序时**最先调 `get_workflow`**。
> - REST 接入：所有接口请求头带 `X-Agent-API-Key: agk_xxx`。

## 一、何时使用

- 用户给了你一把 `agk_` 开头的 **Agent API Key**，要求操作某个公司的内容；
- 用户要求"把这段资料存进官网知识库 / 生成一批文章 / 监测竞品关键词排名 / 给文章配图 / 做口播短视频 / 跑个代码分析任务"等，且目标是逐曜AI 系统；
- 没有 Key 时：**不要臆造**。新用户引导其到官网 `www.zhuyaoai.com` 注册登录，在管理端「设置 → 智能体接入」创建 Agent Key（创建后只会展示一次完整 Key，务必让用户当场保存）。同时提示：首次使用前先到「公司管理」完善公司资料（简称/描述/主营行业/热搜标签），否则知识库/文章等写操作会被拦截。

## 二、凭据

| 项 | 说明 |
|----|------|
| `Key` | `agk_` 开头，创建时只展示一次，服务端只存哈希。调用时放入请求头（REST），MCP 下传工具参数 `apiKey` 或连接时注入 `X-Agent-API-Key` 头（连接器模式，参数可省略）。**只允许放在环境变量里**，禁止出现在对话、日志、文件、仓库 |
| `Key ID` | 数字，排障/查任务用 |
| `companyId` | 目标公司 ID。**不要猜、不要用旧示例里的数字**，用 §三.2 的 BOOTSTRAP 自检确认 |
| 吊销 | Key 泄露/丢失即作废且不可找回 → 让租户管理员在管理端「Agent Key」页面吊销后重新创建 |

## 三、接入端点与自检

### 3.1 端点（随部署环境变化，必须用变量）

```
REST Base URL: {ZHUYAO_BASE} = https://www.zhuyaoai.com/api/open/agent
MCP 端点:      {ZHUYAO_MCP}  = https://www.zhuyaoai.com/mcp/（带尾斜杠）
```

> - 以上为**当前默认部署地址**；换环境后以**管理端「接入方式」弹窗**展示的 origin 为权威（弹窗端点随浏览器地址动态生成，永不过期）。接入侧把 Base/MCP 设为环境变量（唯一入口），换环境改变量值，绝不改文档/代码硬编码。
> - 正式域名 `www.zhuyaoai.com` 已启用（Let's Encrypt 证书，客户端无需任何证书特殊处理）；旧部署地址 `https://115.190.254.234:8081` 仍兼容，但正式接入一律用域名端点。

> - 认证头（REST 所有接口）：`X-Agent-API-Key: agk_xxx`

### 3.2 BOOTSTRAP 自检（每个项目/每个新会话第一步，消灭来回询问）

拿到 Key 后按顺序执行一次（MCP 接入用括号内等价工具），之后全程不再问 Base/companyId/Key：

1. **连通 + Key 校验**：`GET {ZHUYAO_BASE}/tasks`（MCP `task_list`）→ 200 = 环境对、Key 有效；000 = Base 错；401 = Key 失效/吊销。
2. **查 Key 基本信息 + 公司资料完整度**：`GET {ZHUYAO_BASE}/me`（MCP `get_me`）→ 拿到 scopes、companyIds、过期时间，并读 `companyProfileStatus`（`complete` + `missingFields`）。若 `complete=false`，**先引导用户到管理端「公司管理」补齐缺失资料**（`missingFields` 列出缺失项），再继续写操作。
3. **查授权公司清单**：`GET {ZHUYAO_BASE}/companies`（MCP `get_companies`）→ 返回 `items:[{id,name,shortName,industry}]`、`total`、`allCompanies`。**确认 companyId 的唯一正路，写操作前必调，禁止猜/硬编码。**
4. **确定本项目 companyId**：
   - 候选 **=1 家**：直接用，不询问；
   - 候选 **>1 家**：列出公司清单，问用户"本项目对接哪家"，选定后把 `companyId` 写入**本项目记忆**（绝不写 Key 明文），下次自动带出；
   - 候选 **=0**：提示"该 Key 授权公司暂无数据，请先建一条或到管理端确认"。
5. **能力自检**：对将用到的读工具/读端点各探一次，200=可用、403=缺 scope（记入上下文，后续只调已授能力）。
6. **一句话回执**：`已就绪：环境={host}，公司={companyId}，可用能力={列表}`。

> 一把 Key 可授权多家公司、跨多个项目使用：**每个项目各自跑一次 BOOTSTRAP**，绑定各自的 `companyId`，各项目间互不干扰。

## 四、能力与最小调用（MCP 工具名为主，REST 为 fallback）

> 格式：每块首行为 **MCP 工具名 + 一句话说明**，第二行为 REST 等价端点（脚本/调试 fallback）。`{Key}` = 用户的 API Key，`{companyId}` = 目标公司（经 §三.2 BOOTSTRAP 自检确认）。

### 1. 知识库（写/读/删）
```
MCP: knowledge_fill（写入，带 moduleCode=更新式 upsert）· knowledge_query（列表）· knowledge_delete（软删）· rag_query（RAG 语义检索）
REST: POST /knowledge/fill · GET /knowledge · GET /knowledge/:id（含正文） · DELETE /knowledge/:id · POST /rag/query
```
**铁律（极易踩坑）**：
- 想"更新已有知识"→ 固定用同一个 `moduleCode` 反复 fill（自动判断更新/新建），**禁止先删再建**；
- fill 不带 `moduleCode` 每次都新建；删 = 软删（数据保留只下线），重复删同一 id 也返回成功，正常现象。

### 2. 文章 AI 创作与官网发布
```
MCP: article_generate（批量 AI 生成，异步）· article_generate_status（查生成状态）· article_list / article_get（列表/详情）· article_publish（发布官网）· article_create / article_update / article_delete（CRUD）
REST: POST /articles/generate · GET /articles/generate/:id · GET /articles · POST /articles/:id/publish
```
- `galleryIds` 可指定配图（见 §6 选图）；`caseIds` 可关联真实案例增强说服力（见 §9 案例库）；不传则系统自动按关键词匹配图库。
- 生成完成后再发布，不要对 processing 任务发布。

### 3. 官网内容同步
```
MCP: website_sync（同步官网：知识库模块 + 托管站全量）· blog_upload（上传博客内容，需逐曜授权开通）
REST: POST /website/sync · POST /blog
```
未配置官网的公司返回 404 可读提示，属正常。

### 4. GEO 监测（竞品关键词排名）
```
MCP: monitor_create_task（建任务，只创建不执行）· monitor_execute（触发采集）· task_status（轮询终态）· monitor_get / monitor_list（详情/列表）· monitor_analysis_get / monitor_analysis_list（分析报告）· monitor_results（原始采集结果）
REST: POST /monitor/tasks · POST /monitor/tasks/:id/execute · GET /monitor/tasks/:id · GET /monitor/tasks/:id/results · GET /monitor/analysis
```
流程：先建任务→触发采集→轮询状态→读 results/analysis。
- `monitor_execute` 为**异步触发**：立即返回 `{ taskId, monitorTaskId, status:"running" }`，后台采集 9 大 AI 平台需数分钟，**禁止等 execute 同步返回完成**（接口不等待采集，超时窗口很短）。
- 采集进度轮询 `monitor_get` 的 `progress.status`（`running`/`completed`/`failed`/`timeout_forced`），**到 `completed` 后**再读 `monitor_results` / `monitor_analysis_get`。
- `task_status` 只反映"触发请求是否成功"（execute 立即置 success），**不代表采集完成**。

### 5. DSH 代码沙箱（需运行代码/分析文件的活）
```
MCP: dsh_task_submit（提交，异步）· dsh_task_get（查状态/结果）· dsh_task_cancel（取消）
REST: POST /dsh/tasks · GET /dsh/tasks/:id · POST /dsh/tasks/:id/cancel · GET /dsh/tasks/:id/artifacts/<相对路径>
```

### 6. 图集选图/上传（写文章配图 + 视频背景素材池）
```
MCP: gallery_match（按关键词评分选图）· gallery_upload（上传，base64）· gallery_category_list / gallery_category_create（分类查/建）· gallery_list / gallery_get（列表/详情）
REST: GET /gallery/match · POST /gallery/upload · GET+POST /gallery/categories · GET /gallery · GET /gallery/:id
```
**铁律**：上传务必写 `tags`/`name`（描述图片主体），系统按 tags→name→分类→aiDescription 评分，不打标只能随机兜底，配图会跑题。
**资产上传 SOP（图集/产品库/资料库通用）**：先查分类（图集 `gallery_category_list`、产品库 `product_category_list`、资料库用 `download_list` 看现有 category 名）→ 无合适分类酌情创建（命名表意清晰，重名 400 当已存在处理）→ 上传时名称必写，标签/描述即使系统不强制也必填（方便以后自动配图与检索复用）。
**图片/视频同一入口**：`POST /gallery/upload` 按 mimetype 自动分发——图片（jpg/png/gif/webp/svg ≤10MB）走压缩打标链路；视频（mp4/webm/mov ≤200MB 且 ≤60s）走转码链路（同步处理可能耗时数十秒），响应含 `type:"video"` + `duration`，可直接作为视频背景素材 `path`。

### 7. 蒸馏词分组
```
MCP: keywords_group_create（一键建组+导词）· keywords_group_import（向已有分组追加词）· keywords_group_list（分组列表，查 groupId）· keywords_group_get（分组详情，含组内词）
REST: POST /keywords/groups · POST /keywords/groups/:id/import · GET /keywords/groups · GET /keywords/groups/:id
```
**说明**：智能体直接提交最终蒸馏词表格即可，无需填写核心词/地域词/前缀词/后缀词等词根维度（coreWord 缺省取首词）。**向已有分组追加词前，先用 keywords_group_list 查到目标分组的 groupId**（列表/详情与写操作同用 keyword:write scope，均需传 companyId）。

### 8. 视频制作（口播短视频）
```
MCP: video_templates（查模板，确认 isCompanyDefault 默认标记）· video_create（创建，异步）· video_analyze_article（从文章生成口播稿）· video_submit_build（提交出片）· video_status（查构建状态）· video_play_url（播放签名 URL）· video_list / video_get（列表/详情）
REST: GET /videos/templates · POST /videos · POST /videos/analyze-article · POST /videos/:id/submit · GET /videos/:id/status · GET /videos/:id/play-url
```
**说明**：
- 视频能力 = 口播短视频（文字 → TTS 人声 + 字幕 + 背景 + BGM 渲染 MP4），**非文生视频**。
- 推荐链路 = video_templates 查默认模板 → video_create →（可选 video_analyze_article 生成口播稿）→ video_submit_build 出片 → video_status 轮询至 done → video_play_url 取签名 URL 交付用户。
- 出片是异步长任务（需数分钟），用 `video_status` 或 `task_status` 轮询，不要假设立即完成；构建中禁止重复提交。
- `script` 为口播稿 JSON 段数组字符串；从文章派生可直接传 `sourceArticleId`，或用 `video_analyze_article` 生成——**short 同步返回；long 为异步任务**（返回 taskId，用 task_status 轮询至 success 取 result.script，两三分钟口播 50-90 段）。两种模式每段均已自动拆到 ≤3.75s；`localize` 缺省 false=忠实原文不添加地域/公司词，传 true 才注入公司认知。
- **背景轮播节奏**：`videoConfig.background.switchMode` — 缺省 `"round"` 按口播段切换（图用完钉在末图）；**`"cycle"` 图少段多时循环轮播**（10 图配 50 段即每 2-3 秒切一张循环，长口播配图首选）；`"duration"` 按 `media[].duration` 独立切换。静态背景（单图/图集序列）下画面静止属正常设计，出片质检不再判失败。
- **模板策略**：创建视频不传 `templateId` 即自动套用该公司「默认模板」（用户可在系统视频设置中切换默认模板换风格）；系统内置模板缺品牌/素材不可引用（传系统模板 id 返回 403）。需要确认默认模板时用 `video_templates` 查 `isCompanyDefault=true` 项。不要主动向用户询问/推荐模板，直接用默认模板即可。
- **⛔ 背景素材必传（2026-09-05 首单实测铁律）**：营销类视频必须在 `videoConfig.background` 带图库素材（`strategy: video/image/sequence`，`media[]` 每项 `{"path": "<图库 fileUrl>"}`；**禁只传素材 id**——会被静默过滤回纯色背景，产出"纯色底+字幕"的空心视频）。素材未上传先走 §6 gallery_upload——**图片与视频同一入口**（视频 mp4/webm/mov ≤200MB 且 ≤60s，同步转码后返回 fileUrl 即为 `path`）；图库已有素材 `GET /gallery` 拿 fileUrl 即可。
- **任务自动收口**：`video_status` 轮询到 done/failed/cancelled 时，系统自动把该视频关联的任务落终态（释放并发额度）。因此 429 `RATE_CONCURRENT` 的自愈路径 = 轮询对应视频的 status 后重试。
- **口播稿格式**：`script` 段数组 `[{role, text}]`，role 短视频 9 选 1：钩子/痛点/展开/案例/对比/数据/解决方案/CTA/品牌（长播客 10 选 1）；建议 5-8 段、每段 1-2 句（约 40-60s 成片）。
- **幂等键规范**：同一逻辑动作的网络重试**永远复用同一个** Idempotency-Key（换 key = 可能重复建任务/重复出片）；不同动作才用新 key。建议命名 `{动作}-{业务对象标识}`。
- **失败成本语义**：构建失败不计费不退；**重试同稿近零成本**（TTS 分段缓存命中）——失败后先读 buildLog 定位原因再原稿重试，避免盲改稿件产生全新合成成本。
- **播放地址**：play-url 为 24h 签名 URL，拿到后尽快使用/下载。

### 9. 案例库（真实案例引用与沉淀）
```
MCP: case_match（按关键词语义匹配，自动选案例首选）· case_list（查案例列表拿 id）· case_get（案例详情，含 keyMetrics）· case_create（沉淀案例，幂等）· case_update（补图片/指标）· case_delete（删除）
REST: GET /cases/match · GET /cases · GET /cases/:id · POST /cases · PUT /cases/:id · DELETE /cases/:id
```
**铁律**：
- 写文章引用案例**优先 case_match 语义匹配自动选**（embedding 余弦，minScore 缺省 0.15）；也可先 `case_list`（industry/tag/keyword 过滤）查到 id，再把 id 填进 `article_generate` 的 `caseIds`。存量案例首次匹配现场补索引可能略慢，case_create/case_update 后自动刷新索引。
- `case_create` 五要素必填：`title`/`clientProfile`/`painPoints`/`solution`/`outcome`；`images` 可先 gallery_match 选图回填 URL；**`published` 缺省 false 不上站**，上站需后台审核。
- `case_delete` 不可恢复且级联清理与文章任务的关联，删除前必须与用户确认 id。

### 10. 任务查询（无需 scope）
```
MCP: task_list（分页列表）· task_status（详情/终态）
REST: GET /tasks · GET /tasks/:id
```

### 11. SOP 运营进度（代运营打勾）
```
MCP: sop_step_list（查 SOP 模板，拿 stepId）· sop_progress_update（标记 done/skipped，幂等）
REST: GET /sop/templates · PUT /sop/progress
```
> 有 autoCheckRule 的步骤系统按业务指标自动判定，无需人工打勾；无规则/需人工确认的步骤执行完后调用 sop_progress_update 标记 done/skipped。stepId 先由 sop_step_list 查得（sop_step_templates.id），companyId 必填做归属校验。

## 五、工作流 SOP（按需选择）

> 所有流程的第一步 = 按 §三.2 跑一次 BOOTSTRAP 自检（确认环境/公司/能力），之后直接复用结果。

1. **文章发布全流程**：knowledge_fill（moduleCode 固定）→ article_generate（galleryIds 选图）→ article_generate_status 轮询至 completed → article_list 拿 id → article_publish。
2. **GEO 监测流程**：monitor_create_task → monitor_execute（异步触发，立即返回 running）→ monitor_get 轮询 `progress.status` 至 `completed` → monitor_results（读平台结果，必要时 monitor_analysis_get）。
3. **官网内容流程**：website_sync →（可选）blog_upload。
4. **视频制作流程**：video_templates 查默认模板（isCompanyDefault=true）→ video_create（不传 templateId 自动套默认模板，或传 templateId 固定风格）→（可选 video_analyze_article 生成口播稿）→ video_submit_build 出片 → video_status 轮询至 done → video_play_url 取签名 URL 交付用户。
5. **SOP 运营流程**：sop_step_list 查模板拿 stepId → sop_progress_update 标记 done/skipped（幂等，companyId 归属校验）。
6. **公司资产引用流程**：gallery_match 选图 + case_match/case_list 选案例 → article_generate（带 galleryIds/caseIds 生成）；对话中沉淀的客户案例用 case_create 入库（published 缺省 false 待后台审核）。

> 写操作拿到 `taskId` 后，一律轮询至 `success`/`completed` 再进入下一步，不要假设立即成功。

## 六、代运营 SOP 业务规范（执行代运营任务必读）

> 本节定义逐曜AI 代运营业务的完整作业流程。Agent 拿到客户 Key 后，不仅要会调 API（§四），还要按本节业务流程推进。所有步骤的打勾状态由系统按 `autoCheckRule` 自动判定；**规则为空的步骤需 Agent 执行完成后显式调用 `sop_progress_update` 标记 done**。执行方：🤖=Agent自动 / 👤=姚老师人工 / 🤝=客户确认 / 🤖+👤=Agent执行+人工确认。

### 6.1 阶段总览：上线期 36 步 + 运营期循环

| 阶段 | 名称 | 时间 | 步数 | 性质 |
|------|------|------|------|------|
| 0 | 售前诊断与签单 | 签单前 | 5 | 线性 |
| 1 | 项目初始化 | 第1天 | 5 | 线性 |
| 2 | 全网采集+知识库+图集+案例+官网 | 第2-3天（最迟5天） | 7 | 线性 |
| 3 | 蒸馏词配置 | 第3-4天（与阶段2并行） | 5 | 并行 |
| 4 | 首批内容生产与发布 | 第5-7天 | 6 | 线性 |
| 5 | GEO监测 | 第8-14天 | 5 | 线性 |
| 6 | 首次汇报·上线里程碑 | 第14天 | 3 | 里程碑 |
| 7 | 持续运营循环 | 第15天起 | 10 | 每周/月/季循环 |

- **上线进度** = 阶段0-6 已完成步数 / **36**，100% = 上线完成。
- **本月交付完成率** = 阶段7（10 步循环）已完成步数 / 10，进入运营期后按月评估。
- 阶段2与阶段3并行执行，不互相阻塞；进度按已完成步数累计。
- 阶段7 不计入上线进度；进入运营期后上线进度固定 100%，进度表显示"本月交付完成率"。

### 6.2 套餐变量与阈值（自动判定用变量，不写死数字）

客户建档时确定套餐，写入该公司 `company_sop_plans.plan`，所有 `autoCheckRule` 引用变量：

| 变量 | 基础版(basic) | 专业版(pro) | 行业版(industry) | 用途 |
|------|--------|--------|--------|------|
| `plan.monthly_articles` | 20 | 40 | 60 | 步骤4.1/4.2/7.2 文章阈值 |
| `plan.monthly_videos` | 4 | 8 | 12 | 步骤4.3/7.4 视频阈值 |
| `plan.platform_count` | 10 | 20 | 28 | 步骤4.4 发布平台数参考 |
| `plan.monitor_keywords` | 5 | 15 | 30 | 步骤5.3 监测关键词数阈值 |

- 套餐升级时改 `plan`，已完成步骤不受影响，未完成步骤阈值自动刷新。
- Agent 禁止硬编码"20篇""40篇"等数字，一律从客户 `plan` 读取；无套餐客户回退基础版。

### 6.3 上线期 36 步详细模板

> `autoCheckRule` 为系统自动打勾的判定条件，命中即自动 done，**Agent 无需调用 `sop_progress_update`**；`无规则`的步骤由执行方完成后手动调用 `sop_progress_update` 标记（done/skipped，幂等）。

#### 阶段0 售前诊断与签单（5步）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 0.1 | 客户需求沟通：行业/目标客户/获客方式/预算 | 👤 | 无规则 → 人工标记 |
| 0.2 | GEO诊断：搜品牌名+3行业词，记录9大AI平台曝光 | 🤖 | 无规则 → 豆包手动标记 |
| 0.3 | 竞品分析：搜3竞品，对比AI曝光 | 🤖 | 无规则 → 豆包手动标记 |
| 0.4 | 方案报价：推荐套餐，说明交付物和预期效果 | 👤 | 无规则 → 人工标记 |
| 0.5 | 签合同付款：签电子合同，客户付款，开通服务 | 👤 | 无规则 → 人工标记（签单日=后续计划日期锚点）|

#### 阶段1 项目初始化（5步，第1天）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 1.1 | 豆包创建客户项目：独立项目/会话 | 🤖 | 无规则 → 豆包手动标记 |
| 1.2 | 发送基础信息表：公司全称/地址/电话/官网/主营/竞品 | 👤 | 无规则 → 人工标记 |
| 1.3 | 逐曜创建公司+API密钥：companyId + API Key | 🤖 | `company_created == 1`（公司创建即自动命中） |
| 1.4 | 配置密钥到客户项目：Key 配到客户项目环境变量 | 🤖 | 无规则 → 豆包手动标记 |
| 1.5 | 创建客户专属文件夹：知识库文档/素材/报告目录 | 🤖 | 无规则 → 豆包手动标记 |

#### 阶段2 全网采集+知识库+图集+案例+官网（7步，第2-3天，最迟5天）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 2.1 | 全网信息采集：企查查/官网/抖音/小红书/地图/点评 | 🤖 | 无规则 → 豆包手动标记 |
| 2.2 | 信息对比与清洗：列不一致清单 | 🤖 | 无规则 → 豆包手动标记 |
| 2.3 | 知识库搭建(9模块)：intro/brand_story/product_system/pricing/faq/scenario/testimonials/variable/forbidden | 🤖 | `knowledge_module_count >= 9`（活跃知识库去重 moduleCode≥9）|
| 2.4 | 图集素材库(≥20张)：上传打标签+AI描述+默认分类 | 🤖 | `gallery_count >= 20` |
| 2.5 | 案例库(≥3个)：背景+挑战+方案+结果+数据（用 case_create 录入，published 缺省 false 待审核） | 🤖 | `case_count >= 3` |
| 2.6 | 客户确认锁定知识库：导出全文审核，反馈修改后锁定 | 🤝 | 无规则 → 客户确认后豆包手动标记 |
| 2.7 | 官网搭建+知识库同步：建站+同步+提交收录 | 🤖 | `website_count >= 1`（published 官网数≥1）|

#### 阶段3 蒸馏词配置（5步，第3-4天，与阶段2并行）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 3.1 | 导入行业基础词库：建材/机械/餐饮等预置词库 | 🤖 | 无规则 → 豆包手动标记 |
| 3.2 | 筛选核心词(5-10)：客户最想被搜到的词 | 🤖 | `keyword_count >= 5` |
| 3.3 | 扩展长尾词(20-50)：问题型/对比型/地域型 | 🤖 | `keyword_count >= 20` |
| 3.4 | 分组+标注搜索意图：品牌/行业/产品/问题/竞品/地域词 | 🤖 | 无规则 → 豆包手动标记 |
| 3.5 | 客户确认核心词：确认方向，导入系统蒸馏词模块 | 🤝 | 无规则 → 客户确认后豆包手动标记 |

#### 阶段4 首批内容生产与发布（6步，第5-7天）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 4.1 | 触发文章生成：用蒸馏词按套餐数量触发 | 🤖 | `article_count >= {{plan.monthly_articles}}` |
| 4.2 | 文章审核(6维≥70)：低分重写 | 🤖 | `article_count >= {{plan.monthly_articles}}` |
| 4.3 | 视频生成：优质文章转口播稿→渲染→成品 | 🤖 | `video_count >= {{plan.monthly_videos}}` |
| 4.4 | 配置发布平台+排期：按套餐10/20/28平台+分散时间 | 🤖 | 无规则 → 豆包手动标记 |
| 4.5 | 执行发布+重试：发布并记录状态，失败排查重试 | 🤖 | `publish_success_count >= 1` |
| 4.6 | 官网上线首批文章：发布到官网，检查渲染适配 | 🤖 | `website_count >= 1` |

#### 阶段5 GEO监测（5步，第8-14天）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 5.1 | 创建监测任务：按套餐关键词量 5/15/30 | 🤖 | `monitor_task_count >= 1`（active 任务数≥1）|
| 5.2 | 首次基线采集：9大AI平台首采，记录基线 | 🤖 | `monitor_result_count >= 1` |
| 5.3 | 记录排名趋势：上升/下降/持平 | 🤖 | `monitor_result_count >= {{plan.monitor_keywords}}` |
| 5.4 | 竞品对比监测：行业版套餐，竞品曝光对比 | 🤖 | 无规则 → 豆包手动标记（行业版）|
| 5.5 | 异常告警配置：排名骤降/竞品反超关注项 | 🤖 | 无规则 → 豆包手动标记 |

#### 阶段6 首次汇报·上线里程碑（3步，第14天）

| 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|--------|--------|---------------|
| 6.1 | 首次数据汇总：发布数/平台/收录情况/排名基线 | 🤖 | 无规则 → 豆包手动标记 |
| 6.2 | 首次成果汇报：线上会议15分钟+简报，过数据 | 🤖+👤 | 无规则 → 汇报完成后标记 |
| 6.3 | 基线锁定+下月方向：确认基线，定下月内容策略 | 👤 | 无规则 → 人工标记（触发阶段7运营循环）|

### 6.4 阶段7 持续运营循环（第15天起，每周循环，不进上线进度）

进入运营期后，上线进度固定100%，进度表显示"本月交付完成率"。以下步骤按周期循环：

| 周期 | 步骤 | 做什么 | 执行方 | autoCheckRule（系统自动判定） |
|------|------|--------|--------|------|
| 每周 | 7.1 | 周度内容计划：看监测数据定本周主题和数量 | 🤖 | 无规则 → 豆包手动标记 |
| 每周 | 7.2 | 文章批量生成：按套餐月量均摊每周 | 🤖 | `month_article_count >= {{plan.monthly_articles}}` |
| 每周 | 7.3 | 文章审核优化：6维≥70，低分重写，违规词拦截 | 🤖 | `month_quality_avg >= 70` |
| 每周 | 7.4 | 视频生成发布：优质文章转口播→渲染→发布 | 🤖 | `month_video_count >= {{plan.monthly_videos}}` |
| 每周 | 7.5 | 多平台发布：按排期发，失败重试 | 🤖 | `month_publish_success_rate >= 90` |
| 每周一/四 | 7.6 | 监测采集：触发9平台采集（异步，轮询progress至completed） | 🤖 | `month_monitor_count >= 2` |
| 每周五 | 7.7 | 数据复盘：对比上周排名/首推率/咨询量 | 🤖 | 无规则 → 豆包手动标记 |
| 按需 | 7.8 | 知识库更新：新产品/新案例/新荣誉，同步官网 | 🤖+👤 | 无规则 → 手动标记 |
| 每月 | 7.9 | 月度汇报：月报+沟通会+下月计划 | 🤖+👤 | 无规则 → 手动标记 |
| 每季度 | 7.10 | 季度续约评估：3个月效果总览+ROI+续约方案 | 👤 | 无规则 → 人工标记 |

**运营期核心闭环：** 监测数据 → 分析效果 → 调整内容方向 → 生产新内容 → 发布 → 再监测。不是"写完一批就完事"，是"看数据决定下一批写什么"。

### 6.5 异常告警规则（系统自动生成，运营总览聚合展示）

| 类型 | 触发条件 | 严重度 | 处理方 |
|----------|----------|--------|--------|
| `publish_fail` | 近7天发布失败数 > 0 | 🔴高 | 🤖自动重试 |
| `monitor_stale` | 有监测任务但近7天无新采集 | 🟠中 | 🤖自动重试 |
| `customer_timeout` | 客户确认类步骤（执行方🤝）到计划日期未完成 | 🟠中 | 👤催客户 |
| `overdue` | 其他步骤超计划日期未完成 | ⚪低 | 👤跟进 |

- 异常在运营总览「待处理异常」区聚合展示，按客户+严重度排序。
- Agent 处理完异常后调 `sop_progress_update` 标记对应步骤状态。

### 6.6 运营总览数据指标（运营总览页聚合展示）

| 指标 | 计算方式（对应系统返回字段） |
|------|----------|
| 客户总数 | 运营总览 `summary.totalCompanies` |
| 上线完成 | `summary.launchDone`（上线进度=100%的客户数）|
| 平均上线进度 | `summary.launchAvg`（全部客户上线进度均值%）|
| 平均本月交付率 | `summary.monthlyAvg`（全部客户阶段7交付率均值%）|
| 待处理异常 | `summary.alertCount`（未处理异常总数）|
| 近7天汇报客户数 | `summary.nextReportSoon`（7天内到汇报日的客户数）|
| 全局首推率 | `trend.mentionRate`（近14天目标品牌被提及平台占比）|
| 全局平均排名 | `trend.avgPosition`（近14天关键词平均排名，仅计被提及）|
| 全局发布成功率 | `trend.publishSuccessRate`（本月发布成功率均值）|

---

## 七、调用铁律

1. **Key 仅环境变量**：Key 只从环境变量读，对话/日志/文件/仓库绝不出现明文；泄露即吊销重建（§二）。
2. **最小权限**：只调 Key 已授权的能力；遇 403 立即停并报"缺哪个 scope/公司"，绝不越权扩权或换 Key。
3. **占位内容拦截**：`content`/`name`/`category` 命中模板串（`# 标题`、`知识库名称`、`分类`）→ 拒调，向用户要真实内容。
4. **软删不重建**：更新一律 `fill + 固定 moduleCode`（MCP `knowledge_fill` 带 moduleCode），禁止"先删再建"（§四.1）。
5. **幂等**：写操作必带 `Idempotency-Key` 头（REST）或 `idempotencyKey` 参数（MCP，23 个写工具支持），自定义字符串，批量每条唯一；断线重试传同一个 key 不会重复创建，处理中重复提交返回 409。
6. **限流退避**：同 Key ≤60 次/分，遇 429 指数退避（≤3 次后停，报"限流"）。
7. **异步轮询**：写操作拿到 `taskId` 后轮询至 `success`/`completed` 再下一步，不要假设立即成功。
8. **错误区分（403 两种）**：
   - `message` 含"需要 X scope / 权限不足" = **scope 不足**（Key 没授该模块），响应体带 `authorizedScopes` 数组可查；
   - `message` 含"不在授权范围 / companyId 越权" = **companyId 用错**，响应体带 `authorizedCompanyIds` 数组；
   - 401 = Key 失效/吊销，400 = 参数/违规词。
   - **429 三种来源（2026-09-05 起看 errorCode，处置动作完全不同）**：`RATE_5001`=真限流（指数退避后重试）；`RATE_DAILY_QUOTA`=当日配额用完（次日自动恢复，当天重试无效）；`RATE_CONCURRENT`=并发任务占满（轮询对应视频的 status 触发自动收口后即可重试）。响应体 message 已同步区分。
   - **400 含"请先完善公司资料"** = 目标公司资料不全（`missingFields` 列出缺失项，`settingsUrl` 指向管理端补全页）。先引导用户在管理端补齐资料再重试，**不要反复重试**（补资料前必然仍 400）。
   - MCP 接入：`tools/call` 返回 `isError` 时，业务错误在 `content[0].text` 的 JSON 里，按同样规则解读。
9. **严禁编造**：拿不到 Key/companyId/明确需求时，向用户确认或按 §三.2 BOOTSTRAP 自检，不要猜。

## 八、参考资料附件

本技能包 zip 内含以下附件（与 SKILL.md 同在一个包内），需要时按需读取：

- `references/rest-guide.md`：REST 接入速查（完整调用指南，接口清单/错误码/完整示例）
- `references/mcp-guide.md`：MCP 接入速查（端点到接入示例）

> 本文档为总入口（能力清单/工作流/铁律以本文为准）；遇到本文未覆盖的细节再读对应附件。
