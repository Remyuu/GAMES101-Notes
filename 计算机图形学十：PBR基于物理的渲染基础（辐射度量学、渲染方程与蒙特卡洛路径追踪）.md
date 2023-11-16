您好，如果您觉得本站的浏览体验不佳，可以下载本文pdf阅读，谢谢。
[计算机图形学十：PBR基于物理的渲染基础（辐射度量学、渲染方程与蒙特卡洛路径追踪）.pdf](https://remoooo.com/usr/uploads/2023/06/3293924482.pdf)

1. 辐射度量学（Radiometry）
2. 双向反射分布函数（BRDF）
3. 反射方程（The Reflection Equation）
4. 渲染方程（The Rendering Equation）
5. 诺伊曼级数（Neumann series）
6. 蒙特卡洛积分（Monte Carlo Integration）
   1. 简介、定义、估计量无偏证明、高维推广、方差收敛、高维“稀疏性”
   2. 采样策略：反函数法、拒绝采样法 
   3. Py代码实现
7. 蒙特卡洛路径追踪（Monte Carlo Path Tracing）
8. "俄罗斯轮盘赌"（Russian Roulette）
9. Whitted-style VS. Path Tracing
10. 一些前沿的领域Modern Concepts

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/oEZFrVs6ekPDOvG-20230627154517296-20230627154955535.png" alt="image-20230625213627435" style="zoom:50%;" />

<!--more-->

# 计算机图形学十：PBR基于物理的渲染基础（辐射度量学、渲染方程与蒙特卡洛路径追踪）

## 回顾

在前面的文章中，我们实现了Blinn-Phong光照模型，我们还实现了Whitted-style光线追踪。

但是上述计算结果并不完全准确。

### 回顾Blinn-Phong光照模型

Blinn-Phong光照模型主要包括三个部分：环境光照（ambient），漫反射（diffuse）和镜面反射（specular）。

在这个模型中，

- 环境光照通常被用来模拟间接光或没有明确方向的光；
- 漫反射则模拟了光照在粗糙表面上的散射；
- 镜面反射则用来模拟在平滑表面上的高光效果。

但它也有一些**局限性**。首先，它假设表面的反射特性是局部的，忽略了全局的间接光照。其次，它的镜面反射模型也过于简化，不能很好地模拟复杂的高光效果。最后，Blinn-Phong模型对于特定的材质，如金属或透明材质，也往往不能给出准确的结果。

因此，对于复杂和高精度的渲染项目，我们需要另外寻求一种更为准确的模型，物理基础的渲染（PBR）。

### 回顾Whitted-Style光线追踪模型

Whitted-Style光线追踪主要考虑了直接照明（即光从光源直接射到物体上）以及理想的镜面反射和折射。

Whitted-Style光线追踪模型往往适合含有镜面或玻璃等高光材料的场景。

然而，它有一些明显的**局限性**：

1. **不考虑全局照明**：Whitted-Style光线追踪忽略了间接照明的影响，即光从光源射到一个物体，然后反射或折射到另一个物体，再反射或折射到眼睛。这种间接照明在现实世界中占据了很大部分，尤其在室内或阴影区域。
2. **简化的反射和折射模型**：Whitted-Style光线追踪使用了理想的镜面反射和折射模型，这在现实世界中是非常罕见的。大多数物体表面都会散射入射的光线，即使是镜子和玻璃也会有一些微小的不规则性。

因此，虽然Whitted-Style光线追踪在某些情况下表现相当良好，但它不能准确模拟现实世界中的所有光照情况。为了得到更真实的效果，我们可以使用如全局照明算法和物理基础的渲染（PBR）等技术。

## 辐射度量学（Radiometry）

> 上述各种缺点，都能在辐射度量学（Radiometry）之中完美解决！

为了更好更准确地模拟光的行为，研究者们引入了辐射度量学。

首先最基本的框架是：

- **辐照度（Irradiance）辐射度（Radiance）stand for “对象表面的光强属性”**
- **辐射强度（Radiant Intensity）stand for “光源的光强属性”**

辐射度量学是研究和测量光的物理量的科学，以下是辐射度量学的重要学术用语：

1. **辐射能量（Radiant Energy）**：辐射能量是电磁辐射的能量，单位是焦耳（Joule）。
2. **辐射通量（Radiant Flux）**：这是衡量光源总发出能量的物理量，单位是瓦特（Watt）。它不依赖于方向，是光的功率测量。
3. **辐射强度（Radiant Intensity）**：这是在特定方向上的辐射通量，单位也是瓦特。但是，这个概念是方向性的，是单位立体角（Steradian）中的辐射通量。
4. **辐照度（Irradiance）**：这是指单位面积上的辐射通量，单位是瓦特每平方米（Watt per square meter）。它衡量的是光源对物体表面的照度强度。
5. **辐射度（Radiance）**：这是在特定方向上的单位面积的辐射通量，单位是瓦特每平方米每立体角（Watt per square meter per steradian）。这个概念同时考虑了空间和方向的分布，是描述光的强度和方向的最完整的度量。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/2jK3icg9ONRV4pw.png" alt="image-20230625151101027" style="zoom: 33%;" />

> 需要注意的是，4. 5. 并没有合适的中文翻译，不同资料可能有不同的翻译，因此下文直接使用英文。

### 辐射能量（Radiant Energy）

辐射能量是电磁辐射的能量，单位是焦耳（Joule），用 $Q$ 表示：

$$
Q [J = Joule]
$$

辐射能量是一个标量，不考虑方向。

### 辐射通量（Radiant Flux，也称为 Power）

辐射通量是单位时间内发射、反射、传输或接收的辐射能量。

它实际上是辐射能量的流量，可以理解为光的功率。

辐射通量的单位是瓦特（Watt），通常用 $\Phi$ 表示：
$$
\Phi \equiv \frac{\mathrm{d} Q}{\mathrm{~d} t}[\mathrm{~W}=\mathrm{Watt}][\operatorname{lm}=\text { lumen }]^{\star}
$$
和辐射能量一样，辐射通量也是一个标量量，不考虑方向。

### 辐射强度（Radiant Intensity）

这是单位立体角（Steradian）内的辐射通量，即单位方向上的光通量。

单位是瓦特每立体角（Watt per steradian）。

$$
\begin{gathered}
I(\omega) \equiv \frac{\mathrm{d} \Phi}{\mathrm{d} \omega} \\
{\left[\frac{\mathrm{W}}{\mathrm{sr}}\right]\left[\frac{\mathrm{lm}}{\mathrm{sr}}=\mathrm{cd}=\text { candela }\right]}
\end{gathered}
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/pIaxTAE6bshFRZ2.png" alt="image-20230625151650861" style="zoom:50%;" />

#### 单位角和单位立体角（Angles and Solid Angles）

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/jsvASIYVEWmDnyx.png" alt="image-20230625151849501" style="zoom:50%;" />

单位指的是单位圆or单位球，即半径为1。

单位角属于中学知识，此略。

对于一个封闭球面的立体角，球内任意一点的立体角为 $4π(steradian)$ ，球外是 $0(steradian)$ 。

#### 立体角推导

首先定义 $\theta , \phi $ 分别是向量与垂直轴的夹角和向量在底平面上投影向量与其中一轴的夹角。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/SkQpGiCWFymMn8d.png" alt="image-20230625153225065" style="zoom: 33%;" />

根据二维角的定义，直接写出球面上的微小元 $dA_j$ ：

$$
dA_j = r^2 sin(\theta)d \phi
$$

再根据极小立体角的定义，得到 $\omega$ ：

$$
\mathrm{d} \omega=\frac{\mathrm{d} A_j}{r^2}=\sin \theta \mathrm{d} \theta \mathrm{d} \phi
$$

对一个极小立体角在一整个球面做积分：

$$
\begin{aligned}
\Omega & =\int_{S^2} \mathrm{~d} \omega \\
& =\int_0^{2 \pi} \int_0^\pi \sin \theta \mathrm{d} \theta \mathrm{d} \phi \\
& = \int_0^\pi \sin \theta d \theta \int_0^{2 \pi} d \varphi \\
& =[-\cos \theta]_0^\pi(2 \pi)\\
& =4 \pi
\end{aligned}
$$

立体角的国际制单位是球面度（steradian，sr）。

其中， $\omega $ 的几何意义是一个单位向量。 $\omega $ 在一开始就已经由 $\theta , \phi$ 所决定。随后在此两个角的基础上增加微小增量 $d\theta , d\phi$ ，得到 $d\omega$ 。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/EbpFLYDJBXHyTl8.png" alt="image-20230625160405622" style="zoom:33%;" />

### 辐照度（Irradiance）

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/gJvSzUtkf8A2njh.png" alt="image-20230625164827083" style="zoom:50%;" />

照度（Irradiance）是指单位时间内射入或者射出某一平面单位面积的光的通量（即功率）。

单位为瓦特/平方米（ $W/m^2$ ）：

$$
\begin{gathered}
E(\mathbf{x}) \equiv \frac{\mathrm{d} \Phi(\mathbf{x})}{\mathrm{d} A} \\
{\left[\frac{\mathrm{W}}{\mathrm{m}^2}\right]\left[\frac{\operatorname{lm}}{\mathrm{m}^2}=\operatorname{lux}\right]}
\end{gathered}
$$

描述物体表面被光源照亮的程度。

借助此概念也可以轻松得到Lambert's Cosine Law的照度E。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/hwu4g6JrvWHeA2a.png" alt="image-20230625161426594" style="zoom:50%;" />

### 辐射度（Radiance）

辐射度的定义是：在**特定方向上**，通过特定表面区域的辐射通量（Radiant Flux）的密度。

换句话说，辐射度描述了**在给定方向上**，每单位面积通过的光通量。

在定义中关于接收面积的部分是**每单位垂直面积**，这就是为什么定义中有一个余弦量。

辐射度的单位是瓦特每平方米每立体角（Watt per square meter per steradian，W·m⁻²·sr⁻¹）：

$$
\begin{gathered}
L(\mathrm{p}, \omega) \equiv \frac{\mathrm{d}^2 \Phi(\mathrm{p}, \omega)}{\mathrm{d} \omega \mathrm{d} A \cos \theta} \\
{\left[\frac{\mathrm{W}}{\mathrm{srm}^2}\right]\left[\frac{\mathrm{cd}}{\mathrm{m}^2}=\frac{\operatorname{lm}}{\mathrm{sr} \mathrm{m}^2}=\mathrm{nit}\right]}
\end{gathered}
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/aVK1QEPjhdiu7IZ.png" alt="image-20230625161933924" style="zoom:50%;" />

### 区分辐照度和辐射度

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/qwAISxeQnFsX3gr.png" alt="image-20230625162635792" style="zoom:50%;" />

- 辐照度考虑 $dA$ 面积接受到的所有能量；

- 辐射度则只考虑考虑 $dA$ 面积接受到的某个方向 $\omega$ 的能量。

下面公式第一行中，等号左边是辐照度（Irradiance）的微分，右边是辐射度（Radiance）。

将第一行左右两边积分，得到第二行辐照度（Irradiance）。

$$
\begin{aligned}
d E(\mathrm{p}, \omega) & =L_i(\mathrm{p}, \omega) \cos \theta \mathrm{d} \omega \\
E(\mathrm{p}) & =\int_{H^2} L_i(\mathrm{p}, \omega) \cos \theta \mathrm{d} \omega
\end{aligned}
$$

其中，

- $E(p)$ 的物理含义是点p上每单位照射面积的功率，即辐照度（Irradiance）；
- $L_i(p,\omega)$ 指入射光每立体角，每垂直面积的功率

因此积分式子右边的 $cos\theta$ 解释了面积上定义的差异。

而对 $d\omega$ 积分，则是相当于对所有不同角度的入射光线做一个求和。

辐照度（Irradiance）是由所有不同方向的辐射度（Radiance）累加得到。

简而言之，下图中的 $d A$ 是Irradiance中定义所对应的， $dA^{\perp}$  是Radiance中所定义。即：

$$
dA^\perp=dAcod\theta
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/HsiFVNRLG1CEBwX.png" alt="image-20230625162457138" style="zoom:50%;" />

那么结论就是：

$$
\mathrm{L_i}(\mathrm{p}, \omega)=\frac{\mathrm{dE}(\mathrm{p})}{\mathrm{dA^\perp}}=\frac{\mathrm{dE}(\mathrm{p})}{\mathrm{d} \omega \cos \theta}
$$

## 双向反射分布函数（BRDF）

通过上文辐射度量学介绍，我们可以利用一个函数描述光线的反射。

具体来说，光线从某个方向打到一个点上，然后反射出去，且考虑反射后光线的能量是多少。

BRDF描述的是：

在一个微小面积元上接受到了某个方向 $\omega_i $ 能量是 $E$ 的入射光，我们将该点的辐照度（Irradiance） $\omega_i$方向上的辐射度（Radiance）记为：

$$
d E\left(\omega_i\right)=L\left(\omega_i\right) \cos \theta_i d \omega_i
$$

然后将反射到某个方向 $\omega_r$ 的辐射度（Radiance）记为：

$$
d L_r\left(x, \omega_r\right)
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/cCs1EOXqLBQrIYj.png" alt="image-20230625172128669" style="zoom:50%;" />

反射出去的光线的Radiance（ $d L_r\left(x, \omega_r\right)$ ）通过BRDF计算。

$$
f_r\left(\omega_i \rightarrow \omega_r\right)=\frac{\mathrm{d} L_r\left(\omega_r\right)}{\mathrm{d} E_i\left(\omega_i\right)}=\frac{\mathrm{d} L_r\left(\omega_r\right)}{L_i\left(\omega_i\right) \cos \theta_i \mathrm{~d} \omega_i} \quad\left[\frac{1}{\mathrm{sr}}\right]
$$

BRDF接受两个参数：入射光方向 $\omega_i$ ，反射光方向 $\omega_r$ 。

函数值为反射光的radiance与入射光的iiradiance的比值，上式第二个等号分母中利用iiradiance和radiance的关系展开。

抛开数学公式，BRDF描述的是光线和物体是如何作用的，会决定不同的物体材质。

借助BRDF，我们可以定义反射方程（The Reflection Equation）。

## 反射方程（The Reflection Equation）

在BRDF中，我们可以知道，在某微小面元在入射光某一方向射入之后吸收能量，然后反射能量在另一个方向 $w_r$ 上。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/KkqizIwm9nvUl4Q-20230627155052233.png" alt="image-20230625204500403" style="zoom:50%;" />

我们现在研究由不同方向的入射光线的Irradiance对 $w_r$ 的贡献。

将每一个入射方向的光线的Irradiance都乘上对应的BRDF函数，就得到了在这个微小面元上所有的入射方向反射到 $w_r $ 的辐射度Radiance，公式如下：

$$
L_r\left(\mathrm{p}, \omega_r\right)=\int_{H^2} f_r\left(\mathrm{p}, \omega_i \rightarrow \omega_r\right) L_i\left(\mathrm{p}, \omega_i\right) \cos \theta_i \mathrm{~d} \omega_i
$$

至此，通过反射方程和辐射度量学的概念，结合双向反射分布函数（BRDF），我们拥有了一个更准确和完善的光照模型，并且解决了文章开头提出的问题！

----

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/TZAxeRpqKsNDiB7.png" alt="image-20230625210032121" style="zoom:50%;" />

但是只要我们想要通过代码实现，就会发现反射方程面临一项重要的挑战：**这是一个递归（或者说迭代）的方程**。

反射的光照取决于入射的光照，但入射的光照又可能是其他表面反射的结果。这种递归的关系可能会一直持续下去，直到达到一些光源，这些光源的光照不依赖于其他表面的反射。

一般来说，我们的解决方法是：**使用递归的光线追踪算法**。每次一个光线撞到一个表面，我们就生成一个新的光线，这个新的光线的方向是根据BRDF和入射光线的方向确定的。然后我们追踪这个新的光线，直到它撞到一个光源或者超过了预设的递归深度限制。

然而，这种递归光线追踪算法的计算成本是非常高的，特别是对于处理全局照明和复杂的反射/折射效果的场景。

但是在此之前，我们引入渲染方程的概念，完善现有的反射方程概念，如添加自发光项等。

最后再研究怎么优化算法。

## 渲染方程（The Rendering Equation）

渲染方程是由James Kajiya在1986年提出的，它描述了在给定光照条件下，光如何与场景中的物体交互，以及如何计算最终的图像。在论文《The Rendering Equation》中给出了入射光的函数和BRDF（双向反射分布函数）。

> 此外，还有一些现代的图形学和机器学习研究中，他们使用或者扩展了这个渲染方程。例如，一些用于视图合成的研究中，提出了在特征空间而不是颜色空间编码渲染方程的方法，以便更好地模拟复杂的视图依赖效应。还有一些研究使用了Neural Radiance Fields (NeRF) 方法，通过在光线上取样点并使用渲染方程整合信息来渲染单独的光线。
> 
> 以下是相关资料：
> 
> 1. https://dl.acm.org/doi/10.1145/15886.15902
> 2. https://ar5iv.org/abs/2303.03808
> 3. https://ar5iv.org/abs/2106.05264

渲染方程在反射方程的基础上添加了一个自发光项（Emission term）：

$$
L_o\left(p, \omega_o\right)=L_e\left(p, \omega_o\right)+\int_{\Omega^{+}} L_i\left(p, \omega_i\right) f_r\left(p, \omega_i, \omega_o\right)\left(n \cdot \omega_i\right) \mathrm{d} \omega_i \\ 
\tag{1}
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/mBwR7drTV3U59oK.png" alt="image-20230625213525244" style="zoom:50%;" />

- 无论是面光源还是物体，都相当于无穷多个点光源的集合，因此我们理论上就可以对所有方向进行积分

我们将上面的方程简写成：

$$
I(u)=e(u)+\int I(v) K(u, v) d v
$$

where:

- $I(u)$ 是辐射出去的光

- $e(u)$ 是自发光项

- $I(v)$ 是其他物体的表面发出射入到$u$的光

- $K(u, v) d v$则是将通过其他物体射进来的光乘上BRDF反射到摄像机。

----

### Neumann Series

诺伊曼级数（Neumann series）是一种在函数分析和线性代数中广泛使用的级数。

If $\lim _{n \rightarrow \infty} \mathbf{A}^n=\mathbf{0}$, then $\mathbf{I}-\mathbf{A}$ is nonsingular and

$$
(\mathbf{I}-\mathbf{A})^{-1}=\mathbf{I}+\mathbf{A}+\mathbf{A}^2+\cdots=\sum_{k=0}^{\infty} \mathbf{A}^k
$$

其中，I 是单位矩阵，A 是任意的线性映射或者矩阵。

详细资料查看：[Courant and Hilbert 1953]

----

回到渲染方程，通过Neumann Series，我们可以将渲染方程 eq.(1) 写成如下形式：

$$
I=g \epsilon+g M I
$$

 $M$ 是由 eq.(1) 给出的线性算子，现在重写方程：

$$
(1-g M) I=g \epsilon
$$

由于1是identity operator，将方程正式写为：

$$
\begin{aligned}
I & =(1-g M)^{-1} g \epsilon \\
& =g \epsilon+g M g \epsilon+g M g M g \epsilon+g(M g)^3 \epsilon \cdots
\end{aligned}
\tag{2}
$$

where:

- 第一项是**Emission directly from light sources**: 这是直接光源发射的光线，例如从灯泡或太阳等主动发光的物体发出的光线。
- 第二项是**Direct Illumination on surfaces**: 这是由直接光源导致的直接照明。例如，阳光直接照在地面上，或者室内灯光直接照在墙壁上，都是直接照明。
- 第三项是**Indirect Illumination (One bounce indirect)**: 这是由于光线从光源出发，反射或折射一次后到达表面的照明。例如，光线从墙壁反射到附近的桌子上，或者从镜子反射到墙壁上，这些都属于一次反弹的间接照明。在这种情况下，镜子和玻璃等透明或反射表面都会产生这种效应。
- 第四项及以后是**Two bounce indirect illum**: 这是光线经过两次或更多次反弹后照亮的表面，即多次反弹的间接照明。例如，光线可以从一面墙反射到另一面墙，然后再反射到天花板上，这就是二次反弹的间接照明。在复杂的环境中，光线可能经过多次反弹，这就需要进行多次迭代计算。

因此这也正是渲染方程的物理意义。

效果（摘自https://dl.acm.org/doi/10.1145/15886.15902）：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/oEZFrVs6ekPDOvG-20230627154517296-20230627154955535.png" alt="image-20230625213627435" style="zoom:50%;" />

## 蒙特卡洛积分（Monte Carlo Integration）

> 上面的内容我们通过辐射度量学引入了渲染方程，提出了一个完全基于物理的光线传播模型，但是没有涉及如何求解/实现。具体来说，就是不会通过渲染方程计算像素。接下来，将会使用蒙特卡洛路径追踪实现这个目标。首先介绍蒙特卡洛积分（Monte Carlo Integration）。

### 蒙特卡洛积分简介

蒙特卡洛是一类算法的统称，蒙特卡洛积分只是其中的一个应用。

蒙特卡洛（Monte Carlo）积分是一种数值积分方法，它使用随机采样来估计一个函数的积分。这种方法非常适合处理那些难以找到解析解（或者说，无法用封闭形式表示的解）的复杂积分问题。

蒙特卡洛积分的一个关键特点是，它并不依赖于被积函数的形式，只要我们能从定义域中随机采样，并能计算函数值，就能使用蒙特卡洛方法来估计积分。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/qPmEjilvQ96K7Ik.png" alt="image-20230625232126824" style="zoom:50%;" />

以下是蒙特卡洛积分的基本思想：

考虑一个函数 $f(x)$，我们想要计算在某个区间$[a, b]$上的积分$∫_a^b f(x) dx$。如果直接计算这个积分很困难，我们可以使用蒙特卡洛方法。

首先，我们在区间$[a, b]$内随机选择$n$个点$x_i$，然后计算这些点上的函数值$f(x_i)$。然后，我们对这些函数值求平均，得到一个估计值。

然后，我们将这个估计值乘以区间的长度(b - a)，得到最终的积分估计值。

以上就是蒙特卡洛积分的基本思想。蒙特卡洛积分的精度取决于样本点的数量：样本点越多，积分的估计值就越接近真实的积分值。但是，增加样本点的数量也会增加计算的复杂性。现实开发中一般会寻求一个平衡。

这时候你可能会想到黎曼积分，没错！如果对上图区间$[a,b]$进行**均匀采样**的话，就相当于将整个积分面积分成多个矩形，然后将这些矩形的面积全部加起来，且如果这里取无穷多个点，就是黎曼积分。

另外，等间隔的采样也等同于一阶欧拉积分。

### 蒙特卡洛积分定义

我们希望求出一个函数 $f(x)$ 在积分域 $[a,b] $ 上的积分值：

$$
\int_a^b f(x) d x
$$

选定一个采样的分布 $ p(x)$ ：

$$
X_i \sim p(x)
$$

通过该分布进行多次函数值的采样：

$$
F_N=\frac{1}{N} \sum_{i=1}^N \frac{f\left(X_i\right)}{p\left(X_i\right)}
$$

By the way，为什么公式中出现了 $\frac{1}{p(X_i)}$ ？

个人的理解是，蒙特卡洛积分的数学期望不应该和采样方法有关。举个简单的例子，有一个100个元素的数组，其中有50个0，50个1。

利用MC积分我们希望最终求得的期望 $E=0.5$ 。

但，

- 如果我们的采样方法会更偏向于采样“1”，那么最终的期望 $E > 0.5$ ；
- 如果我们的采样方法会更偏向于采样“0”，那么最终的期望 $E < 0.5$ ；

也就是说，为了去掉具体采样策略导致的偏差，我们需要把每个点把对应的概率除掉，总的来看就是除以该点的概率密度函数。

### 蒙特卡洛积分估计量无偏证明

欲证明估计量无偏，即证蒙特卡洛法的积分估计量的数学期望等于被积函数的积分真值，即证明下式成立：

$$
F_N=\int f(x) d x
$$

证：

$$
\begin{aligned}
& 假设X_i(i=1,2,3,...,N)是 \Omega 内的随机变量。\\
& \Omega 取值在ab之间。
\end{aligned}
$$

构造：

$$
F_N=\frac{1}{N} \sum_{i=1}^N \frac{f\left(X_i\right)}{p\left(X_i\right)}
$$

取：

$$
Y = \frac{f(X)}{pdf(X)}
$$

计算蒙特卡洛法积分的数学期望：

$$
\begin{aligned}
& E\left[F_N(X)\right] \\
& =E\left[\frac{1}{N} \sum_{k=1}^N \frac{f\left(X_k\right)}{p d f\left(X_k\right)}\right] \\
& =\frac{1}{N} \sum_{k=1}^N E\left[\frac{f\left(X_k\right)}{p d f\left(X_k\right)}\right] \\
& =\frac{1}{N} \sum_{k=1}^N E\left[Y_k\right] \\
& =\frac{1}{N} \sum_{k=1}^N \int_a^b \frac{f(x)}{p d f(x)} \cdot p d f(x) d x \\
& =\frac{1}{N} \sum_{k=1}^N \int_a^b f(x) d x \\
& =\int_a^b f(x) d x
\end{aligned}
$$

可以得到的结论是，由于估计量的数学期望等于被估计参数的真实值，所以我们可以说，蒙特卡洛积分的估计是**无偏的**。

### 蒙特卡洛积分高维推广

我们也可以将定义域拓展至任意维度 $\Omega$ ，方法如下：

$$
\begin{aligned}
& I=\int_{\Omega} f(x) d \mu(x) \\ \\
& F_N=E\left[\frac{1}{N} \sum_{i=1}^N \frac{f\left(X_i\right)}{p_{\left(X_i\right)}}\right] \\
& =\frac{1}{N} \sum_{i=1}^N \int \frac{f(x)}{p(x)} p(x) d \mu(x) \\
& =\int_{\Omega} f(x) d \mu(x) \\
& =I \\
& 其中， f(x) \ne 0
\end{aligned}
$$

### 蒙特卡洛积分方差收敛

分析随着样本增加，数学期望 $F_N$ 的方差 $\sigma^2 [F_N]$ 的变化，以此得到MC积分方法收敛速度特性。

$$
\begin{aligned}
&\begin{aligned}
\sigma^2\left[F_N\right] & =\sigma^2\left[\frac{1}{N} \sum_{i=1}^N \frac{f\left(X_i\right)}{p\left(X_i\right)}\right] \\
& =\frac{1}{N^2} \sum_{i=1}^N \sigma^2\left[\frac{f\left(X_i\right)}{p\left(X_i\right)}\right] \\
& =\frac{1}{N^2} \sum_{i=1}^N \sigma^2\left[Y_i\right] \\
& =\frac{1}{N^2}\left(N \sigma^2[Y]\right) \\
& =\frac{1}{N} \sigma^2[Y]
\end{aligned}\\
可得：\\
&\sigma\left[F_N\right]=\frac{1}{\sqrt{N}} \sigma[Y]
\end{aligned}
$$

这就是蒙特卡洛积分的"根n收敛"性质。

- **积分估计量的收敛与被积函数的维度等都无关，只跟样本数有关。**

估计误差通常按照样本数$n$的平方根进行减小，即标准误差（误差的标准差）和$\frac{1}{\sqrt{n}}$成正比。这是大数定律和中心极限定理的结果。

所以我们说，MC积分提高精度的速度并不快。比如，要使标准误差减小一半，我们需要增加样本数量到原来的四倍。

大多数数值方法（如牛顿法、梯度下降法等）都依赖于函数的平滑性或者连续性，如果函数存在不连续的地方，这些方法可能就会出问题。

但是MC方法积分的基本原理是：随机抽样，然后计算样本的平均值来估计积分。这个过程不需要对函数进行微分或者插值，所以即使函数不连续或者不平滑，也不会影响蒙特卡洛方法的应用。

### 高维"稀疏性"问题

在高维空间中，随机投点可能会导致大部分样本都分布在空间的边缘区域，而不是在我们关心的函数值较大的区域。这会导致估计的不准确，并且需要非常多的样本才能得到一个好的估计。比如说现在我们有100个采样点均匀分布在样本空间中，

- 如果维度是1，那么每个维度分配有100个样本点
- 如果维度是2，那么每个维度分配有10个样本点
- 如果维度是3，那么每个维度分配只剩下约4.641589个样本点
- 如果维度是4，那么每个维度分配只剩下约3.16227766个样本点

蒙特卡洛方法提供了一些减少方差的技术来解决这个问题，如重要性采样（Importance Sampling）、策略梯度（Policy Gradients）、马尔科夫链蒙特卡洛（Markov Chain Monte Carlo）等。这些方法的主要思想是改变样本的抽样分布，使得更多的样本落在我们关心的区域，从而提高估计的准确性。

### MC采样点选择策略

蒙特卡洛方法的关键步骤之一就是抽样，即从一个概率分布中生成观察值。在理论上，这个概率分布可以是任何形式的，但在实际应用中，我们通常需要能够方便地从这个分布中抽样。

在计算机中，最容易实现的抽样方法就是对均匀分布的抽样，因为它不需要复杂的计算。而对于其他形式的概率分布，我们通常需要转化成对均匀分布的抽样。这个转化过程可以通过各种不同的方法实现，如反函数法、拒绝法、马尔科夫链蒙特卡洛方法等。

接下来我们简单讲解前两种。

### 选择策略1. 反函数法（Inverse Transform Sampling）

反函数法（Inverse Transform Sampling）是一种从已知的一维分布生成随机数的方法，其基础在于累积分布函数（CDF）的性质。

#### 基本原理

对于随机变量$X$，其累积分布函数（CDF，Cumulative Distribution Function）定义为：

$$
F(x)=P\{X \leq x\}, \quad-\infty<x<\infty
$$

其中$F(x)$的值域为 $[0,1]$ ，是一个非递减函数。

设U为一个在 $[0,1]$ 上的均匀随机变量，我们可以证明，如果$F$是连续的，那么 $F^{-1}(U)$ 的分布函数就是 $F(x)$ ，也就是说 $F^{-1}(U)$ 与 $X$ 有相同的分布。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/2JFwqHrW8zXx9Kv.png" alt="image-20230626112212616" style="zoom:50%;" />

也就是说，通过从均匀分布中抽取随机数$U$，然后将$U$输入到CDF的反函数$F^{-1}$中，我们就可以得到一个服从目标分布的随机数。

#### 证明过程[摘自Wiki](https://en.wikipedia.org/wiki/Inverse_transform_sampling#Proof_of_correctness)：

----

Let $F$ be a cumulative distribution function, and let $F^{-1}$ be its generalized inverse function (using the infimum because CDFs are weakly monotonic and right-continuous): 

$$
F^{-1}(u)=\inf \{x \mid F(x) \geq u\} \quad(0<u<1) .
$$

Claim: If $U$ is a uniform random variable on $[0,1]$ then $F^{-1}(U)$ has $F$ as its CDF.
Proof:

$$
\begin{aligned}
& \operatorname{Pr}\left(F^{-1}(U) \leq x\right) \\
& =\operatorname{Pr}(U \leq F(x)) \quad\left(F \text { is right-continuous, so }\left\{u: F^{-1}(u) \leq x\right\}=\{u: u \leq F(x)\}\right) \\
& =F(x) \quad \text { (because } \operatorname{Pr}(U \leq u)=u, \text { when U is uniform on }[0,1])
\end{aligned}
$$

----

#### 步骤

1. 首先，知道抽样的分布的累积分布函数（CDF）以及它的反函数。
2. 然后，从$[0, 1]$的均匀分布中抽取一个随机数$U$。
3. 最后，计算$F^{-1}(U)$，这个值就是一个服从目标分布的随机数。

需要注意的是：

- 此方法要求能够容易地计算CDF的反函数。

### 选择策略2. 拒绝采样法（Rejection Sampling）

当我们不容易计算CDF的反函数时，我们可以采用拒绝采样法（Rejection Sampling）。

#### 基本原理

拒绝采样的基本思想是构造一个易于抽样的参考分布（proposal distribution），然后以一种特定的方式从该分布抽样，并"拒绝"掉一些不符合目标分布特性的样本。其目的是得到一个符合目标概率分布特性的样本集。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/BQnKolpWt3MjH2E.png" alt="image-20230626120035004" style="zoom:50%;" />

直白地讲，我们建立一个参考分布$q(x)$，然后采样一些参考分布上的点，然后判断采样出来的样本点是否符合原来的概率密度函数$p(x)$，如果符合则"保留"，不符合就"拒绝"。

#### 过程

1. 首先，选择一个我们能方便从中抽样的参考分布（proposal distribution）$q(x)$。
2. 然后，从这个参考分布$q(x)$中抽取一些样本。
3. 对于每一个抽取到的样本，我们会计算一个接受概率，这个接受概率是目标分布$p(x)$在这个点的值与参考分布$q(x)$在这个点的值的比值，再乘以一个缩放因子M的倒数。
4. 接着，我们再生成一个$[0,1]$之间的均匀随机数。如果这个随机数小于等于前面计算的接受概率，那么我们就接受这个样本；否则，我们就拒绝这个样本。

比较直观的解释是，上下两个曲线接近的地方接受概率较高，也即在这样的地方采到的点就会比较多，反之。

### More todo...

> 这里其实还有很多内容需要学习，但是都相对复杂，放到其他文章讲解吧。
> 
> ---- 马尔可夫链蒙特卡洛
> 
> -- 方差缩减

### 蒙特卡洛积分优缺点

优点：

1. **适应性**：蒙特卡洛方法不依赖于问题的维度，所以非常适合于处理高维度问题。
2. **简单易实现**：基本思想和实现都相对简单。
3. **只需要函数值**：蒙特卡洛方法不需要知道函数的导数或者其他高阶信息，只需要能计算函数在采样点的函数值。

缺点：

1. **收敛速度慢**：蒙特卡洛积分的收敛速度为"根n"（其中n是采样次数），这意味着为了增加一个精确位数，需要增加10倍的采样数。
2. **方差可能大**：如果函数在定义域的某些区域内变化剧烈，而这些区域又很难被采样到，那么估计的方差可能会非常大，这会影响到估计的精确度。
3. **依赖于随机数的质量**：蒙特卡洛方法的精度和稳定性高度依赖于生成的随机数的质量。如果随机数的质量不好（如分布不均匀、存在相关性等），可能会导致估计结果的偏差和误差增大。

### 蒙特卡洛积分代码实现

用python简单实现一个被积函数为 $x^3$ 的蒙特卡洛积分 $[2,5]$ 估计的例子：

![image-20230626001659060](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/3KpHydM1oJzTUAl.png)

- 根据定积分得到积分结果是 $\frac{609}{4} = 152.25$ .

这里分别使用四种随机采样方法采样，分别是：

1. 均匀采样
2. 正态分布
3. Halton序列
4. Sobol序列

----

这里采用的是**均匀采样**，python代码如下：

```python
import numpy as np
# 定义被积函数
def f(x): return x**3
# 定义积分的区间
a, b = 2, 5  # 可以根据需要更改
# 定义样本数量
N = 10000000  # 可以根据需要更改
# 生成区间[a, b]上的随机样本
X = np.random.uniform(low=a, high=b, size=N)
# 计算样本上的函数值
Y = f(X)
# 计算平均值，并乘以区间长度，得到积分的蒙特卡洛估计值
integral_estimate = (b - a) * np.mean(Y)
print(f"蒙特卡洛估计值：{integral_estimate}")
```

> 结果输出：
> 
> 蒙特卡洛估计值：152.25343412130383

还算是准确。

我们也可以用**正态分布**采样：

```python
import numpy as np
import scipy.stats as stats
...
# 生成正态分布的随机样本
mu = (a + b) / 2  # 均值，区间中点
sigma = 0.5  # 标准差，可以根据需要调整
X = np.random.normal(loc=mu, scale=sigma, size=N)
# 对于落在区间外的样本，忽略它们
X = X[(X >= a) & (X <= b)]
# 计算样本上的函数值，注意乘以正态分布的概率密度函数
Y = f(X) / stats.norm.pdf(X, mu, sigma)
# 计算平均值，注意不再乘以区间长度，因为正态分布是在全实数轴上定义的
integral_estimate = np.mean(Y)
print(f"蒙特卡洛估计值：{integral_estimate}")
```

需要注意，由于我们使用正态分布采样，部分样本会落在$[2, 5]$区间外，我们在计算前选择忽略这部分样本。这样的处理在统计学中称为截尾（truncation）。

> 结果输出：
> 
> 蒙特卡洛估计值：152.67692954205035

**Halton序列**的python代码：

```python
import numpy as np
import ghalton
...
# 初始化一个 Halton 序列生成器
sequencer = ghalton.Halton(1)
# 生成 Halton 序列，并缩放到区间 [a, b]
X = np.array(sequencer.get(N)).flatten() * (b - a) + a
# 计算样本上的函数值
Y = f(X)
# 计算平均值，并乘以区间长度，得到积分的蒙特卡洛估计值
integral_estimate = (b - a) * np.mean(Y)
print(f"蒙特卡洛估计值：{integral_estimate}")
```

> 结果输出：
> 
> 蒙特卡洛估计值：152.24989931167778

**Sobol序列**重写的python代码：

```python
import numpy as np
from scipy.stats import qmc
...
# 生成 Sobol 序列，并缩放到区间 [a, b]
sobol_engine = qmc.Sobol(d=1, scramble=False)
X = sobol_engine.random(N) * (b - a) + a
# 计算样本上的函数值
Y = f(X)
# 计算平均值，并乘以区间长度，得到积分的蒙特卡洛估计值
integral_estimate = (b - a) * np.mean(Y)
print(f"蒙特卡洛估计值：{integral_estimate}")
```

> 结果输出：
> 
> 蒙特卡洛估计值：152.24996719136274

## 蒙特卡洛路径追踪（Monte Carlo Path Tracing）

回顾上文讲的 eq.(1) 渲染方程（The Rendering Equation）：

$$
L_o\left(p, \omega_o\right)=L_e\left(p, \omega_o\right)+\int_{\Omega^{+}} L_i\left(p, \omega_i\right) f_r\left(p, \omega_i, \omega_o\right)\left(n \cdot \omega_i\right) \mathrm{d} \omega_i \\ 
\tag{1}
$$

其中，这个方程涉及以下两点：

1. 求解半球积分较为困难
2. 方程涉及递归的形式

这个时候就可以用蒙特卡洛积分求解该方程了。

计算之前，我们先将发光项舍去，方便计算。也就是将 eq.(1) 改写为 eq.(3) ：

$$
L_o\left(p, \omega_o\right)=\int_{\Omega^{+}} L_i\left(p, \omega_i\right) f_r\left(p, \omega_i, \omega_o\right)\left(n \cdot \omega_i\right) \mathrm{d} \omega_i \\ 
\tag{3}
$$

将 eq.(2) 应用蒙特卡洛积分得到：

$$
\begin{aligned}
L_o\left(p, \omega_o\right) & =\int_{\Omega^{+}} L_i\left(p, \omega_i\right) f_r\left(p, \omega_i, \omega_o\right)\left(n \cdot \omega_i\right) \mathrm{d} \omega_i \\
& \approx \frac{1}{N} \sum_{i=1}^N \frac{L_i\left(p, \omega_i\right) f_r\left(p, \omega_i, \omega_o\right)\left(n \cdot \omega_i\right)}{p\left(\omega_i\right)}
\end{aligned}
$$

在此不妨考虑最简单的情况，对半球面 $H^2$ 进行均匀采样，此时pdf就是：

$$
pdf(\omega_i) = \frac{1}{2 \pi}
$$

我们目前只考虑直接光照，也就是说，只有采样方向 $w_i$ ：

```
shade(p, w_o)
    Randomly choose N directions wi~pdf
    L_o = 0.0
    For each w_i
       Trace a ray r(p, w_i)
       If ray r hit the light
          L_o += (1 / N) * L_i * f_r * cosine / pdf(w_i)
    Return L_o
```

计算在某一点p，朝向某一方向$w_o$的光线的颜色（或亮度）$L_o$。通过模拟光线从p点出发，随机散射到各个方向，然后看这些方向是否能打到光源（light）

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/qwAISxeQnFsX3gr-20230627155150509.png" alt="image-20230625162635792" style="zoom:50%;" />

但是目前还不够完善，只有直接光是不够的，我们需要添加间接光，也就是光线打到非光源物体的光。如下图所示，在P点接收到Q点物体反射的光也需要考虑：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/7ZrfyRtGkIWmeou.png" alt="image-20230626184956410" style="zoom:50%;" />

在上面代码中加入间接光照分支：

```
shade(p, wo)
    Randomly choose N directions wi~pdf
    Lo = 0.0
    For each wi
        Trace a ray r(p, wi)
        If ray r hit the light
            Lo += (1 / N) * L_i * f_r * cosine / pdf(wi)
        **Else If ray r hit an object at q**
            **Lo += (1 / N) * shade(q, -wi) * f_r * cosine / pdf(wi)**
    Return Lo
```

不仅计算了直接从光源射向表面的光照效果，还计算了光线在环境中反射和散射之后产生的间接光照效果。

当一条从点p发出的光线r并没有直接打到光源，而是打到了另一个物体上的点q时，它会递归地调用shade()函数来计算在q点处，朝向-wi方向的光线的颜色或亮度。这实际上是在模拟光线从p点反射到q点，然后再从q点反射的过程。

这里的"-wi"是因为我们要考虑的是从q点反射出去的光线，它的方向应该是与从p点到q点的光线方向相反的。

但该方法至此有一个非常致命的问题：**指数爆炸**。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/cwmxBQVDjntg64s.png" alt="image-20230626191009334" style="zoom:50%;" />

如果在每次反射时都采样多个方向，那么光线的数量将会随着反射次数的增加而指数级增加，这会导致计算量变得非常大，这就是所谓的“指数爆炸”问题。

为了避免这个问题，一种可能的解决方案是每次反射时只采样一个方向，也就是设置N=1。这样，不论反射多少次，光线的数量都不会增加，从而避免了计算量的指数级增长。代码如下：

```
shade(p, wo)
    Randomly choose **ONE** direction wi~pdf(w)
    Trace a ray r(p, wi)
    If ray r hit the light
        Return L_i * f_r * cosine / pdf(wi)
    Else If ray r hit an object at q
        Return shade(q, -wi) * f_r * cosine / pdf(wi)
```

但是，这种方法也有一个缺点，那就是它可能无法准确地模拟所有可能的光线路径。因为每次只采样一个方向，所以有很多可能的光线路径将不会被考虑到，这可能会导致渲染效果的不准确。

为了减少这种不准确性，一种常见的做法是进行多次渲染，并且每次渲染都使用不同的随机采样方向，然后将多次渲染的结果取平均。这种方法被称为多次渲染或者蒙特卡洛渲染。

在上文我们说到，蒙特卡洛积分具有"根n收敛"性质，且蒙特卡洛积分是无偏估计，所以我们可以通过多次寻找路径就可以将光线的Radiance拟合出非常接近真实的结果。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/DbNjOMxlIGsQB3c.png" alt="image-20230626191828564" style="zoom:50%;" />

接下来改变代码思路也很直接，为了获得更精细的图像，每个像素可以进一步细分成多个样本。每个样本都代表了像素内的一个小区域，我们将从每个样本处发出一条光线。

**对于像素内的每个样本**，执行以下步骤：

1. **发射一条光线r(camPos, cam_to_sample)**：从相机的位置（camPos）向样本的位置发射一条光线。这条光线的方向（cam_to_sample）是从相机的位置指向样本的位置。
2. **如果光线r在场景的某一点p处碰到了**：如果光线r在场景中的某一点p处碰到了（也就是说，光线被场景中的某个物体遮挡住了），那么执行以下步骤：
   - **像素的亮度由shade()函数计算**，并与1/N相乘（N是像素内样本的数量）。shade()函数根据光线在p点的反射情况计算亮度。

```
ray_generation(camPos, pixel)
    Uniformly choose N sample positions within the pixel
    pixel_radiance = 0.0
    For each sample in the pixel
        Shoot a ray r(camPos, cam_to_sample)
        If ray r hit the scene at p
            pixel_radiance += 1 / N * **shade(p, sample_to_cam)**
    Return pixel_radiance
```

至此，我们解决了“指数爆炸”的问题。但是还有最后一个问题，我们的递归函数**没有出口**。解决递归函数出口的方式就是开枪把函数枪毙了（bushi），是使用"俄罗斯轮盘赌"（Russian Roulette）。

### "俄罗斯轮盘赌"（Russian Roulette）

"俄罗斯轮盘赌"（Russian Roulette）是一种解决这个问题的方法。这个名字来自于一种危险的赌博游戏，参与者在左轮手枪的某个弹仓装上子弹，转动弹仓，然后对着自己的头开枪，赌运气看子弹是否会击中自己。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/iu9YIvJqhHdGVFg.png" alt="image-20230626193811974" style="zoom:50%;" />

在Path Tracing中，我们需要计算出每个像素的颜色，这通常需要多次追踪光线的反射。同时又因为蒙特卡洛积分的方差收敛性，不断计算反射对最终的贡献可能会越来越小，因此我们可以使用"俄罗斯轮盘"中止光线的不断反射。

实现俄罗斯轮盘赌的一种方法是，对于每次反射，生成一个[0,1]间的随机数，如果这个数小于某个阈值（例如，光线的强度），那么就停止追踪。如果继续追踪，那么就要对光线的强度进行补偿，通常是除以继续追踪的概率。这样可以确保最终结果的无偏性，也就是说，即使使用了俄罗斯轮盘赌，最终结果的期望仍然等于没有使用时的结果。

"俄罗斯轮盘赌"的基本思想是对低概率事件的贡献进行削减，这样可以避免做一些计算效果不大却消耗资源的工作，从而降低计算量。

在概率统计中，方差（Variance）是描述随机变量或一组数据离散程度的度量。而在此处，我们所说的“方差”指的是光线强度的方差，也就是说，它描述的是光线强度的波动程度。如果我们对所有光线都进行完全的追踪，那么会产生很多光线强度极小、对最终结果贡献不大的路径，严重消耗性能，而且可能导致结果的不稳定。

这里所说的贡献程度一般会使用BRDF（双向反射分布函数）来计算。但是有时候，会直接使用光线的强度作为继续追踪的概率。这就要看具体的需求了。

来到具体实现环节。

先人工设定一个俄罗斯轮盘赌的“存活”率 P_RR 。

在 $[0, 1]$ 的均匀分布中随机选择一个值 ksi。

- 如果 ksi 大于 P_RR，那么就提前返回 0.0，结束这条路径的追踪。
- 如果 ksi 小于(等于) P_RR，那么就在概率密度函数 pdf(w) 下随机选择一个方向 wi，发射光线 r(p, wi)

如果光线 r(p, w_i) 打到了光源，那么返回 L_i * f_r * cosine / pdf(w_i) / P_RR 。其中 $L_i$ 是光源的辐射度，$f_r$ 是 BRDF 函数，cosine 是光线 wi 和物体表面法线的夹角余弦，pdf(w_i) 是 wi 这个方向的概率密度。结果除以 P_RR 是为了补偿俄罗斯轮盘赌的影响，保持结果的无偏性。

如果光线 r(p, w_i) 打到了另一个物体上的点 q，那么就递归调用 shade(q, -w_i) 来计算在 q 点的Radiance，然后乘以 f_r * cosine / pdf(w_i) / P_RR 并返回。

```
shade(p, w_o)
    Manually specify a probability P_RR
    Randomly select ksi in a uniform dist. in [0, 1] If (ksi > P_RR) return 0.0;
    Randomly choose ONE direction wi~pdf(w)
    Trace a ray r(p, w_i)
    If ray r hit the light
        Return L_i * f_r * cosine / pdf(w_i) / P_RR 
    Else If ray r hit an object at q
        Return shade(q, -w_i) * f_r * cosine / pdf(w_i) / P_RR
```

注意：俄罗斯轮盘赌在这里的主要作用是进行方差减小和减少无效计算，通过引入一个固定的概率来控制是否提前结束路径的追踪。而并没有用于计算或考虑路径的贡献。

如果希望根据路径的贡献来动态调整继续追踪的概率，那么需要引入重要性采样（Importance Sampling）。重要性采样就是根据函数值的重要性（比如在这里就是路径的贡献）来调整采样的概率，从而使采样更加高效。

这种方法通常来说和俄罗斯轮盘赌一起使用，以进一步提高渲染的效率。

### 直接光照采样优化

回顾我们一开始介绍渲染方程（The Rendering Equation）的 eq.(2) 简化方程：

$$
I = g \epsilon+g M g \epsilon+g M g M g \epsilon+g(M g)^3 \epsilon \cdot
\tag{2}
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/nu6zHfb4SxkaAsN.png" alt="image-20230626200705945" style="zoom:50%;" />

我们知道第一项计算的是直接光照，不需要计算其他的光线。那么在这个过程中就有大量的光线被“浪费”了。

于是我们可以直接对光源进行随机采样(如果中间没有别的物体)，而不是对整个半球面随机采样。

至于如何对发光面直接采样，就是一个纯数学的问题了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/qkbgN8r4Oza5eY2-20230627155213975.png" alt="image-20230626201404796" style="zoom:50%;" />

如上图所示，假设光源的面积为A，那么对光源进行采样的 pdf = 1/A ，因为：
$$
\int \operatorname{pdf} d A=1
$$
但是我们需要明确的是在渲染方程中积分的方向是从 $p$ 点往任意 $\omega_i$ 方向积分的。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/KkqizIwm9nvUl4Q-20230627155218290.png" alt="image-20230625204500403" style="zoom:50%;" />

所以，如果想要对光源进行随机采样的并依然使用蒙题卡洛积分，我们就需要找到 $d\omega_i$ 与 $dA$ 的关系，也就是说将渲染方程的积分变量从原来的 $d\omega_i$ 改为 $dA$ 。

立体角的定义就是单位球上投影的面积。

$$
d \omega=\frac{d A \cos \theta^{\prime}}{\left\|x^{\prime}-x\right\|^2}
\tag{4}
$$

然后将 eq.(4) 代入到 eq.(1) 渲染方程中，得到 eq.(5) ：

$$
\begin{aligned}
L_o\left(x, \omega_o\right) & =\int_{\Omega^{+}} L_i\left(x, \omega_i\right) f_r\left(x, \omega_i, \omega_o\right) \cos \theta \mathrm{d} \omega_i \\
& =\int_A L_i\left(x, \omega_i\right) f_r\left(x, \omega_i, \omega_o\right) \frac{\cos \theta \cos \theta^{\prime}}{\left\|x^{\prime}-x\right\|^2} \mathrm{~d} A
\end{aligned}
\tag{5}
$$

直接光照采样优化的原理就已经介绍完毕了，我们直接使用 eq.(5) 改写路径追踪（Monte Carlo Path Tracing）的伪代码：

```python
def shade(p, wo):
    # 计算来自光源的贡献
    # 在光源上均匀采样
    x_prime = uniformly_sample_light()
    pdf_light = 1 / Area_of_light
    # 计算直接照明
    L_dir = L_i * BRDF * cos(theta) * cos(theta_prime) / (distance_to_light^2) / pdf_light
    # 计算来自其他反射物的贡献
    L_indir = 0.0
    # 测试俄罗斯轮盘赌
    if Russian_Roulette_test(P_RR):
        # 在半球中均匀采样
        wi = uniformly_sample_hemisphere()
        pdf_hemi = 1 / (2 * pi)
        # 追踪一条光线
        r = trace_ray(p, wi)
        # 如果光线击中一个非发光物体
        if r.hits_non_light_object():
            q = r.hit_point()
            L_indir = shade(q, -wi) * BRDF * cos(theta) / pdf_hemi / P_RR
    # 返回总辐射度
    return L_dir + L_indir
```

别忘了最后一个细节，由于计算直接光照是直接找到发光面的，而在发光面和shading point中有可能有物体的遮挡。判断是否有物体遮挡，该做法也很简单，只需从shading point x 向光源 x’ 发出一条检测光线判断是否有物体与之相交即可。伪代码如下：

```
Shoot a ray from p to x’
If the ray is not blocked in the middle
```

## Whitted-style VS. Path Tracing

**Whitted-style光线追踪（Whitted-style Ray Tracing）：**

- 是最早的光线追踪算法。
- 它主要处理镜面反射和折射，并简化或忽略其他全局光照效应，例如漫反射间接照明。
- 为了获得间接照明的效果，WS光追会使用环境光（ambient light）来进行近似。
- 对物体进行硬阴影处理，不考虑如软阴影、散射等效果。

**路径追踪（Path Tracing）：**

- 路径追踪是一种基于蒙特卡洛方法的全局光照渲染算法，其能处理各种光线交互，包括漫反射和镜面反射、折射、散射等。
- 路径追踪算法通过模拟光线在场景中的多次反射（甚至无数次）来模拟全局照明效果，能够处理复杂的光照场景，比如颜色混合、柔和阴影、深度模糊等。
- 路径追踪计算成本较高，尤其在处理一些特定的光线交互（如折射、漫反射）时，可能需要很高的采样率才能避免噪声。

## 一些前沿的领域

- **Unidirectional Path Tracing（单向路径追踪）**：基于蒙特卡洛的全局光照算法，它能够处理各种光线交互，包括漫反射、镜面反射、折射、散射等。

- **Bidirectional Path Tracing（双向路径追踪）**：对单向路径追踪的扩展。特别擅长处理小光源、透射材质等。

- **Photon mapping（两步渲染技术）**：首先，它从光源发射光子，光子在场景中进行若干次碰撞后存储到光子图（Photon Map）中，然后再使用光子图进行光照计算。由于它分离了直接照明和间接照明的计算，使得它对场景的变化具有良好的抗干扰性。

- **Metropolis light transport**：Metropolis光传输是一种基于Metropolis-Hastings算法的全局光照算法。核心思想是使用一个Markov链（Markov chain）在路径空间进行的、采样，通过接受/拒绝准则（acceptance/rejection criterion）选择新的路径。优势在于它可以聚焦于对图像贡献大的重要区域。

- **VCM / UPBP**：VCM（Variance Clamping Metropolis Light Transport）一种改进的MLT方法，它通过使用方差截断（variance clamping）来控制噪声，同时保持了MLT的优点。UPBP（Unified Points, Beams, and Paths）是一种尝试统一光线、光束和光点（points, beams, and paths）的方法，它可以处理从几何体到体积、从点光源到面光源的各种光传输模式。

## Reference

1. Fundamentals of Computer Graphics 4th
2. GAMES101 Lingqi Yan
3. https://zhuanlan.zhihu.com/p/450731138
4. https://blog.csdn.net/qq_38065509/article/details/106496354
5. https://zhuanlan.zhihu.com/p/146144853
6. https://en.wikipedia.org/wiki/Inverse_transform_sampling
7. https://people.eecs.berkeley.edu/~jordan/courses/260-spring10/lectures/lecture17.pdf
