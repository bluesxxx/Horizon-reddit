# RSS Cache Fetcher — 每天 22:00 执行

每天 22:00 抓取 8 个 subreddit 的 Reddit RSS 热帖，解析为 JSON 并缓存到本地。设计目标：即使 VPN 断开，也能使用 ≤24h 前的缓存数据生成日报（9:00 日报任务自带实时抓取，本地缓存仅作第三道兜底）。

## Step 1: 拉取 8 个 subreddit RSS

用 `web_fetch`（timeout=15s）依次抓取以下 URL（**走 Cloudflare Worker 反代**，因为国内网络封锁 reddit.com，直连必失败）。相邻请求等 4s。

```
https://reddit.77666677.xyz/r/freightforwarding/hot/.rss
https://reddit.77666677.xyz/r/logistics/hot/.rss
https://reddit.77666677.xyz/r/importing/hot/.rss
https://reddit.77666677.xyz/r/supplychain/hot/.rss
https://reddit.77666677.xyz/r/AliBaba/hot/.rss
https://reddit.77666677.xyz/r/CustomsBroker/hot/.rss
https://reddit.77666677.xyz/r/FulfillmentByAmazon/hot/.rss
https://reddit.77666677.xyz/r/ecommerce/hot/.rss
```

反代说明：Worker（reddit-rss-proxy）已在海外节点请求 Reddit 并伪装浏览器 UA，无需再自定义请求头。返回的仍是 Reddit 原始 Atom XML 格式，Step 2 解析逻辑不变。

如果单源超时、429 或 502（Worker 上游失败）：等 12s 后重试最多 2 次。仍失败则跳过该 sub。

**禁止直连 reddit.com 或其镜像**（r.jina.ai、rsshub 等均被封锁），只使用上述反代地址。

## Step 2: 解析 XML 为 JSON 数组

对每个 RSS XML，提取 `<entry>` 标签内的：
- `<title>` → title
- `<link href="...">` → link
- `<updated>` 或 `<pubDate>` → pubDate
- `<category term="...">`（可能有多个）→ tags[]

输出格式：`[{ title, link, pubDate, tags: [] }, ...]`

## Step 3: 合并去重 + 过滤

对每个 subreddit：
1. 读取现有缓存 `D:\Horizon\data\cache\{subreddit}.json`（不存在则为 `[]`）
2. 新数据按 link 去重后合并到数组头部（最新在前）
3. 过滤掉 pubDate 早于 72 小时的条目（保留最近 3 天）
4. 每个 subreddit 缓存上限 200 条（旧条目自动裁掉）
5. 写回 `D:\Horizon\data\cache\{subreddit}.json`

## Step 4: 更新 manifest

写完 8 个文件后，复写 `D:\Horizon\data\cache\manifest.json`：
```json
{
  "last_fetch": "2026-09-01T15:00:00+08:00",
  "total_once": 187,
  "counts": {
    "freightforwarding": 25,
    "logistics": 25,
    ...
  }
}
```

## Step 5: 报告

在最终回复中报告：上线时间、各 sub 缓存条数、新增条数。如果有 sub 整体失败（全 3 次 retry 都 fail），也注明。

## 关键约束

- 所有路径使用 `D:\Horizon\data\cache\` 绝对路径
- JSON 文件 UTF-8 编码、pretty print（indent=2）
- 如果全部 8 个 sub 都失败（网络问题），报告失败原因，不修改 manifest
- 来源链接保持 Reddit 原始 URL（https://www.reddit.com/r/.../comments/XXX/...）
