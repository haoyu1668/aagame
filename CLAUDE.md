# CLAUDE.md — AAGame 项目接手必读

> **任何 Claude(网页版 / Code Tab / 桌面版)启动后,第一件事先读完本文件 8 节,再动手。**
> 本文件是「开机包」:把 36 万字主文档浓缩成必读核心。细节查项目库主文档 `AAGame项目文档_v2_28_81.md`。

---

## 0. 最高指令(读完这节才能动手)

- 这是一个**德州扑克俱乐部 Mini App**,在 **Telegram 内**运行(竖屏)。
- 当前阶段:**MVP 1 内测开发期**,进度 **30/35(86%)**(本 sprint 推进 5 项 + 发现 N6+N5 早完成,详见 release note v2.28.83)。
- 主背景色 `#16181f`,选中绿 `#4dff4d`。竖屏布局,参考 AAPoker / TGPoker / KKPoker。

### ⛔ 三条红线(违反 = 项目受损,务必遵守)

1. **大整顿延期** — 死代码清理要等 B16 完成(进度 34/35)才做。**现在只修 bug,不删任何"看起来没用"的代码**(很多是牌桌/游戏引擎要复用的素材)。唯一例外:修 bug 时顺手删 1-2 行零风险垃圾(如 `console.log` 调试残留),并记录。
2. **双输出** — 改 `aagame.html`(A 类前端)必须同步导出对应 `.md` 文档(B 主文档 / C 恢复包)。
3. **Step 0 先行** — 接到任务先列子项 → 等用户确认 → 才动手。不跳步直接改代码。

### 其它常驻铁律

- **货币术语锁死**:记分牌(严禁 USDT / 筹码 / 积分 / 俱乐部币)。
- **看到与本文件冲突的现象,先搜项目库验证,别用"通用经验"下判断**(历史上多次因此误判)。
- **用户不写代码**:用"复制粘贴 + 截图"节奏,或用 Code Tab 直接改文件 + 给 diff。
- **部署动作(push GitHub / Vercel / Railway)必须用户确认**。

---

## 1. 测试账号 & 平台账号

本项目有 **4 个 Telegram 测试号** + **1 个 GitHub 账号**。
⚠️ **不许把 GitHub 账号当成 Telegram 号**(历史踩坑)。

| 角色 | 显示名 | username | telegram_id | users.id |
|---|---|---|---|---|
| 主用 / owner | 伯恩 | Boen33 | 6003437589 | 2 |
| 副号 1 | HB袁华 | (无) | 8035745580 | 1 |
| 副号 2 | 啊勇 | xj0044 | 8015799096 | 41 |
| 客服(可测) | HB飘雪 | aapkkf | 8204789093 | 19 |

- **GitHub / Vercel 账号 = `haoyu1668`**(这不是 Telegram 号!)。
- **ID=11111 俱乐部("测试1")由「伯恩号」创建**,伯恩在其中 role=`admin`(创建者,后端无单独 owner 角色)。
- HB袁华已加入 11111(role=member)。HB飘雪 / 啊勇 尚未加入(见第 4 节 bug 2)。

---

## 2. 名字清单(防困惑)

| 名字 | 值 | 说明 |
|---|---|---|
| 项目代号 / GitHub / Vercel | AAGame | 内部 + 仓库名 |
| Bot 用户名 | @PKAA_bot | Telegram @ ID |
| Mini App Short name | **AAPK** | **启动后窗口顶部显示这个,正常,不是 bug** |
| Mini App Title | AAPUKE | BotFather 设置 |
| 货币术语 | 记分牌 | App 内货币 |
| 客服 | @aapkkf | App 内客服 |

> ⚠️ 新 Claude 若看到 Mini App 顶部显示 "AAPK" 而困惑 —— 那是正常的 Short name,不要当 bug 去"修"。

---

## 3. 产品逻辑(MVP 1 当前版)

### 核心流程

1. 新号点 Bot 启动 Mini App → **后端自动加入唯一俱乐部**(MVP 1 只有 1 个俱乐部,无需审批)→ 默认进入 **③【俱乐部内部牌桌大厅】**。
2. 玩家在大厅顶部点 **"申请发放"** 或 **"申请回收"** → 弹窗输金额 → 提交 → 写入 `transfer_requests` 表(状态 pending)。
3. admin 端 **N9 通知胶囊** 数字 +1(30 秒轮询触发)。
4. admin 点 N9 → 跳 **"记分牌请求"页**(`page-coin-requests`)→ 点"通过" → 钱包变化。
5. **一个玩家同时只能有 1 个 pending 申请**(grant/revoke 共享配额)。
6. admin 没及时处理 → 玩家再点申请 → toast **"上次申请还在处理中,请联系客服"**(铁律 0.10 锁死文案)→ 玩家私聊 @aapkkf。

### 三池子模型

- `treasury_balance`(俱乐部资产)/ `vault_balance`(金库)/ `jackpot_balance`(Jackpot)。
- 钱来自 owner 充值:owner 联系 @aapkkf 转 USDT → 平台后台 SQL 给 treasury +N。
- 申请发放 = treasury 减、玩家 wallet 加;申请回收 = 反向。
- MVP 1 阶段 vault / jackpot 占位,玩牌抽水(批次 5-6 才实现)。

### ⚠️ 不要踩的逻辑坑

- **N2.1"输俱乐部 ID + 申请加入"是 MVP 2 才启用,MVP 1 不用**(代码保留,别删别改逻辑)。
- 金额:单次上限 **1,000,000**,必须整数。
- 后端**故意没做"防自审"**(owner 要自己给自己塞起始资本)→ 单账号可自提自批,这是设计不是 bug。

### 🎴 德州扑克术语对照表(N4 表单 / B9-B15 引擎用,2026-05-28 用户钉死)

| 中文术语 | 实际含义 | 表单字段 | 待 B9-B15 实现的行为 |
|---|---|---|---|
| **抓头** | Straddle —— 枪口位置(UTG)玩家自愿加倍盲注 | `straddleEnabled` | B11 起手前给 UTG 玩家"加倍盲注"选项 |
| **把抽** | 每手按 % 从最终底池抽水 | `rakeType:'pot'` + `rakePercent` | B16 每手结算扣底池 × %(<= rakeCap) |
| **局抽** | 桌子时长(如 1h)到期,从赢家身上抽 % | `rakeType:'round'` + `rakePercent` + `durationHours` | B16 时长到期点算赢家盈利,扣 % |
| **买入抽** | 玩家每次买入扣 % | `rakeType:'buyin'` + `rakePercent` | B9 入座买入时扣 % |
| **入池率限制** | VPIP 低于阈值不能坐;新人前 N 手豁免 | `pflrLimitEnabled / pflrLimitRate / pflrLimitExemptHands` | B9 入座前查玩家 VPIP,不符 403 + toast"你的入池率低于 X%,无法坐下" |
| **补码上限** | 同桌买入累计上限 = 初始 × 倍数 | `rebuyLimitEnabled / rebuyLimitMultiplier` | B9 买入时累计校验 |
| **止损上限** | 单桌输到 N 倍买入自动离桌 | `stoplossEnabled / stoplossMultiplier` | B9 每手结算后查累计盈亏 |
| **延迟看牌** | Showdown 时获胜方可选择不亮牌 | `delayedRevealEnabled` | B11 摊牌环节给赢家"亮 / 弃"选项 |
| **付费看公牌 / 看手牌 / 切牌** | 玩家可付 N BB 查看 | `payToSee*Enabled` + `*Bb` | B11 / B12 中各对应交互点 |
| **观众禁言 / 抽签入座 / 保险 / 无限注** | 字面 | 对应 bool 字段 | B9 / B14 / 引擎初始化时读 |

**核心原则**:N4 表单**只采集**,**不执行**。所有规则在 B9-B15 引擎里读 `tables.settings` 字段执行。

---

## 4. 当前进度 & 3 个待修 bug

### 已完成 23/35

- 后端:B1 schema / B2 验签 / B3 用户 / B4 俱乐部(5位靓号)/ B5 邀请码(墓碑)/ B6 加入退出 / B7 钱包 / B18-21 申请发放回收 / N9配套接口。
- 前端:F1 验签 / F8 创建俱乐部 / F10 俱乐部列表 / N1 加入UI / N2→N2.1 / N9 通知胶囊 / page-coin-requests 审批页。
- 部署:D1 Railway / D2 Vercel / D3 Bot 全部 ACTIVE。

### 未完成(进度 28/35,本 sprint 推进 5 项)

- ~~B8 牌桌服务~~(已完成,`3538df8` / `9366962`)/ ~~**B8.5 入座离座**~~(**2026-05-29 新增完成**:后端 `0f8f935`+`4572f26` 共 4 路由含原子 wallet 联动 + my-seat 便利路由;前端 `560b11f`+`3123900`+`2888f29` 含真后端 sit/leave + 5s 轮询 GET seats 玩家可见对方)/ **B9 创建牌局** / **B10 WebSocket** / **B11-B15 游戏引擎** / **B16 结算入库**。
- ~~N3 封号 阶段 1+2 全闭环~~(`64004aa`+`ae42030`+`176f06f`+`3265afa`)/ ~~N4 创建牌桌表单 v2~~(`11d418a`+`76f2b7d`+`2b5c621`)/ ~~**N6 洗牌动画**~~(**2026-05-29 发现早已完成**:HTML `#shuffle-stage` line 13018-13022 + CSS keyframes line 2802-2855 + JS `window.playShuffleAnimation(callback)` line 39830 + hook 在 DEV `#dev-new-hand` 按钮)/ ~~**N5 保险池 UI**~~(**2026-05-29 发现早已完成**:完整"买保险"弹窗 HTML line 13400-13439 + CSS 5604+ + 状态值显示(边池/胜率/赔率)+ 滑杆 + 不买/买入按钮 + 12s 倒计时)/ **F9 真实 WS 切换**(待 B10 后端 WS 接通,前端从 mockServer 切真 WS)。

### B8 minimum slice 已完成 / 后续待做

**完成项**:
- DB:`tables` 加 5 列(`game_type / min_buyin / max_buyin / ante / status`)+ `idx_tables_status` 索引
- 后端 `POST /api/clubs/:clubId/tables`(admin 限,含 game_type 白名单/盲注关系/座位 2-9/短牌 max 6 座 校验)
- 后端 `GET /api/clubs/:clubId/tables`(member + 非封号可访问,沿用 N3 拦截)
- 后端字段 camelCase 出口(`gameType/blindsSb/blindsBb/minBuyin/maxBuyin`)
- 前端 `fetchClubTablesAndRender(clubId)` 在 `openClubLobby` 内调,映射 → 替换 `clubRooms[clubId]` → 重渲染
- 前端 demo / 失败兜底:无 token 跳过,catch 保留 cache,mock 88888 等保持不动(红线 1)
- 已部署:Railway(`3538df8`)+ Vercel(`9366962`)+ DB 迁移已跑

**未做(后续 ticket)**:
- 后端:`PATCH /tables/:id`(改盲/座/名) / `DELETE /tables/:id`(归档)
- 前端:**N4 创建牌局表单**(目前建桌需 Postman 或 SQL,N4 是其前端 UI)
- 前端:房间行的 `seats` / `time` 是固定 0/`00:00:00` —— 等 B9/B10 接入实时会话后才有真值
- B9 创建牌局 / B10 WebSocket / B11-B15 引擎 / B16 结算 —— 大块工程,独立 ticket

### N3 阶段 1 已完成 / 阶段 2 待做

**完成项(阶段 1)**:
- DB:`club_members` 加 `blacklist BOOLEAN DEFAULT FALSE` 列
- 后端 `PATCH /api/clubs/:id/members/:userId/blacklist`(admin 限,owner 不可封,不可自封)
- 后端 F10 / GET /clubs/:id 响应增加 `myBlacklisted` 字段
- 后端 `POST /api/clubs/:clubId/transfer-requests` 入口加 blacklist 校验 → 403
- 前端 F10 mapping 加 `myBlacklisted: !!c.myBlacklisted`
- 前端 `openClubLobby` 根据 myBlacklisted 给"申请发放/回收"按钮加 `.blacklisted` class
- 前端 click handler 检测被封号 → toast 「你已被封号,无法提交申请,请联系俱乐部管理员」

**阶段 1 验证方式**:Railway Postgres 直接 SQL `UPDATE club_members SET blacklist=TRUE WHERE club_id=11111 AND user_id=41;` 封啊勇,啊勇启动 Mini App 看大厅按钮灰显 + toast。

**留给阶段 2(下次对话)**:
- 后端:新建 `GET /api/clubs/:id/members` 接口(返回真实 users.id + role + blacklist)
- 前端:替换 `clubMembersData` mock,从该接口拉取
- 前端:把成员页"封号"按钮的客户端 mutation(34764/34771)改成 `apiClient.patch` 调上面 PATCH 路由
- **前置依赖**:阶段 2 必须先做 "成员列表接口",否则 mock uid 跟真 users.id 对不上,封号按钮 404

### ✅ 3 bug 全部归档(2 修 + 1 推迟到 MVP 2)

1. ~~**F10 替换失效**~~ ✅ **已修 2026-05-28(前端 commit `a4269e6`)**
   - 实际根因不是数据替换,F10 数据层 OK。是 [37708] 300ms `setTimeout` auto-enter 在 F10 完成前用 mock `clubs[0]='88888'` 进了内部大厅 UI(`lobby-name/lobby-id/房间列表`)。F10 后来只刷"列表",没刷"内部大厅"。
   - 修复:F10 success 后再 `openClubLobby(真 clubs[0].id)` + 同步 `currentClubInfoId`。副作用:启动时 mock 88888 一闪即过(<2s)再切真 11111,可接受。
   - 顺手:apiClient 严格校验 `data.ok===true` 也有过坑,F10 改裸 fetch 兼容 3 种 shape `{ok:true,clubs:[...]} / {clubs:[...]} / [...]`。
2. ~~**自动加入部分失效**~~ ✅ **已修(后端 commit `708af4c`,先于本次对话)**
   - 后端在 `src/routes/user.js:24` 加 `ensureAutoJoinSingleClub(userId)` helper,`/auth/telegram` 和 `/user/me` 都调,做兜底自动加入。
   - 飘雪 / 啊勇 已 SQL 补登 `club_members` + `wallets`(详见后端 CLAUDE.md)。
   - 根因:玩家在 `/auth/telegram` 那一刻若 clubs 数量不对(0 或 2+),又没主动重启 Mini App,就永久卡在外。
3. ⏭️ **N1 搜索 11111 显示"不存在"** —— **正式推迟到 MVP 2,非 MVP 1 bug**
   - 原因 1:N2.1"输 ID 申请加入"本就是 MVP 2 设计(MVP 1 是自动加入,见第 3 节)。
   - 原因 2:后端 **B17 加入申请服务不存在**。`/api/clubs/join` 走的是邀请码,`/api/clubs/:id` 要求是成员(403),都不是"搜索公开俱乐部 + 申请加入"用的。
   - 当前 `ALL_CLUBS_REGISTRY` mock 是 v2.28.73 写下时就声明的"等 B17 到位后改 `apiClient.post('/api/clubs/join', {clubId})`"(见 index.html:37512 注释)。
   - 不修不是技术 bug,是产品节奏决策 —— 红线 1 + 第 8 节 #4 都印证。

---

## 5. 部署流程(已锁定,别自创)

### 前端(A 类 aagame.html)

1. 从 Claude 项目库下载 `aagame.html`。
2. 改名 → `index.html`(Vercel 入口必须叫这个)。
3. GitHub `haoyu1668/aagame` 仓库 → `Add file → Upload files` → 拖 index.html → Commit。
   - ⛔ **不要"先删旧文件再传"**(同名自动覆盖,踩坑 76)。
4. Vercel 自动部署 → 域名 `https://aagame-six.vercel.app`。
5. Telegram 完全关闭重开,清缓存。

### 后端(D 类)

- `D:\Apps\aagame-backend` 里 `git push` → Railway 自动部署。

### ⛔ 别做

- 别给前端建多余本地 git 仓库做花样部署。
- 任何部署动作(push)前先让用户确认。

---

## 6. 数据库 schema 关键字段(改 SQL 前别瞎猜)

改任何 SQL 前,先跑 `SELECT column_name, data_type FROM information_schema.columns WHERE table_name='表名';` 查字段名,**别凭记忆猜**(历史踩坑:猜 `tg_user_id` 实际是 `telegram_id`)。

- **users**: `id`, `telegram_id`(不是 tg_user_id!), `username`, `first_name`, `last_name` (共 9 列)
- **club_members**: `club_id`(integer), `user_id`(bigint), `role`(varchar), `joined_at`(timestamptz)
- **clubs**: `id`, `name`, `treasury_balance`, `vault_balance`, `jackpot_balance`
- 8 张表:users / clubs / club_members / wallets / transfer_requests / tables / hands / invite_codes

---

## 7. 环境信息

| 项 | 值 |
|---|---|
| 后端项目 | `D:\Apps\aagame-backend`(Node + Express + Postgres) |
| 前端项目 | `D:\Apps\aagame-frontend`(index.html) |
| 后端 URL | https://aagame-backend-production.up.railway.app |
| 前端 URL | https://aagame-six.vercel.app |
| Railway 项目名 | acceptable-mercy(中文机翻"可接受的怜悯") |
| Railway 状态 | ACTIVE,commits 85197f0 + f58c319 |
| Postgres 查询 | Railway → Postgres → Database → Data → SQL 输入框 |

---

## 8. Claude 历史易犯错(引以为戒)

本项目换对话时,Claude 反复犯以下错(每次耗用户大量时间纠正):

1. ❌ 建多余本地仓库 / 自创部署流程 → ✅ 用第 5 节已锁定流程。
2. ❌ 把 `haoyu1668` 当 Telegram 号 → ✅ 它是 GitHub 账号,TG 号见第 1 节。
3. ❌ 把 mock 数据(88888 / demo 牌桌 01101-01103)当 bug 去"修"或"删" → ✅ 它们是 v2.28.73 兜底 mock,保留到大整顿。
4. ❌ 把"被邀请 / 输 ID 申请加入"当 MVP 1 逻辑 → ✅ MVP 1 是"自动加入",N2.1 申请加入是 MVP 2。
5. ❌ 猜数据库字段名 → ✅ 先查 information_schema。
6. ❌ 看到 AAPK 标题以为不一致 → ✅ 那是 Mini App Short name,正常。

**通用原则**:遇到任何"看起来像 bug / 像多余 / 像不一致"的东西,**先 `project_knowledge_search` 搜主文档验证,再开口**。本项目已偏离很多行业常规,通用经验经常出错。

---

## 9. 待办大 Ticket(2026-06-01 后)

### #149 牌桌会话整体审计(德州扑克规则合规)

- **状态**:Tier 0 已修(2026-06-01)。Tier 1 / Tier 2 待办(留独立 ticket)。
  - **已做(Tier 0,前端 index.html only,+46/−14,详见 `RELEASE_NOTE_149_Tier0.md`)**:
    问题1 waitbb 红卡背(渲染加 else 清 my-cards)/ 问题2 对手读秒+tick(删 setActingSeat 内 startTimer)/
    问题3 转正延迟看牌卡红背(加 _mySeatDR 守卫)/ 问题4 .folded 多源(轮询全座位权威对账)。
  - **待办 Tier 1(audit 顺带,值得修)**:I4 离桌不停 timer/poll · G3 muck 权不限赢家 ·
    H1 后端不读 delayedRevealEnabled · H1 muck 时序错配 · I3 dissolve 判定重复+无行锁 ·
    I1 两离座路由局中语义不一致 · G4 奇数筹码 remainder 不按位置 · K3 live 不下发分层边池。
  - **待办 Tier 2(大工程,独立 ticket)**:C4 dead button 未实现 · G2 逐位摊牌顺序缺失 ·
    D1/D4 街结束判定靠计数近似 · I2 sit-out 完全缺失 · 保险池后端纯占位(B16+)· A1 正式状态机(大整顿)。
- **背景**:2026-05-31 到 06-01 长对话做了 150 个 task,发现 4 个牌桌会话逻辑问题需要整体审计:
  1. **waitbb 玩家不该看到任何卡**(包括卡背)— 用户中途坐下显示 my-cards 红色卡背
  2. **timer/tick 只对 current actor 启动** — 当前对手 acting 时也会 startTimer(visual countdown 设计本意,但用户感觉异常)
  3. **延迟看牌中途坐下逻辑可能错乱** — delayedRevealEnabled=true 时,中途参与的玩家卡片可能停在卡背状态
  4. **弃牌灰显有多条路径加 .folded class** — fullResetTable + polling 防御已加,但根本 fix 需要审

- **流程要求**(严格按此走):
  1. 新对话冷启动 → 读 CLAUDE.md 拿项目状态
  2. 列一份"德州扑克牌桌会话规则清单"(参考 PokerStars / WSOP 国际标准)
     - 覆盖:座位生命周期 / waitbb / blinds 旋转 / dead button / SB+BB / 行动顺序 / timer / 弃牌 / showdown / 延迟看牌 / muck / 摊牌优先级 / 离桌 / 解散
  3. **不要立刻改代码** — 先让用户审你列的清单
  4. 用户同意后,对照代码 audit,列不合规点
  5. 用户审 plan,再批量修

- **关键背景**(节省你重复查的时间):
  - bet/raise 决策点 3 处(line 29403 / 32590 / 32985)都已改成 `opts.includes('bet')`,任何"硬塞 bet"的新代码必错
  - waitbb 严格规则已实现(只在 BB 位强制参与),代码在 `hand.js` `startHandCore` 中 `checkSessionLifecycle` 之后
  - 自动解散 lifecycle 3 个触发点:seat.js sit + startHandCore prefix + GET /current-hand polling(2026-06-01 加)
  - 旁观系统真接通,schema `table_spectators` 表已建,前端 `_b9SpectatorHeartbeatTimer` 30s 心跳
  - 大量 mock 数据已删(`demoRecordPlayers` 31 / `demoSpectators` 24 等),不要再退回 mock

- **工时上限**:2-3 小时(给用户报具体估时)
- **风险**:核心游戏逻辑,改错影响所有玩家 → **务必先列清单 + 用户审,再动代码**

---

*CLAUDE.md · 对应主文档 v2.28.81 · 进度 ~95%(150 task 完成后)· 最后更新 2026-06-01*
