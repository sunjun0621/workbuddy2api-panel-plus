# PATCHES — 相对上游的改动

基线：`zhizhishu/workbuddy2api-panel` @ `1e23c2b`（v1.11.9-panel，2026-09-28）
本分支版本号：`1.11.9-plus.1`

---

## 1. 补齐 4 个「体验指定专家」成长任务（主要改动）

### 背景

上游 2026-09-28 停更后，官方在成长中心新上线了 4 个指定专家体验任务，每个 +50 credits +5 能量。
上游的 `internal/panel/autotask.go` 只有批量的 `expert_5`（任意 5 位专家）和 `Expert_lighthouse`（轻量云专家），
对这 4 个新任务没有对应的 `autoAction`，点「一键完成」时会显示「无可用动作」而跳过。

### 做法

新增 `runExpertUseByID(expertID, nameZH, professionZH, version)` 工厂函数，把 `expert_5` 已验证的公式
参数化成「单专家」版本，然后为 4 个任务各注册一条 `autoAction`：

| task_code | 任务名 | 专家 id |
|---|---|---|
| `expert_use_maidehao` | 中国电商运营专家 | `ex_GDl31WahyIH0` |
| `expert_use_douyinyin` | 抖音策略师 | `ex_FbY5nlWTqDod` |
| `expert_use_shangjia_xuanchuan` | 商家宣传页与海报设计师 | `ex_Weo3x2EZuSDm` |
| `expert_use_bowangwang` | 直播电商教练 | `ex_WZY136wV5aMt` |

链路完全复用上游已有的事件构造函数，没有新增任何私有协议：

1. **召唤链** — `DesktopExpertSummonSequence(e)`：`web_element_click` → `summon_click` → `summoned`
2. **真实对话** — `DesktopChatWithExpert(a, expertID)`：带 `X-Expert-Id` 发一轮真实 chat，拿服务端返回的 `requestId`
3. **使用事件** — `DesktopChatSequence(...)` + `DesktopExpertActualUseEvent(e, conv, req)`：`expert_actual_use`

关键点：`requestId` 必须是服务端真实返回的，自造的不计数；专家信息优先从市场列表拉真实条目，
拉不到时才用任务下发的兜底值（id / 名称 / 版本号），因此不会出现「自造专家 id 不计数」的问题。

### 实测

单任务调用（`POST /panel/api/accounts/{user_id}/tasks/auto`，body `{"task_code":"..."}`）逐个验证，
4 项全部一次点亮，进度 1/1，自动领取奖励，账号总进度 21/26 → 25/26。

---

## 2. 模块路径改为本仓库

`github.com/linguo2625469/workbuddy2api-panel` → `github.com/sunjun0621/workbuddy2api-panel-plus`
（`go.mod` + 所有 `*.go` 的 import；README / LICENSE 内的上游链接保留原样作为署名）。

## 3. 版本号

`cmd/server/main.go` 的 `appVersion`：`1.11.9-panel` → `1.11.9-plus.1`，透出到 `/panel/api/overview`。

## 4. LICENSE 增加本分支署名

保留 Sliverkiss（原始项目）与 linguo2625469（上游 fork）两行版权，追加本分支一行。MIT 条款不变。

---

## 未实现的任务

`skill_share`（把技能分享给好友，100 credits）没有实现。

- 客户端侧事件族已在 `app.asar` 中定位到：`skill_share_entry_click` → `_security_result` → `_channel_action` → `_deeplink_result` → `_conversion`
- 但载荷字段未逆向完成，且该动作本质是**对外发送一条真实消息给好友**，不适合自动化批量触发
- 建议手动完成：技能列表 → 任意技能 → 详情页「分享」→ 发给文件传输助手

其余 `Expert_Philanthropy`（公益专家）上游即未支持，需服务端校验真实捐款回执，无法自动化。

---

## 构建

```bash
# 需要 Go 1.22.5+
go build -trimpath -ldflags="-s -w" -o wb2api ./cmd/server
./wb2api -config config.json     # 默认监听 127.0.0.1:7863，面板在 /panel/
```

无 CGO 依赖，产物是单个静态二进制；交叉编译 `GOOS=linux GOARCH=amd64` 可直接跑在容器里。

---

## 兼容性提醒

- 上游所有上报接口都是从桌面端 `app.asar` 逆向而来，**WorkBuddy 客户端升级后可能失效**。
  本项目按 v5.9.3 客户端验证通过；升级后若任务不再点亮，先比对请求头 / 事件字段是否变化。
- 本仓库只是补了 4 个任务的参数，**不改变上游的风险属性**：它通过伪造多端客户端指纹调用 `/v2/report` 计分，
  非官方路径。账号被风控的代价由使用者自行承担，详见 README「安全与合规」。
