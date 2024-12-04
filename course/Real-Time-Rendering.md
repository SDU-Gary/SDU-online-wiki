---
title: 实时渲染
description: 很惭愧，就做了一点微小的贡献
published: true
date: 2023-12-20T04:35:42.412Z
tags:
editor: markdown
dateCreated: 2023-12-15T09:04:29.818Z
---

# Notes on Real-time Rendering

> 「我即将融入剧烈争斗的大人世界，要在那里边孤军奋战，必须变得比任何人都坚不可摧。」

## 复习向

### 复习课笔记

1.  MSM 了解即可
2.  环境光直接光照：Split Sum；环境光全局光照：PRT
3.  RSM / LPV 是 diffuse only，而 VXGI 可以做 glossy 的次级光源
4.  LPV 之所以加速，是因为减少了查询（指在 SM 上查询多个 pixel）
5.  SSAO 假设环境光 uniform；SSDO 能实现漫反射，有色溢
6.  看到这里默写一下 Microfacet BRDF 公式（
7.  F 项和基础反射率（即颜色）有关，G 项和 D 项和 roughness 有关
8.  Split Sum 和 Kulla-Conty 生成的预计算表，其两个维度都是 roughness 和入射角度（可以带上余弦）
9.  BRDF 的性质包括：非负；线性，多个 BRDF 可以相加；可逆，入射与出射可以互换；能量守恒
10. Mipmap 查一次区域，要做一次纹理查询；SAT 则要 4 次
11. Shadow Volume 不需要可编程的渲染管线，从 SM 开始则要（开始写 shader）
12. Shadow Volume 第一趟也是先渲染拿到 depth buffer，第二趟做 depth-pass 或者 depth-fail。这样以后，如果值为 0 就说明不在阴影中
13. SM 把渲染方程中的 Visibility 项拿出来了
14. PCSS 中，blocker 向光源移动，filter size 变大，阴影变软。想一想那个相似三角形的式子
15. VSSM 的第一步（blocker search）是用加起来等于$z_{\text{avg}}$那个式子做的，比例系数用 Chebyshev 不等式估计，并且假设$z_{\text{unocc}}=t$，也就是说非遮挡物的深度都和 shading point 的深度一致；第三步（PCF）的加速方法是赌正态分布，然后 Chebyshev。
16. VSSM 的第一步和第三步都是一次纹理查询（不考虑 Mipmap 间的插值的话），并且这里不存在采样
17. 因此，VSSM 是无噪声的（因为没有采样），而 PCSS 是有的
18. 环境光直接光照是没有考虑 Visibility 的，拎出了渲染方程中的 Light 项（当 glossy 时积分域小，当 diffuse 时 smooth）
19. Kulla-Conty 近似的时候，本来是要五维的，有点高。但是我们把 BRDF 拆了，只记录 roughness，这是因为 BRDF 的描述基于 NDF，而 NDF 的性质是由 roughness 决定的。这样，roughness 加上角度就构成了一张二维表
20. PRT 是唯一一个能实现环境光贴图下 shadow 的算法
21. RSM 要存的信息：Depth + 位置、法向、flux
22. RSM 是要区域查询的，会采样，因此我们就去搞 LPV 了
23. 再想想 SSR

### Shadow

#### Projection Shadows

1.  绘制 ground plae
2.  绘制投影三角形（projected triangles；z 缓冲区关闭）
3.  渲染其他几何图形

其实大约就是这个吧：https://blog.csdn.net/m0_37609239/article/details/123869095

#### Shadow Volume

1.  渲染场景
2.  渲染 Shadow Volume 以填充 stencil buffer
3.  渲染场景

我的补充是：

1.  假设场景完全处于阴影之中，渲染场景
2.  对于每个光源：
    1.  使用该场景中的深度信息，在 stencil buffer 中构造一个 mask，该 mask 仅在可见表面不在阴影中的地方有孔
    2.  使用 stencil buffer 遮盖阴影区域，再次渲染场景，就好像它完全照亮一样。将此渲染添加到场景中。

##### Depth Pass

如果阴影的前表面和后表面在单独的通道中渲染，则可以使用 stencil buffer 来计算对象前面的前表面和后表面的数量。如果物体的表面处于阴影中，则其与 camera 之间的正面阴影表面将多于背面阴影表面。然而，如果它们的数量相等，则物体的表面就不在阴影中。

对于 Depth Pass 来说，所有照亮的表面将对应于 stencil buffer 中的 0，其中 camera 和该表面之间的所有阴影体积的前表面和后表面的数量相等。

当 camera 本身位于 Shadow Volume 内时（例如，当光源在物体后面移动时），这种方法会出现问题。从这个角度来看，眼睛首先看到的是该阴影体积的背面，这会为整个 stencil buffer 添加$-1$的偏差。

##### Depth Fail

与计算对象表面前面的阴影表面不同，可以同样轻松地计算对象表面后面的阴影表面，并获得相同的最终结果。这解决了 camera 处于阴影中的问题，因为 camera 和物体之间的阴影体积不被计算在内，但引入了 Shadow Volume 的后端必须被盖住的条件，否则阴影最终会在体积点处丢失向后无穷远。

其实就是反过来看吧（

#### Shadow Mapping

##### SM

SM（Shadow Mapping）**算法流程**：该算法分两步渲染场景。第一步（Light Pass）从光源视角渲染，得到场景中可见物体的**深度图**。第二步（Camera Pass）从相机视角渲染，使用第一步得到的深度图作为**阴影图**，决定当前点是否在阴影中。

**优点**：完全基于图像空间，无需考虑场景几何关系。生成阴影图后，直接用于决定阴影，而不需要再参考场景本身。

**缺点**：可能产生**自遮挡**和**走样**问题。深度值不连续导致不该有阴影的地方也产生了阴影。尤其光线几乎与物体表面平行时最严重。

**改进**：加入深度**偏移（bias）**，只有深度差超过一定阈值才认为发生遮挡。但偏移过大会造成阴影**悬空**。

**存储**：需要存储从光源视角渲染所得到的深度图，作为阴影判断的依据。

**适用条件**：光源较小（点光、方向光）；材质较平滑（漫反射）时算法更准确。

##### PCSS

PCF（Percentage Closer Filtering）是一种用于减少阴影 Mapping 带来锯齿的滤波技术。其**原理**是对每个片元在其周围进行**多次采样**比较，计算阴影的平均值，生成**柔和的阴影边缘**。

PCSS（Percentage Closer Soft Shadows）是在 PCF 基础上改进的真实柔和阴影算法。它考虑了**光源大小**和**可见性**，模拟了光在阴影区域的**衍射和柔和效果**。

**算法流程**：

1.  计算遮挡物的平均深度作为 blocker depth（很耗时）
2.  用 blocker depth **估计半影区域**，动态调整**滤波大小**
3.  基于滤波大小作**PCF**实现柔和阴影（很耗时）

距离阴影近的地方滤波小，硬阴影；远的大，软阴影。

**存储需求**：需要存储阴影 Mapping 生成的深度图。

**优点**：生成真实自然的柔和阴影。

**缺点**：算法复杂，查询全部深度信息非常慢。可以采样和图像模糊来优化。

##### VSSM

VSSM（Variance Soft Shadow Mapping）是一种**加速 PCSS**的阴影算法。其核心思想是利用**方差**信息估计阴影的**柔和度**，避免了遍历所有深度值。

**算法流程**：

1.  阴影贴图中每个像素存储**深度和平方深度**
2.  计算**均值和方差**，拟合区域内深度的**正态分布**
3.  使用 **Chebyshev 不等式**快速估计阴影比例
4.  利用比例信息计算遮挡物平均深度
5.  做**加权 PCF**实现柔和阴影

**优点**：大大**加速**柔和阴影的生成。

**缺点**：存储需求加大，需要额外的通道。

**假设**：深度分布近似正态分布，非遮挡物深度相近。这些假设通常成立。

##### MSM

MSM（Moment Shadow Mapping）是在 VSSM 的基础上改进的阴影算法。它利用了**高阶矩**（moments）来表示深度的复杂分布，突破了 VSSM 假设深度近似正态分布的限制。

**算法改进**：在生成阴影贴图时，不仅存储深度的平方，还存储**立方、四次方等高阶信息**。这些信息组合起来，可以更准确地拟合深度的累积分布函数（CDF）。

这样，当场景比较简单时，深度不再符合正态分布的假设，MSM 仍能很好地估计阴影比例，有效消除了 VSSM 中的漏光问题。

**缺点**：存储开销进一步加大，需要额外的通道。运算也更复杂。

**结论**：MSM 通过存储高阶矩信息，改进了深度分布的拟合，能处理更复杂的场景，生成更精确的柔和阴影。

##### DFSS

距离场（Signed Distance Field）存储了空间中各点到场景物体的**有向最短距离**。它可以精确描述物体边界。

DFSS（Distance Field Soft Shadows）**原理**：利用距离场值表示点附近的“安全距离”，在这个距离内没有遮挡。以此估计覆盖物体的“**安全角度**”，进而近似计算遮挡量和软阴影。

**算法流程**：

1.  沿视线方向**步进采样**（Ray Marching）
2.  在每个样本点计算**最小遮挡角度**
3.  根据遮挡角度估算软阴影

**优点**：计算速度快，质量高，实现简单。

**缺点**：需要事先计算和存储距离场；存储开销大，容易产生缝隙 Artifact。

**总结**：基于距离场的软阴影算法，利用距离场的良好属性实现快速高质量的软阴影。但需要承担额外的存储和计算成本。调节参数$k$生成软硬阴影效果，速度快、质量高、存储高

### Environment Lighting

### Split Sum

**Split Sum**是一种用于**实时渲染图像式照明（IBL）的方法**。

其核心思想是将**预计算结果**存储在**查找表（LUT）中**，在渲染时进行查表，避免了大量在线计算。

**算法流程**：

1.  将反射率分量提取出来
2.  对剩余部分利用 **Schlick 近似**和 **Beckmann 模型** 进行简化
3.  最终只剩粗糙度和入射角两个变量
4.  将简化结果事先计算好，存储在二维查找表中

这样，实时渲染时就只需要索引查找表，不再需要复杂积分运算，从而大大**提高了效率**。

**存储需求**：一个二维浮点数查找表。表中存储不同参数下的预计算积分值。

**优点**：计算速度快。

**缺点**：存储占用大，预计算工作量大。

**光路**：`L(D|G)E`

### PRT

**基本思路**：将渲染方程中的光照项和光传输项分离，前者使用球谐函数（SH）表达，后者包含其余项。光传输项由于只与场景几何和材质相关，可以预计算；光照项则可以灵活变化。

**算法流程**：

1.  用 SH 拟合光照项
2.  预计算光传输项的 SH 系数
3.  渲染时，两项做向量点乘即得最终像素值

**适用条件**：场景全部静态；光照可以变化。

**存储需求**：光传输项的 SH 系数。

**优点**：效率高，可以实现动态光照。

**缺点**：需要大量预计算，对高频现象（明显阴影、镜面高光等）表示能力较弱。

**其他基函数**：小波变换等也可用来提升重建质量，但不如 SH 易处理光源变化。

**光路**：如果直接光照在非 specular 材质上多 bounce 后进入眼睛，那么光路就是`L(D|G)*E`，称为`LGGE`光路。如果直接光照在 specular 材质上，反射到 diffuse 材质上后多 bounce 后进入眼睛，那么光路就是`LS*(D|G)*E`，称为`LSDE`光路。

### GI

#### 3D

##### RSM

RSM（Reflective Shadow Mapping）是一种基于阴影映射的实时全局光照算法。其核心思路是：

1.  使用阴影映射找到**次级光源**（即直接被照亮的区域）（很慢）
2.  将次级光源视为**面光源**，计算其对需着色点的**光照贡献**

**主要假设**：次级光源为漫反射材质；阴影映射空间距离近接物体之间距离也近。

**存储需求**：深度、世界坐标、法向量、光源 flux 等属性。可以说是“存出射光信息”。

**优点**：易实现，基于成熟的阴影映射。

**缺点**：性能开销大，质量不高，存在多处不合理假设。

**适用条件**：小范围光照效果（如手电筒）。对全局光照质量要求不高。

**光路**：`LD(D|G)E`

##### LPV

LPV（Light Propagation Volumes）是一种基于三维网格的全局光照算法。其核心思路是:

1.  在场景中构建**三维网格**（voxel），存储各向同性的辐照度
2.  在网格中**传播**光照，最后达到稳定
3.  每个碎片直接采样网格，进行光照计算

**存储需求**：三维辐照度纹理。

**优点**：算法简单，易实现间接光照。比较快（因为避免了区域查询）；应该对次级光源的材质没有要求

**缺点**：漏光严重（与网格大小相关）；没有考虑遮蔽；存储开销大。

**改进**：提高网格精度，代价是更大的存储和计算量。

**光路**：`LD(D|G)E`

##### VXGI

VXGI 是一种基于体素（voxel）的全局光照算法。其核心思路是:

1.  将场景**体素化**为层次结构
2.  **光照传播**：计算每个体素的光照分布信息
3.  **Cone-Tracing**：求交体素，计算其对着色点的光照贡献

**适用条件**：可以处理各向异性材质，效果好，但需要事先体素化场景。

**存储需求**：每个体素的光照、法线分布。和 RSM、LPV 不同，可以说是“存入射光信息”。

**优点**：质量高，支持各向异性材质。

**缺点**：前处理量大，对动态场景不友好。

**光路**：`LG(D|G)E`或者`LG(D|G)E`

#### Screen Space

##### SSAO

SSAO 是一种基于屏幕空间信息的简易全局光照算法。其核心思路是:

1.  假设视点周围存在**常亮的环境光**
2.  在局部区域内**采样**，统计**遮蔽比例**
3.  根据遮蔽比例调整局部光照

**主要假设**：仅考虑漫反射材质；采样点遮蔽代表视点遮蔽。

**采样信息**：深度、法线、随机采样点。

**优点**：算法简单，易实现柔和阴影。

**缺点：**存在明显错误遮蔽。

**改进**：提高采样质量，引入法线权重等。

##### SSDO

**SSDO 算法（屏幕空间定向遮蔽）通过在屏幕空间估计场景中次级光源对间接光照的贡献，以实现颜色溢出等效果。**

算法的基本思路是:对于每个像素点，随机发射一些射线。这些射线打到的物体表面点成为次级光源，会对当前像素点产生间接光照贡献。具体来说:

算法**假设间接光照主要来自相邻区域的直接光照面**。对每个像素点，随机发射几条射线。如果射线击中某一直接光照面，就认为该面对当前像素点提供间接光照。这与 SSAO 的思路不同，SSAO 假设间接光照来自环境的各个方向。

**通过累加每个次级光源的贡献**，可以估算出像素点接受的间接光照。次级光源提供的直接光照，对当前像素点来说就是间接光照。所以可以实现颜色溢出等效果。

**优点是计算成本低**，实现简单，比 SSAO 能实现更丰富的效果。主要**缺点是只考虑屏幕空间信息**，容易漏算真实的间接光照贡献。而且**只能处理局部的间接光照**。

算法需要存储每个像素的位置、法线等信息，以及直接光照的颜色。在计算时要做遮蔽检测，即检测次级光源到当前像素间是否被遮挡。

**光路**：`LDDE`

##### SSR

**SSR（屏幕空间光线追踪）是一种基于屏幕空间信息实现全局光照效果的算法。**

它的**基本思路**是:

- 对场景中的每个像素点，向屏幕空间随机发射光线进行追踪，找到光线的第一个交点。

- 这个交点成为次级光源，其直接光照就是当前像素的间接光照。

- **将所有次级光源的贡献累加**，可以估算出每个像素点接收的全局光照。

**实现上**，需要构建深度图像的 mipmap，并利用深度信息判断光线何时与屏幕空间物体相交。相交后计算光照贡献。此外还需要**存储每个像素的位置、法线等信息**。

**优点**是算法简单，质量较好，能很好地渲染镜面反射。**主要缺点**来自屏幕空间方法:

- 无法考虑屏幕空间之外的光照交互、遮挡等信息。
- 会出现边缘掉落等问题。

SSR 支持各向异性材质，但假设次级光源为漫反射面。还原现实中的各类反射效果。

**光路**：`LD(G|S)E`

## 图形学回顾

现代图形学流水线：（模型）-> 顶点处理 -> 三角面片处理 -> 光栅化 -> 片元（Fragment）处理 -> 帧缓冲（Frame buffer）操作

VBO（Vertex Buffer Object）：GPU 中用来存放模型的区域，和 obj 文件非常相似。

顶点着色器的工作：

1.  顶点变换。位置坐标等。
2.  片段着色器中的要用的量。比如 texture coordinates。

GLSL 在 GAMES 202 中还没有到 in/out 的时代。顶点着色器的主要关键字如下：

- `attribute`。顶点属性，只出现在顶点着色器中。
- `varying`。在片段着色器中有同名定义，顶点着色器写好送给片段着色器。
- `uniform`。相当于全局变量，顶点着色器和片段着色器都可以访问。

片段着色器的主要关键字如下：

- `varying`。已经做好插值了的值，从顶点着色器来的。
- `uniform`。相当于全局变量，顶点着色器和片段着色器都可以访问。

### Rendering Equation

描述了光线的传播

$$
L_{o}(p, \omega_{o})=L_{e}(p, \omega_{o})+\int_{H^{2}}f_{r}(p, \omega_{i}\rightarrow\omega_{o})L_{i}(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

其中：

- $L_{o}(p, \omega_{o})$ 为 outgoing radiance，表示从点 $p$ 出发沿方向 $\omega_{o}$ 的辐射度。~~什么辐射度，我只知道 radiance~~
- $L_{e}(p, \omega_{o})$ 为 emission。它表示表面自身发出的辐射度，比如自发光材质。
- $f_{r}(p, \omega_{i}\rightarrow\omega_{o})$ 为 BRDF。它接受两个向量作为输入：入射光线的方向和观察者的视线方向。然后，它输出一个表示光线在该表面上反射的强度的值。不严谨地说，带不带$\cos$项其实是无所谓的，都是 BRDF。
- $L_{i}(p, \omega_{i})$ 为 incident radiance。它表示了从周围环境中来自方向$\omega_{i}$的光线的能量。

这样，方程就考虑了全局光照（global illumination）的影响。通过对周围环境中所有入射方向的辐射度进行积分，可以考虑到光线在环境中的散射、反射和遮挡等效应，从而获得更真实的渲染结果。

在积分中，$\cos\theta_{i}$ 表示了入射光线的入射角度与表面法线的夹角的余弦值。这是因为入射角度越接近表面法线，入射光线对表面的影响越大。$\cos\theta_{i}$ 的作用是对入射光线的辐射度进行调整，使得入射角度较小的光线在积分中贡献更多的反射辐射度。这个 $\cos\theta_{i}$ 项是由于光线在表面上的能量分布与入射角度有关。通过乘以 $\cos\theta_{i}$，可以考虑到光线在不同入射角度下的能量变化，从而更准确地计算出反射辐射度。

在渲染方程中，$L_{i}(p, \omega_{i})$ 与 BRDF $f_{r}(p, \omega_{i}\rightarrow\omega_{o})$ 相乘，表示了入射光线与表面的相互作用。这个乘积表示了从方向 $\omega_{i}$ 入射的光线经过表面反射后的辐射度。

而在**实时渲染**中，渲染方程如下：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

显式地引入 visibility，本质上和上面的渲染方程等价。

其中：

- $L_{o}(p, \omega_{o})$ 为 outgoing lighting。
- $L_{i}(p, \omega_{i})$ 为 incident lighting（from source）。
- $f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}$ 为（cosine-weighted）BRDF。
- $V(p, \omega_{i})$ 为 visibility。它描述了从点 p 出发沿方向 $\omega_{i}$ 的光线是否能够到达光源或其他可见点。
- $\int_{\Omega^{+}}$ 表示对所有入射方向 $\omega_{i}$ 进行积分。$\Omega^{+}$ 表示半球面，即所有入射方向的集合。

---

对于渲染来说，渲染方程中的光应该要互相影响。假设所有的光都不反射，只渲染来自光源的光，称为直接光照（Direct Illumination）；加上间接光照（例如 one-bounce 的间接光），统称全局光照。

## 实时阴影

### Shadow Mapping

SM 是一种 2-Pass 的算法，所谓 2-Pass 或者说两趟，就是说场景会被渲染两遍。

第一次渲染从光源的视角渲染场景，渲染场景中可见物体的深度图（可见的，也就是说深度最浅的）。

第二次从相机用 light pass 中拿下的 Shadow Map（是一种 texture），渲染整个场景。

SM 是一种完全在图像空间中的算法。

优点：无需关注场景的几何关系，一旦 Shadow Map 生成，就可以直接用阴影图，而不必再看原场景。

缺点：会产生自遮挡（self occlusion）和走样问题（aliasing issue）。

由于 Shadow Map 记录的深度值是不连续的（所以实质上可以看成是阶梯状的锯齿，上面的锯齿当然可以遮挡下面的锯齿），因此可能产生自遮挡的问题，在不该产生阴影的地方产生了阴影。所以显然，最不容易自遮挡的地方就是光线垂直照射的时候，而当光线非常偏（几乎和物体表面平面平行的时候），自遮挡最严重。

Solution：认为发生遮挡，不仅要深度有大小关系，而且要“明显地”有大小关系，那就是说，如果差距不大，那就不认为发生了遮挡，也就不会产生自遮挡。即加一个 bias。然而，添加 bias 虽然可以有效缓解自遮挡，但却可能造成悬空，也就是说，本来应该遮挡的地方，因为在 bias 阈值内，因此渲染看上去不被遮挡，就造成了阴影悬空。

\*参考阅读：[《GAMES202：高质量实时渲染》1 实时阴影：阴影映射（Shadow Mapping）、PCSS、VSSM、SDF Shadows](https://zhuanlan.zhihu.com/p/563672775)，第一章**阴影映射（Shadow Mapping）\***

### Second-depth Shadow Mapping

其实就是，Shadow Map 不仅存最小深度，还同时存储次小深度。将最小深度和次小深度取平均，即得到 midpoint。用 midpoint 当作遮挡关系的阈值（threshold）。~~但是实际上没人用。~~

缺点：

- 要求物体必须 watertight，比如平面的纸就不行。
- 实现稍微复杂了点，而且常数变大了，在 RTR 也很关注常数

### 来点数学

**Schwarz 不等式**

$$
\left [ \int_{a}^{b} f(x)g(x) \, \mathrm{d}x\right ]^{2} \leq \int_{a}^{b} f^{2}(x) \, \mathrm{d}x \int_{a}^{b} g^{2}(x) \, \mathrm{d}x
$$

**Minkowski 不等式**

$$
\left( \int_a^b \left[ f(x) + g(x) \right]^2 \,\mathrm{d}x \right)^{\frac{1}{2}} \leq \left( \int_{a}^{b} f^{2}(x) \,\mathrm{d}x \right)^{\frac{1}{2}} + \left( \int_{a}^{b} g^{2}(x)\,\mathrm{d}x \right)^{\frac{1}{2}}
$$

**RTR 中的近似**

不关心不等，关心*近似相等*。也就是将这些不等式当成约等式用。

比如：

$$
\int_{\Omega}f(x)g(x)\,\mathrm{d}x\approx \frac{\int_{\Omega}f(x)\,\mathrm{d}x}{\int_{\Omega}\,\mathrm{d}x}\cdot\int_{\Omega}g(x)\,\mathrm{d}x
$$

什么时候这么做准确呢？

- 当$g$的积分区限很小的时候
- 当$g$足够光滑

有一个满足就能大胆约等。

### 化简渲染方程

运用上述约等式，即可将渲染方程：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

约等于：

$$
L_{o}(p, \omega_{o})\approx \frac{\int_{\Omega^{+}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}}{\int_{\Omega^{+}}\,\mathrm{d}\omega_{i}}\cdot \int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

也就是把可见性项拿出来。

什么时候足够准确？

1.  积分区限足够小。点光源/方向光源时就可以，足够准确。
2.  积分式足够光滑。光照各处几乎不变的光源或者是 diffuse 的 BRDF 时，就比较准确。

所以反过来说，Shadow Mapping 不准的时候，就是：

- 环境光（可以视作很大的面光源）导致 BRDF 不够 smooth
- 材质很 glossy，这样 BRDF 也不够 smooth

### PCF

Percentage Closer Filter（PCF）是一种用于阴影贴图采样的滤波器技术，用于减少阴影贴图采样带来的锯齿边缘。PCF 通过在每个阴影像素周围进行多次采样，并根据采样结果计算出阴影的平均值，从而实现柔和的阴影边缘。

\*参考阅读：[《GAMES202：高质量实时渲染》1 实时阴影：阴影映射（Shadow Mapping）、PCSS、VSSM、SDF Shadows](https://zhuanlan.zhihu.com/p/563672775)，第二章**PCF（Percentage Closer Filtering）反走样\***

PCF 本身是为了做抗锯齿的，不过也能用来生成软阴影，那就是 PCSS。

PCF 对于遮挡检验得到的 01 矩阵做 filtering，而不是对最后的结果做 filtering。

**为什么不是对 Shadow Map 做 filtering？**

因为就不是这么用的，对 Shadow Map 做 filtering 毫无意义。

做法：

1.  对每个 fragment 做多个深度比较（比如 7x7）
2.  对比较得到的 01 值做平均，也可以加权平均

Filter 的 size 越小，阴影越硬；filter 越大，阴影越软。那么当 size 很大时，PCF 可以用来做软阴影。

### PCSS

PCSS，即 Percentage Closer Soft Shadows，是一种用于实时渲染中生成更真实、柔和阴影的技术。它是对传统 Shadow Mapping 方法的改进，旨在解决硬阴影（Hard Shadows）的锯齿边缘和不真实外观的问题。

PCSS 的核心思想是考虑阴影中的光源大小和光源的可见性，以模拟光在阴影区域的衍射和柔和效果。传统的 Shadow Mapping 只考虑了物体是否在阴影中，而没有考虑光源的大小和阴影的柔和程度。

PCSS 发轫于 PCF，因为 PCF 本身有使阴影软化的效果。不过关键在于，**遮挡物和阴影的距离影响 filter 的 size**。

算法：

1.  从 Shading point 找，被遮挡自然就是被遮挡物（blocker）遮挡了。将 blocker 的深度记下来，取平均，得到 average blocker depth
2.  半影估计。用 average blocker depth 来决定 PCF 的 size
3.  做 PCF。因为 size 能动态调整，因此距离近的阴影硬，远的软，符合认知

第一步做 blocker search 时，一种方法是采用固定的范围，例如 4x4。另一种更好的方法是动态计算遮挡范围，我们计算 Shadow Map 的时候在光源处设置过相机，把 Shadow Map 放在相机的近截面，然后将光源和要渲染的点相连，在 Shadow Map 上截出来的面就是要查询计算平均遮挡距离的部分，这部分的深度求一个均值，就是 blocker 到光源的平均遮挡距离。

PCF 的实质是加权平均，是 filter，是卷积。

$$
\left[ w\ast f \right](p)=\sum_{q\in \mathcal{N}(p)}w(p,q)f(q)
$$

在 PCSS 中，则有：

$$
V(x)=\sum_{q\in \mathcal{N}(p)}w(p,q)\cdot\chi^{+}\left[ D_{SM}(q)-D_{scene}(x) \right]
$$

其中，$\chi^{+}$是一个符号函数，当变量为正时值为$1$，否则为$0$。所以这玩意就是一个阴影比较的结果的数学表达式。

也可以看到，PCF 并不是在 filter 阴影图，因为：

$$
V(x)\ne\chi^{+}\left\{ \left[w\ast D_{SM}\right](q)-D_{scene}(x) \right\}
$$

更不是直接对生成的图像做 filter。

---

PCSS 效果很好，但是朴素方式非常慢，因此需要加速。

慢的原因在于，在第一步和第三步中，都需要查询区域内全部*纹素*（texel，它包含纹理图像中的一个像素的信息）的深度信息。

最简单的方式就是不查询全部，而是采样。这样就是近似，缺点是会产生 noise。最朴素的想法就是生成图像后，再在图像空间上滤波降噪来找补。

\*参考阅读：[《GAMES202：高质量实时渲染》1 实时阴影：阴影映射（Shadow Mapping）、PCSS、VSSM、SDF Shadows](https://zhuanlan.zhihu.com/p/563672775)，第四章**PCSS（Percentage Closer Soft Shadows）\***

### VSSM

VSSM 是加速 PCSS 的一种新思路。Variance Soft Shadow Mapping（VSSM）通过在深度贴图中存储更多的信息来解决这些问题。具体来说，对于每个像素，传统 Shadow Mapping 只存储一个深度值，而 VSM 存储两个值：深度值和深度值的平方。这两个值的组合可以提供更多的信息，用于计算阴影的柔和度。

做 PCF 的过程，关键就在找区域内 01 值的比例，这个过程其实相当于在考试中估计自己的分数对应的排名。为了避免遍历所有的信息（所有 texel 的深度 / 所有学生的分数），那就相信正态分布。

确定正态分布，就只需要两个参数：

1.  均值
2.  方差

这样，就是说要从 SM 快速得到这两个参数，以确定正态分布，来快速估算区域内的 01 值的比例。

快速估算的方法，一方面可以直接查表（就像我们在概率论课中做的那样），另一方面可以用更简单的方式，即运用*Chebyshev*不等式估计。

不过到这里，只是优化了第三步。但是第一步还没什么优化，因为虽然得到区域平均深度，但是第一步需要的并不是它，而是遮挡物的平均深度。对于这里，有关系式：

$$
\frac{N_{1}}{N}\mathcal{z}_{unocc}+\frac{N_{2}}{N}\mathcal{z}_{occ}=\mathcal{z}_{avg}
$$

其中，$\frac{N_{1}}{N}$就是非遮挡物的比例，$\mathcal{z}_{unocc}$就是非遮挡物的平均深度；第二项同理。

那么，如何估计$\frac{N_{1}}{N}$、$\frac{N_{2}}{N}$呢？那就正是用正态分布、*Chebyshev*不等式的时候。

就立刻有：

$$
\frac{N_{1}}{N}=P(x>t)\\
\frac{N_{2}}{N}=1-P(x>t)
$$

现在，求的是$\mathcal{z}_{occ}$，两个系数能用上式估计，$\mathcal{z}_{avg}$是均值，就只有$\mathcal{z}_{unocc}$不知道了。于是大胆假设：

$\mathcal{z}_{occ}=t$，也就是说，假设非遮挡物的深度都和 shading point 的深度一致。

至此，VSSM 流程结束。

#### 均值

最简单的方式就是 Mipmap，缺点在于 Mipmap 毕竟是不准的，不同层级间还要插值，而且只能做正方形的。

更好的方式是做 Summed Area Tables（SAT）。这事就很好做了，这玩意就是二维前缀和。

#### 方差

获取方差的方式是：

$$
Var(X)=E(X^{2})-E^{2}(X)
$$

期望就是均值，因此就拿下了，只要让 SM 存的不仅是深度，再加个通道记录深度平方就可以了。

#### Chebyshev 不等式

对于单峰的概率密度函数，$t$大于均值，有：

$$
P(x>t)\le\frac{\sigma^{2}}{\sigma^{2}+(t-\mu)^{2}}
$$

其中$\mu$是均值，$\sigma^{2}$是方差。

这个式子对于随机变量的分布并没有要求。

### MSM

当 VSSM 的假设被挑战时，生成的效果就不太好，主要表现为漏光，在不该亮的地方亮了。

比如，当场景相对复杂时，假设服从正态分布还是合理的；但当场景非常简单时，它反而不一定服从正态分布，这样就导致估计比例这件事情的误差变得很大。变暗是可接受的，但变亮则非常显眼。

Moment Shadow Mapping（MSM）改进了 VSSM。它使用多阶的 moments（矩），来表示复杂函数。比如改进 VSSM，那么就在生成 SM 的时候，记录$z$，$z^{2}$，$z^{3}$，$z^{4}$，这样就能更精准地拟合 CDF，也就突破了高斯分布的局限性，能够有效改进 VSSM。

球谐函数（Spherical Harmonics）是定义在单位球面上的一组正交函数，SH 就可以用来做$z$函数。

### Distance Field Soft Shadows

距离函数：定义空间中的任何一个点到某物体表面上的最近距离。SDF 就是 Signed Distance Field，也就是说记录的是有向距离。

在传统方式中，如果对移动边界做插值，那么只能得到模糊的边界，而不是正确移动的边界。SDF 则可以插值出正确的边界。

#### Ray Marching

Ray Marching 想要做的事是找到光线和物体的交。Ray Marching 的基本思想是从视点发射射线，然后沿着射线进行步进，直到达到场景中的物体表面或达到最大步进距离。在每个步进点上，算法会根据场景中的几何形状和材质属性进行采样，并根据采样结果调整射线的步进距离。这样，算法可以有效地探测到场景中的交点，并进行光照计算。

从点$p$到某个点做 ray marching，一个关键思想是：SDF 意味着安全距离（不然的话 SDF 的值就会被范围内存在的物体更新了），所以可以快速来到$SDF(p)$（画出来是一个圆或者球）的边缘，然后重复上述过程。

#### SDF Shadow Mapping

可以用 SDF 来**近似**得到范围内有多少遮挡的部分。还是运用“安全距离”的思想，这时候一个点$p$同它的安全距离画出来的球，从视点过去其实就是一个 cone（锥体）。这样，相当于运用 SDF 获得了“安全角度”。

根据“安全角度”，就可以估算出遮挡的量。如果安全角度非常大，意味着大范围内都没有遮挡，那么自然就很亮，visibility 近似为 1；反之同理。

那么如何求的“安全角度”呢？

还是做 Ray Marching 的流程，假设步进经过三个点$p_{1}$，$p_{2}$，$p_{3}$，那么就能得到各自对应的$\theta_{1}$，$\theta_{2}$，$\theta_{3}$。取最小的$\theta$就行了。

不过计算角度虽然可以用反三角函数轻松搞定，但是反三角函数对于实时渲染而言也太过复杂了，因此有另一种方式计算。

与其使用：

$$
\arcsin\frac{SDF(p)}{p-o}
$$

不如使用：

$$
\min\left\{ \frac{k\cdot SDF(p)}{p-o}, 1.0\right\}
$$

其中$p$是找出来最小的角度对应的点，$o$是视点。使用下面这种方式，就是认为这个比值已经足够可以表达大小关系，不需要用反三角函数算那么精确。

$k$的值可以调节软硬阴影的“硬度”。

优点：

- 快（但这个有点作弊，因为生成 SDF 是不计入软阴影渲染时间的）
- 高质量

缺点：

- 需要预计算 SDF
- 高额的存储开销
- 接缝处等易有 Artifact

## 实时环境光照

环境光贴图有 Spherical map 和 Cube map，但是我们在渲染这里暂时不关心。我们只关心一个 shading point 能得到来自各个方向的光。

可以称之为 Image-Based Lighting（IBL），也就是说如何（在不考虑阴影的情况下）对一个点着色。

既然不考虑阴影，那么自然渲染方程中的 Visibility 项就不用考虑了。那么有：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

通用方法：Monte Carlo。算任何数值积分都行。缺点是需要大量样本才行。

如何才能避免采样呢？

### 简化 Lighting term

容易发现：

- 如果 BRDF 比较 glossy，那么反射光场的 lobe 其实比较细长（体现为高光），积分区限就小。
- 如果 BRDF 比较 diffuse，那么反射光场的 lobe 覆盖很大的区域，但是漫反射的 lobe 却很光滑。

于是：

$$
\int_{\Omega}f(x)g(x)\,\mathrm{d}x\approx \frac{\int_{\Omega_{G}}f(x)\,\mathrm{d}x}{\int_{\Omega_{G}}\,\mathrm{d}x}\cdot\int_{\Omega}g(x)\,\mathrm{d}x
$$

这一近似就能拿来用了。这里把$f$函数拿出来后，积分区限也只要取在$g$函数有值的地方就行了。

正是因为 BRDF 总是适合运用上述近似，就可以把上面的渲染方程改写为：

$$
L_{o}(p, \omega_{o})\approx \frac{\int_{\Omega_{f_{r}}}L_{i}(p, \omega_{i})\,\mathrm{d}\omega_{i}}{\int_{\Omega_{f_{r}}}\,\mathrm{d}\omega_{i}}\cdot \int_{\Omega^{+}}f_{r}(p, \omega_{i}, \omega_{o}) \cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

这就是把 Lighting term 从乘积的积分中拆出来了。

而拆出来的 Lighting term 其实就是 filtering，把环境光贴图给模糊了。

尤其是，这个事是 prefiltering，因此在渲染的时候拿来用就可以了。不同大小的 filtering，还可以用插值算中间结果，也就能得到任意 filter size 的环境光贴图。

---

这样以后，和过去根据 BRDF 的 lobe 去查询区域相比，现在只需要（如果是 glossy 的话）直接冲着镜面反射方向要值就行了，因为那个值是 filter 过的，因此能表示一片区域；如果是漫反射，就直接问 normal 方向要平均或者拿全图平均。这样，只要一次查询就能拿到一个区域的值，就很好。

### 预计算

$$
L_{o}(p, \omega_{o})\approx \frac{\int_{\Omega_{f_{r}}}L_{i}(p, \omega_{i})\,\mathrm{d}\omega_{i}}{\int_{\Omega_{f_{r}}}\,\mathrm{d}\omega_{i}}\cdot \int_{\Omega^{+}}f_{r}(p, \omega_{i}, \omega_{o}) \cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

对于这个式子，前半部分已经搞定了，但是要做好后半部分还是相当麻烦的。如果说对于各种 BRDF 都做预计算，那么这个空间是相当巨大的。

如何让它可以预计算呢？

回顾 Microfacet BRDF，有：

$$
f(\mathbf{i},\mathbf{o})=\frac{\mathbf{F}(\mathbf{i},\mathbf{h})\mathbf{G}(\mathbf{i},\mathbf{o},\mathbf{h})\mathbf{D}(\mathbf{h})}{4(\mathbf{n},\mathbf{i})(\mathbf{n},\mathbf{o})}
$$

其中：

- $\mathbf{h}$半程向量，$\mathbf{o}$指向光源，$\mathbf{i}$指向视点
- $\mathbf{F}(\mathbf{i},\mathbf{h})$是 Fresnel 项
- $\mathbf{G}(\mathbf{i},\mathbf{o},\mathbf{h})$是 shadowing-masking term
- $\mathbf{D}(\mathbf{h})$是法线的 distribution

关注 Fresnel 项和 NDF 项。

**Schlick 近似**

Schlick 近似可以用来近似估计 Fresnel 项。

$$
R(\theta)=R_{0}+(1-R_{0})(1-\cos{\theta})^{5}\\
R_{0}=(\frac{n_{1}-n_{2}}{n_{1}+n_{2}})^{2}
$$

其中，$R_{0}$是初始反射率，是一个颜色。 改变$\theta$即可近似 Fresnel 项。

**Beckmann 分布**

$$
\mathbf{D}(\mathbf{h})=\frac{e^{-\frac{\tan^{2}\theta_{h}}{\alpha^{2}}}}{\pi\alpha^{2}\cos^{4}\theta_{h}}
$$

其中，$\alpha$表示材质的粗糙程度。改变$\theta_{h}$（其实和上面的$\theta$近似）即可近似 NDF 项。

---

经过这两项的近似，预计算的变量就只有：$R_{0}$，$\alpha$，$\theta$。但是三维还是太多。

将 Schlick 近似带入，得：

$$
\int_{\Omega^{+}}f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}\approx\\ R_{0}\int_{\Omega^{+}}\frac{f_{r}}{F}(1-(1-\cos{\theta_{i}})^{5})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}+\int_{\Omega^{+}}\frac{f_{r}}{F}(1-\cos{\theta_{i}})^{5}\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

这样，积分式的基础反射率就被拆出来了，剩下的式子中不再含有基础反射率$R_{0}$，只有表示 roughness 的$\alpha$和入射角$\theta_{i}$。

这样，积分值就可以预计算出一张二维图，以表示 roughness 的$\alpha$和$\cos{\theta}$作为坐标轴，做表，这样的图相当于一张纹理。要用积分值，就查表就行了。

By the way，这种方法叫 Split Sum。

_参考阅读：[《GAMES202：高质量实时渲染》2 实时环境光照：Split Sum、PRT](https://zhuanlan.zhihu.com/p/563676455)，第二章 **The Split Sum Approximation**。_

### PRT

有了 Environment Lighting 后，在 RTR 中得到物体各个地方的阴影是非常难的。

两个对环境光的理解：

- 既然环境光来自四面八方，那么每个方向都可以视作是一个光源，也就转化成了 many-light problem。这样的话开销就线性于 SM 的开销。
- 视作一个采样的问题。知道 shading point 上半球各个方向不同的光线，解渲染方程，就要采样。难度在于，从 shading point 到各个方向的 visibility term 是不好知道的。此外，BRDF 可能是很高频的，特别是 glossy 的 BRDF；环境光（L 项）的 support 其实很大。这样积分式中，$L_{i}$、$f_{r}$、$V$项都拆不出来。

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

<div align="center"><i>（回看Rendering Equation）</i></div>

这里提了一嘴**环境光遮蔽**（Ambient Occlusion，后面会提），用 AO 近似 Visibility 是可以的，但是前提是认为环境光是 Constant Environment Lighting（不能是任意环境光）。

Industrial solution：从最亮的光源（比如通常是太阳）下生成一个阴影（或者多一点，两三个差不多得了）。

---

PRT（Precomputed Radiance Transfer）虽然预计算了，但是做这个做的还挺好的。

_参考阅读：[《GAMES202：高质量实时渲染》2 实时环境光照：Split Sum、PRT](https://zhuanlan.zhihu.com/p/563676455)，第四章 **PRT**。_

首先，乘积的积分可以视作滤波（因为可以视作卷积）。

$$
\int_{\Omega}f(x)g(x)\,\mathrm{d}x
$$

这个可以看做时域上两个信号$f(x)$和$g(x)$的卷积，那么也就可以看作是频域上两个信号频谱的乘积。

如果两个频谱中，有一个是低频的，那么它们相乘的结果就是低频的。结果低频，意味着信号足够 smooth。（有点像 Low Pass Filter）

#### Basis Functions

用基函数的线性组合可以拟合一个函数，用来拟合的这组函数就叫基函数（Basis Functions）。

$$
f(x)=\sum\limits_{i}c_{i}\cdot B_{i}(x)
$$

Taylor（多项式基函数）级数展开，Fourier 级数展开都是拟合的方式，SH（Spherical Harmonics，球谐）函数拟合也是。

SH 是**一系列 2D 基函数**，定义在球上（可以看做对方向的函数，方向可以用$\theta$，$\phi$来描述，也就是二维的）。

另一种理解是，SH 像一维中的 Fourier 基函数。

![Spherical_Harmonics_Visualized](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Spherical_Harmonics_Visualized.png)

如果用$SH(l,m)$来描述某一个球谐函数的话，$SH(0,0)$就是 uniform 的，在球上任何方向都一样。同一层的球谐函数，即$SH(l,\cdot)$的一组函数频率都是一样的。

第$l$阶的$SH$函数有$2l+1$个，编号从$-l$到$0$再到$l$。阶数越高，频率越高。

Q：为什么不用 2D 的 Fourier（既然球面上的函数可以展开成 2D 的），而要用$SH$呢？

A：在渲染中常见的球面上的函数，如果先用 2D 的 Fourier，再根据不同频率近似，再逆变换到球面上，很可能球面上会出现缝。而$SH$函数本来就定义在球面上，在球面上很连续，不会出现这种问题。

每个$SH$基函数$B_{i}(\omega)$是用 Legendre 多项式写的。

---

基函数有了，系数$c_{i}$也有很好的性质。

如果使用$SH$来拟合，其系数如下：

$$
c_{i}=\int_{\Omega}f(\omega)B_{i}(\omega)\,\mathrm{d}\omega
$$

这个过程称之为 Projection（投影），即知道一个函数和基函数，求其系数的过程。

至于说算这个积分，那就 sampling。（其实开销不小，很可能是 RTR 承受不起的，因此要预计算）

另外，Product Integral 本质上就是点乘（这和我们以前做投影时做向量点乘是相通的），因为函数的积分就是向量点乘。

重建（Reconstruction）：使用（截断的，也就是说一部分）系数和基函数恢复原始函数。

---

说说 prefiltering。有了一个球面上的环境光，对**某个点 prefiltering 后某个方向上做查询**，其效果基本和**没有 prefiltering，在该点上多个方向上多次查询**等价。

回顾 IBL 的渲染方程：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

其实就是$L_{i}$（光照项）和$f_{r}$（BRDF）的 product integral。而 BRDF 是定义在整个半球上相当 smooth 的函数，虽然$L_{i}$可以很复杂，有高频有低频，但如果 BRDF 是 diffuse 的话，那它就像 low pass filter 一样。

这样，差不多只要$3$阶的$SH$，就能把 diffuse BRDF 恢复得很好。

我们的目的是：任意频率的 environment lighting 照亮 diffuse 物体，在不考虑 visibility 的情况下，求 shading 的值是多少。

**既然 BRDF 只要$3$阶就能描述，那么光照项也没有必要用很高的频率描述了，同样不需要超过$3$阶。**

---

**PRT**

现在有了上述技能，我们还希望：

1.  引入阴影（上面的论述中并不考虑阴影）
2.  对 BRDF 不做限制（上面的论述提的是 diffuse BRDF）

PRT 可以做到这些，不仅如此，还能做全局光照。

再次回顾渲染方程：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

本质上是对 triple product 的积分（光照项，BRDF，visibility）。

求解它最直接的思路当然是 sampling（开销巨大）。

---

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

因为有 visibility，所以自然就能引入阴影。

PRT 认为**场景中只有光照变化**（变化是指，可以换光照，或者同个光照旋转）。

这样，triple product 可以看成两项之积，也就是：

1.  lighting 项：$L_{i}(p, \omega_{i})$
2.  lighting transport 项：$f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})$，也就是其他与光照无关的全部项

PRT 将 lighting 项用$SH$重建；既然认为只有光照变化，那么 lighting transport 就是不变的，相当于任意 shading point 的固有性值，也就可以**在渲染之前预计算好**。

而我们发现，在只有光照变化的场景中，lighting transport 项还是个球面函数（固定观察视角），也就也能用$SH$重建。

讨论 BRDF：

#### Diffuse Case

这个时候 BRDF 是个 constant value，可以直接拉出来（注意这是等式）：

$$
L_{o}(p, \omega_{o})=\rho\int_{\Omega^{+}}L_{i}(p, \omega_{i})V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

其中$\rho$就是 BRDF 的值。

用$SH$重建，那么 lighting 项就可以写成：

$$
L_{i}(p,\omega_{i})\approx\sum{l_{i}B_{i}(p,\omega_{i})}
$$

其中$l_{i}$是光照系数（lighting coefficient）。

渲染方程也就可以写成：

$$
L_{o}(p, \omega_{o})\approx\rho\sum{l_{i}\cdot\int_{\Omega^{+}}B_{i}(p,\omega_{i})V(p, \omega_{i})\,\mathrm{d}\omega_{i}}
$$

> 闫令琪语：在微积分里面，咱们经常会探讨说什么时候可以交换求和与积分。在图形学里从不考虑，啊，我们认为它一定可以交换。……在 PRT 的场合下，是绝对不需要考虑的。就是说我放心地交换积分与求和。
>
> 弹幕说：实变函数满足 Fubini 定理，才可以交换顺序。

然后，对于这个积分，$l_{i}$也是常数，因此可以拿出来。这样，积分式实际上就变成了 light transport 这个球面函数投影到某个 basis function 上的系数，这自然是可以预计算的。~~With some mathematical magic~~ 只有 lighting 发生变化，那么任何一个 shading point，都可以投影到任何一个 basis function 上去。比如说，投影到前$3$阶的$SH$，然后算出个数就可以了。

这样以后，即有：

$$
L_{o}(p, \omega_{o})\approx\rho\sum{l_{i}T_{i}}
$$

其中$T_{i}$是上面预计算得到的系数。这意味着，对于任何 shading point 的计算（shading，包括 shadow），只要做一次向量点乘即可。

---

换个方式看，还是从渲染方程出发：

$$
L_{o}(p, \omega_{o})=\rho\int_{\Omega^{+}}L_{i}(p, \omega_{i})V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

用上$SH$，有：

$$
L(\omega_{i})\approx\sum\limits_{p}c_{p}B_{p}(\omega_{i})\\
T(\omega_{i})\approx\sum\limits_{q}c_{q}B_{q}(\omega_{i})
$$

在积分中展开，有：

$$
L_{o}(p, \omega_{o})=\rho\int_{\Omega^{+}}L_{i}(p, \omega_{i})V(p, \omega_{i})\,\mathrm{d}\omega_{i}\\
\approx\sum\limits_{p}\sum\limits_{q}c_{p}c_{q}\int_{\Omega^{+}}B_{p}(\omega_{i})B_{q}(\omega_{i})\,\mathrm{d}\omega_{i}
$$

看起来这双重求和一眼$O(n^{2})$，但是向量点乘是$O(n)$。原因是$SH$是正交的，因此只有在$p$和$q$相同，也就是$B_{p}$ 和$B_{q}$是相同的基函数的时候，积分结果才会是$1$，否则就是$0$，因此实际上仍然是$O(n)$。

---

Light transport 项不能变，首先就意味着 visibility 不能变，意味着每个地方的遮挡关系不能变，这就是说场景不能动，必须是静态的；Light 项使用$SH$描述的，只要环境光贴图事先算好，就能事先确定$L_{i}$，就能做到换光源。

这样的方法，本身不能直接允许光源旋转，但是根据$SH$的性质，我们发现光源的旋转能与$SH$的系数变化对应起来（这是$SH$的性质），因此能立刻确定光源旋转后的$L_{i}$。

---

$SH$的美好的性质：

- 正交。将一个$SH$函数投影到另一个$SH$函数上，得到的就是$0$.
- 做投影和重建很简单，即 product intgral。
- 做旋转很简单，正如上面提到支持光源旋转的原因。（有点像旋转坐标轴）
- 做卷积很简单。
- 基函数数量少。

当然了，$SH$函数的重建能力还是有所欠缺，对于高频信号需要的阶数极多，不过对于 diffuse BRDF，$3$阶$SH$还是够了的。

#### Glossy Case

Diffuse 和 glossy 的区别就在于 BRDF。在 diffuse 的场景中，BRDF 可以视作一个常数，但是在 glossy 的场景中却不行（此时就是完整的四维的，包括两维输入方向，两维输出方向）。

这意味着，给定任何一个观察方向，light transport 项都会投影出一组完全不同的向量。

这样，如果还是套用 diffuse case 的方程：

$$
L_{o}(p, \omega_{o})\approx\rho\sum{l_{i}T_{i}(\omega_{o})}
$$

我们发现，$T_{i}$从原来的常数，变成了一个关于观察方向$\omega_{o}$的函数。那么这里$T$就从原来的向量变成了一个矩阵，就是说用上了不同观察方向$\omega_{o}$；换个角度说，glossy case 得到的必然是一个向量（因为不同观察方向得到的结果不同），而一个向量只有点乘一个矩阵才能得到向量，因此$T$也就必须是一个矩阵。

Glossy case 下，$SH$就不能只取$3$阶了。而当物体非常 glossy，比如几乎接近 specular 的时候，那就太高频了，已经到 PRT 无法解决的程度了。

#### 多 Bounce 的 Light Transport

首先说如何描述光路。

如果光线直接进入眼睛，那么光路就是`LE`。

如果直接光照打在 glossy 材质上进入眼睛，那么光路就是`LGE`。

如果直接光照在非 specular 材质上多 bounce 后进入眼睛，那么光路就是`L(D|G)*E`，称为`LGGE`光路。

如果直接光照在 specular 材质上，反射到 diffuse 材质上后多 bounce 后进入眼睛，那么光路就是`LS*(D|G)*E`，称为`LSDE`光路，这种反射情况称为"caustics"。

---

我们发现这些光路都是`L`和`Light transport`的结构，因此只要是 PRT 的思想，即使是多 bounce 的 light transport，也没有什么特殊的。

至于做法，就是算 light transport 时，把每趟计算，各 bounce 中的$B_{i}$都看成是一种光照，然后复用 no bounce 的 PRT 即可。

$$
T_{i}\approx\int_{\Omega^{+}}B_{i}(p,\omega_{i})V(p, \omega_{i})\,\mathrm{d}\omega_{i}
$$

<div align="center"><i>Light transport项</i></div>

即使物体的材质非常复杂（比如一个生锈的铁壶，那么生锈处和金属处的 BRDF 自然不同，此时整个物体的材质将达到六维，两维表示位置，另外四维表示该位置的 BRDF），但是都可以预计算。

#### 缺陷

1.  $SH$不够高频，对高频信号的重建能力弱。
2.  要求场景、材质静态。
3.  预计算一样有较大的负担。

### 其它基函数

- Wavelet
- Zonal Harmonics
- Spherical Gaussian
- Piecewise Constant

不过只介绍 wavelet，主题是二维的 Haar wavelet。

不同于$SH$定义在球面上，小波定义在图像块上，而且定义域不同。

![Haar_Wavelet](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Haar_Wavelet.png)

<div align="center"><i>2D Haar Wavelet可视化</i></div>

原理：投影到 wavelet 上后，发现很多时候存在大量的基函数，其系数为$0$或者说接近$0$. 那么就可以只取系数最大的一部分，drop 掉其余部分。

这就是一种 non-linear approximation，优势在于可以全频率表示，重建低频或高频均可。

对 Cubemap 表示的环境光贴图，对其六个图都做 wavelet 变换（即将图投影成小波的系数）。将高频信息留在右上、右下、左下，左上部分集中稍低频的内容，还可以接着做小波变换。

![Wavelet_Transform](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Wavelet_Transform.png)

BTW，Jpeg 采用的就是**类似**小波变换的变换，**离散余弦变换**（DCT），也就是先投影，再取非零系数。压缩效果很好。

用 wavelet 取代$SH$，效果很好，因为能够显示更高频的信息（包括更精致的阴影、更真实的 glossy 表面）。然而小波的致命缺陷在于，它无法 handle 快速旋转。这是因为虽然$SH$的 rotation 非常容易，但小波却没有这样的性质，因此每一次 rotation，小波都要重新求解。

> 闫令琪大二下入坑图形学，大三发 Pacific Graphics 2012。所谓幸福感，难道不是像沉默在悲哀的河流底下微微闪耀着沙金一样的东西吗？经历过无限悲哀后，看到一丝朦胧的光明这种奇妙的心情。
>
> 也许有一天，哪怕是卑微渺小的自己，也能有这样冲破满是迷雾的牢笼的体悟——我是如此祈求着。

## 实时全局光照

正如**_Rendering Equation_**一章中提过的说法：

> 对于渲染来说，渲染方程中的光应该要互相影响。假设所有的光都不反射，只渲染来自光源的光，称为直接光照（Direct Illumination）；加上间接光照（例如 one-bounce 的间接光），统称全局光照。

环境光照也是直接光照。显然，间接光照是不好做的那个。

一切被直接光照（其光源称为 primary light source）照到的物体，都会继续将自己作为光源（称为次级光源，secondary light source），再去照亮别的物体。

_参考阅读：[《GAMES202：高质量实时渲染》3 实时全局光照：RSM、LPV、VXGI、SSAO、SSDO、SSR](https://zhuanlan.zhihu.com/p/556057984)。_

### 3D Space

#### RSM

Reflective Shadow Mapping（RSM）是一种基于 Shadow Mapping 的实时全局光照（Global Illumination，GI）算法。

为了 shading 某点的间接光照，需要解决：

1.  哪些是次级光源？也就是说，哪些地方会被直接照亮？
2.  各个 surface patch 对某点上间接光照的贡献如何计算？

问题 1 可以被 Shadow Map 直接解决。问题 2 的回答是，将 surface patch 看作是 area light。

SM 天然就能很容易地描述次级光源，需要计算每个 texel 作为次级光源的贡献，因而是个计算量很庞大的工作。

另外，由于上面都是从某点处观察各个方向的光线得到的，但是这并不能直接得到从 camera 看向那点的结果。就是说，从那点看过去，和从 camera 看过去，得到的结果是不一样的。

为了不依赖观察方向，引出了一个**假设：所有的反射物（即次级光源，也就是被直接照亮的 patch）都是 diffuse 的**。这样，不管从哪里看向次级光源，结果都一样。

---

算次级光源的贡献的时候，直接对立体角采样做积分开销太大，并不合适；更好的做法是直接对次级光源发射出的光线采样，然后直接对 shading point 着色。

那么从对立体角积分，到对 Area 积分，公式如下：

$$
L_{o}(p, \omega_{o})=\int_{\Omega_{patch}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}V(p, \omega_{i})\,\mathrm{d}\omega_{i}\\
=\int_{A_{patch}}L_{i}(q\rightarrow p)f_{r}(p, q\rightarrow p, \omega_{o})V(p, \omega_{i})\frac{\cos{\theta_{p}}\cos{\theta_{q}}}{||q-p||^{2}}\,\mathrm{d}A
$$

甚至于当 patch 比较小的时候，甚至不需要积分，直接乘 A 面积即可。

**For a diffuse reflective patch**

其 BRDF（指的是次级光源 reflector 的 BRDF，不是上面积分公式中 p 点的 BRDF）可以认为是常数，$f_{r}=\rho/\pi$。

而$L_{i}$，有：

$$
L_{i}=f_{r}\cdot\frac{\Phi}{\,\mathrm{d}A}
$$

其中$\Phi$是 incident flux。~~总之是一些光学魔法~~ 发现带进去$\mathrm{d}A$甚至能被消掉，甚好。

牵扯到 visibility 的问题，我们发现不可能对每个 shading point，都去生成 Shadow Map 去检查 visibility。不好算，那就不算了。（总之就是如此）

**不过不是所有的 pixels 都会有贡献。**

- Visibility（但是较难处理）
- Orientation（就是根据法向，次级光源不可能照亮 shading point）
- Distance（如果非常远，它的贡献就很小，那就干脆别管了）

对于第三点 distance，我们希望只关心距离 shading point“足够近”的次级光源。在世界坐标中找，比较繁琐，那么干脆投影到 Shadow Map 上，在 SM 中找近的。（**这里算是引入了一个比较大但是不算不合理的假设，即在 SM 中距离近的，在世界坐标中就距离近**）至于 SM 上找，那就可以有不同的采样方式。

综上，RSM 需要存储的信息有：

- **Depth** SM 本来就要存深度
- **World Coordinate** 世界坐标判断两点距离
- **Normal** 反射物法向，以及算$\cos$项
- **Flux** 和光源有关的属性

用在手电筒这种小区域上效果很好。

优点：

- 容易实现（因为基于 SM）

缺点：

- 性能开销线性于光源数
- 没有考虑 visibility（上面说了）
- 假设太多，不够真
- 采样和质量的 tradeoff

说它是图像空间的方法，是因为 SM 的第一个 pass 得到的就是一张图，基于这张图做的后续操作；不过话又说回来，正是因为 RSM 在这种情况下已经足够获取它所需要的所有信息了，所以说它是 3D 空间的方法也没问题，因为它不会受到 camera pass 是否可见造成的影响。

#### LPV

Light Propagation Volumes（LPV）在基于 3D 空间中光线传播的特定。

如果在任何一个 shading point 上，如果可以立即获得间接光照到达 shading point 时来自所有方向的 radiance，那么自然可以做间接光照了。

我们发现：光线在空间中传播时，radiance 是一个不变的量。

那么 LPV 首先将空间分为 3D 网格（分为 Voxel），这些格子用来传播间接光照的 radiance。这样，求某格子收到的 radiance，只要把次级光源给出的 radiance 按格子求了就行。

步骤：

1.  找出次级光源，即哪些点收到直接光照
    1.  用 RSM 一样的方法找出次级光源（同样是有多少光源就做多少遍）
2.  将上述点注入（injection）到三维网格中

    1.  划分 3D 的格子（一般直接用三维纹理搞定了：texel 对应 3D 空间中的 voxel）
    2.  在任何格子内部，将所有虚拟光源往各个不同方向的 radiance 加起来
    3.  格子内现在记录了向不同方向的 radiance 值，可以拿$SH$压缩，至少两阶四个（另外，用上了$SH$，其实就几乎可以默认为只能处理 diffuse 材质了）

3.  在 3D 的网格中做传播，即 radiance propagation
    1.  认为 radiance 都是从网格中心向各个方向沿直线出发的
    2.  六个面传播 radiance，每个格子都把 radiance 加起来，还是用$SH$表示
    3.  重复上述操作直到迭代到稳定状态（大约四五次即可）

传播完成后，整个场景就完全覆盖了。这样，已经知道了任意 shading point 来自各个方向的 radiance，也就能直接拿来渲染了。

显然和 RSM 一样，LPV 也没有考虑 visibility。

**缺点：**

漏光。因为认为格子内的 radiance 都保持一致，那么有点地方（比如狭窄的墙或者细缝）明明不可能接收到 radiance，但是却会在 LPV 下被照亮。**本质在于狭窄到比格子更细**。更细分格子虽然可以解决这个问题，但是会带来存储增加、传播开销增大的问题。

#### VXGI

不同于 RSM，RSM 中次级光源都是每个像素表示的微小的表面，而 VXGI 中场景完全离散化成一系列的格子，并且是 hierarchical 的。一言以蔽之，就是从 pixel 表示变成了 voxel 表示。

VXGI 是一个 two-pass 的算法，其第二趟从 camera 出发，发射 camera ray，打到任何一个像素上，对 glosst 材质，其反射光类似于锥，拿这个反射光锥和场景中的 voxel 求交，那么交上的 voxel 对 shading point 的贡献就可以计算出来，从而来计算光线的传播。这个过程称为 cone-tracing。

**Pass 1：**

也就是 Light Pass，决定哪些 voxel 会被直接照亮。这些反射物可以使 diffuse 的，也可以是 glossy 的。这是因为，VXGI 的 voxel 中记录了光源分布，表面法向分布，这样一来，根据材质，就可以算出出射分布，比 LPV 那种限定 diffuse 然后$SH$压缩要准。

**Pass 2：**

也就是 Camera Pass，做 cone-tracing。最朴素的做法是遍历所有 voxel 看看相交没，但是不如在 hierarchical 的 voxel 上找对应的层级，快速找到这个 cone 能 cover 的 voxel，那么他们就能对 shading point，也就是 cone 的起点有所贡献。

---

上述 glossy 材质只产生一个 cone，如果是 diffuse 材质，那么相当于产生了若干个 cone（比如 8 个来 cover），对每个 cone 都重复上述做法。

相对来说，LPV 只要做一遍 propagation，并且使用$SH$表示，因此更快、更不准；VXGI 效果更好，但是相应的速度就慢了。另外，体素化是非常麻烦的事情，特别是动态场景。

### Screen Space

所谓屏幕空间，就是说只能拿到屏幕上能看到的东西，也就是做 GI 之前能得到的，比如直接光照。

要而言之，GI in SS，起手信息就是做了直接光照的屏幕场景。

#### SSAO

Screen Space Ambient Occlusion（屏幕空间的环境光遮蔽，SSAO）做了对全局光照的近似，使得物体之间有恰当的阴影，而这些阴影原来应该由间接光照产生。

假设：

1.  对任意 shading point，不知道间接光照。所以不如像 Blinn-Phong 着色模型一样，假设任何 shading point 从各个方向来的间接光照都是常数。
2.  虽然来自各个方向的间接光照都是常数，但是不见得每个方向都能接收得到。显式地考虑 visibility，如果有被挡住的部分，那么这里的间接光照就要暗一些。
3.  假设为 diffuse 材质。

回到渲染方程：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

首先使用近似：

$$
\int_{\Omega}f(x)g(x)\,\mathrm{d}x\approx \frac{\int_{\Omega_{G}}f(x)\,\mathrm{d}x}{\int_{\Omega_{G}}\,\mathrm{d}x}\cdot\int_{\Omega}g(x)\,\mathrm{d}x
$$

事实上，可以把$\frac{\int_{\Omega_{G}}f(x)\,\mathrm{d}x}{\int_{\Omega_{G}}\,\mathrm{d}x}$看成是函数$f$在积分域$\Omega_{G}$上的平均，因此也可以写成：

$$
\int_{\Omega}f(x)g(x)\,\mathrm{d}x\approx
\overline{f(x)}
\cdot\int_{\Omega}g(x)\,\mathrm{d}x
$$

处理渲染方程，得到：

$$
L_{o}^{Indir}(p, \omega_{o})\approx
\frac{\int_{\Omega^{+}}V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}}{\int_{\Omega^{+}}\cos{\theta_{i}}\,\mathrm{d}\omega_{i}}\cdot
\int_{\Omega^{+}}L_{i}^{Indir}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

为什么处理完以后乘积左边还有带着$\,\mathrm{d}\omega_{i}$的$\cos$项呢？因为投影立体角（Projected Solid Angle）$\,\mathrm{d}x_{\perp}=\cos{\theta_{i}}\,\mathrm{d}\omega_{i}$，所谓立体角其实就是单位球面上的一个面积，乘上$\cos$项以后正好就是投影到球的大圆（也是单位圆）上的一个圆（的面积）上。

![Projected_Solid_Angle](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Projected_Solid_Angle.png)

这样就可以理解成在单位圆上做积分，从连着$\cos$项在立体角上积分，变成了不带$\cos$项而在投影立体角上积分（就是说怎么理解$\cos$项的事），被积分项的意义就变成了单位圆上的面积的微元，那么积出来就是单位圆的面积，也就是$\pi$。

一言以蔽之，就是说$\cos{\theta_{i}}\,\mathrm{d}\omega_{i}$本身就是一个微元，可以写成$\mathrm{d}A$，然后$A$是单位圆的面积的微元。

~~使用一些数学魔法~~发现近似式中，分母的积分式是对整个半球积分，立体角$\omega_{i}$可以拆成$\sin{\theta}\,\mathrm{d}\theta \,\mathrm{d}\phi$，积出来就是一个常数$\pi$。

BTW，这一恒等式已有定数，称为$k_{A}$：

$$
k_{A}=\frac{\int_{\Omega^{+}}V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}}{\pi}
$$

其意义是：任何一个点向四面八方看过去，看到的 visibility 的加权平均。

这样一来，渲染方程就是：

$$
L_{o}^{Indir}(p, \omega_{o})\approx
k_{A}\cdot
\int_{\Omega^{+}}L_{i}^{Indir}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

对于近似中积分式，我们有以下处理：

1.  已经假设间接光照是常数了，因此$L_{i}^{Indir}(p, \omega_{i})$那就是个常数；由于和角度已经没关系了，所以可以写成$L_{i}^{Indir}(p)$直接提出来。
2.  已经假设是 diffuse 材质了，因此其 BRDF 为常数，即$\frac{\rho}{\pi}$（其中$\rho$是 diffuse albedo），因此也可以直接提出来。
3.  最后的$\cos$项积分本身会得到$\pi$。

> Diffuse albedo（漫反射反射率）是描述材质表面对漫反射光线的反射特性的属性。它表示了材质表面对入射光均匀地反射多少光线，而不会发生明显的方向性反射。漫反射反射率通常用符号$\rho$表示。
>
> 对于漫反射材质，其 BRDF（双向反射分布函数）描述了入射光线在材质表面上的反射分布情况。BRDF 定义了出射辐射度与入射辐射度之间的关系，其中入射辐射度是指从一个给定方向进入材质表面的光线辐射度，而出射辐射度是指从该表面反射出来的光线辐射度。
>
> 对于漫反射材质，其 BRDF 是常数$\frac{\rho}{\pi}$。这意味着无论入射光线的方向如何，材质表面上的反射光线都均匀地分布在所有可能的出射方向上。这是因为漫反射材质对入射光线的反射是均匀且无方向性的，不会发生明显的镜面反射。

于是，有：

$$
L_{o}^{Indir}(p, \omega_{o})\approx
k_{A}\cdot
L_{i}^{Indir}(p)\cdot
\frac{\rho}{\pi}\cdot
\pi\\
=k_{A}\cdot
L_{i}^{Indir}(p)\cdot\rho
$$

这样相当于说，AO 中的间接光照强度，相当于一个常数乘以半球表面上的平均 visibility。

渲染方程就可以写成：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}\\
=\frac{\rho}{\pi}\cdot
L_{i}(p)\cdot
\int_{\Omega^{+}}V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

---

那么，如何在屏幕空间中算出平均 visibility 呢？我们首先要限定一定范围内的遮挡情况，因为不同于环境光时假设无限远，这里的光源距离物体是有一定距离的，超出这个范围的光我们应该直接 drop 掉，不再考虑。（这也是一个假设，实际上不够准确）

不过在 Screen Space 中，我们没办法直接 trace 光线。因此 SSAO 的做法（同时也是一个大胆的假设）是：对任意的 shading point，都往其周围半径为$r$的球中采样很多点。

![Use_SSAO_Calculating_Average_Visibility](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Use_SSAO_Calculating_Average_Visibility.png)

这样以后，虽然没有办法直接 trace 光线，但是深度信息是有的。在 Shadow Map 上直接判断可不可见就行。即，拿采样点的深度和 SM 中 camera 看过去的最小深度比较，来决定采样点是否可见。

_第二个球中最下面的红点被 SM 错误判断了，但是这完全可以接受。_

其次，只有半球需要考虑，因此上面图中的球可以直接水平截掉上面一半。现在我们都认为是能拿到法向信息的，这样只要选取法向指向的那半球就可以了，同时甚至还能根据法线方向对不同方向的 visibility 做加权。

当然了，SSAO 会产生一些 false occlusions，也就是说会错误遮挡原本不应该遮挡的地方。这个问题的本质在于：俩东西在屏幕空间中离得挺近的，但是实际上可能离得还挺远的。采用 HBAO（也就是上面说的考虑了法向信息的 AO）可以有效缓解这个问题。

#### SSDO

> 首先错过了闫令琪直播吃键盘，令人扼腕叹息。

所谓 SSDO，就是 Screen Space Directional Occlusion（屏幕空间的方向性遮挡）。它是 SSAO 的一个提升，与其说大胆假设间接光照到处都一样，不如考虑地更精准一些。

那为什么这么说呢？因为间接光照的信息其实是可以知道的，和 RSM 类似。也就是说，提供间接光照的，其实就是被直接光照照亮的次级光源；次级光源提供的直接光照，就是场景中的间接光照。

我们要用直接光照的信息，但不是从 RSM 中获取，而是在屏幕空间中获取。SSDO 的做法和 path tracing 类似。即，对于任意 shading point，想要计算它接收到的光照，做类似 path tracing 的操作，随机地向某方向发射光线；发射的光线打到的点，自然对 shading point 有所贡献。

所以，SSAO 和 SSDO 的假设看上去其实是相反的，即：

- AO：先假设 shading point 会接受到四面八方的间接光照，然后再根据阻挡情况，把一定比例的间接光照 block 掉
- DO：不假设 shading point 接受四面八方的间接光照，而是只考虑次级光源的直接光照

![Differences_Between_AO_and_DO](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Differences_Between_AO_and_DO.png)

这是因为，SSAO 假设间接光照来自比较远的地方，而 SSDO 假设间接光照只来自于非常近的地方。因此，AO 要看一定范围内，哪些地方会阻挡间接光照，而 DO 要看哪些地方能够提供间接光照。它们都不完全正确。

在效果上，SSAO 做不到像 SSDO 能做的 color bleeding，也就是色溢。

所谓 color bleeing，就是说光线从一个 diffuse 表面到另一个 diffuse 表面不断传播，于是不同颜色的 diffuse 表面就会互相照亮对方。显然，SSAO 只做亮暗，无法做到 color bleeding。

---

看看渲染方程：

$$
L_{o}^{dir}(p, \omega_{o})=
\int_{\Omega^{+},V=1}
L_{i}^{dir}(p)
f_{r}(p, \omega_{i}, \omega_{o})
\cos{\theta_{i}}\,
\mathrm{d}\omega_{i}
$$

这就是说，和 path tracing 一样，直接光照当然是可见（也就是$V=1$）的部分给出的。这种情况就没有间接光照。

$$
L_{o}^{indir}(p, \omega_{o})=
\int_{\Omega^{+},V=0}
L_{i}^{indir}(p)
f_{r}(p, \omega_{i}, \omega_{o})
\cos{\theta_{i}}\,
\mathrm{d}\omega_{i}
$$

这一部分，从 shading point 往某个方向打的时候，$V=0$意味着达到了某个点，那么这个点的 radiance 自然要算作间接光照的贡献。

要计算这些贡献的和，假设材质是 diffuse 的，那么它反射到任何一个 shading point 的 radiance，于从 camera 看到的它反射出的 radiance 是一样的；这种情况下，就可以直接计算。至于如何找到哪些会被挡住，做法和 SSAO 是一样的，即：从某点出射的光线，不必真的 trace 它会不会被挡住，而是仍然看从 camera 到它会不会被挡住。

找到了被挡住的点，自然就可以根据它们的法向，来算出它们对 shading point 的间接光照的贡献。

缺点在于，因为它在屏幕空间做，所以有一些看不到的面**本应该**有所贡献，但是因为在屏幕空间中做，所以这些贡献是计算不上的。（这是所有屏幕空间做法的缺点）

另外，SSDO 本身决定了它只能做小范围的间接光照。

#### SSR

Screen Space Reflection（或者说 Screen Space Ray-tracing 可能更准确）是一种实现全局光照的方法，它在屏幕空间中做 ray tracing，以达到 GI 的效果。

SSR 要做到：

1.  对任何光线和屏幕空间场景（其实就是一个壳）求交
2.  算相交的点的贡献

SSR 的思想在于：反射的部分，绝大多是其实是屏幕空间中已经有了的部分，因此其实可能并不需要额外信息，只要重用场景中已有的信息即可。（例如雨后街道倒影，反射的是场景中有的街道上的东西）

由于实现的是屏幕空间的 ray tracing，自然，SSR 对材质并没有要求。

流程是：

- 首先已知不带 SSR 的屏幕空间场景（当然了，还有法向信息和深度信息两张贴图）
- 对于任意 shading point，向场景中发射光线，与场景求交（详细来说，干的是 ray marching 的活，与最浅深度相交）
  - 做 ray marching，从 shading point 出发，每隔一定步长，看看到那里是不是小于最小深度
  - 显然，如果步长太长，最后求到的交就不准；如果步长太小，开销就很大
  - 解决方法是 hierarchical ray trace
- 将 SSR 的结果加到屏幕空间上，也就是 shading

其中，所谓的 hierarchical ray trace，需要用到 Depth Mip-Map，存储的是一个区域内的最浅深度。这样，使用这个加速结构的意义就是：如果说查询 Depth Mip-Map，如果连上层的一个最浅深度都交不上，那么就更不用说下级的所有深度了，那就可以快速跳过这一部分。

![Depth_Mip-Map](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Depth_Mip-Map.png)

示例代码：

```cpp
min = 0;
while (level > -1)
    step through current cell;
	if (above Z plane) ++level;
	if (below Z plane) --level;
```

找到交点后，将`level`置成负的，就可以跳出循环。

---

SSR 的 shading 还是拿渲染方程：

$$
L_{o}(p, \omega_{o})=\int_{\Omega^{+}}L_{i}(p, \omega_{i})f_{r}(p, \omega_{i}, \omega_{o})V(p, \omega_{i})\cos{\theta_{i}}\,\mathrm{d}\omega_{i}
$$

如果 BRDF 完全是 specular 的，则其 BRDF 就是一个 delta 函数，积分就没了，直接找反射光反射的 radiance 即可；glossy 的话就 Monte Carlo 之类的方法多采样几根。

虽然如此，SSR 由于假设了从反射物反射到某点的 radiance 和反射到 camera 的值一样（这很重要），因此**反射物**（比如 ppt 中经典的布料）必须是 diffuse 的，而地板什么的材质无所谓。

SSR 的缺点并不来自于光追，而还是在于屏幕空间：

1.  Hidden Geometry Problem。由于屏幕空间只有一层壳，所以说有一些屏幕空间看不见的部分，本应该有贡献但是没被考虑。
2.  Edge Cutoff。直说了，就是一个东西在屏幕空间中只有一部分（比如很高的楼，那它上面的部分大概率就不在屏幕空间中），那缺失的部分自然也不能被 SSR 反射出来。（一个取巧的办法是，将 cut 的边缘做一个平滑滤波之类的，看起来糊了那也算合理）

---

SSR 的其他特性：

- 没有可见性和距离平方衰减的问题，因为是用 BRDF sampling 做的，在立体角范围内对 shading point 做 tracing，而不是对次级光源的 area 做积分算贡献
- 对 shading point 有贡献的次级光源一定是可见的，遮挡关系是正确的
- 支持各种 BRDF 的就不说了（比如 diffuse 反射得比较糊之类）
- 天然支持 Contact Hardening。也就是说，和地板（就这么说吧，反正是这么个东西，地板上面展示 SSR 的反射结果）接触地近的，反射出来的比较清晰；远端就比较模糊
- Specular Elongation。就是说如果认为地板各项同性，那么反射的 lobe 其实有点像一个拉长的椭圆；于是光线反射出去以后，看到的东西就好像被拉长了
- 不挑地板。也就是说不论接受反射的是什么材质，都可以

SSR 的空间复用的 trick：

![SSR_Hit_Point_Reuse_Across_Neighhbors](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/SSR_Hit_Point_Reuse_Across_Neighhbors.png)

由于用上了邻居 trace 的结果，原本 trace 的两条光线，起到了四条光线的作用。

也可以像 split sum 一样，先 filter 一遍，那么在镜面反射的方向上查一次，就好像是在没有 filter 的情况下，对那个区域查好多次，这样对屏幕空间中 glossy 的部分，查一次就好了，非常方便。缺点是，在 split sum 中环境光是无穷远的，但是在 SSR 的情况下，屏幕空间可不是这样的，这样的 filter 就不好做。

SSR 的优点：

- 做 glossy 和 specular 表面的反射很快
- 质量挺好的
- 没有 spike，也没有遮挡上的问题

缺点：

- diffuse 场景就没那么高效了
- 继承了来自 Screen Space 方法的缺点

## PBR

所谓 PBR，就是 Physicaly-Based Rendering。这涉及到材质、光照、camera、light transport，当然材质是其中非常重要的一部分。

不论是 Microfacet 还是 Disney，在 RTR 中其实都不够 physically-based，Disney 更加 artist-friendly。

Microfacert / Disney 主要处理表面材质，而对于 volumes（比如云、头发等等），则专注处理一次/多次 scattering。

### Microfacet BRDF

所谓微表面材质，其实就是假设了微观表面都是镜面反射，根据微表面的分布，最后材质能有 diffuse、glossy、specular 的分别。

$$
f(\mathbf{i},\mathbf{o})=\frac{\mathbf{F}(\mathbf{i},\mathbf{h})\mathbf{G}(\mathbf{i},\mathbf{o},\mathbf{h})\mathbf{D}(\mathbf{h})}{4(\mathbf{n},\mathbf{i})(\mathbf{n},\mathbf{o})}
$$

- **Fresnel 项** $\mathbf{F}(\mathbf{i},\mathbf{h})$，决定从一个角度看去，有多少能量会被反射
- **Shadowing-masking 项** $\mathbf{G}(\mathbf{i},\mathbf{o},\mathbf{h})$，或者说**G 项**，
- **Normals Distribution 项** $\mathbf{D}(\mathbf{h})$，即微表面法线分布

#### Fresnel 项

当入射角度和法线几乎垂直（即 grazing angle）时，Fresnel 项会反射最多的能量。

![Fresnel_Term_Dielectric](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Fresnel_Term_Dielectric.png)

<div align="center"><i>Fresnel Term (Dielectric, &#951;=1.5)</i></div>

![Fresnel_Term_Conductor](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Fresnel_Term_Conductor.png)

<div align="center"><i>Fresnel Term (Conductor)</i></div>

> 绝缘体是指那些不导电的材质，如塑料、玻璃、水等。在渲染中，绝缘体材质的外观通常由其折射率（Index of Refraction）和衰减系数（Attenuation）决定。折射率描述了光在材质中传播时的弯曲程度，而衰减系数则决定了光线在材质内部的衰减程度。绝缘体材质通常会发生折射、散射和吸收等光的相互作用，使得光线在材质表面和内部产生复杂的效果，如透明、散射、反射等。
>
> 导体是指那些能够导电的材质，如金属。导体材质的外观主要由其电导率（Conductivity）和折射率（Index of Refraction）决定。电导率描述了材质对电流的导电能力，而折射率描述了光在材质中传播时的弯曲程度。导体材质通常会发生反射和吸收等光的相互作用，而不会发生折射。导体材质的外观通常呈现出金属光泽，如镜面反射和漫反射。

Fresnel 项的准确公式需要考虑 s 极化和 p 极化，如下：

![Fresnel_Formula_Accurate](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Fresnel_Formula_Accurate.png)

在 RTR 中，使用 Schlick 近似，有：

$$
R(\theta)=R_{0}+(1-R_{0})(1-\cos{\theta})^{5}\\
R_{0}=(\frac{n_{1}-n_{2}}{n_{1}+n_{2}})^{2}
$$

其中，$\theta$是入射角（光线与法线夹角），$R(\theta)$是反射率，$R_{0}$是基础反射率（带有 RGB 颜色）。当$\theta=90\degree$时（即 grazing angle），$R(\theta)$就是$1$；当$\theta=0\degree$时（正对着看），$R(\theta)$就是$R_{0}$。

#### NDF

现在来聊聊 NDF（Normal Distribution Function），即微表面的法向分布。

![NDF_in_a_Visual_Way](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/NDF_in_a_Visual_Way.png)

有多种方式描述 NDF，介绍一下。

##### Beckmann

Beckmann NDF 是一种常用的表面微平面法线分布函数，用于描述微表面的法线分布情况，有点像高斯分布，其公式如下：

$$
D(\mathbf{h}) = \frac{\exp(-\frac{\tan^2(\theta_{h})}{\alpha^2})}{\pi \alpha^2 \cos^4(\theta_{h})}
$$

其中：$D(\mathbf{h})$ 是 Beckmann NDF 的值；$\theta_{h}$是法向向量（$\mathbf{n}$）和半程向量（Half Vector, $\mathbf{h}$）的夹角；$\alpha$是表面的 roughness，其值越小，表面越接近镜面，可以说和高斯分布中的$\sigma$类似。

之所以指数上面用的是$\tan$，原因在于 Beckmann NDF 是定义在 Slope Space（坡度空间）上的。

![The_Way_You_Define_Beckmann_NDF_in_Slope_Space](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/The_Way_You_Define_Beckmann_NDF_in_Slope_Space.png)

好处是，和高斯一样，积分域都是无穷大，但是其角度的范围一定在$(-\pi/2,\pi/2)$中。这样实际上保证了永远不可能出现面朝下的微表面。直接用角度的话，反而就不能保证这一点了。

至于分母项，是一种归一化。因为我们希望 NDF 在 Projected Solid Angle 上积分为$1$。

##### GGX

GGX（Trowbridge-Reitz）相比于其他法线分布函数，如 Beckmann，具有更好的性质。它在法线分布的尖锐区域和模糊区域之间具有平滑的过渡，因此可以更准确地模拟各种表面的反射特性。公式如下（应该是吧，闫令琪没说 ~~然后我笔记也没记~~）：

$$
D_{\text{GGX}}(\mathbf{h}) = \frac{\alpha^2}{\pi \left(\cos^2(\theta_{h}) (\alpha^2 + \tan^2(\theta_{h}))^2\right)}
$$

其中，$D_{\text{GGX}}(\mathbf{h})$ 是 GGX 模型的值，$\mathbf{h}$ 是半程向量（Half Vector），$\theta_{h}$ 是法向向量（$\mathbf{n}$）和半程向量（$\mathbf{h}$）的夹角，$\alpha$ 是表面的粗糙度（roughness）参数。

![GGX_With_Long_Tail_Comparing_to_Bckmann](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/GGX_With_Long_Tail_Comparing_to_Bckmann.png)

可以看到，同 Beckmann 相比，GGX 具有长尾（Long Tail）的性质，即使到 grazing angle 仍然离$0$比较远。这使得 GGX 渲染的高光过度自然。

##### GTR

Extending GGX，或者说 GTR（Generalized Trowbridge-Reitz）对 GGX 作出了一些改进，主要体现在有着更长的尾巴。

![GTR_Function](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/GTR_Function.png)

对于 GTR 而言，引入参数$\gamma$，当$\gamma=2$时，GTR 退化为 GGX。当$\gamma$值非常大时，GTR 趋近于 Beckmann。

#### Shadowing-Masking 项

也就是$G$项。它解决的是微表面间的自遮挡问题（这个问题在 grazing angle 时尤为严重）。

> 其实这个说法我感觉有点误导，应该这么说：
>
> 自遮挡才是正确的，如果微表面不自遮挡，渲染出来的结果势必非常亮。为什么呢？比如说 grazing angle 的时候，BRDF 式子分母中的入射光、出射光几乎和法向垂直，那么其余弦值几乎为$0$。它们相乘，还在分母，BRDF 的值就贼大，简直是亮的不行。
>
> 所以要通过 Shadowing-Masking 项补偿（或者说 mask）一部分，来使得那一部分变暗到适当的程度。

之所以叫 Shadowing-Masking 项，是因为：

- 当光线照射到微表面上却被微表面自遮挡时，那就 shadowing 了
- 由光路可逆，从微表面出射的光线也会被微表面自遮挡，对于眼睛来说这部分光就被 masked 了

本质上，这两点是一样的。正是因为，微表面自遮挡本身会导致材质一部分变暗，因此引入 Shadowing-Masking 项。我们希望，在正对法向看的时候，$G$项应该接近于$1$；而在 grazing angle 的时候，$G$项应该剧烈地减少到接近于$0$。

##### Smith Shadowing-Masking

说说 Smith 的$G$项。它分开考虑 shadowing 和 masking，即：

$$
G(\mathbf{i},\mathbf{o},\mathbf{m})\approx
G_{1}(\mathbf{i},\mathbf{m})
G_{2}(\mathbf{o},\mathbf{m})
$$

![Smith_Shadowing_Masking](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Smith_Shadowing_Masking.png)

差不多长这样，然后就达成了我们想要的效果。

缺点：能量不守恒。走个白炉测试就知道了。

> 在不考虑 fresnel 项时，在一个各方向各位置 radiance 相同的环境光中放置一个物体，该场景能够测试材质是否满足能量守恒的特性，被称为白炉测试（white furnace test）。
>
> https://airguanz.github.io/2019/03/18/white-furnace-test.html

> 就我的理解来说，其实是布置一个空的背景，给定一个 uniform 的全局光照，使得照出来如果能量守恒，那么反射的颜色就恰好是背景的颜色。

在 roughness 非常大的时候，那么光线就非常容易被遮挡，自然能量损失就巨大。

##### Kulla-Conty BRDF

**Patch**：把丢失的能量补充回来！

在 RTR 中的方法和离线渲染有所不同。基本的思想是，从一个微表面反射出去的光线，要么没有被其他微表面挡住（那就出去了，也就能被成功看到），要么被挡住（这意味着会有后续 bounce，直到它离开微表面），于是“被遮挡”和“发生下次 bounce”实际上可以等同。

这种近似方式就是 Kulla-Conty 近似，用一种经验性的方式补偿多次反射丢失的能量。

如果只考虑一次反射，那么有多少能量离开表面呢？

$$
E(\mu_{o})=
\int_{0}^{2\pi}
\int_{0}^{1}
f(\mu_{o},\mu_{i},\phi)
\mu_{i}\,\mathrm{d}\mu_{i}\,\mathrm{d}\phi
$$

其中，$\mu=\sin{\theta}$。这个公式拿 BRDF，$\cos$项，lighting 项一块，积了个分。首先认为所有入射光入射的 radiance 从任何一个方向$l_{i}$都是$1$。BRDF 假设各向同性（isotropic），即$\phi$和$i$、$o$无关。为什么说积了$\cos$项但是式子里没有呢？~~一点小小的数学震撼~~ 因为如果要把一个球面展开成$\theta$和$\phi$的积分，立体角展开成$\sin{\theta}\,\mathrm{d}\theta\,\mathrm{d}\phi$。这样以后，定义$\mu=\sin{\theta}$，化出来式子就长这样。

这个式子的值介于$0$到$1$之间，也就是说有多少的能量在这一 bounce 中被遮挡掉了。那么，为了补偿这一 part 的丢失，那就加上$1-E(\mu_{o})$即可。

所以我们补一个 BRDF，让它积分出来的值等于$1-E(\mu_{o})$。然后考虑 BRDF 的可逆（或者说对称性），有$c(1-E(\mu_{i}))(1-E(\mu_{o}))$，$c$是一个归一化函数。

> 不这么设计当然可以，因为原则上我们只要一个积出来值对的函数就行了，任何函数积出来值对了都可以。但为什么这么设计呢？因为这样比较简单。~~没看出来~~

然后直接揭晓答案：

$$
f_{ms}(\mu_{o},\mu_{i})=
\frac
{(1-E(\mu_{i}))(1-E(\mu_{o}))}
{\pi(1-E_{avg})}\\
E_{avg}=
2\int_{0}^{1}E(\mu)\mu\,\mathrm{d}\mu
$$

显然，$c$就是分母那套玩意。

推导过程：

$$
E_{ms}(\mu_{o})=
\int_{0}^{2\pi}
\int_{0}^{1}
f_{ms}(\mu_{o},\mu_{i},\phi)
\mu_{i}\,\mathrm{d}\mu_{i}\,\mathrm{d}\phi\\
=2\pi\int_{0}^{1}\frac
{(1-E(\mu_{i}))(1-E(\mu_{o}))}
{\pi(1-E_{avg})}
\mu_{i}\,\mathrm{d}\mu_{i}\\
=2\frac{1-E(\mu_{o})}{1-E_{avg}}
\int_{0}^{1}(1-E(\mu_{i}))\mu_{i}\,\mathrm{d}\mu_{i}\\
=\frac{1-E(\mu_{o})}{1-E_{avg}}(1-E_{avg})\\
=1-E(\mu_{o})
$$

~~昏了。~~

不过话虽如此，$E_{avg}(\mu_{o})=2\int_{0}^{1}E(\mu_{i})\mu_{i}\,\mathrm{d}\mu_{i}$仍然未知（as analytic）。我们可以使用 split sum 中处理难以求解析解积分的方法，比如预计算（打表！）。不过与此同时，我们也不希望维度太高。

那首先这个式子就依赖于观测方向$o$，以及 BRDF。不过 BRDF 太过复杂，只记录 roughness 就够了。这是因为，BRDF 的描述基于 NDF，而 NDF 的性质是由 roughness 决定的。**这样，预计算就只依赖于$\mu_{o}$和 roughness 两个维度了。**

---

如果有颜色，那就是说有额外的能量损失。这时候的做法是：

1.  先考虑没有颜色，那么按照上面的步骤走；
2.  再考虑颜色造成的额外损失，来决定最终颜色。

这依赖于平均 Fresnel 项（也就是决定平均会反射掉多少能量）：

$$
F_{avg}=\frac
{\int_{0}^{1}F(\mu)\mu\,\mathrm{d}\mu}
{\int_{0}^{1}\mu\,\mathrm{d}\mu}\\
=2\int_{0}^{1}F(\mu)\mu\,\mathrm{d}\mu
$$

能被直接看到的部分：$F_{\text{avg}}\cdot E_{\text{avg}}$

而一个 bounce 后，能看见的就是：$F_{\text{avg}}(1-E_{\text{avg}})\cdot F_{\text{avg}}E_{\text{avg}}$

很显然，$k$个 bounces 后，能看见的就是：$F^{k}_{\text{avg}}(1-E_{\text{avg}})^{k}\cdot F_{\text{avg}}E_{\text{avg}}$

把它们加起来，得到 Kulla-Conty 的颜色项：

$$
\frac
{F_{\text{avg}}E_{\text{avg}}}
{1-F_{\text{avg}}(1-E_{\text{avg}})}
$$

我们将它直接乘在不带颜色的 BRDF 项上即可。

### LTC

LTC（Linearly Transformed Cosines）能够帮助我们做微表面 shading。

LTC 主要针对 GGX，不过其他模型也可以。同时，LTC 是做 shading 的，并不考虑做 shadow。

快速 shading 取决于不同类型的光源，LTC 解决的是在微表面模型下，使用多边形光源去 shading 的结果。

LTC 的核心思想是将微表面的 NDF 通过线性变换表示为一个余弦函数。与此同时，对多边形光源（指光源的形状）也可以通过类似的方法，线性变换为余弦表示。好处在于，在一个 cosine lobe 上对变换后的光源算积分，是有解析解的。

我们发现：任何 cosine lobe 可以通过$M$变换成一个 2D 的 BRDF lobe，而反过来，2D BRDF lobe 可以通过$M^{-1}$变换成 cosine lobe。做此变换，那就要将所有的方向都都给变换了，即$\omega_{i}$经过$M^{-1}$变换为了$\omega^{\prime}_{i}$。既然所有的方向都变换了，那么原本的积分域$P$经过$M^{-1}$也就变换为了$P^{\prime}$。

我们假设这个多边形光源中的光是 uniform 的，那么：

$$
L(\omega_{o})=L_{i}\cdot
\int_{P}F(\omega_{i})\,
\mathrm{d}\omega_{i}
$$

因为假设 uniform 了，所以$L_{i}$是一个常数，可以直接拿到渲染方程外面去；BRDF 项和$\cos$项写成一个大$F$，$P$就是多边形光源覆盖的立体角内。

那么，因为：

$$
\omega_{i}=\frac
{M\omega^{\prime}_{i}}
{||M\omega^{\prime}_{i}||}
$$

这个式子就是对方向$\omega$做变换，然后归一化回单位球上。

所以有：

$$
L(\omega_{o})=L_{i}\cdot
\int_{P}\cos(\omega^{\prime}_{i})\,\mathrm{d}
\frac
{M\omega^{\prime}_{i}}
{||M\omega^{\prime}_{i}||}
$$

引入 Jacobian 项，有：

$$
L(\omega_{o})=L_{i}\cdot
\int_{P^{\prime}}\cos(\omega^{\prime}_{i})J
\,\mathrm{d}\omega^{\prime}_{i}
$$

然后它是 analytic 的，也就是说有解析解，可以直接解出来。~~什么数学魔法~~

---

现在就缺这个变换矩阵$M$了。做法是对 BRDF 做预计算。

### Disney Principled BRDF

微表面模型的缺点是：

1.  无法表示世界上的所有材质（比如多层材质，如刷了清漆的木桌子）
2.  不好用，不够 artist-friendly

Disney Principled 的原则是：

- 参数直观一些（别老是物理量，咱美工看不懂喵）
- 参数越少越好
- 最好是$0$到$1$的；为了营造一些特殊效果，倒也可以推到小于$0$或者大于$1$的
- 组合起来也得健壮

优点：

1.  用起来很简单
2.  因为能组合，所以参数空间很大，随便调，表现能力也强
3.  开源实现（算是对各种 BRDF 的拟合吧）

不过它不是那么 physically-based，这问题也不大。

### NPR

非真实感渲染（Non-Photorealistic Rendering, NPR）其实就是风格化。在 RTR 中，NPR 当然也要求快速、可靠。

实时 NPR 通常：

- 来自于真实渲染
- 抽象
- 增强重要部分

---

先说说描边。描的边包括：

- Boundary 边界
- Crease 折痕
- Material edge 材质边界
- Silhouette edge 我把它翻译成剪影边缘。它必须是有多个面共享的边界，因此和 Boundary 区分开来，而且还得是物体外边界上

用 Shading 的做法描 silhoutte 的边，我们知道观察法向如果和法线接近垂直，那么我们几乎就可以断定它就是 silhoutte 的。简而言之，就是造成 grazing angle 的地方。（Watertight 的话）

如果不想要因为设置了 grazing angle 而造成阈值跳变的话，可以用类似 sigmoid 的方式设置，就能有平滑过渡的描边。

缺点是不同位置描边的粗细程度可能不同。

用几何的方式描边，那就把所有背面的面都往外扩一圈，这样渲染到背面的时候都会往外扩一圈，从正面看过去就像是描好边了的样子。至于怎么外扩呢，方法就很多了，不谈。

用图像后处理的方式描边，那就用类似于 Sobel 找出边缘，然后增强就好了。

---

再说说常见的：大量的色块。

其实也很简单，就是对颜色做阈值化就行了。看起来就很卡通；如果想要更好的效果可以在不同的部分选取不同的阈值。

---

说说素描效果。

做法是：先定义好不同密度的素描纹理，再在纹理图上查对应的笔触。之所以这么做，是因为素描效果首先体现了画面上格子的密度，其次我们还希望素描的笔触要比较连续。

为了让素描效果不会因为镜头拉远，而缩小成一团黑色，可以对素描笔触做 Mip-Map，只不过这个 Mip-Map 维持了笔触的密度，也就是说，如果我们把 Mip-Map 每一个子图放大到同样大小，那么它的密度“看上去”反而是不一样的。

![Strokes_Surface_Stylization](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Strokes_Surface_Stylization.png)

## 光追，启动！

光追可以做的事几乎囊括了上文中 RTR 所有的方面，包括软阴影、反射（甚至于到 specular 材质）、环境光遮蔽、全局光照。

光追的本质是 tree traversal，指对光线与场景中的物体相交进行有效处理的过程。光线追踪算法通常使用一种加速结构，如包围盒层次结构（Bounding Volume Hierarchy，BVH）或者 KD-Tree 来组织场景中的物体，以便**快速确定光线与物体的相交关系。**总之这个事情 GPU 不好做。RTX 是加的一块硬件专门做光追。

---

一个概念：spp（sample per pixel）。

![The_Simplest_1_spp_Ray_Tracing](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/The_Simplest_1_spp_Ray_Tracing.png)

如图所示，最简单的 1spp 光路中，每个像素至少要有 3 条光线，包括：

- 从 camera 打到 primary hitpoint 的 rasterization 用的光线（其实效果和光栅化一致，所以未必一定要 trace 一条 primary 光线，不如直接光栅化一遍。因此不称之为 ray，而写的是 rasterization。光栅化和 trace primary ray 等价，但是光栅化显然更快）
- primary hitpoint 的 shadow ray，也就是连到光源看有没有遮挡用的光线
- 从 primary hitpoint 到 secondary hitpoint 的一次 bounce
- 对 secondary hitpoint 发 shadow ray，看看有没有遮挡

Path Tracing 本身是一种 Monte Carlo 积分方法，会有噪声。显然，1spp 的光追，其噪声是极其恐怖的。**所以，RTRT 的核心科技在于降噪（Denoising）。**

### 降噪

目标：

- 质量（不要 overblur，不要 artifacts，保持细节）
- 速度（希望小于 2ms）

评价为几乎不可能。

实现的关键在于时域滤波。总是假设时序上，上一帧是已经 denoised 的了，同时假设整个序列是相当连续的，没有突变。我们希望重用已经处理好的上一帧，来快速实现对当前帧的降噪。

一个概念：Motion Vector。它表示的是同一个地方，从前一帧移动到下一帧的位移。运用 motion vector，可以找到上一帧的对应位置。

由于我们认为场景基本连续，shading 也基本连续，因此上一帧已经降噪好了的可以拿来复用。

---

G-Buffer，这里的 G 表示 Geometry。它可以存包括诸如直接光照（深度）的信息、法向信息、Diffuse Albedo、世界坐标等。**注意：G-Buffer 的信息来自屏幕空间。**

我们认为生成 G-Buffer 是非常轻量级的操作。G-Buffer 在用光栅化代替 primary ray tracing 的时候就顺便拿到了。

### Back Projection

对于当前帧$i$上的像素$x$，我们想知道在$i-1$帧，哪一个像素，它包含了当前帧上像素$x$。Back Projection 就是一种求解的方法。

那么首先我们要知道当前帧$i$上的像素$x$的世界坐标。（这个直接从 G-Buffer 拿就行了，没有的话，世界坐标$s=M^{-1}V^{-1}P^{-1}E^{-1}x$，即对 GAMES101 中的 MVP-视口变换做了一次逆变换；MVP 变换是先$M$后$V$后$P$，因此在这个式子中是先$P^{-1}$，当然深度信息也是要的）

如果从上一帧到当前帧有：$s^{\prime}\xrightarrow{T}s$，那么，当然有：$s^{\prime}=T^{-1}s$

由于整个序列的渲染我们都知道，因此$T$也是知道的。

现在上一帧中$s^{\prime}$的世界坐标已知，我们要求其屏幕空间坐标，使用$x^{\prime}=E^{\prime}P^{\prime}V^{\prime}M^{\prime}s^{\prime}$。$M^{\prime}$、$V^{\prime}$、$P^{\prime}$都是上一帧的 MVP 矩阵，$E^{\prime}$也是上一帧的视口变换矩阵。

### Temporal Denoising

已经通过 Back Projection 得到上一帧的信息了，开始处理当前帧。

约定$\tilde{x}$表示$x$是 unfiltered，而$\bar{x}$表示 filtered。

那么，当前帧（第$i$帧）中，先做空域滤波：$\bar{C}^{(i)}=Filter\left[\tilde{C}^{(i)}\right]$。

这样以后，运用时域的信息，做：$\bar{C}^{(i)}=\alpha\bar{C}^{(i)}+(1-\alpha)\bar{C}^{(i)}$，这就是将上一帧和当前帧做了一个线性的 blending。一般来说，$\alpha$取$0.1$到$0.2$这样。

提到了为什么一张 undenoised 的光追渲染图看起来比较暗，原因是因为很多点的值超过了 255，被直接 clamp 掉了，实际上就丢失了能量。Filtering 本身不会使一张图变亮或者变暗。

### Temporal Failure

- 显然场景一切换（不只是场景，物体、光源等都是，但凡是突变了都是），直接就坏事了（需要 burn-in period）
- 往后走：屏幕空间（从边缘）纳入了越来越多的信息，temporal 肯定是搞不定的
- Disocclusion：随着时间推移，原本被遮挡的东西突然出现了。但是这种情况下，通过 motion vector 找到的就不是正确的像素了（因为在上一帧它还被挡住，自然指向的是深度更浅的东西）

~~其实主要是 Screen Space 的问题（~~

强行复用，会造成 lagging（拖尾）现象。

解决方法主要有两种：clamping 和 detection。但它们都重新引入了噪声（也就是当前帧更 noisy 了）

**Clamping**

对于公式$\bar{C}^{(i)}=\alpha\bar{C}^{(i)}+(1-\alpha)\bar{C}^{(i)}$，我们的做法是：

应用上一帧信息的时候，不管上一帧的值是什么，都将其拉近到足够接近当前帧的结果，再线性 blending。其实就是将上一帧的结果限制到一个范围之内。比如，可以取当前帧当前点的 7x7 范围内，算一个均值、方差，然后 clamp 到两个或三个$\sigma$之内。

**Detection**

用一个类似于`object id`的东西来探测 temporal failure。基本思想是当 motion vector 指向了另一个物体表面时，就认为它不再靠谱。

不再靠谱时，可以做的事包括：调整$\alpha$（因为认为上一帧不再靠谱了）

---

Temporal failure 还会在 shading 中造成问题。

经典例子：

![Temporal_Failure_in_Shading](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Temporal_Failure_in_Shading.png)

背后是一个移动的面光源，柱子不动。这样实际上几何没有任何变化（即地板、柱子、camera 等等），任何像素的 motion vector 严格来说都是零。

这就造成了会重用上一帧阴影的值，于是造成了阴影拖尾，或者说 detached shadows。

在 glossy 场景中，则有：

![Temporal_Failure_in_Shading_Glossy](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Temporal_Failure_in_Shading_Glossy.png)

地板不变时，当上面的物体移动，那么反射出来的东西也会有同样的问题，即 detached / lagging shadows。

总而言之，就是几何没变但 shading 剧烈地变了，motion vector 不足以跟上表达这种变化。

### 空域滤波

显然光时域是不够的，所以空域滤波也是很重要的。在 RTRT 中的空域滤波也是平滑掉 noise，算是蛮 low pass filter 的。

约定：

- 噪声图像表示为$\tilde{\text{C}}$
- 滤波核$K$
- 输出 filtered 的图像$\bar{\text{C}}$

#### Gaussian Filter

解决 Monte Carlo 产生的噪声，最简单的做法是 Gaussian Filter。

闫令琪给出的伪代码：

```python
for each pixel i
    sum_of_weights = sum_of_weighted_values = 0.0
    for each pixel j around i
    	Calculate the weight w_ij = G(|i - j|, sigma)
        sum_of_weighted_values += w_ij * C^{input}[j]
        sum_of_weights + w_ij
    C^{output}[i] = sum_of_weighted_values / sum_of_weights
```

也没什么特殊的，数字图像处理该讲都讲了。

#### Bilateral Filter

Gaussian 滤波的缺点是，整张图会被平等地模糊掉。显然我们更希望能保持一些高频信息，比如边界。

我们可以将“边界”理解为：颜色剧烈变化的地方。

双边滤波的保边的基本思想是：如果两个像素的颜色差距不大，那就按照 Gaussian 的来，否则那就很可能是边界，不希望边界那头会对这头有所贡献。

$$
w(i,j,k,l)=\exp{(-\frac{(i-k)^{2}+(j-l)^{2}}{2\sigma_{d}^{2}}-\frac{||I(i,j)-I(k,l)||^{2}}{2\sigma_{r}^{2}})}
$$

其中$(i,j)$是一个点的坐标，$(k,l)$是另一个。$I(i,j)$表示的是像素的值。这样，当两个像素的颜色差距过大时，就会抑制算出来的权重，使得贡献变小。

然而，仅凭颜色差距未见得能区分好噪声和边界，因此双边滤波还有所欠缺。

#### Joint Bilateral Filter

JBF（联合双边滤波）和 Gaussian Filter（仅考虑距离）、Bilateral Filter（考虑距离和颜色差别）相比，考虑了更多的特征。JBF 特别适用于 Monte Carlo 光线追踪产生的噪声。

通过 G-Buffer，我们有很多特征可以拿来指导滤波，更何况，G-Buffer 本身是完全没有噪声的。

---

说起来，Bilateral Filter 中考虑颜色差距，实际上还是用 Gaussian 函数算的。但是，未必一定要用 Gaussian 函数。

![Different_Functions_to_Guide_the_Filter_Kernel](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Different_Functions_to_Guide_the_Filter_Kernel.png)

<div align="center"><i>一切随着距离衰减的函数都可以</i></div>

应当这样考虑 JBF：

![The_Info_JBF_Uses_In_RTRT_Denoising](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/The_Info_JBF_Uses_In_RTRT_Denoising.png)

从 G-Buffer 中能有深度和法向信息。颜色也有。这三个信息的用法直接看图就行了，很直观。

可以说，多个参数共同影响、贡献，其实就相当于几个 Gaussian 函数相乘。比如 Bilateral 滤波器中，两个数在$\exp$中相减，拆出来就是相乘。控制的$\sigma$就是各自的$\sigma$。

$$
w(i,j,k,l)=\exp{(-\frac{(i-k)^{2}+(j-l)^{2}}{2\sigma_{d}^{2}}-\frac{||I(i,j)-I(k,l)||^{2}}{2\sigma_{r}^{2}})}
$$

---

当 Kernel size 比较大的时候，如何加速？

FFT 到频域上，相乘再 IFFT 回来，对于 GPU 来说不是一个好主意，因为 FFT 虽然在 CPU 上挺快的，但在 GPU 上却不是这样。

一个方法：拆成水平和竖直两趟，如果是 2D Gaussian 函数的话。

可以这么做是因为 Gaussian 函数本身就是用一维 Gaussian 函数定义的：

$$
G_{2D}(x,y)=G_{1D}(x)\cdot G_{1D}(y)
$$

以及，滤波和卷积本质上是一样的，而：

$$
\iint F(x_{0},y_{0})G_{2D}(x_{0}-x,y_{0}-y)
\,\mathrm{d}x\,\mathrm{d}y\\
=\int(\int F(x_{0},y_{0})G_{1D}(x_{0}-x)\,\mathrm{d}x)G_{1D}(y_{0}-y)\,\mathrm{d}y
$$

从计算机科学的角度看，由于分开两趟做，需要多存储一个中间结果，因此相当于是空间换时间。

![Separate_Gaussian_Passes](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/Separate_Gaussian_Passes.png)

可惜这个性质并不是所有函数都有。2D Gaussian 函数可以，但是 Bilateral 滤波核，仅仅是两个 Gaussian 函数相乘，就不那么好拆分了，更别说 JBF 了。

---

第二个加速方法：用逐步增大的 filter size 做多趟。

一个例子是 a-trous wavelet，基本思想是：

- 多趟，每一趟都是 5x5 的滤波器
- 每一趟中的滤波器是有不同的间隔的

![A-Trous-Wavelet](https://cloud.icooper.cc/apps/sharingpath/PicSvr/PicMain/A-Trous-Wavelet.png)

（说起来抽象，不会去看课程视频）

使用大滤波核实际上是要抛弃更多频率，而采样是在搬移频谱。

对于采样来说，采样率决定了“搬移”多远，如果采样比较稀疏，那么频谱和频谱的距离就小，如果特别稀疏，那么频谱就会混叠，表现为走样，或者说 filter 出来有问题。

这个方法每一趟在去除掉一些频率，而巧妙设计的间隔保证了恰好“搬移”频谱的间隔是上一趟 pass 留下来的最高频率的两倍，保证了不会产生走样。~~什么魔法~~

缺点是我们很可能用的不是朴素的 Gaussian 函数，因此原先的高频并没有被完全去除，自然也就打破了上面的分析。

### Outlier Removal

由于 Monte Carlo 产生的噪声可能超级亮（比如说远大于 255），这样滤波器也救不了，因为它的值实在是太强了。它会被滤波器扩的更大（blocky）。

这些超级亮的噪声就是 outlier。我们希望在滤波前就能干掉 outlier，即使这会导致能量不守恒。

检测 outlier 的过程：

1.  对每个 pixel，看它的周围（比如 7x7）
2.  算颜色的均值和方差
3.  那就认为它应该落在$[\mu-k\sigma,\mu+k\sigma]$中，否则就是 outlier

通常$k\in[1,3]$.

至于说 removal，其实不完全算，因为是将其 clamp 到算出来的范围内。

工业界的 clamp 更为复杂。
