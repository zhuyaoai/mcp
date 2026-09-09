# 逐曜AI GEO 全链路 MCP Server

> 让你的 AI 智能体直接驱动一套成熟的 GEO 获客系统：管理企业知识库、批量创作 AI 文章与口播视频、自动配图、沉淀客户案例，并追踪品牌在豆包、DeepSeek 等 9 大 AI 平台的可见度。

[![协议](https://img.shields.io/badge/protocol-MCP-blue)](https://modelcontextprotocol.io) [![接入](https://img.shields.io/badge/接入-免费-green)](https://www.zhuyaoai.com) [![传输](https://img.shields.io/badge/transport-streamableHttp-informational)](https://www.zhuyaoai.com/mcp/)

---

## 这是什么

**逐曜AI**（[www.zhuyaoai.com](https://www.zhuyaoai.com)）是一个 GEO（Generative Engine Optimization，AI 搜索优化）SaaS 系统。本仓库提供它的 **MCP Server 接入说明**：任何支持 MCP 协议的智能体（WorkBuddy、扣子、Claude、自研 Agent 等）都可以通过 60+ 个语义化工具，把"内容生产 → 官网同步 → AI 可见度监测"整条链路接进对话里。

能力域一览：

| 能力域 | 能做什么 |
|--------|----------|
| 知识库 / RAG | 企业知识 9 大模块写入（更新式 upsert）、语义检索 |
| 文章 | 批量 AI 生成（异步）、多平台格式改写、发布到自有官网 |
| 官网 | 知识库模块 + 托管站内容同步、宣传博客上传 |
| 图集 | 图片/视频上传（自动压缩打标）、按关键词语义选图、相册管理 |
| 案例库 | 客户案例增删改查（五要素结构化）、语义匹配自动选案例 |
| 视频 | 文章转口播稿 → TTS → 渲染出片（异步）、内置 24 首可商用 BGM（也可传音频直链，自动下载混入）、播放签名 URL |
| GEO 监测 | 创建品牌监测任务、采集 9 大 AI 平台回答、可见度/首推率分析 |
| AI 检测 | 单次诊断：多 AI 平台交叉对比、信源重叠分析 |
| 蒸馏词 | 行业关键词分组管理（地域词/长尾词体系） |
| DSH 沙箱 | 代码执行 / 数据分析异步任务 |
| 产品库 / 资料库 | 产品图组、文档资料上传建草稿 |
| SOP 运营 | 代运营 SOP 模板查询与进度打勾 |

## 快速开始

### 第 1 步：获取 Agent Key

1. 到 [www.zhuyaoai.com](https://www.zhuyaoai.com) 注册登录
2. 管理端「设置 → 智能体接入」创建 Agent Key（`agk_` 开头，**只展示一次**，妥善保存）
3. 首次使用前先在「公司管理」完善公司资料（简称/描述/主营行业），否则部分写操作会被拦截

### 第 2 步：接入

**方式 A：WorkBuddy 连接器**（推荐，开箱即用）

下载本仓库 Release 或 `connector/` 目录打包的 zip，在 WorkBuddy 开放平台提交/安装，用户侧填入 `agk_` Key 即可。凭证仅存本机，不经云端。

**方式 B：任意 MCP 客户端**

MCP 端点（streamableHttp，**带尾斜杠**）：

```
https://www.zhuyaoai.com/mcp/
```

客户端配置示例（请求头注入 Key，之后所有工具调用免传凭证）：

```json
{
  "mcpServers": {
    "zhuyaoai": {
      "type": "streamableHttp",
      "url": "https://www.zhuyaoai.com/mcp/",
      "headers": {
        "X-Agent-API-Key": "${ZHUYAO_AGENT_KEY}"
      },
      "timeout": 30000
    }
  }
}
```

**方式 C：REST 直调**

所有 MCP 工具都有等价 REST 端点（`https://www.zhuyaoai.com/api/open/agent/...`），认证头：

```
X-Agent-API-Key: agk_xxx
```

示例：

```bash
curl https://www.zhuyaoai.com/api/open/agent/me \
  -H "X-Agent-API-Key: agk_你的Key"
```

### 第 3 步：先调 `get_workflow`

不确定操作顺序时，**最先调用 `get_workflow` 工具**——它返回内置的工作流蓝图（步骤、依赖、必填参数），例如：

- **article_publish_flow**：写知识库 → AI 生成文章 → 发布官网 → 建 GEO 监测
- **asset_reference_flow**：图集选图 + 案例匹配 → 带素材生成文章
- **video_flow**：查模板 → 创建视频项目 → 生成口播稿 → 出片 → 取播放链接

## 关键设计

- **双模式鉴权**：工具参数传 `apiKey`，或连接级注入 `X-Agent-API-Key` 请求头（连接器模式），二选一
- **异步任务模型**：写操作（生成/出片/采集）立即返回 `taskId`，用 `task_status` / 对应状态工具轮询至终态——单请求 30s 上限，长任务不阻塞连接
- **幂等**：写操作支持 `Idempotency-Key`，断线重试不会重复创建
- **多租户隔离**：每个 Key 绑定授权公司清单，全部数据访问强制 companyId 收敛，越权一律 404/403
- **限流**：每 Key 每分钟 60 次；429 响应带 `errorCode` 细分（真限流 / 日配额 / 并发占满），按提示退避即可

## 目录结构

```
.
├── README.md                    # 本文件
├── connector/                   # WorkBuddy 连接器包（connector-meta/mcp.json/token-schema/icon/skills）
└── docs/                        # MCP 工具清单、REST 等价端点速查
```

## 安全与合规

- Agent Key 服务端只存哈希、泄露即吊销；接入侧只放环境变量，禁止写入代码/日志/对话
- 内容创作类能力均为生成辅助；发布到第三方平台（短视频/自媒体账号矩阵等）不在本开放接口范围内，请在逐曜AI 系统内操作
- 本接入免费，价值通过逐曜AI 系统订阅提供，不依赖任何平台分账

## 支持

- 官网：[www.zhuyaoai.com](https://www.zhuyaoai.com)
- 接入问题：注册后在管理端「智能体接入」页获取接入弹窗与文档
- MCP / REST 双协议能力等价，故障排查见 `docs/`

---

**License**: 文档与配置示例按 MIT 提供；逐曜AI 系统服务本身受服务条款约束。
