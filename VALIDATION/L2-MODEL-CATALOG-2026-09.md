# L2 校准记录：三家模型目录 + 官方单价（2026-09-08 核对）

> 目的：为集思 `priceTable` 与执行器选型提供**已核对**的依据。
> 原则：只记录从官方 API/官方定价页/官方文档核到的东西；估算值必须显式标注 ESTIMATE。
> 核对日期：2026-09-08（Kimi 账号 SSO 登录成功；智谱控制台登录待用户浏览器完成——短信接口带人机验证，curl 走不通）。

## 1. 模型目录（官方 API `GET /models`，2026-09-08 实查）

### Kimi（`GET https://api.moonshot.cn/v1/models`）
| 模型 | 说明 |
|---|---|
| kimi-k3 | 旗舰，1M 上下文，始终推理（reasoning_effort low/high/max，默认 max） |
| kimi-k2.7-code | Coding 模型，256k，多模态输入，仅思考模式 |
| kimi-k2.7-code-highspeed | K2.7 Code 高速版（180 tok/s，短上下文 260 tok/s） |
| kimi-k2.6 | 通用，256k，视觉+文本，思考/非思考可选 |

### DeepSeek（`GET https://api.deepseek.com/models`）
| 模型 | 说明 |
|---|---|
| deepseek-v4-flash | 1M 上下文，输出最大 384K，思考/非思考 |
| deepseek-v4-pro | 同上（Pro 档） |
| deepseek-v4-flash-vision-exp | Flash 视觉实验版 |

### 智谱（`GET https://open.bigmodel.cn/api/paas/v4/models`，当前代 10 个）
glm-4.5 / glm-4.5-air / glm-4.6 / glm-4.7 / glm-5 / glm-5-turbo / glm-5.1 / glm-5.2 / **glm-5.3** / **glm-5.3-flash**

官方文档模型概览（docs.bigmodel.cn/cn/guide/start/model-overview）另列全量 36 个，包括：
- 主力文本：GLM-5.3(1M ctx)、GLM-5.3-Flash(1M ctx, 多模态)、GLM-5.2、GLM-5.1、GLM-5、GLM-5-Turbo、GLM-4.7、GLM-4.7-FlashX、GLM-4.6、GLM-4.5-Air、GLM-4.5-AirX、GLM-4-Long(1M)
- **免费文本模型**：GLM-4.7-Flash(200k)、GLM-4.5-Flash(128k)、GLM-4-Flash-250414(128k)
- 视觉：GLM-5V-Turbo、GLM-4.6V、GLM-4.1V-Thinking-FlashX、**免费** GLM-4.6V-Flash / GLM-4.1V-Thinking-Flash / GLM-4V-Flash
- 图像/视频/语音/向量：CogView-4、CogView-3-Flash(免费)、CogVideoX-2/Flash(免费)、GLM-OCR、GLM-TTS、GLM-Realtime、Embedding-3 等
- 平台目录接口 `GET https://open.bigmodel.cn/api/biz/model/getAiModelListForFlow?isShow=1` 另有 55 个含历史模型的列表（含 glm-z1-* 推理系列、codegeex-4），但**未含 glm-5.x**，以 `/models` + 文档页为准。

## 2. 官方单价（CNY / 1M token）

### Kimi（官方定价页 platform.kimi.com/docs/pricing/*，页内 dateModified 2026-08-16）
| 模型 | 输入·缓存命中 | 输入·未命中 | 输出 | 上下文 |
|---|---|---|---|---|
| kimi-k3 | 2.00 | 20.00 | 100.00 | 1,048,576 |
| kimi-k2.6 | 1.10 | 6.50 | 27.00 | 262,144 |
| kimi-k2.7-code | 1.30 | 6.50 | 27.00 | 262,144 |
| kimi-k2.7-code-highspeed | 2.60 | 13.00 | 54.00 | 262,144 |
| k2.7-code (Batch) | 0.78 | 3.90 | 16.20 | 262,144 |
| k2.6 (Batch) | 0.66 | 3.90 | 16.20 | 262,144 |

限速档位（充值与限速页，2026-09-01）：Tier0 ¥0(并发1/RPM3/TPM500K/TPD1.5M)、Tier1 ¥50(15/100/2M)、Tier2 ¥100(40/100/3M)、Tier3 ¥500(50/200/3M)、Tier4 ¥5K(60/200/4M)、Tier5 ¥20K(100/300/5M)。——虎符背压的"provider API limits"数据源。

### DeepSeek（官方定价页 api-docs.deepseek.com/zh-cn/quick_start/pricing/，峰谷双价）
| 模型 | 缓存命中(谷/峰) | 输入未命中(谷/峰) | 输出(谷/峰) | 并发 |
|---|---|---|---|---|
| deepseek-v4-flash | 0.05 / 0.10 | 1.5 / 3.0 | 4.5 / 9.0 | 2500 |
| deepseek-v4-pro | 0.15 / 0.30 | 4.5 / 9.0 | 13.5 / 27.0 | 500 |
| deepseek-v4-flash-vision-exp | 0.05 / 0.10 | 1.5 / 3.0 | 4.5 / 9.0 | 2500 |

高峰时段 = 北京时间周一至周五 9:00-12:00、14:00-18:00；其余（含夜间 00:30-08:30、周末）= 半价。
→ 夜间打题用 DeepSeek 成本减半，与实测"夜间便宜"一致。

### 智谱
- **GLM-5.3**：输入 ¥8 / 输出 ¥28 / 缓存命中 ¥2.3 每百万（晚点 LatePost 官方口径：GLM-5.3-Flash 0.8/2.8/0.23 为其十分之一；360 模型广场聚合页 7.6/26.6/1.9 基本一致）。
- **GLM-5.3-Flash**：¥0.8 / ¥2.8 / 命中 ¥0.23（晚点/chinaz 2026-08）。
- **GLM-4.7**：¥2 / ¥8 / 命中 ¥0.2（360 模型广场页；官方价格页未列 4.7+ 世代）。
- **glm-4.6 / glm-4.5-air**：仍 ESTIMATE（¥1/¥4 与 ¥0.5/¥1），待价格页核对。
- 平台价格页 operation 配置（登录后 `GET /api/biz/operation/query?ids=1122,...`）可列出全部老世代模型单价；其中 **GLM-Z1-Flash、GLM-4-Flash-250414、GLM-4V-Flash 等标注"免费"**。
- **账户实况（2026-09-08 登录后实查）**：余额 ¥159.92，累计充值 ¥300，总消耗 ¥140.08（`/api/biz/account/query-customer-account-report`）。
- 智谱登录会话：cookie `bigmodel_token_production`（JWT）+ `Authorization: Bearer <同JWT>` 头才可调 biz 接口；session 已存 `.secrets/`（本地不公开）。

## 3. 对集思 priceTable 的影响（已落实，见 shence-jisi 提交）
- 旧表严重低估 Kimi：k2.6 输出写 ¥3 → 实际 **¥27**（9 倍）；k3 输出 ¥32 → 实际 **¥100**。已按上表修正。
- 新增 `cacheRead`（缓存命中价）维度：命中 token 按命中价计，不再混入未命中输入价。
- DeepSeek 增加 `idle`（谷时段价）：jisi_usage 按 sidecar 的 `at` 时间戳 + 北京时区判定峰/谷计价。
- priceOrder 按真实单价重排（便宜→贵 tiebreak）。

## 4. 遗留
- 智谱控制台登录 ✅ 已完成（用户浏览器滑块验证 + 短信，session cookie 已存 `.secrets/`）；余额/价格数据已读。
- Kimi 平台控制台会话：旧短信登录接口返回 401（已下线），控制台登录弹窗 chunk 已从 CDN 撤下；已用 OAuth 设备流（auth.kimi.com，kimi-cli 同款）完成 SSO 登录，但 kimi-code 令牌读不了平台计费面板。Kimi 花费改用官方单价 × usage sidecar 精确核算。
