# AE 合成清单

> 目标：把 7 个 pass + light.json 合成成画面。
> 前提：TBH 和 Blender 的素材都渲好了。

---

## 一、层序（从下往上）

```
⑧ 调色 / 颗粒 / 暗角
⑦ 大气（雾、景深）        ← env_depth + light.json 的 depth
⑥ 前景遮挡                ← env_matte
⑤ 眼睛发光（Add + Glow）  ← char_eyes
④ 描边                    ← char_line
③ 角色（Normality 重打光） ← char_color + char_normal + AE 灯
② 角色影子                 ← 已烤在 env_beauty 里
① 环境                    ← env_beauty
```

---

## 二、Normality 怎么配（要点）

> **注意**：以下基于插件的参数面板，具体位置可能随版本不同。

### ① 准备两个层

- 角色图（`char_color`）；
- 法线图（`char_normal`）。

**尺寸必须一致**，然后：

1. 选中两个层 → `图层 > 预合成`（Pre-compose）；
2. 勾 **「移动所有属性到新合成」**。

### ② 加 Normality

选中预合成 → `效果 > miningde Normality`。

### ③ 接法线

`Normals` 组：

| 参数 | 填 |
|---|---|
| Normal Map | **选法线图那一层** |
| Alpha Map | 角色层（如果需要） |
| X / Y / Z | Red / Green / Blue（默认） |
| Invert Y | 视法线格式而定 |

### ④ 加灯

**Normality 读的是 AE 合成里的灯**（它自己的 `Lights` 组只有全局设置）。

1. `图层 > 新建 > 灯光`；
2. 类型选 **点光**；
3. Normality 会自动用它打光。
4. 也可以直接用脚本：`tools/light_to_ae.jsx`（从 `light.json` 建灯 + 打关键帧）。

### ⑤ 调风格

| 组 | 调什么 |
|---|---|
| **Toon** | 暗部/高光颜色、强度、宽度 ← **最像手绘** |
| **Rim Light** | 边缘光的颜色、角度、宽度 |
| **Shading** | Diffuse、环境色 |
| **Specular** | 高光强度、扩散 |

---

## 三、light.json 怎么用

### 建灯（自动）

1. `文件 > 脚本 > 运行脚本文件` → 选 `tools/light_to_ae.jsx`；
2. 选 `light.json`；
3. 脚本会为每个灯建一个 AE 点光，逐帧打位置/颜色/强度。

**两个可调常量**（脚本开头）：

```js
var Z_SIGN = -1.0;          // 明暗方向反了 → 改成 1
var INTENSITY_SCALE = 1.0;  // 太亮/太暗 → 调这个
```

### 角色深度

`light.json` 里每帧有 `depth`（相机到角色的距离）。

**做景深时**：角色层用这个值当关键帧，不要用 `env_depth`（那是环境的）。

---

## 四、眼睛发光

1. `char_eyes` 放在角色上面；
2. 混合模式 **Add**（或 Screen）；
3. 加 **Glow**（发光）效果；
4. **遮罩**：用 `char_color` 的 alpha 或脸的遮罩，让光只在脸上扩散；
5. **不要**让白色高光发光（`char_eyes` 里就不该包含高光）。

---

## 五、描边

`char_line` 放角色上面，可以：

- 单独调色（Tint）；
- 加轻微模糊；
- 混合模式 Normal。

---

## 六、大气 / 景深

用 `env_depth`：

- **雾**：`效果 > 3D 通道 > 深度遮罩` 或第三方雾效；
- **景深**：AE 自带的「镜头模糊」+ 深度图。

> 角色是平的，**整层用同一个 `depth` 值**就行。

---

## 七、常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| Normality 没反应 | 法线图没接 | `Normals > Normal Map` 选层 |
| 光照不跟着动 | 灯没关键帧 / 脚本没跑 | 跑 `light_to_ae.jsx` |
| 明暗方向反了 | 坐标符号 | 改 `Z_SIGN` |
| 太亮/太暗 | 强度映射 | 改 `INTENSITY_SCALE` |
| 角色边缘发黑 | 预乘 alpha | 检查是否 Premultiplied |
| 深度图全白/全黑 | 映射范围不对 | 调 `0–200` 的远端值 |

---

## 八、还没验证的

- `light_to_ae.jsx` 的 `Z_SIGN`（没跑过）；
- Normality 的具体参数名（以实际面板为准）；
- `char_normal` 的通道格式（等绑定做完）。
