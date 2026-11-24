# Lean 准入门 · 三条件 + 架构验证的定位

Lean 是定理证明,不是找 bug 工具。仅当**全部三条**满足才上,否则留在 TLA+/Alloy:

1. **有界穷举不够**:需要「任意规模/任意交错」都成立,而不仅是穷举到的状态空间;
2. **性质是纯数学命题**:数据结构不变量、算法正确性、语言/策略语义——能写成无侧效应的数学定义与定理;
3. **有人类愿意持续维护证明**:证明工程是月级投入,不是一次性脚本。

## 推荐实践形态:Lean 规范神谕 + 差分测试(cedar-spec 范式)

业界标杆是 AWS 的 **cedar-spec**(github.com/cedar-policy/cedar-spec):

- `cedar-lean/`:Lean 4 形式化 Cedar 策略语言的**语义**(作者izers/实体/决策规则),证明核心定理,并提取可执行 checker 库;
- `cedar-drt/`:对 Rust 生产实现做 fuzzing/property-based **差分随机测试(DRT)**——同一批策略+请求喂给 Lean checker 与 Rust 实现,任何分歧即缺陷;
- 效果:Lean 的无界证明只压在「语义规范」这一小层,业务代码留在原栈;Rust 实现的 bug 由 DRT 自动捕获,不需要把 Rust 翻译成 Lean。

搬运到别的系统 = 「用 Lean 写清楚**协议/ DSL/ 算法的规范语义**并提取可执行 oracle,再对实现做差分」,而不是形式化整个架构。

## 架构验证:Lean 不是对的锤子

「架构图验证」(框、边、信任边界、通路性质,如『Agent 到 MySQL 的每条路径必经 sqlguard』)是**一阶关系结构**问题,工具选型:

| 对象 | 工具 | 说明 |
| --- | --- | --- |
| 结构断言(连通性/必经点/无绕行) | **Alloy**(alloytools.org,`brew install alloy`) | box-and-arrow 关系建模 + 自动找反例实例,天生干这个 |
| 图性质批量断言 | **Z3 SMT**(Python `z3-solver`) | 把架构编码成一阶约束,`solver.check()` 出模型/反例 |
| 行为/时序(状态机、协议) | TLA+/TLC/Apalache | 见 [tla-plus.md](tla-plus.md) |
| 语义规范需无界证明 | Lean 4 + mathlib | 仅在上方三条件满足时 |

架构 skill 现状:本机库(magpie/pstack/codex)**没有**现成 Lean 或 Alloy 架构验证 skill;
最接近的是 `archify`(画架构图,不验证)与 `create-formal-verification-skill`(生成
TLA+ 型验证 skill 的规程)。结构断言目前需要手工 Alloy/Z3 建模,或按
create-formal-verification-skill 的规程把「通路性质」转成 TLA+ 模型(把每条边建成
一个 action,不变量断言「不存在绕过闸的到达序列」,配合 statechart 出反例)。

## Lean 工具链(三条件满足时)

```bash
curl https://elan.lean-lang.org/elan-init.sh -sSf | sh   # elan
elan default stable                                        # Lean 4
# mathlib 按需;在线试:https://live.lean-lang.org/
lake build && lake test
# 差分 oracle 提取:cedar-lean 用 build_lean_lib.sh 产动态库给实现侧链接
```

## 何时果断放弃

- 三条件任一不满足;
- 性质本质是工程约束(部署拓扑/权限配置)→ Alloy/Z3 或 TLA+;
- 时间盒(如一周)内没跑通「玩具性质全证明」闭环 → 抽象层级错了,回 TLA+。
