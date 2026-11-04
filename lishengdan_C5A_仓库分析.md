# lishengdan_C5A_仓库分析

> 分析对象：`neolaf2/neoskills` v0.4.2（MIT）｜ 分析方式：**不止读代码，实际装了一遍并跑通**
> 环境：Windows 10/11 · Python 3.13.14 · git 2.55 · 本地克隆 `1sd-sd/neoskills`
> 证据：见同目录 `证据/01`–`05`

---

## 一、这个仓库是做什么的（一句话）

**技能界的 Homebrew**：把分散在各个 agent 目录里的技能原件收进一个 git 仓库（tap），再用**逐技能符号链接**部署到 Claude Code / OpenCode / OpenClaw，并在技能之上叠一层**属性图（ontology）**做发现、依赖、生命周期管理。

它解决的不是"怎么写技能"，而是"技能多了以后怎么管"。

---

## 二、目录结构

```
neoskills/
├── README.md / CLAUDE.md / TASKS.md     # 用户文档 / AI 协作指南 / 进度与路线图
├── pyproject.toml                        # Python 3.13+，click + rich + pyyaml + jinja2 + gitpython
├── skills/                               # 仓库自带的两个技能
│   ├── bank-status/
│   └── skill-dedup/
├── agents/                               # 4 个 agent 定义（scanner / importer / deployer / dedup）
├── commands/ns.md                        # 斜杠命令入口
├── docs/                                 # ontology-design.md + plans/
├── src/neoskills/                        # 65 个 py 文件，7531 行
│   ├── core/          (15)  # workspace / cellar / tap / linker / resolver / frontmatter / index / manifest / checksum / namespace / mode / auth / config / models
│   ├── cli/           (18)  # 21 个命令，一命令一文件（含 ontology_cmd）
│   ├── ontology/      (13)  # graph / loader / writer / engine / taxonomy / lifecycle / composition / export / versioning / scaffold / models
│   ├── adapters/       (6)  # 不同 agent 的目录约定适配
│   ├── runtime/        (3)  # claude 插件运行时（MCP 工具）
│   ├── plugin/         (3)  # 插件打包
│   ├── meta/           (2)  # 元技能（对技能本身操作）
│   └── translators/    (1)  # 跨 agent 格式翻译
└── tests/                                # 172 个用例（unit + integration）
```

**规模读数**：112 个文件 / 1.4MB / 7531 行 src / 172 用例 / 21 个 CLI 命令 / 2 个内置技能。

---

## 三、里面有哪些技能，各自做什么

仓库自带技能只有 2 个，但 `agents/` 里另有 4 个"做技能管理"的 agent 定义。

| 技能 | 做什么 | 输入 | 输出 |
|---|---|---|---|
| `bank-status` | 汇报技能库现状 | 读 `~/.neoskills/registry.yaml`、`state.yaml`、`LTM/mappings/targets/` | 银行总量 / 目标列表 / 已嵌入 / 最近 5 条导入 |
| `skill-dedup` | 跨库去重 | 扫描 bank、target 目录、`~/.claude/plugins/`、远程 repo | 三类重复的清单 + `--resolve` 可自动合并（带 `.dedup-backup` 备份） |
| `agents/skill-scanner.md` | 扫描目标目录，发现技能 | target 路径 | 技能清单 |
| `agents/skill-importer.md` | 把目标里的技能导入 bank | 技能名 | tap 内的规范化副本 |
| `agents/skill-deployer.md` | 部署（建链接） | 技能名 + target | symlink + 健康检查 |
| `agents/skill-dedup.md` | 去重的 agent 化包装 | — | 同 skill-dedup |

### 最有意思的一个：`skill-dedup`

理由不是它代码最多，而是它**处理的是技能系统的熵增**——技能库用久了必然出现三种病：

1. **完全重复**（SHA256 相同）→ 可以安全合并
2. **同名分叉**（ID 相同、内容不同）→ 要判断哪份"更富"（文件更多）再回灌
3. **名字相似**（不同 ID、描述相近）→ 必须人工裁决，不能自动合并

它把"看起来一样但其实不一样"的三件事拆开，并给不同的处理策略——这是**运维思维**，不是脚本思维。
更关键的是它的默认行为：`--dry-run` 先预览、改动前留 `.dedup-backup`、只对 exact 做自动合并。**自动化只用在确定性高的那一类上**，这个克制值得抄。

---

## 四、架构：四个值得学的设计决策

### 1. 文件系统即数据库（没有 state.yaml）

`core/linker.py` 的 docstring 写得很清楚：*"No state.yaml — derives all state from filesystem inspection."*

- **好处**：不会出现"状态文件说已链接、实际没链"的漂移；删掉目录就等于恢复。
- **代价**：每次操作都要扫盘（`TASKS.md` 里 "graph persistence cache" 就是为这个还债）。
- **判断**：在 100–500 个技能的量级上，这个取舍是对的——一致性 bug 比性能更贵。

### 2. 零拷贝部署（逐技能 symlink）

```
~/.claude/skills/kstar-loop  -->  ~/.neoskills/taps/mySkills/skills/kstar-loop
```

一个真源、多 agent 共享、改完立即生效、`unlink` 不留垃圾。
这比"复制 + 定期同步"高一个维度：**复制会产生分叉，链接不会。**

### 3. 本体用 sidecar，运行时物化成内存图

每个技能旁挂 `ontology.yaml`，`SkillGraph`（453 行）在内存里建：
- `nodes` / `edges` / `domains` 三个主存储
- `_by_domain` / `_by_type` / `_by_state` 三个倒排索引 → 分面查询 O(1)
- 明确写着 "Designed for ~100-500 skills"

**不用 Neo4j 也能有图**，这是工程判断力：图数据库的运维成本远高于它在这个量级带来的收益。

### 4. 渐进增强（L0→L3）与生命周期状态机

| 级别 | 内容 | 谁来做 |
|---|---|---|
| L0 | 只有 SKILL.md | 默认（存量技能零成本接入） |
| L1 | + type/domain/tags | `ontology enrich` |
| L2 | + 边（requires/extends/composes_with/conflicts_with/supersedes/derived_from） | 作者声明 |
| L3 | + 生命周期/版本/能力清单 | 长期维护 |

`LifecycleState.valid_transitions()` 是一个显式状态机：
`candidate → validated → operational → refined → deprecated → archived`，还允许 un-deprecate。

**这个设计的价值**：它承认"技能会腐烂"，并把腐烂变成可管理的状态，而不是假装不会发生。

---

## 五、我实际跑了一遍（不是只读代码）

| 步骤 | 结果 |
|---|---|
| `pip install -e .`（venv，Python 3.13.14） | 成功 |
| `neoskills --version` | 0.4.2 |
| `neoskills --help` | 21 个命令 |
| `neoskills init` | 建出 config.yaml + taps/ + cache/ |
| `neoskills create demo-skill -d "演示：把会议录音整理成纪要"` | 生成 SKILL.md + ontology.yaml，自动推断 domain=general |
| `neoskills ontology load` | 图里 1 节点 1 边（belongs_to），enrichment L1 |
| `neoskills doctor` | System is healthy |
| `pytest` | 172 项中 **158 通过 / 14 失败**，失败全在 `test_linker.py`、`test_resolver.py`，原因是 Windows 默认不允许普通用户建符号链接 |

**这一步改变了我的判断**：README 读起来是"跨平台"，实际在 Windows 上，它最核心的能力（symlink 部署）需要管理员权限或开发者模式。文档里没提。

---

## 六、改进建议（6 条，都有实测证据）

### 建议 1：`write_frontmatter` 不要转义中文（已修，见分支）

`src/neoskills/core/frontmatter.py:42`

```python
yaml.dump(metadata, default_flow_style=False, sort_keys=False)   # 缺少 allow_unicode
```

实测 `neoskills create demo-skill -d "演示：把会议录音整理成纪要"` 写出的是：

```yaml
description: "\u6F14\u793A\uFF1A\u628A\u4F1A\u8BAE\u5F55\u97F3\u6574\u7406\u6210\u7EAA\
  \u8981"
```

YAML 语义等价，但 ① 人不可读 ② **只做字符串/正则匹配的 agent 读不到中文描述**，技能触发率直接归零。

修复：加 `allow_unicode=True`；补 3 项往返测试。已在 `fix/frontmatter-unicode` 分支提交（161 passed，无回归）。

### 建议 2：`create` 生成的骨架应该在结构上就"能用"

实测产物（证据 03）：

```markdown
# Demo Skill

TODO: Add skill content here.
```

- `description` 只有一句（10 个字符），远低于可靠触发的长度
- 正文只有 TODO，没有"何时用 / 输入 / 步骤 / 输出 / 失败处理"

用 `skilllens` 体检这个脚手架：**39/100，D 级**（结构 0/20、可执行 0/24）。

建议：脚手架直接生成六段骨架（What / When / Input / Steps / Tools / Output + 失败路径），
并在 `create` 结束时跑一次结构自检，把缺失项打印出来。**新建即合格**比"事后补"便宜得多。

### 建议 3：CI 只跑 Ubuntu，Windows 问题没人发现

`.github/workflows/ci.yml` 的 job 全是 `runs-on: ubuntu-latest`，而 14 个用例在 Windows 上必失败（symlink 权限）。
后果：Windows 用户第一次 clone 就跑出一片红，会以为仓库坏了。

建议：加 `windows-latest` matrix，并对需要符号链接权限的用例加
`@pytest.mark.skipif(sys.platform == "win32" and not _can_symlink(), reason=...)`，
再在 README 的 Requirements 里写一行 "Windows 需开启开发者模式"。

### 建议 4：工作区根目录写死 `Path.home()`，无法隔离

`src/neoskills/core/workspace.py:34`：`self.root = root or Path.home() / ".neoskills"`

代码层支持传 `root`，但 CLI 没有 `--root`，也没有环境变量。我实测 `export HOME=/tmp/nshome` 后
`neoskills init` 仍然写进了 `C:\Users\L1983\.neoskills`。

后果：① 单元/集成测试互相污染 ② 一个人不能同时维护两套技能库 ③ CI 里没法做干净环境。

建议：加 `NEOSKILLS_HOME` 环境变量 + 全局 `--root` 选项，优先级 `--root > env > ~/.neoskills`。

### 建议 5：`ontology load` 每次全量扫盘

`TASKS.md` 已列为待办（graph persistence cache），我这里补一个量化角度：
当前 `load` 是 walk 全部 `ontology.yaml` + 解析 YAML，1000 个技能量级约 0.5–1s（YAML 解析为主）。
建议缓存到 `.neoskills/cache/graph.json`，用 **目录 mtime + 文件数**做粗粒度失效，而不是逐文件 mtime（逐文件反而更慢）。

### 建议 6：搜索没用上本体的倒排索引

`neoskills search` 是子串匹配，而 `SkillGraph` 已经有 `_by_domain/_by_type/_by_state` 三个倒排索引。
既然已经付了建图的成本，`search --domain` / `--type` / `--state` 几乎没有额外开销。
建议把 `search` 和 `ontology discover` 合并成一条入口，避免用户记两套语法。

---

## 七、我从这个仓库学到的（关于技能设计）

1. **分发与质量是两层，不能混在一起。**
   neoskills 管"装得对"（版本、链接、依赖），但不管"写得好"。我之前把这两件事混在 C4 技能里，
   结果是每个技能都在重复实现"怎么被找到"。分层之后，两边都能做深。

2. **元数据要渐进，不能强制。**
   强制 schema 会让存量技能无法接入；L0 起步 + 逐级 enrich，才有人愿意用。我自己的 skilllens 也照这个思路改过：
   N/A 规则不进分母（不强迫每个包都有 references）。

3. **"文件系统即状态"是一种可复制的降复杂度手法。**
   凡是"状态文件 + 真实状态"两份数据的地方，最后都会漂移。少维护一份真相，就少一类 bug。

4. **自动化的边界要按确定性划。**
   skill-dedup 只对 SHA256 相同的做自动合并，同名分叉只给建议，名字相似交给人。
   这比"全都自动"安全得多——**不确定的判断交给人，是设计而不是退让。**

5. **承认腐烂，才能管理腐烂。**
   生命周期状态机 + deprecated 状态，让"这个技能过时了"变成一个正常的、可记录的事件，
   而不是"默默没人用"。

---

## 八、结论

`neoskills` 是一个**工程判断力明显高于平均水准**的项目：21 个命令、172 个用例、7531 行代码，
却刻意避开图数据库、避开状态文件、避开强制 schema——每一次"不做"都有理由。

它的短板集中在**跨平台与脚手架质量**：Windows 上核心能力受限且未在文档声明，
`create` 生成的骨架不能直接用。这两点都不是架构问题，是"最后一公里"问题，
而这恰好是我这次实际跑过才发现的——**读 README 永远发现不了这些。**
