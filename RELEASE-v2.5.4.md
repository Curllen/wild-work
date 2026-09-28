# wild-work v2.5.4

> 发布主题：**新增第 9 个渠道「智谱清言（glm）」** + 一批由社区贡献与日志实证驱动的修复。
> 本版共 7 个提交、51 个文件、+8132 行；含 1 个新渠道、3 个社区 PR、4 个 issue 修复。

---

## ✨ 新渠道：智谱清言（`glm/*`）

对接智谱清言（chatglm.cn）**网页版私有接口**（非 open.bigmodel.cn 开放平台），
按「额度」计费而非 token，登录即送 200 积分/天。

- **4 个实测可用智能体**：ChatGLM / AI搜索 / 清言PPT / 视频助手；
  三个 chatglm 变体（普通/推理/沉思）共用底层模型 `moe_53f`，模型名带上游代号后缀
  （如 `glm/chatglm:moe_53f`），旧名保留为别名不断供
- **登录**：面板一键拉起**独立 profile** 的 Edge/Chrome（CDP 自动捕获 Cookie，
  零第三方依赖、零 WebView2），多账号逐个添加互不干扰；
  自动路径失败时自动降级「手工粘贴 `chatglm_refresh_token`」兜底
- **签到即保活**：每日积分由服务端按天被动发放（`daily_login_score` 已接入，幂等安全），
  自动化边界 = 保活对话 + 积分读取，不碰任何写操作（伙伴/群聊任务明确不做）
- **跨平台**：Windows/macOS/Linux 均可（只需系统装有 Edge 或 Chrome）；
  贡献者 ttales430 已将协议逆向成果独立开源为 [glm2api](https://github.com/ttales430/glm2api)

> ⚠️ 新账号添加后积分为 0 是正常的：需在**智谱清言手机 App** 登录一次才发放额度
> （+3000，随后 +500）——这是上游行为，对照实验证实（详见 AGENTS.md R23/R32）。

**附带修复两个真实 bug**（glm 接入过程中发现，已上升为通用决议）：

- **R34 调度器漏启动**：新增渠道时漏写 `go xxxSch.Run(sctx)` 会导致该渠道自动签到/保活
  从不运行——不报错、不崩溃、手工触发仍可用，极难发现。新增静态扫描回归测试钉死此形态
- **R35 流式超时掐断长流**：`http.Client.Timeout` 覆盖整个响应体读取，长回答会在超时点被
  掐断且无终止帧（实测触发 5 次）。glm 采用独立 `StreamHTTP`（不设 Timeout，照 traework 模式）

## 🔧 修复

### TraeWork：专属积分已升级为通用积分，撤回「不可用」标注（#44）

2026-09-23 引入的 `product_id==209` 判据被本仓日志实锤证伪：99 次对话期间
可扣池余额纹丝不动（恒 3086），而「不可用」小计 2200→1846（-354，恰为对话消耗量）——
**扣费实际发生在被判为「不可用」的池上**。撤回该判据后：

- 面板不再把 200 档签到积分误标「不可用」，可用余额与官网对得上
- 账号池路由恢复使用这批真实可扣的积分（每账号约 +2000）

### 费率面板：模型列表不再整组消失（#40）

QoderCN/QoderCOM（按设计无静态兜底表）在缓存过期后整组从「模型列表和费率」消失，
根因是缓存过期即回退空静态表 + 模型缓存没有任何后台填充源。修复后：

- 过期缓存沿用（陈旧比空好），后台刷新周期性填充模型缓存
- 冷启动后无需等某个客户端碰巧请求 `/v1/models`

### WorkBuddyAI：9 个模型倍率从 unknown 变为真实值（#39）

并入上游 `/v3/config` 目录（实测国际版可用）：

| 模型 | 倍率 | 说明 |
|------|------|------|
| deepseek-v4.1-flash | x0.00 | 免费 |
| gpt-6-astra | x6.67 | |
| kimi-k2.8-preview | x0.77 | |
| **deepseek-v4.1-flash-sg** | x0.03 | 新增模型 |
| **glm-5.3-flash** | x0.06 | 新增模型（官方客户端同价） |

模型清单 18 → 29；同 id 倍率冲突时以 v3 为准（hy4-preview 修正为 x0.29）。
其余 6 个模型（deepseek-v3/glm-5.1/glm-5v-turbo/hy4-preview-f/kimi-k2.7/minimax-m3）
上游 v2/v3 均未下发费率，继续显示 unknown——不编数据。

### qoder 系：偶发 101 Signature invalid（PR #45）

签名串与 `cosy-date` 头各自取一次时间，长会话（数 MB 上下文）下跨秒概率随 body
增大而升高 → 上游 101。四处独立副本同改：`AuthHeader` 返回签名所用时间戳，
`ApplyHeaders` 复用它，全链路只剩一次取时；回归测试以「每次取值前进 1 秒」的
注入时钟锁死契约。

### TraeWork/TraeCode：模型目录被 1MB 上限截断（#41，PR #43）

`solo_agent` 的目录响应 1.28MB 被 `1<<20` 截断 → JSON 解析失败 → 静默回退静态表
（面板只有 16 个模型）。上限提至 8MB 并改为**超限显式报错**（不再把「自己截断」伪装成
「上游坏 JSON」）；实测 `/v1/models` 从 16 恢复到 49。

## 🚄 性能

### 管理面板打开卡 3~10 秒（#47，PR #49）

echarts 的旧 CDN 源响应 `no-store`，**每次打开面板都重下 751KB**。换用
registry.npmmirror.com（同一份文件，sha256 一致）后：

| | 首次加载 | 二次打开 |
|---|---|---|
| 资源耗时 | ~100 ms | **0 ms（缓存命中）** |
| DOMContentLoaded | 132 ms | 52 ms |

## 📦 升级说明

- 直接替换 `wild-work.exe`，配置/账号/流水**零迁移**
- 智谱清言：面板「＋ 智谱清言」→ 在弹出的独立浏览器窗口登录 → 自动完成；
  若自动路径不可用，界面会给出手工粘贴指引
- 升级后建议在面板点一次「刷新积分」：TraeWork 账号的「不可用」积分将并入可用余额

## 🙏 致谢

本版包含 4 个社区贡献（均已合入 master 并保留作者署名）：

| 贡献者 | 内容 |
|--------|------|
| [@xxhhlk](https://github.com/xxhhlk) | #43 模型目录上限修复、#45 签名时间戳同源 |
| [@ttales430](https://github.com/ttales430) | #46 智谱清言渠道（含协议开源 [glm2api](https://github.com/ttales430/glm2api)） |
| [@jianzhangg](https://github.com/jianzhangg) | #49 echarts CDN 换源 |

以及 issue 报告者 [@HappyCode-HC](https://github.com/HappyCode-HC)（#39/#40/#44）、
[@ciot-plus](https://github.com/ciot-plus)（#41）、[@jianzhangg](https://github.com/jianzhangg)（#47）、
[@jianzhangg](https://github.com/jianzhangg)（#42）。

## 📋 全部变更

```
b384648 fix(workbuddyai): 并入 /v3/config 目录，补齐 extraModels 的真实倍率 (issue #39)
ea7b8ae perf(panel): echarts CDN 换用 registry.npmmirror.com (issue #47)  [PR #49]
b2daf6e fix(panel): 费率面板模型列表不再因缓存过期整组消失 (issue #40)
080ee99 fix(traework): 200 档签到（pid=209）已升级为通用积分 (issue #44)
65ea72d feat(glm): 新增智谱清言渠道                                   [PR #46]
1fbc3f7 fix(cosy): 签名时间戳与 cosy-date 头同源                      [PR #45]
e58cc78 fix(traework): 模型目录读取上限 1MB → 8MB (issue #41)          [PR #43]
```
