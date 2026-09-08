# Agent OpenAPI 调用指南

> 版本：v1.14（2026-09-07）
> 面向对象：接入本系统的 Agent / 智能体 / 外部自动化程序（调用方）
> v1.14 更新：案例库发布权限收紧——新增 `case:review` scope（默认授权清单已收录）：持此 scope 的 Key 可在 case_create/case_update 直写 published=true 发布到官网（成功后自动触发官网同步），未持有的 Key 直写返回 403，只能存草稿（published 缺省/置 false）；案例引入 草稿→待审核→已发布/已驳回 状态机，published 由状态机联动维护
> v1.13 更新：新增案例语义匹配 GET /cases/match（scope 复用 `case:read`）——embedding 余弦相似度自动选案例，写文章引用案例优先用匹配而非人工翻列表；存量案例首次匹配现场补索引（每次最多补 20 条）；case_create/case_update 落库后自动异步刷新索引
> v1.12 更新：文章发布到官网支持草稿直发——status=1 草稿 / 3 发布失败自动提升为已发布再同步（返回 statusPromoted:true），生成中/审核不通过仍拒绝；免去"先在网页手动改发布状态"的两步操作
> v1.11 更新：新增案例库 CRUD（scope `case:read`/`case:write`，§二.11 + §二.7/§三 接口清单）——GET/POST /cases、GET/PUT/DELETE /cases/:id；article_generate 的 caseIds 引用链路补齐「查案例」一环；published 缺省 false 上站需后台审核
> v1.10 更新：analyze-article 的 long 模式改异步任务（返回 taskId，轮询 GET /tasks/{taskId} 取 result.script，支持 Idempotency-Key）；short/long 口播段均自动拆到 ≤15 字（≈2-3.75 秒/段）；背景轮播新增 switchMode="cycle"（图少段多时循环轮播）
> v1.9 更新：analyze-article 新增 `localize` 忠实度参数（缺省 false 忠实原文，不注入公司/地域词）；short 模式口播段自动拆到 ≤3.75s（背景按段切换不再触发画面冻结）；图集轮播新增 `background.switchMode: "duration"`（按 media[].duration 独立切换）；澄清 article 创建字段名（tags 逗号字符串 / categorySlug+websiteId，无 category 字段）
> v1.8 更新：新增公司资产上传 SOP——图集/产品库分类查建（GET+POST /gallery/categories、GET+POST /product-categories，scope 复用 gallery:write / product:write），上传前先查分类、无合适酌情创建；名称必写、标签/描述即使不强制也必填（便于自动配图与复用检索）
> v1.7 更新：蒸馏词模块补齐只读能力——新增 GET /keywords/groups（分组列表，分页 + 名称搜索，含 keywordCount）与 GET /keywords/groups/:id（分组详情含组内关键词），复用 `keyword:write` scope；向已有分组追加词前先查 groupId
> v1.6 更新：新增 AI 检测能力（scope `ai-detect:read`/`ai-detect:write`，§二.10 AI 检测章节 + §七 速查表）；与 GEO 监测（持续追踪品牌排名）不同，AI 检测是「单次诊断 + 多 AI 平台交叉对比」，且为租户级语义（同租户所有公司共享任务列表）
> v1.5 更新：403 响应结构化透传 `authorizedCompanyIds`/`authorizedScopes`（不再是纯文本提示，区分 scope 不足与 companyId 越权）；示例 companyId 统一用 `{companyId}` 占位（以 GET /companies 实际返回为准）
> v1.4 更新：新增 BOOTSTRAP 自查询端点（GET /me、GET /companies，认证即可访问无需 scope）；新增知识库详情接口 GET /knowledge/:id（返正文 content）；管理端 Key 列表直接显示绑定公司名称标签
> v1.3 更新：补全 DSH 任务接口（§二.8）、关键词分组接口、monitor results 接口；修正 rag/query 分类与示例 URL；新增 scope 速查表（§七）
> v1.2 更新：图集素材新增「上传」与「按关键词语义匹配选图」接口，支持智能体把图片入库并指定为文章配图（§二.7）
> v1.1 更新：知识库 fill 的「更新式写入（upsert）」与 DELETE「软删」语义说明（§二.6）+ 对应示例

---

## 〇、Base URL 与变量约定（先读）

| 变量 | 说明 |
|------|------|
| `{Base}` | 实际部署地址：`https://www.zhuyaoai.com/api/open/agent`（正式域名，LE 证书）；管理端「接入方式」弹窗展示当前实例真实 Base |
| `{companyId}` | 目标公司 ID，**不要猜**——先用 `GET {Base}/companies` 拿授权公司列表取实际 id |
| `{Key}` | `agk_` 开头完整密钥，仅环境变量/请求头，禁止落明文 |

> 示例请求体中 `companyId` 写 `{companyId}` 为占位符，复制到真实脚本时替换。

## 一、接入三步

### 1. 获取 Agent API Key

由租户管理员在管理端创建 Key，拿到三个凭据：

| 凭据 | 说明 | 安全要求 |
|------|------|----------|
| `Key` | 形如 `agk_` 开头的 64 位十六进制串 | 只显示一次，服务端仅存 SHA-256 哈希 |
| `Key ID` | 数字，用于查询任务/排障 | 公开 |
| `callbackSecret` | 48 位十六进制（配置了 webhook 才有） | 只显示一次，用于验签回调 |

创建 Key 时必须显式指定权限范围（scopes），**未指定任何权限的 Key 将无法创建**。

### 2. 确认权限范围（scope）

当前已注册 scope 一览。

> **新能力默认全量开放（2026-09-06 产品决策）**：新增能力的 scope（如案例库 `case:read`/`case:write`）自动对所有有效 Key 生效（后端 `AGENT_AUTO_GRANT_SCOPES` 合并），无需重建或补配 Key；`GET /me` 返回的 scopes 已含默认授权项。唯一例外：scopes 为空数组的异常 Key 仍 fail-closed 拒绝一切。

| Scope | 对应接口 |
|-------|----------|
| ——（无需 scope，认证即可访问） | **GET /me**（Key 自描述：scopes/companyIds/过期时间）、**GET /companies**（当前 Key 授权公司列表，含 id/name/shortName/industry，BOOTSTRAP 首选调用） |
| `knowledge:write` | POST /knowledge/fill（写知识库：传 moduleCode = 更新式写入 upsert，语义见 §二.6） |
| `knowledge:read` | GET /knowledge（列表，返元数据）、**GET /knowledge/:id**（详情，返完整条目含正文 content） |
| `knowledge:delete` | DELETE /knowledge/:id（软删，语义见 §二.6） |
| `article:create` | POST /articles |
| `article:read` | GET /articles、GET /articles/:id |
| `article:update` | PUT /articles/:id |
| `article:delete` | DELETE /articles/:id |
| `article:publish` | POST /articles/:id/publish |
| `article:generate` | POST /articles/generate、GET /articles/generate/:id、POST /articles/generate/:id/stop、POST /articles/generate/:id/resume（文章 AI 批量生成） |
| `publish:trigger` | POST /publish |
| `website:sync` | POST /website/sync |
| `blog:upload` | POST /blog（博客内容上传，需逐曜授权开通） |
| `video:read` | GET /videos、GET /videos/templates、GET /videos/:id、GET /videos/:id/status、GET /videos/:id/play-url |
| `video:write` | POST /videos（创建）、POST /videos/analyze-article（从文章生成口播稿）、POST /videos/:id/submit（提交出片） |
| `video:publish` | POST /videos/:id/publish（发布抖音，独立授权） |
| `monitor:read` | GET /monitor/analysis、GET /monitor/analysis/:id、GET /monitor/tasks、GET /monitor/tasks/:id、GET /monitor/tasks/:id/results |
| `monitor:write` | POST /monitor/tasks（创建 GEO 监测任务）、POST /monitor/tasks/:id/execute（触发采集） |
| `gallery:read` | GET /gallery、GET /gallery/:id、GET /gallery/match（图集素材只读 + 按关键词选图） |
| `gallery:write` | POST /gallery/upload、GET+POST /gallery/categories（图集素材上传与分类查建） |
| `case:read` | GET /cases（案例列表，分页 ≤100 + industry/tag/keyword 过滤）、GET /cases/match（按关键词语义匹配案例）、GET /cases/:id（案例详情，越权 404） |
| `case:write` | POST /cases（创建案例）、PUT /cases/:id（更新案例）、DELETE /cases/:id（删除案例，不可恢复） |
| `case:review` | 发布闸门（无独立端点）：case_create/case_update 直写 `published: true` 需持此 scope，未持有返回 403（只能存草稿）；持有时发布成功自动触发官网同步 |
| `rag:query` | POST /rag/query（知识库 RAG 语义检索，只读） |
| `dsh:task` | POST /dsh/tasks、GET /dsh/tasks/:id、POST /dsh/tasks/:id/cancel、GET /dsh/tasks/:id/artifacts/*file（DSH 任务提交/查询/取消/产物下载） |
| `keyword:write` | POST /keywords/groups、POST /keywords/groups/:id/import、GET /keywords/groups、GET /keywords/groups/:id（蒸馏词分组创建/导入/列表/详情，列表用于查 groupId） |
| `sop:read` | GET /sop/templates（SOP 模板 + 套餐定义，拿 stepId） |
| `sop:write` | PUT /sop/progress（SOP 运营进度显式标记，豆包/运营 Agent 执行后打勾） |
| `ai-detect:read` | GET /ai-detect/platforms、GET /ai-detect/tasks、GET /ai-detect/tasks/:id、GET /ai-detect/tasks/:id/source-overlap（AI 检测平台/任务/详情/信源重叠，只读） |
| `ai-detect:write` | POST /ai-detect/tasks（创建 AI 检测任务）、POST /ai-detect/tasks/:id/execute（触发检测）、POST /ai-detect/tasks/:id/retry（重试）、POST /ai-detect/tasks/:id/cross-validate（交叉验证分析） |
| `product:write` | POST /products/upload、GET+POST /product-categories（产品上传与分类查建：多图 files[]，首图=封面、余图=详情图） |
| `product:read` | GET /products（产品草稿列表）、GET /products/:id（产品草稿详情） |
| `download:write` | POST /downloads/upload（资料上传建草稿：单文件 file） |
| `download:read` | GET /downloads（资料草稿列表）、GET /downloads/:id（资料草稿详情） |

> 任务查询接口（GET /tasks、GET /tasks/:id）无需特定 scope 即可调用；可见任务类型自动按 Key 的 scope 收窄（如仅授 `knowledge:read` 的 Key 查询 publish 任务返回空列表/404，而非数据泄露）。

### 3. 确认公司授权范围（companyIds）

- 创建 Key 时指定 `companyIds`：非空数组 = 仅能操作这些公司；空数组 = 该租户旗下全部公司。
- 写接口必须传 `companyId`，且必须在授权范围内（越权访问返回 403）。

### 4. BOOTSTRAP 自检（每项目/每会话一次，消灭来回询问）

1. `GET {Base}/tasks` → 200 环境对、Key 有效；401 = Key 失效/吊销。
2. `GET {Base}/me` → scopes、companyIds、过期时间。
3. `GET {Base}/companies` → `items:[{id,name,shortName,industry}]`、`total`、`allCompanies`（true = 全局 Key）。
4. companyId 确定：以 `GET /companies` 返回且在 Key 授权范围内的为准；多家则列清单问用户"本项目对接哪家"并写入项目记忆；无数据提示先建数据。**不要猜测或硬编码 companyId**。
5. 能力自检：对将用到的读端点各探一次，200=可用、403=缺 scope。
6. 回执：`已就绪：环境={host}，公司={companyId}，可用能力={列表}`。

> 一把 Key 可授权多家公司、跨多个项目：每个项目各自跑一次 BOOTSTRAP，绑定各自 companyId。

---

## 二、调用规范

### Base URL

```
https://<你的域名>/api/open/agent
```

### 认证（所有接口）

```
X-Agent-API-Key: agk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 幂等（所有写操作）

写接口（POST/PUT/DELETE 类）建议带 `Idempotency-Key` 请求头（自定义字符串，如 `order-20260818-01`）：

- 同 Key + 同幂等键重放 → 返回首次执行结果（`idempotentReplay: true`），不重复执行
- 请求处理中重复提交 → 409 Conflict

### 限流

- 按 Key 维度：每分钟 60 次（超限 429）
- 写操作另有每日配额与并发上限（同 Key 同时 running 任务 < 5，超限 429）

### 错误码

| HTTP | 含义 |
|------|------|
| 200/201 | 成功 |
| 400 | 参数校验失败 / 回调地址校验拦截（内网、非 https 等） |
| 401 | Key 无效、过期、已吊销 |
| 403 | scope 不足（message 含"需要 X scope"，响应体带 `authorizedScopes`）/ companyId 越权（message 含"不在授权范围"，响应体带 `authorizedCompanyIds`） |
| 404 | 资源不存在 |
| 409 | 幂等键冲突（请求处理中） |
| 429 | 限流 / 日配额 / 并发超限 |

响应统一包装：`{"code":0,"message":"success","data":{...}}`；错误时 `code` 为非 0，`message` 为可读原因。

### 知识库写入与删除语义（智能体必读）

> 本节是知识库两个接口最容易踩坑的地方，请按下面的规则调用，**不要盲目重复测试**。

**写入（POST /knowledge/fill）—— 记住"按 moduleCode 认领"**

- **不带 `moduleCode`**：每次调用都新建一条记录，绝不覆盖任何旧数据。
- **带 `moduleCode` + `companyId`**（推荐用于维护某类固定知识）：
  - 该公司下已有这条 `moduleCode` 的**活跃**记录 → **原地更新**这条记录（name/content/category 等都被新值替换），返回 `result.updated=true`，记录 `id` 不变。
  - 没有活跃记录（从未建过、或已被删除）→ **新建**一条，返回 `result.updated=false`。
- **结论**：要"更新已有知识"，固定用同一个 `moduleCode` 反复调用即可，系统自动判断是更新还是新建。**不要"先删再建"**——直接 fill 更新最安全。
- **特别注意**：被删除的记录不会"复活"。它已打删除标记，再 fill 同 `moduleCode` 会新建一条全新的，而不是把旧记录翻回来。

**删除（DELETE /knowledge/:id）—— 记住"这是软删，不是物理删除"**

- 删除 = 给该记录打"已删除"标记，**数据仍存在数据库中**，只是不再出现在查询列表、官网同步结果中。
- 返回 `deleted=true`；**重复删除同一条也返回成功**（幂等，不报错），这是设计行为。
- 想删哪条：先用 `GET /knowledge` 查列表拿到 `id`，再 DELETE。
- **只有确实要让某条知识"下线"时才用 DELETE**；只是想改内容，一律用 fill 更新。

**一条铁律**

| 意图 | 正确做法 |
|------|----------|
| 新建知识 | fill（不带 moduleCode，或带新 moduleCode） |
| 更新已有知识 | fill（带同一个 moduleCode，原地覆盖） |
| 让知识下线（不再展示） | DELETE /knowledge/:id |
| "删掉重来" | ❌ 禁止——更新就用 fill，不需要先删 |

### 图集素材：上传与选图（智能体必读）

图集素材是写文章时的配图素材池。智能体可以把图片上传进图库，写文章时让系统自动配图。

**上传（POST /gallery/upload，multipart/form-data，需 `gallery:write`）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | file | 是 | 图片（jpeg/png/gif/webp/svg，≤10MB） |
| `companyId` | int | 是 | 目标公司（须在 Key 授权范围内，越权 403） |
| `name` | string | 否 | 素材名，默认取文件名。**智能体必写**（描述图片主体） |
| `categoryId` | int | 否 | 分类。**先查再传**（见下） |
| `tags` | string | 否 | 逗号分隔标签。**智能体必写**（配图评分权重最高） |
| `aiDescription` | string | 否 | AI 语义描述（≤1000 字），增强长尾匹配 |
| `textPosition` | string | 否 | top/upper/middle/lower/bottom，默认 middle |

**分类（v1.7 新增，先查再传，无合适则建）**

```
GET  /api/open/agent/gallery/categories?companyId={companyId}    # 分类树（含二级子分类）
POST /api/open/agent/gallery/categories                          # 创建分类
```

POST body：`{ "companyId": {companyId}, "name": "装修实景", "parentId": null, "sort": 0 }`（parentId 不传为一级分类）。
上传 SOP：先 `GET /gallery/categories` 找语义合适的分类拿 `categoryId`；确无合适再 `POST` 新建（命名表意清晰，方便以后复用检索），再用新 id 上传。

> **命名与打标决定配图质量**：写文章自动配图时，系统按 `tags`（权重最高）→ `name` → 分类 → `aiDescription` 对图库评分。想让图片在写文章时被准确匹配到，**上传时务必写好 `tags` 与 `name`**（描述图片主体，如"现代客厅 装修实景"）。图片缺标签时只能走随机兜底，可能与文章主题无关。

**选图（GET /gallery/match，需 `gallery:read`）**

按关键词对图库评分匹配，返回匹配度排序的候选图：

```
GET /api/open/agent/gallery/match?keyword=装修&companyId={companyId}&categoryId=3&limit=6&minScore=0.15
```

参数：`keyword`（必填）、`companyId`（必填，越权 403）、`categoryId`/`limit`（默认 6，≤20）/`minScore`（默认 0.15）可选。

**写文章配图闭环（推荐工作流）**

1. 上传图片并写好 `tags`/`name`（或先用 match 查已有素材，拿候选 `id`）；
2. 调 `POST /articles/generate` 写文章时，把选中的图片 `id` 放进 `galleryIds` 数组；
3. 系统写文章时**优先用这批图**配正文插图与封面；未指定 `galleryIds` 时，系统自动按关键词从图库匹配。

### DSH 任务：提交与查询（智能体必读）

DSH（DeepSeek Harness）是系统内置的代码执行与分析沙箱，可作为智能体的"副手"完成需要运行代码、读写文件、安装依赖的任务。任务异步执行，提交即返回 taskId。

**提交任务（POST /dsh/tasks，需 `dsh:task`）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `companyId` | int | 是 | 目标公司（须在 Key 授权范围内） |
| `prompt` | string | 是 | 任务指令（自然语言描述要做什么） |
| `model` | string | 否 | 指定模型，缺省用系统默认 |
| `maxTokens` | int | 否 | 最大 token 数（1–100000） |
| `tokenBudget` | int | 否 | token 预算上限 |
| `expectedOutput` | object | 否 | 期望产物声明，完成后按 files 校验 |
| `expectedOutput.files` | array | 否 | 期望产物文件列表 |
| `expectedOutput.files[].path` | string | 是* | 产物文件相对路径（*声明了 files 时必填） |
| `expectedOutput.files[].required` | bool | 否 | 是否必须存在（默认 true） |
| `expectedOutput.files[].match` | string | 否 | 文件内容匹配规则 |

```bash
curl -X POST 'https://<域名>/api/open/agent/dsh/tasks' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -H 'Idempotency-Key: dsh-20260822-01' \
  -d '{
    "companyId": {companyId},
    "prompt": "分析项目目录结构，输出树形图到 tree.txt",
    "expectedOutput": {
      "files": [{ "path": "tree.txt", "required": true }]
    }
  }'
```

**查询任务（GET /dsh/tasks/:id）**

返回任务状态（pending/running/completed/failed/cancelled）、输出日志、产物文件列表。

**取消任务（POST /dsh/tasks/:id/cancel）**

仅运行中的任务可取消，已完成的返回错误。

**下载产物（GET /dsh/tasks/:id/artifacts/*file）**

`*file` 为产物文件相对路径，直接返回文件二进制流。

> DSH 限流较宽松（每分钟 120 次），但任务并发受系统配额限制；长任务建议配合 webhook 回调。

### 视频：创建/出片/发布（智能体必读）

视频能力 = 口播短视频（文字 → TTS 人声 + 字幕 + 背景 + BGM 渲染 MP4），非文生视频。完整链路 = `POST /videos` 创建 → `POST /videos/analyze-article`（可选，从文章生成口播稿）→ `POST /videos/:id/submit` 出片 → 轮询状态 → `POST /videos/:id/publish` 发布抖音。

**创建（POST /videos，需 `video:write`）**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `companyId` | int | 是 | 目标公司（须在 Key 授权范围内） |
| `title` | string | 是 | 视频标题 |
| `templateId` | string | 否 | 公司视频模板 ID：后端把模板 config 铺进 videoConfig/voiceConfig（仅限自身公司模板；**不传时自动套用该公司「默认模板」**；系统内置模板缺品牌素材不可引用会 403） |
| `videoConfig` | object | 否 | 视频配置（传 templateId 时可叠加覆盖模板项；不传时作为完整自备配置）。背景轮播节奏 `videoConfig.background.switchMode`：缺省 `"round"` 按口播段切换（图用完钉在末图）；`"cycle"` 图数<段数时按轮次循环轮播（10 图配 50 段即每 2-3 秒切一张）；`"duration"` 按 `background.media[].duration` 独立切换、与口播段解耦（cycle/duration 为 v1.9+ 新增） |
| `voiceConfig` | object | 否 | 音色配置（maleVoice/femaleVoice/singleVoice 等） |
| `script` | string | 否 | 口播稿 JSON 段数组字符串（role 9 选 1，与系统 SCRIPT_ANALYSIS_PROMPT segments 格式一致；缺失时后续 submit 会解析报错） |
| `sourceArticleId` | int | 否 | 从已有文章派生（与 script 二选一或组合；whole 模式按此拉全文） |
| `desc` / `hashtags` | string | 否 | 作品描述（≤1000 字）/ 话题标签（逗号分隔） |

**从文章生成口播稿（POST /videos/analyze-article，需 `video:write`）**：参数 `companyId`/`articleId` 必填，`mode`（short 短视频 / long 播客长文，缺省 long）、`provider`、`localize`（本地化改写，缺省 false）可选。

> **调用形态按 mode 区分（v1.10）**：
> - **short（同步）**：40-60s 短视频口播，单次 AI 调用（约 30-60s），每段已自动拆到 ≤15 字（≈2-3.75 秒），直接返回口播稿结果。
> - **long（异步任务）**：播客长文全文保真（≥原文 85%），分段生成耗时数分钟——**返回 `{taskId, result{status:'running'}}`，用 `GET /tasks/{taskId}` 轮询至 success，`result.script` 即口播稿段数组**（或配置 webhook 自动推送）。每段同样已自动拆到 ≤15 字，两三分钟口播约 50-90 段。带 `Idempotency-Key` 防重复提交。
>
> **localize（忠实度）**：缺省 `false` = 忠实原文——不注入公司认知，禁止添加原文没有的地域词/公司名/品牌词，省略品牌段；传 `true` = 注入公司认知允许品牌段（营销口播场景）。

**提交出片（POST /videos/:id/submit，需 `video:write`）**：异步长任务（出片需数分钟）。返回 `taskId` + `videoId` + `buildTaskId`。轮询 `GET /videos/:id/status`（构建阶段与进度：scripting/tts/rendering/done/failed）或 `GET /tasks/:id`（agent 任务状态实时回填：running → success 表示出片完成，result 附 outputMp4/thumbnailUrl）。口播稿为空或含违禁词返回 400；构建中重复提交返回 400。

**播放（GET /videos/:id/play-url，需 `video:read`）**：构建完成（done）后可取播放签名 URL（对象存储 24h 有效，降级本地 Nginx 路径）。

**发布（POST /videos/:id/publish，需 `video:publish`）**：独立授权，与出片分开控制。`accountId` 可选——不传则系统自动选该公司第一个可用的抖音账号（platformType=39 且已登录），凭证/账号在系统内流转，agent 无需感知账号明细；`publishMode` 可选（publish 正式发布 / draft 仅存草稿，缺省取账号配置）。仅构建完成（done）的视频可发布。

**模板列表（GET /videos/templates，需 `video:read`）**：查询当前 Key 可见公司的视频模板（只返回公司自身模板，含 `isCompanyDefault` 默认标记；公共模板 companyId=0 不返回）。参数 `companyId` 可选——传入须在 Key 授权范围内（越权 403），不传则返回全部可见公司模板（每项带 companyId）。返回精简字段 `id/companyId/name/isCompanyDefault/baselineId/createdAt/updatedAt`（不含 config）。**用途**：创建视频前先查模板，确认哪个是「默认模板」（`isCompanyDefault=true`）；不传 `templateId` 创建即自动套用它（效果与管理后台一致），传 `templateId` 可固定指定模板风格。

> 视频模板是品牌配置资产，不开放模板 CRUD；agent 需要特殊风格时走 `videoConfig` 自备，不污染模板库。**模板策略（2026-08-23 拍板）**：agent 出片默认走「公司默认模板」（在视频设置中把某模板设为默认即生效，改默认模板即可换出片风格）；系统内置模板（大字报/播放器卡片等）缺品牌信息与背景素材，agent 不可引用（传入 403）；agent 不应自行编排/反复向用户询问模板，减少误判。

### AI 检测：多 AI 平台交叉诊断（智能体必读）

**一句话定位**：输入一个关键词，让多个 AI 平台各答一遍，拉出各平台回答正文 + 信源，做交叉对比（共识/冲突/独特三栏）与信源重叠分析。与 GEO 监测（品牌收录排名持续追踪）不同，AI 检测是**单次诊断 + 多 AI 对比**。

**标准三步流程（需 `ai-detect:read` + `ai-detect:write`）**：

1. **查平台（GET /ai-detect/platforms）**：拿可用平台 code（含豆包三模式 `doubao:quick`/`doubao:think`/`doubao:expert`、kimi、yuanbao、qwen、wenxin、zhipu、deepseek、xunfei、ai360）。不传 platforms 时系统默认全平台。
2. **创建（POST /ai-detect/tasks）**：body `{ companyId, keyword, scenario?, platforms? }`。`companyId` 必填且须在 Key 授权范围（越权 403）。只落库 `pending`，不自动执行。
3. **执行 + 轮询（POST /ai-detect/tasks/:id/execute → GET /ai-detect/tasks/:id）**：execute 是 fire-and-forget，立即返回 `{taskId, status:'running'}`；轮询 `GET /ai-detect/tasks/:id`，`status` 变为 `completed` 后 `results[]` 含各平台回答（content）、信源（references[]，title+url）、信源域名（sourceDomains）、竞品提及、自有域名引用。

**可选进阶**：
- **信源重叠（GET /ai-detect/tasks/:id/source-overlap）**：看哪些信源被多个 AI 平台同时引用，判断信息源广度。
- **交叉验证（POST /ai-detect/tasks/:id/cross-validate）**：对已保存结果调 DeepSeek 分析，返回共识/冲突/独特三栏结论（补充 AI 分析视角）。
- **重试（POST /ai-detect/tasks/:id/retry）**：带 `{platform, mode}` 仅重试单个平台；不传则整任务重试。

> AI 检测是**租户级**功能：同一租户下所有公司共享检测任务列表（GET /ai-detect/tasks 返回租户全部任务），与知识库/文章的按公司隔离语义不同。任务查询/执行均按租户过滤（跨租户 越权 404）。

### 案例库：真实案例引用与沉淀（智能体必读）

**一句话定位**：案例素材库是结构化的客户案例（背景/痛点/方案/过程/效果五要素 + 量化指标 + 图片），写文章引用真实案例能显著增强说服力。`POST /articles/generate` 早就支持 `caseIds`，v1.11 起补齐了「查案例/沉淀案例」的入口。

**标准流程（需 `case:read` / `case:write`）**：

1. **查案例（GET /cases，需 `case:read`）**：`?page=&pageSize≤100=&industry=&tag=&keyword=`（关键词扫标题/摘要/客户背景/痛点）。列表不含对话原文，拿 id 即可。
2. **语义匹配（GET /cases/match，需 `case:read`）**：`?keyword=&companyId=&limit≤20=&minScore≤1=`，embedding 余弦相似度排序（minScore 缺省 0.15），**写文章引用案例优先用匹配自动选**。存量案例首次匹配会现场补 embedding 索引（每次调用最多补 20 条）可能略慢；case_create/case_update 落库后自动异步刷新索引，无需手动维护。
3. **引用（POST /articles/generate）**：把案例 `id` 放进 `caseIds` 数组；系统生成文章时把案例素材注入正文论证。也可 `GET /cases/:id` 取完整详情（含实施过程与 `keyMetrics` 量化指标）后手动改写。
4. **沉淀（POST /cases，需 `case:write`，幂等）**：五要素必填 `title/clientProfile/painPoints/solution/outcome`，可选 `summary/industry/serviceType/process/keyMetrics(对象)/tags[]/images[]`。**`published` 缺省 false 不上站**；直发需 `case:review` scope（未持有写 published=true 返回 403），发布成功自动触发官网重建。草稿如需人工把关，保持 published 缺省即可，后台会走 待审核→已发布 流程。
5. **维护（PUT/DELETE /cases/:id，需 `case:write`）**：PUT 仅更新传入项（无需 companyId）；DELETE 不可恢复且级联清理与文章任务的关联，删前确认 id。案例配图可先用 GET /gallery/match 选图，把图片 URL 回填到 `images`。

---

## 三、接口清单

### 文章 AI 生成参数说明（POST /articles/generate）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `companyId` | int | 是 | 目标公司 |
| `name` | string | 是 | 任务名称（也是文章主题/关键词，≤200 字） |
| `totalCount` | int | 否 | 生成篇数，缺省由服务端默认 |
| `aiModel` | string | 否 | 指定 AI 模型 |
| `promptId` | int | 否 | 指定单个提示词模板 ID |
| `keywordGroupIds` | int[] | 否 | 关联蒸馏词分组 ID |
| `knowledgeIds` | int[] | 否 | 指定引用的知识库条目 ID |
| `galleryIds` | int[] | 否 | 指定配图素材 ID（通过 GET /gallery/match 选取） |
| `caseIds` | int[] | 否 | 关联案例 ID（通过 GET /cases 查询，v1.11 起可查，语义见 §二.11） |
| `topicIds` | int[] | 否 | 关联话题 ID |
| `promptIds` | int[] | 否 | 指定提示词模板 ID |
| `conversionTargetKeys` | string[] | 否 | 转化目标标识 |
| `intentFilter` | string | 否 | 意图过滤条件 |
| `targetPlatforms` | string[] | 否 | 目标发布平台 |

> 异步接口：返回 `articleTaskId`，通过 GET /articles/generate/:id 轮询状态（pending → processing → completed/failed）。completed 后通过 GET /articles 获取生成的文章列表。

### 写操作（同步返回任务结果）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /knowledge/fill | 知识库写入（质量闸门校验，违规词 400；带 moduleCode = 更新式写入 upsert，语义见 §二.6） |
| DELETE | /knowledge/:id | 删除知识库条目（软删，数据保留仅下线；重复删返回成功；被文章任务引用时 409；语义见 §二.6） |
| POST | /articles | 创建文章（字段名注意：`tags`=逗号分隔字符串非数组；无 `category` 字段——挂官网栏目用 `categorySlug` 且必须配 `websiteId`；两者可选，乱传未声明字段会被 400 拒绝） |
| PUT | /articles/:id | 更新文章 |
| DELETE | /articles/:id | 删除文章 |
| POST | /articles/:id/publish | 文章发布到官网 |
| POST | /publish | 触发多平台发布 |
| POST | /website/sync | 官网内容同步（知识库模块 + 托管站全量） |
| POST | /articles/generate | 触发文章 AI 批量生成（异步，返回 articleTaskId，轮询状态） |
| POST | /articles/generate/:id/stop | 停止生成中的文章任务（仅 status=processing 可停） |
| POST | /articles/generate/:id/resume | 恢复失败的文章任务（仅 status=failed 可恢复） |
| POST | /monitor/tasks | 创建 GEO 监测任务（不自动执行） |
| POST | /monitor/tasks/:id/execute | 触发监测任务采集（异步，返回后轮询状态/结果） |
| POST | /gallery/upload | 图集素材上传（multipart/form-data，含 name/tags/aiDescription 打标，语义见 §二.7） |
| POST | /dsh/tasks | 提交 DSH 任务（DeepSeek Harness 代码/分析任务，异步，返回 taskId，语义见 §二.8） |
| POST | /dsh/tasks/:id/cancel | 取消运行中的 DSH 任务 |
| POST | /keywords/groups | 创建蒸馏词组（一键表格导入：可同时传 keywords 数组建组+导词，coreWord 缺省取首词） |
| POST | /keywords/groups/:id/import | 批量导入关键词到指定分组（groupId 用 GET /keywords/groups 查） |
| GET | /keywords/groups | 蒸馏词组列表（分页 ≤100 + ?keyword 名称搜索，含 keywordCount；scope 同 keyword:write） |
| GET | /keywords/groups/:id | 蒸馏词组详情（含词根维度与组内关键词分页列表） |
| POST | /videos | 创建视频项目（异步返回 taskId；可传 templateId 引用公司视频模板铺配置，或自备 videoConfig/script，语义见 §二.9） |
| POST | /videos/analyze-article | 从文章生成口播稿（同步轻量操作，直接返回口播稿段数组） |
| POST | /videos/:id/submit | 提交视频出片（异步，出片需数分钟；轮询 GET /videos/:id/status 或 GET /tasks/:id，语义见 §二.9） |
| POST | /videos/:id/publish | 发布视频到抖音（accountId 可不传，系统自动选公司第一个可用抖音账号） |
| POST | /ai-detect/tasks | 创建 AI 检测任务（body: companyId/keyword/scenario/platforms；只落库 pending，不自动执行） |
| POST | /ai-detect/tasks/:id/execute | 触发 AI 检测（fire-and-forget 立即返回 running，轮询 GET /ai-detect/tasks/:id 看结果） |
| POST | /ai-detect/tasks/:id/retry | 重试检测（可选 body: platform/mode 单平台重试；不传整任务重试） |
| POST | /ai-detect/tasks/:id/cross-validate | 对已保存结果做 AI 交叉验证（DeepSeek 输出共识/冲突/独特三栏） |
| POST | /products/upload | 产品上传建草稿（multipart，files[] 多图，首图=封面、余图=详情图；published 强制 false 不上站） |
| POST | /downloads/upload | 资料上传建草稿（multipart，file 单文件；category 缺省"资料"；published 强制 false 不上站） |
| POST | /cases | 创建案例（JSON body；五要素必填 title/clientProfile/painPoints/solution/outcome；published 缺省 false 不上站，直发需 case:review，语义见 §二.11） |
| PUT | /cases/:id | 更新案例（仅更新传入项，无需 companyId） |
| DELETE | /cases/:id | 删除案例（不可恢复，级联清理与文章任务的关联） |

### 读操作（只读、分页、租户隔离）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /me | 当前 Key 自描述（keyId/tenantId/scopes/companyIds/isActive/expiresAt，无需 scope，BOOTSTRAP 用） |
| GET | /companies | 当前 Key 授权公司列表（items:[{id,name,shortName,industry}] + total + allCompanies；allCompanies=true = 全局 Key；无需 scope） |
| GET | /knowledge?category=&search=&companyId= | 知识库列表（仅元数据，不含正文；可选 companyId：传入时须在 Key 授权公司内，越权 403） |
| GET | /knowledge/:id | 知识库详情（返完整条目含正文 content；越权 404；审计/优化正文必用） |
| GET | /articles?page=&pageSize=&websiteId= | 文章列表 |
| GET | /articles/:id | 文章详情 |
| GET | /videos?page=&pageSize=&status=&search= | 视频项目列表 |
| GET | /videos/templates?companyId= | 公司视频模板列表（含 isCompanyDefault 默认标记；companyId 可选，越权 403） |
| GET | /videos/:id | 视频详情 |
| GET | /videos/:id/status | 视频构建状态与进度（scripting/tts/rendering/done/failed） |
| GET | /videos/:id/play-url | 视频播放签名 URL（对象存储 24h 有效，降级本地） |
| GET | /monitor/analysis?page=&pageSize=&keyword=&status= | GEO 监测分析列表 |
| GET | /monitor/analysis/:id | 监测分析详情 |
| GET | /monitor/tasks?page=&pageSize= | GEO 监测任务列表 |
| GET | /monitor/tasks/:id | GEO 监测任务详情 |
| GET | /monitor/tasks/:id/results?platform=&keyword=&page=&pageSize= | GEO 监测原始结果（按平台×关键词，含评分+快照 JSON） |
| GET | /gallery?page=&pageSize=&categoryId=&search= | 图集素材列表（含 sm/md/lg/webp 派生 URL） |
| GET | /gallery/:id | 图集素材详情 |
| GET | /gallery/match?keyword=&companyId=&categoryId=&limit=&minScore= | 图集素材按关键词评分匹配选图（语义见 §二.7） |
| POST | /rag/query | 知识库 RAG 语义检索（只读，body: query/companyId/topK） |
| GET | /dsh/tasks/:id | DSH 任务状态/结果查询 |
| GET | /dsh/tasks/:id/artifacts/*file | DSH 任务产物文件下载 |
| GET | /articles/generate/:id | 文章生成任务状态（pending/processing/completed/failed + 进度） |
| GET | /ai-detect/platforms | AI 检测可用平台列表（含 doubao/kimi/yuanbao/qwen/wenxin/zhipu/deepseek/xunfei/ai360 及豆包三模式） |
| GET | /ai-detect/tasks | AI 检测任务列表（租户级，同租户所有公司共享检测任务） |
| GET | /ai-detect/tasks/:id | AI 检测任务详情（含各平台 results：回答/信源/信源域名/竞品提及/自有域名引用） |
| GET | /ai-detect/tasks/:id/source-overlap | AI 检测信源重叠分析（各平台引用同一信源的交叉统计） |
| GET | /tasks?page=&pageSize=&type= | 任务列表 |
| GET | /tasks/:id | 任务详情 |
| GET | /products?companyId=&page=&pageSize=&keyword=&categoryId= | 产品草稿列表（companyId 必填，越权 403） |
| GET | /products/:id?companyId= | 产品草稿详情（越权 404） |
| GET | /product-categories?companyId=&page=&pageSize= | 产品分类列表（分页；上传产品前先查） |
| POST | /product-categories | 创建产品分类（body: companyId+name(+sortOrder/isActive)；重名 400 当已存在处理） |
| GET | /downloads?companyId=&page=&pageSize=&category=&keyword= | 资料草稿列表（companyId 必填，越权 403） |
| GET | /downloads/:id?companyId= | 资料草稿详情（越权 404） |
| GET | /cases?page=&pageSize=&industry=&tag=&keyword= | 案例列表（分页 ≤100，关键词扫标题/摘要/客户背景/痛点；列表不含对话原文） |
| GET | /cases/match?keyword=&companyId=&limit=&minScore= | 案例语义匹配（embedding 余弦排序，minScore 缺省 0.15；语义见 §二.11） |
| GET | /cases/:id | 案例详情（含实施过程与 keyMetrics；越权 404） |

### 公司资产上传 SOP（图集/产品库/资料库通用，v1.8）

1. **先查分类**：图集用 `GET /gallery/categories`、产品库用 `GET /product-categories`；资料库 category 是自由文本，用 `GET /downloads?companyId=X` 看现有资料用了哪些分类名并复用。
2. **无合适分类酌情创建**：图集 `POST /gallery/categories`、产品库 `POST /product-categories`（重名 400 = 已存在，重新查列表拿 id 即可）；命名表意清晰，方便以后复用检索。
3. **名称必写**：图集 `name`（描述图片主体）、产品 `name`、资料 `title`，禁止留空落文件名兜底。
4. **标签/描述即使系统不强制也要填**：图集 `tags`+`aiDescription`（配图评分权重最高）、产品 `tags`+`summary`、资料 `description`——这是以后自动配图与检索复用的关键索引，缺了只能随机兜底。

### 文章发布到官网参数说明（POST /articles/:id/publish）

按网站 `publishMode` 分派：

- **receiver 模式**（客户自建 CMS + receiver 接收）：真正推送到客户站，透传分类；返回含 `pushed`/`remoteArticleId` 等推送结果。
- **托管/managed 等非 receiver 模式**：仅建立本地关联（`website_articles`），不推送客户 CMS，返回 `note` 说明。
- **草稿可直接发布（v1.11）**：文章 status=1 草稿 / 3 发布失败 会自动提升为 2（已发布）再同步官网，返回含 `statusPromoted:true`——无需先在网页手动改状态；status=0 生成中 / 4 审核不通过 仍被拒绝。

| 字段 | 类型 | 必填 | 说明 |
|------|------|:---:|------|
| websiteId | number | 是 | 目标官网站点 id（须属于文章所在公司，越权 403） |
| categorySlug | string | 否 | 客户 CMS 分类 slug。receiver 模式透传给客户站；非 receiver 模式落 `website_articles.categorySlug` |

```bash
curl -X POST 'https://<域名>/api/open/agent/articles/123/publish' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -d '{"websiteId": 5, "categorySlug": "news"}'
```

receiver 模式返回示例：

```json
{
  "published": true,
  "pushed": true,
  "remoteArticleId": 888,
  "articleId": 123,
  "websiteId": 5
}
```

非 receiver 模式返回示例：

```json
{
  "published": true,
  "articleId": 123,
  "websiteId": 5,
  "websiteArticleId": 55,
  "note": "官网模式 hosted：仅建立本地关联，未推送到客户 CMS（receiver 模式才支持推送）"
}
```

---

## 四、任务回调（Webhook，可选）

创建 Key 时配置 `webhookUrl`（仅 https；内网 IP、云元数据地址、localhost、非 443 端口一律拒绝）。

### 触发时机

任务进入终态（`success` / `failed`）自动投递一次：

- 事件名：`task.success` / `task.failed`
- 同任务同事件只投递一次（失败自动重试：1 分钟 → 5 分钟 → 30 分钟，最多 3 次）

### Payload（POST，Content-Type: application/json）

```json
{
  "event": "task.success",
  "taskId": 8,
  "type": "website",
  "action": "sync",
  "status": "success",
  "result": { "websiteId": 5, "hostedSync": true },
  "error": null,
  "finishedAt": "2026-08-18T12:24:51.271Z",
  "deliveredAt": "2026-08-18T12:24:51.280Z"
}
```

### 验签（必做，防伪造）

请求头：`X-Agent-Webhook-Signature: sha256=<hex>`

其中 `<hex>` = **原始请求体字符串** 的 HMAC-SHA256，密钥为 `callbackSecret`。**验签必须使用原始 body 字节，不能 JSON 重排后再算。**

```python
import hashlib, hmac, json

def verify_signature(raw_body: bytes, signature: str, callback_secret: str) -> bool:
    # signature 形如 "sha256=<64位hex>"
    expected = "sha256=" + hmac.new(
        callback_secret.encode(), raw_body, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

# 用法：先把 raw body 存下来，再 json.loads 解析业务
```

### 幂等消费建议

回调可能重试（重复收到同 event 的请求），回调方应按 `taskId + event` 去重。

---

## 五、完整示例

### 示例 1：写入知识库（带幂等）

```bash
curl -X POST 'https://<域名>/api/open/agent/knowledge/fill' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -H 'Idempotency-Key: kb-fill-20260818-01' \
  -d '{
    "companyId": 95,
    "name": "洛阳装修知识",
    "type": "product",
    "content": "# 洛阳装修注意事项\n...",
    "category": "装修攻略"
  }'
```

返回：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "taskId": 8,
    "result": { "id": 12, "name": "洛阳装修知识" }
  }
}
```

### 示例 2：更新已有知识（upsert，带 moduleCode 原地覆盖）

> 用同一个 `moduleCode` 再次调用，系统会更新原记录（`updated=true`），而不是新建。

```bash
curl -X POST 'https://<域名>/api/open/agent/knowledge/fill' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -H 'Idempotency-Key: kb-fill-20260818-02' \
  -d '{
    "companyId": 95,
    "name": "洛阳装修知识（更新版）",
    "type": "product",
    "moduleCode": "company_intro",
    "content": "# 洛阳装修注意事项（修订后）\n...",
    "category": "装修攻略"
  }'
```

返回（注意 `updated: true`，且 `id` 与第一次创建时相同）：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "taskId": 9,
    "result": { "id": 12, "name": "洛阳装修知识（更新版）", "updated": true }
  }
}
```

### 示例 3：删除知识库条目（软删）

> 先查列表拿到 `id`，再 DELETE。软删后数据保留，只是不再展示；重复删除同一条也返回成功。

```bash
# 1) 查列表拿到要删的 id
curl -s 'https://<域名>/api/open/agent/knowledge?search=洛阳装修' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'

# 2) 删除
curl -X DELETE 'https://<域名>/api/open/agent/knowledge/12' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'
```

返回：

```json
{
  "code": 0,
  "message": "success",
  "data": { "deleted": true, "id": 12, "companyId": 95 }
}
```

### 示例 4：查询任务结果

```bash
curl -s 'https://<域名>/api/open/agent/tasks/8' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'
```

### 示例 5：官网同步

```bash
curl -X POST 'https://<域名>/api/open/agent/website/sync' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -H 'Idempotency-Key: ws-sync-20260818-01' \
  -d '{"companyId": {companyId}, "moduleCode": "service"}'
```

> 未配置官网的公司返回 404 可读提示；配置了官网触发知识库模块 + 托管站全量同步。

### 示例 6：上传图集素材（multipart/form-data）

```bash
curl -X POST 'https://<域名>/api/open/agent/gallery/upload' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -F 'companyId={companyId}' \
  -F 'name=现代客厅装修实景' \
  -F 'tags=装修,客厅,实景' \
  -F 'aiDescription=现代简约风格客厅装修完工实景图，浅色系' \
  -F 'file=@/tmp/living-room.jpg'
```

返回（data 含 id/fileUrl/thumbnailUrl/smUrl/mdUrl/lgUrl/webpUrl）：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 66,
    "fileUrl": "/uploads/xxx.jpg",
    "thumbnailUrl": "/uploads/thumb_xxx.jpg",
    "smUrl": "/uploads/sm_xxx.jpg",
    "mdUrl": "/uploads/md_xxx.jpg",
    "lgUrl": "/uploads/lg_xxx.jpg",
    "webpUrl": "/uploads/xxx.webp"
  }
}
```

### 示例 7：按关键词匹配选图

```bash
curl -s 'https://<域名>/api/open/agent/gallery/match?keyword=装修&companyId={companyId}&limit=6' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'
```

返回按匹配度排序的候选图（含 `score`），选中后把图片 `id` 放入 `POST /articles/generate` 的 `galleryIds`。

### 示例 8：提交 DSH 任务并查询结果

```bash
# 1) 提交任务
curl -X POST 'https://<域名>/api/open/agent/dsh/tasks' \
  -H 'Content-Type: application/json' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx' \
  -H 'Idempotency-Key: dsh-20260822-01' \
  -d '{
    "companyId": {companyId},
    "prompt": "统计 uploads 目录下各类型文件数量，输出到 report.txt",
    "expectedOutput": { "files": [{ "path": "report.txt" }] }
  }'

# 返回：{ "code": 0, "data": { "taskId": 69, "status": "pending" } }

# 2) 轮询状态
curl -s 'https://<域名>/api/open/agent/dsh/tasks/69' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'

# 3) 完成后下载产物
curl -s -O 'https://<域名>/api/open/agent/dsh/tasks/69/artifacts/report.txt' \
  -H 'X-Agent-API-Key: agk_xxxxxxxx'
```

---

## 六、推荐接入模式（Agent 最佳实践）

1. **建 Key 用最小权限**：只授当前任务需要的 scope，`companyIds` 收敛到实际公司。
2. **写操作必带 Idempotency-Key**：断线重试不会造成重复数据。
3. **长任务用回调 + 轮询兜底**：配置 webhookUrl 收终态通知；兜底轮询 /tasks/:id。
4. **验签 + 去重**：回调方按 callbackSecret 验签，按 taskId+event 去重后处理。
5. **异步优先**：写接口通常同步返回业务结果，但建议拿到 taskId 后不阻塞，用回调/轮询确认最终状态。

---

## 七、Scope 速查表（按能力分组）

> 建 Key 时按实际需要勾选；未勾选的 scope 对应接口返回 403。

### 知识库

| Scope | 接口 |
|-------|------|
| `knowledge:write` | POST /knowledge/fill |
| `knowledge:read` | GET /knowledge、GET /knowledge/:id（详情含正文 content） |
| `knowledge:delete` | DELETE /knowledge/:id |

### 文章

| Scope | 接口 |
|-------|------|
| `article:create` | POST /articles |
| `article:read` | GET /articles、GET /articles/:id |
| `article:update` | PUT /articles/:id |
| `article:delete` | DELETE /articles/:id |
| `article:publish` | POST /articles/:id/publish |
| `article:generate` | POST /articles/generate、GET /articles/generate/:id、POST /articles/generate/:id/stop、POST /articles/generate/:id/resume |

### 发布与官网

| Scope | 接口 |
|-------|------|
| `publish:trigger` | POST /publish |
| `website:sync` | POST /website/sync |
| `blog:upload` | POST /blog |

### 视频

| Scope | 接口 |
|-------|------|
| `video:read` | GET /videos、GET /videos/templates、GET /videos/:id、GET /videos/:id/status、GET /videos/:id/play-url |
| `video:write` | POST /videos、POST /videos/analyze-article、POST /videos/:id/submit |
| `video:publish` | POST /videos/:id/publish |

### GEO 监测

| Scope | 接口 |
|-------|------|
| `monitor:write` | POST /monitor/tasks、POST /monitor/tasks/:id/execute |
| `monitor:read` | GET /monitor/analysis、GET /monitor/analysis/:id、GET /monitor/tasks、GET /monitor/tasks/:id、GET /monitor/tasks/:id/results |

### AI 检测

| Scope | 接口 |
|-------|------|
| `ai-detect:read` | GET /ai-detect/platforms、GET /ai-detect/tasks、GET /ai-detect/tasks/:id、GET /ai-detect/tasks/:id/source-overlap |
| `ai-detect:write` | POST /ai-detect/tasks、POST /ai-detect/tasks/:id/execute、POST /ai-detect/tasks/:id/retry、POST /ai-detect/tasks/:id/cross-validate |

### 图集

| Scope | 接口 |
|-------|------|
| `gallery:read` | GET /gallery、GET /gallery/:id、GET /gallery/match |
| `gallery:write` | POST /gallery/upload、GET+POST /gallery/categories |

### RAG 检索

| Scope | 接口 |
|-------|------|
| `rag:query` | POST /rag/query |

### DSH 沙箱

| Scope | 接口 |
|-------|------|
| `dsh:task` | POST /dsh/tasks、GET /dsh/tasks/:id、POST /dsh/tasks/:id/cancel、GET /dsh/tasks/:id/artifacts/*file |

### 蒸馏词

| Scope | 接口 |
|-------|------|
| `keyword:write` | POST /keywords/groups、POST /keywords/groups/:id/import、GET /keywords/groups、GET /keywords/groups/:id |

### SOP 运营进度

| Scope | 接口 |
|-------|------|
| `sop:read` | GET /sop/templates（SOP 模板 + 套餐定义，返 steps[].id 即 stepId；**`?companyId=X` 返回该公司生效模板 = 全局打底 + 公司专属定制合并**，每步带 `source: global/custom` 标记，不传返回全局默认模板） |
| `sop:write` | PUT /sop/progress（显式标记某客户某 SOP 步骤进度） |

### 产品库

| Scope | 接口 |
|-------|------|
| `product:write` | POST /products/upload、GET+POST /product-categories（多图 files[] 建产品草稿+分类查建；published 强制 false 不上站） |
| `product:read` | GET /products、GET /products/:id（产品草稿列表/详情） |

### 资料下载

| Scope | 接口 |
|-------|------|
| `download:write` | POST /downloads/upload（单文件 file 建资料草稿，category 缺省"资料"；published 强制 false 不上站） |
| `download:read` | GET /downloads、GET /downloads/:id（资料草稿列表/详情） |

> **公司级定制（2026-08-25）**：SOP 支持每账号下每公司独立改模板——全局默认模板（companyId=null）打底，公司专属定制（companyId=X）覆盖同名 stepKey，只影响该公司进度。标记进度时 stepId 必须是**该公司生效模板**里的 id（用 `GET /sop/templates?companyId=X` 查到的那份，含全局 id 与公司定制 id）。

> **使用场景**：无 autoCheckRule / 需人工确认的步骤，豆包/运营 Agent 执行完后调用打勾；有自动判定规则的步骤系统按业务指标自动判定，无需调用。`companyId` 必填并做归属校验（防越权），`stepId` 为 sop_step_templates.id（先 GET /sop/templates 拿到），`status` ∈ pending/doing/done/skipped，重复调用幂等覆盖。
>
> body 示例：
> ```json
> { "companyId": 95, "stepId": 12, "status": "done", "remark": "官网已上线" }
> ```

> **一键表格导入（推荐）**：智能体直接提交最终蒸馏词表格即可，无需填写核心词/地域词/前缀词/后缀词等词根维度（智能体会自行组词）。
>
> `POST /keywords/groups`，body：
> ```json
> {
>   "companyId": 95,
>   "name": "装修-智能体词表",
>   "keywords": [
>     { "keyword": "洛阳装修公司", "searchVolume": 800, "difficulty": 40, "intent": "高" },
>     { "keyword": "洛阳整装定制" }
>   ]
> }
> ```
> 响应：`{ id, name, coreWord, imported, skipped }`（coreWord 缺省取首个关键词，source 自动标记为 `agent_api`）。
> 向已有分组追加词走 `POST /keywords/groups/:id/import`（body 同上仅需 `companyId` + `keywords`）。**groupId 先查再写**：`GET /keywords/groups?companyId=X&page=1&pageSize=50`（支持 `?keyword=` 按名称搜索，返回 `{ total, page, pageSize, items[{ id, name, coreWord, status, articleCount, keywordCount }] }`）；需要组内词明细用 `GET /keywords/groups/:id?companyId=X`（返回词根维度 + `keywords[]` 分页 ≤100）。

### 无需 scope

| 接口 | 说明 |
|------|------|
| GET /tasks、GET /tasks/:id | 任务查询（可见范围按 Key 已有 scope 自动收窄） |
