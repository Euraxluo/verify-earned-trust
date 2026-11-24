# verify-earned-trust

一个 agent skill：把「流程 / 协议 / 架构」的验证固定成七步，交付反例、覆盖表与回流清单，而不是一句「已验证」。

给 agent 一个流程或设计，它会先盘点已有的强制手段，再按对象的形状选工具跑模型，最后用双 agent 拷问把文档缺口挖出来，并给每条缺口指定回流去向。

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
