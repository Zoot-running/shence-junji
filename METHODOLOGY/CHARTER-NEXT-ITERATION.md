# 立项书:下一迭代(全局解题图账本 + 能力账本种子)

> 2026-09-13,v15 局后与用户讨论定稿。背景:v13 满分榜一已固化;
> 本迭代修三件事,全部围绕"经验不该从零开始、分支不该丢失、主 agent 要有账"。

## 1. 目标(字典序)

- **硬目标(唯一)**:三项机制落地并各自通过最小验证——
  ①execution 能力账本种子进托管镜像;②执行者终态结构化报告(deadEnds/forks/
  observations);③主 agent 全局解题图账本(xiaochang_graph + 派单自动携带)。
- 次要 1:下一正式局用上①②③且花费回落(v15 的 ¥213 → 目标 ≤¥50 量级);
  次要 2:不破坏 v13 已验证的满分链路(guard/wait/收割专员全部保留)。
- 非目标:不再改平台链路(网关 20s/轮 是环境事实)。

## 2. 约束

- 硬:托管规则 6 合规——进镜像的账本种子**只含 execution 维度**(模型×难度×胜负,
  已审无题目内容);idea 维度/画像/死路历史**不进镜像**;凭据仍走 env_config。
- 软:复用虎符快照做图账本的持久化载体(guard 重拉不丢);"靠配置不靠纪律"。

## 3. 机制设计

### ① 能力账本种子(jisi + 镜像)
- 本地 `jisi-model-ledger.json` 提取 `models[].dimensions.execution`(样例:
  flash easy 21/21、hard 15/15、medium 28/28——flash 是已验证主力)打成种子文件;
- 镜像打包;沙箱内 jisi 启动时若账本为空则装载种子;
- 效果:选模型不再从零试探,v15 的"flash+k2.6 双路"账单根除。

### ② 执行者终态结构化报告(runner + hufu)
- `xiaochang_report` 的 detail 之外加结构化字段:
  `deadEnds:[{path, conclusion, evidence}]`、`forks:[{branch, whyUntried}]`、
  `observations:[fact]`;
- hufu feed 时解析并按题存入图账本。

### ②b 执行中分叉即时报(F32)——分叉发生的瞬间就变成调度输入
- 新工具 `xiaochang_fork`(执行者会话可用):执行者遇到"两步/多步思路"时立即调用,
  参数 {branches:[{branch, whyViable, needs}], chosen: index}——
  **自己选一路继续走,其余分叉连同上下文立刻交给主 agent**;
- 机制:runner 把 fork 条目写入该题图账本(forks),并**立即向主 agent 会话投递
  通知消息**(与 settle 通知同通道)——正在 xiaochang_wait 阻塞的主 agent 被
  "会话消息"唤醒,读 fork 上下文后**马上 enqueue 未走分叉**(新种子、prompt
  自带该分叉的 whyViable/needs),并行度即时放大;
- 效果:分叉从"终态报告里的一条记录"升级为"运行中的实时调度事件"——
  主 agent 不用等执行者收工就能开新兵。

### ③ 全局解题图账本(hufu + runner)——主 agent 的账
- 结构:item → entries[{kind, path, conclusion, evidence, by, at}];
  存于虎符战役快照(随快照持久化,guard 重拉后新会话读图即恢复全局认知);
- 消费 A:`xiaochang_enqueue` 自动把该题"已知死路+未走分叉+事实"附进新执行者
  prompt(任何种子/模型)——重复试错归零、分支不丢;
- 消费 B:新工具 `xiaochang_graph`(或并入 xiaochang_status)展示每题路径 DAG
  (未试/进行中/已证死/未走分叉/已解)——主 agent 掌控全局的依据;
- 分叉入队:forks 变成可派的新任务种子(同一题、新 seed、prompt 自带上下文)。

## 4. 验收口径

1. 种子账本进镜像、审计 CLEAN(无题目内容)、沙箱内 jisi_model_report 显示种子数据;
2. 结构化报告字段写入并在 hufu 账本按题可查(单测);
3. xiaochang_graph 输出每题路径状态(单测 + 验证轮实拉);
4. 派单 prompt 自动携带死路清单(单测断言);
5. 验证轮:三机制在托管沙箱实测通过(轻量验证,force 收官);
   ②b 单独验证:执行者调用 xiaochang_fork → 主 agent 在 wait 中被消息唤醒、
   立即 enqueue 未走分叉(时间线可见);
6. 下一正式局:花费 ≤¥50、尝试破 1h26m(v13 纪录)。

## 5. 复盘回填区(战后)

## 附:调度架构裁定与演进后门(2026-09-13 定)

- 实测(v13 满分局):主 agent 事件处理余量 ~10×(事件率 <1.5/min vs 容量 10+/min),
  **单调度器 + 事件队列 + 批处理为当前正确形态**,不做分级调度。
- **演进后门(用户思路,记档待决)**:若未来事件率触及调度天花板——"主 agent 开局
  识别不相交任务,起多个调度器、每调度器管一个任务"——届时把该思路呈现给用户判断,
  不自行实施。
