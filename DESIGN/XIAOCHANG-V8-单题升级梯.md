# XIAOCHANG v8.4 单题升级梯(从入队到 hint 的全流程)

> 2026-09-21。单题视角的升级全流程固化稿(机制全自动, 主 agent 只做"投思路/裁决"两个动作)。
> 全部触发条件为客观信号(旗/工件/fork/败绩轮数), 不读执行者自述措辞。

## 状态机与触发

```
入队 queued ──槽授予──▶ granted(时间盒 30min + 拍快照)
   ▲                        │ 执行者 settle → 机制分类
   │                        ▼
   │   ┌─ 有旗 ──────────▶ pending-flag(主 agent 交卷, 宽限 15min)
   │   ├─ 无旗 ──────────▶ settleNoFlag+1(迟到 settle 也计, 时间盒回队后不豁免)
   │   ├─ blocker 结论×1 ▶ verify-blocker(机制自动派验证兵; 确认→待裁决, 推翻→普通规则)
   │   ├─ 有进展(fork/工件新增) ─▶ rearm(progressStreak+1; 连 2 次→全开)
   │   ├─ 零进展×1 ──────▶ rearm-all-in(未试思路全派≤3路 + 机制自动 R2 加模型 + 多模型混打)
   │   ├─ 零进展×2 ──────▶ pending-adjudication(挂待裁决, 离开自动轮转)
   │   └─ 死路封印簇≥3 ──▶ 机制自动派验证兵翻案回合(verifierDispatched 每码≤1)
   │
   ├─ 时间盒到期 ────────▶ 关容器回队(账本保留, 梯刻度不动)
   ├─ 主 agent 裁决 continue/rotate ─▶ 回队(梯清零 / 换实例)
   └─ 主 agent 裁决 complete/failed ─▶ solved / dead(终态出队)
```

## hint 闸(升级梯最末级)

- 放行条件(客观判死): `settleNoFlag≥2`(不依赖 R2)| `ideaRound≥2 且 settleNoFlag≥1` | `blockerConfirmed`。
- 判死条件满足 → runner 把题放进 hintGateOpen 集合, **xiaochang_wait 主动推送"hint 闸已开(code)"** 并附评估指引(剩余分>扣分就取; 不取显式记理由并转集思多轮)。
- 主 agent 决策取 → 扣该题 ~10%(每题上限 1 次); 取完把 hint 转成新 directive 投回队列——**hint 是换方向, 不是解题**。
- 设计哲学: hint 保持最后手段, **不做"更早开闸"的强化**——高分靠升级梯的逐级广试探(多路混打/R2 加模型/验证兵/翻案兵), 不靠 hint。

## 与思路管线/账本的交织

- 每轮授予消耗一条未试 directive(派兵即消费); rearm-all-in 消耗全部未试思路多路混打。
- R2 二次征集(r2Due)在授予时机制自动发; fanout 报告落信箱 → wait 推"思路已回" → 主 agent 裁决采纳(idea_adopt 落账本①记未消费)→ enqueue 带 ideaIds 标记已消费。
- 死路条目带 testedVariants 实测清单; 封印簇(同向≥3)触发验证兵。
- pacing 约束(限速/封禁目标)注入每一轮执行令; settle 交接"未竟动作"自动转未走分叉注入下一轮。

## 实测参照(20777 c-03)

round1 无旗 settle → settleNoFlag=1 + 零进展×1 → rearm-all-in(R2+3路混打) → round2 无旗 settle → settleNoFlag=2 → **闸开 22:47:53, wait 推送, 主 agent 22:47:57 取 hint(4 秒响应)** → hint 转 fanout+七连 directive → 平台 16min 后截断。

## 已知的固有延迟与候选(不立即实施)

- 闸开时间 ≈ 2 轮 × 单轮时长(执行者跑满盒则 2×30min)——"技术栈不符/无攻击面"类结论的题可考虑下次授予时间盒减半以加速败绩累积(v8.5 候选)。
- b 链限速目标依赖 pacing 参数被主 agent 实际传用(已升为下局 order 硬条款)。


## 时序与上限问答(评审确认)

1. **首次生成执行者之前有集思吗?** 有——setup 自动发开局暖账 fanout(每题一路 deepseek-flash, hard≥700 加一路 glm-5.3)。但存在**开局竞态**: setup 同时武装全部题入队, 槽一空即授予(首波 1-2 分钟内落地), 暖账答案要 2-5+ 分钟才回——**首轮执行者多数拿不到集思答案**, 带的是主 agent 现场 directive(或兜底"按账本+画像自由突破")+ 模板帧 + 空账本。实测(20777): 首波授予 19:45 落在 b 链, 主 agent 19:44-19:47 才完成 54 次 idea_adopt——暖账思路主要喂给第 2+ 轮。首轮定位=快速侦察轮; 若坚持首轮带集思, 只能延迟授予等暖账(槽空转, 不值, 不做)。
2. **零进展×1 全开为何限 3 路?** `nSpawn = min(multiSpawn, 3)`(grantAndSpawn 单点可调)。3 = 同一授予内单容器并行执行者上限: 容器题多路执行者共享一个容器实例(同槽), 3 路是 8核16G 沙箱资源与 token 成本的经验权衡, 非硬道理。注意"全开"的真实并行 = 3 路执行者 + R2 fanout 加模型(集思路数另算)。
3. **零进展×1 先全派还是先征集?** 同一次授予内**先触发 R2(异步 fire-and-forget)、随后立即 3 路全派**; R2 答案 2-5 分钟后回来进思路管线, 服务下一轮。顺序理由: 等答案再派=槽空转 5 分钟, 不值。若想"本轮就用征集结果", 由主 agent 手动 jisi_fanout 提前铺(20777 主 agent 对 hard 题即如此)。


## v8.5 调度权回归主 agent(2026-09-21 用户裁定, 四点全过)

**哲学**: 机制只给"事实+动作", 调度决策全归主 agent; 不加硬限制; 目标是**把全容器资源用满**(容器内只有 DSH, 缺省全量可用)。

**事实(status)**:
- 每 granted 题: 在途执行者清单(itemId+已跑分钟) + 槽剩余时间盒分钟;
- 资源读数: 全容器 cgroup(memory.max/current, cpu.max/stat) + loadavg 兜底, status 展示 `resources: mem=used/max load=...`; 金柝告警继续作事件补充。

**动作**:
- `enqueue(dispatchNow=true)`: 题已持槽时立即用新 directive 派新执行者挂当前容器(与在途并行)——思路一回来当场上车, 不等下一轮;
- ~~`enqueue(spawn=N)`~~ **已废(v8.5.1)**: 开题时预置兵数=在错误时间点做决策(入队时看不到授予时的在途数/时间盒/资源), 且与动态派单冲突——见 v8.5.1 节;
- `report(interruptItemIds=[...])`: 精确收兵(全关/关几个/继续派都由主 agent 定); **人工收兵的执行者 settle 计"人工收兵"审计、不计无旗败绩**(否则主 agent 不敢提前收兵, 又退回跑满盒)。

**裁定记录**: ①收兵不计败绩=同意; ②dispatchNow 共享容器剩余时间盒、机制只给事实=同意; ③资源读数盯全容器 cgroup、尽量用满=同意; ④信任主 agent、不加硬限制=同意。

## v8.5.1 升级梯扁平化(2026-09-21 用户裁定: 扁平化 + 验证兵降级为建议)

**引子(用户两连问)**:
1. "spawn:N 和 dispatchNow 是不是有冲突? 开题前就指定最大 agent 数量, 对动态派单是限制——应该查当前在途数就行, 不要在开题时限制。" → **删 spawn 参数**。兵数是"看到事实之后"的决策, 不该是"入队之前"的预置; 授予路数回机制底线, 增量全靠 dispatchNow。
2. "升级梯的算法现在还成立吗?" → 分层审计结论: **计数层(settleNoFlag/zeroProgressStreak/progressStreak)与护栏层(时间盒/宽限/判死挂起)成立; 自动升级层(全开3路/auto-R2/末段auto-R2/自动验证兵)不成立**——那是机制在替主 agent 做调度决策, 正是"模型成了程序"的病灶。

**裁定(扁平化)**:
- 授予路数恒 1(机制底线); 删除 multiSpawn/r2Due/bloackerCheck 升级机制;
- settleAction 收敛为: pending-flag | rearm(有进展) | rearm-zero(零进展×1) | adjudicate(零进展×2);
- 零进展/连击只记账(事实)并靠 settle 唤醒 + status 提示行呈现; 加码=主 agent 按事实 dispatchNow, 集思=主 agent 显式 xiaochang_refanout;
- 末段 auto-R2 删除(末段赶工纪律已写进开战令, 由主 agent 执行, 机制不代劳)。

**裁定(验证兵降级, 用户原话)**: "死路降级到待裁决清单, 按簇分类, 给出每个簇的计数, 建议派兵验证。因为现在机制已经不知道运行情况了, 盲目派兵可能导致卡死。"
- sealedClustersOf(≥3 同向死路)只算事实 → 写 pendingAdj 待裁决条目: "死路封印簇: 方向×计数|… — 建议派验证兵翻案(附可抄的验证兵令文)";
- 机制不再自动派验证兵回合; blockerCheck 状态机整体删除; blocker 结论走普通零进展计数, 死路经 fork/report 落账由主 agent 读图判断;
- hint 闸的 blockerConfirmed 提前开门条件一并删除(闸 = 纯败绩计数事实, 无验证兵状态依赖)。

**与 v8.5 的关系**: 决策①收兵不计败绩、②共享时间盒、③资源用满、④无硬限制全部保留; 本次是同一哲学向"升级梯"内部的推广——**机制只剩: 事实(计数/读数/清单)、动作(派/收/重排)、护栏(时间盒/宽限/交还裁决)。**

