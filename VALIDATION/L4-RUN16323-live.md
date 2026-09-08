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
