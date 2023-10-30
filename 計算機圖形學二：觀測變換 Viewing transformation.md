本章內容：

1. 三維變換 3D transformations
2. 羅德里格斯旋轉公式 Rodrigues' rotation formula
3. 模型變換 modeling tranformation
4. 視圖變換 **camera/view** transformation
5. 投影變換（正交、透視）projection tranformation
6. 視口變換 viewport tranformation
7. 變換模型代碼(c++) C++ Code

<!--more-->

# 第二章 視圖、投影變換

上一章節內容：

- 什麼是變換矩陣
- 二維空間的變換：旋轉rotation、縮放scale、剪切shear
- 齊次座標系Homogeneous coordinates

上一章的內容比較基礎，理解難度不大。本章相較於上一章節，難度較大。

這一章內容：

- 三維變換 3D transformations
  
  - 羅德里格斯旋转公式 Rodrigues' rotation formula（難點）

- 觀測變換（視圖變換） Viewing transformation
  
  - 模型變換 modeling tranformation
  - **攝像機**/**視圖**變換 **camera**/**view** transformation
  - 投影變換 projection tranformation
    - 正交變換 Orthographic projection
    - 透視變換 Perspective projection（難點）
  - 視口變換 viewport tranformation
  - 變換模型代碼(c++) C++ Code

## 2.1 3D變換 3D Transformations

- 3D座標 = $(x,y,z,1)^T$

- 3D向量 = $(x,y,z,0)^T$

- 在齊次座標系中，$\left(\begin{array}{}x\\y\\z\\w\end{array}\right)$代表三維空間中的一個點$\left(\begin{array}{}x/w\\y/w\\z/w\\1\end{array}\right),w\ne0$.

- 用4x4的矩陣表示三位仿射變換：
  
  $$
  \left(\begin{array}{l}
x^{\prime} \\
y^{\prime} \\
z^{\prime} \\
1
\end{array}\right)=\left(\begin{array}{lllc}
a & b & c & t_x \\
d & e & f & t_y \\
g & h & i & t_z \\
0 & 0 & 0 & 1
\end{array}\right) \cdot\left(\begin{array}{l}
x \\
y \\
z \\
1
\end{array}\right)
\tag{1}
  $$

### 2.1.1 縮放與平移 Scale&Translation

基本與二維的旋轉類似。

- 縮放 Scale

$$
\mathbf{S}\left(s_x, s_y, s_z\right)=\left(\begin{array}{cccc}
s_x & 0 & 0 & 0 \\
0 & s_y & 0 & 0 \\
0 & 0 & s_z & 0 \\
0 & 0 & 0 & 1
\end{array}\right)\\
\tag{2}
$$

- 平移 Translation

$$
\mathbf{T}\left(t_x, t_y, t_z\right)=\left(\begin{array}{cccc}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{array}\right)
\tag{3}
$$

### 2.1.2 拉伸 Shear

可以僅僅進行二維的拉伸：

與2D變換一樣，任何3D變換矩陣都可以用SVD分解為旋轉、縮放和旋轉。任何對稱的3D矩陣都具有旋轉、縮放和逆旋轉的**特征值分解**。最後，三維旋轉可分解為三維拉伸（Shear）矩陣的乘積。
$$
\operatorname{shear-x}\left(d_y, d_z\right)=\left[\begin{array}{ccc}
1 & d_y & d_z \\
0 & 1 & 0 \\
0 & 0 & 1
\end{array}\right]
\tag{4}
$$

### 2.1.3 旋轉 Rotate

比二維旋轉複雜得多，因為可能旋轉的軸更多。但是，我們如果只想繞著某一個軸旋轉，則只會改變另外兩個軸。

#### 2.1.3.1 繞一個軸旋轉

- 比方說，我們繞z軸旋轉，則只需要改變x和y軸，z軸保持不動（右側是基於齊次座標下的，下面同理）：

$$
\operatorname{rotate-\mathrm {z}}(\phi)=\left[\begin{array}{ccc}
\cos \phi & -\sin \phi & 0 \\
\sin \phi & \cos \phi & 0 \\
0 & 0 & 1
\end{array}\right]
\tag{5}
$$

$$
\mathbf{R}_x(\alpha)=\left(\begin{array}{cccc}
1 & 0 & 0 & 0 \\
0 & \cos \alpha & -\sin \alpha & 0 \\
0 & \sin \alpha & \cos \alpha & 0 \\
0 & 0 & 0 & 1
\end{array}\right)
\tag{6}
$$

- 類似的，繞x軸則只改變y和z：

$$
\operatorname{rotate-x}(\phi)=\left[\begin{array}{ccc}
1 & 0 & 0 \\
0 & \cos \phi & -\sin \phi \\
0 & \sin \phi & \cos \phi
\end{array}\right]
\tag{7}
$$

$$
\mathbf{R}_z(\alpha)=\left(\begin{array}{cccc}
\cos \alpha & -\sin \alpha & 0 & 0 \\
\sin \alpha & \cos \alpha & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{array}\right)
\tag{8}
$$

- 繞y軸同理：

$$
\operatorname{rotate-y}(\phi)=\left[\begin{array}{ccc}
\cos \phi & 0 & \sin \phi \\
0 & 1 & 0 \\
-\sin \phi & 0 & \cos \phi
\end{array}\right]
\tag{9}
$$

$$
\mathbf{R}_y(\alpha)=\left(\begin{array}{cccc}
\cos \alpha & 0 & \sin \alpha & 0 \\
0 & 1 & 0 & 0 \\
-\sin \alpha & 0 & \cos \alpha & 0 \\
0 & 0 & 0 & 1
\end{array}\right) \\
\tag{10}
$$

但是我們發現，y軸 **(9),(10)** 這裡的有一些**奇怪**。

- 思考：**為什麼繞著y軸旋轉時，方向是負的呢？**
- 提示：x軸叉乘z軸，等於-y。

#### 2.1.3.2 復合旋轉

- 將 $\mathbf{R}_x(\alpha),\mathbf{R}_y(\alpha),\mathbf{R}_z(\alpha)$ 復合：

$$
\mathbf{R}_{xyz}(\alpha)=\mathbf{R}_x(\alpha)\mathbf{R}_y(\alpha)\mathbf{R}_z(\alpha)
$$

- 即歐拉角 Euler angles。

- **四元數**不在本章節討論範圍。

#### 2.1.3.3 羅德里格斯旋转公式 Rodrigues' rotation formula

- 公式的作用：把任意的旋轉都寫成矩陣的形式。

- 繞$n$軸旋轉角度$\alpha$:

$$
\mathbf{R}(\mathbf{n}, \alpha)=\cos (\alpha) \mathbf{I}+(1-\cos (\alpha)) \mathbf{n} \mathbf{n}^T+\sin (\alpha) \underbrace{\left(\begin{array}{ccc}
0 & -n_z & n_y \\
n_z & 0 & -n_x \\
-n_y & n_x & 0
\end{array}\right)}_{\mathbf{N}}
\tag{11}
$$

- 其中，$n$軸默認是過圓點$(0,0)$的且是單位向量；$N=n^*$，即$N$是$n$的矩陣形式（需要可以回顧上一章節 [1.2.7](https://remoooo.com/cg/1.html#ci_title16) 的內容）。

- 推導思路：假設是向量$a$繞著n軸旋轉至**$b$**，則將**$a$**分解為平行於n軸的分量（發現是不變的）加上垂直於n軸的分量。

- 詳細推導過程：https://www.bilibili.com/video/BV1Eu411r7GC/ 

## 2.2 觀測變換 View Transformation

前面的內容，我們學習了如何使用矩陣變換在2D或3D空間中對物體進行旋轉、平移、縮放和裁切等一系列幾何變換（geo- metric transformations）操作。幾何變換的第二個用途就是用於觀測變換，也就是一種3D到2D的映射。下圖展示了將對象從原始座標轉換到屏幕空間的空間和變換序列：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/auH6dk4fpKo9wUL.png" alt="image-20230409145104403" style="zoom:50%;" />

如圖所示，觀測變換分為四個步驟：

1. 模型變換 **m**odeling tranformation
2. 攝像機/視圖變換 camera/**v**iew transformation
3. 投影變換 **p**rojection transformation
4. 視口變換 viewport transformation

提取上面三個步驟的首字母，**MVP**，也就很容易記住觀測變換的三大步驟了。

接下來，我們依次講解。

### 2.2.1 模型變換 modeling tranformation

- 調整場景中模型的位置、大小、旋轉至合適的形態

### 2.2.2 攝像機/視圖變換 camera/**v**iew transformation

- 我們把計算機圖形想像成一個攝影的過程，攝像機就是鏡頭的中心。

- 攝像機/視圖變換（下文簡稱攝像機變換）的目的是，**獲取所有在攝像機視線範圍的物體與攝像機之間的相對位置**。非常直觀的做法就是以攝像機為中心，換一句話說，就是把攝像機調整至世界座標的原點上。怎麼讓攝像機與世界座標原點重合呢？

#### 2.2.2.1 攝像機定義

- 攝像機的位置 $e$
- 注視的角度 $g$
- 攝像機上方向 $t$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/mhkljyRMaio7BQ9.png" alt="image-20230409151353699" style="zoom: 33%;" />

- 通過上面三個定義，即可建立以$e$為原點，基底為 $u,v,w$ 的攝像機座標系（如下圖所示）。

- $$
  \begin{aligned}
w & =-\frac{g}{\|g\|} \\
u & =\frac{t \times w}{\|t \times w\|} \\
v & =w \times u
\end{aligned}
\tag{12}
  $$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/nh1NygOwr6xQjtH.png" alt="image-20230409152654276" style="zoom:50%;" />

- $u,v,w$ 分別對應笛卡爾座標系中的 $x,y,z$ 
- $g$ 向量對應 $z$ 軸（ $w$ 軸）的負方向
- $e$ 向量對應座標原點

#### 2.2.2.2 將相機座標系移動至原世界座標系的原點

分為兩個步驟：

1. 將相機移動至原點
2. 通過旋轉矩陣將座標系重合

第一步非常簡單，只需要將當前相機座標減去相機座標，即：$\left[\begin{array}{cccc}1 & 0 & 0 & -x_e \\0 & 1 & 0 & -y_e \\0 & 0 & 1 & -z_e \\0 & 0 & 0 & 1\end{array}\right]$.

第二步，由於旋轉變換矩陣是一種正則基矩陣（正交矩陣 The orthogonal matrix），矩陣$M^T=M^{-1}$，所以將單位矩陣$I$旋轉至**當前**相機狀態的矩陣的逆矩陣則是：$\left[\begin{array}{cccc}x_u & y_u & z_u &0 \\x_v & y_v & z_v & 0 \\x_w & y_w & z_w & 0 \\0 & 0 & 0 & 1\end{array}\right]$.

所以，定義相機變換矩陣 $M_{cam}$ ：
$$
\mathbf{M}_{\text {cam }}=\left[\begin{array}{cccc}
\mathbf{u} & \mathbf{v} & \mathbf{w} & \mathbf{e} \\
0 & 0 & 0 & 1
\end{array}\right]^{-1}=\left[\begin{array}{cccc}
x_u & y_u & z_u & 0 \\
x_v & y_v & z_v & 0 \\
x_w & y_w & z_w & 0 \\
0 & 0 & 0 & 1
\end{array}\right]\left[\begin{array}{cccc}
1 & 0 & 0 & -x_e \\
0 & 1 & 0 & -y_e \\
0 & 0 & 1 & -z_e \\
0 & 0 & 0 & 1
\end{array}\right]
\tag{13}
$$

### 2.2.3 投影變換 projection transformation

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/3aTJFQLzfWZuDOB.png" alt="image-20230409155021065" style="zoom: 33%;" />

https://stackoverflow.com/questions/36573283/from-perspective-picture-to-orthographic-picture

----

- 投影變換分為**正交投影變換**和**透視投影變換**。

- 投影變換，簡單的說就是3D->2D的變換。

- <img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/dZ6G1kAMvqNJCnw.png" alt="image-20230409154758458" style="zoom:25%;" />左圖是正交投影變換、右圖是透視投影變換。

#### 2.2.3.1 正交投影變換 The Orthographic Projection Transformation

- 攝像機在世界座標原點，看向 $-Z$ 軸， 正上方是 $Y$ 軸。
- 將某可視區域**長方體**壓縮至 $[-1,1]^3$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/74AvTNa9Hdx5z8e.png" alt="image-20230409155838430" style="zoom:50%;" />

- 定義這個**長方體**，即攝像機可視空間：
  - **x = l ≡ left plane,**
  - **x = r ≡ right plane,**
  - **y = b ≡ bottom plane,**
  - **y = t ≡ top plane,** 
  - **z = n ≡ near plane,** 
  - **z = f ≡ far plane.**

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/gCU9hcOuTMv2nxp.png" alt="image-20230409160222459" style="zoom:50%;" />

- 則很容易推倒出將**長方體**變換到中心是$(0,0,0)$ 且所在區域是$[-1,1]^3$的變換矩陣$M_{orth}$：

$$
\mathbf{M}_{\text {orth }}=\left[\begin{array}{cccc}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{n-f} & -\frac{n+f}{n-f} \\
0 & 0 & 0 & 1
\end{array}\right]
\tag{14}
$$

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/SZBCnaNwIPUqylm.png" alt="image-20230409161026394" style="zoom: 33%;" />

#### 2.2.3.2 透視投影變換 Perspective Projection

- 透視和正交的區別（左透視、右正交）：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/8hQcfE5LyJx4j17.png" alt="image-20230409161413677" style="zoom: 33%;" />

- 透視投影過程很簡單，**我們先將Frustum“壓縮”成Cuboid**，也就是**先將透視投影轉換為正交投影**，然後就可以利用2.2.3.1的知識做投影變換了。
- 壓縮如下圖所示。我們需要找出“壓縮”這一步的變換矩陣$M_{\text {persp } \rightarrow \text { ortho }}^{(4 \times 4)}$。

求解步驟如下：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/sflDA65h8d4S3xb.png" alt="image-20230409161624353" style="zoom:50%;" />

- 1. 將$(x,y,z)$投影到屏幕上，就變成了$(x',y',z')$
  2. 原點是視點，$z=-n$表示投影平面，利用**相似三角形**得出投影後的$x,y$座標

- 現在我們清楚的是，xOy平面的變換過程，但是z怎麼變化尚不清楚。
  
  1. $x'=\frac{n}{z}x$
  2. $y'=\frac{n}{z}y$
  3. $z'=??$

- 回顧一個齊次座標的性質
  
  - $(x,y,z,1)=(kx,ky,kz,k\ne0)$
  - $(x,y,z,1)=(xz,yz,z2,z\ne0)$
  - 例子：$(2,0,0,2)=(1,0,0,1)$，都代表三維空間中的點$(1,0,0)$。

- 在齊次座標中，我們可以根據上面的推倒，寫出：

$$
\left(\begin{array}{c}x \\y \\z \\1\end{array}\right)\stackrel{M_{persp\rightarrow ortho}^{(4 \times 4)}} \Rightarrow\left(\begin{array}{c}n x / z \\n y / z \\\text { unknown } \\1\end{array}\right)
\stackrel{\text {mult. by z}}{==}
\left(\begin{array}{c}n x \\n y \\\text { still unknown } \\z\end{array}\right)
\tag{15}
$$

- 我們想要求處上面方程中的變換矩陣$M_{\text {persp } \rightarrow \text { ortho }}^{(4 \times 4)}$，則只需解**方程（16）**：

$$
M_{persp\rightarrow ortho}^{(4 \times 4)}
\left(\begin{array}{c}x \\y \\z \\1\end{array}\right) =
\left(\begin{array}{c}n x \\n y \\n\cdot \text {unknown} \\z\end{array}\right)
\tag{16}
$$

- 由於等號右邊第三行未知，我們只能寫出“壓縮”變換矩陣的一部分：

$$
M_{p e r s p \rightarrow \text { ortho }}=\left(\begin{array}{cccc}
n & 0 & 0 & 0 \\
0 & n & 0 & 0 \\
? & ? & ? & ? \\
0 & 0 & 1 & 0
\end{array}\right)
\tag{17}
$$

- 利用透視投影的性質：**將Frustum“壓縮”成Cuboid後，n面和f面的$z$座標不變。**

- 我們將 $z=n$ 帶入**方程（16）**中，得到**方程（18）**：

$$
M_{persp\rightarrow ortho}^{(4 \times 4)}
\left(\begin{array}{l}x \\y \\n \\1\end{array}\right)
=
\left(\begin{array}{l}n x \\n y \\n^{2} \\n\end{array}\right)
\tag{18}
$$

- 由**方程（18）**，將變換矩陣$M$的第三行單獨拿出來，並且方程右邊的$n^2$肯定與$x,y$**無關**，所以得到**方程（19）**：

$$
\left(\begin{array}{llll}0 & 0 & A & B\end{array}\right)
\left(\begin{array}{l}x \\y \\n \\1\end{array}\right)
=n^2
\Rightarrow
An + B =n^2
\tag{19}
$$

- 且我們欲求的變換矩陣$M_{persp \rightarrow ortho}$第三行的前兩個數字也確定了

- 又因為遠平面$f$的中間點$(0,0,f)$經過變換依然是$(0,0,f)$，所以將$(0,0,f)$帶入**方程（16）**，得到**方程（20）**：

$$
\left(\begin{array}{cccc}
n & 0 & 0 & 0 \\
0 & n & 0 & 0 \\
0 & 0 & A & B \\
0 & 0 & 1 & 0
\end{array}\right)
\left(\begin{array}{l}0 \\0 \\f \\1\end{array}\right)
=
\left(\begin{array}{c}0 \\0 \\f^2 \\f\end{array}\right)
\Rightarrow
Af+b=f^2
\tag{20}
$$

- 由**方程（19）（20）**，解得：

$$
\left\{\begin{array}{l}
A=n+f \\
B=-n f
\end{array}\right.
\tag{21}
$$

- 則求出“壓縮”變換矩陣$M_{persp \rightarrow ortho}$：

$$
M_{persp\rightarrow ortho}=\left(\begin{array}{cccc}
n & 0 & 0 & 0 \\
0 & n & 0 & 0 \\
0 & 0 & n+f & -nf \\
0 & 0 & 1 & 0
\end{array}\right)
\tag{22}
$$

- 所以，透視投影$M_{persp}=M_{ortho}M_{persp\rightarrow ortho}$

- 計算最終結果：

$$
\mathbf{M}_{\mathrm{persp}}=\left[\begin{array}{cccc}
\frac{2 n}{r-l} & 0 & \frac{l+r}{l-r} & 0 \\
0 & \frac{2 n}{t-b} & \frac{b+t}{b-t} & 0 \\
0 & 0 & \frac{f+n}{n-f} & \frac{2 f n}{f-n} \\
0 & 0 & 1 & 0
\end{array}\right]
\tag{23}
$$

#### 2.2.3.3 關於透視矩陣的練習題

- **問：在Frustum內部，任意一點經過了“壓縮”變換矩陣$M_{persp \rightarrow ortho}$的變換後，會向哪個方向移動？**

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/8hQcfE5LyJx4j17-20230719181618709.png" alt="image-20230409161413677" style="zoom: 33%;" />

- 答：對於Frustum中的任意一點 $\left(\begin{array}{l}x \\y \\z \\1\end{array}\right)$ ，其中 $(n<x<f)$ ，經過“擠壓”變換都有：
- $M_{persp \rightarrow ortho}\left(\begin{array}{l}x \\y \\z \\1\end{array}\right)=\left(\begin{array}{l}nx \\ny \\(n+f)z-nf \\z\end{array}\right)$ 
- 由於齊次座標的性質，上面等價於：
- $\left(\begin{array}{l}nx/z \\ny/z \\(n+f)-nf/z \\1\end{array}\right)$
- 現在問題轉換為判斷變化前後 $z$ 分量的大小關係。
- $\text{let } z=\frac{n+f}{z}$ 經過變化得到點 $z'$
- 易得 $z'=\frac{n^2+f^2}{n+f}$
- 使 $z,z'$ 做差，得 $z'-z = \frac{(n-f)^2}{2(n+f)}$ 
- 又因為 $0>n>f$ 
- 所以 $z'-z<0$ ，所以變化後的點向遠平面靠近（距離屏幕更遠）。

### 2.2.4 視口變換 viewport transformation

- 視口（viewport），其實就是攝像機畫面。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/PEwQca1MtKTgqbL.png" alt="image-20230409173942468" style="zoom:50%;" />

- 做完模型、攝像機、投影變換之後，所有需要的物體都在$[-1,1]^3$這個立方體內了。
- 我們將$x=-1$放在屏幕的左側，$x=+1$仿仔屏幕的右側，$y=-1$放在屏幕的底部，$y=+1$放在屏幕的頂部。
- 屏幕上每一個像素都“擁有”1個以整數座標為中心的單位正方形。在渲染的時候，需要將 $[-1,1]^2$ 映射到 $[-0.5,n_x-0.5]*[-0.5,n_y-0.5]$ ，公式如下：

$$
M_{\mathrm{viewport}}=\left[\begin{array}{cccc}
\frac{n_x}{2} & 0 & 0 & \frac{n_x-1}{2} \\
0 & \frac{n_y}{2} & 0 & \frac{n_y-1}{2} \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{array}\right]
\tag{24}
$$

- 如果需要將 $[-1,1]^2$ 映射到 $[0,n_x]*[0,n_y]$ ，則公式如下：

$$
M_{\mathrm{viewport}}=\left[\begin{array}{cccc}
\frac{n_x}{2} & 0 & 0 & \frac{n_x}{2} \\
0 & \frac{n_y}{2} & 0 & \frac{n_y}{2} \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{array}\right]
\tag{25}
$$

- $z$方向沒有任何改變

### 2.2.5 觀測變換矩陣$M$

$$
M=M_{viewport}M_{persp}M_{cam}M_{model}
\tag{26}
$$

- 模型、相機、投影、視口

### 2.2.5 變換模型代碼(c++) C++ Code（附）

1. 模型變換 Modeling Transformation

這裡的3D變換是繞 $z$ 軸旋轉，即**變換矩陣（6）**：
$$
\mathbf{R}_x(\alpha)=\left(\begin{array}{cccc}
1 & 0 & 0 & 0 \\
0 & \cos \alpha & -\sin \alpha & 0 \\
0 & \sin \alpha & \cos \alpha & 0 \\
0 & 0 & 0 & 1
\end{array}\right) \\
\tag{6}
$$
直接通過3D變換就可以寫出```get_model_matrix()```模型變換函數。

需要注意：

- 函數傳入的是角度制，需要先將角度轉換為弧度。

```c++
Eigen::Matrix4f get_model_matrix(float rotation_angle)
{
    Eigen::Matrix4f model = Eigen::Matrix4f::Identity();
    auto radian = (float)(rotation_angle/180.0*MY_PI);
    Eigen::Matrix4f translate;
    translate << cos(radian),-sin(radian),0,0,
                 sin(radian),cos(radian),0,0,
                 0,0,1,0,
                 0,0,0,1;
    model = translate * model;
    return model;
}
```

2. 攝像機/視圖變換 camera/view transformation

默認相機的朝向一定正確，只需要調整當前攝像機的位置向量```eye_pos```至世界座標的原點即可。

```c++
Eigen::Matrix4f get_view_matrix(Eigen::Vector3f eye_pos)
{
    Eigen::Matrix4f view = Eigen::Matrix4f::Identity();
    Eigen::Matrix4f translate;
    translate << 1, 0, 0, -eye_pos[0],
                 0, 1, 0, -eye_pos[1],
                 0, 0, 1,  eye_pos[2],
                 0, 0, 0, 1;
    view = translate * view;
    return view;
}
```

3. 投影透視變換 Perspective Projection

本函數需要寫出兩個矩陣：

$$
M_{persp\rightarrow ortho}=\left(\begin{array}{cccc}
n & 0 & 0 & 0 \\
0 & n & 0 & 0 \\
0 & 0 & n+f & -nf \\
0 & 0 & 1 & 0
\end{array}\right)
\tag{22}
$$




$$
\mathbf{M}_{\text {orth }}=\left(\begin{array}{cccc}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{n-f} & -\frac{n+f}{n-f} \\
0 & 0 & 0 & 1
\end{array}\right)
\tag{14}
$$

```c++
Eigen::Matrix4f get_projection_matrix(float eye_fov, float aspect_ratio, float zNear, float zFar){
    Eigen::Matrix4f projection = Eigen::Matrix4f::Identity();
    Eigen::Matrix4f M_p2o = Eigen::Matrix4f ::Identity();
    M_p2o <<    zNear,0,0,0,
                0,zNear,0,0,
                0,0,zNear+zFar,-zNear*zFar,
                0,0,1,0;
    Eigen::Matrix4f M_orth = Eigen::Matrix4f ::Identity();
    float fov = 0.5*eye_fov*MY_PI/180;
    float top = tan(fov) * zNear;
    float bottom = -top;
    float right = top * aspect_ratio;
    float left = -right;
    M_orth <<   2/(right-left),0,0,-(right+left)/(right-left),
                0,2/(top-bottom),0,-(top+bottom)/(top-bottom),
                0,0,2/(zNear-zFar),-(zNear+zFar)/(zNear-zFar),
                0,0,0,1;
    projection = M_orth * M_p2o;
    return projection;
}
```

4. 羅德里格斯旋轉公式代碼實現

實現以下公式即可。
$$
\mathbf{R}(\mathbf{n}, \alpha)=\cos (\alpha) \mathbf{I}+(1-\cos (\alpha)) \mathbf{n} \mathbf{n}^T+\sin (\alpha) \underbrace{\left(\begin{array}{ccc}
0 & -n_z & n_y \\
n_z & 0 & -n_x \\
-n_y & n_x & 0
\end{array}\right)}_{\mathbf{N}}
\tag{11}
$$

```c++
Eigen::Matrix4f get_rotation(Vector3f axis, float angle){
    Eigen::Matrix4f M_rodrigues = Eigen::Matrix4f::Identity();
    Eigen::Matrix3f M_temp_rodrigues = Eigen::Matrix3f::Identity();
    Eigen::Matrix3f M_I = Eigen::Matrix3f::Identity();
    Eigen::Matrix3f M_N;
    M_N <<  0, -axis[2], axis[1],
            axis[2], 0, -axis[0],
            -axis[1], axis[0], 0;
    auto radian = (float)(angle/180.0*MY_PI);
    M_temp_rodrigues << cos(radian)*M_I+(1-cos(radian))*axis*axis.transpose()+sin(radian)*M_N;
    M_rodrigues <<  M_temp_rodrigues(0,0), M_temp_rodrigues(0,1), M_temp_rodrigues(0,2), 0,
            M_temp_rodrigues(1,0), M_temp_rodrigues(1,1), M_temp_rodrigues(1,2), 0,
            M_temp_rodrigues(2,0), M_temp_rodrigues(2,1), M_temp_rodrigues(2,2), 0,
            0, 0, 0, 1;
    return M_rodrigues;
}
```

## Reference

[1] Fundamentals of Computer Graphics 4th

[2] GAMES101 Lingqi Yan
