# 分层验证：让 agent 的「已验证」有据可查 {#layered-verification}

`verify-earned-trust` 把流程与架构验证固定成七步。先盘点已有强制手段，用 Alloy 找结构反例，用 TLA+ 找并发反例。Lean 只在三条全满足时上，最后用双 agent 拷问补文档缺口，交付四段报告。

- 反例优先：Alloy 给违规实例图，TLA+ 给违规交错，任何人能复跑[](source:src-alloy)。
- 不迷信重工具：Lean 有准入门，三条缺一条就留在 TLA+/Alloy。
- 缺口必须有去处：拷问结果三分类，每条标注回流方向，不挂「待讨论」。
- 规则能落成 lint：坑→规则流水线把原则变成 CI 检查，先见红再合入。

@answer scope="团队内部分享 · 2026-11-23 · verify-earned-trust v0.1"
@meta locale=zh-CN style=editorial

@source src-alloy title="Alloy 6" url=https://alloytools.org
@source src-tla title="TLA+ tools v1.7.4" url=https://github.com/tlaplus/tlaplus/releases/tag/v1.7.4
@source src-cedar title="cedar-spec" url=https://github.com/cedar-policy/cedar-spec
@source src-lean title="Lean 4" url=https://lean-lang.org

## 七步流水线怎么跑？ {#pipeline}

核心判断只有一句：验证手段按强制力从重到轻选，代码库结构 > 静态强制 > 运行时与形式化 > 文字规则。文字规则只兜最后，不替代能机器强制的层。

@workflow id=pipeline-flow title="分层验证的七步"
| inv | 盘点 | backend |
| alloy-struct | Alloy 结构 | backend |
| tla-behavior | TLA+ 行为 | backend |
| gate | Lean 准入? | backend |
| proof | Lean 证明 | backend |
| grill-agents | 双 agent 拷问 | backend |
| lint-rules | 转 lint | backend |
| report | 四段报告 | frontend |
| inv > alloy-struct | |
| alloy-struct > tla-behavior | |
| tla-behavior > gate | 需要更强 |
| gate > proof | 是 |
| gate > grill-agents | 否 |
| proof > grill-agents | |
| grill-agents > lint-rules | 缺口 |
| lint-rules > report | |

@steps id=pipeline-steps
| st-inv | 盘点 | 列「对象 × 已有验证手段」覆盖表。复用已有的，缺口按层补。 | pipeline-flow | inv |
| st-alloy | Alloy 结构 | 通路断言、权限矩阵、生命周期时序、数据不变量。48 个模式先查目录。 | pipeline-flow | alloy-struct |
| st-tla | TLA+ 行为 | 并发交错、分布式协议、状态机矩阵。五件套缺一不可。 | pipeline-flow | tla-behavior |
| st-gate | Lean 准入? | 三条同时满足才上：无界证明、纯数学命题、有人长期维护。 | pipeline-flow | gate |
| st-proof | Lean 证明 | 只压语义规范，业务代码留在原栈，靠差分 oracle 对账。 | pipeline-flow | proof |
| st-grill | 双 agent 拷问 | 拷问者出题、观察者逐题核对，两席不同模型族。 | pipeline-flow | grill-agents |
| st-lint | 转 lint | 缺口里能机器强制的，转成 lint、import 契约或 ast-grep 规则。 | pipeline-flow | lint-rules |
| st-report | 四段报告 | 覆盖表、模型结论、拷问缺口三分类、未验证声明。 | pipeline-flow | report |

## 每类对象用哪把锤子？ {#tools}

入口都在 skill 目录里，命令可直接复跑。Dafny 与 Lean 属于准入门之后的选项，不是默认路径。

| 要验的对象 | 工具 | 入口 | 默认 |
| --- | --- | --- | --- |
| 架构通路、权限矩阵、时序 | Alloy 6 | `scripts/alloy_run.sh <model.als>` | 结构验证主力 {good:主力} |
| 并发、协议、状态机 | TLA+/TLC/Apalache | `references/tla-plus.md` 五件套 | 行为验证主力 {good:主力} |
| 数据不变量、API 契约 | Dafny | `dafny verify` | 无界证明才用 {caution:按需} |
| 语义规范 | Lean 4 | 三条准入 | 月级投入 {caution:少用}[](source:src-lean) |
| 设计文档完备性 | 双 agent 拷问 | `references/grill-observer.md` | 每页 ≥10 问 {good:必做} |
| 原则 → lint | ast-grep / import 契约 | `references/pit-to-lint.md` | 接 CI |

## Alloy 侧有什么现成弹药？ {#alloy}

模型库与运行脚本自带，不装插件。本机实测 2026-11-23：静态模型 3 条断言全 holds，耗时 2.1 秒。

@facts id=alloy-facts
| 模式库 | 48 个生产模式，分结构、时序、UX、管道、验证五族 |
| 可跑模型 | 4 个 .als，共 59 条断言、30 个 run 场景 |
| 运行管线 | Java 17+ 自动探测，找不到走 Docker fallback |
| 反空洞检查 | 每条 check 配 run 场景，证明前提可满足 |

| 模型文件 | 断言 | 场景 | 验什么 |
| --- | --- | --- | --- |
| `static-model-example.als` | 3 | 3 | 软删与部分唯一索引 |
| `temporal-model-example.als` | 2 | 3 | 生命周期时序 |
| `ux-verification-example.als` | 12 | 8 | 角色×状态×字段矩阵 |
| `pipeline-example.als` | 42 | 16 | DAG 依赖与构建 |

```bash title="跑一个现成模型"
git clone https://github.com/Euraxluo/verify-earned-trust
cd verify-earned-trust
./scripts/alloy_run.sh assets/static-model-example.als
```

边界审查是第四道闸：模型里每个元素标 `Include`、`ExcludeSafe`、`ExcludeRisky` 或 `Stub`。风险排除必须带 gap assertion，写明丢了什么，防「空洞验证」。

## TLA+ 五件套长什么样？ {#tla}

目录结构就是交付标准。`run.sh` 断言「配置 × 不变量」与 EXPECTED 表一致，MISMATCH 表示有人改了东西，先查哪边变了。[](source:src-tla)

@tree id=tla-tree title="formal 目录"
| root | formal/名称 | - |
| model | Model.tla | root |
| cfg | cfg 矩阵 | root |
| run | run.sh | root |
| probe | repro_probe.py | root |
| test | 回归测试 | root |

@callout id=tla-callout level=warning title="从未见红的检查什么也证明不了"
| 交付前必须在故意弄坏的变体上看到不变量 VIOLATED：删一个检查，或换两步顺序。
| 看到反例后先翻译成「第几步破了什么」，再讲给人听，不贴整段 TLA+。

反例也要落到真实实现：`repro_probe.py` 按反例序列打真实代码，确认缺陷形态与模型一致。回归测试保留 xfail，修复后转绿再删标记。

## 双 agent 拷问怎么防自我背书？ {#grill}

拷问者按设计树出题，一轮只问前沿问题，整个前沿问完再等回答。出题是它的活，事实也它自己查，只有决策交给人。

@sequence id=grill-seq title="双 agent 拷问"
| actor doc | 设计文档 | external |
| actor griller | 拷问者 A | backend |
| actor observer | 观察者 B | backend |
| actor team | 你 / 团队 | frontend |
| team | griller | 贴流程原文，只出题 |
| griller | team | ≥10 问，每题带推荐答案 | return |
| team | observer | 交题单与文档原文 |
| observer | team | 逐题三分类表 | return |
| phase | 缺口回流 |
| team | doc | 含糊项补边界条件 |
| team | doc | 缺口转模型或 lint | emphasis |

三分类只有三种：已覆盖并标注位置、含糊、缺口。缺口项全部要有回流去向，不许挂「待讨论」。

@callout id=grill-callout level=warning title="同族模型互相背书无效"
| 拷问者与观察者不得同族。观察者必须逐题出表，合并陈述视为无效。

## 坑怎么变成 CI 里的规则？ {#lint}

原则先形式化成可判定的命题，才可能变 lint。不可判定的原则不许硬塞 lint，转 TLA+/Alloy 或测试守卫，并标注 `[MODEL]` 或 `[TEST]`。

| 原则的形状 | 可判定形式 | 落点 |
| --- | --- | --- |
| 「X 禁止依赖 Y」 | 图上的边不存在 | import-linter / dependency-cruiser |
| 「X 禁止出现 API Z」 | 标识符不出现 | `ruff banned-api` / `eslint no-restricted-*` |
| 「X 只能经 G 访问 Y」 | 所有路径必经点 G | import 契约 + ast-grep |
| 「凡有 A 字段必带 B」 | 结构共现 | ast-grep / 自定义 ruff / mypy |
| 「状态 S 后不得动作 T」 | 时序不变量 | TLA+/Alloy + 测试守卫 |
| 「必须用 X 而非 Y」 | 禁 Y 逼 X | banned-api，msg 写替代方案 |

```yaml title="ast-grep：禁止绕过 review_svc 直连 ORM"
id: no-review-model-import
language: Python
rule:
  pattern: from services.review_svc.models import $$
```

每条规则先自证：写一个必然违反的样例，跑出红，删样例再跑绿。没见过红的规则不许合入，这条与模型检查的反空洞纪律同源。

## Lean 什么时候才值得上？ {#lean}

Lean 是定理证明，不是找 bug 工具。三条全满足才上，缺一条就留在 TLA+/Alloy。

| 准入条件 | 不满足时 |
| --- | --- |
| 有界穷举不够，需要任意规模都成立 | 留在 TLA+/Alloy，加边界场景 |
| 性质是纯数学命题，无副作用 | 架构约束转 Alloy 或 Z3 |
| 有人愿意长期维护证明 | 时间盒内跑不通玩具性质就回 TLA+ |

推荐形态照 `cedar-spec`：Lean 只写协议或 DSL 的语义规范，提取可执行 oracle，生产实现做差分随机测试，任何分歧即缺陷[](source:src-cedar)。这样无界证明只压在一小层，业务代码不动。

## 哪些做法不算验证？ {#anti}

这些做法会把验证变成假绿，评审时必须退回。

- 把有界穷举的结论说成「已证明」。
- 结果不对就改期望值，MISMATCH 不查哪边变了。
- 检查从未见过红，就当守卫。
- 拷问者与观察者同族模型，互相背书。
- 上 Lean 只因「听起来更强」。

## 团队怎么开始？ {#start}

先挑一条真流程，走完一圈。第一次别追全覆盖，先拿到一个能复跑的反例。

- [ ] 安装：把仓库软链到 agent 的 skills 目录，例如 `ln -s "$PWD/verify-earned-trust" ~/.claude/skills/verify-earned-trust`。
- [ ] 挑一条有并发或权限的真实流程，先写「对象 × 已有手段」覆盖表。
- [ ] 从 `assets` 选最接近的 `.als`，改一条断言，跑 `scripts/alloy_run.sh` 看到结果。
- [ ] 组织一次双 agent 拷问，两席不同模型族，要求逐题表格。
- [ ] 把至少 3 条缺口转成 lint 或模型，并在 CI 里看到一次红。

整包可拷走：目录没有私有依赖，Java 17+ 或 Docker 二选一。在已装该 skill 的 agent 里说「验证这个流程」「做形式化验证」「找流程 bug」会触发。完整规程在 `SKILL.md`。
