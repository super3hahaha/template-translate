# template-translate

把 tester-app 的 Google Play 回复**模板**批量预翻译成多语言的 Claude Code skill。
由 tester-app「模板管理 → 补全多语言」非交互调用，结果写回 app 本地（`~/.tester-app/templates/`
的 `templates.json` 里每条模板的 `translations`），供 review-reply skill 命中后直接取用、省掉运行时翻译。

## 用法

非交互，唯一入口是一个 JSON 文件路径（顶层有 `templates` 字段）：

```
/template-translate <json路径>
```

输入 / 输出 schema 见 `SKILL.md`。

## 与 review-reply 的关系

- review-reply：评论 ↔ 模板匹配 + 运行时兜底翻译。
- template-translate：把模板**一次性预翻译**好存本地（本 skill）。
- 翻译纪律两者一致（邮箱/版本号/产品名/emoji/占位符不动、gp 350 字符）。

## 发版

push 到 `main` 由 `.github/workflows/auto-release.yml` 自动 patch+1 cut release。
tester-app 的 `skill_sync.rs` 拉 `releases/latest` 下发到 `~/.claude/skills/template-translate/`。
