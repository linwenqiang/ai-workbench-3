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

**长样本不是标签堆，是作者把正文贴进了简介。** 去重后只有 2 条唯一作品，但**形态不同**（crawler 已实拍确认）：

| aweme_id | 长度 | `aweme_type` | 形态 | 内容特征 | 收藏/点赞 |
|---|---|---|---|---|---|
| 7642677084858420490 | 885 字 | `0` | **视频**（有 `video_download_url`，无图集） | `▎第一：` 分段 + 结尾「最后说个得罪人的」 | 0.68× |
| 7694446460812660002 | 861 字 | `68` | **图文/图集**（`note_download_url` 6 张图） | `①②③④` 图片序号 + 工具清单 | 1.52× |

所以是「**1 条口播稿 + 1 条图文长文案**」，不是 2 条口播稿。这个区别不是抠字眼：

- 885 那条才是口播样本，能学开头设问、分段推进、结尾反转
- 861 那条是图文清单，`①②③④` 是**图片序号**，作为口播样本是错配；学它的信息组织可以，但不能当口播节奏的参照
- 收藏反超点赞只出现在图文清单类（1.52×），口播观点类仍是点赞 > 收藏（0.68×）。**说明「收藏 > 点赞」是内容类型特征，不是「结构好」的特征**

形态判定不用猜：`aweme_type == 68` 是图文，`== 0` 是视频。字段现成，直接读。

另外：`desc` 会因多次爬取重复落盘（那 5 条里 861 字那条重复了 4 次），**入库前必须按 `aweme_id` 去重**，否则同一篇样本会占满预算、还会让模型过度拟合单一风格。


**这改变了三件事**：

1. **样本可以给多，但要分两类**。普通 desc 很短（中位 60 字），按 300 字截断即可，给 5–8 条；「作者贴了正文」的长 desc 不截断，每批最多 1–2 条。混着给，输入仍然在 1500–2000 tokens。
2. **能学的和不能学的要分清**。短 desc 能学：选题角度、标题钩子、话题标签策略、信息密度；**不能**学口播节奏和句子长短。长样本里只有**视频类**（`aweme_type == 0`）能学口播的开头、分段和结尾；图文类（`68`）只能学信息组织。提示词里必须分别标注，否则模型会把「标题体」或「图文体」当「口播体」，产出一堆短句堆砌。
3. **数据维度换成点赞/收藏/评论**。播放量拿不到，用收藏数做「值得反复看」的代理指标比点赞更稳。但注意：收藏反超点赞只发生在图文清单类（1.52×），口播观点类仍是点赞更高（0.68×）——**这是内容类型特征，不能当「结构好」的证据**。

**口播全文要不要走 ASR：暂时不需要。** 上文这 2 条样本证明，少数作者会把完整正文贴在简介里，占比 0.4% 但正好是最优质的那批。只要采集侧把长样本标出来优先喂，就能拿到完整结构样本，零额外成本、零额外延迟。什么时候再考虑 ASR：等这条链路跑通、确认「只有 2 条长样本」不足以覆盖选题范围时再说。真要做也不在文案站里做——那是采集侧的独立任务（下载 + 转写），成本和延迟另算。

---

## 3. 提示词骨架（可直接抄）

### 3.1 组织思路

三层，对应三种稳定性问题：

1. **system = 不变量**。人设、说话方式、禁用词、输出格式。一次写死，长期不改。好处有两个：内容上定住「像人说话」的基线；成本上让这段前缀持续命中缓存。
2. **user = 变量**。选题 + 独特角度 + 5–8 条对标样本。样本要**分类标注**：哪些是「只有标题和简介」，哪些是「完整口播稿」（`aweme_type == 0` 且长），哪些是「图文长文案」（`68`）。这三类在提示词里的角色完全不同，混为一谈就会产出跑偏的稿子。
3. **格式用分隔符，不用 JSON**。口播稿是长文本，塞进 JSON 字符串后换行和引号容易出问题，解析失败还要重试。用 `=== 口播稿 ===` 这种标记切三段，正则一取就完事。要结构化元数据再单独开 JSON 模式（见 3.6）。

### 3.1.1 为什么必须声明「这不是口播稿」

模型看到一段以 `#` 开头、带话题标签、一两句话就结束的文本，会把它判定为「文案样本」，进而模仿其长度和句式。结果是口播稿写成 8 个短句的堆砌，没有一个完整的论述推进。

一句 `这些是抖音作品的标题和简介，不是完整口播稿` 就能挡住。这条没有官方依据，是从「样本格式决定输出格式」这个常识推的，属于经验判断，建议在 7.3 的 A/B 里验证一次（加这句话 / 不加，各生成 10 条对比）。

### 3.1.2 样本里的序号符号：转成口语序数，而不是删掉

长样本里有两类排版符号：视频稿那篇用 `▎第一：`，图文稿那篇用 `①②③④`（图片序号）。符号念不出来，但模型会照学——喂了带序号的样本，输出也会开始出现「①」「第一：」。

不要二选一（要么禁掉丢结构，要么留着被学走）。**`①②③④` 和 `▎第一：` 本来就是口语序数的替身**，作者念的时候说的就是「第一类，开发与上线」「第二，编码与调试」。所以做等价转换：

```
① 开发与上线          → 第一类，开发与上线
▎第一：对上一代的叙事去魅了  → 第一件事，对上一代的叙事去魅了
```

system 里不写「不出现序号符号」，而写「要分层次就用口语序数说出来」。层次留下了，符号不会漏进口播，也不用在「丢优点」和「学符号」之间选。

两个相关的调整：

- 黑名单里原来有「最后」，现在去掉了。中文口语「最后说个得罪人的」非常自然，禁掉反而让结尾变生硬。禁的是「首先 / 其次 / 综上所述」这类书面连接词，不是「最后」这个词本身。
- 这一步的效果**未验证**，7.3 有对应 A/B 项。可能出现的失败：模型既没用口语序数，也没写结构。退路是输出清洗阶段兜底剥离符号（见 3.7），但那样结构信息就真丢了。

**注意：861 字那篇是图文（`aweme_type == 68`），`①②③④` 是图片序号，本来就该有。** 它作为口播样本来喂是错配——学它的信息组织可以，但别拿它的符号形态当口播参照。选样本时优先视频类长稿。

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
- 不写序号符号：①②③、▎、▶、第 X 点、1. 2. 3.。要分层次就用口语序数说出来：「第一类，」「第二，」「接着是」「更关键的是」。
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

下面 {N} 条是同赛道对标作品的信息，分三类：标「完整口播稿」的是视频作者把全文贴进了简介，可以学开头、分段和结尾；标「图文长文案」的是图集作品的文字，可以学信息怎么组织，但不要学它的序号排版；没标的只有标题和简介，可以学选题角度和钩子，但不要照着它们的长度写口播稿。三类都不要抄句子：

【样本 1】
标题与简介：{desc 原文，普通样本截断到 300 字}
数据：点赞 {liked_count} / 收藏 {collected_count} / 评论 {comment_count}
值得学的是：{选题怎么切的角 / 标题怎么勾人 / 标签怎么选}

【样本 2 · 完整口播稿】
标题与简介：{desc 全文，不截断}
数据：点赞 {liked_count} / 收藏 {collected_count} / 评论 {comment_count}
值得学的是：{开头怎么设问 / 分段怎么排 / 结尾怎么收}

【样本 3 · 图文长文案】
标题与简介：{desc 全文，不截断}
数据：点赞 {liked_count} / 收藏 {collected_count} / 评论 {comment_count}
值得学的是：{信息怎么归类 / 清单怎么排优先级}

【样本 4】
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
| 默认 | 6–8 条；短 desc 截断 300 字，长 desc（≥500 字且判为正文）不截断 |
| 长样本 · 视频（`aweme_type == 0`） | 每批最多 1–2 条。唯一能学口播结构和节奏的样本，优先喂 |
| 长样本 · 图文（`aweme_type == 68`） | 每批最多 1 条。只学信息组织，别学序号排版；它的 `①②③④` 是图片序号 |
| 对标池里同题作品少 | 4 条也能跑，但风格会偏干，`temperature` 降到 1.1 |
| 想覆盖不同角度 | 高收藏 2 条 + 中位 2 条 + 争议性 2 条，混着给比全给头部稳 |

**怎么自动判「作者贴了正文」**：

```python
import re

def is_long_desc(desc: str) -> bool:
    """正文被贴进简介；只是长度判定，不代表形态"""
    tags = re.findall(r"#\S+", desc)
    tag_len = sum(len(t) for t in tags)
    return (
        desc.count("\n") >= 3
        and len(desc) >= 500
        and tag_len / max(len(desc), 1) < 0.10
    )

def sample_kind(item: dict) -> str:
    """video_full / image_text_full / short"""
    if not is_long_desc(item["desc"]):
        return "short"
    return "image_text_full" if int(item.get("aweme_type", 0)) == 68 else "video_full"
```

形态不要靠猜，`aweme_type` 已经在落盘字段里：**68 = 图文，0 = 视频**。实测 885 字那篇是 `0` + 有 `video_download_url`；861 字那篇是 `68` + 6 张图。

纯标题体是单行、短、标签占比高，不会被 `is_long_desc` 命中。

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

DESC_LIMIT = 300     # 普通样本字符上限。长样本不截断
MAX_VIDEO_FULL = 2   # 每批最多喂几条视频长稿
MAX_IMAGE_FULL = 1   # 每批最多喂几条图文长文案

def fmt_count(n):
    n = int(n)
    return f"{n/10000:.1f}w" if n >= 10000 else str(n)

def is_long_desc(desc: str) -> bool:
    """正文被贴进简介；只是长度判定，不代表形态"""
    tags = re.findall(r"#\S+", desc)
    tag_len = sum(len(t) for t in tags)
    return (
        desc.count("\n") >= 3
        and len(desc) >= 500
        and tag_len / max(len(desc), 1) < 0.10
    )

def sample_kind(item: dict) -> str:
    """68 = 图文，0 = 视频"""
    if not is_long_desc(item["desc"]):
        return "short"
    return "image_text_full" if int(item.get("aweme_type", 0)) == 68 else "video_full"

def dedupe(refs):
    """同日多次运行会重复写入，按 aweme_id 去重"""
    seen, out = set(), []
    for r in refs:
        if r["aweme_id"] in seen:
            continue
        seen.add(r["aweme_id"])
        out.append(r)
    return out

LABEL = {
    "video_full": "完整口播稿",
    "image_text_full": "图文长文案",
    "short": "",
}

def build_user(topic, refs):
    refs = dedupe(refs)
    head = (
        f"选题：{topic['title']}\n"
        f"值得做的理由：{topic['why']}\n"
        f"必须体现的独特角度：{topic['angle']}\n"
        f"制作难度：{topic['difficulty']}\n\n"
        f"下面 {len(refs)} 条是同赛道对标作品的信息，分三类："
        f"标「完整口播稿」的是视频作者把全文贴进了简介，可以学开头、分段和结尾；"
        f"标「图文长文案」的是图集作品的文字，可以学信息怎么组织，但不要学它的序号排版；"
        f"没标的只有标题和简介，可以学选题角度和钩子，但不要照着它们的长度写口播稿。三类都不要抄句子：\n\n"
    )
    blocks, used = [], {"video_full": 0, "image_text_full": 0}
    for i, r in enumerate(refs, 1):
        desc = r["desc"]
        kind = sample_kind(r)
        cap = MAX_VIDEO_FULL if kind == "video_full" else MAX_IMAGE_FULL
        if kind != "short" and used[kind] >= cap:
            kind, desc = "short", desc[:DESC_LIMIT]
        elif kind == "short":
            desc = desc[:DESC_LIMIT]
        else:
            used[kind] += 1
        label = f"【样本 {i} · {LABEL[kind]}】" if kind != "short" else f"【样本 {i}】"
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
6. **混合不同数据档位的样本**。全挑收藏最高的，模型会学着写夸张标题；掺 2 条中位但内容扎实的，稿子更像人。播放量拿不到，用收藏数当「值得反复看」的代理——但注意这是**内容类型**信号（工具清单类收藏高），不是质量信号，别拿它当「这条写得好」的唯一依据。
7. **样本里的排版符号要转译，不要照抄也不要删净**。`①②③` 是可念的序数，转成「第一类，」「第二，」；删掉会把层次一起删掉，照抄则会让 Lin 念的时候卡在符号上。详见 3.1.2。

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
2. 去掉 markdown 符号：行首 `#{1,6}\s*`、`\*\*`、`^\s*[-*]\s+`。
3. 兜底剥离排版符号：`▎` `▶` `①②③…` `^\d+[.、]\s*`。理想情况提示词已经把序号转成口语序数了，这里只是防漏网——直接删符号会把语义一起删掉，所以一旦这一步命中率高，说明提示词没生效，该回去查 3.2 而不是靠清洗补救。
4. 长度检查：口播稿 180–400 字（给点余量）。低于 180 说明被截断或没写够，重生成；高于 400 直接让模型压缩，不要自己截。
5. 长度异常排查：口播稿和样本 desc 的中位长度差一个量级，若生成稿 < 220 字，大概率是模型在模仿标题体。这种情况**不要**简单重生成，先确认提示词里那三类样本的标注还在。
6. 禁词检查：命中黑名单就重生成一次，仍命中就把稿子标记为「需人工改」，不要静默通过。
7. 洗稿检查：跟每条样本的 desc 做滑窗比对，连续重合就标红。**阈值按样本类型分三档**：
   - 短样本（标题/简介）：8 字
   - 视频长稿：12 字
   - 图文长文案：12–15 字。这类常是工具清单，含 GitHub / Cloudflare / Supabase / VS Code 这类专有名词，8 字阈值必然误报——工具名连着两三个就够 8 字了。误报的代价是把最好的参考样本判成抄袭。

   更稳的做法：比对前先把专有名词表（工具名、版本号、公司名）从两侧文本里剔除，再跑 8 字滑窗。剔除表可以随对标池一起维护。
8. `finish_reason == "length"` → 说明 `max_tokens` 不够，加预算重跑，不要用半截稿。

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
- [ ] **长样本不截断是否真的优于截断**（同一选题，1 条视频长稿 + 5 条短 vs 6 条全截断，各生成 10 条盲评）
- [ ] **序号转口语序数后，层次是否还在**（长样本喂进去，看输出有没有「第一类 / 接着 / 更关键的是」的推进，还是退化成一口气流水账）
- [ ] **视频长稿与图文长文案要不要分开 A/B**（两类形态不同，混在一组比「加不加结构声明」会互相污染结论）
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
- 长样本单独打标记，且**按形态分两类**：`aweme_type == 0` 是视频长稿（学结构节奏），`68` 是图文长文案（学信息组织，别学序号排版）。字段现成，不要靠 `note_download_url` 是否为空这类间接规则去猜
- 样本的序号符号做「转译」不做「删除」：`①②③` → 「第一类，」「第二，」。删掉会连层次一起删掉
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
| 采集侧只有 `desc`、没有播放量、1131 条长度分布、长样本结构与形态、重复落盘 | 转引自 `research/crawler-douyin.md` + crawler 对 2 条长样本的逐条核查（源码 + 本机落盘数据 + 实拍 `aweme_type`） |
| 「结构清晰带来收藏」这个因果 | **未验证，且证据方向相反**：34 条唯一评论里 0 条提到结构/排版；收藏反超点赞只出现在图文清单类（1.52×），口播类为 0.68×，更像内容类型特征 |
| 单条成本 0.003–0.006 元 | 按官方价格 + token 估算推导，非实测 |
| 提示词全文、禁令词表、20 字/句、5–8 条样本 | 经验判断，需 A/B 验证 |
| 「这不是完整口播稿」这句声明的作用 | 经验判断，未验证，见 7.3 |
| 温度 1.2、8 字重合阈值、180–400 字区间 | 经验值，见 7.3 待验证 |
| 本机无 key | 实查，见第 6 节 |
