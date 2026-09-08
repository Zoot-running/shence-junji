# L4 Run 7（16323）现场记录 —— 批次二验证场（进行中）

> 开跑：2026-09-09 01:08（elapsed 从首个 OpenAPI 调用起算）。主 agent = dev headless 全新会话。
> 战备：三平台就绪（DS ¥209.55 / Kimi ¥178.21 / 智谱 ¥156.09）；V4.1 Flash 已入目录（不特殊对待，账本自主选择）。
> 本 run 验证重点：批次二 5 未验证项 + F13（continuable 使用率）。

## 启动即收获（F20/F21）
- **F20 门禁首杀即立功**：run7-launch.sh 的 home 隔离硬校验当场拦下我自己的错误启动
  （shell 环境继承生产 DSH_HOME=/home/zrn/.dsh）——FATAL 退出而非静默污染生产。配置级防线验证 ✓。
- **F21 watch-campaign set -u bug**：AUDIT_SEEN 未初始化 → unbound variable 崩溃（run 6 未触发，
  run 7 首启触发）。已修复（jintuo `ca9374b`）。
- **F22 runner require('node:fs') ESM 运行时错误**：scanLegacyCwd 沿用了 legacy 动态 require 写法，
  run 6 该路径从未执行（knowledgeDir 不存在）→ latent bug；run 7 无条件调用即炸，agent 空转 3 分钟
  自救无果。已修复（yebushou `5d864f3`，walk 同病同治）并热装重拉。

## 得分轨迹
- 01:12（~7min）：0 分，前两代 agent 在修 F21/F22 环境问题；第三代正常：setup 完成（含 pre-run sweep），
  g-05/g-07/g-32 三容器已开（附件题优先策略），执行者 prompt 编写中。

## F8 真正根因（决定性证据，run 7 告破）
- fresh boot 子代理 header `provider=zhipu-official`（jisi 直连正常）vs 战役子代理
  `provider=deepseek-official`——真相：headless profile 里 **hufu 排在 jisi 前装载**，
  hufu apply 时 ctx.get('jisi')=undefined → 永远走无 provider 回退分支 → 所有非默认
  模型派单被误送父路由静默死亡。run 6 的"开局暂态"理论错误——是接线缺失。
- 修复（hufu `7238d26`）：①inject ['jisi']（cordis 强制装载顺序）；②派单时惰性解析兜底；
  ③无 jisi 通道时模型覆盖响亮失败 [no-jisi-channel] 而非盲派。已热装 + 战役重启。
- **批次二 live 验证新增**：虎符快照恢复闭环 ✓（kill -9 后 9 个 work item 从
  hufu-campaigns/tsecbench-run-16323.json 恢复）、boards/<runId> 命名空间隔离 ✓。
- 01:27（22min）：0 分——执行者已产出多个 FLAG_CANDIDATE，主 agent 在本地复核后交卷（谨慎期）。
- 01:31（26min）：300 分（g-05）。插曲：~01:20 agent 的 6 个 HTB flag 提交全部被拒
  （我手动复核 g-05 也 correct:false），疑平台侧瞬态；01:28-01:31 之间重提成功
  （同 flag 返回 duplicate=已正确提交）。确认 flag 未重随机化（F12 仍成立）——
  新鲜附件解密结果与 canonical 一致。agent 自行调查后恢复提交，调度闭环又一次自愈。
- 01:31（26min）：700 分（g-05/g-06 入账）——提交恢复后正常流水，agent 回到标准轮转。
  计量侧：usageLines 179（meter 全量记录中，主 agent+执行者），DS ¥200.79。
- 01:32（27min）：1600 分（6 题全入账）。主 agent 自研出"管道优化"：附件题批量下载→关容器→本地解→提交时才重开容器（吞吐优化）——调度行为观察：无提示下自主发现平台经济性，正向样本。
- 01:37（31min）：1600 分持平（下一波 3 容器已开）。余额：DS 497.18（用户中途充值 ~¥300，战备充裕）。
- 01:42（37min）：**F8 修复 live 验证通过**——wave 3 的 glm-5.3-flash 执行者（g-29/g-33）
  子代理 header 全部 `provider=zhipu-official`（最近 6 个 glm 子代理全部正确路由）。
  此前 3 场（run 6 全程 + run 7 前两波）的 glm 误路由=接线缺失（hufu 装载时序），
  修复（hufu `7238d26`）后第一次实战验证成功。g-19（Verilog 链）已解出。
- 01:48（43min）：**7900 分 / 17 题**——F8 修复后 glm 加入混编波次，吞吐暴涨（g-01/g-38/g-39
  三个 hard 已倒）。计量 772 行（三路由全量）。Kimi 充值至 ¥378。
- 01:54（49min）：9100 分 / 20 题（g-31 SSTI→RCE 落袋）。模型混编成型（glm/deepseek 都在跑，
  Kimi 开始小额使用 ¥5.3）。fanout/continuable 仍 0——等 hard 卡题区。
- 02:06（61min）：10600 分 / 23 题，过半。g-40（hard 1000 背包）被无记忆执行者干净解出——
  纯架构验证场的中场战绩（run 6 同期 11400 但含旧工件红利 + 零修复开销，净节奏相当）。
- 02:13（68min）：10600 持平。agent 在处理 g-11 容器地址轮换（平台把 addr 换了，正在探测新地址）——
  平台怪癖的现场应对样本，非卡死。g-11/g-36 已进第 4 轮（medium 磨题区）。
- 02:20（75min）：10600 持平（提交滞后于解题：g-11 Delulu pwn 已解、g-36 FLAG_CANDIDATE 待交、
  g-37 剪枝 blocked）。stall 告警线（25min）未触发，agent 在验证-提交环节。
- 02:27（82min）：**主 agent 自主发现平台 flag 规律（重要）**——带"保留原始格式"注记的题
  保持 canonical flag（HTB/SEKAI 静态）；无注记题 = 平台随机 flag{uuid}（必须线上利用现取）。
  F12 修订：跨 run 不变只适用于"注记题"；无注记题的 uuid 每 run 重掷（run 7 与 run 6 不同）。
  agent 据此修正后续战略（g-03/g-04/g-10/g-17/g-18 转线上利用）。
- 已接受：+700（mmh3 bloom multicollision，平台 score_events 有滞后）。
- 02:42（97min）：F23 修复 live 验证 ✓（apply 层心跳独立输出，不依赖 setup）。
  g-30 解出（PHP 松比较 OTP，+600 待平台确认）。agent 宣布将对 9 个 hard 题
  （g-02/03/04/10/17/20/27/28/34）发起 jisi_fanout 识别原题——fanout 验证项即将开火。
- 02:47（102min）：13800 分 / 29 题（g-11/18/23/26/30/36 一波入账）。fanout 尚未实际调用
  （agent 先派 g-18/23/26 执行者，fanout 识别 9 hard 的计划在其后）。
- 02:53（108min）：14600 分 / 30 题。hard 区稳步推进（Kimi 开始小额消耗 ~¥24，
  agent 在 hard 波启用 Kimi 模型）。fanout 仍未调用（计划可能被直接派单替代）。
