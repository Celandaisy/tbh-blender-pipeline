# Blender 出片设置（环境层）

> 目标：把 3D 环境渲成**给合成用的 EXR 序列**，带上需要的通道。
> 前提：相机和角色已经导入（见《使用说明》）。

---

## 〇、先分清两类 Pass

| | **组图**（组成画面） | **改进**（驱动效果） |
|---|---|---|
| 本文件的 | `env_beauty`、`env_matte` | `env_depth`（还有 `env_mist`） |
| 用途 | 拼成画面 / 决定遮挡 | 当参数驱动景深、雾 |
| 色彩管理 | 要 | **绝对不要** |
| 压缩 | 可 DWAA | **必须 ZIP** |
| Kuwahara 滤镜 | 可以（只给 beauty） | **绝对不能** |

**`env_depth` 被平滑或压缩，景深就会出现台阶；`env_matte` 被 gamma 处理，遮挡就会错。**

---

## 一、渲染设置

| 项 | 值 | 说明 |
|---|---|---|
| Render Engine | EEVEE 或 Cycles | 两者都可以；影子在两边都测过 |
| Resolution | 跟 Harmony 一致（如 1920×1080） | 导入 camera.json 时会自动设好 |
| Frame Rate | 跟 Harmony 一致（如 24） | 同上 |
| Color Management | **不用改** | EXR 写的是场景线性数据，View Transform 不影响 |

> **View Transform（AgX/Filmic）只影响屏幕和 PNG，不影响 EXR。**
> 所以渲 EXR 时不用管色彩管理。

---

## 二、要开的 Pass

在 `Properties > View Layer > Passes` 里勾：

| Pass | 勾选 | 用途 |
|---|---|---|
| **Combined** | 默认 | 环境 beauty |
| **Z** | ✅ | 深度（雾、景深、排序） |
| **Mist** | 可选 | 距离雾（也可以拿 Z 自己算） |
| **Object Index** | ✅ | 前景遮挡 matte |
| **Cryptomatte Object** | 可选 | 比 Object Index 更精细 |

**不需要**：Normal、AO、Diffuse/Glossy 这些光照分解（卡通用不上）。

> **多通道 EXR 读不出来怎么办**：Blender 5.2 的「文件输出」节点加插槽得走 UI，
> 而且多 part EXR 在别的软件里也不一定认。
> **更稳的办法是深度单独渲一遍**：合成器里 `渲染层.深度 → 映射范围(0-1) → 合成`，
> 输出 **16 位灰度 PNG**。多渲一次，但到处都能用。

---

## 三、前景遮挡 matte 怎么做

**目的**：让「挡住角色的 3D 物体」在合成里能盖住角色。

1. 选中**前景物体**（挡在角色前面的那些，比如柱子、树）；
2. `Properties > Object > Relations > Pass Index` 设成 **1**（默认是 0）；
3. 渲染时 `Object Index` 通道里，这些物体的像素值就是 1；
4. 合成里用 `1 - matte` 或者直接拿它当遮罩。

> 如果前景物体很多、还要按物体分别处理，用 **Cryptomatte** 更好。

---

## 四、影子的设置

角色的影子由 `TB_CharShadow` 那个平面投出来（贴的是角色图）。

1. 确认 `TB_CharShadow` 已经贴了角色图（导入角色时指定）；
2. 地面要能接影子（普通材质就行）；
3. 渲染时影子**会直接进 beauty**——不需要单独的 shadow pass。

> **注意**：角色图要先渲出来，才能给影子平面用。
> 所以顺序是：Harmony 渲角色 → Blender 渲环境（带影子）。

---

## 五、输出（Compositor）

Blender 的 `File Output` 节点可以一次写多个 pass。

### 1. 打开合成

`Properties > Output > Post Processing > Use Nodes` 勾上，切到 Compositor。

### 2. 连法

```
Render Layers ─┬─ Image        → File Output (env_beauty)
               ├─ Depth (Z)    → File Output (env_depth)
               ├─ Mist         → File Output (env_mist)
               └─ Object Index → File Output (env_matte)
```

### 3. 每个 File Output 的设置

| 项 | 值 |
|---|---|
| File Format | **OpenEXR** |
| Color | RGBA（beauty）；**RGB**（depth/matte） |
| Color Depth | 32 bit（beauty）；16 或 32（数据通道） |
| Codec | **ZIP**（无损；数据通道必须无损）；beauty 可用 DWAA |

### 4. 路径与命名

```
<镜头目录>/env/env_beauty_####.exr
<镜头目录>/env/env_depth_####.exr
<镜头目录>/env/env_matte_####.exr
```

`#` 是帧号占位。

---

## 六、Kuwahara（风格化）

背景的油画感滤镜在 Blender 里做：

- 如果 Blender 的合成器里有 Kuwahara 节点，直接接在 Image 后面；
- 滤镜要作用在 **beauty** 上，**不要**作用在 depth/matte 上（那是数据）。

---

## 七、检查清单

渲染前：

- [ ] 相机动画在（拖时间轴能动）
- [ ] 角色导入过（有 `TB_CharProxy` / `TB_CharShadow`）
- [ ] `TB_CharShadow` 贴了角色图
- [ ] 前景物体设了 Pass Index = 1
- [ ] Z / Object Index 勾上了
- [ ] File Output 的格式是 OpenEXR，数据通道用 ZIP

渲染后：

- [ ] `env_beauty` 有 alpha
- [ ] `env_depth` 是单通道，数值范围合理
- [ ] `env_matte` 里前景物体是白的
- [ ] 影子在 beauty 里

---

## 八、常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| depth 全白/全黑 | 相机裁剪面太大 | 调 `Clip Start/End` |
| matte 全黑 | 没设 Pass Index | 给前景物体设 1 |
| 没影子 | 影子平面没贴图 / 没开 Ortholock | 见《使用说明》第五节 |
| 影子是方块 | 还在用旧的盒子版本 | 重装插件，用平面版 |
| EXR 发灰 | 当作 sRGB 读了 | 在合成里改成 Linear |
