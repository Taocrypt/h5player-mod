[音视频增强脚本（Taocrypt魔改版）：无极调速|长按倍速|快乐刷剧|视频下载|画面截图等「适用大部分网站」](https://www.anygen.io/share/491LVKlFFDhNVCxttkJmtV)

## 摘要

本文系统梳理我对 h5player 脚本的“魔改”过程，重点在于将移动端“长按倍速”手势与 h5player 原有倍速逻辑进行深度桥接与共存。改造不仅保持了原脚本的全部功能，还在 DPlayer 等自定义播放器场景下通过全局捕获层实现了手势兼容，最终达成“在绝大多数网站上稳定可用”的目标。

## 背景与原脚本概览

h5player 是一个针对 H5 音视频网站的增强脚本，提供倍速控制、截图、画中画、网页全屏、画质调节、下载能力等丰富功能，覆盖 B 站、抖音、优酷、爱奇艺、YouTube、知乎、微博以及各类课程平台与网盘站点。其内部对 `HTMLMediaElement` 属性（如 `playbackRate`、`volume`、`currentTime`）进行了劫持与锁机制设计，以增强抗干扰能力，避免站点自行重置速度或音量。

与此同时，“手机长按倍速”这一轻量手势逻辑在移动端场景非常自然：长按视频左半区以 1.0× 为基速，右半区以 2.0× 为基速；长按期间上滑加速、下滑减速，松手恢复原速。该脚本完全基于原生 API 编写，递归支持 Shadow DOM 与同源 iframe，并提供轻提示（右上角半透明倍速）与轻微振动反馈，体验上更贴合触屏设备。

## 改造动机与目标

在实际使用中，移动端手势与 h5player 的倍速管理可能产生冲突：站点脚本或 h5player 内部锁机制会在某些时刻“回写”速度，使得长按调速不生效。改造目标如下：

- 保留 h5player 原有全部功能与配置能力；
- 将“长按倍速”完整注入并与 h5player 的倍速 API 协同，确保设置生效且不被轻易回写；
- 在 DPlayer 等自定义覆盖控件场景下，确保手势事件能够被捕获并映射到真实的 `video` 元素；
- 跨 Shadow DOM 与同源 iframe 深度扫描，降低漏绑概率。

## 改造过程详解

### 1. 合并策略与元数据处理

将“长按倍速”脚本主体（IIFE 部分）追加到 h5player 代码末尾，避免多重 `// ==UserScript==` 头部冲突；保留并更新 @name/@description 多语言元数据，说明“Taocrypt 魔改版”的新增能力与兼容性；清理所有无意义的站点页面残留文本，使脚本自 `// ==UserScript==` 起始。

### 2. 速率设置的协同与桥接

改造核心在于 **兼容层**：

```js
function setPlaybackRateCompat(video, rate) {
  // 优先获取 h5player 实例（window._h5Player 或常量 h5Player）
  const t = (window._h5Player && typeof window._h5Player.setPlaybackRate === 'function')
            ? window._h5Player
            : (typeof h5Player !== 'undefined' ? h5Player : null);
  if (t && typeof t.setPlaybackRate === 'function') {
    // 将当前触发的 video 作为活动实例，并初始化其代理与锁
    if (video && t.playerInstance !== video) {
      t.playerInstance = video;
      try { t.initPlayerInstance(false); } catch (e) {}
    }
    // 协同设置倍速：先解锁、再设置、再短锁，防止站点回写
    try { t.unLockPlaybackRate(); } catch (e) {}
    t.setPlaybackRate(rate, true);
    try { t.lockPlaybackRate(800); } catch (e) {}
    return;
  }
  // 无 h5player 时的事件协商与兜底
  try {
    const evt = new CustomEvent('h5player:requestSetPlaybackRate', { detail: { video, rate } });
    document.dispatchEvent(evt);
  } catch (e) {}
  try {
    video.playbackRate = rate;
    try { video.dispatchEvent(new Event('ratechange')); } catch (e) {}
  } catch (err) {
    const desc = Object.getOwnPropertyDescriptor(HTMLMediaElement.prototype, 'playbackRate');
    if (desc && typeof desc.set === 'function') {
      desc.set.call(video, rate);
      try { video.dispatchEvent(new Event('ratechange')); } catch (e) {}
    }
  }
}
```

该桥接使长按倍速直接走 h5player 的源逻辑，兼具解锁/短锁节奏，显著提高“设定后不被回写”的稳定性。

### 3. 事件捕获优先级与传播策略

为减少与站点自定义控件的冲突，手势监听改为在捕获阶段执行：

- `touchstart/touchmove/touchend` 采用 `{ passive: false, capture: true }`；
- 将原本的 `stopImmediatePropagation()` 调整为更温和的 `stopPropagation()`，降低与宿主脚本的事件冲突概率；
- 在滑动阶段主动 `preventDefault()`，提升在覆盖层上的手势生效率。

### 4. DPlayer 等自定义播放器的兼容

面对 DPlayer 这类“控件层覆盖 video”的场景，新增 **全局捕获层**：

- 在 `document` 捕获阶段监听触摸事件；
- 通过点击坐标递归查找命中区域内的 `video`（含 Shadow DOM）；
- 命中后复用同一套长按逻辑，并通过兼容层桥接到 h5player 源倍速；
- 该层仅在命中 `video` 时工作，避免无谓干扰。

这一策略显著提升了在自定义播放器上的手势触发成功率，确保“长按倍速”在更多实际站点生效。

## 新增功能特性总览

下表总结了本次改造后的核心特性与设计要点。

| 特性 | 设计要点 | 预期效果 |
|---|---|---|
| 源逻辑桥接 | 长按倍速走 h5player `setPlaybackRate/lockPlaybackRate`，含解锁/短锁节奏 | 设置更稳，不易被站点脚本改回 |
| 全局捕获层 | `document` 捕获 + 坐标命中 `video`，递归 Shadow DOM | DPlayer 等控件覆盖场景下依旧可触发长按 |
| 事件传播策略 | 捕获阶段监听、滑动阶段 `preventDefault()` + `stopPropagation()` | 减少冲突，提高手势有效性 |
| 深度扫描 | 初始高频扫描 + `MutationObserver` 持续监听 + iframe 递归 | 降低漏绑概率，适配动态加载 |
| 轻提示与振动 | 右上角倍速提示 + `navigator.vibrate`（设备支持） | 反馈清晰，手感良好 |
| 多语言元数据 | 更新 @name/@description（zh/zh-TW/ja/ko/ru/de/en） | 便于国际用户识别改造版 |
| 无冗余文本 | 清理脚本头部无意义页面残留 | 文件更干净，可直接安装 |

## 安装与使用指南

安装建议采用“单脚本模式”：在脚本管理器（Tampermonkey / Violentmonkey）中新建脚本，将合并版代码整体粘贴并保存。为避免双重绑定导致提示重影或倍速冲突，**请禁用或删除独立的“手机长按倍速”旧脚本**，仅保留合并版。

移动端手势操作说明如下：

- 长按视频左半区：以 **1.0×** 为基速进入长按模式；
- 长按视频右半区：以 **2.0×** 为基速进入长按模式；
- 长按期间上滑：提高倍速（最高 **16×**）；
- 长按期间下滑：降低倍速（最低 **0.25×**）；
- 松手：恢复至长按前的原始倍速。

## 兼容性与适配建议

在绝大多数网站场景下，合并版脚本能稳定工作。但若个别站点对触摸事件做了强拦截或采用复杂的播放器封装（自定义组件、闭合 Shadow DOM、跨域 iframe 等），仍可能需要额外适配：

- 若出现“长按无效”，可先确认是否命中到主 `video`（含全屏/网页全屏时的容器变化）；
- 个别站点可能在极短时间内通过自身脚本改回倍速，可适当提高短锁时间（例如 1200ms）；
- 遇到跨域 iframe 无法访问 DOM 的情况，逻辑会自动忽略该容器，不影响主页面运行。

## 常见问题（FAQ）

**Q1：桌面端是否需要长按倍速？**  
A：桌面端通常使用快捷键与菜单进行倍速控制，长按倍速主要面向移动端触屏场景。两者可在同一脚本中共存。

**Q2：是否会影响 h5player 的其它功能？**  
A：不会。改造严格遵循“保留原功能”的原则，未删减任何原有能力。必要时通过桥接与短锁保障倍速生效。

**Q3：为何新增了全局捕获层？**  
A：为解决自定义控件覆盖 `video` 的场景（如 DPlayer）。全局层在捕获阶段工作且仅在命中到 `video` 时启用，不会影响普通页面行为。

## 版本信息与致谢

- 作者与改造：**Taocrypt（mod）**；原作者：**ankvps**  
- 许可协议：**GPL**  
- 修改日期：**2026-01-03**  
- 主要改动：
  - 注入手机长按倍速手势，桥接至 h5player 源倍速；
  - 新增全局捕获层，适配 DPlayer 等控件覆盖；
  - 深度扫描与事件策略优化；
  - 多语言元数据更新与无意义文本清理。

## 结语

通过本次改造，移动端的“长按倍速”手势与 h5player 的倍速生态得以真正融合，既保留原有强大的增强能力，又赋予触屏设备更自然高效的操作方式。欢迎在更多站点上试用，并反馈兼容问题与改进建议，以进一步完善整体体验。

该脚本使用Anygen AI经过多轮对话修改完成，当前版本经测试已完美兼容源脚本功能与长按倍速功能，对我来说使用体验已经接近完美，如果觉得对你有帮助或者对此Agent感兴趣可以走我的AFF链接注册或者为我点个赞支持一下：https://www.anygen.io/home?invitation_code=CXQW7U02BIDP3O4
