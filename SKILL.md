---
name: template-translate
description: >
  把 Google Play 回复模板批量预翻译成多种语言。输入是一个 JSON 文件路径（含 product +
  templates[]，每条带 text/lang/target_langs），输出同名 *.result.json（每个模板 id →
  各语言译文）。由 tester-app 通过「/template-translate <json路径>」调用，给模板管理页的
  「补全多语言」用；非交互批量翻译，只翻不匹配、不现编。
---

# Template Translate Skill

把 tester-app 的 Google Play 回复**模板**从源语言（`en` 或 `zh-CN`）**预翻译**成多种目标语言，
结果写回 app 本地，供 review-reply skill 命中模板时**直接取用**（省掉运行时翻译）。

- **非交互批量调用**：唯一入口是一个 `.json` 文件路径（顶层有 `templates` 字段），由 tester-app 写好后通过 prompt 传入。
- **只翻译，不做别的**：不匹配评论、不现编回复、不改输入文件。每条模板按它自带的 `target_langs` 逐语言忠实翻译。
- 输出写到与输入同目录的 `<stem>.result.json`。

> 与 review-reply 的分工：review-reply 负责「评论 ↔ 模板」匹配 + 运行时兜底翻译；本 skill 负责把模板**一次性预翻译**好存本地。两者翻译纪律一致（见下「翻译纪律」）。

---

## 输入

tester-app 写一个 JSON 文件，路径通过 prompt 传给 skill。结构：

```json
{
  "product": "MP3 Cutter",
  "channel": "gp",
  "templates": [
    {
      "id": "mp3cutter-001",
      "lang": "en",
      "text": "Hello there! Thank you for your valuable feedback...",
      "target_langs": ["ar", "cs", "de", "es", "ru", "zh-rCN", "zh-rTW"]
    }
  ]
}
```

字段说明：
- `product`：产品名（仅用于日志/上下文，不影响翻译）。
- `channel`：`"gp"`（Google Play，**默认**，每条译文目标 ≤ 350 字符）或 `"email"`（无限制）。缺省按 `"gp"`。
- `templates[]`：本批要翻译的模板。
  - `id`：模板 id，原样回填到输出。
  - `lang`：该模板**源语言**（`en` 或 `zh-CN`）。`text` 就是这个语言。
  - `text`：模板正文（源语言）。
  - `target_langs[]`：这条模板**本次要翻成哪些语言**（app 原生语言码，见下表）。后端已按「覆盖/只补缺失」算好，skill 照单翻译，不要自行增减语言。

### 语言码（app 原生码，原样作为输出 key）

| 码 | 语言 | 码 | 语言 | 码 | 语言 |
|----|------|----|------|----|------|
| ar | 阿拉伯 | fr | 法语 | pt | 葡萄牙 |
| cs | 捷克 | in | 印尼 | ro | 罗马尼亚 |
| de | 德语 | it | 意大利 | ru | 俄语 |
| es | 西班牙 | ja | 日语 | th | 泰语 |
| fa | 波斯 | ko | 韩语 | tr | 土耳其 |
| | | ms | 马来 | uk | 乌克兰 |
| | | nl | 荷兰 | vi | 越南 |
| | | pl | 波兰 | zh-rCN | 简体中文 |
| | | | | zh-rTW | 繁体中文 |

还可能出现更多 Android 资源码（app 端可勾选的扩展语言）：`bn`=孟加拉、`da`=丹麦、`el`=希腊、`fi`=芬兰、`hi`=印地、`hu`=匈牙利、`mr`=马拉地、`sk`=斯洛伐克、`sv`=瑞典，以及带 `-rIN` 区域后缀的印度语言 `kn-rIN`=卡纳达、`ml-rIN`=马拉雅拉姆、`pa-rIN`=旁遮普、`ta-rIN`=泰米尔、`te-rIN`=泰卢固、`ur-rIN`=乌尔都。

注意 `in`=印尼语（Android 码，不是 `id`）、`zh-rCN`/`zh-rTW`、`*-rIN` 都是 Android 资源码。**一律按 `target_langs` 给的码翻译、输出 key 与之一字不差**（含 `zh-rCN`/`zh-rTW`/`in`/`*-rIN`），不要改写成 ISO 码。

---

## 处理步骤

1. **读输入 JSON**（路径由 prompt 给出）。读 `channel`（缺省 `gp`）。
2. **逐条模板、逐目标语言翻译**：把 `text` 从 `lang` 忠实翻译到每个 `target_langs[i]`。
   - 源语言 == 目标语言的情况一般不会出现（后端已排除 `en` 自身），若真出现则原样返回。
   - 翻译纪律见下节。
3. **写出结果文件**：与输入 JSON 同目录，**把输入文件名的 `.json` 后缀替换成 `.result.json`**。
   例：`template-translate-1733600000.json` → `template-translate-1733600000.result.json`。
4. **【强制】写完自检 JSON 合法性**（tester-app 用 `serde_json` 严格解析，非法整批作废）：
   ```
   python -c "import json,sys; json.load(open(sys.argv[1],encoding='utf-8')); print('JSON OK')" "<out>"
   ```
   报错就修好再重写，直到 `JSON OK`。

> 模板可能很多（上百条 × 二十多语言）。可分批读写、用脚本组织，但**最终只产出一个 `<stem>.result.json`**，且必须是完整合法 JSON。

---

## 输出 JSON schema

```json
{
  "input": "template-translate-1733600000.json",
  "product": "MP3 Cutter",
  "results": {
    "mp3cutter-001": {
      "ar": "مرحبًا! شكرًا على ملاحظاتك القيّمة...",
      "de": "Hallo! Vielen Dank für dein wertvolles Feedback...",
      "zh-rCN": "你好！感谢你的宝贵反馈……",
      "zh-rTW": "你好！感謝你的寶貴回饋……"
    }
  },
  "warnings": []
}
```

字段定义：
- `results`：对象，key = 模板 `id`，value = `{ 语言码: 译文 }`。
  - **只包含本批输入里给的 id 和 target_langs**，不要多翻、不要漏翻。
  - 语言码与输入 `target_langs` 一字不差。
- `warnings[]`：处理中的问题（如某条翻译后超 350 已精简 / 跳过），每条带 `id` 说明。

---

## 翻译纪律（与 review-reply 一致）

### 0. 长度（按 channel）
- `gp`（默认）：每条译文目标 **≤ 350 字符**（含空格/标点/emoji）。
  - 源文已超 350 → 大概率是邮件模板，照实翻译但在 `warnings` 记一笔（不强切）。
  - 翻译自然膨胀到超 350 → **适度精简**保留主旨；压不下去就照实翻译并 `warnings` 记一笔，**不做「砍后半句」式硬切**。
- `email`：无限制，照实翻译。

### 1. 忠实、不增删、不编造
- 保持语义和语气（道歉 / 感谢 / 求评分 / 引导排查等），不改模板的称谓策略（`Hi friend` / `Dear user` 等保留风格）。
- 不添加源文没有的信息，不删源文的要点。

### 2. 原样保留（不翻译、一字不改）
- 邮箱（如 `filemanager.feedback@gmail.com`）。
- 版本号（如 `version 1.4.8`）。
- 产品名 / 专有名词（`XFolder`、`MP3 Cutter`、`Android`、`Google Play`）。
- emoji / 表情符号全部原样保留，位置不变。
- 占位符 / 变量（如 `{name}`、`%s`、`%1$s`）原样保留。

### 3. 写 JSON 的硬性纪律（否则整批作废）
- 译文里的双引号必须转义成 `\"`。**更稳妥：译文里尽量不用 ASCII 直引号 `"`**，要引用 UI 选项名时英文用 `'...'`，中文用「」。
- 换行写成 `\n`，不要裸换行。不要尾逗号、不要注释。
- 写完**一定**跑上面的 python 校验，只有 `JSON OK` 才算完成。

---

## 调用纪律

1. **不发 chat 文本**：非交互调用，最终只写出 `<stem>.result.json`，然后回报一行 `Written: <path>`。
2. **不改输入文件**，只读。
3. **不增减语言、不增减模板**：严格按输入的 `id` × `target_langs` 翻译。
4. 无法处理（输入损坏 / 文件缺失）→ 仍写出 `result.json`，`results` 留空，`warnings` 写明原因。

---

## 触发示例

由 tester-app 调用，参数是一个 JSON 文件路径：
```
/template-translate /Users/.../.tester-app/templates/template-translate-1733600000.json
```
（顶层有 `templates` 字段的 `.json` → 走批量翻译流程）
