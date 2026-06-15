# Nitter RSS 抓取 X/Twitter 帖子

## 元数据

- **类型**: API Guide
- **适用场景**: 以时间倒序（chronological）批量抓取任意 X/Twitter 账号的帖子原文，绕过 X API 的热度排序限制
- **前置条件**: 无（无需 API key、无需登录）
- **创建日期**: 2026-06-15
- **最后更新**: 2026-06-15

---

## 问题背景

X/Twitter 官方 API 对未登录/访客模式下返回的是热度排序（Top）的数据，而非时间倒序（Latest）。使用 `UserTweets` GraphQL 查询返回缓存的热门推文，可能滞后数月。Nitter RSS 提供纯时间倒序且无需认证的替代方案。

## 抓取方法

### 核心原理

Nitter 是一个轻量级 X/Twitter 前端，其 `/rss` 端点返回标准 RSS 2.0 格式的帖子列表，按发布时间倒序排列。格式简单，可直接用 `webfetch` 工具抓取。

### URL 格式

```
https://nitter.net/<username>/rss
```

### webfetch 示例

```
webfetch("https://nitter.net/quarktalksss/rss", format="text")
```

### 解析规则

RSS 中每个 `<item>` 包含：

| XML 元素 | 含义 |
|---|---|
| `<title>` | 帖子文本（纯文本，不含 HTML） |
| `<pubDate>` | 发布时间（GMT，格式如 `Mon, 15 Jun 2026 04:14:08 GMT`） |
| `<guid>` | 唯一 ID，包含 tweet ID 数字 |
| `<link>` | 帖子 permalink |
| `<description>` | 完整帖子内容（含 HTML、引用推文、图片） |

时间范围过滤：将 `pubDate` 解析为时间戳即可判断是否在目标窗口内。注意 RSS 默认返回最近 15-50 条推文，足够覆盖最近 48h。

### 多账号并行抓取

```python
accounts = ["user1", "user2", "user3"]
for acct in accounts:
    r = requests.get(f"https://nitter.net/{acct}/rss")
    # 解析 RSS -> 提取 <item> -> 按 <pubDate> 过滤 -> 输出
```

## Nitter 实例

当 `nitter.net` 不可用时，可尝试以下替代实例：

- `nitter.net`（主站）
- `nitter.poast.org`
- `nitter.privacydev.net`

如所有 Nitter 实例均不可用，可降级使用以下备用方案（时效性较差）：

1. **Brave Search**: `https://search.brave.com/search?q=site:x.com+<username>` — 返回索引过的 X 页面，含 tweet 文本和日期
2. **X GraphQL TweetResultByRestId**: 需先知道 tweet ID，通过 `https://api.x.com/graphql/8CEYnZhCp0dx9DFyyEBlbQ/TweetResultByRestId` 查询

## 已知限制

- Nitter 实例可能间歇性不可用（503/502 错误）
- RSS 仅返回最近 ~50 条推文，对高频账号可能不够
- 部分 Nitter 实例存在请求频率限制
- 引用推文（reply/quote）和转推在 RSS 中有标记

## 与 X API 对比

| | X GraphQL API (访客) | Nitter RSS |
|---|---|---|
| 排序方式 | 热度排序 | 时间倒序 ✅ |
| 需要认证 | 需 guest token | 无需 ✅ |
| 数据完整性 | 仅热门推文 | 全部推文 ✅ |
| 可靠性 | 稳定 | 实例可能宕机 |
| 请求频率 | 易触发 403 | 较宽松 |
