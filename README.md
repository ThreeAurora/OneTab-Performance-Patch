# OneTab 不卡补丁（OneTab Performance Patch / 「魔改版」）

OneTab 在保存了几千个标签页后，**点击条目恢复到浏览器时，整个页面会卡顿十几秒动不了，滚动也会明显掉帧**。这是一个**已验证、一行 CSS 就能极大缓解**的补丁配方库。

**本项目不含 OneTab 的任何代码。** OneTab 是闭源商业扩展（版权归其开发者所有）。这里发布的全部是我们自己的原创内容：补丁、构建脚本、数据迁移工具和诊断报告。你按下面步骤从商店版自己组装出「魔改版」，效果如下：

在 4396 条收藏的数据集上：
| 场景 | 原生版 | 补丁后 |
|---|---|---|
| **点击恢复标签页** | 卡顿十几秒，页面完全不响应 | 秒开无感觉，几乎不卡 |
| 鼠标滚动命中测试 | HitTest p50 12.25 ms | 6.86 ms（-44%），144Hz 屏不掉帧 |

（只需要追加这一行 CSS，不碰任何 JS 代码：）
```css
.tab { content-visibility: auto; contain-intrinsic-size: auto 26px; }
```

完整证据链（真实 trace 分析、对照实验、排除项）见 [docs/findings.md](docs/findings.md)。

## 为什么会有这个补丁

OneTab 的列表页把**所有条目一次性渲染进 DOM**（约 4.2 万个节点），没有虚拟滚动、没有 `IntersectionObserver`、没有分批渲染。条目多时浏览器每次鼠标命中测试都要对整个列表算一遍——卡顿根因就在这。给屏幕外条目加上 `content-visibility: auto` 之后，它们会整体跳过布局和命中测试：

```css
.tab { content-visibility: auto; contain-intrinsic-size: auto 26px; }
```

（`contain-intrinsic-size` 让浏览器知道屏幕外条目占多高，滚动条位置和拖拽落点不会飘。）

## 为什么不能直接改商店版

Edge/Chrome 会校验商店扩展的文件完整性——直接改安装目录里的文件，扩展页会提示「此扩展可能已损坏」并拒绝启用。所以正确做法是：把商店版**整个复制一份**作为本地扩展加载（开发者模式），在副本上打补丁。开发者模式加载的本地扩展不做完整性校验。

## 快速开始（Edge / Chrome）

> 全程不动商店版，两个版本可并存。

### 方式 A：一键构建（推荐）

```powershell
# 1. 找到商店版目录，例如 Edge：
#    C:\Users\<你>\AppData\Local\Microsoft\Edge\User Data\Default\Extensions\hoimpam…\<版本号>_0
#    Chrome 则在其 User Data\Default\Extensions 下
# 2. 构建魔改版（自动复制 + 改 manifest + 打补丁）：
.\build_modded.ps1 -SourceDir "C:\…\2.18_0" -TargetDir "…\OneTab-Modded"
# 3. 打开 edge://extensions（或 chrome://extensions）→ 打开「开发人员模式」
#    →「加载解压缩的扩展」→ 选择 TargetDir
```

脚本会自动做三件事：
1. 复制商店版目录为副本；
2. 修改副本的 `manifest.json`：去掉商店 `key`（扩展 ID 由路径生成，与商店版错开，可并存）、去掉 `update_url`、显示名为「OneTab 魔改版」、快捷键改为 `Alt+Shift+2`（避开商店版的 `Alt+Shift+1`）；
3. 把上面的补丁追加到副本 `onetab.css`（幂等，重复运行不会重复打）。

### 方式 B：纯手动

1. 从商店安装 OneTab，找到它的安装目录，整体复制一份；
2. 编辑副本 `manifest.json`：删掉 `"key"` 和 `"update_url"` 两行，把名字改成你能认出来的（如 `OneTab 魔改版`），版本号 +1；
3. 把 `onetab.patch.css` 的内容追加到副本的 `onetab.css` 末尾；
4. 同方式 A 第 3 步加载副本。

## 数据迁移（只在你需要在新版里看到原数据时做）

魔改版 ID 与商店版不同，浏览器按扩展 ID 隔离数据，所以旧数据不会自动出现。两步搬过去：

1. 打开**商店版** OneTab 页面 → F12 → Console → 粘贴运行 `tools/dump_onetab_items.js` → 下载 `onetab-raw-backup-<时间>.json`；
2. 把该 JSON 放进魔改版目录，改名 `onetab-raw-backup.json`，打开**魔改版** OneTab 页面 → F12 → Console → 粘贴运行 `tools/backfill_onetab_items.js` → 页面刷新后数据全部恢复。

（本质是原样导出 / 写回 IndexedDB 的 `item`/`attr`/`shareUpdate` 三个库，tab、分组、回收站、任务都不丢。）

## 回退

删除魔改版扩展即可，商店版全程没被动过。官方更新到新版后想跟进：把新版商店目录再跑一遍 `build_modded.ps1`（脚本幂等，会覆盖副本并重新打补丁）。

## 已知注意事项

- `content-visibility: auto` 会使屏幕外条目跳过布局——以及**跳过屏幕外目标的命中测试**，对本场景（拖动到可见区域）实测无影响；若官方实现虚拟滚动，此方案可整体退役。
- 建议顺手清空回收站：`findings.md` 显示 4396 条里 553 条在回收站，白付渲染成本。
- 配合 `docs/findings.md` 里的建议 B/C（清 trash、归档旧分组）可进一步压降。

## 许可证

MIT，见 [LICENSE](LICENSE)。OneTab 商标与代码版权归其开发者所有，本项目仅作描述性引用。

---

## English TL;DR

OneTab renders every saved tab into the DOM upfront (no virtualization). With thousands of items, **clicking a tab to restore it freezes the whole page for ~10+ seconds**, and scrolling/hovering also drops frames (mouse hit-testing: p50 ≈ 13 ms at 4,396 items vs a 6.9 ms frame budget on 144 Hz). One CSS rule fixes most of it:

```css
.tab { content-visibility: auto; contain-intrinsic-size: auto 26px; }
```

On our 4,396-item dataset: **restoring a tab went from ~10+ s of frozen UI to near-instant**; hit-test cost drops 12.25 ms → 6.86 ms (−44%), no JS changes. Because browsers verify store-extension file integrity, you can't just edit the installed copy (it gets flagged "damaged"); instead copy the extension folder, patch the copy (`build_modded.ps1` does this automatically: copy + strip `key`/`update_url` + rename + apply patch), and load it via `chrome://extensions` → Developer mode → Load unpacked. The patch & tools are MIT; OneTab's own code and branding belong to its developers.