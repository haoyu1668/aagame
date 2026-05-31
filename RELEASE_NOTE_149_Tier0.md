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
*对应 commit:见 git log #149 Tier 0 · index.html +46/−14*
