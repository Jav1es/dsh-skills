---
name: static-portfolio-build
description: 从零搭建/改造纯静态多页作品集（无框架、无构建），含数据驱动渲染、SVG 示意图、3D 与动效、性能优化、上线前验收。当用户要求"做作品集网站 / 多页站点 / 加作品卡片 / 美化页面 / 上线前检查 / 页面掉帧优化 / 部署到 GitHub Pages"时使用。含本次实战踩过的坑与验证方法。
metadata:
  scope: web-design
---

# static-portfolio-build — 静态多页作品集

> 技术红线：**只用 HTML5 + CSS3 + 原生 JS**。禁 Bootstrap/Tailwind 等 UI 框架，禁外部 CDN 字体。
> 主色 ≤3 种、字体 ≤2 种、每图必须有 `alt`（SVG 用 `role="img"` + `aria-label`）。
> 首屏 3G ≤3s（1.6Mbps / RTT 400ms 估算）。

## 一、架构：数据驱动，一处维护

**不要在每个 HTML 里手写作品内容。** 抽成 JSON 数据 + 一个渲染层：

```
works-data.json    数据源（脚本生成，勿手改）
works-data.js      同数据内联版（window.__WORKS_DATA__）—— 供 file:// 双击打开绕过 CORS
works.js           渲染层：ROLES / cardHTML() / tileHTML() / bindFilter() / bindExpand()
extract-works.mjs  从旧单页抽取
patch-works.mjs    重编号/补模块/分配 motif（必须幂等！）
```

**为什么必须内联一份 JS 数据**：`fetch('./works-data.json')` 在 `file://` 协议下被 CORS 拦。用户双击打开就白屏。

**幂等补丁脚本**：`patch-works.mjs` 要能重复运行。用 `EXTRA_TITLES` 过滤已补过的条目，否则第二次运行抛"未在映射表中"。

**模块编号与真实目录对齐**：作品编号不要凭类目排，要按本地目录/GitHub 仓库的真实归属排。按"有目录的对齐目录号，无目录的填空位"保证连续无缺口。改编号时全站引用（入口卡片、岗位页 banner、关于页类目）同步——**最好让入口卡片由数据自动生成**，杜绝手写不同步：

```js
document.querySelectorAll('[data-owns]').forEach(function (el) {
  var r = WORKS.ROLES[el.getAttribute('data-owns')];
  if (r) el.textContent = '承载 ' + r.owns.length + ' 个模块（' + r.owns.join(' / ') + '）';
});
```

## 二、没有截图怎么办：内联 SVG 示意图

作品多半没有截图。按类目分配几何母题，生成内联 SVG（零外部图片、首屏零图片请求）：

| 类目 | 母题 | 元素 |
|---|---|---|
| 规划 | roadmap | 节点 + 虚线连接 + 里程碑 |
| 工作流 | pipeline | 输入→处理→审核→输出 + 箭头 marker |
| 智能体 | neural | 分层节点 + 全连接边 |
| 数据 | metrics | 柱状 + 折线趋势 |
| 开源 | terminal | 深色窗口 + 彩色代码行 + 光标 |
| 行业 | grid | 蓝图网格 |

每张必须带 `role="img"` + 语义化 `aria-label`。

**踩坑记录**：
- SVG 里画 `<text>` 会被卡片渐变遮挡 → 标签移出 SVG，用 HTML 覆盖层 `.w-cover-tag`（`position:absolute`）
- `preserveAspectRatio` 用 `meet` 不用 `slice`，避免裁切
- **数组索引错位**：母题数据写成 `[x,y,文字,颜色]` 但渲染按 `[x,y,width,color]` 取 `l[2]` → 矩形全失效只显示光标。写母题时先确认索引语义一致。

## 三、3D 与动效（好看但不能掉帧）

### 背景：SpaceX 式透视网格

```css
.bg-grid-floor {
  position: absolute; left: -30%; right: -30%; top: 16%; height: 90%;
  background-image:
    repeating-linear-gradient(90deg,  rgba(10,10,10,.16) 0 1px, transparent 1px 74px),
    repeating-linear-gradient(180deg, rgba(10,10,10,.15) 0 1px, transparent 1px 40px);
  transform: perspective(520px) rotateX(72deg) translateZ(0);
  transform-origin: 50% 0%;
  mask-image: linear-gradient(to bottom, transparent 0%, #000 11%, #000 52%, transparent 90%);
  animation: floorDrift 30s linear infinite;   /* 改 background-position，不用 transform */
}
```

再叠地平线辉光线 + 三团漂移光斑（各自 24/29/34s 呼吸）。

### 性能铁律（这是掉帧的分水岭）

**绝不要用 `filter: blur()` 和 `backdrop-filter`。** 这两类每帧重新栅格化/重新采样背后内容，是滚动掉帧的头号原因：

- 柔光斑用 `radial-gradient(circle, rgba(...).10, transparent 68%)` —— 渐变本身就是柔的，不需要 blur
- 毛玻璃导航改用近不透明背景 `rgba(255,255,255,.96)`
- section 垫底用半透明白底，不用 backdrop-filter

**动画只碰 `transform` / `opacity`**（合成器处理，不触发重排重绘）。滚动视差用 `requestAnimationFrame` 节流 + `translate3d`。

### 卡片 3D 倾斜

`pointermove` 写 CSS 变量，不动 DOM：

```css
.tilt { transform: perspective(1000px) rotateX(var(--rx,0deg)) rotateY(var(--ry,0deg)); }
.tilt.is-tilting { transition: transform .06s linear; }  /* 跟手时降 transition */
```

`.tilt .w-body { transform: translateZ(18px); }` 造视差浮起；`::before` 径向高光跟随 `--mx/--my`。

**hover 位移与进场动画会抢 transform** → hover 只改 box-shadow/border-color，位移全交给 `.tilt`。

## 四、进场动画：绝不能用纯 CSS `opacity:0`

**这是最大的坑。** 纯 CSS 起始隐藏 + JS 加 `.is-revealed`，一旦 IntersectionObserver 不回调（无头浏览器、JS 报错、异常路径），**整页内容永久不可见**。

正确做法：**由 JS 按需加隐藏态类**，未加类的默认可见：

```css
.w-card.reveal-armed { opacity: 0; transform: translateY(18px); }
.is-revealed { opacity: 1 !important; transform: none !important; }
```

```js
list.forEach(function (el) {
  var r = el.getBoundingClientRect();
  var inView = r.top < innerHeight + 150 && r.bottom > -150;
  if (!inView) el.classList.add('reveal-armed');   // 只藏视口外的
  io.observe(el);
});
// 兜底：3 秒后仍未显示的强制显示
setTimeout(function () { /* force reveal */ }, 3000);
```

视口内的立即可见，视口外的才武装 —— 首屏不会被动画拖累，异常路径也不会白屏。

## 五、脚本注入：校验 `<script>` 开闭配对

给多个页面批量注入代码时，**必须校验 `<script>` 与 `</script>` 数量相等**。

**真实事故**：注入时多写了一个 `<script>` 开标签 → 后面的内联脚本块被 JS 解析器吞掉 → 四个岗位页渲染代码全部失效。**页面看着完全正常**（标题、导航、职责清单都在），只有作品区空白，肉眼几乎发现不了。

正则注入时注意：`LastIndexOf('<script>')` 会命中 `<script src=...>`，导致定义被插到标签**外面**变成可见文本。改用明确的锚点串。

## 六、验证方法（可复用）

### 截图验证（关键：同名文件走缓存）

`vision_html_screenshot` 对同名文件会走缓存。改 CSS 后必须 `Copy-Item` 到 `.verify/` 临时名再截图，验完删沙盒。

### 同步探针（不要用 load + setTimeout）

无头环境里 `window.load` + `setTimeout` 不可靠。**把探针写进渲染回调内部**（同步执行），输出到 `position:fixed;bottom:0` 的 `<pre>`：

```js
WORKS.loadData(function (data) {
  root.innerHTML = WORKS.listByRole(data.items, 'fde');
  var d = document.createElement('pre');
  d.style.cssText='position:fixed;bottom:0;left:0;right:0;z-index:99999;background:#000;color:#0f0;font:11px monospace;padding:4px;margin:0';
  d.textContent = 'CARDS=' + root.querySelectorAll('.w-card').length;
  document.body.appendChild(d);
});
```

⚠️ 探针不要用 `top` 定位 —— 卡片 3D 倾斜的 transform 会把 fixed 元素挤偏。

### 批量核验（比逐页读文件省）

- 死链：正则抽 `./xx/yy.md` → `Test-Path` 逐条验
- 编号连续性：`Compare-Object` 期望序列 vs 实际
- 目录对齐：**PowerShell 比较 `'01'` 会被转成 int 1**，必须 `[string]` 强转
- 昂贵特性残留：`[regex]::Matches($css,'filter:\s*blur|backdrop-filter')`，注意区分注释

## 七、部署到 GitHub Pages

```bash
gh repo create <user>/<repo> --public --description "..."
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main
```

**部署前必查**：
1. `git ls-files` 核对依赖目录**真的被跟踪了**（本地有文件 ≠ git 跟踪）
2. 远端已有内容时先 `git ls-tree --name-only origin/main` 对比，别覆盖丢东西
3. 本地相对路径依赖的兄弟项目目录：要么一起推，要么改成 GitHub 仓库地址
4. GitHub Pages 需 Settings → Pages → Source 选分支
5. 有子应用（如 `hitech-self-assessment/`）时别被主站替换覆盖

**性能估算**：`总字节 / 8 / 1.6e6 + 4 * 0.4`（秒）

## 八、交付前自检（对齐 web-delivery-checklist）

- [ ] 四/五个页面全部**截图验证过**（不是只验首页）
- [ ] 死链 0、缺 alt 0
- [ ] `<script>` 开闭配对、CSS 花括号平衡
- [ ] `filter: blur` / `backdrop-filter` 实际生效数 = 0
- [ ] 3G 估算 < 3s
- [ ] 主色 ≤3、字体 ≤2、零外部 CDN 资源
- [ ] 移动端（390px）汉堡菜单可展开收起、`aria-expanded` 同步
- [ ] `prefers-reduced-motion` 下关闭全部动画
