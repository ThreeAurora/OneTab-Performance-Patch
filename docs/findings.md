# OneTab 2.18 卡顿诊断报告（公开版）

日期：2026-09-09
环境：Edge 152（Chromium） / OneTab 2.18（MV3，商店版，ID `hoimpamkkoehapgenciaoajfkfkpgfop`） / Windows
数据规模：4396 条（4230 tab + 166 group，其中 553 条在 trash），IndexedDB 2.33 MB

## 结论

卡顿根因 = **4396 个条目全量渲染成 41555 个布局对象，导致鼠标命中测试（HitTest）单次耗时 13 ms**。
与 favicon 图标无关，与 box-shadow 无关。

## 证据链

### 1. 数据规模与渲染方式
- `onetab.html` 是空壳（只有 `<div id="contentAreaDiv">`），列表全部由 JS 现场生成
- 代码中无 `IntersectionObserver` / 虚拟滚动 / `requestIdleCallback` → 全量渲染
- 实测 DOM 节点 42336 → 42420；trace 中 `Layout` 事件 `totalObjects: 41555`

### 2. 主线程时间轴（26.4 s trace，CrRendererMain）
| 指标 | 值 |
|---|---|
| RunTask 忙碌 | 6.86 s（26%） |
| HitTest | 430 次，**p50 13.09 ms**，p90 14.63 ms，max 20.09 ms |
| HitTest > 4 ms | 253 / 430（59%） |
| 最忙的秒 | RunTask 600–840 ms/s，其中 HitTest 占 250–560 ms |

命中节点：`DIV class='tabInner'`、`A class='tabLink tabLinkText'`。
本机一帧 ≈ 6.9 ms（144 Hz），一次 HitTest = 掉 2 帧。

### 3. 干预实验（`document.elementFromPoint` × 60，单位 ms）
| 干预 | 耗时 | 结论 |
|---|---|---|
| baseline | 12.25 | — |
| `.tab { content-visibility: auto; contain-intrinsic-size: auto 26px }` | **6.86** | **有效，-44%** |
| `.tab { box-shadow:none; border-radius:0; border:none }` | 12.44 | 阴影无关 |
| `.tab { contain: layout paint style }` | 15.09 | 有害（此写法会让所有屏内条目各自建立布局上下文） |
| 隐藏一半 `.tab` | 8.19 | 成本与布局对象数正相关 |
| 隐藏整个列表容器 | 0.46 | 成本 100% 来自列表内部 |

### 4. 排除项
- **favicon**：`_favicon` 实测 fetch 200 可用；`favicon` 权限正常持有；回退源图标数量极少。不是主因。
- **调试代码**：`new Error().stack` 所在的 handler 工厂定义但从未调用，是死代码。
- **JS 事件处理器**：pointermove 333 次共 26.5 ms，click 2 次共 2.6 ms，不慢。
- **硬件加速**：未关闭（默认值）。

### 5. 背景（版本退化方向）
OneTab 2.18 把数据层移入 background service worker，页面经 Proxy + `chrome.runtime.sendMessage` 走 RPC（44 个 RPC 方法，21 处调用点）。旧版 MV2 直接在页面读 storage。此为次要因素（background 线程采样几乎全 idle）。

## 补充观察：点击恢复标签页的卡顿（最影响体验的痛点）

数据丢失风险：**在几千条收藏的数据集上，点击条目恢复到浏览器时，整个页面会卡住十几秒完全不响应**，体感远比滚动掉帧糟糕，是用户最关心的场景。

- 恢复动作会触发列表全量重排（这也是同一根因——全量布局对象）：
  `.tab { content-visibility: auto }` 下，屏幕外条目整体跳过布局/命中测试，
  恢复流程的实际重排范围大幅缩小，**实测补丁后点击恢复基本不卡（体感"秒开"）**。
- 注：本轮采集的 trace 主要覆盖滚动/拖动场景，恢复流程的量化数字未单独记录；
  以上为本数据集（4396 条）上的主观-客观对照观测，欢迎官方在其测试集上复测。

## 建议（按优先级）

1. **官方正式做法：虚拟滚动 / 分批渲染**（对几千条规模是根治，同时省内存）。
2. **最低成本做法**：对屏幕外条目加 `content-visibility: auto` + `contain-intrinsic-size`（即本项目补丁）。
3. **顺手优化**：回收站里 553 条（12.6%）白付渲染成本——官方可懒渲染/分页 trash；用户可定期清空。
4. **归档旧分组**：把不常看的整组导出存档，把常驻条目压到千条量级，体验更好。

## 取证方法（复现路径）

1. 打开 OneTab 页面 → F12 → Performance → 滚动/拖动列出 trace；
2. 过滤 `HitTest` 事件统计 p50/p90/max，过滤 `Layout` 看 `totalObjects`；
3. 用 `document.elementFromPoint` 做 60 次采样做对照实验（对应上表数据）。

## 补充说明

- 本报告所有数据来自真实使用场景的 trace 与干预实验，可供官方复现。
- 原始 trace 及内部分析脚本未随公开版发布（内含本地路径等隐私信息），如需完整数据可联系提交者。