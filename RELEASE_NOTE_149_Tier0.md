# Release Note — Ticket #149 Tier 0 修复(2026-06-01)

> 双输出(红线 2)配套文档。对应 A 类 `aagame.html` / `index.html` 改动。
> 可直接并入项目库 B 主文档 或作为 C 恢复包片段。

## 背景

Ticket #149「牌桌会话整体审计」第 1 步列出德州扑克会话规则清单(11 组),
第 4 步对前后端代码全量 audit,产出《不合规点表》。
本次(Tier 0)**只修 4 个已知问题,全部前端 `index.html` 单文件,后端零改动**。
Tier 1(I4/G3/H1/I3/I1/G4/K3)与 Tier 2(C4 dead button / G2 逐位摊牌 / D1-D4 街结束 /
I2 sit-out / 保险池后端)均留独立 ticket,本次不动(守红线 1)。

## 4 处改动明细

### 修复1 — waitbb 玩家显示红卡背(高,前端纯 UI bug)
- **现象**:中途坐下处于 waitbb 的玩家,`.my-cards` 显示静态写死的红 K♥/10♦。
- **根因**:后端正确返 `myHoleCards:[]`;前端三处叠加 —— ① `.my-cards` HTML 静态写死两张红牌
  (index.html:13346)② 渲染只在 `myHoleCards.length===2` 才调 `renderMyHoleCardsB9`,空数组时不清空
  ③ CSS `hand-active .my-cards{display:flex!important}` 无条件显示。
- **改法**:渲染处加 `else` 分支,无底牌时清空 `.my-cards.innerHTML` + 去 `delayed-back`/`muck-back`。
  静态牌保留不删(红线 3:DEV 切换器用),仅运行时清。
- **位置**:index.html 底牌渲染 `if (hand.myHoleCards.length===2){...} else {...}`(约 34429)。

### 修复2 — 对手 acting 时我端也读秒 + tick 声(中,设计噪音)
- **现象**:轮到对手行动,我端也起 15s 倒计时环 + 滴答声。
- **根因**:`setActingSeat()` 对任意座位(含对手)无条件 `startTimer`(index.html:30683);
  与我自己回合的计时器(由 actionArea data-state p1-p4 的 MutationObserver 驱动,32399)是两套并存机制。
- **改法**:删 `setActingSeat` 内的 `startTimer` 调用,只保留绿环 `classList.add('acting')`。
  我自己计时器不受影响(observer 仍驱动);对手 acting 只亮绿环,不读秒不滴答。
- **安全性**:已验证我方计时不依赖该行。不影响超时 auto-fold/check(`onTimeout` 仍由 ACTION_REQUEST 驱动)。
- **位置**:index.html:30682-30683。

### 修复3 — 转正 + 延迟看牌卡红卡背(中,衍生于问题1)
- **根因**:延迟看牌块(index.html:33999)在 `delayedRevealEnabled` 时,对不在 `hand.seats` 的
  waitbb/旁观玩家(`_mySeatDR=null`)仍命中 `!armed && !_myActing` → 给本该空的 my-cards 加 `delayed-back`(红背)。
- **改法**:加 `_mySeatDR` 守卫,不在本手时直接 `remove('delayed-back')`,只有在本手才走 armed 盖牌逻辑。
- **位置**:index.html 延迟看牌块(约 34003)。

### 修复4 — `.folded` 灰显多源易残留(中-高)
- **根因**:`.folded` class 有 3 处 imperative 加(ACTION_EXEC 29938 / 引擎 30322 / 态驱动 renderSeats 16584），
  live B9 牌局不走 renderSeats 整体重渲,缺单一权威渲染源。原有"防御性清"只覆盖 my-seat。
- **改法**:把轮询里的防御对账**从 my-seat 扩展到全座位** —— 每 2s 以后端 `hand.seats[].folded`
  为唯一真值,对所有 `.seat.sN` 做 add/remove('folded')(跳过我自己 seatNo,走 my-seat 那段)。
  保留即时 addClass 给瞬时反馈,轮询每 tick 纠偏漂移,不推翻 imperative 层(守红线 1)。
- **位置**:index.html 轮询诊断块(约 34147)。

## 验证建议(真机)
1. waitbb 玩家中途坐下 → `.my-cards` 应为空,**无红卡背**。
2. 对手回合 → 只见对手头像绿环,**我端无读秒环、无 tick 声**;轮到我时计时正常。
3. delayedRevealEnabled 桌,waitbb 玩家转正 → 卡片应正常显真牌,**不卡红背**。
4. 多人弃牌后 → 弃牌座位灰显干净,无残留;后端纠偏每 2s 生效。

## 影响面
- 仅前端轮询/UI 行为,无后端、无 DB、无 schema 改动。
- 不涉及 mock 数据(88888/demo 牌桌),不删任何代码(守红线 1/3)。

---

# 追加修复(同对话续,2026-06-01 真机测试驱动)

> Tier 0 四项上线后,真机连测中又发现并修复了一批牌桌会话问题。
> 前端在 `aagame-six`(Vercel),后端在 `aagame-backend`(Railway)。

## 前端(index.html)

- **修复1b**:waitbb 清 `.my-cards` 后,牌型引擎仍读静态 `DEMO_HOLE_CARDS.my=['K♥','10♦']`
  显示假牌型("两对 K-8")。→ 不在本手时清空 `DEMO_HOLE_CARDS.my` + 移除残留牌型标签。
- **修复1c(回归修复)**:修复1 用"myHoleCards 是否 2 张"判据 + 每手只跑一次的渲染块,
  导致 waitbb 转正首帧无 myHoleCards 时误清 my-cards 且不再补渲 → **"看不到手牌"**。
  → 撤销该 else 清空;新增**每-poll 自愈块**:判据改为"我是否在 hand.seats"(`_myActiveSeat`),
  在局有牌但 my-cards 空 → 补渲染;确认不在局 → 才清静态红卡背。
- **修复5 / 5b**:真桌对手头像**无牌背**。根因 CSS 3366 `display:none!important` 隐藏所有对手
  `.seat-cards-back`,只给 my-cards 加了 hand-active override,漏了对手。
  → 加 `hand-active` override 显示在局对手牌背(排除 空座/满员/等大盲/弃牌);
  fetchAndRenderTableSeats 从 hand.seats 富化 `player.folded` 让 renderSeats 可靠隐藏弃牌对手牌背。
- **修复6**:对手牌背"直接 pop 出来",非"飞牌落地后才出现"。根因发牌动画用内联 opacity
  被每 2s renderSeats 重建冲掉。→ 改用持久 class `holes-revealed` 门控:每手开始移除(藏),
  发牌动画飞牌落地后(~1.5s)或中途进场(无动画)再加上(显)。
- **修复7**:弃牌气泡过早消失。→ `clearActionTags(includeFold)` 默认保留 `.action-fold`,
  仅新一手 fullResetTable 传 true 才清;renderSeats 据 `player.action='fold'` 持久重建弃牌气泡
  → 弃牌气泡留到本手派彩结束。
- **修复8**:call/raise/check/bet 气泡被每 2s renderSeats 抹掉。→ 维护 `_b9StreetActions`
  记录对手本街动作,renderSeats 据此持久重建;街/手边界(clearActionTags)清空记录
  → 动作气泡活到本街结束才消失(弃牌除外,走修复7)。
- **修复10**:牌局中点"退出房间"被留在桌上干等。→ 申请退出(pending_leave)成功即
  cleanupTableSession + doReturnToLobby **立即回大厅**;后端轮到他时自动弃牌、本手结束退筹码回钱包。
- **修复11**:对手头像缺绿色倒计时圈(修复2 去 tick 声时连视觉圈一起砍了)。
  → setActingSeat 里用**独立 rAF 纯视觉动画**推进对手 `.seat-turn-ring` 的 `--turn-p`(~15s),
  无 tick 声、无 onTimeout、不碰"我的"行动计时器。

## 后端(aagame-backend / src/routes/hand.js)

- **修复9**:街结束判定从 `COUNT(*) 总动作数 >= 活跃人数` 改为 **"每个活跃(未弃/未 all-in)
  玩家本街都真行动过"**(去重座位交集)。原 bug:已弃牌玩家的动作"顶替"未行动活跃玩家名额
  → 街提前结束、**抓头/BB 的最后 option 被跳过**(日志 hand 536:straddle s2 翻前从未行动直接进 flop)。
- **修复12**:行动超时默认动作从"一律 fold"改为 **"能 check 就 check,否则 fold"**。
  无需跟注(current_bet 已是最高)→ 超时自动 check(不再误弃能免费过牌的人,如 BB option);
  面对下注 → fold。新增 `advanceStreetOnTimeout` helper(镜像 POST /actions 街结束逻辑,
  不依赖 res),处理 check 关闭本轮时的"推进街/发公牌/摊牌",避免全员掉线 check 到底卡住。
  MAX_LOOPS=12 仍兜底。
- **(配套设计,无需改)场景:关闭小程序**:`tryExpireTimedOutAction` 在每个 GET /current-hand
  轮询入口跑(hand.js:1125),只要桌上还有人轮询,掉线玩家轮到他时由服务端超时自动处理。

## 部署
- 前端:Vercel,逐 commit push(修复1b…11)。
- 后端:Railway,commit `eaf3693`(修复9)、`a10f93a`(修复12)。无 DB 迁移。

---
*Tier 0 原始 commit:见 git log #149 Tier 0 · index.html +46/−14。追加修复见上。*
