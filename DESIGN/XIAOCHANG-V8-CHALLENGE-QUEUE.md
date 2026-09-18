# 校场 v8 设计：题队列编排（挑战即队列单元，资源三合一原子授予）

> 2026-09-18 定稿。前置复盘见 `METHODOLOGY/RETRO-18728-19429-改动故障因果图谱.md`。
> 定位：校场 = 场景编排层，只带两样东西——①场景资源工具（平台原语适配器+账本/战报记忆件）
> ②调度方案（本设计）。基础设施四件（金柝/集思/虎符/行营）不动。
> 状态：**已实现并本地验证（2026-09-19）**。单测 62/62 绿（challenge-orch 内核 28 + 宿主集成 6）；dryrun-local LLM 局 8/8 全解（原子授予 17 对/回队无冷却/部分旗续打/升级梯 R2 实证）；十二场景 = 干跑证据 9 项 + 确定性集成测试 3 项（时间盒实测触发 7 次/裁决 continue 回队）交叉全覆盖。托管打包与开 run 待用户发话。

## 一、要立起的不变量（从五条失败链提炼）

1. **一个题在同一时刻最多属于一种状态**，且状态由机制维护，不靠主 agent 脑内清单；
2. **容器槽、执行者、账本思路三合一原子授予**：grant = 启容器（无则启）+ 从账本生成执行者 + 注入地址，一个动作完成；
3. **执行者开工即持槽**：执行者生命周期内不存在"排队等容器"状态，5 分钟摘除机制整体删除；
4. **容器调度决策权归主 agent，执行权归队列机制**：主 agent 决策（裁决/判死/置顶/换实例/hint），机制执行默认策略（授予/回队/升级梯/时间盒），subagent 零容器工具；
5. **未破题永不静默消失**：要么出旗，要么自动回队，要么在主 agent 的待决清单上（持续置顶直到裁决）。

## 二、核心模型

```
                  ┌────────────── 题队列 (challenge queue) ──────────────┐
                  │  key = unique_code（hufu ResourceQueue 已是 code 键） │
题: pending ────► queued ──grant──► granted(running, 执行者持槽)          │
  │                                  │ settle 结算                       │
  │                                  ▼                                   │
  │               出旗 ──► solved（关容器, 出队, 交主 agent 提交）          │
  │               未破 ──► 有进展? ──是──► 回队(不加路, 新兵继承账本)       │
  │                         │ 否(零进展)                                  │
  │                         ▼                                            │
  │               升级梯: ×1 全开(未试思路全派+R2全模型+多模型混打)         │
  │                       ×2 挂主 agent 裁决 + hint 闸开                  │
  │               时间盒到期(30min 无旗) → 关容器 + 回队(账本保留, 自动)     │
  │                                                                      │
  │   pending-adjudication 出边(主 agent 裁决后):                          │
  │     判死 → dead 出队(永不再自动入队)                                   │
  │     续打/拉hint/换实例 → 重新入队, 升级梯清零(zeroProgressStreak=0)     │
  │     不裁决 → 不自动入队(停止空转烧钱); 待决清单持续置顶, >30min 加⚠️   │
  │                                                                      │
  └──► 出队仅两条路: solved 或 主 agent 判死(blocker 类需验证兵确认)       │
```

## 三、数据结构（runner 侧新增/修改）

```ts
// 每题的编排态（新增, 与平台 progress 并列, 机制唯一写者）
interface ChallengeOrch {
  state: 'queued' | 'granted' | 'pending-adjudication' | 'solved' | 'dead'
  attempts: number                 // 已授予次数(回队+1)
  zeroProgressStreak: number       // 连续零进展次数(升级梯刻度)
  lastGrantAt?: number             // 上次授予墙钟
  lastSettleFingerprint?: string   // 上次 settle 结论指纹(同结论检测, 如 "攻击面缺失")
  priority: number                 // 主 agent 可调; 缺省=分值密度
  neverDispatched: boolean         // 从未被授予过(提权信号)
  grantedUntil?: number            // 时间盒到期墙钟(授予时= now+timeboxMs)
  directives: Directive[]          // 思路包(主 agent enqueue 写入+账本沉淀)
}

interface PendingAdjudication {
  code: string
  kind: 'needs-rotate' | 'needs-verdict' | 'hint-candidate' | 'blocker-verified'
  summary: string                  // 首行收敛格式: `code + 状态 + 需要什么动作`
  detailPath: string               // 细节在账本文件, 主 agent 按需展开
  createdAt: number
}

// 裁决出边(与 report 工具 verdict 一一对应, 机制执行去向):
//   判死     → state=dead, 出队, 关容器, 永不再自动入队
//   续打     → 重新入队, zeroProgressStreak=0, attempts 保留
//   拉hint   → hint 内容写账本①后重新入队, 梯清零
//   换实例   → close 容器后重新入队
//   不裁决   → 不自动入队(挂裁决=停止空转烧钱); 待决清单持续置顶, >30min 未裁决加 ⚠️ 强化提示

// 零进展判定(全部文件/计数级, 零模型主观):
//   settle 无旗 AND 战报新增发现行≤1 AND 新 fork 条目=0 AND 无新工件文件
// provider 故障类失败沿用现有签名过滤剔除
interface SettleProgress {
  win: boolean
  findingsDelta: number            // FINDINGS.md 开工前 vs settle 后行数差
  forkDelta: number
  artifactsDelta: number           // /opt/work/{code}/ 新增文件数
}
```

参数（首版默认值，均可由主 agent 裁决覆盖）：

| 参数 | 默认 | 依据 |
|---|---|---|
| `timeboxMs` 单次授予时间盒 | 30min | 19429 无种子局中位 7min / p90 27min，30 覆盖 p90；b-02 长链被切一次靠账本断点续打 |
| 升级梯 | 零进展×1 全开 → ×2 挂裁决+hint | 用户定稿 |
| 验证兵触发 | blocker 类结论×1 即派 | 19429 c-03 实锤两条同结论烧 4h |
| 从未开工提权 | 每 30min 未授予 +1 档（上限 3 档） | 兜底 e3-03/e3-04 类尾部题 |
| 回队冷却 | **无** | 零进展×2 挂裁决=天然限流器（题离开自动轮转）；槽空时队首立即授予（含刚 settle 的题），不浪费空槽；有进展长链题永不被冷却 |

## 四、工具面改动清单

| 工具 | 改动 | 说明 |
|---|---|---|
| `xiaochang_start_container` | **主 agent 侧删除** | 开容器唯一路径=队列 grant。19429 病灶（四次裸启动零派兵）物理消失 |
| `xiaochang_start_container` | **执行者侧删除** | 执行者持槽开工，无等待态；换实例走 settle 报告（needs-rotate）→ 主 agent 裁决 close+回队 |
| `xiaochang_enqueue` | 语义改为**入题队列** | 参数：code + priority + directives[]（思路包）。机制在授予时从 directives+账本生成执行者。仍主 agent 专属 |
| `xiaochang_dispatch` | 删除/降级为 no-op 提示 | 授予即派兵，主 agent 不再有"派单"动作 |
| `xiaochang_close` | 保留，主 agent 专属 | rotate/判死/时间盒到期时释放槽；执行者侧删除 |
| `xiaochang_report` | 扩展裁决面 | 新增 verdict 动作：`判死` / `续期(timebox)` / `回队(加路)` / `换实例` / `拉hint`。裁决后待决事项摘除 |
| `xiaochang_wait` | 事件面扩展 | 现有 settle 通知外，新增 grant/timebox-expiry/blocker-verified 事件 |
| `xiaochang_status` | 仪表重做 | ①待决事项置顶（未裁决持续重渲染）②未破题**全量**按风险排序（0 在途优先/从未开工优先/时间盒倒计时）③队列行=题+优先级 |
| `xiaochang_hint` | 闸门放宽 | `filteredFailedOf` 把 blocked 态计为失败（治 19429 c-03 "blocked 不算 filtered-failed" 堵死救援） |
| `xiaochang_fork` | blocker 检测 | 死路结论匹配 blocker 词表（无攻击面/攻击面缺失/环境缺失/平台未发布/不可达）→ 标记该题触发验证兵 |

## 五、执行者会话生命周期

1. 授予时机制生成执行令：`{code, addrs, boardPath, knowledgePath, directive(来自 directives 队首+账本①)}`；
2. 执行者开工第一动作仍是"读账本+战报"（18728 实证：零冲突衔接的关键纪律，保留）；
3. 验证兵模式：directive 首段="独立复验 blocker：重跑探测，输出确认/推翻+证据；推翻则继续解"；
4. settle 结算（机制自动）：win→solved；未破→SettleProgress 分类→回队/升级梯/挂裁决；
5. 同一授予槽内不派第二执行者（单兵先行）；加路发生在回队后的下一次授予。

## 六、hufu 改动（最小，原语不动）

- `ResourceQueue` 原语零改动（acquire/release/evict/waiters/grantedCount 已满足：code 键、同键合并、单一 grant 点）；
- 新增一个 runner 侧包装：`grantAndSpawn(code)`——grant 成功后构造执行令+spawn 会话+写 grantedUntil。位于 runner，不进 hufu。

## 七、dryrun 回归场景（tools/mock-tsecbench.mjs 扩展 + 新开战令）

| # | 场景 | 断言 |
|---|---|---|
| 1 | 3 槽 4 容器题：队头题授予时槽满 | 队列等待，无"排队位 5 分钟"文案（该状态已删除） |
| 2 | 执行者 settle 未破零进展×1 | 自动回队+全开：下一授予出现未试思路+新模型执行者 |
| 3 | settle 未破有进展 | 回队不加路，新执行者 directive 引用账本新增发现 |
| 4 | 30min 时间盒（mock 加速时钟） | 关容器+回队；长链题账本进度保留，续打从断点起 |
| 5 | blocker 结论 fork 一次 | 自动派验证兵；推翻→续打；确认→挂主 agent 待决 |
| 6 | hint 闸：blocked settle | filtered-failed 计数+1，闸门放行 |
| 7 | 尾部题从未开工 2h | 优先级自动提权，队列顺序变化可见 |
| 8 | 主 agent 无 start_container 工具 | 工具列表断言：主会话无该工具、执行者也无 |
| 9 | 回队无冷却：题 settle 未破且槽空 | 立即重新授予（含刚 settle 的题），空槽不闲置；有进展长链题不被冷却 |
| 10 | 挂裁决 → 主 agent 续打/拉hint/换实例 | 题重新入队，升级梯清零（attempts 保留），不再立即二次挂裁决 |
| 11 | 挂裁决 → 主 agent 判死 | 出队终态（dead），关容器，之后任何 wait/status 均不再出现该题 |
| 12 | 挂裁决 → 主 agent 不裁决 | 题不自动入队（停止空转烧钱）；待决清单持续置顶，>30min 加 ⚠️ |

## 八、实施里程碑

1. **M1**：数据结构+状态机（ChallengeOrch/PendingAdjudication/SettleProgress）+ 单元测试；
2. **M2**：原子授予（grantAndSpawn）+ 工具面改动（删 start_container/dispatch、enqueue 新语义、report 裁决扩展）+ 单测；
3. **M3**：回队+升级梯+时间盒+验证兵+优先级提权+仪表重做+hint 闸；单测；
4. **M4**：dryrun-local 十二场景全绿（mock 扩场景+新开战令）；
5. **M5**：同步 github；**打包/上传/开 run 等用户发话**（规则 6 审计照旧：seeds 禁入）。

## 九、明确不动清单

- hufu `ResourceQueue`/`campaign`/`binding` 原语；jisi 先验与 fanout 面；行营方法论；金柝；
- 打包管线与 hosted-pack-audit（规则 6 照旧）；submit 单点；知识账本四段结构；
- 军机"策略练兵场"（继续挂起）。
