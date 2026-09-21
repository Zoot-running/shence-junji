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
