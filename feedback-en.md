# Feedback for OneTab team (one-tab.com/feedback)

Paste this into the form at https://www.one-tab.com/feedback
(内容为英文，适配官方表单；提交前请把尖括号内占位符替换为你的实际信息)

---

**Category:** Bug report / performance issue

**Title:** OneTab page drops frames with thousands of saved tabs — root-caused, with a verified one-line CSS fix

Hi OneTab team,

First of all, thank you for maintaining OneTab — I use it every day with roughly 4,200 saved tabs and it is essential for me. After the 2.x rewrite (data layer moved into the background service worker), opening and scrolling the OneTab page started visibly dropping frames, even on a fast machine with a 144 Hz display. I profiled it and wanted to share a clear root cause plus a fix I verified with real data.

**Reproduction / environment**
- Browser: Edge 152 (Chromium, MV3) on Windows; OneTab 2.18
- Data size: 4,396 saved items (4,230 tabs + 166 groups, 553 currently in Trash), IndexedDB ≈ 2.3 MB
- Symptom: scrolling / moving the mouse over the list is janky; hit-testing feels heavy in DevTools Performance traces

**Root cause (from a 26.4 s trace while scrolling)**
- The page renders ALL items into the DOM upfront: no virtual scrolling, no `IntersectionObserver`, no `requestIdleCallback`. DOM grows to ~42,000 nodes; the `Layout` event reports `totalObjects: 41,555`.
- Mouse hit-testing is the bottleneck: 430 `HitTest` events, **p50 13.09 ms, p90 14.63 ms, max 20.09 ms**; 59% of hit tests exceeded 4 ms. On a 144 Hz display one frame budget is ~6.9 ms, so a single hit test costs 2–3 frames.
- I ruled out favicon loading, `box-shadow`, and JS event handlers with before/after experiments — the cost is proportional to the number of rendered layout objects.

**Verified fix (one CSS rule, no JS changes)**
```css
.tab { content-visibility: auto; contain-intrinsic-size: auto 26px; }
```
Average hit-test cost drops from **12.25 ms → 6.86 ms (−44%)** on the same 4,396-item dataset (`document.elementFromPoint` × 60 sampling). I have been running it for days with no functional regressions — drag & drop reordering, grouping, search, restore all behave normally. (`contain-intrinsic-size` keeps the scrollbar geometry and drop targets accurate even though offscreen items are skipped.)

See the full experiment table (including a `contain: layout paint style` variant that was *harmful*, 12.25 → 15.09 ms) and the reproduction steps here: https://github.com/<your-username>/OneTab-Performance-Patch/blob/main/docs/findings.md

**Suggestions for the official implementation (in order of preference)**
1. Virtualized rendering (or chunked/interleaved rendering) for large lists — the real fix; it also bounds memory usage.
2. At minimum, apply `content-visibility: auto` + `contain-intrinsic-size` to offscreen items in `onetab.css` (trivially small diff).
3. Consider lazy-rendering or paginating the Trash section — 553 of my 4,396 items sit there and currently pay full render cost.

I'm happy to share the full trace or help reproduce further. Thanks again for a genuinely useful tool!

---

P.S. 提交建议：表单是官方唯一反馈入口（他们没有公开 GitHub 仓库），开发者实际会看反馈并回复。若 1–2 周无回应，可在 Edge/Chrome 商店的 OneTab 评论/评分区简述一句"几千条收藏时滚动掉帧，已按 one-tab.com/feedback 提供详细数据"，提高看到概率。