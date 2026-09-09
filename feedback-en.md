# Feedback for OneTab team (one-tab.com/feedback)

Paste this into the form at https://www.one-tab.com/feedback

---

**Category:** Bug report / performance

**Hi OneTab team,**

With a few thousand saved tabs, OneTab becomes painful to use on my (fast) machine:

1. **Clicking a tab to restore it freezes the whole page for ~10+ seconds.**
2. Scrolling / moving the mouse over the list also drops frames (144 Hz display).

**Root cause I found:** the OneTab page renders **every** item into the DOM up front (no virtualization). With 4,396 items that's ~42,000 nodes / 41,555 layout objects, so basically every interaction that touches the list forces a full re-layout.

**A one-line CSS fix works very well for me** (no JS changes, just appended to `onetab.css`):

```css
.tab { content-visibility: auto; contain-intrinsic-size: auto 26px; }
```

Result on my 4,396-item data: **restoring a tab went from ~10+ s of frozen UI to near-instant; scrolling/over hit-testing roughly halved** (12.25 ms → 6.86 ms per hit test by sampling, but honestly the big win is the restore flow — it went from actively painful to not noticeable). Search, drag & drop reordering, grouping all still work.

Could you try the same for the official build? Even better long-term would be virtualized rendering (or `content-visibility` in `onetab.css`) and lazy-rendering / paginating the Trash section.

Happy to provide the full trace. Really hope this helps — OneTab is one of my most-used extensions.

*P.S. This report was written with the help of machine translation, so the wording may be slightly off. The essentials are: huge freeze when restoring a tab from a big list, root cause is full rendering of all items, and a one-line CSS fix works. I'm happy to clarify anything.*

---

P.S. 表单是官方唯一反馈入口（无公开 GitHub 仓库），开发者会看并回复（商店评论区常见官方回复）。提交后 1–2 周若无回应，可在 Edge/Chrome 商店 OneTab 评论区补一句"几千条收藏时点击恢复卡顿十几秒，已走 one-tab.com/feedback 附详细数据"，提高触达概率。