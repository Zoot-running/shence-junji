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
