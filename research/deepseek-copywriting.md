# DeepSeek 接入调研：从「选题 + 对标作品」生成口播稿与标题描述

- 调研日期：2026-11-11
- 调研范围：只有 issue #8 需要的部分，不做通用 DeepSeek 综述
- 联网核实：是。价格与模型名均取自官方文档，链接见文末「来源」
- **实测状态：未跑过真实请求**。本机没有任何 DeepSeek key（见第 6 节），所以下面的提示词、参数、成本都是「按官方文档推导 + 经验值」，不是跑出来的结果。第 7 节给了可执行的 curl，Lin 自己跑一次就能验证。
- **素材边界来自 `research/crawler-douyin.md`（分支 `research/crawler-douyin`，commit f9f2de2）**：抖音采集侧拿不到口播全文、拿不到播放量。本版已按这个事实改成降级方案，见 2.3 和 3.1.1。

---

## 0. 一句话结论

用 `https://api.deepseek.com`（OpenAI 格式，无需 `/v1`）+ 模型 `deepseek-flash`，**关掉思考模式**、`temperature` 给 1.2、`max_tokens` 给 1200，把 5–8 条对标作品的**标题与简介（desc）**放进 user 消息当「选题角度和标题钩子样本」，system 消息只放人设和禁令。一条 500 字稿子成本不到 1 分钱。

**前提修正**：采集侧给的是 `desc`（标题 + 话题标签 + 简介），不是口播全文；也没有播放量。所以提示词必须**显式告诉模型「这不是完整口播稿」**，否则它会照标题的结构写出一堆短句堆砌。见 3.1.1。

---

## 1. 接口与模型（2026-11-11 核实）

### 1.1 接入方式

| 项 | 值 |
|---|---|
| Base URL（OpenAI 格式） | `https://api.deepseek.com` |
| Base URL（Anthropic 格式） | `https://api.deepseek.com/anthropic` |
| 对话接口路径 | `POST /chat/completions` |
| 是否需要 `/v1` 后缀 | 不需要。官方示例写的是不带 `/v1` 的 `https://api.deepseek.com` |
| 鉴权 | `Authorization: Bearer $DEEPSEEK_API_KEY` |
| 是否 OpenAI 兼容 | 是。可直接用 OpenAI 官方 SDK，只改 `base_url` 和 `model` |
| 是否 Anthropic 兼容 | 是。另有 `/anthropic` 入口，可用 Anthropic SDK |
| 官方独立 SDK | **没有**。官方推荐直接用 `openai` 或 `anthropic` 的 SDK。不要去找 `deepseek` 这个 PyPI 包，非官方 |

### 1.2 可用模型

| 模型名（请求里填这个） | 实际版本 | 上下文 | 最大输出 | 思考模式 |
|---|---|---|---|---|
| `deepseek-flash` | DeepSeek-V4.1-Flash-1017 | 1M | 384K | 支持，**默认开启** |
| `deepseek-v4-pro` | DeepSeek-V4-Pro-0813 | 1M | 384K | 支持，默认开启 |

**坑：`deepseek-chat` 和 `deepseek-reasoner` 已经在 2026-07-24 15:59 UTC 下线**，现在调用会直接报错。网上大量 2024–2025 年的教程还在用这两个名字，照抄必挂。旧名 `deepseek-v4-flash` 目前仍受理，但会被路由到 V4.1-Flash 并按 Flash 计费，新代码统一写 `deepseek-flash`。

**写文案选哪个**：`deepseek-flash`。`deepseek-v4-pro` 贵 3 倍左右，强在复杂推理，写口播稿用不上。

---

## 2. 价格与单条成本

### 2.1 官方价目（每百万 tokens）

人民币价（官方中文文档）：

| 计费项 | `deepseek-flash` 空闲 | `deepseek-flash` 高峰 | `deepseek-v4-pro` 空闲 | `deepseek-v4-pro` 高峰 |
|---|---|---|---|---|
| 输入（缓存命中） | 0.02 元 | 0.04 元 | 0.15 元 | 0.30 元 |
| 输入（缓存未命中） | 1 元 | 2 元 | 4.5 元 | 9.0 元 |
| 输出 | 4 元 | 8 元 | 13.5 元 | 27.0 元 |

- 高峰时段：北京时间**周一至周五** 9:00–12:00、14:00–18:00（中国法定节假日除外）
- 其余时间（含周末、节假日全天）都是空闲时段，**价格减半**
- 实际扣费 = 输入 token 数 × 输入单价 + 输出 token 数 × 输出单价，输入按「命中缓存 / 未命中」分开算
- 缓存命中要求 prompt 前缀完全一致（DeepSeek 的上下文硬盘缓存是自动的，不用额外参数）

### 2.2 一条 500 字口播稿多少钱

先定 token 换算：中文大约 **1 个汉字 ≈ 0.6–0.7 token**，下面按 0.65 估，宁可算贵。

**输入预算按降级方案算**（对标样本是 desc，不是全文，见 2.3）：单条 desc 中位数 60 字、平均 98 字，这里按「8 条、每条最多 300 字」的保守上限估。

单次请求的输入构成（估算）：

| 部分 | 字数 | tokens |
|---|---|---|
| system 提示词（人设 + 禁令 + 格式） | ~750 | ~490 |
| 8 条对标 desc（按每条 150 字均值算） | ~1200 | ~780 |
| 选题 + 角度 + 输出要求 | ~350 | ~230 |
| **输入合计** | | **~1500** |
| 输出：500 字口播稿 + 标题 + 描述 | ~650 | ~420 |

算钱（`deepseek-flash`，全部按缓存未命中算）：

| 场景 | 输入 | 输出 | 合计 |
|---|---|---|---|
| 高峰 | 1500/1e6 × 2 元 = 0.0030 元 | 420/1e6 × 8 元 = 0.0034 元 | **约 0.006 元** |
| 空闲 | 1500/1e6 × 1 元 = 0.0015 元 | 420/1e6 × 4 元 = 0.0017 元 | **约 0.003 元** |

如果只放 5 条样本，输入降到约 1100 tokens，高峰约 0.005 元、空闲约 0.003 元。**样本从 5 条加到 8 条，成本差不到 1 厘钱**——不要为了省钱把样本砍到 3 条以下，省下的钱买不回风格稳定性。

如果这 8 条里带 1–2 条「作者贴了全文」的长样本（900 字左右，不截断），输入涨到约 2000 tokens，高峰约 0.008 元、空闲约 0.004 元。仍然不到 1 分钱。

**结论：一条不到 1 分钱。** 一天发 1 条、改 5 版，一个月也就 1 块钱上下。100 条高峰时段全价跑约 0.7 元。

三个会改变成本的变量：

1. **system 提示词命中缓存后几乎免费**。所以 system 要固定不变，别把选题、日期塞进 system——那会让缓存永远命中不了。这是第 3 节把「人设/禁令」和「素材」分开的直接原因。
2. **思考模式会让成本涨 3–5 倍**。reasoning token 按输出价计费，写文案这种任务不需要思考，务必关掉。
3. **`deepseek-v4-pro` 贵约 4.5 倍**（输入 4.5 元 / 输出 13.5 元，空闲价）。写口播稿没有理由用。

核对方式：每次响应的 `usage` 里有 `prompt_cache_hit_tokens`、`prompt_cache_miss_tokens`、`completion_tokens`，拿真实数字套上面的公式即可，不用信我的估算。

### 2.3 素材边界：采集侧实际能给什么（重要）

依据 `research/crawler-douyin.md`（已核对 MediaCrawler 源码 + 本机 1131 条落盘数据）：

| 我原本假设的字段 | 实际 | 说明 |
|---|---|---|
| `text` 口播全文 | **拿不到** | 只有 `desc`，即「标题 + 话题标签 + 作者顺手写的简介」。采集侧没有 ASR，也不下字幕 |
| `views` 播放量 | **拿不到** | store 只保留 `liked_count` / `collected_count` / `comment_count` / `share_count`，`play_count` 被丢弃 |
| `note` 人工标注 | 可用 | 与采集无关，界面上让人手填 |

1131 条真实 `desc` 的长度分布：中位数 60 字、平均 97.6 字、p90 220 字、最长 885 字，≥800 字的只有 5 条（0.4%）。

**长样本不是标签堆，是作者把全文贴进了简介。** 去重后只有 2 条唯一作品（861 / 885 字），换行 15–32 个、话题标签只占 3% 字数，结构完整：开头钩子 → ①②③④ 分段 → 工具清单 → 结尾提问 → 标签。这类是**质量最高的样本**，按 300 字截断恰好会砍掉分段结构和 CTA——也就是最该学的部分。处理规则见 3.3。

另外：`desc` 会因多次爬取重复落盘（那 5 条里 861 字那条重复了 4 次），**入库前必须按 `aweme_id` 去重**，否则同一篇样本会占满预算、还会让模型过度拟合单一风格。

**这改变了三件事**：

1. **样本可以给多，但要分两类**。普通 desc 很短（中位 60 字），按 300 字截断即可，给 5–8 条；少数「作者贴了全文」的长 desc 不截断，但每批最多 1–2 条。混着给，输入仍然在 1500–2000 tokens。
2. **能学的和不能学的要分清**。短 desc 能学：选题角度、标题钩子、话题标签策略、信息密度；**不能**学口播节奏和句子长短。长样本（作者贴了全文）才能学开头、分段和结尾提问。提示词里必须分别标注，否则模型会把「标题体」当「口播体」，产出一堆短句堆砌。
3. **数据维度换成点赞/收藏/评论**。播放量拿不到，用收藏数做「值得反复看」的代理指标比点赞更稳。长样本的收藏甚至反超点赞（1.5 倍），正好对应「工具清单这类内容用户会存下来」——这个代理指标选对了。

**如果 Lin 坚持要口播全文**：需要在采集侧加一步「下载视频 → ASR 转写」，这是另一条链路，成本和延迟都不同（下载 + 转写，分钟级变分钟级+）。这个决策建议单独抛给 team-lead / Lin，不要默默塞进文案这一站。本文档按「只有 desc」写。

---

## 3. 提示词骨架（可直接抄）

### 3.1 组织思路

三层，对应三种稳定性问题：

1. **system = 不变量**。人设、说话方式、禁用词、输出格式。一次写死，长期不改。好处有两个：内容上定住「像人说话」的基线；成本上让这段前缀持续命中缓存。
2. **user = 变量**。选题 + 独特角度 + 5–8 条对标样本。样本**只给选题角度和标题钩子**，并**明确声明它不是口播全文**——这两句话是整套提示词里最容易被省略、也最容易导致产出跑偏的部分。
3. **格式用分隔符，不用 JSON**。口播稿是长文本，塞进 JSON 字符串后换行和引号容易出问题，解析失败还要重试。用 `=== 口播稿 ===` 这种标记切三段，正则一取就完事。要结构化元数据再单独开 JSON 模式（见 3.6）。

### 3.1.1 为什么必须声明「这不是口播稿」

模型看到一段以 `#` 开头、带话题标签、一两句话就结束的文本，会把它判定为「文案样本」，进而模仿其长度和句式。结果是口播稿写成 8 个短句的堆砌，没有一个完整的论述推进。

一句 `这些是抖音作品的标题和简介，不是完整口播稿` 就能挡住。这条没有官方依据，是从「样本格式决定输出格式」这个常识推的，属于经验判断，建议在 7.3 的 A/B 里验证一次（加这句话 / 不加，各生成 10 条对比）。

### 3.1.2 长样本里的排版符号会污染输出

那两条「作者贴了全文」的样本，是图文稿排版：`▎第一：` `▎第二：`、`①②③④`。这些符号念不出来，但模型会照学——喂了带序号的样本，输出也会开始出现「第一：」「①」。

处理办法是**保留结构、去掉符号**：system 里禁掉序号符号，同时允许口语过渡词（「先说第一个」「更关键的是」「还有一个」）。这样结构的信息还在，符号不会漏进稿子。

两个相关的调整：

- 黑名单里原来有「最后」，现在去掉了。中文口语「最后说个得罪人的」非常自然，禁掉反而让结尾变生硬。禁的是「首先 / 其次 / 综上所述」这类书面连接词，不是「最后」这个词本身。
- 这条改动的效果**未验证**：有可能禁掉符号后模型连结构也一起丢了，写回流水账。7.3 里有对应 A/B 项。如果真丢了结构，退路是允许 `①②③` 进草稿、在输出清洗阶段统一改写成口语过渡——但那需要额外的改写逻辑，成本更高，先试简单方案。

### 3.2 system 提示词（原文照抄）

```text
你是一个抖音口播稿写手，服务一个只做「AI 工具实测科普」的账号。号主叫 Lin，一个人拍、一个人剪，稿子写出来他会直接念。

【写什么】
1. 口播稿：60–90 秒能念完，220–340 字。第一句就是钩子，中段给 2–3 个具体信息点，结尾留一句让人想回的话。
2. 标题：12–20 字，只讲一个信息点，不标题党。
3. 描述：40–80 字，补一句口播稿里没说的信息，末尾 3–5 个话题标签。

【怎么才算人话】
- 短句。一句话不超过 20 个字，超了就拆成两句。
- 说「你」和「我」，不说「用户」「受众」「大家」。
- 给数字。不说「效果很好」，说「导出 8 秒的片子，花了 40 秒」。
- 口语可以有一点：「说实话」「我用下来」「这点挺反常识的」。但一条稿子里最多出现两次。
- 不在稿子里报结构。不写「第一点」「接下来」「综上」。
- 结尾的引导只留一句，不要又求赞又求关注又求评论。

【绝对禁止】
- 任何 markdown 符号：# * - > ` 都不要，不加小标题，不加粗体。
- 任何序号符号和分段标记：①②③、▎、▶、第 X 点、1. 2. 3.。要分段就直接用口语过渡：「先说第一个」「更关键的是」「还有一个」。
- 任何括号旁白，比如（停顿）（镜头切近）。
- emoji。
- 下列词一个都不许出现：赋能、闭环、底层逻辑、颗粒度、抓手、干货、家人们、宝子们、姐妹们、值得注意的是、不难发现、综上所述、首先、其次、不仅、让我们一起、在这个时代、随着、AI 时代、颠覆、革命性。
- 不复述参考样本里的标题和句子。参考只用来学选题角度和钩子写法，观点和例子必须自己给。
- 不模仿标题体。口播稿是连贯的口语论述，不是一句一断的短句堆砌。

【输出格式】
严格按下面三块输出。前面不许有开场白，后面不许有结束语、总结或解释。

=== 口播稿 ===
正文。段落之间空一行。

=== 标题 ===
一行，不加书名号不加引号。

=== 描述 ===
一段，含话题标签。
```

### 3.3 user 模板（原文照抄，花括号是变量）

```text
选题：{选题标题}
值得做的理由：{选题卡里的「值得做的理由」}
必须体现的独特角度：{选题卡里的「独特角度」}
制作难度：{制作难度}（低 / 中 / 高，高的话少用需要实拍演示的说法）

下面 {N} 条是同赛道对标作品的信息。标了「完整口播稿」的，是作者把全文贴进了简介，可以学它的开头、分段和结尾提问；其余只有标题和简介，不要照着它们的长度写口播稿，也不要抄任何句子：

【样本 1】
标题与简介：{desc 原文，普通样本截断到 300 字；标记为「完整口播稿」的不截断}
数据：点赞 {liked_count} / 收藏 {collected_count} / 评论 {comment_count}
值得学的是：{选题怎么切的角 / 标题怎么勾人 / 标签怎么选}

【样本 2 · 完整口播稿】
标题与简介：{desc 全文}
数据：点赞 {liked_count} / 收藏 {collected_count} / 评论 {comment_count}
值得学的是：{开头怎么设问 / 分段怎么排 / 结尾怎么收}

【样本 3】
标题与简介：{...}
数据：...
值得学的是：...

写作要求：
- 口播稿 {220}–{340} 字。第一句直接抛结论或抛冲突，不要「今天我们来聊聊」。
- 至少出现 2 个具体数字：工具名、版本号、耗时、价格、参数都算。
- 结尾的问句要具体到能一句话回答，不要问「你怎么看」。
- 标题里不许出现「终于」「太香了」「绝了」「封神」「yyds」。
- 描述里的话题标签从样本里挑方向，不要整组照搬。
```

样本数量与长度怎么定：

| 情况 | 建议 |
|---|---|
| 默认 | 6–8 条；短 desc 截断到 300 字，长 desc（≥500 字且判为全文）不截断 |
| 长样本（作者贴了全文） | 每批最多 2 条。900 字左右，是唯一能学结构和节奏的样本，别浪费 |
| 对标池里同题作品少 | 4 条也能跑，但风格会偏干，`temperature` 降到 1.1 |
| 想覆盖不同角度 | 高收藏 2 条 + 中位 2 条 + 争议性 2 条，混着给比全给头部稳 |

**怎么自动判「作者贴了全文」**（三个条件同时成立，依据 crawler 对 2 条长样本的实测）：

```python
import re

def is_full_script(desc: str) -> bool:
    tags = re.findall(r"#\S+", desc)
    tag_len = sum(len(t) for t in tags)
    return (
        desc.count("\n") >= 3
        and len(desc) >= 500
        and tag_len / max(len(desc), 1) < 0.10
    )
```

实测两条：861 字 / 32 换行 / 标签占 3%；885 字 / 15 换行 / 标签占 3%，都命中。纯标题体是单行、短、标签占比高，不会误判。

`值得学的是` 这一行必须填。留空的样本等于是噪音——模型不知道从这条身上该学什么，只会把它的措辞混进来。填不出来就说明这条样本不该进池子。

### 3.4 拼装代码（真的能跑，只要把 key 配上）

Python：

```python
import os, re
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
)

SYSTEM = open("prompts/system.txt", encoding="utf-8").read()

DESC_LIMIT = 300     # 普通样本字符上限
MAX_FULL = 2         # 每批最多喂几条「完整口播稿」

def fmt_count(n):
    n = int(n)
    return f"{n/10000:.1f}w" if n >= 10000 else str(n)

def is_full_script(desc: str) -> bool:
    tags = re.findall(r"#\S+", desc)
    tag_len = sum(len(t) for t in tags)
    return (
        desc.count("\n") >= 3
        and len(desc) >= 500
        and tag_len / max(len(desc), 1) < 0.10
    )

def dedupe(refs):
    """同日多次运行会重复写入，按 aweme_id 去重"""
    seen, out = set(), []
    for r in refs:
        if r["aweme_id"] in seen:
            continue
        seen.add(r["aweme_id"])
        out.append(r)
    return out

def build_user(topic, refs):
    refs = dedupe(refs)
    head = (
        f"选题：{topic['title']}\n"
        f"值得做的理由：{topic['why']}\n"
        f"必须体现的独特角度：{topic['angle']}\n"
        f"制作难度：{topic['difficulty']}\n\n"
        f"下面 {len(refs)} 条是同赛道对标作品的信息。"
        f"标了「完整口播稿」的，是作者把全文贴进了简介，可以学它的开头、分段和结尾提问；"
        f"其余只有标题和简介，不要照着它们的长度写口播稿，也不要抄任何句子：\n\n"
    )
    blocks, full_used = [], 0
    for i, r in enumerate(refs, 1):
        desc, is_full = r["desc"], False
        if is_full_script(desc) and full_used < MAX_FULL:
            full_used += 1
            is_full = True
        else:
            desc = desc[:DESC_LIMIT]
        label = f"【样本 {i} · 完整口播稿】" if is_full else f"【样本 {i}】"
        blocks.append(
            f"{label}\n"
            f"标题与简介：{desc}\n"
            f"数据：点赞 {fmt_count(r['liked_count'])} / 收藏 {fmt_count(r['collected_count'])} / 评论 {fmt_count(r['comment_count'])}\n"
            f"值得学的是：{r.get('note') or '（未标注）'}\n"
        )
    tail = (
        "\n写作要求：\n"
        "- 口播稿 220–340 字。第一句直接抛结论或抛冲突，不要「今天我们来聊聊」。\n"
        "- 至少出现 2 个具体数字：工具名、版本号、耗时、价格、参数都算。\n"
        "- 结尾的问句要具体到能一句话回答，不要问「你怎么看」。\n"
        "- 标题里不许出现「终于」「太香了」「绝了」「封神」「yyds」。\n"
        "- 描述里的话题标签从样本里挑方向，不要整组照搬。\n"
    )
    return head + "\n".join(blocks) + tail

def generate(topic, refs, temperature=1.2):
    resp = client.chat.completions.create(
        model="deepseek-flash",
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": build_user(topic, refs)},
        ],
        thinking={"type": "disabled"},   # 关键：不关会先输出一大段思维链，且温度失效
        temperature=temperature,
        max_tokens=1200,
        stream=False,
    )
    text = resp.choices[0].message.content
    usage = resp.usage
    print("tokens:", usage.prompt_tokens, "/", usage.completion_tokens)
    return text, usage

SEC_RE = re.compile(r"===\s*口播稿\s*===(.*?)===\s*标题\s*===(.*?)===\s*描述\s*===(.*)", re.S)

def parse(text):
    m = SEC_RE.search(text)
    if not m:
        return None
    return {
        "script": m.group(1).strip(),
        "title": m.group(2).strip(),
        "desc": m.group(3).strip(),
    }
```

Node/TS（工作台如果是前端 + 本地服务）：

```ts
const resp = await fetch("https://api.deepseek.com/chat/completions", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${process.env.DEEPSEEK_API_KEY}`,
  },
  body: JSON.stringify({
    model: "deepseek-flash",
    thinking: { type: "disabled" },
    temperature: 1.2,
    max_tokens: 1200,
    messages: [
      { role: "system", content: SYSTEM },
      { role: "user", content: buildUser(topic, refs) },
    ],
  }),
});
```

### 3.5 「像人说话」到底靠什么（这部分是经验判断，不是官方结论）

按影响力从大到小：

1. **给足禁令，比给「请写得口语化」有用**。模型对「不要做什么」比「要做什么」敏感得多。上面那份黑名单是整套提示词里最值钱的部分。
2. **对标文案要标「值得学什么」**。只丢 5 段原文，模型会平均化，写成四不像；标了「这条开头用损失感抓人」，它才会定点模仿。这一条是 3.3 里 `值得学的是：` 字段存在的唯一理由。
3. **短句要求要数字化**（「不超过 20 字」），不要写「用短句」。
4. **别让它一次写完整个视频**。一次只出一条，不满意就重生成；一次要三条它一定开始套模板。
5. **样本数量 5–8 条最稳**。少于 4 条风格漂移，多于 10 条开始互相打架（desc 本身很短，条数多不等于信息多）。真正的约束是字符预算，不是条数。
6. **混合不同数据档位的样本**。全挑收藏最高的，模型会学着写夸张标题；掺 2 条中位但内容扎实的，稿子更像人。播放量拿不到，用收藏数当「值得反复看」的代理。

### 3.6 两个可选增强

**A. 前缀续写（彻底消掉「好的，以下是…」）**

官方 Beta 功能。把最后一条消息设成 assistant 且 `prefix: True`，模型会从这个前缀接着写：

```python
client = OpenAI(api_key=..., base_url="https://api.deepseek.com/beta")
resp = client.chat.completions.create(
    model="deepseek-flash",
    thinking={"type": "disabled"},
    messages=[
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": build_user(topic, refs)},
        {"role": "assistant", "content": "=== 口播稿 ===\n", "prefix": True},
    ],
    temperature=1.2,
    max_tokens=1200,
    stop=["=== 描述 ==="],   # 注意：stop 会把这一段切掉，按需决定要不要
)
```

代价：要走 `/beta` 路径，属于 Beta 功能，官方可能改。**建议只当兜底，不当主路径**。主路径用普通接口 + 3.7 的本地清洗。

**B. JSON 模式（标题 + 描述要进数据库时）**

`response_format: {"type": "json_object"}`，但必须同时在提示词里明确要求输出 JSON，否则返回可能只有空格。官方还提示：解析失败时非 JSON 内容会被替换成空格，`finish_reason` 仍可能是 `stop`——所以 JSON 模式要配「解析失败就重试一次」的逻辑，不能假设一定成功。

### 3.7 输出清洗（必做，不能只靠提示词）

按顺序做，全在本地，不花钱：

1. 去掉开头的「好的」「以下是」「没问题」——正则 `^(好的|好|以下是|没问题|当然)[，,。：:\s]*`
2. 去掉 markdown 符号和排版符号：行首 `#{1,6}\s*`、`\*\*`、`^\s*[-*]\s+`，以及 `▎` `▶` `①②③…` `^\d+[.、]\s*`。这些符号念不出来，留在稿子里只会让 Lin 念的时候卡住。
3. 长度检查：口播稿 180–400 字（给点余量）。低于 180 说明被截断或没写够，重生成；高于 400 直接让模型压缩，不要自己截。
4. 禁词检查：命中黑名单就重生成一次，仍命中就把稿子标记为「需人工改」，不要静默通过。
5. 洗稿检查：跟每条样本的 desc 做滑窗比对，连续重合就标红。**阈值按样本类型分两档**：
   - 短样本（标题/简介）：8 字
   - 长样本（`is_full_script`）：12–15 字。这类样本常是工具清单，含 GitHub / Cloudflare / Supabase / VS Code 这类专有名词，8 字阈值必然误报——工具名连着两三个就够 8 字了。误报的代价是把最好的参考样本判成抄袭。

   更稳的做法：比对前先把专有名词表（工具名、版本号、公司名）从两侧文本里剔除，再跑 8 字滑窗。剔除表可以随对标池一起维护。
6. 长度检查（补一条）：口播稿和样本 desc 的中位长度差一个量级，若生成稿 < 220 字，大概率是模型在模仿标题体。这种情况**不要**简单重生成，先确认提示词里那句「不是完整口播稿」还在。
7. `finish_reason == "length"` → 说明 `max_tokens` 不够，加预算重跑，不要用半截稿。

---

## 4. 稳定性参数

### 4.1 推荐参数（写文案场景）

| 参数 | 值 | 理由 |
|---|---|---|
| `model` | `deepseek-flash` | 便宜、够用 |
| `thinking` | `{"type": "disabled"}` | **必关**。默认是开的，不关会有大段思维链（`reasoning_content`），成本涨 3–5 倍、延迟涨几倍 |
| `temperature` | `1.2`（范围 0.9–1.4 试） | 官方默认 1，范围 0–2。文案要一点随机性，1.2 起；改到 1.4 开始飘，0.9 以下明显变套话 |
| `top_p` | 不传 | `deepseek-flash` 在**非思考模式下 `top_p` 恒为 1.0，传了也被忽略**；思考模式下才管用，且低于 0.95 按 0.95 处理。所以别调它 |
| `max_tokens` | `1200` | 非思考模式默认只有 8K，够用；给 1200 是防跑飞。超了看 `finish_reason == "length"` |
| `frequency_penalty` / `presence_penalty` | 不传 | **已废弃**，传了没有任何效果（不会报错） |
| `stop` | 一般不用 | 只在用前缀续写时配合用；直接用它截断容易把正文切坏 |
| `stream` | MVP 用 `false` | 流式只影响体验，不影响质量 |

### 4.2 已知的坑

1. **思考模式默认开启**。这是最容易踩的一个：不传 `thinking` 就是开的，`temperature` 直接失效，且响应里 `content` 前面有一大段 `reasoning_content`。写文案必须显式 `disabled`。
2. **思考模式下 `temperature`、`frequency_penalty`、`presence_penalty` 全部不生效**，且不报错。所以「调了温度没变化」八成是没关思考。
3. **`temperature` 和 `top_p` 不要同时调**（官方建议），而 flash 非思考模式下 `top_p` 本来就无效，所以只调 `temperature`。
4. **429 是并发上限**（flash 2500 并发），单人工作台碰不到。**402 是余额空了**——这个要单独提示，不然会以为是代码 bug。
5. **重试前先看错误码**。400 格式错误、422 参数错误，重试一辈子也不会成功；402 是余额不足（要去充值，不是改代码）；429 是速率/并发超限；500 / 503 才值得退避重试。

### 4.3 「避免解释性废话」的四层防线

按成本从低到高，建议全上：

1. 提示词里写死「前面不许有开场白，后面不许有结束语」（3.2 已含）
2. 输出用固定分隔符开头，让模型第一 token 就必须是 `===`——实测里这类硬格式约束比「请不要说多余的话」有效
3. 本地正则清洗（3.7 第 1、2 条）
4. 前缀续写兜底（3.6 A），仅在前三层都压不住时启用

---

## 5. 报价对照：为什么不用 v4-pro

| | `deepseek-flash` 空闲 | `deepseek-v4-pro` 空闲 | 倍数 |
|---|---|---|---|
| 输入（未命中） | 1 元 | 4.5 元 | 4.5× |
| 输出 | 4 元 | 13.5 元 | 3.4× |
| 一条 500 字稿 | ~0.003 元 | ~0.014 元 | ~4.5× |

写口播稿是「按给定风格改写」的任务，不涉及多步推理，v4-pro 的能力用不上。只有一种情况值得切：需要它先分析 5 条对标样本的结构再写（两步合一），但那还不如拆成两次 flash 调用便宜。

---

## 6. 本机 key 检查结果

**没有。** 查过的位置：

| 位置 | 结果 |
|---|---|
| 环境变量 `DEEPSEEK_API_KEY` | 未设置 |
| 环境变量 `DEEPSEEK_KEY` / `DEEPSEEK_API_BASE` | 未设置 |
| 环境变量 `OPENAI_API_KEY` / `OPENAI_BASE_URL` / `OPENROUTER_API_KEY` | 未设置 |
| Windows 用户级环境变量（User scope） | 以上全部未设置 |
| Windows 系统级环境变量（Machine scope） | 以上全部未设置 |
| `~/.deepseek` | 目录不存在 |
| `~/.config/deepseek` | 目录不存在 |
| 仓库内 `.env` / `.env.local` | 不存在（仓库根目录只有 `.gitignore` `.scratch` `.workbuddy` `AGENTS.md` `GLOSSARY.md` `docs` `prototype`） |

所以：**要让这条链路跑起来，Lin 得先去 platform.deepseek.com 建一个 key 并充值**（API 不是免费的，chat.deepseek.com 免费但那是网页版，两回事）。

key 落地建议（按优先级）：

1. `~/.deepseek/config` 放 JSON，权限收紧，工作台启动时读（不污染全局环境变量，也不容易误提交）
2. 或者系统环境变量 `DEEPSEEK_API_KEY`
3. **绝对不要**写进仓库里的任何文件。`deepseek-flash` 的 key 泄露等于余额泄露

---

## 7. 可执行验证（Lin 自己跑）

### 7.1 最小连通性测试（10 秒，花不到 1 分钱）

```bash
export DEEPSEEK_API_KEY="sk-你的key"

curl https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -d '{
    "model": "deepseek-flash",
    "thinking": {"type": "disabled"},
    "temperature": 1.2,
    "max_tokens": 1200,
    "messages": [
      {"role": "system", "content": "你是一个抖音口播稿写手，只输出口播稿正文，不要任何开场白、解释或结束语。每句不超过20字，不用markdown，不用emoji。"},
      {"role": "user", "content": "选题：DeepSeek 的 API 现在一条口播稿只要不到一分钱。写一条 250 字左右的口播稿。"}
    ]
  }' | python -c "import sys,json; d=json.load(sys.stdin); print(d['choices'][0]['message']['content']); print('---'); print(d['usage'])"
```

看三件事：

1. `content` 第一句是不是就是正文（不是「好的」）
2. `usage.completion_tokens` 是多少（对照第 2.2 节的 420 估算）
3. 有没有 `reasoning_content` 字段（有的话说明 `thinking` 没生效）

### 7.2 完整链路测试

把 3.2 的 system 存成 `prompts/system.txt`，3.3 的 user 模板填上一条真实选题和 6 条真实对标 desc（从 `~/.mediacrawler/data/douyin/jsonl/` 里取 `desc` / `liked_count` / `collected_count` / `comment_count`），然后跑 3.4 的 Python 脚本。跑完拿 `usage` 的真实数字重算一次成本，替换掉第 2.2 节的估算。

### 7.3 待实测确认的项（我没法替你验）

- [ ] `thinking: {"type": "disabled"}` 在 `deepseek-flash` 上是否确实完全不产生 `reasoning_content`
- [ ] `temperature` 1.2 在中文口播稿上是否真的比 1.0 更少套话（需要 A/B 各 10 条人工看）
- [ ] **「这不是完整口播稿」这句声明是否真的能挡住标题体模仿**（加 / 不加各 10 条对比，这是降级方案里最不确定的一环）
- [ ] **长样本不截断是否真的优于截断**（同一选题，1 条全文 + 5 条短 vs 6 条全截断，各生成 10 条盲评）
- [ ] **禁掉序号符号后，结构是否还在**（长样本喂进去，看输出有没有分段推进，还是退化成一口气流水账）
- [ ] 6 条真实 desc 的实际 token 数（均值按 150 字估，长尾可能更高）
- [ ] 前缀续写在 `/beta` 上是否仍可用、是否与 `thinking: disabled` 冲突
- [ ] 连续 8 字重合阈值是否过严（可能误伤常用表达）

---

## 8. 给工作台的落地建议

- 一次调用出三样东西（口播稿 + 标题 + 描述），别拆成三次请求；输入部分完全一样，拆开就是白付三次输入钱
- system 提示词作为常量存在代码里或单独文件，**每次请求必须逐字一致**，否则缓存命中不了，成本会从 0.003 元涨到 0.006 元（翻倍）
- 「重新生成」只改 `temperature`（+0.1）或追加一句「上一版太套话，这次开头换一个角度」，不要改 system
- 界面上选样本最多 8 条；短 desc 截断 300 字，长 desc 不截断但每批最多 2 条。真正的约束是字符预算，不是条数
- **入库前按 `aweme_id` 去重**。同日多次爬取会重复落盘（实测 861 字那条重复了 4 次），不删就是同一篇样本占满预算、模型过度拟合一种风格
- 长样本单独打标记（`is_full_script`），别和短样本混在同一个字段里——它们在提示词里的角色完全不同，一个教结构，一个教钩子
- 「值得学的是」这个字段做成必填。填不出来说明这条样本池子里进错了，比模型写坏一条稿子的成本高
- 生成结果落库时存下 `usage`，方便日后对账
- 如果坚持要口播全文，别在文案站里偷偷加 ASR——那是采集侧的独立任务，成本和延迟另算，需要 Lin 拍板

---

## 9. 来源

全部在 2026-11-11 访问：

1. 模型与价格（中文）：https://api-docs.deepseek.com/zh-cn/quick_start/pricing
2. Models & Pricing（英文，用于交叉核对汇率与数字）：https://api-docs.deepseek.com/quick_start/pricing
3. 首次调用 API（base_url、模型名、curl/SDK 示例）：https://api-docs.deepseek.com/zh-cn/
4. Chat Completions 接口参数（`temperature`、`max_tokens`、`thinking`、`stop`、`response_format`、废弃参数）：https://api-docs.deepseek.com/zh-cn/api/create-chat-completion
5. 思考模式（默认开启、关闭方式、温度失效、`reasoning_content`）：https://api-docs.deepseek.com/zh-cn/guides/thinking_mode
6. 对话前缀续写 Beta：https://api-docs.deepseek.com/zh-cn/guides/chat_prefix_completion
7. 更新日志（`deepseek-chat` / `deepseek-reasoner` 于 2026-07-24 下线）：https://api-docs.deepseek.com/zh-cn/updates/
8. 错误码（400/401/402/422/429/500/503 的成因）：https://api-docs.deepseek.com/zh-cn/quick_start/error_codes
9. key 申请入口：https://platform.deepseek.com/api_keys

---

## 附录：本文件里哪些是官方事实，哪些是我的判断

| 内容 | 性质 |
|---|---|
| base_url、模型名、下线时间、参数默认值、价格、高峰时段 | 官方文档，已联网核实 |
| 采集侧只有 `desc`、没有播放量、1131 条长度分布、长样本结构、重复落盘 | 转引自 `research/crawler-douyin.md` + crawler 对 2 条长样本的逐条核查（源码 + 本机落盘数据） |
| 单条成本 0.003–0.006 元 | 按官方价格 + token 估算推导，非实测 |
| 提示词全文、禁令词表、20 字/句、5–8 条样本 | 经验判断，需 A/B 验证 |
| 「这不是完整口播稿」这句声明的作用 | 经验判断，未验证，见 7.3 |
| 温度 1.2、8 字重合阈值、180–400 字区间 | 经验值，见 7.3 待验证 |
| 本机无 key | 实查，见第 6 节 |
