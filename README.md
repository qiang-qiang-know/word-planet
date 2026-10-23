# 单词星球 Word Planet

给 6–7 岁中国孩子做的英语启蒙小游戏：**答对一个单词，星球就亮起来一点。**

![预览图](preview.png)

打开 https://qiang-qiang-know.github.io/word-planet/ 直接玩（手机竖屏最佳，桌面浏览器也可以）。

## 跑起来

```bash
node build.mjs            # 源码 → dist/index.html（单文件，可双击打开）
node serve.mjs            # http://127.0.0.1:4173
node tests/all.mjs        # 29 个测试
```

`dist/index.html` 把 JS、CSS、字体（base64）全部内联进一个 HTML，**289 KB**，
没有任何外部请求，可以直接扔到任何静态托管（GitHub Pages / Vercel / OSS / 微信里直接发文件）。

## 玩法

- 4 座岛 × 3 关：听音寻宝（听发音选图）→ 认词大关（看图选词）→ 星球挑战（混合题型，题量更多、金币翻倍，怪物戴皇冠）
- 32 个核心单词，全部带 emoji 图形，**零阅读门槛**：指令全靠图标 + 语音
- 答错不惩罚：不扣血、不重来，只把正确答案点亮并读一遍
- 每座岛自己会亮：星数攒满，岛的颜色从灰暗变饱和
- 金币攒着给吉祥物哔哔换帽子；还有收藏册

### 四条设计红线

1. **不写字当指令**——6 岁孩子不认字，按钮要么是图标，要么是 word + 图。
2. **答错不惩罚**——不扣血扣分，只"点亮正确答案"，让孩子自己发现。
3. **一次只给一个决定**——每屏只有一个显而易见的下一步。
4. **所有进度看得见**——星星、金币、岛的亮度，即时反馈。

## 声音

**一个音频文件都没有**，全部实时合成。换来的是 0 KB 体积、0 授权风险、离线可用。

- 音效走 WebAudio：`tap` 点击、`whoosh` 进关、`hit` 命中、`correct` 上行琶音、
  `coin` 金币、`wrong` 下行滑音、`star` 结算逐颗、`fanfare` 三星。
  每个音都带 ADSR 包络（`exponentialRampToValueAtTime` 从 0.0001 起），不会爆音。
- 答对是**三层叠加**：低鼓 + 四音琶音 + 130ms 后的金币，单层听起来"飘"。
- 发音走系统 TTS：`rate 0.78`、`pitch 1.12`（放慢、略提高，6–7 岁听得清），
  优先挑 `Samantha / Zira / Google US English` 这类英文女声。
- 静音开关在 HUD 右上角，**写进存档**（`profile.muted`），下次打开还记得。
  静音时音效连节点都不创建，`speak()` 仍会调用 `onend`，不会有流程卡在等朗读结束。
- 每次 `speak()` 带 **1200ms 看门狗**：`onstart` 没来就认为这台设备念不出声。

⚠️ iOS Safari 要求音频必须在用户手势里启动，所以 `initAudio()` 挂在
`pointerdown` 和「开始冒险」按钮上；不要改成页面加载时就 `new AudioContext()`。

## 微信内置浏览器（重要）

微信里打开这份游戏，**声音是最容易出问题的一环**，实测两种病：

- 安卓 X5 内核：`window.speechSynthesis` 直接不存在，`speak()` 静默失败。
- iOS WKWebView：对象存在、`speak()` 也不报错，但**第一次朗读不在用户手势里就永久静音**
  （页面加载后 `setTimeout` 触发的朗读不算手势）。

所以做了三件事，改这块前请连着看：

1. **手势解锁**：`unlockAudio()` 在 `pointerdown` / 「开始冒险」被点时执行——`AudioContext.resume()`
   播一帧静音、再念一句空 utterance（`volume 0`）把语音队列点燃。之后的对局内朗读才有效。
2. **1200ms 看门狗 + 整关降级**：`speak()` 的 `onstart` 没来就判定这设备念不出声，
   本关听音题即时换成看图选词（`speakSafeDef()` 把 `listen-pick` 映射成 `pick-word`，
   对局中途失效也能就地换题），并弹一条 toast 告诉家长。宁可换题型，也不要让孩子
   对着一张点不响的喇叭干瞪眼。
3. **大喇叭本身可重播**：`.speaker` 绑了 `onclick`——家长把手机给孩子之前先点一下，
   这一下同时也是手势解锁。

`speechUnavailable()` 是唯一判据：`!hasVoice() || speechBroken`。
`speechBroken` 只在**手势解锁之后**才可能置真（没解锁时不响是正常的，不能记成设备坏）。

`shots/14-nospeech.png` 就是降级后的样子：听音题变成「看图 + 4 个单词」，没有大喇叭。

第一次朗读的看门狗放宽到 **2.5 秒**（后面每次 1.2 秒）：安卓冷启动 TTS 引擎要 1.5–2 秒很常见，
用 1.2 秒会把"其实能念"的设备误判成哑巴，然后整关听音题被降级。

## 深色模式（必读）

**症状**：在小米手机自带浏览器 / 开了「深色模式」的浏览器里打开，整屏变成黑底、奶白卡片全黑、
深色文字变白——看起来像样式全丢了。

**原因**：这些浏览器会对**没有声明 `color-scheme`** 的页面做"强制反色"（force dark）：
把浅色背景压暗、深色文字提亮。本作本来就是深色背景 + 奶白卡片，正好被算法当成"浅色页面"翻掉。

**修复**：同时声明

```html
<meta name="color-scheme" content="dark" />
```

```css
:root { color-scheme: dark; }
```

声明之后浏览器认为"这页已经是深色了"，不再自动反色。`tests/verify-build.mjs` 会校验这两处都在，
删掉会直接构建失败。

**怎么自己复现/验收**：

```bash
# 复现坏样子（强制反色 + 强行关掉声明）
harness.html?to=battle-word&seed=progress&light=1
# 修复后的样子（同样的强制反色，但声明生效）
harness.html?to=battle-word&seed=progress
```

两屏都加 `--force-dark-mode --enable-features=WebContentsForceDark` 用 Edge headless 截。
对照图：`shots/17-dark-broken.png`（坏）与 `shots/18-dark-fixed.png`（好）。

## 测试

```bash
node tests/all.mjs            # 29 个：数据 8 + 存档 13 + 微信降级 2 + 全流程/商店/静音 3 + 音频 3
node tests/verify-build.mjs   # 校验单文件构建产物完整性
```

测试不依赖 jsdom：`tests/dom-harness.mjs` 是一个约 280 行的极简 DOM 垫片，
只实现本项目用到的那部分 API（含冒泡事件与 `querySelector`）。端到端测试会真的
点开场 → 选岛 → 打一关 → 看结算 → 进收藏册，断言星数、金币、存档落盘全部正确。

音频测试用假的 `AudioContext` / `speechSynthesis` 记录调用，断言"该响的时候真响了、
静音之后一个节点都不建、onend 照常回调"——没有声卡也能跑。

⚠️ **测试顺序不能随便改**（`tests/all.mjs` 里有注释）：每组的垫片会替换
`globalThis.window`，谁先装谁生效。`fixes.test.js` 排在烟测之前，因为它要一份
"没有语音引擎"的干净环境；`audio.test.js` 排最后，并且用带 query 的动态 import
拿一份全新的 audio 模块实例，避免和前面的测试共享状态。

## 视觉验收（真浏览器截图）

不是"我觉得应该没问题"，是真的开浏览器截图看。`.shots/` 是一套零依赖截图台：

```bash
powershell -File .shots/shoot.ps1                 # 出 12 张 iPhone 画框截图
powershell -File .shots/shoot.ps1 -Only '0[567]'  # 只重出某几张
```

- `.shots/harness.html?to=<屏幕>&seed=<fresh|progress>` 用真实点击走位：开场 → 选岛 →
  进关 → 答题，然后停在目标屏。`nospeech=1` 模拟微信里念不出声。
- 截图用 Edge headless，`--force-device-scale-factor=2` + `#app` 钉成 430×932，
  `.shots/crop.py` 裁成 860×1864 的精确手机画框。
- harness 会在截图前注入 `animation:none;transition:none`。**这一步不能省**：
  带 `slideUp` 的弹出层第一次截出来是半透明的，看图会误判成"卡片太透"。
- `?measure=1` 会把各元素的 `getBoundingClientRect` 与关键 `getComputedStyle` 写进
  `<pre id="M">`，配合 `--dump-dom` + `node .shots/extract.mjs <dir> <名字>` 取回。

## 技术选择

| 选择 | 为什么 |
| --- | --- |
| 单文件 `index.html` | 发微信、发网盘、双击都能跑，不依赖服务器和网络 |
| 无框架，原生 ES Modules | 总共 52 KB JS。引入 React/Vue 反而让包体积和心智负担都翻倍 |
| 自己写的极简 DOM 垫片 | jsdom 装不上（离线/体积），而项目用到的 DOM API 不到 20 个 |
| emoji 当图形 | 32 个词零素材成本、跨平台一致、天然"贴纸感" |
| Web Audio 合成音效 | 0 KB、可参数化、音量包络可控，不存在版权问题 |
| 系统 TTS 读单词 | 真人录音要 32 条且不好换词；TTS 换成人口音也能听 |

## 目录

```
src/
  core/      数据与规则（words / quiz 出题 / progress 存档 / audio 音频）
  ui/        app.js 渲染与交互，styles.css 全部样式
tests/       node --test 可直接跑的测试 + dom-harness.mjs 垫片
.shots/      截图台（harness.html / shoot.ps1 / crop.py / extract.mjs）
dist/        构建产物（单文件）
shots/       已截好的验收图
```

## 尺寸适配

`.app` 是 `min(100vw, 460px) × min(100dvh, 950px)`，JS 算 `--u = 宽 / 430` 写进
`:root`，所有尺寸用 `em` 表达（1em ≈ 一个基准单位），因此任何手机宽度都等比缩放，
不会出现"选项溢出、按钮被切"。

调布局时先跑 `?measure=1` 量真实几何再改，别靠眼睛猜。

## 想加新词

改 `src/core/words.js` 里对应岛的词表（`en` / `zh` / `emoji`），跑 `node tests/all.mjs`。
测试会自己覆盖新数据（题数、选项去重、星级、金币都是按数据算的）。
