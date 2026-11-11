# Issue #5 调研：MediaCrawler 抖音「创作者主页」模式能不能抓全对标作者的文案与数据

> 结论基于本机已安装的 MediaCrawler 源码（`~/.mediacrawler`）与已有爬取产物（`~/.mediacrawler/data/douyin/jsonl/`）。
> **本次未完成真实爬取**：`browser_data/` 下两个用户目录都是空的，无可用登录态，跑起来会卡在扫码。按任务要求不卡在扫码，改为源码 + 已有产物核对。
> 每条结论都标注来源；没验证到的明确写「未验证」。

---

## 0. 一句话结论

| 问题 | 结论 |
|---|---|
| 1. 从作品链接拿作者 | **能，但要两步**：短链 302 解析出 aweme_id → 调 detail 接口读 `author.sec_uid`。当前 store 把作者匿名化了，**不能**靠"先落盘再读文件"绕过去 |
| 2. 创作者模式一次拿全 / 限 30 条 | **能拿全（翻页到底，不限条数）**；**限 30 条做不到**——`--crawler_max_notes_count` 在 creator 模式完全不生效，必须改代码 |
| 3. 字段 | 文案/点赞/评论/收藏/时间/链接/封面 **全有**；**播放量 play_count 拿不到**（store 层直接丢弃） |
| 4. 登录态 / headless | 登录态可持久化到 `browser_data/`，但**当前是空的 → 首次必须 Lin 扫码**；扫码后能 headless 无人值守。有效期**未验证** |
| 5. 风控 | 源码里有多处明确风控信号（blocked / Argus 403 / 滑块）；**限流阈值与封号征兆未验证** |

---

## 1. 从一条作品链接怎么拿到作者

### 1.1 短链解析（已验证可用，无需登录）

- `DouYinClient.resolve_short_url()` — `media_platform/douyin/client.py:376-399`：`follow_redirects=False` 发 GET，取 302 的 `Location` 头。
- `parse_video_info_from_url()` — `media_platform/douyin/help.py:101-138`：正则 `/video/(\d+)` 抽 aweme_id，也支持 `modal_id` 参数形式和纯数字 ID。

**实测**（curl，无登录态）：

```
https://v.douyin.com/drIPtQ_WPWY/
→ 302 → https://www.iesdouyin.com/share/video/7525538910311632128/?region=CN&mid=...&did=MS4wLjABAAAA...
```

跳转 URL 里的 `MS4wLjABAAAA...` 是 **did（设备 id）不是 sec_uid**，别误取。`/share/video/<id>/` 也能被 `/video/(\d+)` 正则匹配到，所以短链这条路对 MediaCrawler 是通的。

### 1.2 作者标识只能从 detail 接口拿

- 创作者模式只吃 `sec_user_id`：`parse_creator_info_from_url()` — `help.py:141-164`，只认 `/user/<sec_uid>` 形式的 URL 或 `MS4wLjABAAAA` 前缀的裸 ID。
- 作品 detail 接口 `/aweme/v1/web/aweme/detail/` — `client.py:228-250`，返回 `aweme_detail`，作者信息在其 `author` 子对象里（`store/douyin/__init__.py:96` 就是取 `aweme_item.get("author", {})`）。

### 1.3 关键坑：本仓库把作者身份匿名化了

- `store/douyin/__init__.py:104-105`：`creator_hash = anonymize_user_id(user_info.get("uid"))`、`nickname = mask_nickname(...)`。
- `tools/user_hash.py`：uid → sha256 截断 16 位；昵称 → `林***I`。
- `store/douyin/__init__.py:155-157`：`save_creator()` 是**空实现**，创作者资料（含 sec_uid/粉丝数）根本不落盘。
- `tests/test_douyin_no_user_info.py` 就是专门断言 `sec_uid` 等敏感字段不得出现在任何存储值里。

**推论**：要做「贴作品链接 → 爬作者全部作品」，必须在同一次运行内、落盘之前从 `aweme_detail.author.sec_uid` 取值传给 creator 模式；**不能**走 "先跑 detail 存 jsonl，再读 jsonl 找作者" —— jsonl 里只有 `creator_hash`。

- **未验证**：`aweme_detail.author.sec_uid` 在真实响应里的字段名。现有唯一证据是 test fixture（`tests/test_douyin_no_user_info.py:66` 的 mock），不是真实接口响应。验证它必须带登录态调一次 detail 接口。

---

## 2. 创作者模式能不能一次拿全？条数怎么限？

### 2.1 拿全：能，而且是无上限翻页

`DouYinCrawler.get_creators_and_videos()` — `core.py:273-297`：

```python
all_video_list = await self.dy_client.get_all_user_aweme_posts(sec_user_id=user_id, callback=self.fetch_creator_video_detail)
```

`get_all_user_aweme_posts()` — `client.py:361-374`：`while posts_has_more == 1`，每页 `count: 18`（`client.py:350-358`），`max_cursor` 递进，**直到 `has_more == 0`**。

所以：**一次命令就能拿到该作者全部作品**，不需要循环调用。

### 2.2 限 30 条：做不到，参数是假的

- `--crawler_max_notes_count` 定义在 `cmd_arg/arg.py:287-294`，默认取 `config.CRAWLER_MAX_NOTES_COUNT`。
- `config/base_config.py:104`：`CRAWLER_MAX_NOTES_COUNT = 15`。
- **全仓只有 `search()` 用它**：`core.py:132-142`（`dy_limit_count = 10` 固定页大小，`while (page - start_page + 1) * 10 <= CRAWLER_MAX_NOTES_COUNT`）。
- `grep -rn CRAWLER_MAX_NOTES_COUNT media_platform/douyin/ main.py` 只命中 `core.py:133/134/142`，全部在 search 分支。

**结论**：creator 模式对 `--crawler_max_notes_count` 完全无感。作者有 500 条作品就翻 28 页、对 500 条逐个调 detail 接口（`core.py:299-310`），既不符合「最近 30 条」的需求，也是最容易踩风控的跑法。

**要限 30 条必须改代码**，两个可选点：
1. `client.py:361` 的 `get_all_user_aweme_posts` 加 `max_count` 参数，`result` 或累计条数到 30 就 break（省接口调用，推荐）。
2. `core.py:299` 的 `fetch_creator_video_detail` 截断——但这只省了 detail 调用，作品列表页还是会翻到底。

- **未验证**：`/aweme/v1/web/aweme/post/` 是否按发布时间倒序返回。源码没有任何排序参数，抖音 web 端该接口默认时间倒序，但**没有在本仓库内找到断言**，不写死。

---

## 3. 每条作品能拿到哪些字段

### 3.1 实测落盘字段（已有产物，可直接采信）

`~/.mediacrawler/data/douyin/jsonl/detail_contents_2026-11-10.jsonl` 首条 keys：

```
aweme_id, aweme_type, aweme_url, collected_count, comment_count, cover_url,
create_time, creator_hash, desc, last_modify_ts, liked_count,
music_download_url, nickname, note_download_url, share_count, source_keyword,
title, video_download_url
```

字段来源：`store/douyin/__init__.py:98-117`（`update_douyin_aweme`）。

### 3.2 需求字段对照

| 需求字段 | 结论 | 实际字段名 / 位置 |
|---|---|---|
| 文案全文 desc | ✅ 能 | `desc`（同时冗余到 `title`），**全文不截断** |
| 播放 play_count | ❌ **拿不到** | 全仓 `media_platform/douyin`、`store/douyin`、`model/` 搜 `play_count` **零命中**；store 只取 statistics 的 digg/collect/comment/share 四项（`store/douyin/__init__.py:106-109`） |
| 点赞 digg_count | ✅ 能 | `liked_count`（**字符串**，需 `tonumber`） |
| 评论 comment_count | ✅ 能 | `comment_count`（字符串） |
| 收藏 collect_count | ✅ 能 | `collected_count`（注意输出名多了个 d；字符串） |
| 发布时间 create_time | ✅ 能 | `create_time`，**秒级**时间戳 |
| 作品链接 aweme_url | ✅ 能 | `aweme_url` = `https://www.douyin.com/video/{aweme_id}` |
| 封面 cover | ✅ 能 | `cover_url`（`extract_cover_url`，`store/douyin/__init__.py:112`） |
| 作者 sec_uid | ❌ 拿不到 | 只有 `creator_hash`（匿名哈希） |
| 作者昵称 | ⚠️ 脱敏 | `nickname` 形如 `林***I` |
| 分享 share_count | ✅ 附带 | `share_count` |

**播放量是这个方案唯一的硬缺口**。原始 detail 响应的 `statistics` 里是否有 `play_count` **未验证**（无登录态调不了接口）；但可以确定：**即使接口有，当前 store 也会丢掉**，要补就得改 `store/douyin/__init__.py`。

---

## 4. 登录态能不能持久、headless 能不能无人值守

### 4.1 持久化机制：有

- `config/base_config.py:54`：`SAVE_LOGIN_STATE = True`。
- `core.py:342-355`：persistent context，目录 `browser_data/<PLATFORM>_user_data_dir`（`base_config.py:98`）。
- 登录判定：`DouYinClient.pong()` — `client.py:163-176`，依次看 localStorage `HasUserLogin=1` → cookie `LOGIN_STATUS=1` → **兜底看 `sessionid` / `sid_tt` 是否存在**（这段是本仓库打的补丁，注释写明抖音新版不再下发 `LOGIN_STATUS`）。

### 4.2 但当前登录态是空的 → 首次必须扫码

实测：`~/.mediacrawler/browser_data/dy_user_data_dir` 和 `cdp_dy_user_data_dir` **都是空目录**（没有 Cookies / Network 文件）。

→ 现在直接跑 creator 模式，一定会进 `DouYinLogin`（`core.py:104-116`）弹二维码。**这就是需要 Lin 做的唯一一件事：有头模式跑一次，用抖音 App 扫码。**

之后 `--headless true` 可复用该目录无人值守。

### 4.3 扫一次管多久：未验证

源码里没有任何 cookie 过期判断、续期逻辑或 TTL 常量。`sessionid` 的实际有效期由抖音服务端决定，本仓库无从推断 → **未验证**。经验上抖音 web sessionid 能撑数天到数周，但这条**没有一手证据，不作为结论**。

### 4.4 headless 稳定性

- 默认走 CDP 模式：`base_config.py:60` `ENABLE_CDP_MODE = True`，连本机 Chrome 的 9222 端口（`core.py:79-86` → `tools/cdp_browser.py`）。CDP 模式本质是接一个真实浏览器，比纯 headless 更像人，但也要求本机 Chrome 起着。
- 抖音首登后经常追加手机号验证 / 滑块，**必须有人在场**：`references/platforms.md:18`「抖音：风控最敏感的平台之一，首次登录扫码后可能出现手机号验证，需有头模式人工处理」；`login.py:66-69` 注释更直白——「滑块验证的识别准确率不高，如果没有特殊要求，建议不要在抖音登录」。
- 结论：**扫码那一次必须有头 + 人工；扫码成功后，日常定时任务是能 headless 跑的（登录态未失效期间）**。失效后需要再扫一次。

---

## 5. 风控表现

源码里写死的失败信号（都可直接用作监控点）：

| 信号 | 位置 | 含义 |
|---|---|---|
| `response.text == ""` 或 `"blocked"` → 抛 `account blocked` | `client.py:143-145` | 账号被拉黑 / 请求被拦 |
| 403 `Blocked by ArgusSecurityPlugin Uifid Not Found` | `client.py:62-67` 注释 | 缺 `x-tt-argus` / uifid 网关头 |
| 403 `... Signature Not Found` | 同上 + `client.py:234-246` | uifid 与 `s_v_web_id` 不同源（detail 接口要求 uifid/verifyFp/fp 成套） |
| search 响应无 `data` 字段 | `core.py:163-165` | 日志直接写「账号也许被风控了」 |
| 页面标题含「验证码中间页」 | `login.py:87-89` | 登录后跳滑块验证页 |

另外：无登录态直接拉分享页 `https://www.iesdouyin.com/share/video/<id>/` 实测只返回 **argus csp 脚本 + captcha/verify 字样**，没有 `_ROUTER_DATA` 里的视频数据 → **「不登录直接抓 HTML 拿文案/播放量」这条路不可靠**。

- **未验证**：具体限流阈值（多少次请求会触发）、封号征兆。没登录态没跑真实爬取，不编数字。

---

## 6. 对 issue #5 的落地建议（按依赖顺序）

1. **补 author.sec_uid 提取**：在 detail 分支里，从 `aweme_detail["author"]["sec_uid"]` 取作者，别等落盘。current store 匿名化是刻意设计（见 `tests/test_douyin_no_user_info.py`），改动要单独决定——它和「对标分析要作者身份」这个需求是直接冲突的，需要 Lin 拍板。
2. **给 creator 模式加 30 条上限**：改 `client.py:361 get_all_user_aweme_posts` 加 `max_count`，同时把 `--crawler_max_notes_count` 接进来（现在接了但不生效，是个陷阱）。
3. **补 play_count**：先带登录态打一次 detail 接口，确认 `statistics.play_count` 是否存在；存在再改 `store/douyin/__init__.py` 加字段。不存在就得换接口或降级接受。
4. **登录**：有头跑一次给 Lin 扫码，之后 headless。

---

## 7. 卡在哪 / 需要 Lin 做什么

| 事项 | 需要谁 |
|---|---|
| 首次扫码（`browser_data/dy_user_data_dir` 为空） | **Lin**：有头模式跑一次，抖音 App 扫码；若弹手机号验证也要 Lin 处理 |
| 是否允许在落盘结果里保留 `sec_uid`（当前被匿名化） | **Lin 决策**：不给就等于放弃「贴链接找作者」这条入口 |
| `statistics.play_count` 是否真实存在 | 需扫码后才能验证（我这边无法调接口） |
| 创作者作品列表是否时间倒序 | 需扫码后才能验证 |

---

## 附：本次核对过的源码位置

| 文件 | 关注点 |
|---|---|
| `media_platform/douyin/client.py:228-250` | detail 接口 + argus 参数 |
| `media_platform/douyin/client.py:341-374` | user_info / aweme post 翻页 |
| `media_platform/douyin/client.py:376-399` | 短链 302 解析 |
| `media_platform/douyin/core.py:273-310` | creator 模式主流程（无条数上限） |
| `media_platform/douyin/help.py:101-164` | 视频/主页 URL 解析 |
| `store/douyin/__init__.py:94-157` | 落盘字段映射 + 匿名化 |
| `tools/user_hash.py` | uid/昵称脱敏实现 |
| `config/base_config.py:54/60/98/104/120/144` | 登录态、CDP、条数、并发、间隔 |
| `cmd_arg/arg.py:287-294, 399-405` | CLI 参数与 creator_id 注入 |
