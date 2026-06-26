# epub-reader-docs

公开的 EpubReader 自托管同步服务器**接入文档**。给写第三方客户端（Web / Windows / macOS / iOS / 其他）的人提供 API 契约和集成指南。

仓库内容：

| 文件 | 说明 |
|---|---|
| [`docs/CLIENT_INTEGRATION_GUIDE.md`](./docs/CLIENT_INTEGRATION_GUIDE.md) | 自包含的接入指南：90 秒 curl 烟雾、鉴权流程、REST + WebSocket 契约、同步状态机、边界 case（含 Phase S 找书 Addendum） |
| [`docs/server/openapi.yaml`](./docs/server/openapi.yaml) | OpenAPI 3.0.3 形式规范，包括 V2 全部端点和 schema |
| [`docs/find-book-api-actual.md`](./docs/find-book-api-actual.md) | Phase S 找书 / 在线下载的逐字段客户端契约（`/search` + `/search/download`） |

## 服务器

生产：`https://us.guyii.net`（默认部署，单用户）。如果要自部署，源码不在本仓库——本仓库**只放协议文档**。

## 版本

文档版本与服务器 `info.version` 对齐。当前：**1.3.0**（在 Phase E V2 基础上新增 Phase S：找书 `/search` + 在线下载 `/search/download`，可选 LLM 重排）。

V2 内容：WebSocket 推送、标签、阅读状态、阅读会话、统计、存储配额、跨引擎 Locator 约定。

## 凭证占位

文档示例里的 `<your-password>` 是占位符，请替换为你自己服务器的 admin 密码。**不要把真密码提交到任何 public 仓库。**

## 反馈 / 问题

如发现协议歧义或文档与服务器实际行为不符，欢迎开 Issue。
