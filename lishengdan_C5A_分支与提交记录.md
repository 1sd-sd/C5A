# lishengdan_C5A_分支与提交记录

> 状态：**两个分支已推送到 fork**（2026-11-04）；只剩开 PR 一步需要你在浏览器点一下（见第三节）
> 分支图（本地实际状态，与 GitHub 上的 fork 一致）：

```
* 0b0b991 docs: 新增中文入门指南 docs/zh-CN-quickstart.md      ← docs/zh-quickstart
| * 914d98b fix(frontmatter): 写出 SKILL.md 时保留非 ASCII 字符  ← fix/frontmatter-unicode
|/
* 7c4dada chore: bump to 0.4.2 — fix version string in installed package   ← main（fork 上游 HEAD）
```

---

## 一、完成的操作链

| # | 操作 | 命令 | 结果 |
|---|---|---|---|
| 1 | Fork 到自己的账号 | GitHub Fork 按钮 / API | `1sd-sd/neoskills`（此步在你要求"不用建仓库"之前已完成） |
| 2 | 克隆到本地 | `git clone git@github.com:1sd-sd/neoskills.git` | 112 个文件 / 1.4MB |
| 3 | 建分支（代码修复） | `git checkout -b fix/frontmatter-unicode` | — |
| 4 | 改代码 + 加测试 | 改 `frontmatter.py` 1 处，新增 3 项测试 | 161 passed（0 回归） |
| 5 | 提交 | `git commit -m "fix(frontmatter): 写出 SKILL.md 时保留非 ASCII 字符"` | `914d98b`，2 files / +46 −1 |
| 6 | 建分支（文档） | `git checkout main && git checkout -b docs/zh-quickstart` | — |
| 7 | 写中文指南 | 新增 `docs/zh-CN-quickstart.md` | 151 行 |
| 8 | 提交 | `git commit -m "docs: 新增中文入门指南 …"` | `0b0b991`，+151 |

---

## 二、两个改动分别解决什么

### 改动 1：`fix/frontmatter-unicode`（代码）

```diff
-    frontmatter = yaml.dump(metadata, default_flow_style=False, sort_keys=False).strip()
+    frontmatter = yaml.dump(
+        metadata, default_flow_style=False, sort_keys=False, allow_unicode=True
+    ).strip()
```

**为什么这是 bug 而不是风格问题**：
`neoskills create demo-skill -d "演示：把会议录音整理成纪要"` 写出的 description 是
`"\u6F14\u793A\uFF1A\u628A\u4F1A\u8BAE\u5F55\u97F3\u6574\u7406\u6210\u7EAA\u8981"`。
YAML 解析回来是对的，但任何**只做字符串/正则匹配**的 agent（包括 Claude Code 的技能触发）读到的是一串转义码——
中文技能等于"描述为空"，永远不触发。

**验证**：

```
main（改动前）        14 failed, 158 passed
fix 分支（改动后）     14 failed, 161 passed     ← 多出的 3 个是新增用例，0 回归
新增用例单独跑        3 passed
```

14 个失败全部是 `test_linker.py` / `test_resolver.py` 里的符号链接用例，
Windows 默认禁止普通用户建 symlink，与本次改动无关（main 上同样失败）。

### 改动 2：`docs/zh-quickstart`（文档）

151 行中文入门指南，不逐句翻译 README，而是补齐中文用户真正缺的三块：

1. **概念类比**：tap ≈ Homebrew tap，cellar ≈ Cellar，linker ≈ brew link
2. **命令速查**：21 个命令按"工作区 / 源管理 / 浏览 / 部署 / 创作 / 本体 / 运行 / 运维"分组
3. **实测踩坑**：Windows symlink 权限、中文被转义、`create` 骨架不能用、`HOME` 隔离无效

---

## 三、分支已在 GitHub 上，剩下开 PR 这一步

2026-11-04 已执行（成功）：

```bash
git push -u origin fix/frontmatter-unicode      # → 1sd-sd/neoskills:fix/frontmatter-unicode
git push -u origin docs/zh-quickstart           # → 1sd-sd/neoskills:docs/zh-quickstart
```

可直接点开核对：

- 代码修复：https://github.com/1sd-sd/neoskills/compare/main...fix/frontmatter-unicode
- 中文文档：https://github.com/1sd-sd/neoskills/compare/main...docs/zh-quickstart

PR 需要你在浏览器发起（对外可见的动作我不替你点，避免以上游维护者名义打扰）：

```bash
start "https://github.com/neolaf2/neoskills/compare/main...1sd-sd:neoskills:fix/frontmatter-unicode?expand=1"
start "https://github.com/neolaf2/neoskills/compare/main...1sd-sd:neoskills:docs/zh-quickstart?expand=1"
```

PR 描述可以直接用本文件第二节（"为什么这是 bug"那段话就是评审最需要的东西）。

> 提示：如果上游 main 有更新，先 `git fetch upstream && git rebase upstream/main` 再推，避免 PR 里出现无关 diff。

---

## 四、这次 Git 操作里我踩到的两个坑（真实记录）

1. **`git reset --soft HEAD~2` 报 "ambiguous argument"**：因为当时 HEAD 只有 2 个提交，
   而我按"3 个提交"的假设写命令。教训：`reset` 之前先 `git log --oneline` 数一下。
2. **`git checkout main` 前忘了 stash**：修改被带到 main 上，`pytest` 结果污染了对比实验。
   后来用 `git stash -u` 重做，才拿到干净的"改动前 158 passed"基线。
   **做 A/B 对比前，先确认工作区是干净的**——否则你测的是两份改动叠加的效果。
