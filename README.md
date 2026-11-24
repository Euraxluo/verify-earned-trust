# verify-earned-trust

一个 agent skill：把「流程 / 协议 / 架构」的验证固定成七步，交付反例、覆盖表与回流清单，而不是一句「已验证」。

给 agent 一个流程或设计，它会先盘点已有的强制手段，再按对象的形状选工具跑模型，最后用双 agent 拷问把文档缺口挖出来，并给每条缺口指定回流去向。

## 30 秒介绍（可直接转发）

> 一个验证流程 / 架构的 skill。它先盘点你已有的检查手段，用 Alloy 找结构反例、TLA+ 找并发反例；Lean 三条全满足才上，最后双 agent 拷问挖设计缺口。交付的是反例实例图、覆盖表和缺口回流清单，不是一句「已验证」。装进 agent 后说「验证这个流程」就会触发。

## 什么时候值得用

| 值得 | 不值得 |
| --- | --- |
| 多服务通路、权限边界、越权路径 | 一次性脚本 |
| 并发、重试、幂等、补偿、乱序 | 无并发无权限的 CRUD |
| 生命周期状态机、终态后再来事件 | 需求还没定，先做拷问 |
| 设计文档完备性评审 | 纯 UI 样式与文案 |
| 架构改造前后对比 | 只想跑个 lint |

判断口径：有并发、权限或时序，且错了代价高，就值得建模。

## 不懂形式化验证怎么用

你不需要会写 Alloy 或 TLA+。模型、运行、反例翻译都是 agent 的活，你只出三样东西：

1. **一个对象**：流程文档、设计文档或代码目录的路径。
2. **一句红线**：「什么绝不能发生」。说不出来就说「从文档里帮我抽不变量，列给我确认」。
3. **决策权**：缺口是改文档还是加 lint，由你拍板。

把这段直接发给 agent：

```text
用 verify-earned-trust 验证 <流程名>。文档在 <路径>，代码在 <路径>。
我不懂形式化验证，按这个顺序来：
1) 先给覆盖表（对象 × 已有验证手段），指出缺哪层；
2) 选出工具并建模，凡是我看不懂的术语用一句话解释；
3) 每条断言都要在故意弄坏的变体上看到一次红，把反例翻成「第几步、谁、什么值」；
4) 按四段报告交付，缺口写清去向（模型 / lint / 文档），不许写「待讨论」。
```

外行验收只看三条：见过一次红、反例是人话、缺口有去向。三条都满足，这次验证就成立。

想先预热：让它跑 `assets/static-model-example.als`（2 秒出结果），看懂输出再换自己的流程。文档没写清的，先做双 agent 拷问再建模；Lean 不要碰，除非有人愿意长期维护证明。


## 别人怎么用（三步）

1. 装上：`git clone … && ln -s "$PWD" ~/.claude/skills/verify-earned-trust`（其他 agent 放各自的 skills 目录）。
2. 说触发词：「验证这个流程 / 这个设计」「做形式化验证」「找流程 bug」，并把流程文档或代码位置给它。
3. 分工：事实由它自己查（文件、工具、接口行为），决策由你拍板；它每轮只问前置已落定的问题，不拿「你确认下 X」烦你。

## 一次典型对话

```text
你：验证一下退款流程，文档在 docs/refund.md，代码在 services/refund/
它：覆盖表（已有测试 / 缺权限矩阵 / 缺幂等检查）→ 建 Alloy 模型找反例
它：发现「已退款订单再次退款」可达，给出实例图 → 转 TLA+ 验并发交错
它：双 agent 拷问，出 12 题、逐题三分类 → 四段报告，缺口回流到 lint 与文档
```

## 流程在哪

agent 只先读 `SKILL.md`，按需再读 references；你自己查也一样。

| 你要做哪一步 | 打开 |
| --- | --- |
| 七步总流程、工具速查、反模式 | `SKILL.md` |
| 结构 / 权限 / 时序验证，运行管线 | `references/alloy-structural.md` |
| 抄现成模式写 `.als` | `references/alloy-patterns.md`（48 个） |
| 并发 / 协议 / 状态机 | `references/tla-plus.md` |
| 判断要不要上 Lean | `references/lean-gate.md` |
| 组织双 agent 拷问 | `references/grill-observer.md` |
| 把缺口固化成 lint | `references/pit-to-lint.md` |


## 为什么需要它

agent 说「已验证」时，通常只有文字规则。这个 skill 按强制力从重到轻选手段：

| 层 | 手段 | 例子 |
| --- | --- | --- |
| 1 | 代码库结构 | 分层、API 设计、既有模式 |
| 2 | 静态强制 | 类型系统、lint、import 契约、ast-grep |
| 3 | 运行时 + 形式化 | 测试守卫、Alloy、TLA+、Dafny |
| 4 | 文字规则 | 只兜底，不替代能机器强制的层 |

## 七步

1. **盘点**：列「对象 × 已有验证手段」覆盖表，复用已有的，缺口按层补。
2. **结构 / 不变量 → Alloy / Dafny**：通路断言、权限矩阵、生命周期时序、数据不变量。
3. **行为 / 协议 → TLA+**：并发交错、分布式协议、状态机矩阵，五件套缺一不可。
4. **Lean 准入门**：无界证明、纯数学命题、有人长期维护，三条全满足才上。
5. **双 agent 拷问**：拷问者出题、观察者逐题核对，两席不得同族。
6. **坑 → lint 固化**：原则转成 lint、import 契约或 ast-grep 规则，接入 CI。
7. **交付**：覆盖表 / 模型结论 / 拷问缺口三分类 / 未验证声明。

## 快速开始

```bash
git clone https://github.com/Euraxluo/verify-earned-trust
cd verify-earned-trust

# 装进 agent 的 skills 目录（选你用的 agent）
ln -s "$PWD" ~/.claude/skills/verify-earned-trust

# 跑一个自带模型：3 条断言全部 holds
./scripts/alloy_run.sh assets/static-model-example.als
```

运行 Alloy 需要 Java 17+ 或 Docker，脚本自动探测；首次运行下载 Alloy 6 jar，缓存在 `scripts/.alloy/`。其余工具按需安装：TLC 走 `tla2tools.jar`，Dafny 与 Lean 属于准入门之后的选项。

## 目录

| 路径 | 内容 |
| --- | --- |
| `SKILL.md` | 入口：七步流程、工具速查、反模式 |
| `references/alloy-structural.md` | Alloy / Dafny 定位、运行管线、四步质量闸 |
| `references/alloy-patterns.md` | 48 个生产验证模式（结构 / 时序 / UX / 管道 / 验证） |
| `references/tla-plus.md` | TLA+ 五件套、建模判据、结果怎么读 |
| `references/lean-gate.md` | Lean 三条件、cedar-spec 差分 oracle 范式 |
| `references/grill-observer.md` | 拷问者 / 观察者规程，缺口三分类 |
| `references/pit-to-lint.md` | 原则 → lint 自动固化流水线 |
| `scripts/alloy_run.sh` | 一键跑 `.als`：Java 17+ 优先，Docker 兜底 |
| `scripts/alloy_format.py` | 实例图美化输出 |
| `assets/*.als` | 4 个可跑模型，共 59 条断言、30 个 run 场景 |
| `agents/openai.yaml` | Codex 侧展示名 |
| `docs/` | 一页介绍（Archify 生成） |

## 自带模型

| 模型 | 断言 | 场景 | 验什么 |
| --- | --- | --- | --- |
| `static-model-example.als` | 3 | 3 | 软删与部分唯一索引 |
| `temporal-model-example.als` | 2 | 3 | 生命周期时序 |
| `ux-verification-example.als` | 12 | 8 | 角色 × 状态 × 字段矩阵 |
| `pipeline-example.als` | 42 | 16 | DAG 依赖与构建 |

实测（2026-11-23，macOS arm64）：`static-model-example.als` 3 条断言全部 holds，耗时 2.1 秒。

## 纪律红线

- 把有界穷举的结论说成「已证明」。
- 结果不对就改期望值，MISMATCH 不查哪边变了。
- 检查从未见过红，就当守卫。
- 拷问者与观察者同族模型，互相背书。
- 上 Lean 只因「听起来更强」。

## 触发

在已安装该 skill 的 agent 里说这些会命中：

- 「验证这个流程」「做形式化验证」「找流程 bug」
- 「验证架构」「review 这个设计有没有缺口」

## 介绍页

[docs/index.html](docs/index.html)（Archify 生成的单文件页面，离线可开）。

## 许可

MIT，见 [LICENSE](LICENSE)。`references/alloy-patterns.md`、`scripts/` 与 `assets/` 部分内容衍生自上游客 MIT 项目，署名见 [THIRD_PARTY.md](THIRD_PARTY.md)。
