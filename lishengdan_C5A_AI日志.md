# lishengdan_C5A_AI日志

> 覆盖：从 Fork 到写出 6 条改进建议的全过程。只记录**改变了产出**的交互。

## 1. 第一轮：让 AI 先读，我再验证

Prompt：

> 我 fork 了 neolaf2/neoskills（v0.4.2）。请只依据仓库里的文件回答四件事：
> ① 这个仓库一句话是做什么的 ② 目录结构（每个模块的职责） ③ 里面的技能有哪些、各自输入输出
> ④ 它用 symlink 部署的具体实现路径（给出文件与行号）。
> 不要推测，找不到就写"找不到"。

产出：结构梳理基本正确，但第 ④ 问它答成了"用 shutil.copytree 复制"——
实际 `core/linker.py` 用的是 `Path.symlink_to()`。**这是典型的"读 README 不看代码"。**
我把这段推翻，自己 grep 了 `symlink_to` 才写进分析。

## 2. 第二轮：装起来（AI 给的操作路径有坑）

AI 建议 `pip install neoskills` 装 PyPI 版。我改成源码 `pip install -e .`，理由是：
**要读的是这个 fork 的代码，装 PyPI 版会拿到别人的构建**。

坑：`requires-python = ">=3.13"`。本机 3.13.14 刚好够，但如果按 AI 最初建议的 3.11 会直接失败。

## 3. 第三轮：发现中文转义（AI 没提，是我看到的）

`neoskills create demo-skill -d "演示：把会议录音整理成纪要"` 之后我 `cat` 了生成的文件，
看到 `description: "\u6F14\u793A…"`。

我把这个问题丢给 AI："这是 bug 还是 YAML 规范行为？"
AI 回答得挺准：YAML 语义等价，但**对做字符串匹配的消费者（agent 触发逻辑）等于失效**。
并定位到 `src/neoskills/core/frontmatter.py:42` 的 `yaml.dump` 缺 `allow_unicode=True`。

**我的判断（不是 AI 的）**：这不是"显示不好看"，而是"技能触发链断了"，所以值得单独一个 PR，
而不是混在文档 PR 里。

## 4. 第四轮：写测试（AI 第一版太弱）

AI 给的测试只有一个 `assert "演示" in text`。我补成 3 项：
中文往返 / 纯 ASCII 不变 / 嵌套 dict 与列表里的中文也要保留。

理由：**i18n bug 通常在嵌套和列表里复发**，只测顶层字符串等于没测。

## 5. 第五轮：对比实验（我犯的错）

第一次跑"改动前基线"时忘了 `git stash`，带着修改 checkout 到 main，得到 158 passed，
但那是"main + 我的修改"。自己发现后重做：

```
main                14 failed, 158 passed
fix 分支            14 failed, 161 passed
```

差 3 = 新增用例数，说明 0 回归。**这个数字是重跑出来的，不是估的。**

## 6. 第六轮：AI 帮我找"看起来像 bug 但不是"的东西

我问："14 个失败是不是我改坏了？"
AI 让我在 main 上跑同一组测试——同样 14 个失败。据此确认是 Windows symlink 权限问题，
并在报告里写清"与本次改动无关，main 上同样失败"。**这一步避免了误判，也让 PR 描述可信。**

## 7. 人工写作的部分

- 全部 6 条改进建议的"证据 + 修法"（AI 只负责提供线索）
- `docs/zh-CN-quickstart.md` 的踩坑四条——这些都是我实际撞到的
- AAR 里的三个失败（clone 重试 6 次、对比实验被污染、最初只想写文档）

## 8. 数字

- 实际运行命令：约 40 条（含 6 次 clone 重试）
- 代码改动：2 文件 / +46 −1
- 文档：151 行
- 从 fork 到完成分析：约 2.5 小时
