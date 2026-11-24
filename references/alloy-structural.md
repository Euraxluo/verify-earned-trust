# Alloy / Dafny 结构验证 · 架构与不变量

架构验证的**主力工具**(Lean 只管语义无界证明,见 [lean-gate.md](lean-gate.md))。

- **Alloy 6**:框-边结构、权限矩阵、状态机、时序——有界反例自动搜索,反例即「违规实例图」;
- **Dafny**:数据不变量、前置/后置条件、无界证明,验证器直接对代码级断言。

## 何时选哪个

| 要验的 | 工具 |
| --- | --- |
| 架构通路(「Agent→MySQL 每条路径必经 sqlguard」) | Alloy 静态模型 |
| 权限矩阵(角色×状态×字段可见性) | Alloy UX 模式 |
| 生命周期/并发交错(「取消后不得发货」) | Alloy 时序(var sig) |
| 数据结构不变量、API 契约(无界) | Dafny |

## 运行管线(本 skill 自带,无外部插件依赖)

```bash
# Alloy:自动找 Java 17+(含 Homebrew/JAVA_HOME 探测),否则 Docker fallback;
# 首跑缓存在 scripts/.alloy/(jar + AlloyRunner)
<skill-dir>/scripts/alloy_run.sh <model.als>
```

实测(本机 Java 11 → Docker fallback 生效,soft-delete 断言 3/3 holds):
输出每条 `check` 断言 ✓ holds / ✗ Counterexample(带 witnesses 与实例图)。

## 建模工作流(formal-modeling 精髓,4 步质量闸)

1. **建模型**:静态(数据不变量/权限矩阵)、时序(状态机/生命周期)、UX/访问控制
   (角色×状态×字段)三型;facts + assertions + run 场景齐全才许跑;
2. **边界审查(质量闸)**:模型里每个系统元素标注 `Include / ExcludeSafe / ExcludeRisky / Stub`;
   风险排除必须带 gap assertion 说明丢了什么——防「空洞验证」;
3. **跑与解读**:反例=具体 bug 报告(哪个交错/实例破了断言);通过≠绿灯,要说明
   「这个 check 到底保证了什么」;
4. **对源对账**:每条断言对照真实代码/规格/文档,判 `Aligned / FixModel / FixSource /
   Drift / Exclusion`,报告落盘后才许进入下一轮迭代。

## 反空洞纪律

- **vacuous quantification**:量化体永假时断言空真——每条 check 配 `run` 场景证明
  前提可满足(详见 alloy-patterns.md #46 Expect Annotations);
- **从未见红的检查什么也证明不了**:交付前在故意破坏的变体上看到 Counterexample。

## 模式库与示例

- [references/alloy-patterns.md](../references/alloy-patterns.md):48 个生产验证模式,
  含目录(结构/时序/UX/管道/验证五族)——写新模型前先查目录抄模式;
- assets/ 四个可跑示例:`static-model-example.als`(软删/唯一索引)、
  `temporal-model-example.als`(时序状态机)、`ux-verification-example.als`(角色×状态×字段)、
  `pipeline-example.als`(DAG 依赖)——`scripts/alloy_run.sh assets/<file>` 直接跑。

## 来源与许可

模式库与运行脚本抽自 in8finity/claude-plugin(MIT)的 formal-modeling skill,
剥离 Claude 插件壳后作为自包含资源集成;升级时对照上游 diff。
