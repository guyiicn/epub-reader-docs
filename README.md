# EpubReader — 本地优先的多端电子书阅读系统

> 一套书库，三端同步。Windows 桌面 / Web / Android，连同一个自托管服务器，带 Z-Library 找书。

## 系统架构

```
                    ┌─────────────────────────────────┐
                    │     epub-reader-server (Ktor)     │
                    │         us.guyii.net              │
                    │  · 书库存储 (SQLite + 文件)        │
                    │  · JWT 认证 + 设备管理             │
                    │  · 增量同步 (since + cursor)       │
                    │  · WebSocket 实时推送              │
                    │  · Z-Library 找书 (Telegram Bot)   │
                    │  · 阅读统计 / 标签 / 存储配额       │
                    └──────────┬────────────────────────┘
                               │ openapi.yaml 契约
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
          ┌──────────────┐ ┌────────┐ ┌──────────┐
          │ Win + Web    │ │ Android│ │ (未来 iOS)│
          │ Tauri+React  │ │ Compose│ │          │
          │ foliate-js   │ │ Readium│ │          │
          └──────────────┘ └────────┘ └──────────┘
```

## 仓库

| 仓库 | 说明 | 语言 |
|------|------|------|
| **[epub-reader](https://github.com/guyiicn/epub-reader)** *(私有)* | Umbrella 仓库（git submodule 串全部 + 契约管理） | — |
| [yueduqi](https://github.com/guyiicn/yueduqi) *(私有)* | Windows 桌面 + Web 客户端（Tauri + React + foliate-js） | TS / Rust |
| [epub_android](https://github.com/guyiicn/epub_android) *(私有)* | Android 客户端（Kotlin Compose + Readium + Room） | Kotlin |
| [epub_reader_server](https://github.com/guyiicn/epub_reader_server) *(私有)* | 同步服务器 + 找书后端（Ktor + SQLite + Docker） | Kotlin |
| **epub-reader-docs** *(本仓库，公开)* | API 契约 + 集成指南 | — |

## 核心功能

- 📚 **书库管理**：导入/封面/分类/搜索/进度，跨端同步
- 📖 **多格式阅读**：EPUB / MOBI / AZW3 / PDF / TXT / Markdown
- 🖊 **标注**：高亮 / 书签 / 笔记，跨端同步
- 🔍 **找书**：Z-Library 1800 万册搜索 → 一键下载入库 → 自动同步全端
- 🔊 **TTS 朗读**：系统原生语音（可插拔云端 TTS）
- 🌐 **OPDS 书源**：浏览公共电子书站下载
- 🔄 **实时同步**：WebSocket 推送 + 增量 pull + 冲突解决（LWW）

---

## 接入文档（给客户端开发者）

本仓库提供完整的 API 契约和集成指南：

| 文件 | 说明 |
|---|---|
| [`docs/CLIENT_INTEGRATION_GUIDE.md`](./docs/CLIENT_INTEGRATION_GUIDE.md) | 自包含的接入指南：90 秒 curl 烟雾、鉴权流程、REST + WebSocket 契约、同步状态机、边界 case |
| [`docs/server/openapi.yaml`](./docs/server/openapi.yaml) | OpenAPI 3.0.3 形式规范，包括 V2 全部端点和 schema |
| [`docs/find-book-api-actual.md`](./docs/find-book-api-actual.md) | Z-Library 找书端点的实际实现文档 |

## 服务器

生产：`https://us.guyii.net`（默认部署，单用户）。如果要自部署，源码在 [epub_reader_server](https://github.com/guyiicn/epub_reader_server) *(私有)*。

## 版本

文档版本与服务器 `info.version` 对齐。当前：**1.3.0**（含 Phase E V2 + Phase S 找书）。

## 凭证占位

文档示例里的 `<your-password>` 是占位符，请替换为你自己服务器的 admin 密码。**不要把真密码提交到任何 public 仓库。**

## 反馈 / 问题

如发现协议歧义或文档与服务器实际行为不符，欢迎开 Issue。
