本章內容：

1. 頻域圖 Frequency Domain
1. 傅立葉變換 Fourier Transform
1. 濾波、卷積 Filtering&Convolution
1. 抗鋸齒 Antialiasing
1. 超採樣抗鋸齒 MSAA

<!--more-->

# 第四章 抗鋸齒 Antialiasing

上一章節內容：

1. 光柵概述
2. 深度緩衝 Z-buffer

上一章難度較小。這一章對於沒有學過信號分析的讀者難度較大。

## 反走样（Antialiasing）

反走样用于处理锯齿状边缘（Jaggies）和其他采样错误。

比如我们想要渲染一个三角形。一般先通过测试像素是否在三角形内。

**测试像素是否在三角形内**: 

在渲染三角形时，我们需要确定哪些像素位于三角形内部。常见的方法是计算像素中心的位置与三角形顶点的关系，判断该像素是否在三角形的包围盒内。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/G2ivynZrpw5ECA6.png" alt="image-20230523172853287" style="zoom:33%;" />

通过上述操作，就会生成如下图案：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/cqLN6jCi9v8wxPr.png" alt="image-20230523173007624" style="zoom:33%;" />

而我们想要的是：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/cQIKmTtdSnYAby4.png" alt="image-20230523173053787" style="zoom:33%;" />

这种差异就叫做Sampling Artifacts。

### 走样（Sampling Artifacts）

**Jaggies**: "Jaggies"是指在图形或文本的边缘出现的锯齿形状。这是因为计算机显示器是由像素组成的，而像素是方形的，当尝试用这些方形像素来表示非垂直或非水平的线条时，就会出现锯齿状边缘。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/F65YahfpHeUQKwW.png" alt="image-20230523174758196" style="zoom:33%;" />

**Moiré Patterns**:Moiré条纹是另一种常见的取样伪像，通常在渲染高频率细节（如细小纹理或远处的网格线）时出现。因为像素的尺寸限制，当试图渲染过于细微的细节时，就会出现这种干涉模式。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/gtjG1FR7rJxLhP3.png" alt="image-20230523174721558" style="zoom: 33%;" />

如何解决这些Sampling Artifacts？

### 预滤波 (Pre-Filtering) 

英文全称：Antialiasing Idea: Blurring (Pre-Filtering) Before Sampling。

反走样的基本思想是在取样之前对图像进行模糊（也称为预滤波）。这种方法可以将高频信息“扩散”到多个像素，从而减少锯齿和Moire模式。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/RHkcumMXtE1FjKI.png" alt="image-20230523175144545" style="zoom:33%;" />

直接点采样和反走样的区别：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/R8qim61hGjoQa4S.png" alt="image-20230523175437264" style="zoom:33%;" />



But，Why？

- 为什么先模糊再采样会获得更好的图像。

## 频域 Frequency Domain

频域，也称为**频率域（Frequency Domain）**：信号或数据可以在两个主要的领域（或空间）中表示：时间域和频率域。时间域显示的是信号随时间的变化情况，而频率域则显示的是**信号的频率成分**。在频率域中，我们可以直观地看到信号包含的不同频率成分及其相对强度。

比如同时按下钢琴（不考虑泛音）的两个键，那么在这段声音的频率域中就会有两个凸起的峰，峰的在x轴（频率）位置对应声音的频率。两个琴键按下的力度也可以在频域图中根据峰的高低得到判断。

## 傅立叶变换 Fourier Transform

用于将信号从时间域转换到频率域，或从频率域转换回时间域。

它的基本原理是将任意信号分解为一系列正弦和余弦波的叠加。
$$
F(\omega)=\int_{-\infty}^{\infty} f(x) e^{-2 \pi i \omega x} d x
$$

## 逆傅立叶变换 Inverse Fourier Transform

逆傅立叶变换是傅立叶变换的逆过程。

将频率域的数据转换回时间域或空间域。

逆傅立叶变换对于信号处理和数据分析非常重要，因为我们常常需要在频率域进行处理和分析，然后再将结果转换回时间域以实际使用。
$$
\begin{gathered}
f(x)=\int_{-\infty}^{\infty} F(\omega) e^{2 \pi i \omega x} d \omega \\
\text { Recall } e^{i x}=\cos x+i \sin x
\end{gathered}
$$

## 采样定理

“采样定理”也称为“奈奎斯特定理”。

奈奎斯特定理指出，如果要在数字化时无损地恢复出一个模拟信号，那么采样频率（采样率）必须至少达到信号最高频率的两倍。这个最小采样频率被称为“奈奎斯特频率”。

1. **低频信号**：对于低频信号，如果采样率足够高（即达到了信号最高频率的两倍），则可以比较精确地重构出原始信号。
2. **高频信号**：如果采样率未达到信号最高频率的两倍，那么在数字化过程中就会丢失部分信息，无法无损地恢复出原始信号。这种情况下，高频信号可能会被误识别为低频信号，产生“混叠”现象。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/csEUtlB39VOar7Q.png" alt="image-20230523182318742" style="zoom:33%;" />

所以对于高频信号，需要更快速的采样才能避免采样结果的失真。

## 滤波（Filtering）

滤波是一种处理信号的方法，其目的是去除或强化信号中的某些频率成分。

- **低通滤波器（Low-Pass Filter）**会去除高频成分，只留下低频成分，这通常会使图像模糊（因为高频成分通常对应于图像的细节和边缘）

- **高通滤波器（High-Pass Filter）**则去除低频成分，只留下高频成分，这常常可以强化图像的边缘
- **带通滤波器（Band-Pass Filter）**则去除过高或过低的频率成分，只保留中间的频率范围

## 卷积（Convolution）

这里的卷积指的是一种数学运算。

它可以用于实现滤波。滤波器（也被称为“卷积核”或“卷积滤波器”）在图像上滑动，计算滤波器覆盖的区域和滤波器的加权平均值，这就相当于在空间域进行了平滑或增强的操作。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/u8QWMc4hTqZDEBe.png" alt="卷积图片" style="zoom:33%;" />

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/HbrnfRvBW8KCDUd.png" alt="卷积结果" style="zoom:33%;" />

## 卷积定理（Convolution Theorem）

- 在空间（或时间）域中两个函数的卷积等于它们在频率域中的乘积，反之亦然。

- 这个定理在图像处理和计算机图形学中有许多应用，比如快速模糊和锐化算法。

两个操作（两个效果相等）：

1. 在空间域进行滤波
2. 通过傅立叶变换将信号转换到频域，乘上卷积核的傅立叶变换，将乘法结果通过逆傅立叶变换转换回时域（或空间域）。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/BKQ5b6xzI2RWw7S.png" alt="image-20230523222649975" style="zoom: 50%;" />

上面的例子中，那个3x3的方块（还有一个系数）也被称为**均值滤波器**（**Box Filter**）。它的工作原理很简单：对每个像素，计算其周围的像素的平均值，然后将结果赋给当前像素。因此，Box filter具有平滑和模糊的效果，能够减少图像的噪声和细节。

## 采样（Sampling）

采样是将连续信号转换为离散信号的过程，它通过在一定的时间间隔内对信号进行测量来实现。

在频率域中，采样过程等价于将原始信号的频谱在频率轴上重复（或镜像）。怎么理解？

首先看一个例子，如下图所示（https://www.researchgate.net/figure/The-evolution-of-sampling-theorem-a-The-time-domain-of-the-band-limited-signal-and-b_fig5_301556095）：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/TcL2pPnbQ6UjJl4.png" alt="image-20230523225752666" style="zoom:50%;" />

上图中，左侧是时域信号，右侧是左侧经过傅立叶变换的频域信号。

(a)是原始信号，(c)称为冲击函数，将(a)(c)相乘得到(e)。

同时，(b)(d)(f)是(a)(c)(e)的频域图。

根据上文的卷积定理，「在空间（或时间）域中两个函数的卷积等于它们在频率域中的乘积」，我们可以得知(b)和(d)的卷积等于(f)。

- 也就是说，**采样就是在重复原始信号的频谱**。

采样是一种过程，将连续的信号转换为离散的信号。在一定的间隔（采样间隔）内记录信号的值。

现在，让我们从频域的角度来理解采样。（也就是上图的右侧）

任何时域（或空间域）中的信号都可以通过傅立叶变换转换到频域，也就是说，我们可以看到信号中各种频率成分的强度。

在这个频域中，低频成分位于中心，而高频成分位于两侧。

当我们对信号进行采样时，我们实际上是在时域（或空间域）中将连续信号变为离散信号。

在频域中，这个操作会使原始信号的频谱在频率轴上重复。换句话说，采样会在频域中创建原始信号频谱的副本，这些副本在频谱中以采样频率的整数倍的位置出现。

这就是“**采样就是在重复原始信号的频谱**”的含义。

但是，如果采样率（即每秒采样点的数量）过低，那么在频域中的这些副本可能会重叠，即所谓的混叠（Aliasing）。

## 混叠（Aliasing）

当采样率低于信号最高频率的两倍时，高频信号可能会被误识别为低频信号，这种现象被称为混叠。

在频率域中，混叠是由于采样导致的频谱重叠所引起的。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/j5HFOPdCWuxUXtc.png" alt="image-20230523231306194" style="zoom:33%;" />

**稠密采样（Dense sampling）和稀疏采样（Sparse sampling）**：稠密采样指的是采样率高的情况，这时采样点之间的间隔较小。反之，稀疏采样指的是采样率低的情况，这时采样点之间的间隔较大。稠密采样更可能恢复出原始信号，而稀疏采样更容易引发混叠。

混叠会使得原本的高频信息被误解为低频信息，这会导致图像和声音等的失真。

因此，按照奈奎斯特定理，采样率应至少达到信号最高频率的两倍，以避免混叠现象的发生。

### 抗混叠方法

1. **增加采样率**。这会增加频率域中副本之间的距离，从而减少混叠。但这种方法可能需要较高的分辨率，而且成本较高。
2. **抗混叠**。这通过在采样前滤除高频成分，即使频率内容在重复前变得“更窄”，从而减少混叠。在图形学中，常用的抗混叠技术包括超采样（Supersampling）、多重采样（Multisampling）和后处理抗混叠（Post-processing antialiasing）等。

## 抗混叠具体方法

#### 常规采样（Regular Sampling）

会出现大量锯齿和混叠效果。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/NdIQh61T9icGKJF.png" alt="image-20230523233307626" style="zoom:50%;" />

#### 抗混叠采样（Antialiased Sampling）

经过抗锯齿的采样，效果非常好，减少混叠。

这种方法在确定像素颜色时考虑了像素区域内的更多信息。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/Sbp6YtsoOe258km.png" alt="image-20230523233319224" style="zoom:50%;" />

#### 实际的预滤波器（A Practical Pre-Filter）

在采样前对图像应用一个宽度为1像素的盒形滤波器，可以模糊图像并过滤掉高频信息，这也是一种简单的抗混叠方法。

![image-20230523234025636](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/kSwbh8QTiJm9oZC.png)

#### 通过计算像素区域内的平均值进行抗混叠（Antialiasing By Averaging Values in Pixel Area）

这种方法通过在每个像素区域内对颜色值进行平均（即卷积操作），然后在每个像素中心处采样，从而实现抗混叠。

在光栅化一个三角形时，如果像素部分覆盖了三角形，那么我们就可以通过计算像素内部被三角形覆盖的面积，来得到像素的平均颜色值。

## 超采样（Supersampling）

一种常见的抗混叠技术，它通过在每个像素内部进行多次采样并平均得到的值，来近似实现一个像素宽的盒形滤波器的效果。

通常步骤如下：

1. **步骤1：在每个像素内部取NxN个样本。**这就意味着，如果我们对每个像素进行4次采样（即N=2），那么实际的采样率就会增加4倍。
2. **步骤2：平均每个像素内部的NxN个样本。**这意味着我们要计算这些样本的平均值，并将结果作为像素的颜色。

## Reference

[1] Fundamentals of Computer Graphics 4th

[2] GAMES101 Lingqi Yan
