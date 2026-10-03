---
title: osu live2d行为界面预览区白屏问题排查
published: 2026-10-03
pinned: false
description: 记录和解决在调试WebView2作为live2D的预览区时，遇到了加载过程出现的白屏问题
tags: [Bug修复, osu-live2d-overlay]
image: https://cloudflare_imgbed.2405969191.workers.dev/file/1791028512834_264a7462-85b0-4766-b5fa-3815139be4b9.png
slug: olobug-fix
---

## 问题描述

在使用WebView2作为osu! live2D行为界面的预览区时，`PreviewPane.xaml`中的Webview2的`Visibility `属性设置为**Visibility.Collapsed**自认为涵盖了加载过程，实际上在加载过程中，WebView2的内容并没有被正确渲染，导致出现白屏现象。

在这个过程中，我们曾认为**Visibility.Collapsed**属性是无效的，还尝试将WenView2的透明度改成100，都无法解决问题，最终我们确认问题来自两个方面

## 问题原因

**① 那块表面本来就有颜色，而它的默认值是纯白。**

WebView2 是一块**原生窗口**（HwndHost），它内部"网页表面"不是透明的，有一层自己的底色，属性名叫 `DefaultBackgroundColor`。它的默认值是**白色**。所以从"控件被放上屏幕"到"页面画出第一笔"之间，那块区域就是纯白 —— 跟 WPF 这边的主题完全无关（试过把窗口、边框、页面背景全改成透明或深色，一点用都没有）。简单来说，就算你将背景设置为透明，WebView2 也会在加载页面之前显示它的默认背景色。

**② 页面自称"画好了"的两条信号，都比"真的画出来"早。**

| 信号 | 原来怎么发的 | 实际含义 | 问题 |
|---|---|---|---|
| `ready` | `web\index.html:570` | 模型 / 贴图 / 动作**都读进内存了** | 不代表画布上有东西 |
| `painted` | 改成修法之前的"双 `requestAnimationFrame`" | "浏览器跑过了一帧" | **也不代表画布上有东西** |

## 最终结论

那么浏览器自认为加载完毕到第一帧画出来前，WebView2到底花了多少时间？我们在 `ready`开始的时候启动探针，每**300ms**对这个画布做一次采样，结论如下

- `ready` 那一刻，画布上非透明像素 **0 / 32044716**；
- 第一帧渲染器 tick 在 `ready+1 ms` 就跑了，但 `ready+300 ms` 时像素**仍然是 0**；
- 直到 `ready+2000 ms` 才有像素（4798 个采样点，集中在 `(44,22)→(188,322)`，约占画布 3%×4%，此时角色出现了）。

也就是说：**模型加载没问题，是"第一次真画"本身要花一秒多**（4 张 4096×4096 贴图上传 + 第一次着色器编译）。宿主却在那之前就露面了，用户看到的只有那块纯白（或者页面还没被覆盖掉的 Chromium 错误页）。

修法：**别在"页面说好了"的时候露面，在"画布上真的有像素"的时候露面。**

## 完整问题解决过程

### 先怀疑"起浏览器太慢"

第一版日志里量的是这几个点（`Infrastructure\DebugLog.cs` 写文件，路径 `<程序目录>\logs\年-月\年-月-日.log`）：

```
[18:16:37.761] 预览区#1：控件已创建
[18:16:38.820] 预览区#1：开始导航 → https://app.local/index.html      ← 花了 1059 ms
[18:16:38.910] 预览区#1：导航完成 成功=True 状态=Unknown
[18:16:38.938] 预览区#1：页面就绪                                     ← 导航只用 118 ms
[18:16:39.237] 预览区#1 [页面] {"type":"ready"}                       ← 再 300 ms
```

统计多天日志：「控件已创建 → 开始导航」200–1100 ms（多数 500–800），而「开始导航 → 页面就绪」只有 60–150 ms。

结论：起一只 WebView2确实要接近一秒，但那一秒里屏幕上**根本不是白的**，因为当时控件还是 `Collapsed`，能看见控件背后的提示文字，在这里我们确认**Visibility.Collapsed**的有效性。但显然问题到这里还没有结束

### 开始抓取这个时间段的WebView2到底展示了什么

靠读代码和肉眼观察得不出准确的结论，所以做了两件事：

1. **宿主侧**：在几个时间点调 `CoreWebView2.CapturePreviewAsync` 把预览区抓成 PNG，存到日志目录；
2. **页面侧**：往画布上做像素采样（`app.renderer.extract.pixels()`），在 `ready` / `ready+300ms` / `ready+2000ms` 三个时刻记录"非透明像素有多少"。

这一轮抓出来的图：

| 文件名 | 相对"导航开始" | 屏幕上实际是什么 |
|---|---|---|
| `预览区#1-导航开始-1452ms.png` | +0.00 s | 纯主题底色，此时控件隐藏 |
| `预览区#1-导航完成-1303ms.png` | +0.34 s | **白底，左上角一个简笔哭脸**，问题从这开始 |
| `预览区#1-收到painted-1593ms.png` | +0.63 s | 同上，**字节数完全相同** |
| `预览区#1-导航完成+700ms-1743ms.png` | +0.78 s | 同上，**字节数完全相同** |
| `预览区#1-导航完成+1400ms-3261ms.png` | +2.30 s | 角色已经画好了（77946 字节） |
| `预览区#1-导航完成+2000ms-3285ms.png` | +2.32 s | 同上 |

**四张白底哭脸的 PNG 字节数一模一样**（2548），说明它们抓的是同一帧画面 —— 那张哭脸是 **Chromium 自己的错误页**，不是我们画的，也不是"没加载出来"。

同时，日志里那一行是：

```
[19:45:15.746] 预览区#1：导航完成 成功=True 状态=Unknown 实际地址=https://app.local/index.html
[19:45:15.782] 预览区#1：页面就绪
[19:45:15.792] 预览区#1 [页面] {"type":"log","text":"语音未启用"}
[19:45:16.084] 预览区#1 [页面] {"type":"ready"}
```

**"导航成功" 和 "屏幕上是一张错误页" 可以同时成立。** 这是这一轮最让人惊讶的发现，也是后来专门加了"地址变化"留痕的原因（见第四节）。

### 找到问题，开始解决

#### 不再为WebView2设置透明背景，而是读取此时背景色达到伪透明效果

`Views\Shared\PreviewPane.xaml.cs` 的 `StartAsync()`，在**拉起浏览器之前**：

```csharp frame="code" title="Views\Shared\PreviewPane.xaml.cs" showLineNumbers startLineNumber=241
//  2026-10-01：把"网页那块表面"的底色设成 当前主题的窗口底色 。
//
// 【为什么要设】WebView2 里那块网页表面 不是透明的 ，它自己带一层底色
//（属性名就叫 `DefaultBackgroundColor`）：
//   · 不设   → 它是**纯白**（页面还没画出来时，面板上就是一块白）
//   · 设成透明 → 在 WPF 窗口这个合成环境里会变成 一块黑
// 两者都跟窗口底色不一样，所以中间那一下会很突兀。
Web.DefaultBackgroundColor = ThemeSurfaceColor();
```

```csharp frame="code" title="Views\Shared\PreviewPane.xaml.cs" showLineNumbers startLineNumber=370
private System.Drawing.Color ThemeSurfaceColor()
{
    try
    {
        if (TryFindResource("BackgroundColor") is Color c)      // Token 里的窗口底色
            return System.Drawing.Color.FromArgb(255, c.R, c.G, c.B);
    }
    catch { /* 资源还没就绪，走下面的兜底 */ }

    return System.Drawing.Color.FromArgb(255, 0x1E, 0x22, 0x28); // 深色底兜底
}
```
:::tip
- **必须在导航之前设**。这个属性只在"控制器刚建好"那一刻读一次，导航开始之后再改不生效。
- **取的是 Token**（`Resources\Styles\Tokens.Dark.xaml` 的 `BackgroundColor = #FF1E2228`、`Tokens.Light.xaml` 的 `#FFF4F6F9`），所以浅色主题下也是对的。
:::

> 例外：**悬浮窗**那边照样把 `DefaultBackgroundColor` 设成真透明（`Views\Overlay\OverlayWindow.xaml.cs:349`）。悬浮窗本身就是一个透明窗口，整个窗口的合成方式跟预览区不一样，那边透明是成立的。
>
> 另：不管设成什么，**从拉起浏览器到首帧之前，这块表面最好一直藏着**，于是颜色这件事正常情况下根本轮不到用户看见，它只是"万一页面彻底坏了、兜底放行"时的保险。

#### `painted` 改成"画布上真的有像素"

`web\index.html`，`loadModel` 末尾，`post({ type: "ready" })` 之后：

```js
// 2026-10-01：`ready` 只代表"模型/贴图/动作都读进来了"，**不代表画出来了**。
//
// 抓图实测：
//   ① `ready` 那一刻画布上"非透明像素 0"，模型还没画；
//   ② 第一帧渲染器 tick 在 ready+1ms 就跑了，但 ready+300ms 时像素**仍然是 0**；
//   ③ 页面原来发的 `painted` 在 ready+16ms —— **是假的**，那时屏幕上还是 Chromium 的错误页。
// 所以"第一帧"必须按"画布上真的有东西"来判，而不是按"跑了几帧"。
let __t0 = performance.now();
let __reported = false;   // 真正的 `painted` 只发一次
let __frames = 0;
let __hasPixels = false;

app.ticker.add(() => { __frames++; });

// 全画布读回太慢，96×96 采样点足够判定"画出来了"
function __sampleHasPixels() {
  try {
    if (!app.renderer.extract) return false;
    const px = app.renderer.extract.pixels(model);
    const m = modelSize();
    const W = Math.max(1, Math.round(m.w)), H = Math.max(1, Math.round(m.h));
    const stepX = Math.max(1, Math.floor(W / 96));
    const stepY = Math.max(1, Math.floor(H / 96));
    for (let y = 0; y < H; y += stepY) {
      for (let x = 0; x < W; x += stepX) {
        if (px[(y * W + x) * 4 + 3] > 0) return true;   // 看 alpha 通道
      }
    }
  } catch (e) { /* 取不到就当还没画出来 */ }
  return false;
}
```
三条放行路径：

```js
// 路径 A（首选）：页面在正常产帧。ticker 回调跑在渲染之前，
//   所以"这一帧跑过了"其实还看不到东西 —— 要等 `__frames > 0` 时读回来的才是上一帧。
//   每次判据是一次像素读回（不便宜），所以每 3 帧才探一次。
app.ticker.add(function __firstFrame() {
  if (__reported || ++__ticks > 480) { app.ticker.remove(__firstFrame); return; }
  if (__frames === 0) return;
  if (__ticks % 3 !== 0) return;
  if (!__sampleHasPixels()) return;
  __hasPixels = true;
  app.ticker.remove(__firstFrame);
  requestAnimationFrame(() => __reportPainted("画布上已经有像素"));
});

// 路径 B：轮询兜底 —— tick 里判一次不一定赶上，每 150 ms 再确认一次，最多 4 秒。
(function __pollPainted() { /* … __sampleHasPixels() → __reportPainted("轮询发现画布上有像素") … */ })();

// 路径 C：**页面在控件收起时被冻住、一帧都不跑** —— 这时"等首帧"会变成死锁
//   （页面等露面、宿主等首帧），所以由页面在 2.5 秒时先放行一条，
//   让宿主把 WebView2 露出来，露出来之后页面就会开始跑帧、接着画。
setTimeout(() => {
  if (__reported || __frames > 0) return;
  log("第一帧判定（页面一直没跑帧，先放行让宿主露面）：ready+" + …);
  post({ type: "painted" });
}, 2500);
```
#### 最后，宿主在收到 `painted` 之后才把 WebView2 的 `Visibility` 改成**Visible**，而不是靠`ready`

```csharp frame="code" title="Views\Shared\PreviewPane.xaml.cs" showLineNumbers startLineNumber=537
//页面第一帧已经画完了，这是"可以露面"的信号
_painted = true;
Dispatcher.Invoke(RevealWeb);
```

```csharp frame="code" title="Views\Shared\PreviewPane.xaml.cs" showLineNumbers startLineNumber=416
//只有这一个地方会把 Web.Visibility 设回 Visible**
private void RevealWeb()
{
    if (!_pageReady || _pageDead) return;

    if (_revealed) return;                  // 幂等：三个调用点谁先到都只做一次
    _revealed = true;

    _revealTimer?.Stop();

    Web.Visibility = Visibility.Visible;

    // 提示行要留到这一刻才收。
    // 以前是在 `导航完成`（文档刚加载完）就收，那时角色还画不出来，
    // 收起之后用户对着的就是一块空的主题底色，不知道在等什么。
    Hint.Visibility = _patchProblem is null ? Visibility.Collapsed : Visibility.Visible;

    DebugLog.Write($"{LogTag}：WebView2 露面（首帧={_painted}，{_since.ElapsedMilliseconds} ms）");
}
```
#### 其他优化

##### **修掉两个观感问题**

- 提示行「预览正在启动…」原来在 `导航完成`（文档刚加载完）就收了，那时角色还画不出来，用户对着的是一块空底色，不知道在等谁。现在它**陪着一直到露面那一刻**才收。
- 「模型补丁有问题：…」这类要长期挂着的话，通过 `_patchProblem` 字段（`:81`、`:227`、`:344`、`:431`）留在提示行上，不会被露面收掉。

##### **留下的一条"防再犯"日志**

`NavigationCompleted` 那行的 `成功=True` 骗过我们一次，所以补了一条：

```csharp
// ★ 2026-10-01：**地址每一次变化都记一行。**
// 落错误页时 core.Source 会变成 chrome-error://chromewebdata/，
// 下次再出这种"成功但哭脸"就能定位。
core.SourceChanged += (_, args) =>
    DebugLog.Write($"{LogTag}：地址变化 新文档={args.IsNewDocument} 地址={core.Source}");
```

（`SourceChanged` 的参数**只有 `IsNewDocument`**，没有 `IsErrorPage` 也没有 `NavigationId` —— 想拿更多信息得去 `NavigationStarting`。）

