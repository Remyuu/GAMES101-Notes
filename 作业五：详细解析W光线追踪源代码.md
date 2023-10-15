本文内容：

1. 回顾渲染流程
2. 反射定理
3. 斯涅尔定律（Snell’s Law）
4. 菲涅耳反射系数（Fresnel reflectance）
5. 自阴影问题（自我交叉）

<!--more-->

作业框架在这里下载：[GAMES101_Homework5_S2021.zip][1]

## 回顾渲染流程



<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/Tt2GVhbpYKNeBny.png" alt="image-20230618140909956" style="zoom: 67%;" />

以下是详细流程：

### 1. **模型转换(Model Transformation)**

这是第一步，每个对象的顶点（通常定义在对象的局部坐标空间）被转换到世界坐标空间。

### 2. **顶点着色（Vertex Shading）**

渲染器的目的是为每一个像素分配颜色。而且必须是从某个特定的3D视角用视锥体（Frustum）向场景捕捉的。在光线追踪中，每个像素都通过生成一条光线来创建图像。当一条光线与场景中的一个对象相交时，我们将像素的颜色设置为交点处对象的颜色。这个过程被称为反向或眼睛追踪，因为我们跟踪的是光线从相机到物体，然后从物体到光源的路径。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/iy2pSAfU8CNeXY7.gif" alt="campixel"  />

自然而然的，建立图像的过程就是从其中的一根根射线开始。

当务之急是为帧的每个像素创建一个主射线。我们通过从相机的原点开始向每个像素的中间对其发射像素即可（如上图所示）。

我们用射线的形式表达这条线，其原点是相机的原点，其方向是从相机原点到像素中心的矢量。

### 3. 坐标转换（模型空间转换到齐次裁剪空间）

要计算像素中心点的位置，我们需要将最初以**光栅空间**表示的像素坐标（点坐标以像素表示，坐标（0,0）为帧的左上角）转换为**世界空间**。世界空间是场景中所有对象、几何体、光源和相机的坐标都被表示的空间。

注意，这里的y轴是向下的，所以在写代码时要将y反转。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/DrGwCuYyem7lKJn.png" alt="cambasic1A" style="zoom: 67%;" />

如上图所示，如果你有一个宽度为`w`，高度为`h`的图像，你会为每个像素（`i`, `j`）创建一个射线。其中`i`在`[0, w)`范围，`j`在`[0, h)`范围。射线的原点是相机的位置，方向向量则可以通过将像素的坐标映射到相机的视锥体（Frustum）上来获得。

想要完成上述步骤一般的方法是：将像素的坐标映射到`[-1, 1]`区间（称为归一化设备坐标，NDC），然后再映射到相机的视锥体上。

```c++
x = (2.0 * i) / w - 1.0
y = 1.0 - (2.0 * j) / h  // 注意这里我们把 y 轴反向了，因为在图像中，y 轴是向下的
z = -1.0  // 通常我们假设相机视锥体的近平面位于 z = -1 处
```

注意这里假设相机位于原点，朝向`z`轴负方向。

接下来的工作就是将不在视锥体范围内的物品删除，然后使用光线-物体交叉测试来找到射线首次命中的物体，然后计算在交点处的光照和颜色，这就是在像素`(i, j)`处要渲染的颜色。如果射线没有命中任何物体，那么通常会渲染一个背景颜色（如天空盒子）。

视图变换后的物体需要被进一步变换到齐次裁剪空间（Homogeneous Clip Space）的时候，有两个操作需要执行：

1. 视场是我们看到的场景的范围，它可以通过角度α来定义。我们可以将屏幕像素坐标乘以该角度的正切除以二的结果来考虑视场。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/1s5yoWUX9ChIL4r.png" alt="camratio" style="zoom: 67%;" />

2. 投影方式（例如正射投影或透视投影）

### 4. 裁剪（Clipping）

一个图元和摄像机视野的关系有3种：完全在视野内、部分在视野内、完全在视野外。根据图元与视野的关系，处理的方式也会有所不同：

1. **完全在视野内：**这类图元完全位于摄像机视野内，无需进行裁剪操作，可以直接进行后续的渲染步骤。
2. **部分在视野内：**这类图元只有一部分位于摄像机视野内，需要进行裁剪操作。裁剪过程会将位于视野外的部分移除，仅保留位于视野内的部分，然后再进行后续的渲染步骤。
3. **完全在视野外：**这类图元完全位于摄像机视野外，不会出现在最终的渲染结果中。这些图元可以直接被剔除，无需进行后续的渲染步骤，这样可以大大提高渲染效率。

因为我们上一步已经将场景转换为了NDC空间一般是 [-1, 1] 范围的立方体，于是我们可以使用一个固定的裁剪盒（通常就是 [-1, 1] 的范围）来进行裁剪。

那些完全在裁剪盒外的元素会被完全剔除，而那些部分在裁剪盒内的元素则会被裁剪，只保留裁剪盒内的部分。然后，这些裁剪后的元素会被映射（或者说是缩放和平移）到屏幕空间，以便于进行最后的渲染。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/DEMoUmbI14CH3YV.png" alt="image-20230618142733351" style="zoom:50%;" />



4. ### **屏幕映射（Screen Mapping）**

在这个阶段，我们需要将3D场景中的物体从归一化设备坐标（Normalized Device Coordinates，NDC）映射到屏幕坐标系（Screen Coordinates）。

如果你的屏幕分辨率是1920x1080，那么屏幕坐标系的范围就是从(0, 0)到(1920, 1080)。

在屏幕映射阶段，我们首先要做的是将NDC中的x和y坐标映射到屏幕坐标系。这可以通过以下公式完成：

```c++
x_screen = ((x_ndc + 1) / 2) * width
y_screen = ((1 - y_ndc) / 2) * height
```

> 重复强调一下：
>
> 注意我们在计算y坐标时使用的是`1 - y_ndc`，这是因为在NDC中，y轴是向上的，而在屏幕坐标系中，y轴是向下的。

对于z坐标，通常我们会保留它在NDC中的值，并且将其传递到光栅化阶段。在光栅化阶段，这个z值通常被用来进行深度测试，以决定哪个物体应该被绘制到屏幕上。这是因为在一个复杂的3D场景中，可能会有多个物体位于同一个像素位置，我们需要某种方式来决定哪个物体是最前面的，也就是说，哪个物体应该被摄像机看到。深度测试就是解决这个问题的一种方法。

接下来，就是光栅化渲染了。

### 总结

3D渲染流程经历的几个空间坐标：

1. **World Space**：在世界空间，场景中的所有对象都被定位并以三维坐标表示。世界空间中的坐标与物体在真实世界中的位置对应。
2. **Camera Space / View Space**：世界空间中的对象被转换到相机空间，也称为视图空间。在这个空间中，相机位于原点，相机的方向定义了空间的方向。
3. **Projection Space / Clip Space**：相机空间中的对象被投影到投影空间，也称为裁剪空间。这里，透视投影或正交投影变换被应用，以考虑视野的大小和形状。
4. **Normalized Device Coordinates (NDC) Space**：在投影空间中，坐标被除以它们的齐次坐标w来执行透视除法，产生归一化设备坐标。这会将所有的坐标转换到一个[-1, 1]的立方体内。在一些旧版本的DirectX中，NDC的z坐标的范围是[0,1]，而在OpenGL和新版本的DirectX中，z坐标的范围通常是[-1, 1]。
5. **Screen Space**：NDC空间中的对象被转换到屏幕空间，其中每个坐标都对应屏幕上的一个像素位置。
6. **Rasterization**：在光栅化阶段，屏幕空间中的对象被转换为像素图像，可以直接显示在屏幕上。



## 光追流程

光线追踪算法的主要运行流程大致如下：

1. **初始化：**首先，场景中的所有物体、相机和光源等元素会被初始化。物体会被赋予形状、大小、位置、材质等属性。相机则会被赋予位置、朝向、视野等属性。光源会被赋予位置、强度、颜色等属性。
2. **迭代：**在程序的主循环中，算法会遍历每个像素。对于每个像素，都会从相机发出一条光线。
3. **光线追踪：**每条光线会被追踪到它碰到的第一个物体。这需要检查光线与场景中所有物体的交点，并找出最近的交点。这部分由`trace`函数完成。
4. **颜色计算：**一旦找到光线和物体的交点，就需要计算在这个交点处光线的颜色。这部分由`castRay`函数完成。它会考虑物体的材质、光线的方向、光源的位置和颜色等因素。如果物体是透明的或者反射的，那么可能需要发出新的光线并追踪它们。
5. **写入像素：**计算得到的颜色会被写入当前像素。所有像素的颜色加起来就形成了最终的图片。
6. **输出：**最后，所有的像素值会被写入到一个图片文件中，这就是最终的渲染结果。

## Renderer.cpp流程详解

在光线追踪中，我们模拟光线从相机（或眼睛）发出，穿过每个像素，到达3D场景中的物体，然后反射或折射，直到光线碰到光源或超过了最大追踪深度。

### 1. `reflect`: 计算反射光线的方向

这个函数需要两个参数：

- 入射光方向 `I` 
- 是表面法向量 `N`

#### 反射定律

- 光线的反射角等于入射角，入射方向、表面法线和反射方向为共面。

反射向量公式：
$$
R = I - 2 \cdot dot(I, N) \cdot N
$$
其中，法线向量$n$一定要是单位向量。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/7O1xXCNTFn9Dro5.jpg" alt="bCilW" style="zoom: 67%;" />

```c++
// Compute reflection direction
Vector3f reflect(const Vector3f &I, const Vector3f &N){
    return I - 2 * dotProduct(I, N) * N;
}
```

>参考链接：https://math.stackexchange.com/questions/13261/how-to-get-a-reflection-vector

### 2. `refract`: 利用斯涅尔定律计算折射光线的方向

在我的[上一篇文章](https://remoooo.com/cg/9-1.html)中，我们讨论了使用递归光线追踪来计算镜面反射。另一种镜面物体是电介质——一种能折射光线的透明材料。钻石、玻璃、水和空气都是电介质。电介质也过滤光；有些玻璃滤掉的红光和蓝光比绿光多，所以玻璃呈现出绿色。

#### 斯涅尔定律（Snell's Law）

当一条光线从折射率为$n$的介质传播到折射率为$n_t$的介质中时，一些光线被透射，并且发生弯曲。如下图所示，在介质中光的反射率 $n_t > n$ 。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/fBsxEyRLm8loiqT.png" alt="image-20230615135352245" style="zoom:50%;" />

斯涅尔定律（Snell's Law）正是描述的是光从一种介质进入另一种介质时，它的折射现象。更具体地说，它给出了入射光线和折射光线相对于介质交界面法线的角度之间的关系。数学表达式如下：
$$
n \sin \theta=n_t \sin \phi
$$
但是利用正弦函数计算两个向量的夹角并不方便，在计算机图形学中一般使用余弦函数。而且两个向量的点积就是一个余弦值，所以我们将上面的表达式通过 $\sin ^2 \theta+\cos ^2 \theta=1$ 转换为用余弦函数来表示：
$$
\cos ^2 \phi=1-\frac{n^2\left(1-\cos ^2 \theta\right)}{n_t^2}
$$
我们来看第一次反射时发生了什么。如下图所示，入射光 $d$ 射入基底为 $n , b$ 构成的坐标系中。反射光是 $r$ ，折射光是 $t$ 。且已知平面法线 $n$ 与反射光 $r$ 的夹角 $\theta$ 和折射光 $t$ 与$-n$ 的夹角 $\phi$ 。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/fKEmUZSXg9iRQCV.png" alt="image-20230615143208005" style="zoom:50%;" />

我们可以用折射角与基底来表示折射光 $t$ ：
$$
\mathbf{t}=\sin \phi \mathbf{b}-\cos \phi \mathbf{n}
\tag{1}
$$
也可以用 $\theta$ 和基底来表示入射光 $d$ ：
$$
\mathbf{d}=\sin \theta \mathbf{b}-\cos \theta \mathbf{n}
\tag{2}
$$
由**公式(1)(2)**可以解出 $b$ ：
$$
\mathbf{b}=\frac{\mathbf{d}+\mathbf{n} \cos \theta}{\sin \theta}
\tag{3}
$$
也就是说，我们可以不需要基底 $b$ 就可以表示折射角、入射角以及反射角。因为一般来说，我们只会得到物体的法向量。

那么将 $t$ 可以写作：
$$
\begin{aligned}
\mathbf{t} & =\frac{n(\mathbf{d}+\mathbf{n} \cos \theta))}{n_t}-\mathbf{n} \cos \phi \\
& =\frac{n(\mathbf{d}-\mathbf{n}(\mathbf{d} \cdot \mathbf{n}))}{n_t}-\mathbf{n} \sqrt{1-\frac{n^2\left(1-(\mathbf{d} \cdot \mathbf{n})^2\right)}{n_t^2}} .
\end{aligned}
\tag{4}
$$
需要注意的问题是：“如果平方根下是负数怎么办？”

这个情况下，代表没有折射光线，所有光线能量都被反射掉了。这就是平时所说的**「光线全反射」**。

也就是说，当光从折射率较高的介质进入折射率较低的介质时，如果入射角超过了一个特定的角度（临界角），光将完全反射回较高折射率的介质。这就是为什么玻璃物体外观比较复杂的原因。

----

**回到代码**，这个函数实现了折射光线方向的计算。传入如下参数：

- `I`：入射光线的方向。
- `N`：表面的法线方向。
- `ior`：折射率 (index of refraction)，通常为目标材料（如玻璃或水）的折射率。

我们一行行看。

```c++
float cosi = clamp(-1, 1, dotProduct(I, N));
```

`cosi` 是入射光线向量和表面法线向量之间的夹角的余弦值。`clamp` 函数确保 `cosi` 的值在 -1 和 1 之间。`dotProduct` 函数计算两个向量的点积，如果两个向量都是单位向量，那么它就等于它们之间的夹角的余弦值。

```c++
float etai = 1, etat = ior;
```

这一行代码设定了两个折射率的数值。`etai` 是入射介质的折射率，这里设为 1（即假设入射介质为真空或空气），`etat` 是折射介质的折射率，值为函数参数 `ior`。

```c++
Vector3f n = N;
```

创建一个新的向量 `n`，初值为表面法线向量 `N`。

```c++
if (cosi < 0) { cosi = -cosi; } else { std::swap(etai, etat); n= -N; }
```

如果 `cosi` 小于0，说明入射光线向量和表面法线向量的夹角超过90度，也就是说，此时入射光是在物体的内部。因为如果光线在物体内部，这意味着光线的方向与法线的方向在同一侧，此时 `cosi` 为正。而我们的法线是向物体的外部为正方向的，所以我们把法线方向取反。这一步的目的是将光线的入射角标准化到 [0, pi/2] 的范围内，也就是确保我们总是以光线与物体表面的交点处的法线方向为参考，来计算光线的折射和反射。

但是我个人觉得这样写不太清晰。我们初始化的变量本来就是按照下面图片的规则编写的。只有当光从介质内射出，才需要调整入射角度等信息。并且在后面的 $t$ 计算中，也应该更换为本文的**公式(4)**，否则会让人一头雾水。也就是说，将上面的代码改为：**（如果修改了这行代码，最后的三目运算符处也要对应修改，详见下文）**

```c++
if (cosi >= 0) { cosi = -cosi; std::swap(etai, etat); n= -N; }
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/fKEmUZSXg9iRQCV-20230719182208851.png" alt="image-20230615143208005" style="zoom:50%;" />

如果`cosi`大于0，那么说明光线是从物体外部射入。此时交换两个介质的折射率，并且将 `n` 取为 `N` 的反向。

```c++
float eta = etai / etat;
```

计算相对折射率，即入射介质的折射率除以折射介质的折射率。

```c++
float k = 1 - eta * eta * (1 - cosi * cosi);
```

计算用于检查全反射的项。源于**斯涅尔定律（Snell's Law）的扩展版**，用于处理全反射情况。刚才我们讲到斯涅尔定律的基础公式：
$$
n \sin \theta=n_t \sin \phi
$$
我们将右边的$\sin \phi$算出：
$$
\sin \phi=\frac{n}{n_t} \sin \theta
$$
当光从高折射率介质（$n_t$）入射到低折射率介质（$n$）时，当入射角（$\theta $）足够大（入射光与法线的夹角），就会发生全反射，没有光线折射进入第二种介质。

为了确定是否发生了全反射，我们可以先计算一个量，称为“全反射临界值”（正是上面的**公式(4)**中根号下的内容）：
$$
k = 1 - (\frac{n}{n_t})^2 \cdot (1 - cos(\theta)^2)
$$
其中，这个公式的右边是通过下面计算的:
$$
(\frac{n}{n_t})^2 \cdot (1 - cos(\theta)^2)
=
(\sin \phi)^2
$$
也就是说，如果公式的 $k<0$ ，那么 $(\sin \phi)^2$ 的值就大于 1，我们认为发生了全反射。

如果这个值小于0，说明发生了全反射。

值得一提的是，如果 `k=1` ，说明`(1 - cosi * cosi)` 必须为0，那么 `cosi` 必定为1或者-1，也就是说入射光线向量和表面法线向量平行或反平行。当 `k` 值为1时，入射光线是沿着法线方向入射或者完全反向。但在实际的物理现象中，`k` 值很少正好为1，因为光线通常不会完全沿着法线方向入射或反射。

```c++
return k < 0 ? 0 : eta * I + (eta * cosi - sqrtf(k)) * n;
```

如果 `k` 小于0，即发生了全反射，返回0；否则，根据折射光线的计算公式计算折射光线的方向并返回。

当 `k < 0` 时，表示发生了全反射，没有折射光线。在理想情况下，这应该返回一个特殊的值，或者抛出一个异常来表示全反射。然而在这个作业框架中，当 `k < 0` 时，选择了返回 0。这是一个设计上的简化，是为了在后续的代码中更方便地检测到全反射的情况（如果返回的向量是0，可能意味着在接下来的代码中要处理全反射的特殊情况）。具体操作在后面的 `castRay` 函数中详细讲解。

这里的表达式本质就是**公式(4)**：
$$
\mathbf{t}  
=
\frac{n(\mathbf{d}-\mathbf{n}(\mathbf{d} \cdot \mathbf{n}))}{n_t}-\mathbf{n} \sqrt{1-\frac{n^2\left(1-(\mathbf{d} \cdot \mathbf{n})^2\right)}{n_t^2}} .
\tag{4}
$$
化简可得：
$$
\mathbf{t}  
=
eta \cdot I - n \cdot (eta \cdot cosi) - n \sqrt{k} \\
其中, eta = \frac{n}{n_t} , I=d , cosi = I \cdot n, k =1-\frac{n^2\left(1-(\mathbf{d} \cdot \mathbf{n})^2\right)}{n_t^2} .
\tag{5}
$$
此处我们应该严格按照公式输入，代码则为：

```c++
return k < 0 ? 0 : eta * I - (eta * cosi + sqrtf(k)) * n;
```

这行代码应该要和上文内容匹配。也就是说，正确的代码有两个版本，他们完全一致，你的代码大概会长这样：

```c++
...
if (cosi < 0) {cosi = -cosi;} else { cosi = -cosi; std::swap(etai, etat); n= -N; }
...
return k < 0 ? 0 : eta * I + (eta * cosi - sqrtf(k)) * n;
...
```

或者是，

```c++
...
if (cosi >= 0) { cosi = -cosi; std::swap(etai, etat); n= -N; }
...
return k < 0 ? 0 : eta * I - (eta * cosi + sqrtf(k)) * n;
...
```

其中， 

- `eta` 是两个介质的相对折射率，是入射介质的折射率除以透射介质的折射率，也就是 `etai / etat`。
- `I` 是入射光线的方向向量。
- `cosi` 是入射光线与法线的夹角的余弦值。
- `n` 是单位法线向量。
- `sqrtf(k)` 是折射光线与法线的夹角的余弦值，其中 `k = 1 - eta * eta * (1 - cosi * cosi)`。如果 `k` 小于0，说明发生了全反射，没有折射光线。

完整代码如下：

```c++
Vector3f refract(const Vector3f &I, const Vector3f &N, const float &ior)
{
    float cosi = clamp(-1, 1, dotProduct(I, N));
    float etai = 1, etat = ior;
    Vector3f n = N;
    if (cosi < 0) { cosi = -cosi; } else { std::swap(etai, etat); n= -N; }
    float eta = etai / etat;
    float k = 1 - eta * eta * (1 - cosi * cosi);
    return k < 0 ? 0 : eta * I + (eta * cosi - sqrtf(k)) * n;
}
```

又或者是这样的：

```c++
Vector3f refract(const Vector3f &I, const Vector3f &N, const float &ior)
{
    float cosi = clamp(-1, 1, dotProduct(I, N));
    float etai = 1, etat = ior;
    Vector3f n = N;
	if (cosi >= 0) { cosi = -cosi; std::swap(etai, etat); n= -N; }
    float eta = etai / etat;
    float k = 1 - eta * eta * (1 - cosi * cosi);
    return k < 0 ? 0 : eta * I - (eta * cosi + sqrtf(k)) * n;
}
```



### 3. `fresnel`: 计算菲涅尔方程

这段代码是计算菲涅耳方程的函数，用于**确定光线在介质边界处的反射和折射的比例**。先给出代码：

```c++
float fresnel(const Vector3f &I, const Vector3f &N, const float &ior)
{
    float cosi = clamp(-1, 1, dotProduct(I, N));
    float etai = 1, etat = ior;
    if (cosi > 0) {  std::swap(etai, etat); }
    // Compute sini using Snell's law
    float sint = etai / etat * sqrtf(std::max(0.f, 1 - cosi * cosi));
    // Total internal reflection
    if (sint >= 1) {
        return 1;
    }
    else {
        float cost = sqrtf(std::max(0.f, 1 - sint * sint));
        cosi = fabsf(cosi);
        float Rs = ((etat * cosi) - (etai * cost)) / ((etat * cosi) + (etai * cost));
        float Rp = ((etai * cosi) - (etat * cost)) / ((etai * cosi) + (etat * cost));
        return (Rs * Rs + Rp * Rp) / 2;
    }
    // As a consequence of the conservation of energy, transmittance is given by:
    // kt = 1 - kr;
}
```

这个函数传入的参数：

- `I`：入射光线的方向。
- `N`：表面的法线方向。
- `ior`：折射率 (index of refraction)，通常为目标材料（如玻璃或水）的折射率。

它的输出是一个介于0和1之间的数值，表示反射光线的强度占总光线（反射光线和折射光线之和）强度的比率。这个数值可以用来计算表面的反射颜色和透明度。接下来，我们一行一行分析。

----

前面两行与上part一样，略过。

```c++
if (cosi > 0) { std::swap(etai, etat); }
```

如果`cosi`大于0，那么光是从`ior`介质入射到空气中，此时需要交换两个介质的折射率。否则什么都不需要动。

```c++
float sint = etai / etat * sqrtf(std::max(0.f, 1 - cosi * cosi));
```

使用斯涅尔定律计算折射角的正弦值。原理是斯涅尔定律（Snell's Law）。数学表达式如下：
$$
n \sin \theta=n_t \sin \phi
$$
其中，代码给定的是入射角的余弦值 cosi，使用三角关系转换 $sin^2(\theta) + cos^2(\theta) = 1 $ 。用 `max` 函数确保根号下没有因为浮点数误差导致的负数。

```c++
if (sint >= 1) { return 1; }
```

如果`sint`大于等于1，表示发生全反射，反射系数为1。

否则，使用菲涅耳公式计算s-极化反射系数`Rs`和p-极化反射系数`Rp`，并返回它们的平均值。该公式考虑了光的极化方向。

```c++
float cost = sqrtf(std::max(0.f, 1 - sint * sint));
cosi = fabsf(cosi);
float Rs = ((etat * cosi) - (etai * cost)) / ((etat * cosi) + (etai * cost));
float Rp = ((etai * cosi) - (etat * cost)) / ((etai * cosi) + (etat * cost));
return (Rs * Rs + Rp * Rp) / 2;
```

菲涅耳反射系数（Fresnel  reflectance）描述了一个光波在两种不同介质的边界上发生的反射现象。具体来说，它定义了从一个介质入射到另一个介质时，反射光的强度和入射光的强度的比例。它是由法国物理学家奥古斯丁·菲涅耳在19世纪初提出的，是光学和电磁学中的基本概念。

光可以被分为两种偏振状态：s-偏振（振动方向垂直于入射平面）和p-偏振（振动方向平行于入射平面）。这两种偏振状态的光的反射率通常是不同的。

菲涅耳反射系数被定义为这两个反射系数的平均值。

#### 光强方程

定义入射光线、反射光线和折射光线各自与法线形成的夹角分别为 $\theta_i , \theta_r$ 和 $\theta_t $ 。

同时，入射光线与反射光线的方向由反射定律约束：
$$
\theta_i = \theta_r
$$
入射光线与折射光线的方向由斯涅尔定律约束：
$$
n_1 \sin \theta_i =n_t \sin \theta_t
$$
一定功率的入射光被界面反射的比例称为反射比 $R$ ；折射的比例称为透射比 $T$ 。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/WG8FpRJBm31Hct4.png" alt="image-20230618223822181" style="zoom:50%;" />

反射比和透射比的具体形式还与入射光的偏振有关。如果入射光的电矢量垂直于右图所在平面（即s偏振），反射比为：
$$
R_s=\left[\frac{\sin \left(\theta_t-\theta_i\right)}{\sin \left(\theta_t+\theta_i\right)}\right]^2=\left(\frac{n_1 \cos \theta_i-n_2 \cos \theta_t}{n_1 \cos \theta_i+n_2 \cos \theta_t}\right)^2=\left[\frac{n_1 \cos \theta_i-n_2 \sqrt{1-\left(\frac{n_1}{n_2} \sin \theta_i\right)^2}}{n_1 \cos \theta_i+n_2 \sqrt{1-\left(\frac{n_1}{n_2} \sin \theta_i\right)^2}}\right]^2
$$
其中， $\theta_t$ 是由斯涅尔定律从 $\theta_i$  导出。

如果入射光的电矢量位于右图所在平面内（即p偏振），反射比为：
$$
R_p=\left[\frac{\tan \left(\theta_t-\theta_i\right)}{\tan \left(\theta_t+\theta_i\right)}\right]^2=\left(\frac{n_1 \cos \theta_t-n_2 \cos \theta_i}{n_1 \cos \theta_t+n_2 \cos \theta_i}\right)^2=\left[\frac{n_1 \sqrt{1-\left(\frac{n_1}{n_2} \sin \theta_i\right)^2}-n_2 \cos \theta_i}{n_1 \sqrt{1-\left(\frac{n_1}{n_2} \sin \theta_i\right)^2}+n_2 \cos \theta_i}\right]^2
$$
透射比无论在哪种情况下，都有 $ T = 1- R $。

如果入射光是无偏振的（含有等量的s偏振和p偏振），反射比是两者的算数平均值：
$$
R=\frac{R_s+R_p}{2} .
\tag{6}
$$
于是这就有了上面的代码。



### 4. `trace`: 遍历所有场景中的物体，看看光线是否与其相交

- **功能**: 判断光线是否与物体相交。
- **参数**:
  - `orig`: 光线的起源
  - `dir`: 光线的方向
  - `objects`: 场景中包含的物体列表
- **输出参数**:
  - `tNear`: 存储到最近交点物体的距离
  - `index`: 如果交点物体是一个网格，存储相交三角形的索引
  - `uv`: 存储交点的u和v重心坐标
  - `*hitObject`: 存储指向相交物体的指针（用于检索材料信息等）。如果光线没有与物体相交，则输出空指针。

```c++
std::optional<hit_payload> trace(
        const Vector3f &orig, const Vector3f &dir,
        const std::vector<std::unique_ptr<Object> > &objects)
{
    float tNear = kInfinity;
    std::optional<hit_payload> payload;
    for (const auto & object : objects)
    {
        float tNearK = kInfinity;
        uint32_t indexK;
        Vector2f uvK;
        if (object->intersect(orig, dir, tNearK, indexK, uvK) && tNearK < tNear)
        {
            payload.emplace();
            payload->hit_obj = object.get();
            payload->tNear = tNearK;
            payload->index = indexK;
            payload->uv = uvK;
            tNear = tNearK;
        }
    }
    return payload;
}
```

其中，hit_payload的结构是：

```c++
struct hit_payload
{
    float tNear;
    uint32_t index;
    Vector2f uv;
    Object* hit_obj;
};
```

简单地说，先将 `tNear` 设置为正无穷大，存储目前找到的最近的物体与光线的交点距离。

同时再声明一个 `std::optional` 类型的变量 `payload` 。`std::optional` 是一个可以为空的容器，它要么包含一个 `hit_payload` 类型的值（也就是说，有一个有效的交点），要么什么都不包含（也就是说，没有找到有效的交点）。

接着函数开始遍历场景中的所有物体。对于每个物体，都会尝试用其`intersect`方法计算光线与该物体的交点。如果找到了交点，**且这个交点比当前已找到的最近交点还要近**，就会更新`payload`和`tNear`。

最后，函数返回 `payload`。如果找到了有效的交点，`payload` 会包含这个交点的信息；否则，`payload` 会是空的。

### 5. `castRay`: 根据光线与物体的交点和物体的材质类型，递归计算出最终像素的颜色

这是Whitted风格光传输算法的实现。

这个函数计算由**位置和方向**定义的**射线**在**交点处**的**颜色**。注意这个函数是递归的（它会调用自己）。

- 如果相交物体的材质是**反射或者既反射又折射**的，那么我们会计算反射/折射方向，并通过递归调用 `castRay()` 函数在场景中投射**两条新的射线**。

- 当表面是**透明**的，我们会使用菲涅耳方程的结果来**混合反射和折射颜色**（它根据表面法线、入射视线方向和表面折射指数来计算反射和折射的量）。

- 如果表面是**漫反射/光泽**的，我们会使用**Phong照明模型**来计算交点处的颜色。

代码比较长，我们首先粗略看看：

```c++
Vector3f castRay(const Vector3f &orig,
     const Vector3f &dir, const Scene& scene,int depth){
    if (depth > scene.maxDepth) {return Vector3f(0.0,0.0,0.0);}
    Vector3f hitColor = scene.backgroundColor;
    if (auto payload = trace(orig, dir, scene.get_objects()); payload){
		......//part_1
        switch (payload->hit_obj->materialType) {
            case REFLECTION_AND_REFRACTION:{
				......}
            case REFLECTION:{
				......}
            default:{
				......}
        }
    }
    return hitColor;
}
```

首先设置了递归深度限制。如果超过了，它会返回黑色（`Vector3f(0.0,0.0,0.0)`），因为我们不希望无限期地追踪射线。

接着，函数设置 `hitColor` 为场景的背景颜色。这是默认的颜色，如果射线没有打到任何物体，函数将返回此颜色。

然后，函数调用 `trace` 函数检查射线是否与场景中的任何物体相交。如果有交点，函数就会获取交点处的表面属性，包括表面法线（N）和贴图坐标（st）。

然后接下来switch语句有三种分支，分别对应上面三种材质类型。

接下来第一段注释，也就是 `part_1` 。

```c++
......
Vector3f hitPoint = orig + dir * payload->tNear;
Vector3f N; // normal
Vector2f st; // st coordinates
payload->hit_obj->getSurfaceProperties(hitPoint, dir, payload->index, payload->uv, N, st);
......
```

计算物体接触点表面的信息，存储在 `hitPoint` 中。

然后调用物体的 `getSurfaceProperties` 方法，根据交点的位置（`hitPoint`），光线的方向（`dir`），以及一些其他的参数（如 `payload->index` 和 `payload->uv`），计算出交点的法线（`N`）和贴图坐标（`st`）。

#### REFLECTION_AND_REFRACTION

光线会有一部分反射，一部分折射，然后计算出反射和折射部分的颜色，按照菲涅尔方程的结果对这两部分颜色进行混合。

```c++
Vector3f reflectionDirection = normalize(reflect(dir, N));
Vector3f refractionDirection = normalize(refract(dir, N, payload->hit_obj->ior));
Vector3f reflectionRayOrig = (dotProduct(reflectionDirection, N) < 0) ?
                             hitPoint - N * scene.epsilon :
                             hitPoint + N * scene.epsilon;
Vector3f refractionRayOrig = (dotProduct(refractionDirection, N) < 0) ?
                             hitPoint - N * scene.epsilon :
                             hitPoint + N * scene.epsilon;
Vector3f reflectionColor = castRay(reflectionRayOrig, reflectionDirection, scene, depth + 1);
Vector3f refractionColor = castRay(refractionRayOrig, refractionDirection, scene, depth + 1);
float kr = fresnel(dir, N, payload->hit_obj->ior);
hitColor = reflectionColor * kr + refractionColor * (1 - kr);
```

详细解析如下：

- `Vector3f reflectionDirection = normalize(reflect(dir, N));`：首先计算出光线在物体表面的**反射**方向，然后归一化。

- `Vector3f refractionDirection = normalize(refract(dir, N, payload->hit_obj->ior));`：然后计算出光线在物体表面的**折射**方向，然后归一化。

- `Vector3f reflectionRayOrig` 和 `Vector3f refractionRayOrig`：计算出反射光线和折射光线的起始位置。这里为了避免自我交叉的问题（即反射光线或者折射光线误打到原物体上），通常会在原始交点的基础上沿着法线方向偏移一点距离，这个偏移的距离通常设定为一个非常小的值，如这里的 `scene.epsilon`。关于自我交叉问题的详细阐述，请看本文的附录 1. 内容。

- `Vector3f reflectionColor = castRay(reflectionRayOrig, reflectionDirection, scene, depth + 1);`：使用反射光线的起始位置和方向，再次进行光线追踪，得到反射光线的颜色。

- `Vector3f refractionColor = castRay(refractionRayOrig, refractionDirection, scene, depth + 1);` ：一样地，使用折射光线的起始位置和方向，再次进行光线追踪，得到折射光线的颜色。

- `float kr = fresnel(dir, N, payload->hit_obj->ior);`：使用菲涅尔方程计算出反射和折射的比例。这个比例是根据入射光线的方向 `dir`，物体表面的法线 `N`，以及物体的折射率 `ior` 计算得出的。这个比例 `kr` 表示的是反射的比例，那么 `1 - kr` 就表示的是折射的比例。

- `hitColor = reflectionColor * kr + refractionColor *  (1 - kr);` ：最后根据反射和折射的比例，对反射光线的颜色和折射光线的颜色进行混合，得到最终的颜色 `hitColor`。

中间的分支类似，不重复阐述了。这里说最下面的分支。

#### Phong illumation & diffuse &specular reflection

处理的是物体材质为漫反射或光泽的情况，它使用了Phong光照模型来计算相交点的颜色。

其中的Lambert光照模型，公式如下：
$$
L_d = k_d(I/r^2)max(0, n·l)
$$
其中 $L_d$ 是着色点的颜色，$k_d$ 是物体表面的反射系数（包含定义材质的RGB颜色），$I$ 是光源的初始强度，$r$ 是光源到着色点的距离，$n$ 是着色点的表面法线，$l$ 是光源方向，$n·l$ 是$n$和$l$的点积（即$n$和$l$的夹角的余弦）。

```c++
// [comment]
// We use the Phong illumation model int the default case. The phong model
// is composed of a diffuse and a specular reflection component.
// [/comment]
Vector3f lightAmt = 0, specularColor = 0;
Vector3f shadowPointOrig = (dotProduct(dir, N) < 0) ?
                           hitPoint + N * scene.epsilon :
                           hitPoint - N * scene.epsilon;
// [comment]
// Loop over all lights in the scene and sum their contribution up
// We also apply the lambert cosine law
// [/comment]
for (auto& light : scene.get_lights()) {
    Vector3f lightDir = light->position - hitPoint;
    // square of the distance between hitPoint and the light
    float lightDistance2 = dotProduct(lightDir, lightDir);
    lightDir = normalize(lightDir);
    float LdotN = std::max(0.f, dotProduct(lightDir, N));
    // is the point in shadow, and is the nearest occluding object closer to the object than the light itself?
    auto shadow_res = trace(shadowPointOrig, lightDir, scene.get_objects());
    bool inShadow = shadow_res && (shadow_res->tNear * shadow_res->tNear < lightDistance2);

    lightAmt += inShadow ? 0 : light->intensity * LdotN;
    Vector3f reflectionDirection = reflect(-lightDir, N);

    specularColor += powf(std::max(0.f, -dotProduct(reflectionDirection, dir)),
        payload->hit_obj->specularExponent) * light->intensity;
}

hitColor = lightAmt * payload->hit_obj->evalDiffuseColor(st) * payload->hit_obj->Kd + specularColor * payload->hit_obj->Ks;
break;
```

- 对于每个光源，我们首先计算从相交点到光源的方向(`lightDir`)和平方距离(`lightDistance2`)。随后将 `lightDir` 归一化为单位向量。
- `LdorN` 就是数学公式中的 $max(l \cdot n)$ 。
- `std::optional`在没有被赋值的情况下，它的状态被设置为`nullopt` 。`std::optional`类型有一个`operator bool()`成员函数，这个函数可以用来检查这个`std::optional`是否有值。如果有值，`operator bool()`会返回`true`，否则返回`false`。
- 然后，**检查这个点是否处在阴影中**，也就是说，是否有其他物体挡在了这个点和光源之间。这是通过发射一条从相交点到光源的射线并找到最近的交点来实现的。也就是说，检查`shadow_res`是否有值。如果`shadow_res`没有值（即`operator bool()`返回`false`），那么表达式的结果就是`false`，因为`&&`操作符有短路行为，即如果左操作数为`false`，则无论右操作数的值是什么，结果都是`false`。如果`shadow_res`有值（即`operator bool()`返回`true`），那么接着就会计算右边的表达式`(shadow_res->tNear * shadow_res->tNear < lightDistance2)` ，其中，`shadow_res->tNear` 是从一个点到其最近物体的距离。
- 如果这个点不在阴影中，我们就把光的强度加到`lightAmt`上。这个强度是光的强度乘以光线和法线的点积，也就是Lambert光照模型的部分。
- 我们还计算了反射方向，然后使用Phong模型的镜面部分来计算镜面高光的颜色。这个颜色是光的强度乘以反射方向和观察方向的点积的物体的镜面反射系数次幂。
- 最后，我们计算的颜色是`lightAmt`（光的强度）乘以物体的漫反射颜色和漫反射系数，再加上镜面高光颜色乘以物体的镜面反射系数。

最后一步，将计算结果保存到 `hitColor` 中。

```c+=
hitColor = lightAmt * payload->hit_obj->evalDiffuseColor(st) * payload->hit_obj->Kd + specularColor * payload->hit_obj->Ks;
```

参照的公式：
$$
\begin{aligned}
L & =L_a+L_d+L_s \\
& =k_a I_a+k_d\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{l})+k_s\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{h})^p
\end{aligned}
$$
但是这里仅仅计算了漫反射项和高光项。

这里需要注意的是，在一些简化的光照模型中，为了简化计算或者获得特定的视觉效果，有时会忽略这个因素。在本作业框架中，看起来似乎就是这样的情况。本框架作者可能假定了光源的强度已经考虑了距离的影响，或者只想要一个简化的模型，忽略了距离的影响。但是在除以距离的平方后，物体变成了黑色。

```c++
lightAmt += inShadow ? 0 : light->intensity * LdotN / lightDistance2;
...
specularColor += powf(std::max(0.f, -dotProduct(reflectionDirection, dir)),payload->hit_obj->specularExponent) * light->intensity / lightDistance2;
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/eJY8bvZ2LEHI4gG.png" alt="image-20230620103904689" style="zoom: 33%;" />

通过Debug我们发现 `lightDistance2` 这一项非常巨大，平均值在6000左右。这样计算光照衰减自然而然物体就是黑色的了。我们可以尝试通过修改Object.hpp文件的光照系数解决物体全黑的问题。

也可以修改main.cpp文件的32-33行的代码以调整光源位置：

```c++
scene.Add(std::make_unique<Light>(Vector3f(-20, 70, 20), 1));
scene.Add(std::make_unique<Light>(Vector3f(30, 50, -12), 1));
//改为：
scene.Add(std::make_unique<Light>(Vector3f(-10, 35, 10), 1000));
scene.Add(std::make_unique<Light>(Vector3f(15, 25, -6), 1000));
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/DRws14brH9uay8g.png" alt="image-20230620105913505" style="zoom: 33%;" />



### 6. `Renderer::Render`: 主渲染函数，它迭代处理图像中的所有像素，生成主光线，并将这些光线投入场景进行渲染

终于写到主渲染函数了，这里会遍历所有的像素，生成主射线，发射光线进行光线碰撞。最后写出到文件。

这部分也是我们作业的TODO部分，接下来我们详细看看吧。

1. **初始化framebuffer**：用一个指定大小的Vector3f向量（大小为场景宽度*场景高度）来创建framebuffer。

   ```c++
   std::vector<Vector3f> framebuffer(scene.width * scene.height);
   ```

2. **计算缩放因子和图像宽高比**：使用场景的视角（fov）来计算缩放因子。图像宽高比是用来保证图像的纵横比正确。

   ```c+=
   float scale = std::tan(deg2rad(scene.fov * 0.5f));
   float imageAspectRatio = scene.width / (float)scene.height;
   ```

3. **遍历每个像素**：对于场景中的每个像素，计算出一条从眼睛位置到该像素的主光线。

   ```c++
   for (int j = 0; j < scene.height; ++j){
       for (int i = 0; i < scene.width; ++i){
       ......
       }UpdateProgress(j / (float)scene.height);
   }
   ```

4. **生成主光线**：使用像素的坐标和缩放因子以及图像宽高比来计算光线的方向。方向向量需要被归一化。

   ```c++
   float x;
   float y;
   x = (2.f*((i+0.5)/scene.width)-1)*scale*imageAspectRatio;
   y = (1-2.f*((j+0.5)/scene.height))*scale;
   Vector3f dir = Vector3f(x, y, -1); // Don't forget to normalize this direction!
   dir = normalize(dir);
   ```

5. **光线追踪**：调用 `castRay` 函数来追踪光线，获取光线击中点的颜色。然后将这个颜色存储在framebuffer中。

   ```c++
   framebuffer[m++] = castRay(eye_pos, dir, scene, 0);
   ```

6. **保存结果到文件**：将framebuffer写入到一个PPM格式的文件中。PPM（Portable PixMap）格式是一种简单的图像格式，能够存储RGB颜色值。

   ```c++
   // save framebuffer to file
   ......
   ```

其中，详细讲讲TODO部分。

首先相机位置在原点，视角向 z 轴负方向，且视口平行于 xy 平面。

```c++
x = (2.f*((i+0.5)/scene.width)-1)*scale*imageAspectRatio;
y = (1-2.f*((j+0.5)/scene.height))*scale;
```

**`x = (2.f\*((i+0.5)/scene.width)-1)\*scale\*imageAspectRatio;`**

计算光线方向的 x 分量。`(i+0.5)/scene.width` 这部分将像素的 x 坐标（即 i）标准化到 [0,1] 范围内，`+0.5` 是为了取像素的中心点。接着，`(2.f * ((i+0.5)/scene.width) - 1)` 将这个值映射到 [-1, 1] 的范围内，这样视口的左边对应 -1，右边对应 1。

然后，我们将这个值乘以 `scale`，这个 `scale` 的计算是基于相机的视野（field of view，简写为 fov）。`scale = tan(deg2rad(scene.fov * 0.5))` 实际上是将 fov 的一半从角度转换为弧度，然后取正切值。这个操作的目的是将像素的坐标根据视野进行缩放，实现了透视投影。

最后，乘以 `imageAspectRatio` 是为了矫正横纵比。如果场景的宽度和高度不同，那么我们需要根据宽高比调整 x 分量，以防止图像被拉伸。

**`y = (1-2.f\*((j+0.5)/scene.height))\*scale;`**

这行代码是计算光线方向的 y 分量，它的操作大致相同于 x 分量的计算，唯一的区别在于没有乘以 `imageAspectRatio`，因为 y 分量不需要进行横纵比矫正。

请注意，在计算 y 分量时，`(1-2.f*((j+0.5)/scene.height))` 这个操作将 y 坐标反向，因此视口的上边对应 -1，下边对应 1。这是因为在计算机图形中，通常将图像的左上角作为原点，而 y 轴方向向下，但在我们的坐标系统中，我们希望 y 轴方向向上。

## Möller–Trumbore intersection algorithm

这是一个快速的光线三角形相交检测算法。

### 任务

```c++
bool rayTriangleIntersect(const Vector3f& v0, const Vector3f& v1, const Vector3f& v2, const Vector3f& orig,
	const Vector3f& dir, float& tnear, float& u, float& v)
{
    // TODO: Implement this function that tests whether the triangle
    // that's specified bt v0, v1 and v2 intersects with the ray (whose
    // origin is *orig* and direction is *dir*)
    // Also don't forget to update tnear, u and v.
    return false;
}
```

### 解析

函数传入了：

- `v0`, `v1`, `v2`：这三个参数代表了构成三角形的三个顶点的位置。
- `orig`：这是光线的原点（起始点）。
- `dir`：这是光线的方向。
- `tnear`：这是一个输出参数，如果光线与三角形相交，它将被设置为从光线原点到交点的距离。
- `u` 和 `v`：这两个也是输出参数，如果光线与三角形相交，它们将被设置为交点在三角形平面内的重心坐标。

可以在我[上一篇文章中查看](https://remoooo.com/cg/9-1.html#ci_title10)详细原理。这里只贴出最终的公式：
$$
\left[\begin{array}{c}t \\b_1 \\b_2\end{array}\right]
=
\frac{1}{\overrightarrow{\mathbf{S}}_1 \cdot \overrightarrow{\mathbf{E_1}}}\left[\begin{array}{c}
\overrightarrow{\mathbf{S_2}} \cdot \overrightarrow{\mathbf{E_2}} \\
\overrightarrow{\mathbf{S_1}} \cdot \overrightarrow{\mathbf{S}} \\
\overrightarrow{\mathbf{S_2}} \cdot \overrightarrow{\mathbf{D}}
\end{array}\right]
$$
算法的大致步骤是：

1. 首先计算三角形的两个边向量 `E1 = v1 - v0` 和 `E2 = v2 - v0`。
2. 计算从三角形一点 v0 指向光源点 orig 的向量 `S = orig - v0`。
3. 计算 `S1` 为 `dir` 和 `E2` 的叉积，`S2` 为 `S` 和 `E1` 的叉积。
4. 计算交点的重心坐标 `u(b_1)` 和 `v(b_2)`，以及从光线原点到交点的距离 `t`。
5. 最后检查 `t`, `u(b_1)` 和 `v(b_2)` 是否在有效范围内（`t >= 0`，`u >= 0`，`v >= 0`，并且 `u + v <= 1`），如果都在有效范围内，那么光线就与三角形相交，函数返回 `true`，否则返回 `false`。

注意，函数中使用了一个很小的正值 `EPLISON` 来处理浮点数精度问题，以避免错误地判定光线与三角形不相交。

## 附录1. 自我交叉问题

自我交叉，也被称为自阴影或精度问题。当数值精度不足以表示实际物理世界的精度时，就会产生这种问题。

具体到光线追踪算法中，当光线射到物体表面时，会产生反射光线或者折射光线。

在计算这些光线的交点时，由于计算机的浮点数精度问题，交点可能会被错误地计算在物体表面的内部或者物体表面的外部，而不是精确地在物体表面。这就可能导致光线追踪算法在下一次射线追踪时错误地判断光线和物体发生了交点，而实际上这个交点是由于浮点数精度错误产生的，并不是真正的交点。这就是自我交叉的问题。

本项目的解决方案就是在计算交点时，稍微偏移一点点距离，使得交点位于物体表面的外部，而不是位于物体表面。这样在下一次射线追踪时，就不会错误地判断光线和物体发生了交点。

## Reference

1. Fundamentals of Computer Graphics 4th

2. GAMES101 Lingqi Yan

3. 《Unity Shader入门精要》 - 冯乐乐 - 9787115423054

4. https://zhuanlan.zhihu.com/p/480405520

5. https://en.wikipedia.org/wiki/Fresnel_equations


[1]: https://remoooo.com/usr/uploads/2023/06/294363858.zip
