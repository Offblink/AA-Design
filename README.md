# AA-Design

从 [AsamiAkito/wisland](https://github.com/AsamiAkito/wisland)（Windows 毛玻璃灵动岛，PySide6，MIT，106★）中提炼的 **UI/UX 设计模式库** —— 21 个独立、自包含的 HTML 演示，每个对应一条可迁移的设计决策。

每个组件是一个单独的 HTML 文件，双击即可在浏览器中运行。无需构建工具、无需 npm install。

与 [Gasp-Design](../Gasp-Design/) 的区别：Gasp-Design 收的是 **GSAP 动画技巧**（怎么动），本库收的是 **产品级界面设计决策**（为什么这样动、这样排、这样配色）—— 每条都附 wisland 源码出处（文件:行号），可直接对照原实现。

## wisland 是什么

贴在屏幕边缘的液态毛玻璃胶囊：收起时是 150×34 的小条（时间天气 / 小形象 / 正在播放，滚轮切换），悬停 220ms 弹性展开成 640px 面板；拖动时变泪滴形、与屏幕边缘之间拉出液桥，扯断后残留水滴缩回；毛玻璃实时取身后画面，主题跟着背景亮度自动切换。

**它的 UI/UX 值得借鉴，因为把「一个常驻悬浮控件」的每个细节都推敲过**：不抢焦点怎么关弹层、内容怎么仲裁优先级、动画怎么省帧、明暗怎么不闪。这些决策与 PySide6 无关，任何前端/桌面框架都能移植。

## 设计原则提炼

### 1. 形态：把「窗口」做成「物质」

wisland 拒绝把悬浮窗当矩形窗口对待，而是赋予它液体物理：

- **表面张力贴边**：胶囊与屏幕边缘不是直角相接，而是用内凹圆角（FILLET=13px）相连，像水滴沾在墙上（`ui/island.py:29` 常量、`ui/shape.py:161 neck_path()`）。
- **液桥拉丝**：拖离时边缘与本体之间维持沙漏形液桥，越拉越细，超过 64px（BREAK）断开；断口留在边缘的半圆水滴带回弹缩回（`ui/shape.py:220 drip_path()`）。靠近 30px（JOIN）内液桥自动重新伸出。
- **速度形变**：拖动时按速度变泪滴 —— 后缘拖尾收窄 38%、前缘鼓起，amt = speed/2600 封顶（`ui/shape.py:108 deform()`）。
- **按压挤压**：鼠标按住时产生跟随光标的高斯凹陷，凹处体积转移到别处（体积守恒 +0.35 补偿），松手欠阻尼回弹外鼓几下（`ui/island.py:1037 _poll_hover()`、k=380 ratio=0.55）。

**可借鉴点**：常驻悬浮控件如果只做「圆角矩形 + 缩放动画」，存在感是弱的。给它一个物理隐喻（液体、磁性、织物），所有交互（拖动、贴边、按压）就都有了统一的视觉语言。见 [liquid-edge-attachment](components/liquid-edge-attachment.html)、[velocity-teardrop](components/velocity-teardrop.html)、[press-squash](components/press-squash.html)。

### 2. 动效：全部阻尼弹簧，不用贝塞尔缓动

整个项目没有一条 `cubic-bezier`。所有形变/位移都是阻尼弹簧（`ui/anim.py:8 Spring`）：

```
c = 2 × ratio × √k        # 阻尼系数
a = -k·(x - target) - c·v # 加速度
```

- **ratio < 1 = 欠阻尼 = 有回弹**，这是 wisland「弹性」手感的唯一来源。各处参数精心调过：导航宽度 k=260/0.56、展开尺寸 k=420/0.62、拖拽位置 k=900/0.78、回吸 k=300/0.38（回弹明显）、按压 k=380/0.55。
- **rest 判定**（|x-target|<eps 且 |v|<eps×8）让定时器自动停帧，静止时零开销。
- **240Hz 子步进**：单帧 dt 可能很大，拆成 ≤1/240s 的小步积分，保证任何帧率下弹簧行为一致（`ui/anim.py:36 step()`）。

**可借鉴点**：弹簧参数即手感调参旋钮 —— 「位置类」用高 k 高 ratio（跟手、微回弹），「尺寸类」中等（明显弹性），「吸附类」低 ratio（大幅回弹）。给每个 UI 属性配一个弹簧，比手写几十条 transition 更一致。见 [spring-playground](components/spring-playground.html)。

### 3. 展开/收起：延迟 + 锁，而不是 hover 立即响应

`_poll_hover()`（`ui/island.py:1037`）的状态机：

- 悬停 **220ms** 才展开（防误触扫过）；移开 **450ms** 才收起（去往面板的路上不闪断）。
- **locks 集合**（pin / focus / menu / lookup）任一在位就不自动收起 —— 输入框有焦点、菜单开着、查词弹窗在，都不会被鼠标移开干掉。
- 单击展开并 pin 600ms（点击后立刻移开也不塌）；**双击切换「长关」**（比图钉更狠的「别烦我」档）。
- 展开时**形状先长到完整尺寸、内容才显示**；收起时内容立即隐藏 —— 全程不需要遮罩，也永远不会有「内容被裁半截」的帧（`ui/island.py:576` 注释）。
- 收起态还有一层「玻璃面纱」`_Veil`：画和本体完全相同的玻璃、不透明度 1→0，保证渐显过程中永远有玻璃底（`ui/island.py:146`，注释明确说 QGraphicsOpacityEffect 会出「有内容没玻璃底」的帧）。

**可借鉴点**：悬浮面板类 UI 的三条铁律 —— **进慢出快有理由**（220/450 不对称）、**状态用锁集合管理**（而不是一堆 if）、**先长形状再填内容**（几何与内容解耦，天然无撕裂）。见 [hover-expand-lock](components/hover-expand-lock.html)。

### 4. 内容仲裁：一个胶囊，五种内容

收起态只有 150×34，却要装时间天气、小形象、歌词、字幕、活动提示。wisland 的解法（`ui/compact.py:407 mode()` 附近）：

- **严格优先级**：临时活动 > 实时字幕 > 正在播放 > 小形象/时间。有人说话时字幕插队，安静 7s 后自动让位（不打断音乐）。
- **尺寸弹簧跟随**：每种 mode 有自己的 size_hint，切换时弹簧变形（idle 150×34 → subtitle 固定 640×60 → media 按歌词宽），0.28s ease_out 交叉淡入。
- **固定宽度防抖**：字幕宽度恒定（说话时绝不伸缩）；歌词宽度按**整首歌最长一句**一次定死，不随每句变（`ui/compact.py set_lyrics()` 注释）。
- **活动提示原地更新**：连续调音量时 `update_activity()` 续期而不是重新入场，不会闪（`ui/compact.py update_activity()`）。
- **时长按内容长度**：翻译结果展示 `3.5 + len×0.12` 秒（读得完才算数，上限 12s）。

**可借鉴点**：小尺寸常驻 UI 的核心不是「塞更多」而是**仲裁**。定优先级、定固定尺寸、定「谁来了谁先说」，比响应式布局更重要。见 [compact-priority](components/compact-priority.html)、[activity-toast](components/activity-toast.html)、[subtitle-hierarchy](components/subtitle-hierarchy.html)。

### 5. 玻璃：便宜但完整的五层配方

`_paint_glass()`（`ui/island.py:967`）从下到上：

1. **backdrop** —— 后台线程 GDI 截身后画面 → 缩小 1/12（HALFTONE 面积平均）→ 3×3 平均模糊 ×2 → 纹理画刷填充任意形状（`ui/glass.py:96 Backdrop`）。截屏要等 DWM 回读 10~30ms，绝不放 UI 线程。
2. **色调 tint** —— 深色 rgba(28,28,34,120) 压一层，保证文字对比度。
3. **纵向光泽** —— top 白 alpha26 → mid 0 → bottom 10，体积感。
4. **轮廓** —— 外侧极淡暗线（1.6px）+ 内侧高光 rim（上 60 / 下 14），与背景分离。
5. **阴影** —— 三层半透明描边（14px/5、8px/9、4px/14）廉价近似柔和投影。

配套的防闪烁细节：新截图与上一张 **0.25s 交叉淡入**（背景突变不跳）；**收起时也预截展开尺寸的背景**（打开第一帧玻璃就是对的）；采样区域**对齐 16px 网格**（区域不随轮廓微动而抖）。

**可借鉴点**：毛玻璃 ≠ `backdrop-filter` 一行。要好看，tint/光泽/轮廓/阴影四层叠加；要不闪，交叉淡入 + 预取 + 网格对齐三件套。见 [frosted-glass](components/frosted-glass.html)。

### 6. 主题：自动切换要带滞回

auto 模式按身后画面亮度切明暗（`ui/theme.py:139 feed_luminance()`）：

- **EMA 平滑**：`lum = lum×0.8 + in×0.2`（瞬时值不直接用）。
- **双阈值滞回**：暗→亮要 >0.64，亮→暗要 <0.42，中间地带不动。
- **3 秒驻留**：持续越过阈值 3 秒才真正翻转 —— 「滚动的网页之类不会让玻璃来回闪」（源码注释原话）。
- 每套色板的**次级文字透明度单独调过对比度**（dark t2=0.66/t3=0.45，light 0.74/0.56，sky 0.78/0.60），因为玻璃底深浅不同；pill 上的文字另有 `pill_fg`（rouge 主题是粉底黑字）。

**可借鉴点**：任何「跟随环境自动切换」的 UI（明暗、紧凑、主题色）都该有 **EMA + 滞回 + 驻留**三件套，否则必然抖。见 [auto-theme-hysteresis](components/auto-theme-hysteresis.html)、[theme-palette](components/theme-palette.html)。

### 7. 不抢焦点的代价：轮询兜底

灵动岛用 `WS_EX_NOACTIVATE` 保持不抢焦点（输入框打字时不打断你正在用的窗口），代价是 **Qt 的失焦关闭、点外关闭全部失效**。wisland 的兜底：

- 右键菜单：25ms 轮询鼠标键 + 光标位置，按下且在菜单外就关；**先等打开菜单的那次按键松开才武装**（armed 状态机，防误关）（`ui/pages/system.py:44 _close_on_outside_click()`）。
- 输入框焦点：`request_focus()` 临时摘掉 NOACTIVATE + AttachThreadInput 抢焦点，`_poll_hover()` 发现前台窗口切走立即释放（`ui/island.py:1197`）。
- 穿透模式（WS_EX_TRANSPARENT）下鼠标完全穿过，用**低级鼠标钩子**只截获落在岛上的右键（`ui/island.py:73 _RightClickHook`）。

**可借鉴点**：常驻悬浮窗「不抢焦点」是体验底线，但要为它准备好轮询兜底 —— armed 状态机是关键（没有它，菜单刚打开就被那次点击自己关掉）。见 [focusless-popup](components/focusless-popup.html)。

### 8. 微交互里的信息层级

- **导航**：选中项宽度弹簧展开显示文字，文字在裁剪区内随展开进度渐显；其余只留图标 55% 不透明度；pill 底块跟着宽度流动（`ui/nav.py:26 NavBar`）。
- **字幕**：上行原文 12px/62% 透明，下行译文 15px 全色 —— 字号 + 透明度双通道分主次；超长**保留尾部**（最新的话最重要，ElideLeft）。
- **粗滑块**：图标在白色填充上翻转为深色、在轨道上是主题色 —— 用对比度翻转代替描边（`ui/widgets.py:247 Slider`）。
- **封面辉光**：hover 放大 7% 上浮 2.5px，身后透出封面主色光晕 —— 主色提取只要把图缩到 1×1 取像素，零成本（`ui/pages/home.py` `_set_cover`）。
- **涨跌色**：A 股习惯红涨绿跌，且 up/down 色**跟着主题走**（light 主题用更深的红 #d42c25 保对比度）。

见 [nav-spring-morph](components/nav-spring-morph.html)、[ios-controls](components/ios-controls.html)、[cover-glow](components/cover-glow.html)、[lyric-transition](components/lyric-transition.html)。

### 9. 性能：省帧是手感的一部分

- 所有弹簧 rest 后 **rAF/定时器停**，静止时零开销。
- 收起态常态动画（呼吸 + 流光）**空闲 30fps、忙碌 60fps**（`ui/island.py _tick()`：`setInterval(16 if compact.busy() else 33)`）。
- **脏矩形**：只重绘 old∪new 路径 +16px（`ui/island.py` path diff）。
- 毛玻璃采样按状态降频：展开 10fps、收起 3fps、拖动 40fps、参与截图时静止 1~1.5Hz（`ui/island.py _request_backdrop()`）。

见 [frame-budget](components/frame-budget.html)。

### 10. 有性格的小形象

PetFace（`ui/compact.py:26`）是「发光双眼」极简吉祥物，但状态机完整：视线弹簧跟随光标、**鼠标靠近时随机换表情**（不与上次重复）、2.4~5.6s 随机眨眼、60s 无人理→困倦、新消息→惊讶（弹簧给速度脉冲）、**按压时挤歪**（两眼分开 9px、外倾 14°、整体向没被按一侧倾斜 18°、压扁 16%）。

**可借鉴点**：吉祥物的「活」不靠关键帧多，靠 **随机性 + 注意力响应**（靠近才变、不理会困、被按会痛）。见 [pet-face](components/pet-face.html)。

## 目录结构

```
AA-Design/
├── README.md            ← 本文件：设计原则提炼（含源码出处）
├── index.html           ← 组件画廊首页
├── assets/              ← wisland 原版效果图（MIT）
└── components/          ← 21 个独立组件
    ├── 液态形态
    │   ├── liquid-edge-attachment.html   🫧 液态贴边（表面张力 + 液桥 + 水滴）
    │   ├── velocity-teardrop.html        💧 速度泪滴形变
    │   ├── press-squash.html             👆 按压挤压（体积守恒凹陷）
    │   └── idle-breathing.html           🌊 常态呼吸 + 流光 + 律动波
    ├── 动效引擎
    │   ├── spring-playground.html        ⚡ 阻尼弹簧调参台
    │   ├── frame-budget.html             ⏱️ 帧调度（停帧/分级/脏矩形）
    │   ├── nav-spring-morph.html         🧭 导航宽度弹簧展开
    │   └── edge-snap.html                🧲 吸附状态机（冲压挤压）
    ├── 玻璃与主题
    │   ├── frosted-glass.html            🪟 毛玻璃五层配方 + 交叉淡入
    │   ├── auto-theme-hysteresis.html    🌗 自动主题滞回（EMA+双阈值+3s）
    │   ├── theme-palette.html            🎨 四套色板分层
    │   └── cover-glow.html               🎵 封面主色辉光
    ├── 交互状态
    │   ├── hover-expand-lock.html        🖱️ 悬停展开与锁状态机
    │   ├── click-drag-timing.html        👆 单双击消歧 / 4px 阈值 / 长按
    │   ├── focusless-popup.html          🚪 不抢焦点弹层（轮询关闭）
    │   └── ios-controls.html             🎛️ 控制中心控件
    └── 内容编排
        ├── compact-priority.html         📶 收起态内容仲裁
        ├── activity-toast.html           🔔 活动提示（原地更新）
        ├── lyric-transition.html         🎤 歌词切换与跑马灯
        ├── subtitle-hierarchy.html       🔤 字幕双行层级
        └── pet-face.html                 👀 小形象状态机
```

## 使用方式

1. 打开 `index.html` 浏览所有组件
2. 点击卡片进入独立组件页面，每个都可交互
3. 每个组件页底部的「为什么值得借鉴 / 实现要点」标注了 wisland 源码出处（文件:行号），可对照 [原仓库](https://github.com/AsamiAkito/wisland) 深读
4. 组件是自包含 HTML，可直接复制到任何项目中改造

## 技术栈

- **原生 JavaScript** —— 无框架、无构建、无 CDN
- **Canvas 2D** —— 液态形变、小形象、律动
- **CSS backdrop-filter** —— 浏览器端毛玻璃（对应 wisland 的截屏管线）
- 动画全部自写阻尼弹簧（移植自 wisland `ui/anim.py`）

## 许可

wisland 采用 MIT License；本库为其设计模式的提取与浏览器重现，同样遵循 MIT。效果图版权归 wisland 原作者。
