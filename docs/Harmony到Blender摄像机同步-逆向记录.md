# 从猜公式到读源码：一次 Toon Boom Harmony → Blender 摄像机同步的逆向记录

> 这是一次真实的技术排查记录，从「以为懂了」到「发现全错」再到「从官方源码里找到答案」。
> 软件环境：Toon Boom Harmony 25 Premium / Blender 5.2 LTS / Windows。
> 状态：**已跑通。** 五类工况（相机运动、模型位移、等比缩放、三轴旋转、动态 FOV），10 帧对比误差 **0–1 px**。

---

## 一、目标

在 Harmony 里做 2D 角色动画和摄像机运动，把摄像机数据导出到 Blender，让 Blender 渲染的 3D 环境和 Harmony 的 2D 画面严丝合缝。

技术上要解决的是：**给定 Harmony 的相机矩阵和 FOV，如何构造一个 Blender 相机，使它看到同一个画面。**

---

## 二、最初的方案（以及它错在哪）

最初的想法很自然：

1. 从 Harmony 读相机矩阵 `M_c` 和模型矩阵 `M_h`；
2. 算相对变换 `M_h⁻¹ · M_c`；
3. 做个基底转换 `B`，得到 Blender 相机：

```
M_cam = B · (M_h⁻¹ · M_c) · B⁻¹
```

听起来没问题，甚至在**第一帧**（相机无旋转）能完美对上。

但一加旋转就崩：

| 帧 | 宽度误差 | 水平误差 |
|---|---|---|
| 1（无旋转） | 0 px | 0.5 px |
| 10（旋转 51°） | **+67 px** | **−33 px** |

**误差随旋转增长**——这不是标定误差，是模型错误。

---

## 三、七条被排除的路线

在找到正确方法之前，我们依次排除了七种假设。这段是整件事最有价值的部分：**排除法本身也是信息**。

| # | 假设 | 怎么排掉的 |
|---|---|---|
| 1 | FOV 动画没读到 | FOV 从第 1 帧到末帧恒定 41.112°，本来就没人动过 |
| 2 | Quaternion 旋转模式导致 | 把相机 Peg 切成 Euler 后重测，误差没改善 |
| 3 | 相机矩阵是「视图矩阵」而非「姿态」 | 反解后相机跑到物体后面去了，不成立 |
| 4 | 24 种轴转换里有对的 | 穷举全部 24 种，没有一种能同时对上两帧 |
| 5 | 读 Peg 的分离属性重建矩阵 | 验算后发现矩阵 = Peg × 偏移，信息量相同，重建无意义 |
| 6 | 从渲染图反解缩放/宽高比 | 这是「拟合」不是「验证」，得到的参数只能凑一帧 |
| 7 | 非均匀空间的共轭变换 | 正共轭、反共轭都试过，误差更大 |

绕了一大圈之后，我们承认：**黑盒逆向到头了。**

---

## 四、转折点：官方源码

真正的突破来自一个想法——**先看看厂商自己是怎么做的。**

Harmony 25 的安装目录里，藏着一整套与 Blender 通信的脚本：

```
resources/scripts/externalRendering/blender/
    TB_BlenderClient.py      (23 KB)
    TB_BlenderInstance.py    (17 KB)
    TB_BlenderUtilities.py   (45 KB)
```

打开 `TB_BlenderUtilities.py`，答案就写在里面：

```python
def setupCamera(cameraMatrix, cameraGroupName, isBaking):
    cameraMatrix = mathutils.Matrix([cameraMatrix[0:4], cameraMatrix[4:8],
                                     cameraMatrix[8:12], cameraMatrix[12:16]])

    rotX = mathutils.Matrix.Rotation(math.radians(90), 4, 'X')  # 重映射到 Blender 坐标
    targetMatrix = rotX @ cameraMatrix

    # 相机只取正交部分，因为 Blender 相机不支持 scale / shear / skew
    translation, quaternion, scale = targetMatrix.decompose()
    orthoMatrix = mathutils.Matrix.LocRotScale(translation, quaternion, None)

    # 非正交残差交给模型组去补偿
```

以及投影设置：

```python
def setupCameraProjection(orthoProjection, aspectRatio):
    cameraObject.data.lens = 36 / (0.75 * aspectRatio)   # 16:9 → 27mm
```

这解释了两件事：

- 轴转换是 **`Rx(+90°)`**；
- 16:9 下 `lens = 27mm`，对应**垂直 FOV 41.1°**，且是**方形像素**。

---

## 五、真正的数据源

但还有个问题：官方传给 Blender 的 `cameraMatrix` 是从哪来的？

答案在 `TB_ExternalRenderingUtilities.js`：

```js
cameraMatrixObj = context.input(0).cameraMatrix();   // 渲染节点上下文
```

**不是** `scene.getCameraMatrix()`——后者返回的是 **OGL 空间**的矩阵，带着非均匀缩放；而渲染节点上下文给的是**场景空间**的矩阵。

两者差多少？我们后来抓到了真值。

---

## 六、拿到真值：`_config.json`

官方在渲染前会把整个配置写成 JSON 落盘（`TB_BlenderInstance.py`）：

```python
configFilePath = self.__outputDstPathBase + "_config.json"
outfile.write(json.dumps(self.__config))
```

于是只要用一次 `Render Blender Batch Node`，就能拿到 Harmony 内部真正使用的相机矩阵：

```json
cameraMatrix:
[0.2222, 0, 0, 0,
 0, 0.2222, 0, 0.823792311054366,
 0, 0, 0.2222, 2.6666717529296875,
 0, 0, 0, 1]

modelMatrix:
[0.2222, 0, 0, 0,
 0, 0.2222, 0, 0,
 0, 0, 0.2222, 0,
 0, 0, 0, 1]
```

关键发现：

- **`0.2222 = 1 / 4.5`**，线性部分被统一缩放了；
- **平移 (0, 0.8238, 2.6667) 与我们用 `getCameraMatrix` 读到的完全一致**；
- 也就是说：官方矩阵 = `getCameraMatrix` 的线性部分 × 1/4.5，平移不变。

第 10 帧的官方矩阵同样验证了这一点——旋转部分模长正好 0.2222，偏航角 51.4°，和平移都与我们的探针数据一一对应。

---

## 七、最终换算

把上面的东西拼起来：

```
Blender 相机 = rotX(90°) × ( getCameraMatrix 的线性部分 × 1/4.5 )
Blender 镜头 = 27mm（36mm 传感器 / 垂直 FOV 41.1° / 方形像素）
Blender 模型 = 原尺寸（与相机的 4.5 倍补偿互相抵消）
```

**第 1 帧的验证结果：**

| 量 | 我们的渲染 | Harmony 预览 | 差 |
|---|---|---|---|
| 立方体宽度 | 261 px | 261 px | **0** |
| 水平中心 | 959.5 | 959.5 | **0** |
| 高度 | 264 px | 224 px | 40 px（遮挡造成） |

高度那 40 px 不是误差：立方体模型底部沉进地面 0.277，渲染时被地面挡住，亮度掩膜框不到。

---

## 八、那 67 px 其实是假信号

在换干净场景重测之前，第 10 帧一直差 67 px。最后一次同帧重测才发现，是三件事叠在一起：

1. **相机被重置过**：切换 Peg 的旋转模式会清空关键帧，探针和预览不是同一个状态；
2. **不同帧**：预览图是 22:10 导的，探针是 22:31 跑的，中间相机被改过；
3. **指标选错了**：用「亮度阈值包围盒」量立方体。我们的渲染是平光（整个轮廓都亮），Harmony 是带明暗的（只有亮面被框到），比出来的宽度天然不同。

**教训：数据必须同帧同状态，指标必须比整体轮廓，而不是亮面。**

---

## 九、方法论上的几点收获

1. **先读源码，再逆向。** 厂商自己的实现永远是最可靠的文档。我们绕了两小时，答案就在安装目录里。
2. **区分坐标空间。** `getCameraMatrix` 是 OGL 空间，渲染节点上下文是场景空间，两者相差一个非均匀缩放（9 / 12 / 4.5）。混用必错。
3. **对称测试物会骗人。** 用立方体做验证，转 0° 和转 90° 在画面上完全一样，掩盖了旋转错误。应该用不对称标记（比如偏移放一个小方块）。
4. **标定一帧、验证另一帧。** 只在单帧上拟合参数，永远能凑出「完美结果」。多帧才是真假试金石。
5. **bbox 不是可靠指标。** 遮挡、暗面、不同渲染器的着色差异，都会让包围盒失真。整体轮廓或逐点投影更可靠。
6. **数据要同帧同状态。** 中间改过相机、换过旋转模式，却不重新采集数据，误差会被误判成公式问题。

---

## 十、最终验证

**第一轮（模型在原点）：** 2m 立方体 + 偏移 1.48m 的标记立方体 + 7.18m 平面，相机偏航 0°→48.8°。

| 帧 | 我们的渲染 | Harmony 预览 | 差 |
|---|---|---|---|
| 1（大立方体） | 宽 261.0，中心 959.5 | 宽 261.0，中心 959.5 | **0 px** |
| 1（整体轮廓） | x=[354,1565] | x=[351,1568] | ≤3 px |
| 10（偏航 48.8°） | x=[429,772] | x=[430,772] | ≤1 px |

**第二轮（模型被移动 + 放大 1.5 倍，这才是真实工况）：**

底模 Peg 设为位置 (2, 1, 0)、缩放 1.5，逐帧对比 1–10 帧（用透明背景的 alpha 当轮廓）：

| 帧 | 我们的渲染 | Harmony | 差 |
|---|---|---|---|
| 1 | (8, 1919, 545, 957) | (8, 1919, 545, 957) | 0 |
| 2–5 | — | — | 0 |
| 6 | (0, 1602, 560, 1079) | (0, 1603, 560, 1079) | −1 |
| 7–8 | — | — | 0 |
| 9 | (0, 1571, 567, 1079) | (0, 1572, 567, 1079) | −1 |
| 10 | (0, 1544, 569, 1075) | (0, 1544, 569, 1075) | 0 |

**10/10 帧，误差 0–1 px。**

**最终完整公式：**

```
M_rel = M_h⁻¹ · M_c                 # 两者都来自 Harmony（OGL 空间）
R, t, s = decompose(M_rel)          # s 丢弃
Blender 相机 = rotX(90°) @ [ R | t × 4.5 ]
Blender 镜头 = 27mm / 36mm 传感器（垂直 FOV 41.112°，方形像素）
Blender 模型 = 原尺寸
```

**最后一块拼图：** `M_h⁻¹` 算出来的平移必须**再乘 4.5** 才是场景单位。
漏掉这一步，相机会被压到模型内部——我们的渲染会整屏发白，
第一次测「模型移动」时就这么翻车过一次。

**第三轮（再加上动态 FOV）：** 打开相机 Override Scene FOV，FOV 从 41° 动画到 55°，
模型仍在 (2,1,0)、缩放 1.5，相机继续偏航。

| 帧 | FOV | 我们的渲染 | Harmony | 差 |
|---|---|---|---|---|
| 1 | 41.0 | (5, 1919, 545, 959) | 同左 | 0 |
| 6 | 48.8 | (0, 1491, 556, 1022) | 同左 | 0 |
| 8 | 51.9 | (0, 1445, 559, 1003) | (0, 1447, 559, 1003) | −2 |
| 10 | 55.0 | (121, 1379, 561, 931) | 同左 | 0 |

FOV → Blender 焦距：

```
lens = 18 / ( tan(fov / 2) × aspect )
```

41.112° 时正好 27mm，与官方公式一致。

**第四轮（再加模型三轴旋转）：** 底模 Peg 加 X=20°、Y=15°、Z=30°，其余不变。

| 帧 | 立方体 | Harmony | 差 |
|---|---|---|---|
| 9 | (494, 964, 497, 927) | (494, 964, 497, 928) | 0 / 0 / 0 / −1 |
| 10 | (494, 938, 502, 903) | (494, 938, 502, 903) | 0 |

10/10 帧立方体完全一致。

过程中还发现一个纯渲染器差异：相机运动到平面下方时，Harmony 对单面平面做**背面剔除**，
Blender 默认双面渲染，于是多出一条几乎侧对镜头的细线，把包围盒撑大。
这不是相机问题，是两者的多边形剔除策略不同。

| 项 | 状态 |
|---|---|
| 官方换算公式 | ✅ 源码 + 真值双重确认 |
| 轴转换 / 镜头 / 像素比 | ✅ |
| 模型移动 / 缩放补偿 | ✅ |
| 多帧验证（1–10 帧） | ✅ 0–1 px |
| 动态 FOV | ✅ 0–2 px |
| 核心链路 | ✅ 五类工况全部通过（含三轴旋转） |

---

## 附录：复现用到的文件

**正式工具**（`E:\text\tools\`）：

```
TB_ExportMatrix.js              # Harmony 端：逐帧导出相机/模型矩阵 + FOV
import_harmony_camera.py        # Blender 端：读 JSON 建相机动画（库/CLI）
harmony_camera_importer.py      # Blender 插件：File > Import > Harmony Camera
```

**验证脚本**（`E:\text\verification\`）：

```
TB_Probe.js                 # Harmony 端探针（JS）
harmony_probe.json          # 探针输出
test_receiver.py            # 公式自检（Blender 无头）
compare_clean.py            # 干净场景对比
clusters.py                 # 轮廓分离测量
sequence_check2.py          # 多帧序列验证
fov_check.py                # 动态 FOV 验证
rot_check.py                # 三轴旋转验证
```

官方源码位置：

```
C:\Program Files (x86)\Toon Boom Animation\Toon Boom Harmony 25 Premium\
    resources\scripts\externalRendering\blender\TB_BlenderUtilities.py
    resources\scriptmodule\TB_ExternalRenderingUtilities.js
```
