---
name: verify-earned-trust
description: Verify a system flow, protocol, or architecture in layers - inventory existing enforcement, model-check critical behavior with TLA+, verify structure and invariants with Alloy/Dafny, gate Lean by three conditions, and hunt design gaps with a griller/observer dual-agent pass. Use when asked to 验证流程/架构, 做形式化验证, 找流程 bug, or review a design for gaps.
---

# 分层验证

验证手段按强制力从重到轻选:代码库结构 > 静态强制(lint/类型) > 运行时+形式化验证 > 文字规则。文字规则只兜最后,不替代可机器强制的层。

对任何对象(流程/协议、架构结构、代码路径)按序执行:

1. **盘点**(必做,不许跳过):列「对象 × 已有验证手段」覆盖表,复用已有的,缺口在哪层就用该层手段补。只有无法机器强制的判断才落到文字规则。
2. **结构/不变量 → Alloy/Dafny**(架构验证主力):通路断言、权限矩阵、生命周期时序、数据不变量。自带运行管线与 48 模式库,详见 [references/alloy-structural.md](references/alloy-structural.md)。
3. **行为/协议 → TLA+ 模型检查**:并发交错、分布式协议、状态机矩阵。五件套纪律,详见 [references/tla-plus.md](references/tla-plus.md)。
4. **Lean 准入门**(三条件全满足才上):语义规范无界证明 + 差分 oracle(cedar-spec 范式)。不满足留在 Alloy/TLA+,详见 [references/lean-gate.md](references/lean-gate.md)。
5. **拷问-观察双 agent**(找设计缺口):详见 [references/grill-observer.md](references/grill-observer.md)。铁律:拷问者与观察者不得同族模型。
6. **坑/架构原则 → lint 自动固化**:把原则清单自动转化成各系统的 lint/import 契约/ast-grep 规则并接入 CI;验证环节的 AI 对照思考(设计是否违反约束)持续产出 GAP 回流此流水线。详见 [references/pit-to-lint.md](references/pit-to-lint.md)。
7. **交付**:四段报告——覆盖表 / 模型结论(不变量+条款锚点+矩阵对表+反例翻译)/ 拷问缺口三分类(标注回流去向)/ 未验证声明。

## 工具速查

| 验什么 | 用什么 | 入口 |
| --- | --- | --- |
| 架构通路/权限矩阵/时序 | Alloy 6 | `scripts/alloy_run.sh <model.als>` + [references/alloy-patterns.md](references/alloy-patterns.md) |
| 数据不变量/API 契约(无界) | Dafny | `dafny verify`(lean-gate.md 邻接,暂未内置脚本) |
| 并发/协议行为 | TLA+/TLC/Apalache/statechart | [references/tla-plus.md](references/tla-plus.md) |
| 语义规范(无界证明) | Lean 4 | 三条件见 [references/lean-gate.md](references/lean-gate.md) |
| 设计文档完备性 | 双 agent 拷问 | [references/grill-observer.md](references/grill-observer.md) |
| 原则→lint 自动固化 / 设计对照约束思考 | 自动化工序+GAP 回流 | [references/pit-to-lint.md](references/pit-to-lint.md) |

## 反模式

- 文字规则顶替可机器强制的约束;
- 改期望迁就结果(MISMATCH 先查哪边变了);
- 把有界穷举结论说成「已证明」;
- 量化体永假的空洞断言(每条 check 配 run 场景证前提可满足);
- 从未见红的检查就当守卫(交付前在破坏变体上看到反例);
- 拷问者=观察者同族模型互相背书;
- 上 Lean 只因「听起来更强」。
