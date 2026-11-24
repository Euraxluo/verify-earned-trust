# TLA+ 模型检查 · 五件套

流程/协议验证的主力手段。复刻 `formal/session_turn/` 套路(已实战验证的范式),五件套缺一不可:

```
formal/<name>/
  Model.tla          # 状态机;文件头注释锚定代码行/函数
  {current,fixed,isolation*}.cfg   # 矩阵:现状/修复后/只修一半
  run.sh             # TLC 断言「配置×不变量」与 EXPECTED 表一致(MISMATCH=有人改了东西)
  repro_probe.py     # 在真实实现上按反例序列打,确认缺陷形态与模型一致
  test_*.py          # 回归守卫:未修 xfail / 修复后转绿删标记
```

## 建模判据

- 不变量写「永不发生 X」(safety)。业务规则(如「code 只能换一次」)直接一条不变量,10 行内见分晓;
- `Next` 写成各 action 的析取(如 `YouPressPlay \/ YouPressStop`),否则 TLC 每步都叫 `Next`,反例没法读;
- 死锁:正确终态也会破 `NoDeadlock`——加一个只在终态为真且什么都不改的 action(如 `Done == phase \in {"paid","failed"} /\ UNCHANGED vars`);
- 状态爆炸控制:小集群(2-3 成员)+ 有界消息队列;隔离配置(只修一半)用于判断缺陷归属。

## 纪律

1. 不变量必须锚定设计条款号,不凭空发明;
2. 模型不许为了让实现变绿而改;建模假设写 VERIFICATION.md 并给代码依据(哪个 try 块、哪段持锁区间);
3. 已知局限老实登记在 VERIFICATION.md,不说成已解决;
4. 「从未见它红过的检查什么也证明不了」——交付前必须在故意弄坏的变体上看到不变量 VIOLATED(删一个检查/换两步顺序),再恢复;
5. 反例先翻译成「第几步破了什么」再讲给人听,不贴整段 TLA+。

## 工具链(三选一,按可用性)

| 工具 | 安装/调用 | 适用 |
| --- | --- | --- |
| **TLC 直跑**(默认) | `curl -sSLo /tmp/tla2tools.jar https://github.com/tlaplus/tlaplus/releases/download/v1.7.4/tla2tools.jar` + JRE 11+,`java -cp /tmp/tla2tools.jar tlc2.TLC <cfg>` | 已有 run.sh 封装的仓库 |
| **pstack statechart** | `<statechart skill dir>/scripts/statechart SPEC.tla [--config F] [--view V] [--out D]`(TS 运行时;首跑用 mise 自动装 TLC+Java,需联网) | 要**反例最短路径人话报告 + Mermaid 状态图**时;比裸 TLC 好读 |
| **Apalache**(符号化) | `brew install apalache`(或官方 bundle),`apalache check --inv=Inv model.tla` | 状态爆炸、TLC 穷不动时的符号模式检查 |

工具选择原则:能装的、下个 agent 一条命令能复跑的优先;TLC 为默认,statechart 用于汇报,Apalache 兜大状态空间。

## 反例可视化

- pstack statechart:每个不变量违规 → 最短路径人话;通过时画状态迁移 Mermaid 图;
- 或沿用 session_turn 的 `mermaid.py`:`python3 mermaid.py traces/current-<Inv>.txt`(反例)/ `--graph traces/*.dot`(全可达图)。

## 结果怎么读

| 现象 | 动作 |
| --- | --- |
| run.sh 全项 ok | 模型与登记期望一致,继续 |
| MISMATCH | 模型或实现被改动;先查哪边变了,再同步 EXPECTED 表与 VERIFICATION.md;不许改期望迁就结果 |
| fixed.cfg 某不变量 VIOLATED | 修复方案有漏洞;回模型找反例,别改实现试错 |
| 回归测试 XPASS | 缺陷已修;删 xfail 转正式守卫 |
| 回归测试由 XFAIL 变 FAIL | 缺陷形态变了(可能更糟);立刻停下查原因 |

## 相关本机 skill

- `/statechart`(pstack):TLA+ 检查 + 人话反例 + Mermaid;其命令/路径随版本变,**用的时候现场调,不要把它的命令抄进生成物**;
- `/create-formal-verification-skill`(pstack):把一次验证固化成项目本地 skill 的完整规程(覆盖记录+同步 commit 登记)。
