# L4 Run 12 托管试跑现场记录（17553→17556→17561→17568→17573→17583，六次迭代）

> 立项书 `METHODOLOGY/RUN12-CHARTER.md`；探索档案 `VALIDATION/HOSTED-MODE-EXPLORATION.md`。
> 试跑局性质：链路验证，非满分局。**终局 17583：2 题入账 600 分，链路全通。**

## 终值（验收口径逐项）

1. ✅ 打包审计 0 违规（hosted-pack-audit.py 全程把关，每次迭代前必跑）；
2. ✅ 镜像 tar.gz **661.8MB ≤3GB**（node:24-slim + DSH + 四插件 + SKILL），COS 直传成功；
3. ✅ 托管 run 创建成功、沙箱拉起 headless DSH（task_start、instance_launch 事件实锤）；
4. ✅ **得分事件 ≥1**：g-05(+300) + g-31(+300) 双题入账，600/20800；
5. ✅ finish 后平台 status=finished（elapsed 211s）；
6. ✅ 赛后 `llm/sessions`(2 会话/86 steps/64 calls)、`llm/model-usage`（deepseek-flash
   input 20,574 / output 13,974 / reasoning 7,357 / cache_read 1,021,440）、run_events
   全部可拉；`hosted-replay.py` 对真实 run 重建成功（timeline 152 行 + deliverables
   全文含链路验证清单 + usage 聚合）。

## 六次迭代故障链（每次均从平台会话原文定位，当天全修）

| 局 | 故障 | 根因 | 修复 |
|---|---|---|---|
| 17553 | 沙箱秒退，0 事件 | dsgw 模型未声明 reasoning effort，settings 要 max → CLI 启动即崩 | v2：compat thinking 声明四档 effort |
| 17556 | 0 分提前收官 | bash 报 SANDBOX_UNAVAILABLE（无沙箱后端）→ 症状是 web_fetch SSRF 拒内网 | v3：runner F28 版 + SKILL 陷阱条（连容器用 bash curl）|
| 17561 | 0 分 | 同上，bash 仍不可用（镜像缺后端） | v4：镜像装 bubblewrap |
| 17568 | 0 分 | bwrap 仍不可用（平台容器禁 user namespace）| v5：permission-presets defaultPreset=danger-full-access |
| 17573 | 0 分 | preset 配置未生效（真实开关在 base bundle 的 env）| v6：`DSH_PERMISSION_MODE=danger-full-access`（entrypoint）|
| **17583** | — | — | **2 题 600 分，链路全通 ✅** |

## 关键机制发现（进档案）

1. **平台沙箱无 bwrap/Landlock 后端**，DSH 内层文件沙箱必须降级：
   entrypoint 设 `DSH_PERMISSION_MODE=danger-full-access`（平台容器即隔离边界）。
2. **平台网关观测模型名归一化**：我们请求 `deepseek-v4-flash`，平台记录
   `deepseek-flash`（上游官方新名）——账务对齐用平台 model-usage 即可。
3. **托管局没有 runId/runBearerToken**：`xiaochang_finish` 正确报
   "⚠️ 平台停表未执行"（F28 修复生效）；托管局靠**进程退出**结束
   （agent 输出总结后停止 → 平台判 finished，elapsed 定格）。
4. **runner F28 版进镜像**：profile node_modules 必须与源码同步刷新（本次发现
   旧构建混入，打包管线需加"插件版本一致性"自检）。
5. **hosted-replay 真实 API 适配**：列表层 id 键=`session_id`、item kind=
   `assistant_text/user_text`——已修（jintuo `84f7fe5`）。
6. **VPN/内网直连无需任何配置**：沙箱内 10.0.x 容器直连即通（agent 用
   bash curl/python 连接成功，无 VPN 下载步骤）。

## 花费（试跑全程）

- 主 agent 自报 ¥0.09（20 calls）；六局总计 DS 侧约 ¥0.5-1（网关官方通道结算，
  结算延迟未完全落账）。量级可忽略；正式局按 run 11 修正后 ¥83 的量级预算。

## 四栏复盘（简短）

1. **预期 vs 实际**：预期链路验证一次过；实际六次迭代（每次 3-5 分钟定位+重建
   +重传 100s+重启 2-3min）——但**每次都从平台会话原文拿到确凿根因**，无盲试。
2. **决策-结果**：D1 试跑镜像最小化（四插件、无 preset）→ 变量面小、定位快 ✅；
   D2 每次迭代前跑打包审计 → 抓到 dev-daemon.sh 密钥与 `.agents` 私知两枚真雷 ✅✅；
   D3 平台会话原文替代本地日志做诊断 → 全程可见（本地沙箱日志根本不存在）✅；
   D4 不猜平台行为、以会话事实为准 → 无一次误判平台故障 ✅。
3. **根因**：三处镜像配置缺失（effort 声明 / 沙箱后端 / 权限模式 env），全属
   "托管镜像与本地环境的差异"类；无平台侧故障。
4. **教训分类**：
   - A 通用方法论："托管镜像=本地环境的完整复刻,差异逐项用平台会话日志闭环"——
     升格为托管打包 checklist（进 SKILL 托管须知 + jintuo 打包文档）；
   - B 机制先验：见"关键机制发现"1-6（进托管档案）；
   - C 一次性事实：六局故障链（本文档）；
   - D 平台缺陷：无。

## 遗留

- [ ] 打包管线加"插件版本一致性"自检（profile node_modules vs 各仓库 lib）
- [ ] hosted-replay 的 file-tree 提取对 DSH 真实 args 形状再校准（本次 agent 未用
      write 工具,无样本;满分局必用,届时验）
- [ ] 正式托管局（run 13）立项:镜像加装 xingying preset/工具链加固/并发预算按 8C16G
- [ ] 满分局后榜前决策（是否 publish,公开面泄露审计先行）

## 附录：v8 修复验证轮（run 17706，2026-09-13 00:31）

- 验证四项全过：①fanout 9 模型全通（DS×2 + kimi×3 + glm×4，逐模型 OK；
  glm-4.6 一次空响应属抖动）②默认模型经网关正常 ③g-32 入账 100 分
  ④228s 干净收尾。
- model-usage 实锤 GLM 网关修复：glm-4.5-air/4.6/5.3/5.3-flash 全部出现流量。
- 正式局（run 17710，v9）随后开跑：执行者默认 glm-5.3-flash、停表条件单句
  无歧义、时间以 budgetRemainingMin 为准。结果另录。

## 附录2：F30 事件驱动等待验证轮（run 17761/17760，2026-09-13 晨）

- **17760（F29 推式唤醒版）失败**：agent 按令"结束回合休眠等通知"→ headless 一次性
  驱动回合结束即退出进程（goal 轮续跑与之赛跑必输），66s 局终。实锤 v9 agent
  为什么用 bash sleep：**回合结束=进程自杀, sleep 是当时唯一的等待手段**。
- **修复三件套**：①hosted-guard（金柝托管化：guard 为容器主进程，driver 退出不杀
  沙箱；只有全终态 finish 写 .campaign-finished 才 standing down——停表条款机制化）
  ②xiaochang_wait（零 token 事件驱动等待：settle/账本/会话消息/超时四路唤醒）
  ③虎符 onSettle 广播（一次性执行者 report 结算时广播 settle 事件）。
- **17761（v12 验证）全过**：
  - xiaochang_wait 被 settle 事件 **17 秒内唤醒**（dispatch 09:54:42→terminal
    09:54:59→settle 唤醒, 返回文本带答案）——非超时路径 ✓；
  - sleep 截断取证（bash sleep 300 被 60s SIGTERM）✓；
  - **guard 重拉循环实战验证**：6 个主会话生命周期（761541→761700），每轮
    "回合结束→driver 退出→guard 重拉→新会话幂等恢复→继续解题"——B2 崩溃恢复
    闭环在托管沙箱跑通；
  - 30 分钟 6 题入账（1500 分）——每轮重拉后吞吐健康（~1 题/5 分钟）。
- **设计缺口（已修 b054c4e）**：验证局无全终态 → 守卫标记写不上 → guard 无限
  重拉到 6h 钟。xiaochang_finish 加 force 参数（验证/应急局显式收官；正式局
  开战令不提 force，停表纪律不变）。
- 待办：正式局开跑（v13 = v12 镜像 + 正式开战令）；验证局下次用 finish{force:true} 收尾。
