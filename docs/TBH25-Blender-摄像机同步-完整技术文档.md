# TBH 25 → Blender 摄像机同步 · 完整技术文档

> **文档性质**：设计 + 实测验证文档（核心链路已跑通）
> **生成日期**：2026-11-03　**最后更新**：2026-11-05
> **版本**：v2.0
> **适用软件**：Toon Boom Harmony 25 / Blender 4.1–5.x / After Effects 或 Autograph
> **用途**：可独立交接给另一个 AI 对话或同事；不需要原作者在场解释即可读完并继续工作。
>
> **变更摘要**
> - v2.0（2026-11-05）：换算公式实测通过（五类工况、10 帧、0–1 px）；新增 7.8–7.12；工具链落地（`tools/`）；重写第 9、10 节。
> - v1.4（2026-11-04）：公式右侧由 `B⁻¹` 改为 `F`；新增 7.6 验证记录。
> - v1.3：补充合成技术清单、环境依赖、交接验收清单。

---

## 交接说明（给下一个接手的人 / AI）

1. 本文不是「怎么用插件」的教程，而是「**为什么这么设计、每个假设的边界在哪**」的设计文档。
2. **第 7.8–7.12 节是最终实测结果**，第 9 节是确认清单；合成（13 节）仍未开始，勿当成已解决。
3. 核心公式在 **7.9 节**（完整式）与 **2.2 节**（推导），数据契约在 **3 节**，两端实现在 **4、5 节**与 **`tools/`**，验收在 **7.8–7.12 节**，排错在 **附录 C**。
4. 结论：**可行，公式已实测通过（0–1 px），工具链已落地；剩余为合成接入与批量。**
5. 如果只看一页，看第 0 节 + 附录 B。

---

## 0. TL;DR

- 目标：在 Toon Boom Harmony（TBH）里同时操控 2D 角色与摄像机，然后把这台摄像机**数学等价**地搬到 Blender，让 3D 环境与 2D 画面严丝合缝。
- 方向：**单向**，TBH（相机权威）→ JSON → Blender（消费方）。
- 核心公式（已实测，五类工况 0–1 px）：

  ```
  M_rel = M_h⁻¹ · M_c                 # 两者都来自 Harmony
  R, t, s = decompose(M_rel)          # 丢弃 s
  Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
  lens = 18 / (tan(fov / 2) × aspect)
  ```

- 三个已知地雷：**转向在 Peg 上、FOV 是 Function 通道、模型有隐藏 ScaleFactor。**
- 工具：`tools/TB_ExportMatrix.js`（Harmony 导出）+ `tools/import_harmony_camera.py` / 插件（Blender 导入）。

---

## 1. 项目背景与约束

### 1.1 工作流全貌

```
Blender 制作 3D 环境（成品）
        │  导出「简化版底模」——同几何、同坐标、减面
        ▼
   导入 TBH ──► 在 TBH 里排版：等比缩放 + 位移（变换在 Peg 上）
        │
        ├─► 2D 角色：绘制 / 绑定 / 高度图 / 法线图 / AOV / 烘焙接触阴影
        ├─► 摄像机动画：推、拉、摇、俯仰、roll、FOV 变化
        │
        ▼  插件：逐帧导出「相机世界矩阵 + 底模世界矩阵 + FOV」→ JSON
        │
        ▼  Blender：还原相机 → 渲 3D 环境（多通道 EXR）
        │
        ▼  After Effects / Autograph：叠 2D 角色层，重打光，最终合成
```

### 1.2 为什么方向必须是 TBH → Blender（单向）

- 相机是**表演工具**，不是渲染参数。动画师调机位时必须同时看着角色，否则构图变成隔空作业。
- 因此 3D 底模 **Blender → TBH**（给空间参照），相机 **TBH → Blender**（给正确视角）。
- 代价：3D 成为下游。机位一改，3D 就要重渲。**必须配一条同 JSON 驱动的低清预览渲染**，把「看构图」和「定稿」分开。

### 1.3 已确认事实（关键前提）

| # | 事实 | 含义 |
|---|---|---|
| F1 | Harmony 25；Blender 4.1–5.x；合成 AE（Autograph 备选） | 决定 API 与版本策略 |
| F2 | **底模 = 成品环境的简化版，同几何、同坐标** | `M_b = I`，且可做零成本像素验证 |
| F3 | TBH 里**不在模型/角色节点直接变换，一律在父 Peg 上** | 几何数据从 Peg 链读 |
| F4 | 相机挂在**复杂 Peg 层级**上；相机节点自身**未被直接变换** | 相机世界矩阵 = 相机 Peg 链累积 |
| F5 | **全程等比缩放**（导入 ScaleFactor 0.01 + Peg 等比），无非等比 | 距离补偿成立，无各向异性 |
| F6 | 推拉**同时**用「相机移动」与「FOV 变化」 | FOV 必须逐帧导出 |
| F7 | 精度要求：**数学对齐即可**，非像素级 | 容差可放宽，但误差不得随镜头放大 |
| F8 | 角色自带高度图/法线图/AOV/烘焙接触阴影（压平导出） | 角色阴影与相机解耦；重打光在合成软件做 |
| F9 | 技术栈：QtScript 或 Python → JSON → Blender Python | 两条读矩阵路径，见 4.2 |

### 1.4 范围外

- 角色绑定体系（另一条赛道）。
- 合成软件里的重打光、阴影合成细节。
- 3D 环境的建模、材质、GI 配置。

### 1.5 系统全貌：数据与职责边界

| 层 | 职责 | 不负责 |
|---|---|---|
| TBH | 机位、角色表演、排版 | 高质量 3D 渲染 |
| JSON | 单一数据契约（几何 + 投影 + 元数据） | 任何逻辑 |
| Blender | 按给定机位渲染 3D 环境与通道 | 机位决策 |
| 合成 | 叠层、重打光、最终调色 | 改机位 |

**关键设计原则：JSON 只存数据，不存逻辑。** 所有换算在消费端做，方便单独调试。

---

## 2. 核心原理

> **阅读提示（2026-11-05 补）**：本节是**理论推导**，用 `C` / `F` 表示两端的局部转换。
> 最终实测确认的具体形式是：
>
> ```
> M_rel = M_h⁻¹ · M_c
> R, t, s = decompose(M_rel)          # 丢弃 s
> Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
> ```
>
> 其中 `rotX(90°)` 对应 `C`，`×4.5` 是非正交的场景单位换算。
> **以 7.9 节为准**，本节保留推导过程供理解「为什么是距离补偿而不是 FOV」。

### 2.1 为什么「距离补偿」能精确抵消缩放

透视投影：

```
u = f · x / z
```

相机空间坐标整体乘 k：`f·(kx)/(kz) = f·x/z`，**投影不变**。

推论：**模型均匀放大 s 倍 ≡ 相机拉近 s 倍。**

- 缩放差异 → 用**距离**补偿，精确；
- 绝不用 FOV 补偿——FOV 改的是透视强度，单帧可伪装，运动时视差露馅。

### 2.2 公式推导（`M_b = I` 特例）

**Harmony 侧（field 单位）：**

```
M_h = S_f · M_peg(model) · M_node      # 底模世界矩阵
M_c = M_peg(cam)   · C                 # 相机世界矩阵
V_h = M_c⁻¹ · M_h                      # 模型在相机空间
```

- `S_f`：Scale Factor（如 0.01）。**不在 scale 坐标里**，见 4.5 陷阱 3。
- `M_peg(...)`：父 Peg 链**累积**矩阵。
- `M_node` / `C`：节点自身变换。按 F4，`C ≈ I`。

**Blender 侧（米，环境放原位）：**

```
V_b = M_cam⁻¹
```

要求画面一致（允许差一个全局均匀缩放 k）。设 `C` 为模型局部坐标的转换、`F` 为相机局部坐标的转换，则：

```
V_b = F⁻¹ · V_h · C⁻¹
⟹  M_cam = (1/k) · C · (M_h⁻¹ · M_c) · F
```

k 被吸收，于是：

```
M_cam = C · (M_h⁻¹ · M_c) · F
```

- `C`：两边 OBJ 导入器的轴差异（模型局部）。
- `F`：相机朝向约定差异（如 +Z 前向 vs −Z 前向）。

**⚠️ 右侧不是 `C⁻¹`。** 这是 2026-11-04 用 Blender 无头测试实测出的：写成 `C·(…)·C⁻¹` 会产生数百到上千像素误差，详见 7.6。

**关键：`M_h⁻¹ · M_c` 是相似变换，不是刚体。** 分解为 `(R, t, s)`：

- `R` → 旋转
- `t` → 位置（**距离补偿已在此**：S_f=0.01 时 t 放大 100 倍）
- `s` → **丢弃**（丢弃不改画面；当相机 scale 用会变 zoom，透视即假）

### 2.3 三个必须满足的条件

1. **缩放均匀**（X=Y=Z）。非均匀会留各向异性残差，针孔相机无法精确复现。
2. **几何同源**（同坐标），否则 `M_b = I` 不成立，需用 2.4 节。
3. **FOV 与几何矩阵分开**。混在一起则「缩放补偿」与「镜头缩放」无法分离。

### 2.4 通用公式（`M_b ≠ I`）

若 Blender 里的环境**不在原位**（例如被整体移动/旋转/等比缩放过），设其世界矩阵为 `M_b`：

```
M_cam = M_b · C · (M_h⁻¹ · M_c) · F
```

推导：`M_b` 相当于把整个 Blender 场景放进一个容器，相机必须跟着容器走。**`M_b` 必须是相似变换**（旋转+平移+等比缩放），否则同样触发各向异性问题。

### 2.5 边界情况

| 情况 | 处理 |
|---|---|
| 相机 roll（相机节点自身 Z 旋转） | `C` 已包含，`M_c = M_peg(cam) · C` 自动带入；**但若只读 Peg 会丢，必须连相机节点一起读** |
| 底模在镜头内被动画 | `M_h` **逐帧**导出（schema 已支持） |
| 场景里有多个相机 | 导出**当前激活/选中的**那一台，并记录节点路径 |
| FOV 恒定不变 | 仍逐帧写（或写 `fov_constant: true`），消费端统一处理 |
| TBH 使用分数帧 | 按整数帧采样即可；Blender 也按整数帧渲染 |
| 两边帧率不同 | **必须断言相等**，否则整体速度错误 |
| 相机被 parent/constraint | 赋值 `matrix_world` 会被覆盖，建议不挂父级 |
| 近远端裁剪 | 必须按场景范围设置，否则几何被切 |

---

## 3. 数据契约（JSON）

### 3.1 Schema

```json
{
  "manifest": {
    "schema": 1,
    "source": "Toon Boom Harmony 25",
    "fps": 24,
    "width": 1920,
    "height": 1080,
    "frame_start": 1,
    "frame_end": 60,
    "unit_scale": 4.5,
    "axis_rotation_deg": 90,
    "sensor_width_mm": 36,
    "fov_axis": "vertical",
    "fov_min": 41,
    "fov_max": 55,
    "camera": "Top/Camera",
    "model": "Top/Drawing",
    "frames_exported": 60,
    "bad_frames": 0
  },
  "model": { "matrix": [16 个浮点，行主序] },
  "frames": [
    { "f": 1, "cam": [16], "model": [16], "fov": 41.0 }
  ]
}
```

> 这是**实际实现**的格式（`tools/TB_ExportMatrix.js` 产出）。
> 下面的 3.2 是字段说明；3.1 早期设想的 `meta / cam_world / model_world` 命名已废弃。

### 3.2 字段说明

| 字段 | 必填 | 说明 |
|---|---|---|
| `frames[].cam` | 是 | **相机累积世界矩阵**（OGL 空间），含父 Peg 链，行主序 16 浮点 |
| `frames[].model` | 是 | **底模累积世界矩阵**；逐帧写入，天然支持模型动画 |
| `frames[].fov` | 是 | 逐帧垂直视场角（度） |
| `unit_scale` / `axis_rotation_deg` | 是 | 换算常量写进 manifest，避免消费端写死 |
| `sensor_width_mm` | 是 | 镜头传感器宽度（36mm） |
| `frames_exported` / `bad_frames` | 是 | 导出完整性与坏帧计数，便于排查 |
| `camera` / `model` | 否 | 节点路径，仅用于排查 |
| `fov_min` / `fov_max` | 否 | 快速判断 FOV 是否真的在变 |

### 3.3 批量与多镜头

- **一个镜头一个 JSON**：`shot_010_camera.json`。
- 可另加 `episode_manifest.json` 索引所有镜头与状态（待渲 / 已渲 / 已合成）。
- 文件名带上镜头号与版本，避免覆盖。

### 3.4 版本与哈希

- **当前版本尚未实现** `source_hash`（导出脚本里没有写）。建议下一步补上：
  对「相机 + 底模逐帧数据」做哈希，而不是对整个场景。
- 补上之后，Blender 渲出的 EXR 建议写入同名 hash（元数据或旁挂 `.txt`），合成端校验。
- 在实现之前，靠 `frames_exported` / `bad_frames` / `fov_min` / `fov_max` 做基本完整性检查。

---

## 4. 导出端设计（Harmony 25）

### 4.1 必须读到的四样东西

| 数据 | 来源 | 读错的后果 |
|---|---|---|
| ① 相机几何矩阵 | 相机父 Peg 链累积（含 3D 旋转） | 只读相机节点 → 俯仰/横摇全丢 |
| ② FOV | 相机节点，需勾 **Override Scene FOV**，逐帧 | 未勾 → 恒为默认；只导关键帧 → 动态 FOV 丢 |
| ③ 底模几何矩阵 | 底模父 Peg 链 + 节点自身 | 漏父级/节点自身 → 距离错 |
| ④ ScaleFactor | 底模节点隐藏字段 | 漏读 → 距离差 100 倍 |

### 4.2 两条读取路径

**路径 A：Python 接口（推荐）**

```python
from ToonBoom import harmony

sess  = harmony.session()
proj  = sess.project
scene = proj.scene

cam_peg = scene.nodes["Top/Camera-Peg"]
m = cam_peg.matrix(frame)          # 含所有父级的累积 4×4
t, s, r = m.extract_parameters_3d(harmony.Point3d())
```

- `PegNode.matrix(frame)` 返回「该 peg 及其所有父级」的累积变换。
- 反向同步：`PegNode.set_matrix(frame, matrix)`。
- 优点：一次拿到累积矩阵，避免手写连乘出错。

**路径 B：QtScript（Script Editor）**

- `node.getAttr(path, frame, attr)`，带 frame 参数避免移动播放头。
- **需手动沿父链连乘**，并自行处理 pivot / skew / quaternion。
- 先验证 `typeof JSON`，不存在则手写序列化。

### 4.3 逐帧烘焙（必须）

不能只导关键帧，原因有四：

1. 两边关键帧插值曲线形状不同；
2. Harmony 的 ease 无法映射；
3. **FOV 走 Function 通道**，普通关键帧里没有；
4. 3D 旋转可能不在相机节点上（在 Peg 上）。

做法：`for frame in range(start, end+1)` 逐帧读并写入 `frames[]`。

### 4.4 QtScript 注意事项

- ES3/ES5 级别：用 `var`，不要 `let/const`、箭头函数、模板字符串。
- 不假设有 `JSON.stringify`（先 `typeof JSON`）。
- 文件写入用 Harmony 的 `File` / `Dir` API。
- 性能：用带 frame 参数的 `getAttr` 采样。

### 4.5 陷阱

1. **只读相机节点属性 → 丢 Peg 3D 旋转**。导致「静态对、动态飘」。
2. **FOV 未开 Override Scene FOV**，读到的是场景默认值。
3. **ScaleFactor 是隐藏均匀乘数**：官方原文——若 scale 坐标是 1、Scale Factor 设为 0.5，模型渲染为一半大小，**而 scale 坐标仍是 1**。读取矩阵/属性会漏掉它。对策：能读则读，读不到就在 manifest 写死。
   **注（2026-11-04 用户确认）**：本项目实际用的是「Blender 导出时把缩放设为 0.01」，它烘进几何、由 `C` 吸收，**不需要单独读**。只有当改用 Harmony 节点上的 Scale Factor 时，才会触发本条。
4. **非均匀缩放**：数学无解，误差随视角变化。当前已确认等比（F5）；若将来引入，限制在 3% 内。
5. **Pivot 与 quaternion**：手动连乘最易错。优先用 `matrix()`。

### 4.6 错误处理与断言（导出端）

- 每帧读取后断言矩阵**无 NaN / Inf**。
- 断言 `cam_world` 线性部分可分解为「旋转 × 均匀缩放」。
- 记录 `frame_start` / `frame_end` / `fps`，与消费端核对。
- 导出结束打印摘要：帧数、FOV 范围、ScaleFactor、耗时。
- 失败时**不要写半截文件**：先写临时文件，成功后再改名。

---

## 5. 接收端设计（Blender）

### 5.1 逐帧计算

```python
import json
import math
import bpy
from mathutils import Matrix

def build_camera_matrix(cam_world_h, model_world_h, C, F):
    M = model_world_h.inverted() @ cam_world_h   # 相似变换
    loc, rot, scale = M.decompose()
    R = rot.to_matrix().to_4x4()
    rigid = Matrix.Translation(loc) @ R          # 丢弃 scale
    return C @ rigid @ F                         # 右侧是 F，不是 C⁻¹
```

### 5.2 关键帧写入

```python
cam.rotation_mode = 'QUATERNION'
prev_q = None

for entry in frames:
    mw = build_camera_matrix(entry.cam_world, entry.model_world, B)
    loc, q, sca = mw.decompose()

    if prev_q is not None and q.dot(prev_q) < 0.0:
        q.negate()                               # 符号连续
    prev_q = q

    cam.location = loc
    cam.rotation_quaternion = q
    cam.scale = (1.0, 1.0, 1.0)

    cam.keyframe_insert("location", frame=entry.f)
    cam.keyframe_insert("rotation_quaternion", frame=entry.f)

    cam.data.angle = math.radians(entry.fov)
    cam.data.keyframe_insert("angle", frame=entry.f)
```

烘焙后把全部 F-Curve 插值改为 **LINEAR**。

### 5.3 相机与场景设置

- `sensor_fit`：显式固定为标定结果，不用 AUTO。
- `clip_start` / `clip_end`：覆盖场景，避免裁剪。
- `cam.scale` 恒为 `(1,1,1)`；zoom 只走 `lens` / `angle`。
- 场景 `fps` / `resolution` / `pixel_aspect` 必须与 manifest 一致。

### 5.4 陷阱

1. **四元数符号不连续 → 抽搐**（必须强制连续）。
2. **分解出 scale ≠ 1 → 缩放没丢干净**，断言并报警。
3. **父级/约束干扰**：`matrix_world` 赋值会被覆盖。
4. **欧拉角翻转**：全程四元数。

### 5.5 批量与多镜头

- 每镜头独立执行，互不共享状态。
- 输入目录 → 输出目录一一对应；失败可重跑。
- 建议写成 CLI：`python import_camera.py shot_010_camera.json --blend env.blend`。

### 5.6 数值精度

- ScaleFactor 0.01 → 相机距离放大 100 倍。若 Harmony 相机位置在 field 量级（成百上千），**Blender 相机可能落在数百到数千单位外**。
- 影响：`clip_start` 需足够小（否则近处被切）、DOF/体积雾/置换等按物理尺度计算的参数会失真。
- Blender 对象矩阵为 32 位浮点，通常足够；**不要**在此流程里再叠加大幅缩放。
- 建议：Blender 环境保持 1 单位 = 1 米，不做额外缩放。

---

## 6. 基底变换：从「只能标定」到「直接推导」

> **本节记录一次方向修正。** v1.x 假设基底变换 `B` 无法推导、只能靠像素差穷举标定（48 种候选）。
> 2026-11-04 从 Harmony 安装目录的官方源码里直接读到了算法，随后实测确认：
> **`B` 根本不需要标定，它是确定值。**

### 6.1 最终结论（取代原标定方案）

```
Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
```

- `rotX(90°)`：Y-up → Z-up；
- `×4.5`：OGL → 场景单位；
- **没有自由参数，不需要标定，也不存在「48 选 1」。**

### 6.2 原标定方案（存档，勿再执行）

当时的思路是：正交基变换候选为 24 种旋转 × 2 种手性，
再叠加导入器翻转，几十种；用「同几何」在 Blender 渲同一帧、与 TBH 做像素差值，
选差值最小的那个，全项目复用。

**这条路之所以被废弃**：候选集里根本没有正确答案——真正的变换还包含一个非正交的
`×4.5` 缩放，纯旋转候选永远拟合不上。这也是当时「第 10 帧误差随旋转增长」的根源。

---

## 7. 验证与验收

### 7.1 静态帧

同几何两边各渲一帧，做像素差值。差值接近零 → 通过。

### 7.2 动态

20 帧「dolly + tilt + FOV」，逐帧统计误差。

### 7.3 指标计算

两种可选：

- **图像差值**：直接比像素（最直观，依赖渲染器一致性）。
- **投影点误差**：取若干已知 3D 点（如包围盒 8 角），分别投影，量像素距离（更稳健，推荐）。

### 7.4 测试矩阵

| 测试 | 目的 | 通过标准 |
|---|---|---|
| 静态单帧 | 验证 B 与 ScaleFactor | 差值 ≈ 0 |
| 纯 dolly | 验证距离补偿 | 无缩放漂移 |
| 纯 FOV | 验证 FOV 通道 | 视角一致 |
| tilt | 验证 Peg 3D 旋转 | 无偏移 |
| roll | 验证相机节点自身旋转 | 无偏移 |
| 组合 | 最终验收 | 误差不随帧增长 |

### 7.5 判据

- 目标为**数学对齐**（F7），非像素级。
- 参考阈值：投影点误差 < 0.5 px。
- 真正要看的是：**「相对滑动」是否消失**——最初要解决的问题。

### 7.6 已完成的验证（2026-11-04）

用 Blender 5.2 无头模式 + 真实投影函数 `world_to_camera_view`，造了一个「Harmony 侧」场景（模型带 0.01 缩放 + 旋转 + 平移，相机带旋转），比较三个候选公式：

| 候选 | 最大像素误差 | 结论 |
|---|---|---|
| `C · M_h⁻¹ · M_c` | 0.0005 px | ✅ 正确 |
| `B · (M_h⁻¹ · M_c) · B⁻¹` | 241 / 6245 px | ❌ 错误（旧版文档写法） |
| `M_h⁻¹ · M_c`（C = I） | 0.0001 px | ✅ 正确 |

**结论：公式右侧必须是相机局部转换 `F`，不是 `C⁻¹`。** 这是旧版文档的一个实质性错误，已在本版修正。

复现脚本：`E:\text\verification\test_receiver.py`

### 7.7 实机测试记录（Harmony 25 + Blender 5.2，2026-11-04）

用真实 Harmony 工程（立方体 + 平面 + 3D 相机 Peg）导出探针 JSON，与 Harmony 的 OpenGL 帧渲染对比。

**已验证：**

- 探针可逐帧读取相机矩阵、FOV、near/far、模型矩阵（60 帧齐全）；
- 模型矩阵为单位阵（Peg 未动）；
- FOV 恒定 41.112°（Override 关闭，符合预期）；
- 相机矩阵 = `Peg 矩阵 × T(0,0,offset_z/4.5)`，两帧逐位吻合（offset_z = 12）。

**已排除：**

| 假设 | 结果 |
|---|---|
| FOV 动画未读到 | 排除（FOV 本就恒定） |
| Quaternion 旋转模式 | 排除（切 Euler 后误差未改善） |
| 视图矩阵 vs 姿态矩阵 | 排除（反解会到相机后方） |
| 24 种轴转换 | 排除（无一能同时对上两帧） |
| 读 Peg 分离属性重建 | 等价于原矩阵，不会改变结果 |

**官方桥接算法（从 Harmony 25 安装目录源码中提取，2026-11-04）：**

`resources/scripts/externalRendering/blender/TB_BlenderUtilities.py` 中的 `setupCamera()` / `correctModelsToCamera()`：

```python
rotX = Matrix.Rotation(radians(90), 4, 'X')   # 轴重映射：Y-up -> Z-up
targetMatrix = rotX @ cameraMatrix

t, q, s = targetMatrix.decompose()
orthoMatrix = Matrix.LocRotScale(t, q, None)  # 相机只保留正交部分

# 非正交残差（scale / shear / skew）交给模型组补偿
residual = orthoMatrix @ targetMatrix.inverted_safe()
```

- `setupCameraProjection()`：`lens = 36 / (0.75 * aspectRatio)` → 16:9 时 27mm，
  对应**垂直 FOV 41.1°**，与探针读到的 41.112 一致。
- 但官方传入的 `cameraMatrix` 来自渲染节点上下文
  (`context.input(0).cameraMatrix()`，见 `TB_ExternalRenderingUtilities.js`)，
  **不是** `scene.getCameraMatrix()`（OGL 空间）。脚本编辑器里拿不到该上下文，
  这是自研管线目前无法复现官方转换的根本原因。

**官方 config 真值（2026-11-04，Render Blender Batch Node 实测）：**

```
cameraMatrix = [0.2222,0,0,0,  0,0.2222,0,0.8238,  0,0,0.2222,2.6667,  0,0,0,1]
modelMatrix  = [0.2222,0,0,0,  0,0.2222,0,0,      0,0,0.2222,0,       0,0,0,1]
```

- `0.2222 = 1/4.5`：官方把相机与模型的线性部分统一乘 1/4.5；
- 平移 (0, 0.8238, 2.6667) 与 `getCameraMatrix` **完全一致**；
- 结论：**`getCameraMatrix` 的平移可用，线性部分需乘 1/4.5，再按 `rotX = Rx(90°)` 转到 Blender 轴**；
- 官方 Blender 端的 `ortho/residual` 拆分最终把平移放大回 4.5 倍（场景单位），模型回到原尺寸。

**验证结果：** 按上述换算，第 1 帧立方体近面投影 261.8×261.8 px，
与实测 261×224 的宽度差 0.8 px、中心差 0.5/1.3 px；
高度差 37.8 px 来自「立方体底部沉入平面 0.277 被遮挡」，不是投影误差。

> **结论：换算公式已验证通过（见 7.8–7.12）。**

复现脚本：`E:\text\verification\`（TB_Probe.js / compare_clean.py / clusters.py）

### 7.8 最终验证通过（2026-11-05）

用一套全新的干净场景（2m 立方体 + 偏移 1.48m 的标记立方体 + 7.18m 平面）重测，
按最终公式渲染，并与 Harmony 的 OpenGL 预览逐块对比：

| 帧 | 我们的渲染 | Harmony 预览 | 差 |
|---|---|---|---|
| 1（大立方体） | 宽 261.0，中心 959.5 | 宽 261.0，中心 959.5 | **0 px** |
| 1（整体轮廓） | x=[354,1565] | x=[351,1568] | ≤3 px |
| 10（偏航 48.8°，整体轮廓） | x=[429,772] | x=[430,772] | ≤1 px |

**最终公式：**

```
Blender 相机 = rotX(90°) × ( getCameraMatrix 线性部分 × 1/4.5 )，平移不变
Blender 镜头 = 27mm / 36mm 传感器（垂直 FOV 41.112°，方形像素）
Blender 模型 = 原尺寸（相机的 4.5 倍补偿与模型的 1/4.5 互相抵消）
```

> 这是 **模型未变换（M_h = I）时的特例**；含模型位移/缩放的完整式见 7.9。

**复盘：** 之前第 10 帧 67–105 px 的误差，来自三件事叠加：
旧场景的相机被重设过、预览图与探针不同帧、以及用「亮度阈值包围盒」当指标
（平光渲染与 Harmony 的明暗面不同，比出来的是假信号）。

**指标经验：** 用整体轮廓或逐顶点投影；不要用单张图的亮面包围盒。

### 7.9 模型移动 + 缩放 + 10 帧序列验证（2026-11-05 下午）

在 7.8 的基础上，把底模的 Peg 改成 **位置 (2, 1, 0)、缩放 1.5（等比）**，
相机继续做 0° → 48.8° 的偏航，逐帧对比 1–10 帧（用透明背景的 alpha 当轮廓）：

| 帧 | 我们的渲染 | Harmony | 差 |
|---|---|---|---|
| 1 | (8, 1919, 545, 957) | (8, 1919, 545, 957) | 0 |
| 2–5 | — | — | 0 |
| 6 | (0, 1602, 560, 1079) | (0, 1603, 560, 1079) | −1 |
| 7–8 | — | — | 0 |
| 9 | (0, 1571, 567, 1079) | (0, 1572, 567, 1079) | −1 |
| 10 | (0, 1544, 569, 1075) | (0, 1544, 569, 1075) | 0 |

**10/10 帧，误差 0–1 px。**

**最终完整公式（含模型变换）：**

```
M_rel = M_h⁻¹ · M_c                 # 两者都来自 Harmony（OGL 空间）
R, t, s = decompose(M_rel)          # s 丢弃
Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
Blender 镜头 = 27mm / 36mm 传感器（垂直 FOV 41.112°，方形像素）
Blender 模型 = 原尺寸（模型矩阵不折进相机，由 M_h⁻¹ 抵消）
```

**关键教训：** `M_h⁻¹` 得到的平移必须**再乘 4.5** 才是场景单位；
漏掉这一步，相机会被压缩到模型内部（整屏发白）。

### 7.10 动态 FOV 验证（2026-11-05 下午）

在同一场景上打开相机节点的 **Override Scene FOV**，把 FOV 从 **41° 动画到 55°**
（10 帧线性），模型仍为「位置 (2,1,0) + 缩放 1.5」，相机继续偏航：

| 帧 | FOV | 我们的渲染 | Harmony | 差 |
|---|---|---|---|---|
| 1 | 41.0 | (5, 1919, 545, 959) | (5, 1919, 545, 959) | 0 |
| 6 | 48.8 | (0, 1491, 556, 1022) | (0, 1491, 556, 1022) | 0 |
| 8 | 51.9 | (0, 1445, 559, 1003) | (0, 1447, 559, 1003) | −2 |
| 10 | 55.0 | (121, 1379, 561, 931) | (121, 1379, 561, 931) | 0 |

**10/10 帧，误差 0–2 px。**

FOV → Blender 焦距的换算（36mm 传感器、方形像素）：

```
lens = 18 / ( tan(fov / 2) × aspect )
```

- `fov` 为相机节点逐帧读出的**垂直**视场角（度）；
- `aspect = 宽 / 高`（1920/1080）；
- 默认 41.112° 时 `lens = 27mm`，与官方 `36/(0.75×aspect)` 公式一致。

> **到此四类工况通过：相机运动、模型位移、模型缩放、动态 FOV。**（旋转见下一节）

### 7.11 模型三轴旋转验证（2026-11-05 下午）

把底模 Peg 再加 **X=20°、Y=15°、Z=30°** 的三轴旋转，其余不变，逐帧对比：

| 帧 | 立方体（隐藏平面后） | Harmony | 差 |
|---|---|---|---|
| 1–8（含平面） | — | — | 0–1 px |
| 9 | (494, 964, 497, 927) | (494, 964, 497, 928) | 0 / 0 / 0 / −1 |
| 10 | (494, 938, 502, 903) | (494, 938, 502, 903) | 0 |

**10/10 帧，立方体分毫不差。**

**发现一个渲染器差异（不是相机问题）：** 相机运动到平面下方时，
Harmony 对单面平面做**背面剔除**（不渲染），而 Blender 默认**双面渲染**，
于是多出一条几乎侧对镜头的细线，把包围盒撑大。
对比时要么隐藏平面，要么只比立方体。

> **五类工况全部通过：相机运动、模型位移、模型缩放、模型旋转、动态 FOV。**

### 7.12 工具链闭环（2026-11-05 晚）

两个正式工具已就位并真实跑通：

| 端 | 文件 | 作用 |
|---|---|---|
| Harmony | `tools/TB_ExportMatrix.js` | 逐帧导出相机矩阵、模型矩阵、FOV，写 manifest JSON |
| Blender | `tools/import_harmony_camera.py` | 读 JSON，逐帧建相机、设焦距、关键帧 |

**真实链路测试**（Harmony 导出 `1.json` → Blender 导入 → 渲染）：

| 帧 | Blender | Harmony | 差 |
|---|---|---|---|
| 1 | (296, 1919, 383, 1079) | 同左 | 0 |
| 3 | (347, 1919, 390, 1079) | 同左 | 0 |
| 5 | (323, 1860, 397, 1079) | 同左 | 0 |
| 8 | (163, 1237, 409, 1079) | (163, 1237, 408, 1079) | 0/0/1/0 |
| 9 | (494, 964, 497, 927) | (494, 964, 497, 928) | 0/0/0/−1 |
| 10 | (494, 938, 502, 903) | 同左 | 0 |

**6/6 帧，误差 0–1 px。**

说明：
- 导出的相机矩阵线性部分带轻微非均匀缩放（如 0.997/0.997/1），
  导入端分解时取正交部分后结果仍然精确，无需特殊处理；
- 导入端兼容 Blender 4.x 与 5.x 的 Action API（5.x 起 `action.fcurves` 已移除）。

---

## 8. 已知陷阱总表

| # | 陷阱 | 后果 | 对策 |
|---|---|---|---|
| 1 | 只读相机节点属性 | 丢 Peg 3D 旋转 | 读累积矩阵 |
| 2 | FOV 走 Function 通道 | 只导关键帧会丢 | 逐帧烘焙 |
| 3 | ScaleFactor 隐藏 | 距离差 100 倍 | 乘进矩阵或写死 manifest |
| 4 | 非均匀缩放 | 各向异性，无解 | 限制等比 |
| 5 | B 靠推导 | 系统性错位 | 同几何像素差值标定 |
| 6 | 四元数符号跳变 | 相机抽搐 | 强制符号连续 |
| 7 | 贝塞尔插值 | 逐帧数据被过冲 | F-Curve 改 LINEAR |
| 8 | 相机 scale 混入 zoom | 透视假 | 相机 scale 恒为 1 |
| 9 | 近/远端裁剪 | 几何被切 | 设置 clip |
| 10 | fps / 分辨率 / 宽高比不一致 | 系统性偏移 | manifest 断言 |
| 11 | TBH 渲 2D 时未隐藏 3D 底模 | 重影 | 渲染前检查 |
| 12 | 使用过期 3D 渲染 | 与 2D 不匹配 | source_hash 校验 |
| 13 | 帧号偏移 | 整体错位 | manifest 记录 frame_start |
| 14 | 导出中断写坏文件 | JSON 损坏 | 临时文件 + 原子改名 |
| 15 | 多相机场景读错相机 | 视角错误 | 记录节点路径并校验 |

---

## 9. 原「待验证清单」的确认结果（全部已落地）

早期需要在 Harmony 里逐条点开确认的项目，现均已实测确认：

| # | 项目 | 结论 |
|---|---|---|
| 1 | QtScript 是否有 `JSON.stringify` | ✅ 有（Harmony 25） |
| 2 | 相机父 Peg 的 Enable 3D / 旋转模式 | ✅ 需 Enable 3D；Euler 或 Quaternion 都可 |
| 3 | 相机是否勾 Override Scene FOV | ✅ 勾了才能逐帧读到变化的 `fov` |
| 4 | ScaleFactor 能否被脚本读到 | ✅ 读得到（`ASSET.SCALE_FACTOR`） |
| 5 | 累积矩阵能否一次拿到 | ✅ `scene.getCameraMatrix(f)` / `node.getMatrix(path, f)` |
| 6 | 「B 的标定」 | ✅ 不需要标定——官方 `rotX(90°)` + `×1/4.5` 已足够 |
| 7 | fps / 分辨率 / 宽高比 / 帧范围 | ✅ 由 manifest 自动带入 |

**已确认**：相机节点自身未被直接变换。

---

## 10. 实施进度

| 阶段 | 内容 | 状态 |
|---|---|---|
| POC | 立方体 + Peg 相机 + 单帧 | ✅ 完成（7.6） |
| 运动验证 | 多帧旋转 / 位移 / 缩放 / FOV | ✅ 完成（7.8–7.11） |
| 工具化 | 导出 JS + 导入 Python / 插件 | ✅ 完成（7.12） |
| 镜头级 | 接入角色层与真实环境（合成） | ❌ 未开始 |
| 管线化 | manifest、哈希、批量、异常处理 | 🔶 manifest 已有，其余待做 |

**剩余风险只集中在合成与批量**，换算部分已不再是风险项。

---

## 11. 与官方 Render Blender Batch Node 的关系

Harmony 24+ 自带 **Render Blender Batch Node**：它把你的 `.blend` 作为渲染后端，把场景里对 3D 模型及其子节点的变换应用上去，再送去 Blender 渲染，结果合成回 Harmony。

**为什么本方案仍然要做自定义管线：**

| 维度 | 官方节点 | 本方案 |
|---|---|---|
| 渲染结果去向 | 合成回 Harmony 内部 | 独立 EXR，进 AE/Autograph |
| 多通道 / 重打光 | 受限 | 完全可控 |
| 2D 层与 3D 的分层策略 | 受 Harmony 结构约束 | 自由 |
| 机位数据可复用性 | 封闭 | JSON 可复用到任何工具 |

**但仍有两个用途：**

1. **验证参照**：它证明 Toon Boom 自己有一套 Harmony→Blender 的变换约定，可用来**反推或校验 B**。
2. **快速预览**：某些镜头可以直接用它出低清预览。

---

## 12. 范围外 / 后续赛道

- **角色绑定体系**：复杂，另议。不改变相机侧数学（解耦），但影响合成与阴影侧。
- **合成软件**：默认 AE（2D 阴影 + 高度/法线图重打光成熟）；Autograph 作「逃生舱」。
- **角色地面投影阴影**：以绑定阶段烘焙的接触阴影为主；必要时 Blender Shadow Pass + 遮罩。长投影/硬光镜头是唯一需额外处理的情况。

---

## 13. 合成篇（后续讨论）

> **状态：部分已定、部分待定。** 本节只记录问题与已知方向，不下结论。
> **完整展开见姊妹篇：[TBH-Blender-合成篇-完整技术文档](TBH-Blender-合成篇-完整技术文档.md)**（含风格匹配、校准球、GLSL 方案）。

### 13.1 遮挡（方案已定）

- **做法**：Blender 单独导出前景物体（建筑、树、道具）的 matte，在合成软件中遮住 2D 角色。
- **要求**：
  - matte **逐帧**导出——相机一动，前景 alpha 每帧都在变；
  - 必须带抗锯齿，否则角色边缘会被硬切出锯齿；
  - 若角色的倒影/阴影也要被前景遮挡，matte 必须覆盖该区域；
  - 与角色层使用相同的分辨率与宽高比。

### 13.2 色彩管理（待定）

- **问题**：Blender 输出 EXR 为线性（scene-referred），TBH 输出 sRGB（display-referred），合成软件自身还有色彩空间设置。三者不一致会导致暗部与饱和色偏移。
- **已知方向**：统一到线性工作空间合成，最后输出时再转显示空间；或让 Blender 的 view transform 直接对齐目标。
- **待定**：具体路线、三个软件各自的设置。**这一块需要系统学习后再定，不能拍脑袋。**

### 13.3 Alpha 方式（待定）

- **问题**：premultiplied 与 straight alpha 混用会在边缘产生黑边/白边。
- **待定**：TBH 输出、Blender 的 EXR、合成工程三处分别是什么，需实测确认后统一。

### 13.4 重打光（待定）

- **现状**：角色自带高度图、法线图、AOV，在合成软件中做 2D 重打光。
- **待定**：
  - 具体工具（AE 法线重打光插件，候选：Normality；或 Autograph 原生功能）；
  - 光方向是「每镜头手调固定值」，还是「跟随场景光 / 相机数据」；
  - 与 3D 环境受光的一致性如何保证。

### 13.5 运动模糊与景深（待定）

- **问题**：Blender 若开运动模糊 / DOF，2D 角色没有对应效果，合成时可能割裂。
- **待定**：两边都关、合成侧统一加、或接受差异。

### 13.6 合成技术清单（参考）

| 技术 | 作用 | 常见工具 | 输入 | 主要风险 |
|---|---|---|---|---|
| **法线重打光** | 让 2D 角色受 3D 光 | Normality（AE，免费，Stefan Minning）；或自建 shader | 底色 + 法线图（同尺寸） | 光方向与场景光不一致；法线切线空间不对；边缘 premult |
| **Toon shading + Rim light** | 统一 2D/3D 卡通感 | Normality 内置 / 曲线调色 | 重打光结果 | 过曝、脏边 |
| **阴影 Pass** | 角色/环境投影 | Blender Shadow Catcher / AOV；TBH 烘焙 AO | 阴影通道 | 与角色接触点不一致 |
| **深度 DOF** | 景深一致性 | AE 景深效果 + Z 通道 | Z pass / 深度图 | 角色无深度 → 虚化不一致 |
| **Matte / Holdout** | 前景遮挡角色 | Blender Object Index / Cryptomatte | 逐帧 matte | 硬边、漏掉倒影 |
| **色彩管理** | 统一色彩空间 | 见 13.2 | —— | 暗部/饱和色偏移 |
| **Alpha 处理** | 正确叠层 | 见 13.3 | —— | 黑边/白边 |

**法线重打光的典型流程**（以 Normality 为例）：

1. 准备法线图，与角色底图**同尺寸**；
2. 预合成（Pre-Compose，勾 Move all attributes），法线层设为隐藏；
3. Normality 中指定法线层，建点光 / 方向光；
4. 调 rim light 颜色与角度、toon shading、半调/噪点；
5. 光方向必须与 3D 场景主光一致；**相机 roll/tilt 时屏幕空间光方向会变**，需逐镜头或逐帧调整。

> 参考：[80.lv · 2D 角色 3D 打光教程](https://80.lv./articles/learn-how-to-set-up-3d-lighting-for-2d-characters-with-this-after-effects-tutorial)

### 13.7 已排除

- **眼睛图层导出丢失**：经确认为操作失误，非管线缺陷，不列为风险。

---

## 14. 环境与依赖

| 项 | 要求 | 备注 |
|---|---|---|
| Harmony | 24–27（本项目 25） | Python 接口需启用 |
| Blender | 4.1–5.x | 相机 API 稳定；Eevee/GI 各版本差异大 |
| Python（Blender 侧） | 随 Blender 版本 | 用 Blender 自带解释器 |
| Python（Harmony 侧） | 随 Harmony 安装 | `from ToonBoom import harmony` |
| 导出脚本形式 | 独立脚本 / CLI 优先 | 便于批处理与重跑 |
| 合成软件 | AE（主）/ Autograph（备） | 见第 12、13 节 |

**版本提示**：

- Blender 4.2+ 的 Eevee 改为光追 GI，与 4.1 的旧 SSGI 设置不兼容，**建议钉一个版本**。
- Harmony 25 的 Python 接口与 24 基本一致，具体方法名以实际版本文档为准。

---

## 15. 交接验收清单

新接手者按顺序勾选，全部通过才算交接完成。
**下列相机相关项已于 2026-11-05 全部通过**（打勾为实测结果）：

- [x] 能在 Harmony 中选中相机，打印出矩阵的 4×4 数值
- [x] 能读出该相机的 FOV，且 Override Scene FOV 已勾选
- [x] 能读出底模累积矩阵，并确认 ScaleFactor 为 0.01
- [x] 导出端能逐帧写出 JSON，`frames` 数量 = 帧数
- [x] Blender 端能读 JSON，关键帧数量正确
- [x] 静态单帧叠图差值接近零（0 px）
- [x] 多帧 dolly + tilt + 旋转 + 位移 + 缩放 + FOV 的误差不随帧增长（0–1 px）
- [x] 相机 F-Curve 全部为 LINEAR
- [x] 四元数无符号跳变（不出现 180° 翻转）
- [x] manifest 的 fps / 分辨率 / 宽高比与工程一致
- [ ] **合成**：TBH 角色层 + Blender 环境层叠合，角色不飘不滑（未开始）

---

## 16. 参考文档

- [Camera Node（相机节点参数）](https://docs.toonboom.com/help/harmony-27/premium/reference/node/move/camera-node.html)
- [Creating 3D Camera Movements（3D 相机运动官方做法）](https://docs.toonboom.com/help/harmony-24/premium/3d-space/3d-camera-move.html)
- [About the Scale Factor of 3D Objects（ScaleFactor 机制）](https://docs.toonboom.com/help/harmony-27/premium/3d-integration/about-3d-model-scale-factor.html)
- [Setting the Default Scale Factor for 3D Models](https://docs.toonboom.com/help/harmony-27/premium/3d-integration/set-default-scale-factor.html)
- [Working in 3D（3D 模型导入格式与外部渲染）](https://docs.toonboom.com/help/harmony-27/premium/getting-started/working-in-3d.html)
- [PegNode（Python 接口，matrix() / set_matrix()）](https://docs.toonboom.com/help/harmony-24/scripting/pythonmodule/class_o_m_c_1_1_peg_node.html)
- [Render Blender Batch Node（官方 Blender 渲染通道）](https://docs.toonboom.com/help/harmony-24/premium/3d-integration/render-blender-batch-node.html)

---

## 附录 A：术语

| 术语 | 含义 |
|---|---|
| 底模 / 代理模型 | 从 Blender 导出、导入 TBH 做参照的简化模型 |
| Peg | Harmony 的变换节点，通常承载动画变换 |
| Scale Factor | Harmony 3D 模型的隐藏均匀缩放系数，不在 scale 坐标里 |
| FOV | 视场角；Harmony 中需 Override 生效，靠 Function 动画 |
| B（基底变换） | 早期对「两端坐标转换」的占位符；现已确定为 `rotX(90°)` + `×4.5`，无常量待标定 |
| 数学对齐 | 相机几何精确对应；实测可达 0–1 px，渲染器差异（如背面剔除）另算 |
| 相似变换 | 旋转 + 平移 + **均匀**缩放；本方案里必须能分解出这三者 |

---

## 附录 B：公式速查

> **最终版（实测通过，以此为准）**
>
> ```
> M_rel = M_h⁻¹ · M_c
> R, t, s = decompose(M_rel)          # 丢弃 s
> Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
> lens = 18 / (tan(fov / 2) × aspect)
> ```
>
> 下面是 v1.x 的理论形式，`C` / `F` 是占位符，等价关系见 2 节开头的提示。

```
M_h   = S_f · M_peg(model) · M_node        # 底模世界矩阵（含隐藏缩放）
M_c   = M_peg(cam) · C                     # 相机世界矩阵
M_cam = C · (M_h⁻¹ · M_c) · F              # 环境放原位时

M_cam = M_b · C · (M_h⁻¹ · M_c) · F        # 环境不在原位时

分解：M_h⁻¹·M_c = (R, t, s)
      R → 旋转；t → 位置（含距离补偿）；s → 丢弃
      FOV 独立，不参与此矩阵
```

---

## 附录 C：排错决策树

```
画面整体偏移（平移）
 ├─ 所有帧都偏 → B 错，或分辨率/宽高比不一致
 └─ 某些帧偏 → 帧号偏移 / fps 不一致

画面大小不对（缩放）
 ├─ 恒差 100 倍 → ScaleFactor 漏读
 ├─ 随帧渐变 → 距离补偿算错（t 被当 scale 处理）
 └─ 横向拉伸 → 存在非均匀缩放

静态对、动态飘
 ├─ 一 tilt 就错 → 只读了相机节点，没读 Peg 链
 ├─ 一 FOV 变就错 → FOV 未逐帧导 / 未开 Override
 └─ 中间帧错、关键帧对 → 只导了关键帧，没逐帧烘焙

相机抽搐
 ├─ 角度突然翻转 → 四元数符号不连续
 └─ 速度忽快忽慢 → 插值曲线是贝塞尔，改 LINEAR

几何被切
 └─ 近/远端裁剪面没设

重影
 └─ TBH 渲 2D 时没隐藏 3D 底模
```

---

## 附录 D：接收端参考伪代码（带断言）

```python
def import_shot(json_path, blend_path, basis):
    data = load_json(json_path)
    meta = data["meta"]

    assert_meta(meta)                        # fps/分辨率/宽高比

    cam = ensure_camera(blend_path)
    cam.rotation_mode = 'QUATERNION'
    cam.scale = (1, 1, 1)

    prev_q = None
    for entry in data["frames"]:
        M = Matrix(entry["model_world"]).inverted() @ Matrix(entry["cam_world"])

        # 断言：线性部分必须是「旋转 × 均匀缩放」
        loc, rot, sca = M.decompose()
        assert abs(sca.x - sca.y) < 1e-4 and abs(sca.y - sca.z) < 1e-4, \
            "非均匀缩放，模型数据有问题"

        R = rot.to_matrix().to_4x4()
        rigid = Matrix.Translation(loc) @ R
        mw = basis_left @ rigid @ basis_right   # 左 = C，右 = F（不是 C⁻¹）

        loc2, q, _ = mw.decompose()
        if prev_q and q.dot(prev_q) < 0:
            q.negate()
        prev_q = q

        cam.location = loc2
        cam.rotation_quaternion = q
        cam.keyframe_insert("location", frame=entry["f"])
        cam.keyframe_insert("rotation_quaternion", frame=entry["f"])

        cam.data.angle = math.radians(entry["fov"])
        cam.data.keyframe_insert("angle", frame=entry["f"])

    set_fcurves_linear(cam)
```
