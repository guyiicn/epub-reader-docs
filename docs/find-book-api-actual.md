# 找书 API — 服务器实现实况

> **给客户端 agent 的契约文档。** 写在生产服务器实测之后，跟 `find-book-api-spec.md` 配套读：以本文为准，spec 是设计意图，**这里是实际落地的样子**。
>
> 服务器：`https://us.guyii.net`
> 路径前缀：`/api/v1`
> 鉴权：`Authorization: Bearer <accessToken>` + `X-Device-Id: <deviceId>`（跟 `/books`、`/progress` 完全一致）
> 日期：2026-06-26
> 服务器版本：跟 `epub_reader_server` git HEAD（commits `6dc9ca9` + `65f5b8d` + `ee8cb80`）

---

## 0. 跟 `find-book-api-spec.md` 的差异 — 一张表看完

| spec 项 | 实际状态 | 备注 |
|---|---|---|
| `GET /api/v1/search?q=&page=` | ✅ 实现 | 请求字段一致 |
| `POST /api/v1/search/download` | ✅ 实现 | 请求字段一致 |
| `SearchResult` 顶层字段 | ⚠️ 微调 | `coverUrl` 字段名保留；新增可选 `?preference=` query 参数（LLM rerank） |
| `SearchBook` 字段 | ⚠️ 新增 `year` | spec 没列 year；实际 Bot 输出有，给客户端展示用 |
| `alreadyInLibrary` | ✅ 实现 | 用 source_command 精确匹配 + (title, author) 大小写不敏感模糊匹配 |
| `bookCommand` 自包含 | ✅ 是 | 客户端只需保存 bookCommand 即可，不需要 sessionToken |
| 下载状态码 201/200/202 | ⚠️ 暂只有 200/201 | 没实现 202 异步 jobId（>50MB CDN fallback 不做） |
| `GET /api/v1/search/download/status/{jobId}` | ❌ 未实现 | 没必要，没有异步路径 |
| 大文件 (>50MB) | ⚠️ 直接 413 | 没接 Playwright + wget；超过 50MB 返 `413 PayloadTooLarge` |
| 文件 >100MB | ✅ 413 | 同 spec |
| Z-Library 不可达 | ✅ 502 | spec 一致 |
| 限流（搜索 10/min、下载 20/h） | ❌ 未实现 | S-3 todo；目前无限流 |
| 4xx Validation 错误 shape | ✅ Problem JSON (RFC 7807 简化版) | 跟其他端点一致 |
| `books.source_command` 列 | ✅ V3 migration 已上线 | 内部，客户端不直接读 |
| 服务器内部 Bot 翻页状态 | ✅ 串行化 | 全局 Mutex，多个搜索请求排队 |
| WebSocket `book.updated` 广播 | ✅ 实现 | 沿用现有 `invalidate{table:"books", ...}` |

**结论**：客户端按 spec 写的 `FindBookApi`/`FindBookPage` **基本无需改**，只需要：
1. **新增可选 query 参数** `?preference=...` 触发 LLM rerank（用户输入"epub 优先 / 中文 / 单本不要合集"等自然语言）
2. **SearchBook 多一个 `year` 字段**（可选，展示用）
3. **错误处理简化**：202/jobId/CDN fallback 这套不会触发，遇 >50MB 文件直接 413

---

## 1. `GET /api/v1/search`

### 请求

```
GET /api/v1/search?q=<keyword>&page=<n>&preference=<text>
Authorization: Bearer <accessToken>
X-Device-Id: <deviceId>
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `q` | string | 是 | 搜索关键词；1-200 字符；trimmed |
| `page` | int | 否 | 默认 1；范围 1-20 |
| `preference` | string | 否 | **新增**：自由文本，触发 LLM rerank（中文/英文都行）。例：`"epub 优先，单本不要合集"` |

### 响应 200

```json
{
  "query": "漫长的告别",
  "total": 175,
  "page": 1,
  "hasNext": true,
  "hasPrev": false,
  "books": [
    {
      "index": 1,
      "title": "漫长的告别 = THE LONG GOODBYE",
      "author": "[美] 雷蒙德 · 钱德勒 (Raymond Chandler) 著 ; 宋碧云 译",
      "year": "2017",
      "language": "Chinese",
      "format": "epub",
      "size": "410 KB",
      "bookCommand": "/book_aJO7axr7Zj3",
      "coverUrl": "https://z-library.sk/book/JO7axr7Zj3?remix_userid=...&remix_userkey=..."
    }
  ],
  "alreadyInLibrary": ["/book_aJO7axr7Zj3"]
}
```

### SearchBook 字段说明

| 字段 | 类型 | 备注 |
|---|---|---|
| `index` | int | 当前页内序号（rerank 后会被重排为 1..N） |
| `title` | string | 原文，可能含 emoji / 标点 |
| `author` | string | 原文，常含 "著 ; 译" 等中文后缀 |
| `year` | string\|null | **spec 没列；实际有**。例 "2017"。Bot 上游可能缺这行所以 nullable |
| `language` | string\|null | 例 "Chinese"、"English" |
| `format` | string | 小写：`"epub"` / `"mobi"` / `"pdf"` / ... |
| `size` | string | 人类可读："410 KB" / "6.19 MB" |
| `bookCommand` | string | `/book_xxxxxxxx`；客户端**透明保存**，下载时原样回传 |
| `coverUrl` | string\|null | Z-Library 直链（含 `remix_userid` / `remix_userkey` 鉴权参数）。**可能跨网络访问不到**——客户端如果展示不出来就 fallback 到占位图 |

### alreadyInLibrary

`string[]`，每项是 books 数组里某个 item 的 `bookCommand`。

服务器使用**两路并集**判定：
1. **精确**：`books.source_command` 精确等于该 bookCommand（之前通过 `/search/download` 下载过）
2. **模糊**：现有用户书库里某本书的 `(lower(title), lower(author))` 等于本次搜索结果的 `(title, author)`（用户**之前手动 SAF 上传过**同一本书）

第 2 项有助于识别"我手动传过同样的书但服务器没记到 source_command"——这种情况客户端也应显示"已在库中"。

### 错误响应

| 状态码 | title | 说明 |
|---|---|---|
| 400 | Validation failed | q 缺失 / q > 200 字 / page 不在 1-20 |
| 401 | Unauthorized | JWT 失效或缺失 |
| 403 | Unknown device | X-Device-Id 不属于当前用户 |
| 502 | Z-Library bridge error | Bot 通信失败 / Telethon session 失效；`detail` 含原始错误描述 |

**没有 429**（限流未实现）。

---

## 2. `POST /api/v1/search/download`

### 请求

```
POST /api/v1/search/download
Authorization: Bearer <accessToken>
X-Device-Id: <deviceId>
Content-Type: application/json

{
  "bookCommand": "/book_aJO7axr7Zj3",
  "title": "漫长的告别 = THE LONG GOODBYE",
  "author": "Raymond Chandler"
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `bookCommand` | 是 | 必须 `/book_` 开头 |
| `title` | 是 | 用作 books 表 title 列；服务器会 trim + 截到 512 字符 |
| `author` | 否 | 默认空串；同样 trim + 截 512 |

### 响应

| 状态码 | 说明 | Body |
|---|---|---|
| **201** | 新下载并入库 | 完整 [Book DTO](#book-dto)（contentHash, fileSize, coverHash, etc.） |
| **200** | dedup 命中（同 source_command 已存在） | 完整 Book DTO（**快路径，~13ms 返回**，不走 Bot） |
| 400 | Validation failed | bookCommand 非 `/book_` 开头 / title 空 |
| 401 | Unauthorized | |
| 403 | Unknown device | |
| 410 | Gone | Bot command 过期/失效（搜索结果上下文已被替换） |
| 413 | Payload Too Large | 文件 > 50MB（Bot 限制）或 > 100MB（服务器硬上限） |
| 500 | Ingest failed | 文件下来了但写入 DB 失败（极少） |
| 502 | Bad Gateway | Bot 不响应、Telethon session 死、网络问题 |

### Book DTO

复用现有 `/books` 端点的 schema（不重新发明）：

```json
{
  "id": "019f01d0-f293-77c7-b8eb-8d9a85c897b7",
  "userId": "b496e3bb-a805-4b09-ba39-5a39d897718e",
  "contentHash": "75d6a229a3546f2bcf191dbc4ea91f035ce0f0e5cb2f953f8ca120531f816a98",
  "title": "漫长的告别 = THE LONG GOODBYE",
  "author": "Raymond Chandler",
  "format": "EPUB",
  "fileSize": 419908,
  "totalChapters": 0,
  "coverHash": "22f123e28080da208c6db05d78bfb15a0ed729a1280641bca24ba7d4af9f9aea",
  "addedAt": 1782441898643,
  "addedBy": "<deviceId>",
  "updatedAt": 1782441898643,
  "deletedAt": null,
  "readingStatus": "NONE"
}
```

**format 大写**：服务器存 `"EPUB"`/`"MOBI"`/`"PDF"`/`"TXT"`；跟搜索结果里的小写 `format`（Bot 原文）不同。

### 客户端无需自己下文件

下载成功后服务器已**写入用户书库**：
1. 文件存到 `/data/books/<contentHash>.epub`
2. cover 提取到 `/data/books/<contentHash>.cover.jpg`
3. books 表插行（含 `source_command = bookCommand`）
4. AuditLog 写 `POST_BOOK`
5. **WebSocket invalidate(books)** 广播——所有其他登录的客户端会收到 invalidate，自动 pull /books 看到新书

也就是说**用户在 Web 端搜+下载一本书，Android 端的 app 1 秒内自动出现这本**（沿用现有 V2 同步管道）。

---

## 3. LLM rerank（可选）

### 触发

`GET /search?q=...&preference=<自由文本>`

### 例

```bash
# 中文 preference
curl "...&preference=epub%20%E6%A0%BC%E5%BC%8F%E4%BC%98%E5%85%88%EF%BC%8C%E5%8D%95%E6%9C%AC%E4%B8%8D%E8%A6%81%E5%90%88%E9%9B%86"

# 英文 preference
curl "...&preference=prefer%20epub%2C%20avoid%20collections%2C%20Chinese%20translation%20preferred"
```

### 行为

1. 服务器拿到正则解析后的 5 本书
2. 调用 `OPENAI_BASE_URL` + `OPENAI_MODEL`（默认 `gpt-4o-mini`）做重排 + 过滤
3. LLM 返回 `{order: [...], dropped: [...]}` — 服务器据此**重新排列 + 删除不相关的**
4. response 里 `books` 数组是过滤+重排后的结果（length 可能 < 5）

### LLM 失败 fallback

LLM 超时 / 错误 / 解析失败 → **静默退回原顺序**，不影响主流程。客户端看到的就是 baseline 结果。

### 延迟

baseline ~5-7s（Bot 才是瓶颈）；rerank 加 0.3-0.8s（取决于 OpenAI 当时压力）。

### 客户端推荐用法

- 给用户提供输入框："想要什么样的版本？（可选）"
- 默认不传 preference
- 用户输入后透传给 `?preference=...`

---

## 4. 关键约束 / 行为细节

### 4.1 **每次搜索都是全新 Bot 会话**

服务器内部用全局 Mutex 串行化所有 Bot 调用。**多用户并发搜索会排队**（当前单用户场景影响不大）。

`page > 1` 的实现方式：发新关键词搜索 → 然后 click next button N-1 次。所以**翻页延迟 ≈ N × 5 秒**。建议客户端 UX 鼓励"搜得准"而不是"翻很多页"。

### 4.2 **Bot command 时效**

`bookCommand` 在 Bot 内部对应一个对话状态。**短时间内有效**（数小时内通常 OK），但**长期保存可能失效**（返 410 Gone）。

客户端如果做"收藏待读"等长期保存的功能，**应该保存 (title, author)** 而不是 bookCommand——下次重新搜索拿新的 command。

### 4.3 **下载即入库**

`POST /search/download` **不是**单纯的"代理下载"——它会**永久写入用户书库**。客户端不应该把它当作"试听"按钮，UI 文案应明示"加入书库"。

### 4.4 **dedup 是按 contentHash**

即使两个不同 bookCommand 对应同一本书（同一 EPUB 文件），第二次下载会被 SHA-256 dedup 识破，文件不会重复存盘，但**第二个 source_command 会被回填到现有行**（多个 bookCommand 都标记为"已在库中"）。

### 4.5 **WebSocket 推送**

下载入库后服务器广播：

```json
{
  "type": "invalidate",
  "table": "books",
  "cursor": 1782441898643,
  "originDeviceId": "<触发下载的 deviceId>",
  "targetId": "<新 book id>",
  "ts": 1782441898643
}
```

客户端拿到后，按现有 V2 sync 协议拉 `GET /books?since=cursor-1`。**originDeviceId 是发起下载的设备**——发起方自己应该忽略这条 invalidate（它在 200 响应里已经拿到 Book DTO）。

---

## 5. 客户端实现 checklist（按 spec 写的话需要哪些微调）

- [x] `GET /search` 请求路径 / 参数 / 鉴权 — 完全一致
- [x] `POST /search/download` 请求 body — 完全一致
- [ ] `SearchBook` 增加可选 `year: string?` 字段
- [ ] response 里**不要**指望 sessionToken / nextPageToken — 没有；用 `hasNext` boolean 判定能否翻页
- [ ] 错误 410（command 过期）路径：建议提示用户"重新搜索"
- [ ] 错误 413（文件太大）：提示"该文件超出免费 Bot 限制，无法下载"
- [ ] 可选：UI 加 preference 输入框，触发 LLM rerank
- [ ] 可选：缓存 alreadyInLibrary，显示"已在库中"badge
- [ ] **不要**实现 `GET /search/download/status/{jobId}` — 没这端点
- [ ] **不要**期待 202 响应 — 全部走同步路径

---

## 6. 调试用 curl 模板

```bash
# 取 token + 注册 device（首次）
TOK=$(curl -s https://us.guyii.net/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"<your-password>","deviceName":"my-client","platform":"WEB"}' \
  | jq -r .accessToken)
curl -X POST https://us.guyii.net/api/v1/auth/devices \
  -H "Authorization: Bearer $TOK" -H 'Content-Type: application/json' \
  -d '{"deviceId":"<your-device-uuid>","name":"my-client","platform":"WEB"}'

DEV="<your-device-uuid>"

# 搜索
curl -s -H "Authorization: Bearer $TOK" -H "X-Device-Id: $DEV" \
  "https://us.guyii.net/api/v1/search?q=$(printf '%s' '漫长的告别' | jq -sRr @uri)" | jq

# 搜索 + LLM rerank
PREF=$(printf '%s' 'epub 单本不要合集' | jq -sRr @uri)
curl -s -H "Authorization: Bearer $TOK" -H "X-Device-Id: $DEV" \
  "https://us.guyii.net/api/v1/search?q=...&preference=$PREF" | jq

# 下载入库
curl -s -X POST https://us.guyii.net/api/v1/search/download \
  -H "Authorization: Bearer $TOK" -H "X-Device-Id: $DEV" \
  -H 'Content-Type: application/json' \
  -d '{"bookCommand":"/book_aJO7axr7Zj3","title":"漫长的告别","author":"Raymond Chandler"}' \
  | jq
```

---

## 7. 已知 todo（服务器侧，跟客户端无关但提醒）

- 限流（spec §4 的 10/min 搜索、20/hour 下载） — 未实现
- audit log 写 `SEARCH` / `DOWNLOAD_BOOK_FROM_SEARCH` op — 需要 V4 migration 扩 `sync_log.operation CHECK`
- `GET /search/covers/{hash}` 封面代理 — 现在客户端直接访问 Z-Library URL；某些网络下不可达
- ~~OpenAPI 同步~~ — ✅ 已完成：`docs/server/openapi.yaml` v1.3.0 已加 `/search` + `/search/download` 两个端点及 schema；`CLIENT_INTEGRATION_GUIDE.md` 已加 Phase S Addendum

这些都不影响契约，等 client agent 那边联调没问题后再补。

---

## 8. 联系

发现协议歧义 / 端点行为跟本文档不符 → 在 server repo 开 issue 或直接告诉 Guyi。生产服务器问题（502 不停 / Bot 卡死）也一并报，我会重启 Telethon session。
