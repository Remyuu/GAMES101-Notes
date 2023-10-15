本章快速瀏覽計算機圖形學相關的數學知識，包裹：
1. 向量 Vector
2. 矩陣 Matrix
3. 基礎變換矩陣 Transformation Matrices
4. 齊次坐標 Homogeneous coordinates

<!--more-->

# 第一章 數學基礎

需要掌握的數學基礎：

1. Linear algebra（重點）
2. calculus
3. statistics

對於計算機圖形學，更多需要掌握的是**線性代數**的

- 向量（點積，叉積）
- 矩陣

## 1.1 向量 Vector

### 1.1.1 寫法

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/IEeurLwTBOXWhSy.png" alt="image-20230408120445445" style="zoom: 33%;" />

- 一般寫作 $$\vec{a}$$ 或者加粗的 **a**
- 或者寫作 $$\overrightarrow{A B}=B-A$$
- 有長度、方向
- 不規定起點

### 1.1.2 標準化 Normalization

- 向量的長度（Magnitude or length）寫作 $$\|\vec{a}\|$$

- 單位向量
  - 長度為1的向量
  - 將向量標準化 $$\hat{a}=\vec{a} /\|\vec{a}\|$$
  - 用於表示向量的方向


### 1.1.3 向量運算 Vector Operations

向量加法 Vector Addition

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/UcG9wnpk3iZa7oY.png" alt="image-20230408135742529" style="zoom: 33%;" />

- 幾何滿足：平行四邊形法則 & 三角形法則
- 代數滿足：向量各分量相加

### 1.1.4 笛卡爾座標系 Cartesian Coordinates

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/BUop9cVvhlNF8AH.png" alt="image-20230408140639520" style="zoom:50%;" />
$$
A=\left(\begin{array}{l}x\\y\end{array}\right)
$$

$$
\quad A^T=(x, y)
$$

$$
\quad\|A\|=\sqrt{x^2+y^2}
$$

### 1.1.5 向量點積 Dot (scalar) Product

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/1ToRDUamExniC4g.png" alt="image-20230408141128845" style="zoom: 67%;" />
$$
\begin{gathered}
\vec{a} \cdot \vec{b}=\|\vec{a}\|\|\vec{b}\| \cos \theta \\
\cos \theta=\frac{\vec{a} \cdot \vec{b}}{\|\vec{a}\|\|\vec{b}\|}
\end{gathered}
$$

- Dot product properties:

$$
a \cdot b=b \cdot a \\
a \cdot(b+c)=a \cdot b+a \cdot c \\
(k a) \cdot b=a \cdot(k b)=k a \cdot b
$$

- Dot Product in Cartesian Coordinates

$$
\begin{aligned}
&-\ln 2 \mathrm{D} \\
& \vec{a} \cdot \vec{b}=\left(\begin{array}{l}
x_a \\
y_a
\end{array}\right) \cdot\left(\begin{array}{l}
x_b \\
y_b
\end{array}\right)=x_a x_b+y_a y_b \\
&-\ln 3 \mathrm{D} \\
& \vec{a} \cdot \vec{b}=\left(\begin{array}{l}
x_a \\
y_a \\
z_a
\end{array}\right) \cdot\left(\begin{array}{l}
x_b \\
y_b \\
z_b
\end{array}\right)=x_a x_b+y_a y_b+z_a z_b
\end{aligned}
$$

- 點積應用
  - 求兩個向量的夾角
  - 找向量在另一個向量的投影
  - 衡量兩個向量的方向的相似度
  - 分解向量
  - ...

### 1.1.6 向量叉積 Cross (vector) Product

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/AywxpIKlro9bfJO.png" alt="image-20230408142620454" style="zoom:50%;" />
$$
\|a \times b\|=\|a\|\|b\| \sin \phi
$$

- 滿足右手系
- Cross product properties:

$$
x×y = +z, \\
y×x = −z, \\
y×z = +x, \\
z×y = −x, \\
z×x = +y, \\
x×z = −y.
$$

- 叉積：笛卡爾公式

$$
\vec{a} \times \vec{b}=\left(\begin{array}{c}
y_a z_b-y_b z_a \\
z_a x_b-x_a z_b \\
x_a y_b-y_a x_b
\end{array}\right)
$$
$$
\vec{a} \times \vec{b}=A^* b=\left(\begin{array}{ccc}
0 & -z_a & y_a \\
z_a & 0 & -x_a \\
-y_a & x_a & 0
\end{array}\right)\left(\begin{array}{l}
x_b \\
y_b \\
z_b
\end{array}\right)
$$

推導過程：
$$
\begin{aligned}
\mathbf{a} \times \mathbf{b}= & \left(x_a \mathbf{x}+y_a \mathbf{y}+z_a \mathbf{z}\right) \times\left(x_b \mathbf{x}+y_b \mathbf{y}+z_b \mathbf{z}\right) \\
= & x_a x_b \mathbf{x} \times \mathbf{x}+x_a y_b \mathbf{x} \times \mathbf{y}+x_a z_b \mathbf{x} \times \mathbf{z} \\
& +y_a x_b \mathbf{y} \times \mathbf{x}+y_a y_b \mathbf{y} \times \mathbf{y}+y_a z_b \mathbf{y} \times \mathbf{z} \\
& +z_a x_b \mathbf{z} \times \mathbf{x}+z_a y_b \mathbf{z} \times \mathbf{y}+z_a z_b \mathbf{z} \times \mathbf{z} \\
= & \left(y_a z_b-z_a y_b\right) \mathbf{x}+\left(z_a x_b-x_a z_b\right) \mathbf{y}+\left(x_a y_b-y_a x_b\right) \mathbf{z} .
\end{aligned}
$$
$$
\mathbf{a} \times \mathbf{b}=\left(y_a z_b-z_a y_b, z_a x_b-x_a z_b, x_a y_b-y_a x_b\right)
$$

### 1.1.7 標準正交基和座標系

- 對任意3個向量有

$$
\begin{gathered}
\|\vec{u}\|=\|\vec{v}\|=\|\vec{w}\|=1 \\
\vec{u} \cdot \vec{v}=\vec{v} \cdot \vec{w}=\vec{u} \cdot \vec{w}=0 \\
\vec{w}=\vec{u} \times \vec{v} \quad \text { (right-handed) } \\
\vec{p}=(\vec{p} \cdot \vec{u}) \vec{u}+(\vec{p} \cdot \vec{v}) \vec{v}+(\vec{p} \cdot \vec{w}) \vec{w}
\end{gathered}
$$

## 1.2 矩陣 Matrix

- 在圖形學中，矩陣可以表示為：平移、旋轉、剪切和縮放

### 1.2.1 定義

- m行n列的數組

$$
\left(\begin{array}{ll}
1 & 3 \\
5 & 6 \\
1 & 4
\end{array}\right)
$$

### 1.2.2 矩陣乘法

$$
\left[\begin{array}{ccc}
a_{11} & \ldots & a_{1 m} \\
\vdots & & \vdots \\ a_{i 1} & \ldots & a_{i m} \\ \vdots & & \vdots \\
a_{r 1} & \ldots & a_{r m}
\end{array}\right]
\left[\begin{array}{ccccc}
b_{11} & \ldots & b_{1 j} & \ldots & b_{1 c} \\
\vdots & & \vdots & & \vdots \\
b_{m 1} & \ldots & b_{m j} & \ldots & b_{m c}
\end{array}\right]
=
\left[\begin{array}{ccccc}
p_{11} & \cdots & p_{1 j} & \cdots & p_{1 c} \\
\vdots & & \vdots & & \vdots \\
p_{i 1} & \cdots & p_{i j} & \cdots & p_{i c} \\
\vdots & & \vdots & & \vdots \\
p_{r 1} & \cdots & p_{r j} & \cdots & p_{r c}
\end{array}\right]
$$

其中，
$$
p_{i j}=a_{i 1} b_{1 j}+a_{i 2} b_{2 j}+\cdots+a_{i m} b_{m j}
$$

### 1.2.3 矩陣性質 Properties

- 沒有交換律
  - AB BA不相等
- 結合律和分配律
  - (AB)C = A(BC)
  - A(B+C) = AB + AC
  - (A+B)C = AC + BC

### 1.2.4 矩陣向量乘法 Matrix-Vector Multiplication

- 關於 y 軸翻轉

$$
\left(\begin{array}{}
-1 & 0 \\
0 & 1
\end{array}\right)
\left(\begin{array}{}
x \\
 y
\end{array}\right)
=
\left(\begin{array}{}
-x \\
 y
\end{array}\right)
$$

- 關於 x 軸翻轉

$$
\left(\begin{array}{}
1 & 0 \\
0 & -1
\end{array}\right)
\left(\begin{array}{}
x \\
 y
\end{array}\right)
=
\left(\begin{array}{}
x \\
-y
\end{array}\right)
$$

### 1.2.5 矩陣轉置 Transpose of a Matrix

- 交換行列（ij -> ji）

$$
\left(\begin{array}{ll}
1 & 2 \\
3 & 4 \\
5 & 6
\end{array}\right)^T=\left(\begin{array}{lll}
1 & 3 & 5 \\
2 & 4 & 6
\end{array}\right)
$$
- 性質

$$
(A B)^T=B^T A^T
$$

### 1.2.6 單位矩陣和逆矩陣 Identity Matrix and Inverses

$$
I_{3 \times 3}=\left(\begin{array}{ccc}
1 & 0 & 0 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{array}\right) \\
$$

$$
A A^{-1}=A^{-1} A=I \\
$$

$$
(A B)^{-1}=B^{-1} A^{-1}
$$

### 1.2.7 矩陣形式的向量乘法

- 點積
$$
\begin{aligned}
& \vec{a} \cdot \vec{b}=\vec{a}^T \vec{b} \\
= & \left(\begin{array}{lll}
x_a & y_a & z_a
\end{array}\right)\left(\begin{array}{l}
x_b \\
y_b \\
z_b
\end{array}\right)=\left(x_a x_b+y_a y_b+z_a z_b\right)
\end{aligned}
$$
- 叉積

$$
\vec{a} \times \vec{b}=A^* b=\left(\begin{array}{ccc}
0 & -z_a & y_a \\
z_a & 0 & -x_a \\
-y_a & x_a & 0
\end{array}\right)\left(\begin{array}{l}
x_b \\
y_b \\
z_b
\end{array}\right)
$$

## 1.3 基礎變換矩陣 Transformation Matrices

- 用矩陣表示轉換
- 縮放（scale）、 翻轉（Reflection）、旋轉（Rotation）、拉伸（shear）

### 1.3.1 縮放矩陣 Scale Matrix

- 縮放定義為：

$$
\operatorname{scale}\left(s_x, s_y\right)=\left[\begin{array}{cc}
s_x & 0 \\
0 & s_y
\end{array}\right] .
$$
- 縮放對矩陣做了什麼？

$$
\left[\begin{array}{cc}
s_x & 0 \\
0 & s_y
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]=\left[\begin{array}{l}
s_x x \\
s_y y
\end{array}\right] .
$$

- 例子：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/pGTvkjmfCMlwRDs.png" alt="image-20230408154330557" style="zoom:50%;" />
$$
\left[\begin{array}{l}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{ll}
s & 0 \\
0 & s
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]
$$
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/pkzUbjwPgaRTs3h.png" alt="image-20230408154430826" style="zoom:50%;" />
$$
\left[\begin{array}{l}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{cc}
s_x & 0 \\
0 & s_y
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]
$$

### 1.3.2 翻轉矩陣 Reflection Matrix

- 定義：

$$
\operatorname{reflect-x}(s)=\left[\begin{array}{}
-1 & 0 \\
0 & 1
\end{array}\right]
,
\operatorname{reflect-y}(s)=\left[\begin{array}{}
1 & 0 \\
0 & -1
\end{array}\right]
$$



<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/L9y5QqtMckUCaxN.png" alt="image-20230408154815241" style="zoom:50%;" />

- 水平翻轉

$$
\begin{aligned}
& x^{\prime}=-x \\
& y^{\prime}=y
\end{aligned} \quad\left[\begin{array}{l}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{cc}
-1 & 0 \\
0 & 1
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]
$$

### 1.3.3 拉伸矩陣 Shear Matrix

- 定義：

$$
\operatorname{shear-x}(s)=\left[\begin{array}{}
1 & s \\
0 & 1
\end{array}\right]
,
\operatorname{shear-y}(s)=\left[\begin{array}{}
1 & 0 \\
s & 1
\end{array}\right]
$$

- 例子：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/UIO1DCYtucQTKbg.png" alt="image-20230408154937046" style="zoom:50%;" />
$$
\left[\begin{array}{}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{cc}
1 & a \\
0 & 1
\end{array}\right]\left[\begin{array}{}
x \\
y
\end{array}\right]
$$

### 1.3.4 旋轉矩陣 Rotation Matrix

- **默認關於原點（0,0）逆時針旋轉**

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/MsPI1HRNmzxd6gS.png" alt="image-20230408155212728" style="zoom:50%;" />
$$
\mathbf{R}_\theta=\left[\begin{array}{cc}
\cos \theta & -\sin \theta \\
\sin \theta & \cos \theta
\end{array}\right]
$$

## 1.4 齊次坐標 Homogeneous coordinates

- 思考：**為什麼引入齊次座標**？
- 答：為了解決**平移**問題，因為「平移」這個操作無法在現有的一處變換矩陣中表示。詳細解釋如下：

我們想要表示下面的平移矩陣，

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/KX5u9kaBCRqlGvt.png" alt="image-20230408162236085" style="zoom:50%;" />
$$
\begin{aligned}
& x^{\prime}=x+t_x \\
& y^{\prime}=y+t_y
\end{aligned}
$$
顯然的，這個變換並不是**線性變換**。
$$
\left[\begin{array}{l}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{ll}
a & b \\
c & d
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]+\left[\begin{array}{l}
t_x \\
t_y
\end{array}\right]
$$

- 如何用**統一**的方式表示**平移**和**基礎變換**呢？
- 答案就是 **齊次坐標** Homogeneous coordinates。

### 1.4.1 定義

在原來的座標/向量中增加多一個維度 $ w $。

- 2D座標 = $(x,y,1)^T$
- 2D向量 = $(x,y,0)^T$
  - 例子：$\left[\begin{array}{}2\\3\\1\end{array}\right]$是一個座標，$\left[\begin{array}{}2\\3\\0\end{array}\right]$是一個向向量。

### 1.4.2 性質

- 用矩陣表示變換（translations）：

$$
\left(\begin{array}{c}
x^{\prime} \\
y^{\prime} \\
w^{\prime}
\end{array}\right)=\left(\begin{array}{ccc}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{array}\right) \cdot\left(\begin{array}{l}
x \\
y \\
1
\end{array}\right)=\left(\begin{array}{c}
x+t_x \\
y+t_y \\
1
\end{array}\right)
$$

- 其中，$w$維度只允許0或1
  - 向量 + 向量 = 向量
  - 座標 - 座標 = 向量
  - 座標 + 向量 = 座標
  - 座標 + 座標 = **不合法！**

- 在齊次座標系中，$\left(\begin{array}{}x\\y\\w\end{array}\right)$代表二維空間中的一個點$\left(\begin{array}{}x/w\\y/w\\1\end{array}\right),w\ne0$.

### 1.4.3 仿射映射&齊次座標 Affine map & HC

- Affine map = linear map + translation

$$
\left[\begin{array}{l}
x^{\prime} \\
y^{\prime}
\end{array}\right]=\left[\begin{array}{ll}
a & b \\
c & d
\end{array}\right]\left[\begin{array}{l}
x \\
y
\end{array}\right]+\left[\begin{array}{l}
t_x \\
t_y
\end{array}\right]
$$

- Using HC

$$
\left(\begin{array}{c}
x^{\prime} \\
y^{\prime} \\
w^{\prime}
\end{array}\right)=\left(\begin{array}{ccc}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{array}\right) \cdot\left(\begin{array}{l}
x \\
y \\
1
\end{array}\right)
$$

### 1.4.4 2D變換 2D Transformations

$$
\begin{aligned}
&Scale:\mathbf{S}\left(s_x, s_y\right)=\left(\begin{array}{ccc}
s_x & 0 & 0 \\
0 & s_y & 0 \\
0 & 0 & 1
\end{array}\right)\\
&Rotation:\mathbf{R}(\alpha)=\left(\begin{array}{ccc}
\cos \alpha & -\sin \alpha & 0 \\
\sin \alpha & \cos \alpha & 0 \\
0 & 0 & 1
\end{array}\right)\\
&Translation:\mathbf{T}\left(t_x, t_y\right)=\left(\begin{array}{ccc}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{array}\right)
\end{aligned}
$$

### 1.4.5 逆變換 Inverse Transform

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/FDT1yqxjb72EHWM.png" alt="image-20230408165113057" style="zoom:50%;" />



### 1.4.5 變換的分解與合成 Composition and Decomposition of Transformations

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/G4rg2jby69ozmwi.png" alt="image-20230408160507059" style="zoom:50%;" />

- 思考：**先平移再縮放還是先縮放再平移？**

- 答：**先縮放再平移**。

- 圖解：

  <img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/I4JgN71mxksnLfZ.png" alt="image-20230408160745053" style="zoom:33%;" />



- 原因是，矩陣不存在交換律。

$$
R_{45} \cdot T_{(1,0)} \neq T_{(1,0)} \cdot R_{45}
$$

- 注意，矩陣是從右算到左：

$$
T_{(1,0)} \cdot R_{45}\left[\begin{array}{l}
x \\
y \\
1
\end{array}\right]=\left[\begin{array}{lll}
1 & 0 & 1 \\
0 & 1 & 0 \\
0 & 0 & 1
\end{array}\right]\left[\begin{array}{ccc}
\cos 45^{\circ} & -\sin 45^{\circ} & 0 \\
\sin 45^{\circ} & \cos 45^{\circ} & 0 \\
0 & 0 & 1
\end{array}\right]\left[\begin{array}{l}
x \\
y \\
1
\end{array}\right]
$$

#### 1.4.5.1 合成

- 我們可以把上面式子中的旋轉矩陣和平移矩陣合而為一，理論依據：

$$
A_n\left(\ldots A_2\left(A_1(\mathbf{x})\right)\right)=\mathbf{A}_n \cdots \mathbf{A}_2 \cdot \mathbf{A}_1 \cdot\left(\begin{array}{l}
x \\
y \\
1
\end{array}\right)
$$

#### 1.4.5.2 分解

- 思考：**怎麼使一個圖像繞任意點 $c$ 旋轉呢**？

- 答：平移 $c$ 點到原點 $(0,0)$ ，旋轉，再平移回去。

  即：$T(c)\cdot R(\alpha)\cdot T(-c)$

----

## Reference

[1] Fundamentals of Computer Graphics 4th

[2] GAMES101 Lingqi Yan