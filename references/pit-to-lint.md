# 坑 → lint 固化 · 规则流水线

把踩过的坑/项目约束转成机器强制规则,是 L2 静态强制层的核心作业。
纪律:**坑先变测试,再变 lint,文档只留规则 ID 和一句为什么**——文字规则会被
agent 无视,lint 不会。

**本流水线是自动化的**:给 agent 一份架构原则/约束清单(或让它从设计文档/
ADR/AGENTS.md 里抽),它按下列工序自主产出各系统的 lint/import 契约/ast-grep
规则并接入 CI——人只审 PR,不手写规则。

## 自动转化总则(架构原则 → 可执行规则)

原则必须先**形式化成可判定的命题**,才可能变 lint。转化判定表:

| 原则的形状 | 可判定为 | 落到哪 |
| --- | --- | --- |
| 「X 禁止依赖 Y」 | 图上的边不存在 | import-linter / dependency-cruiser 契约 |
| 「X 禁止出现 API Z」 | 标识符/调用不出现 | ruff banned-api / eslint no-restricted-imports |
| 「X 只能经 G 访问 Y」 | 所有路径必经点 G | import 契约(层间禁止)+ ast-grep(直连模式) |
| 「凡有 A 字段必带 B」 | 结构共现断言 | ast-grep / 自定义 ruff 插件 / mypy 类型 |
| 「状态 S 后不得动作 T」 | 时序不变量 | lint 表达不了 → TLA+/Alloy + 测试守卫 |
| 「必须用 X 而非 Y」 | 禁 Y 逼 X | banned-api(msg 写替代方案) |

**不可判定的原则不许硬塞 lint**:转 TLA+/Alloy 模型或测试守卫,并在原则条目上
标注 `[MODEL]`/`[TEST]` 而非 `[LINT]`——假装可 lint 的规则只会制造假绿。

## 自动化工序(agent 执行,人审 PR)

输入:架构原则清单(设计文档硬规则/ADR/AGENTS.md/本次新增坑登记)。

1. **抽取**:从输入文档抽候选原则,每条挂来源锚点(条款号/文件行);
2. **分类**:按上面判定表打 `[LINT]` / `[IMPORT]` / `[AST]` / `[TYPE]` / `[MODEL]` / `[TEST]`;
3. **查重**:`ruff rule --all`、eslint 目录、既有 .importlinter/sg 规则——
   已有则配置启用,不重写;
4. **生成**(按目标系统落对应工件):
   - Python:ruff banned-api 段 / `.importlinter` 契约 / `*.sg.yml`(ast-grep);
   - TS/JS:eslint `no-restricted-*` / dependency-cruiser `depcruise.js`;
   - Go:`go vet` 自定义 analyzer / golangci-lint depguard;
   - 每条规则的 message 必须含:原则出处 + 为什么 + 替代方案;
5. **自证**(规则必须先见红):对每条新规则写一个**必然违反它的样例**,
   跑 `ruff check`/`lint-imports`/`sg scan` 确认真的报红,删样例再确认绿——
   没见过红的规则不许合入(同本 skill 反空洞纪律);
6. **接线**:进 pre-commit + CI;规则的复现样例留为测试;
7. **回写**:原则清单里该条目标注 `[LINT:<rule-id>]` 等强制标记——
   下次验证环节(见下节)AI 读清单就知道哪些已机器强制。

产出物固定布局(跨项目一致,agent 冷读可执行):

```
tools/constraints/
  principles.md        # 原则清单:条目+锚点+强制标记([LINT:x]/[IMPORT:y]/[GAP])
  rules/*.sg.yml       # ast-grep 规则
  .importlinter        # import 契约(或 depguard/depcruise 等价物)
  pyproject 追加段      # ruff banned-api 等
```

## 流水线(坑场景,五步,缺步即违规)

```
踩坑/约束 → ①登记 → ②定强制层 → ③先查现成规则 → ④没有才写自定义 → ⑤守卫+文档
```

### ① 登记(坑不登记=白踩)

在 `.ai-memory/daily.md` 或任务卡记:现象 / 根因 / 复现方式 / 想禁止的模式。
这是流水线的输入,没有登记的坑不进入转化。

### ② 定强制层(选最重的可行层)

| 层 | 工具 | 例子 |
| --- | --- | --- |
| 类型/编译器 | mypy strict、TypeScript strict | 「该字段永不 None」→ 类型即约束 |
| lint 规则 | ruff banned-api、eslint no-restricted-* | 「async 里禁 requests」「禁 useEffect」 |
| import 契约 | import-linter / dependency-cruiser | 「main/renderer 禁互导」「tool_layer 禁 import review_svc」 |
| AST 模式 | ast-grep(`sg`) | 语言无关,YAML 写模式,10 分钟一条 |
| 运行时守卫 | pytest xfail 守卫、pre-commit hook | 行为型坑(时序/并发)lint 表达不了 |

判定:坑是**结构/依赖**型 → import 契约;**API 误用**型 → banned-api;
**代码模式**型 → ast-grep;**行为**型 → 测试守卫,别硬塞 lint。

(此五步是上文自动化工序在「单个新坑」场景的缩影:登记即抽取+分类,
其余步骤同序执行。)

### ③ 先查现成(80% 的坑已有人踩过)

- ruff 规则目录(rules.docs.astral.is / `ruff rule --all`):bugbear(B)、pylint(PL)、bandit(S)系;
- eslint:no-restricted-imports / no-restricted-syntax / no-restricted-properties;
- 存在 → 配置启用,禁用消息里写**为什么+替代方案**,agent 读得到。
  ```toml
  [tool.ruff.lint.flake8-banned-api]
  ban-typed-literal-calls = true
  # 每条 ban 附 msg:"用 httpx;requests 在 async 里阻塞事件循环(坑:WPxx 2026-xx-xx)"
  ```

### ④ 自定义规则(真特殊才写)

```yaml
// ast-grep:sg-rule.yml —— 禁止 review_svc 之外 import 其 ORM 模型(越权坑)
id: no-review-model-import
language: Python
rule:
  pattern: from services.review_svc.models import $$
constraints:
  $$:
    regex: .
# 用法:sg scan --rule sg-rule.yml src/
```

Ruff 插件(需要更精细 AST 判断时):实现 `Rule` 类几十行,进 `lint.pydocstyle` 一样
的插件入口。自定义规则同样要进 CI/pre-commit,否则等于没写。

### ⑤ 守卫 + 文档收口

- 规则进 CI + pre-commit(机器强制,人和 agent 一视同仁);
- 违反规则的复现测试留为 xfail/正式守卫;
- AGENTS.md 只留一行:`「禁 X,规则 RUF/SG-xxx,因为 <坑链接>」`,细节指向规则自身。

## 验证环节的 AI 约束思考(design-vs-constraints 检查)

在 verify-earned-trust 的验证流程里,对**任何新设计/新代码变更**追加一道
「AI 对照思考」:让验证 agent 显式回答——这个设计是否违反项目已有约束?

输入(给验证 agent 的材料):

1. 被验对象(设计文档/代码 diff);
2. 项目约束清单(机器可查的部分现场跑,不可查的部分列文本):
   - `ruff check` / `lint-imports` / `sg scan` 的**实际输出**(有违反=直接 FAIL,不用想);
   - pyproject/.importlinter/sg-rule 配置全文(规则 ID + msg 里的「为什么」);
   - AGENTS.md/设计硬规则条文(§16.7 这类,lint 覆盖不了的);
3. 思考提示词(固定模板):
   ```
   对照以下约束逐条判断本设计/变更:违反/满足/不适用,给证据(行号/条款号)。
   对「lint 测不到但条款有要求」的项单独标注 [GAP-未强制] 并建议回流
   (加 lint 规则或 import 契约,按上文①-⑤流水线)。
   ```

输出格式(逐条表,不许合并陈述):

| 约束 | 来源(规则ID/条款) | 判定 | 证据 | 行动 |
| --- | --- | --- | --- | --- |
| tool_layer 不得 import review_svc | .importlinter:forbidden | 违反 | services/tool_layer/x.py:12 | 拒绝/改设计 |
| SQL 必经 sqlguard | 设计§16.7-3 | 满足(测试) | tests/.../test_sql_guard.py | — |
| 记忆不存行级数据值 | 硬规则6 | 满足(审查) | — | — |

判定分级:

- **违反(机器)**:lint/import 契约实际报红 → 直接 FAIL,回改,不需要 AI 判断;
- **违反(判断)**:AI 对照条款判违反 → 列证据进报告,人裁决;
- **[GAP-未强制]**:条款有要求但没有任何机器强制 → **这是流水线的新输入**,
  按上文①登记,②选层,③④固化——验证环节因此持续产出新 lint 规则,
  L2 层越用越厚,文字层越用越薄。

纪律:

- 机器能判的(③④已强制的)不许浪费 AI 思考,AI 只判「条款有、机器没测」的残余;
- 每次验证至少产出:约束对照表 + GAP 清单(哪怕空,写「无 GAP」);
- GAP 不闭环(不登记不固化)= 本次验证未完成。
