# 修复与改进（2026-09-30）

针对代码审查中发现的缺陷做了一批修复，全部为**最小改动面**，不改变任何对外接口与落盘契约。

**验证结果**（在 `rust:1-bookworm` 容器内，与项目 Dockerfile 同环境）：

```
cargo check -p agent2api-server
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 9m 21s
    0 error / 0 warning

cargo test -p agent2api-server --lib
    running 258 tests
    test result: ok. 253 passed; 0 failed; 5 ignored; finished in 4.08s
```

## 🔴 安全

### 1. 调试报文脱敏名单漏掉真正在用的凭据头 → 上游 token 明文落库
`core/debug_traffic.rs`

- **问题**：名单里写的是 `x-autoclaw-token`，但 AutoClaw 出站实际用的是
  **`X-Authorization`**（见 `providers/autoclaw/adapter.rs` 自己的注释：
  `Authorization: Bearer` 会被上游判 Invalid token）。而匹配是**精确相等**，
  于是「打开调试模式」正好会把账号 access token 明文写进 `debug_traffic` 表、
  在面板请求详情里展示、并随导出/截图流进 issue —— 而开调试模式恰恰是用户
  为了提 issue 才做的动作。同批漏网的还有 Trae 的 `X-Cloudide-Token`、
  CodeArts 的 `X-Security-Token` 等。
- **修复**：补全已知头；并把匹配从「精确名单」改成「精确名单 ∪ 关键词子串」
  （`token` / `auth` / `cookie` / `sign` / `secret`），新增 provider 时不再需要
  人工同步这张表。多脱一个头只损失排障信息，少脱一个是事故。

### 2. API Key 比较非恒定时间
`server/http.rs`

- **问题**：`headers_match_key` 用 `==` 比较 API Key，短路会泄漏「前 n 个字节
  匹配」的时序信息。同 crate 已有 `constant_time_eq` 却没用在这里。
- **修复**：`access::constant_time_eq` 提为 `pub(crate)`，两处比较改用它。

### 3. 登录失败锁定按「直连对端 IP」计 → 反代下退化成全局锁
`server/access.rs`、`server/api/panel.rs`

- **问题**：容器 / 反向代理部署下所有请求的对端都是**同一个**代理地址，
  按它计失败次数等于把所有访问者合并进同一个桶：随便一个人连错几次密码，
  **管理员自己也会被锁在门外** —— 正是该模块注释里说要避免的情况。
- **修复**：新增 `access::login_source_ip()`，在对端是 loopback / 私有地址时
  采信 `X-Real-IP`，其次采信 `X-Forwarded-For` 的**最后一段**（nginx 用
  `$proxy_add_x_forwarded_for` 追加，客户端伪造的段只会排在前面）。
  **不无条件信任代理头** —— 对端是公网地址时一概忽略，否则等于给攻击者一个
  「换个头就重置计数」的开关。可用 `AGENT2API_TRUST_PROXY_HEADERS=0` 关闭。

## 🟡 功能性

### 4. 非流式聚合按 TCP 分片解码 → 中文被切成 U+FFFD
`core/upstream/aggregate.rs`

- **问题**：`buffer.push_str(&String::from_utf8_lossy(&chunk))` 对**每个 TCP 分片**
  单独解码，分片边界落在 3 字节汉字中间时，那半个字永久变成 `�`。非流式
  中文长回答因此会零星乱码 —— 而中文正是本项目的主要场景。同目录
  `sse.rs::push` 与 `providers/trae/forward.rs` 用的都是「字节缓冲 + 只在完整
  行上解码」，聚合路径是唯一的反面样本。
- **修复**：改为 `Vec<u8>` 字节缓冲 + 只在完整行上 `from_utf8_lossy`。

### 5. 每个对话请求都无条件同步全量写盘
`server/api/pipeline.rs`

- **问题**：`write_debug_files` 无条件执行，用 `std::fs::write` 在 async handler
  里做阻塞 IO，把整份入站 body（`/v1/*` 的 body 上限已放开，实测可达数 MB）
  覆盖写到配置目录。既阻塞 tokio worker，也在用户不知情时持久化请求体。
- **修复**：挂到 `debug_mode` 开关下，并加 8 MiB 体积闸。

### 6. 代理池写路径全程无锁 → 并发丢失更新
`core/proxy_pool.rs`

- **问题**：`create` / `update` / `remove` / `sync_clash` / `record_test` 都是
  「`read_items()` → 内存改 → `write_items()` 整份覆盖」，彼此无互斥；而
  `GET /api/proxies/pool` 每次都会触发一次 `sync_clash()`，于是「打开页面」
  和「新增 / 测试出口」天然并发 → 静默丢数据（刚加的代理消失、刚测出的
  `lastTest` 被覆盖）。
- **修复**：加一把模块级 `Mutex` 串起全部写路径（只锁写；`read_items` 仍无锁，
  避免 `create` 末尾调用 `list()` 时自锁死）。已确认 `sync_clash` / `record_test`
  内部不调用其它加锁函数，无重入死锁。

### 7. 远程 / Docker 部署下面板网页登录 100% 失败（issue #46）
`core/login.rs`、`server/web_shim.rs`

- **问题**：AutoClaw 的 `navigate_uri` **必须**长成
  `http://localhost:<登记端口>/auth/callback-<vendor>`（上游按白名单逐字校验，
  改不了）。网关跑在远程时，那个 `localhost` 指的是**用户自己那台机器**，
  浏览器跳过去只会「无法访问」，回调永远到不了网关。
- **修复**：补上同类网关（gpt-load / one-api）的标准兜底 —— **让用户把浏览器
  地址栏里那条打不开的回调地址粘回来**。
  - 后端：`submit_login_callback` 新增 AutoClaw 分支，解析粘贴 URL 里的
    vendor / code / state 后复用既有的 `finish_autoclaw_oauth_callback`；
  - 前端：网页面板登录时（`web_shim.rs`）为 autoclaw / autoclaw-intl 弹出
    「粘贴回调地址」浮层，提交后走同一个入口。
  - 不依赖端口映射、隧道或公网地址（授权码是一次性的，且绑定在该次登录的
    `navigate_uri` 上，粘回来即可完成换码）。

### 8. altcha 台账满时整表清空
`server/altcha.rs`

- **问题**：`table.retain(|_, _| false)` 一次性作废**所有**已签发但未使用的题 ——
  正在登录的人会突然收到「校验已过期」。容量只是防御性兜底，不该造成可见故障。
- **修复**：改为按到期时间淘汰最早的一半，让在途的题尽量活到自然过期。

## 🟢 其它

### 9. 登出时 refresh cookie 删不掉
`server/api/panel.rs`

- **问题**：refresh cookie 是 `Path=/api/panel` 下发的，登出却用 `Path=/` 清 ——
  浏览器按「同名 + 同 Path」匹配，根本不会删。服务端链已撤销，所以只是残留
  脏 cookie，但会造成困惑。
- **修复**：两个 cookie 各用各自下发的 Path 清除。

### 10. 远程部署下关闭公开的 OAuth 回调路由（安全加固）
`server/api/session.rs`

- **问题**：`/auth/callback-{vendor}` 挂在 **public 组**（调用方是用户的浏览器，
  它没有 API Key），而回调与登录任务的关联是「同变体里**最近发起**的那一条」
  —— 回调 URL 里不带我们自己的 state（上游会在 `navigate_uri` 后拼它自己生成的
  state，放我们的会撞参数名，见 `providers/autoclaw/oauth` 的说明）。
  桌面形态下这个取舍成立（能打到 loopback 的只有本机进程），但**容器 / 远程
  部署下这条路由公网可达**：管理员点「登录」后、浏览器跳回来之前的那几秒里，
  任何能访问面板的人抢先请求 `/auth/callback-zai?code=<自己的>&state=<任意>`，
  网关就会用攻击者的授权码换码成功，把**攻击者的账号**写成网关登录态 ——
  此后管理员的所有请求都跑在攻击者账号上。
- **修复**：远程形态下这条路由**本来就没有正当用途**（`navigate_uri` 被上游
  白名单钉死在 `http://localhost:<端口>`，浏览器把它解析成用户自己那台机器，
  永远打不到网关），因此按监听地址直接关掉：**绑 loopback（桌面壳 / 本机
  headless）才处理**，否则只回一页指引、**不碰 code**。
  远程部署的正路是「粘贴回调」（`POST /api/session/login/callback`，那条要鉴权）。
- **残留风险**：本机（loopback）形态下，同机进程仍可抢先伪造回调 —— 这是
  源码注释里明确接受过的取舍（本机进程本就等同于用户本人）。要彻底消除需要
  PKCE 或与上游 state 的绑定，两者都依赖上游行为，建议单独评估。

## 未做（需要环境或属于独立工作）

- **AutoClaw 406 / 403**：已定位为上游侧拒绝（详见 issue #58），网关侧无可改项。
- **千问办公（QwenWork）provider**：调研确认它与 **Qoder 是同一套协议**
  （同一推理端点形态、同一 COSY 签名且 RSA 公钥逐字节相同、同样的
  `{statusCodeValue, body}` 双层 SSE 信封、同一续期端点），正确做法是在
  `qoder::endpoints::Region` 里加一个变体而不是另写一家 provider。
  本次已把**接入所需的最后一层抽象补齐**（`cosy::CosyProfile` 参数化，
  `Region::cosy_profile()`，chat / 目录刷新两处调用点改为按地区取档），
  但**没有加变体**：主机名、设备授权路径、模型清单都只有第三方逆向记录、
  未经真实账号验证，而加变体要同步扩 `CatalogState` 与十余处 match ——
  在拿不到账号联调的前提下写进去等于交付一份猜出来的实现。
  待核实项已就地写在 `Region` 枚举的注释里。
- **其余新提供商**：调研结论与实施方案见 [`新增提供商接入指南.md`](./新增提供商接入指南.md)。
  其中库库AI 经调研判定**无法按本项目「桌面客户端登录态转发」范式接入**。
