# Harmony 出片设置（角色 AOV）

> 目标：把角色渲成**给合成用的 EXR 序列**，每个 pass 一个 Write 节点。
> 前提：相机/角色数据已经导出（见《使用说明》）。

---

## 〇、先分清两类 Pass

| | **组图**（组成画面） | **改进**（驱动效果） |
|---|---|---|
| 本文件的 | `char_color`、`char_line`、`char_eyes` | `char_normal` |
| 用途 | 直接叠进画面 | 当遮罩/参数，驱动重打光 |
| 色彩管理 | 要 | **绝对不要** |
| 压缩 | 可 DWAA | **必须 ZIP** |
| 滤镜 | 可加 | **绝对不能** |

**数据通道被 gamma / 有损压缩污染就废了。**

---

## 〇、先设色彩管理

`Scene > Scene Settings > Colour`：

| 项 | 值 |
|---|---|
| Working Colour Space | **`Linear Rec.709 (sRGB)`** |
| Processing Bit Depth | **`32 bits per channel (float)`** |

---

## 一、每个 Write 节点的通用设置

| 项 | beauty / line / eyes | normal / depth |
|---|---|---|
| Colour Space | **`Linear Rec.709 (sRGB)`** | 同左 |
| Image Format | **`EXR`** | **`EXR`** |
| Colours | **`Colour + Alpha`** | Colour（不要 alpha） |
| Depth (bits/channel) | **`32 bits (float)`** | 16 或 32 |
| Compression | **`DWAA`** | **`ZIP`** |

> **normal / depth 必须用 ZIP**：DWAA 是有损压缩，法线会出色带、深度会有台阶。

---

## 二、四个 Pass

### 1. `char_color` —— 角色本体

**接法**：角色整个合成（Composite）的输出 → Write。

- 带 Alpha；
- 命名：`char_color-####.exr`。

### 2. `char_line` —— 描边

**接法**：只把「描边」那部分接出来 → Write。

具体怎么拆取决于你的绑定（你说你能做）。

- **带 Alpha**（方便单独调色/发光）；
- 背景填黑或透明都行，合成里按 Alpha 用；
- 命名：`char_line-####.exr`。

### 3. `char_eyes` —— 眼睛发光

**接法**：只把「要发光的部分」（虹膜/瞳孔/光晕形状）接出来 → Write。

- **带 Alpha**——这是关键，拿它当遮罩在脸上加光；
- 不要包含白色高光（高光不该发光）；
- 命名：`char_eyes-####.exr`。

### 4. `char_normal` —— 法线

**接法**：

```
Volume Object（各部位）
   → Normal Map（雕出起伏）
   → Write
```

如果需要给特定插件用，中间加一个 **Normal Map Converter**：

| 设置 | 说明 |
|---|---|
| Conversion Type | `Genarts`（默认）或 `Custom` |
| Offset / Length / Invert | 按插件要求调 |

**通道约定**：Harmony 的法线是 **R=X、G=Y、B=Z**，值域 0–1。

- 带 Alpha 没意义，用 `Colour`；
- 命名：`char_normal-####.exr`。

---

## 三、`char_depth`：❌ 不需要渲

**实测结论（2026-11-06）**：Harmony 渲出的 EXR **只有 R/G/B/A，没有 Z 通道**。

> 官方文档说 EXR 带 Z，但前提是「场景里有 multiplane 或 3D 元素」。
> 只渲 2D 角色时不会写 Z。

**但角色不需要深度图**：角色是一张正对相机的平面，整张纸到相机的距离**相同**，
所以只需要**每帧一个数字**。

这个数字由 Blender 端自动算出来，写在 `light.json` 里：

```json
{ "f": 1, "dir": [...], "distance": 3.44, "depth": 12.84 }
```

- `distance` = 灯到角色（算光强用）
- `depth` = **相机到角色**（算景深/雾用）

**AE 里给角色做景深时，直接读这个值当关键帧。**

---

## 四、命名与目录

```
<镜头目录>/char/
    char_color-0001.exr
    char_line-0001.exr
    char_eyes-0001.exr
    char_normal-0001.exr
```

**每个 pass 一个序列**，不合并成一个多通道 EXR（AE 里更好认）。
**没有 `char_depth`**——深度用 `light.json` 里的数值。

---

## 五、检查清单

导出前：

- [ ] 工作空间是 `Linear Rec.709`
- [ ] 位深 32-bit float
- [ ] 每个 Write 的格式是 EXR
- [ ] beauty/line/eyes 是 `Colour + Alpha`
- [ ] normal/depth 用 `ZIP`
- [ ] 命名前缀对

导出后：

- [ ] `char_color` 的 Alpha 是角色轮廓
- [ ] `char_line` 只有描边
- [ ] `char_eyes` 只有眼睛（没有高光）
- [ ] `char_normal` 是紫蓝色（法线图的样子）
- [ ] 各文件尺寸一致、帧数一致

---

## 六、常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| 所有 pass 一样 | 接的是同一个 Composite | 每个 Write 接不同分支 |
| Alpha 全白 | 用了 `Colour` 不是 `Colour + Alpha` | 改设置 |
| 颜色发灰 | 输出空间不是 Linear | 检查 Colour Space |
| 法线重打光有色带 | 用了 DWAA | 改 ZIP |
| 法线方向不对 | Y 轴反了 | Normal Map Converter 里 Invert Green |
| 文件巨大 | 32-bit + DWAA | 数据通道用 16-bit + ZIP |
