# 沪牌拍卖尾盘 Simulation — Design

> 极简原型，用来验证手动选择价格与出价时刻是否能在可复现的盘尾剧本下中标。非生产、非官方复刻。

## 1. Goal

在浏览器里回放最后约 10 秒的竞价：

- 程序按 **seed + 月份锚点** 生成可复现的跳价路径与人群出价。
- 可视化价格 bucket、入列顺序、当前可出价区间、胜出队列。
- 玩家可在任意时刻点击有效区间内的 bucket，立刻看到红点落点与排名。
- 默认场景使用 **2025-10** 锚点。

成功标准：同一 seed 刷新结果一致；点击不同时间与价格能直观改变胜负；UI 一眼能读懂「现在能报啥、谁在赢」。

## 2. Non-goals

- 不做登录、验证码 OCR、真实国拍协议。
- 不做玩家网络延迟模拟；手动点击时立即写入。
- 不宣称数字等于官方；月份数据是公开结果锚点 + 合理生成，不是代拍原始入列日志。
- 不做批量蒙特卡洛扫参 UI（可后续加）；本版只做单次可视化回放。

## 3. Locked decisions

| 项 | 选择 |
|---|---|
| 跳价路径 | **B**：seed + 月份参数程序生成，可复现 |
| 技术栈 | **A**：单文件 `index.html`（Vanilla JS + CSS），零构建 |
| 玩家策略 | 手动点击当前有效区间内的 bucket，立即写入一次 |
| 默认月份 | 2025-10（可切换其它已录入月份） |
| 视觉 | 极简：黑白灰底 + 黄高亮胜出 + 红点自己；无营销风装饰 |

## 4. Auction rules (sim model)

与公开规则对齐的最小集合：

1. **价格只上不下**：最低可成交价 `L(t)` 阶梯上跳（步长 100），不回落。
2. **可报区间**：任意时刻可提交价 `P` 必须满足 `L(t) - 300 ≤ P ≤ L(t) + 300`，且 `P` 为 100 的倍数。
   **硬性不变量**：任意播放时刻，所有已经出现的 NPC 报价都不得高于当前有效出价区间上限 `L(t) + 300`。
3. **按各自出价成交**：中标者付自己的有效出价，不是统一价。
4. **排序**：有效出价按 `(price DESC, writeTime ASC)`；前 `quota` 名胜出。
5. **写入时刻**：`writeTime = clickTime + delayY`（NPC 的 delay 在生成时已折进 writeTime）。
6. **处理时校验**：点击只代表入列；数据库在 `writeTime` 处理时，以该时刻的 `L(writeTime) ± 300` 判定修改是否有效。
7. **单次出价简化**：每名模拟竞买人只产生一笔报价。报价处理成功后点才出现；越界或超时则本场没有有效报价。
8. **收盘**：内部时间 `t ∈ [0, D]`，`D = 10`；`t = D` 对应 11:30:00，之后不再接受写入。UI 显示剩余秒 `remaining = D - t`。

理论推论（用于解释 UI，不单独实现）：全场有效出价 ≤ `F + 300`，其中 `F` 为最终最低成交价。

本原型采用**价格接受者假设**：先生成不含玩家的可复现市场 tape，玩家报价不会反向改变 NPC 行为或系统价格路径；但有效玩家报价会放回完整市场中撮合。这样避免一个玩家改变后续随机市场，同时明确它不是交易所级全反馈仿真。

## 5. Data: month anchors

每个月份是一组**锚点**，不是完整真实日志：

```ts
type MonthAnchor = {
  id: string;          // "2025-10"
  label: string;       // "2025年10月"
  quota: number;       // 投放额度（胜出人数）
  participants: number;// 参拍人数（生成 NPC 规模用）
  finalFloor: number;  // 官方最低成交价 F
  averagePrice: number;// 官方平均成交价
  winningRate: number; // 官方中标率（百分比）
  maxPrice?: number;   // 场景最高报价；默认不额外限制
  initialFloorOffset?: number; // 尾盘窗口起始最低价相对 F 的差值，默认 200
  initialBidShare?: number; // 尾盘窗口前已出价人数比例，默认 0.58
  cutoffRemaining: number; // 官方最低成交价档最后中标者距收盘秒数
  cutoffRank: number; // 官方最低成交价截止位次
  jumpAggressiveness: number; // 0..1，越大越晚才跳到 F
};
```

内置至少：

| 月份 | 额度 | 参拍人数 | F | 平均成交价 | 中标率 | 最低成交价截止时间 |
|---|---:|---:|---:|---:|---:|---|
| 2025-10 | 4403 | 23586 | 93800 | 93874 | 18.7% | 11:29:50 第1103位 |
| 2025-09 | 4230 | 25210 | 93700 | 93778 | 16.8% | 11:29:59 第930位 |
| 2025-08 | 4061 | 28264 | 93800 | 93868 | 14.4% | 11:29:51 第1262位 |
| 2025-07 | 4286 | 30798 | 93600 | 93679 | 13.9% | 11:29:59 第887位 |
| 2025-06 | 3809 | 33834 | 94300 | 94368 | 11.3% | 11:29:59 第1212位 |
| 2025-05 | 4101 | 35649 | 94300 | 94368 | 11.5% | 11:29:59 第1302位 |
| 2025-04 | 3874 | 38639 | 94200 | 94391 | 10.0% | 11:29:59 第38位 |
| 2025-03 | 3710 | 40205 | 94100 | 94100 | 9.2% | 11:29:59 第1797位 |
| 2025-02 | 3793 | 39780 | 94100 | 94100 | 9.5% | 11:29:59 第3793位 |
| 2025-01 | 3784 | 42564 | 94100 | 94159 | 8.9% | 11:29:58 第2196位 |

缺官方位次/人数时：用合理默认 `quota ≈ 11000`、`participants ≈ 2.5×quota`，并在 UI 标注「估算」。模拟直接使用月份锚点中的完整额度与参拍人数，并始终保证 `participants > quota`。

## 6. Procedural generation (seed)

### 6.1 PRNG

`mulberry32(seed)` 或等价可复现 32-bit PRNG。所有随机只走这一路。

### 6.2 Initial book

尾盘窗口开始时，已有一部分 NPC 完成其唯一一次报价，保证 book 人数不少于 `quota`，从而可以定义 `L(0)`：

```ts
type EffectiveBid = {
  bidderId: string;
  price: number;
  writeTime: number; // 首次报价使用负数，表示发生在尾盘窗口之前
};
```

1. 直接按月份锚点的完整 `participants` 与 `quota` 生成 `participants - 1` 个 NPC。
2. 默认约 58% NPC 的唯一报价发生在尾盘窗口之前；高竞争场景可下调该参数（2025-10 为 48%）。起始 book 始终超过 `quota` 人，初始报价严格在 `L(0) ± 300` 内构造。
3. 任意时刻的 `L(t)` 均由当时完整有效 book 按 `(price DESC, writeTime ASC)` 排序后的第 `quota` 名价格推导，而不是单独随机生成。
4. 为避免逐事件全量排序，实现可用价格 histogram 求第 `quota` 档；最终名次再做一次稳定排序。

### 6.3 NPC bids

1. 尚未报价的 NPC 在尾盘窗口内提交其唯一报价；写入时刻密度挤在最后 3 秒，少数早到。
2. NPC 延迟已折进 `writeTime`，在处理事件时读取当前 `L(writeTime)`。
3. 报价从合法区间七个 100 元档位中采样，偏向上半区。
4. 每人只入列一次；有效报价只新增到 book，不替换或移动任何旧点。
5. 顺序处理后输出 `{ initialBook, acceptedEvents, floorEvents }`。`floorEvents` 是 book 的派生结果。

### 6.4 Derived F in sim

回放结束后，按排序取第 `quota` 名的 price 作为 **sim F**。  
生成参数以官方 F 为分布锚点，但不通过事后拒绝采样篡改单个结果；若单次偏离，UI 显示 sim 真相，并小字提示官方锚点。

官方 `cutoffRemaining` 仅用于结果对照：模拟截止由最终最低价档中最后一名中标者的 `writeTime` 推导。不得用官方截止时间反推 `L(t)` 何时到达 F。

## 7. Player bid model

播放或拖动到任意时刻，只能点击当前有效区间内的价格 bucket。本场只接受一次点击：

```
clickAbs = 当前播放时刻
writeAbs = clickAbs
Z = 点击的 bucket 价格
if Z ∉ [L(writeAbs)-300, L(writeAbs)+300]:
  bidStatus = invalid-out-of-range
  finalBid = null
else:
  bidStatus = valid
  finalBid = { price: Z, writeTime: writeAbs }
```

UI 显示 `valid | invalid-out-of-range | late`；只有 `valid` 才渲染玩家红点并参与撮合。

## 8. Winning set & highlight

**时间约定（全局唯一）**

- 内部：`t = 0` 模拟窗起点，`t = D = 10` 收盘；**越大越晚**。
- UI：`remaining = D - t`。
- 排序：`(price DESC, writeTime ASC)` —— 同价越早写入越靠前。

**播放语义**：播放头 `t_play` 下，已写入 = `writeTime ≤ t_play`。

胜出集合：以初始 book 为基线，加入 `writeTime ≤ t_play` 的单次有效报价后，按 `(price DESC, writeTime ASC)` 排序，前 `quota` 名构成固定容量的有效出价队列。所有已写入报价点持续显示到收盘，窗口变化只改变其黄/灰状态，不删除或移动点。玩家只影响最终排名，不反向改写预生成市场 tape。

点与黄高亮：

- 每笔报价在 `writeTime` 到达时渲染为灰点，并永久保留到拍卖结束；玩家报价始终渲染为红点。
- 有效出价队列容量固定为 `quota`，成员按 `(price DESC, writeTime ASC)` 动态取前 `quota` 名。
- 灰点同时处于当前有效出价区间 `[L-300, L+300]` 且位于有效出价队列时变黄。
- 黄点掉出有效出价队列时恢复为灰点。
- bucket：若该价位上存在黄点，则显示黄底。

箭头：指向当前可报区间 `[L-300, L+300]` 覆盖的 bucket 范围。

## 9. UI layout

单栏，自上而下：

```
[月份▾] [seed ____] [重新生成]
────────────────────────────────
 Bucket 行： … 93600 93700 93800 93900 94000 94100 …
   每个 bucket 内纵向/横向堆叠 dots（先写入靠近「队尾」标记端）
   悬停 tip：price · writeTime · id
   ▲ 箭头标可报区间
────────────────────────────────
 倒计时  剩余 ss.s
 可报区间  L-300 – L+300
 平均成交价  最低成交价 / 官方F锚点  最低成交价最后中标用户提交时间 / 官方参考
 胜出队列 / 已出价
────────────────────────────────
 ▶ ❚❚  ──●────────  progress   ↻
────────────────────────────────
 我的手动出价
  点击当前有效区间内的价格 bucket
  结果：写入时刻 · 是否合法 · 是否中标 · 名次
```

Playbar：

- play / pause
- drag scrub（拖动时暂停）
- replay（回到 t=0 并 play）
- 播放速率可选 0.1× / 0.5× / 1×，默认 1×

Bucket 数量：以 `F` 为中心左右各约 6～8 档，超出用 `…`；只渲染有点的档 + 当前区间邻域，避免上千 DOM。

Dots：市场计算不抽样；UI 中同价位且胜出状态相同的报价每 100 人聚合为一个点，悬停显示聚合人数和提交时间范围。报价写入后持续存在到收盘，聚合点按最早 `writeTime ASC` 排列。每帧从全部已写入报价中取前 `quota` 名更新黄/灰状态；玩家报价参与聚合，玩家所在聚合点显示为红色，不额外插入或放大独立点。

## 10. Architecture (single file)

```
index.html
  ├── <style> 布局与极简视觉
  ├── <div id="app"> 结构
  └── <script>
        monthAnchors
        prng / generateInitialBook / generateMarketTape
        simulateManualPlayer
        ranking / winningSetAt
        renderBuckets / renderHud / renderPlaybar
        animation loop (requestAnimationFrame)
```

状态对象：

```ts
state = {
  monthId, seed,
  scenario: { initialBook, acceptedEvents, floorEvents, quota, officialF, ... },
  player: { price, clickTime, writeTime, result },
  playback: { t, playing, duration: 10 },
}
```

改 seed/月份 → 重新 `buildScenario()` 并停在 t=0。

## 11. Approaches considered

1. **纯数据回放（固定 JSON 时间线）**  
   真实感强，但不能扫 seed，缺数据就卡死。否决（用户选 B）。

2. **Seed 生成 + 单文件可视化（本设计）**  
   可复现、可调、零构建。推荐。

3. **Vite/React + worker 批量扫参**  
   适合以后做胜率热力图；对本版验证 idea 过重。留作 v2。

## 12. Implementation order

1. 静态壳：bucket 行 + HUD + playbar + 手动出价结果。  
2. PRNG + floor path + NPC + 排序。  
3. 播放头驱动高亮与箭头。  
4. 玩家点击 bucket 立即写入红点与结果文案。  
5. 月份切换与 seed 复现自检。  
6. 性能：dots 抽样、rAF 节流。

## 13. Verification

- 固定 seed=1、月份=2025-10，刷新两次 scenario 哈希一致。  
- 任意月份均满足 `participants > quota`，且 `NPC dots + 玩家 dot = participants`。
- 对每个 `floorEvent` 重放 book，第 `quota` 名价格等于事件记录的 `L`。
- 只能点击当前有效区间内的 bucket，且 `clickTime === writeTime`。  
- 点击后红点立即出现并保留到收盘，位置不随后续报价移动。  
- 点击不同价格与时刻能出现「中标 / 未中标」两种结果。  
- scrub 到盘中，黄点集合随已写入子集变化。
- 任意播放时刻可见点数等于截至该时刻的有效报价人数；黄色点均处于有效区间和前 `quota` 名队列；每个 bucket 内点的 `writeTime` 单调递增。

## 14. Open knobs (not blockers)

- 精确 `quota` / `participants` 可用日后数据表替换。  
- 播放倍速、验证码耗时拆成独立参数。  
- 导出当前 scenario JSON。

---

确认本设计后，下一步写实现计划并落地 `index.html`。
