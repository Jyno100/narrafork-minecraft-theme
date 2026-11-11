# 技术文档 · Minecraft Theme for NarraFork

本文记录这个主题插件的实现细节：插件如何被宿主消费、素材如何从原版还原、场景如何生成、GIF 如何编码、以及主题 token 体系中几个反直觉的语义。

面向要修改或扩展本项目的人。所有数据均来自实测，不是推测。

---

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────┐
│ 素材层        原版方块贴图 1072×1072（davidmc.cn）        │
│               ↓ 逐像素还原 16×16                          │
│              end16.json / ground16.json / village16.json │
├─────────────────────────────────────────────────────────┤
│ 生成层        make-overworld.js   → tx-sky(.night).gif   │
│               make-nether.js      → tx-nether(-light).gif│
│               make-end.js         → tx-end(-light).gif   │
├─────────────────────────────────────────────────────────┤
│ 声明层        manifest.json      三套 theme contribution  │
├─────────────────────────────────────────────────────────┤
│ 打包层        pack3.js            → .nfplugin (ZIP)      │
└─────────────────────────────────────────────────────────┘
                              ↓
                       NarraFork 宿主
              主题编译器 → CSS → UI 应用
```

关键约束：**插件只能声明 token，不能注入 CSS 或 DOM，也不能运行 JavaScript**。所有视觉效果必须落在宿主白名单内的表面上。

---

## 2. 素材还原管线

### 2.1 源素材规格

指定素材站 `davidmc.cn` 提供的方块贴图统一是 **1072×1072**，即 16×16 的 **67 倍**放大（16 × 67 = 1072）。

下载 URL 规律（注意没有 `/materials` 前缀）：

```
https://www.davidmc.cn/downloads/textures/<urlencode(中文名)>.png
```

### 2.2 还原方式

按 texel 中心取样，而不是缩放：

```js
function extract16(file) {
	const img = decode(file);
	const SCALE = img.w / 16;            // 1072 / 16 = 67
	const tile = [];
	for (let y = 0; y < 16; y++) {
		for (let x = 0; x < 16; x++) {
			// 取每个 67×67 方块的中心像素
			tile.push(sample(img, x * SCALE + Math.floor(SCALE / 2),
			                     y * SCALE + Math.floor(SCALE / 2)));
		}
	}
	return { tile, w: img.w, h: img.h, scale: SCALE };
}
```

因为源图是整数放大，中心像素就是原始 texel 的精确值，不存在插值误差。

### 2.3 需要着色的灰度贴图

原版有两个方块贴图是**灰度图**，颜色由群系决定，必须手动乘色，否则会得到灰白方块：

| 贴图 | 群系色 |
| --- | --- |
| `grass_block_top` | `#91BD59` |
| `oak_leaves` | `#77AB2F` |

```js
const tint = (tile, t) => tile.map(([r, g, b, a]) => [
	Math.round((r / 255) * t[0]),
	Math.round((g / 255) * t[1]),
	Math.round((b / 255) * t[2]), a,
]);
```

---

## 3. 场景生成器

三个生成器结构相同，都是「静态地形 + 逐帧动态元素」：

```
buildStatic(scheme)  → 地形数组 + 天空填充     （每套配色算一次）
buildFrames(terrain) → 60 帧，叠加动态元素      （熔岩/生物/龙）
encode(frames, out)  → GIF
```

### 3.1 尺度基准

所有场景统一按 **约 10px = 1 格**设计。这个基准由干草块确定（10px 高 = 1 格），并推得：

| 对象 | 原版高度 | 本场景像素 |
| --- | --- | --- |
| 村民 | 1.95 格 | 20px |
| 末影人 | 2.9 格 | 28px |
| 牛 / 羊 / 猪 / 鸡 | 1.4 / 1.3 / 0.9 / 0.7 格 | 15 / 13 / 11 / 13px |
| 村庄房屋 | ~4 格 | 40-48px |
| 干草块 | 1 格 | 10px |

比例错了会立刻被看出来 —— 早期村民画成 31px（按 16px/格 推算），结果比房子还高。

### 3.2 云层：为什么必须量化

`clouds.png` 是原版 `environment/clouds.png`（256×256，1-bit）。实测其连通结构：

| 指标 | 数值 |
| --- | --- |
| 横向连续段长度中位数 | **2 texel** |
| 横向连续段最大长度 | 23 texel |
| 长度 ≥16 texel 的段占比 | 1% |
| 孤立单像素块 | 502 个（占连通块 **42%**） |

结论：**原版纹理本身没有大云板**。游戏里看着成块，是因为一个 texel 被渲染成 12×12 格的巨大方块。

若按 1 texel = 2px 绘制，得到的是满屏盐粒。最终方案分两步：

**第一步：量化到粗格。** 256×256 → 64×64，每格 4×4 texel，阈值 40%（`n >= 6/16`）。

**第二步：闭运算合并碎块。** 在环面（torus）上做形态学闭运算（先膨胀再腐蚀）：

```js
const at = (g, x, y) => g[((y % N) + N) % N][((x % N) + N) % N];
grid = morph(morph(grid, 1, 1), 1, 0);   // 闭运算 radius 1
grid = dropSmall(grid, 3);                // 丢弃 <3 格的碎块
```

实测各方案对比：

| 方案 | 覆盖率 | 连通块 | 孤立块 |
| --- | --- | --- | --- |
| 量化后原样 | 24.7% | 169 | 84 |
| **仅膨胀** radius 1 | **69.8%** | 3 | 0 |
| 闭运算 radius 1 | **36.5%** | 11 | **0** |
| 闭运算 + 再腐蚀 1 | **10.6%** | 30 | 3 |
| 闭运算 radius 2 | 64.1% | 1 | 0 |

要点：

- 纯膨胀把 24.7% 推高到 69.8%，天空几乎铺满
- 闭运算保持面积不变，只焊接碎片
- 但云板本身很薄，**闭运算后再腐蚀一圈会直接切断**（36.5% → 10.6%）
- 最终定在闭运算 radius 1，稳定性（36.5%）来自 `n >= 6/16` 与 `n >= 7/16` 结果相同

**第三步：保证循环周期。** 64 格 × 8px = **512px**，恰好等于画布宽度：

```
FRAMES × STEP = 60 × 8.53 = 512px = 一个纹理周期
```

这样第 60 帧与第 0 帧在几何上连续，滚动无缝。若改成 1 texel = 8px（不量化），周期会变成 2048px，循环点会突跳 486px。

**第四步：纵向分带 + 抖动。** 云只在上半天空，108px 以下开始按格抖动淡出，152px 以下完全没有。抖动键必须用**取模后的格坐标**，否则同一块云在不同位置的淡出不一致：

```js
const gx = ((cx % N) + N) % N;   // 正确：wrap 后的坐标
// 错误：直接 hash(cx, cy) —— cx 是未 wrap 的滚动坐标
```

### 3.3 帧数与运动归一化 ⚠️

背景统一 **60 帧**、`DELAY = 43`（0.43s/帧，约 2.3 fps，每循环 25.8s）。

改帧数时**不能只改常量**。所有运动都按帧索引累加，帧数翻三倍会让位移跟着翻三倍 —— 生物会走出各自的活动区间、直接走进房屋里，熔岩带也会三倍速。

做法是引入归一化时间，让运动量与帧数解耦：

```js
const ANIM_FRAMES = 20;                  // 原有运动参数是按 20 帧调的
const tf = f * (ANIM_FRAMES / FRAMES);   // 60 帧时 tf 仍走 0→20
```

所有帧驱动的表达式改用 `tf`：生物行走距离、相位、熔岩带相位、恶魂触须摆动。末地生成器原本就用 `t = f / FRAMES × 2π`，天然免疫。

`DELAY` 也要同步换算，否则会变成三倍慢的幻灯片：

| | 20 帧 | 60 帧 |
| --- | --- | --- |
| 每帧时长 | 1.28 s | 0.43 s |
| 每循环 | 25.6 s | 25.8 s |
| 云漂移 | 20 px/s | 20 px/s |

验证方式不是看截图，而是按个体分组做像素断言 —— 每帧把村民的肤色像素聚成独立跨度，再判断跨度是否与房屋区间相交：

```
村民个体跨度: 49-80  与 269-293
落入房屋区间: 0
```

未做归一化时这条断言会立刻失败。

---

## 4. GIF 编码器

自行实现（环境无 ImageMagick/ffmpeg/PIL），导出 GIF89a：

- 逐帧调色板合并成全局色表，`PAL_SIZE = 1 << ceil(log2(count))`
- NETSCAPE2.0 扩展块实现无限循环
- 每帧 GCE 设置 `delay = 43`（GIF 以 1/100 秒为单位，即 0.43s）与 disposal `0x04`

### 4.1 LZW 的坑

标准 GIF LZW 会让码宽随字典增长而递增，这一步**极易 off-by-one**，导致解码器与编码器宽度不同步。

本项目改用**固定码宽 + 定期发 clear 码**：

```js
const codeSize = minCodeSize + 1;        // 固定，永不增长
const MAX_CODE = (1 << codeSize) - 2;    // 阈值
// ...
if (next >= MAX_CODE) {
	emit(clear);                         // 重置字典
	dict = new Map();
	next = eoi + 1;
}
```

代价是压缩率略低，换来的是可预测性。并强制做往返自检：

```js
for (let f = 0; f < frames.length; f++) {
	const enc = lzwEncode(minCodeSize, indexed[f]);
	const dec = lzwDecode(minCodeSize, enc);
	if (dec.length !== indexed[f].length || dec.some((v, i) => v !== indexed[f][i])) {
		console.error("LZW FAILED frame", f); process.exit(1);
	}
}
```

解码器也必须用固定码宽、遇 clear 即重置，否则自检会失败。

---

## 5. 主题 token 体系

### 5.1 表面白名单

主题只能给白名单内的表面设背景。名称 → 宿主选择器的映射是固定的，插件无法指定任意选择器。部分关键项：

| 表面 | 选择器 | 说明 |
| --- | --- | --- |
| `body` | 主题作用域根元素 | 也控制 `--mantine-color-body` |
| `app` | `.nf-app-shell` | 整个应用壳 |
| `main` | `.nf-app-shell-main` | 主内容区（对话背景） |
| `navbar` | `.mantine-AppShell-navbar` | 侧边栏 |
| `header` | `.mantine-AppShell-header` | 桌面端高度为 0，仅承担顶部内边距 |
| `card` / `paper` | `.mantine-Card-root` / `.mantine-Paper-root` | Card 内部渲染 Paper，故有固定的发射顺序 |

### 5.2 `colorScheme` 语义

只有三个值：`light` / `dark` / `both`。

- `light`：只在浅色模式生效（选择器带 `[data-mantine-color-scheme="light"]`）
- `dark`：只在深色模式生效
- `both`：无条件生效；**若同时提供 `tokens.light` 与 `tokens.dark`，则按系统模式分别应用** —— 这就是「跟随系统」

「跟随系统」不是一个独立开关，而是 `colorScheme: "both"` + 双套 token 的自然结果。

### 5.3 `opacity` 是遮罩强度，不是图片透明度 ⚠️

这是最容易搞反的一个。它被编译成：

```css
background-image: linear-gradient(
  color-mix(in srgb, var(--mantine-color-body) N%, transparent),
  color-mix(in srgb, var(--mantine-color-body) N%, transparent)
), url("...");
```

`N` 就是 `opacity × 100`。**数值越大，图案被 body 色盖得越狠，也就是越淡。**

实测教训：想让侧边栏纹理更明显，把 `opacity` 从 0.12 提到 0.55，结果正好相反 —— 材质被糊成一块平色。可用区间大约 **0.05 – 0.30**。

### 5.4 `--mantine-color-body` 会给整个界面蒙灰 ⚠️

所有表面的 scrim 都取这个颜色。如果浅色主题的 body 设成很亮的浅色：

```json
{ "body": "rgba(255, 236, 226, 0.10)" }   // 亮度 241
```

那么**每一层遮罩都在往上叠米白**，侧边栏、卡片、面板一起变得雾蒙蒙 —— 表现为「整屏像隔着毛玻璃」，而单纯换材质并不能解决。

必须把它设成能当遮罩用的中深色：

```json
{ "body": "rgba(58, 30, 22, 0.22)" }      // 亮度 37
```

叠加后的等效背景亮度：`imgLuma × (1 - a) + bodyLuma × a`。上例把天空从 128 压到 99，文字对比度仍有 5.82:1。

### 5.5 `colors.<name>` 给单色时改深浅无效

单色输入只取色相/饱和度，明度走固定阶梯：

```
[0.96, 0.9, 0.8, 0.68, 0.56, 0.46, 0.42, 0.36, 0.29, 0.22]
```

要让灰色系真的变深变浅，必须给 **10 级色阶数组**。

### 5.6 其他编译期约束

- `backgrounds.size` 只接受 `cover | contain | auto`，**默认 `cover`**（不写会被拉伸）
- `backgrounds.position` 只接受 `center | top | bottom | left | right`
- `repeat` 只接受 `no-repeat | repeat`
- `overlay` 只接受 `none | scrim-light | scrim-dark`
- `isSafeColor` 接受 hex / `rgb()` / `rgba()`（含小数 alpha 与百分比）
- CSS 编译上限 **196,608 字节**（当前三主题分别约 73–75 KB）

---

## 6. 浅色方案的画法

地狱与末地本质是暗场景。浅色方案有两种做法，只有一种可行：

| 做法 | 结果 |
| --- | --- |
| 在成品暗图上叠半透明暖色 | 观感是「隔着毛玻璃」，画面被压平 |
| **重绘：地形线性增益 + 单独设计明亮天空** | 保留纹理层次，真实变亮 |

原因是曲线形状：

- **gamma < 1** 会把直方图压进一条窄亮带 → 局部对比度下降 → 「糊」
- **线性缩放** `out = c × k + b` 保持并放大块与块之间的差异

地狱浅色的最终参数：

```js
terr: (c) => {
	const k = 1.55, b = 30;
	return [clamp(c[0] * k + b), clamp(c[1] * k + b), clamp(c[2] * k + b)];
},
skyTop: [150, 88, 62],       // 深暖橙
skyHorizon: [226, 158, 108], // 浅琥珀
```

效果验证：天空亮度 116、地面 126，深色文字对比度 **6.80:1 / 7.33:1**；天空区域亮度标准差 **10.1**（若叠雾会趋近 0）。

---

## 7. 生物绘制

所有生物都是像素函数拼出的，不是贴图。几个通用规律：

### 7.1 颜色不能与背景同色

末地深色模式下，幻翼初版用了 `[58,64,96]`，而夜空云层是 `[58,66,96]` —— **几乎完全相同**，直接融进云里看不见。改成紫色调 `[104,92,156]` 才分离出来。

这类问题无法靠「看起来差不多」判断，必须取具体 RGB 值对比。

### 7.2 纯黑剪影会消失在虚空里

末影人身体接近纯黑，在深色虚空中只剩眼睛可见。解法是**只在一侧加轮廓光**：

```js
// 只描左边，四边都描会像贴纸描边而不是体积
rect(px, x + 2, headTop, 1, headH, E.rim);
```

### 7.3 高度必须实测而非按估算

初版末影人腿与袍摆之间有 1px 空隙，看上去两条腿是悬空的。原因是 `robeBot = baseY - 4` 而腿起始于 `baseY - 3`，少算一行。

提交前用像素断言验证包围盒，而不是肉眼看截图：

```
villagerA height 20px (rows 221-240)  sleevePx 28  handPx 8  OK
```

`28 = 2 袖 × 2px × 7 行`，`8 = 2 手 × 2×2`，数字对得上才说明双臂真的画出来了。

---

## 8. 构建与发布

### 8.1 打包

`.nfplugin` 就是一个 ZIP。打包脚本收录目录下**所有**文件（含 manifest 未引用的）：

```bash
node pack3.js <srcDir> <outFile>
```

### 8.2 版本号是硬性要求 ⚠️

**同版本号但内容 hash 不同，安装会被硬拒**（`VALIDATION_ERROR`）。每次改包必须升版本号。

### 8.3 安装流程

```bash
# 1. 安装（multipart 字段名必须是 archive）
POST /api/plugins/install

# 2. 启用插件
POST /api/plugins/{id}/enable

# 3. 逐个启用主题
PUT /api/plugins/ui/themes/{id}/{themeId}   {"enabled": true}
```

**`/api/plugins/ui/themes` 只返回已启用的主题。** 新主题必须先 PUT 启用一次才会被编译下发 —— 否则会误判为「主题没生效」。

### 8.4 校验清单

发布前逐项确认：

| 项 | 方法 |
| --- | --- |
| Schema 合法 | 按宿主规则重实现校验器，要求 errors 0 / warnings 0 |
| 素材齐全 | manifest 引用的每个 `assets/*` 都存在 |
| 二进制完好 | 打包后对 GIF/WOFF2 计算 SHA-256，与源文件比对 |
| 主题真的生效 | 拉取编译后 CSS，按 surface 逐条解析出实际 URL 与 scrim 值 |
| 宿主下发正确 | 下载 theme-asset，比对 SHA-256 与本地一致 |

其中「拉编译后 CSS 逐条解析」最有效 —— 它能区分「配置写对了」和「编译器实际输出了什么」。

---

## 9. 已知约束

| 约束 | 说明 |
| --- | --- |
| 不能用 `footer` 表面做底部条 | 宿主不渲染 `AppShell.Footer` |
| 桌面端 `header` 高度为 0 | 只承担顶部内边距，图案基本不可见 |
| 对话页有不透明容器 | 取 `var(--mantine-color-body)`，body 太实会把纹理全挡住 |
| 背景图无像素化控制 | token 体系没有 `image-rendering`，放大后由浏览器平滑 |
| `cover` 会裁切 | 512×300 的图在宽扁视口下可能裁掉上下内容 |
| 每个主题需单独启用一次 | 见 8.3 |

---

## 10. 文件清单

```
manifest.json                      三套主题完整 token
assets/
  ├─ tx-sky.gif / tx-sky-night.gif     主世界 白天 / 夜晚
  ├─ tx-nether.gif / -light.gif        地狱 深色 / 浅色
  ├─ tx-end.gif / -light.gif           末地 深色 / 浅色
  ├─ tx-*.png                          方块贴图（地板/墙/边框/UI）
  └─ font-pixel-10.woff2               像素字体子集（4222 字形）
dist/minecraft-theme-<ver>.nfplugin 打包产物
docs/TECHNICAL.md                   本文
```

生成器脚本不随仓库分发（依赖本地素材与工具链）。如需复现，见第 2、3 节的取样与生成规则。
