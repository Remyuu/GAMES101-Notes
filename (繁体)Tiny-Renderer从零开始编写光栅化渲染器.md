# 簡易的光柵化渲染器

本文是一個完整的圖形學入門實踐課程，目前還在更新中，GitHub已開源。理論上本文項目需要20-30個小時完成。不知道爲啥我的網站統計字數也有問題。

主要內容是完全手擼一個光柵化渲染器。本文會從頭複習圖形學以及C++的相關知識，包括從零構造向量模版庫、光柵化原理解釋、圖形學相關基礎算法解釋等等內容。

另外原作者的的透視矩陣部分是經過一定程度的簡化的，與虎書等正統做法不同。我會先按照原文ssloy老師的思想表達關鍵內容，最後按照我的想法完善本文。並且，原項目中的數學向量矩陣庫寫得不是很好，我專門開了一章一步步重構這個庫。

> 原項目鏈接：https://github.com/ssloy/tinyrenderer
>
> 本項目鏈接：https://github.com/Remyuu/Tiny-Renderer

<!--more-->

[TOC]

## 0 簡單的開始

五星上將曾經說過，懂的越少，懂的越多。我接下來將提供一個tgaimage的模塊，你說要不要仔細研究研究？我的評價是不需要，如學。畢竟懂的越多，懂的越少。

在這裏提供一個最基礎的[框架🔗](https://github.com/Remyuu/Tiny-Renderer/tree/1.1_Bresenham’s_Line_Drawing_Algorithm)，他只包含了tgaimage模塊。該模塊主要用於生成.TGA文件。以下是一個最基本的框架代碼：

```c++
// main.cpp
#include "tgaimage.h"

const TGAColor white = TGAColor(255, 255, 255, 255);
const TGAColor red   = TGAColor(255, 0,   0,   255);
const TGAColor blue = TGAColor(0, 0, 255, 255);

int main(int argc, char** argv) {
    TGAImage image(100, 100, TGAImage::RGB);
    // TODO: Draw sth
    image.flip_vertically(); // i want to have the origin at the left bottom corner of the image
    image.write_tga_file("output.tga");
    return 0;
}
```

上面代碼會創建一個100*100的image圖像，並且以tga的格式保存在硬盤中。我們在TODO中添加代碼：

```c++
image.set(1, 1, red);
```

代碼作用是在(1, 1)的位置將像素設置爲紅色。output.tga的圖像大概如下所示：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230830170319909.png" alt="0.0" style="zoom: 400%;" />

## 1.1 畫線

這一章節的目標是畫線。具體而言是製作一個函數，傳入兩個點，在屏幕上繪製線段。

### 第一關：實現畫線

給定空間中的兩個點，在兩點(x0, y0)(x1, y1)之間繪製線段。

最簡單的代碼如下：

```c++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (float t=0.; t<1.; t+=.01) { 
        int x = x0 + (x1-x0)*t; 
        int y = y0 + (y1-y0)*t; 
        image.set(x, y, color); 
    } 
}
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230830171306158.png" alt="image-20230830171306158" />

### 第二關：發現BUG

上面代碼中的.01其實是錯誤的。不同的分辨率對應的繪製步長肯定不一樣，太大的步長會導致：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230830171714587.png" alt="image-20230830171714587" />

所以我們的邏輯應該是：需要畫多少像素點就循環Draw多少次。最簡單的想法可能是繪製x1-x0個像素或者是y1-y0個像素：

```c++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    for (int x=x0; x<=x1; x++) {
        float t = (x-x0)/(float)(x1-x0);
        int y = y0*(1.-t) + y1*t;
        image.set(x, y, color);
    }
}
```

上面代碼是最簡單的插值計算。但是這個算法是錯誤的。畫三條線：

```c++
line(13, 20, 80, 40, image, white); 
line(20, 13, 40, 80, image, red); 
line(80, 40, 13, 20, image, blue);
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230830172534739.png" alt="image-20230830172534739" />

白色線看起來非常好，紅色線看起來斷斷續續的，藍色線直接看不見了。於是總結出以下兩個問題：

1. 理論上說白色線和藍色線應該是同一條線，只是起點與終點不同
2. 太“陡峭”的線效果不對

接下來就解決這個兩個問題。

> 此處“陡峭”的意思是(y1-y0)>(x1-x0)
>
> 下文“平緩”的意思是(y1-y0)<(x1-x0)

### 第三關：解決BUG

爲了解決起點終點順序不同導致的問題，只需要在算法開始時判斷兩點x分量的大小：

```c++
if (x0>x1) {
    std::swap(x0, x1); 
    std::swap(y0, y1); 
}
```

爲了畫出沒有空隙的“陡峭”線，只需要將“陡峭”的線變成“平緩”的線。最終的代碼：

```c++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    if(std::abs(x0-x1)<std::abs(y0-y1)) { // “陡峭”線
        if (y0 > y1) { // 確保從下到上畫畫
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        for (int y = y0; y <= y1; y++) {
            float t = (y - y0) / (float) (y1 - y0);
            int x = x0 * (1. - t) + x1 * t;
            image.set(x, y, color);
        }
    }
    else { // “平緩”線
        if (x0 > x1) { // 確保從左到右畫畫
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        for (int x = x0; x <= x1; x++) {
            float t = (x - x0) / (float) (x1 - x0);
            int y = y0 * (1. - t) + y1 * t;
            image.set(x, y, color);
        }
    }
}
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230830174932047.png" alt="image-20230830174932047" />

如果你想測試你自己的代碼是否正確，可以嘗試繪製出以下的線段：

```c++
line(25,25,50,100,image,blue);
line(25,25,50,-50,image,blue);
line(25,25,0,100,image,blue);
line(25,25,0,-50,image,blue);

line(25,25,50,50,image,red);
line(25,25,50,0,image,red);
line(25,25,0,0,image,red);
line(25,25,0,50,image,red);

line(25,25,50,36,image,white);
line(25,25,50,16,image,white);
line(25,25,0,16,image,white);
line(25,25,0,36,image,white);
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230831145010708.png" alt="image-20230831145010708" style="zoom:200%;" />

### 第四關：優化前言

目前爲止，代碼運行得非常順利，並且具備良好的可讀性與精簡度。但是，畫線作爲渲染器最基礎的操作，我們需要確保其足夠高效。

性能優化是一個非常複雜且系統的問題。在優化之前需要明確優化的平臺和硬件。在GPU上優化和CPU上優化是完全不同的。我的CPU是Apple Silicon M1 pro，我嘗試繪製了9,000,000條線段。

發現在line()函數內，`image.set();`函數佔用時間比率是38.25%，構建TGAColor對象是19.75%，14%左右的時間花在內存拷貝上，剩下的25%左右的時間花費則是我們需要優化的部分。下面的內容我將以運行時間作爲測試指標。

### 第五關：Bresenham's 優化

我們注意到，for循環中的除法操作是不變的，因此我們可以將除法放到for循環外面。並且通過斜率估計每向前走一步，另一個軸的增量error。dError是一個誤差積累，一旦誤差積累大於半個像素（0.5），就對像素進行一次修正。

```c++
// 第一次優化的代碼
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    if(std::abs(x0-x1)<std::abs(y0-y1)) { // “陡峭”線
        if (y0>y1) {
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        int dx = x1 - x0;
        int dy = y1 - y0;
        float dError = std::abs(dx / float(dy));
        float error = 0;
        int x = x0;
        for (int y = y0; y <= y1; y++) {
            image.set(x, y, color);
            error += dError;
            if (error>.5) {
                x += (x1>x0?1:-1);
                error -= 1.;
            }
        }
    }else { // “平緩”線
        if (x0>x1) {
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        int dx = x1 - x0;
        int dy = y1 - y0;
        float dError = std::abs(dy / float(dx));
        float error = 0;
        int y = y0;
        for (int x = x0; x <= x1; x++) {
            image.set(x, y, color);
            error += dError;
            if (error>.5) {
                y += (y1>y0?1:-1);
                error -= 1.;
            }
        }
    }
}
```

> 沒有優化用時：2.98s
>
> 第一次優化用時：2.96s

### 第六關：注意流水線預測

在很多教程當中，爲了方便修改，會用一些trick將“陡峭”的線和“平緩”的線的for循環代碼整合到一起。即先將“陡峭”線兩點的xy互換，最後再image.set()的時候再換回來。

```c++
// 逆向優化的代碼
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    bool steep = false;
    if (std::abs(x0-x1)<std::abs(y0-y1)) {
        std::swap(x0, y0);
        std::swap(x1, y1);
        steep = true;
    }
    if (x0>x1) {
        std::swap(x0, x1);
        std::swap(y0, y1);
    }
    int dx = x1-x0;
    int dy = y1-y0;
    float dError = std::abs(dy/float(dx));
    float error = 0;
    int y = y0;
    for (int x=x0; x<=x1; x++) {
        if (steep) {
            image.set(y, x, color);
        } else {
            image.set(x, y, color);
        }
        error += dError;
        if (error>.5) {
            y += (y1>y0?1:-1);
            error -= 1.;
        }
    }
}
```

>沒有優化用時：2.98s
>
>第一次優化用時：2.96s
>
>合併分支用時：3.22s

驚奇地發現，竟然有很大的性能下降！背後的原因之一寫在了這一小節的標題中。這是一種剛剛我們的操作增加了控制冒險（**Control Hazard**）。合併分支後的代碼每一次for循環都有一個分支，可能導致流水線冒險。這是現代處理器由於預測錯誤的分支而導致的性能下降。而第一段代碼中for循環沒有分支，分支預測可能會更準確。

簡而言之，減少for循環中的分支對性能的提升幫助非常大！

值得一提的是，如果在Tiny-Renderer中使用本文的操作，速度將會進一步提升。這在Issues中也有相應討論：[鏈接🔗](https://github.com/ssloy/tinyrenderer/issues/28)。

### 第七關：浮點數整型化

爲什麼我們必須用浮點數呢？在循環中我們只在與0.5做比較的時候用到了。因此我們完全可以將error乘個2再乘個dx（或dy），將其完全轉化爲int。

```c++
// 第二次優化的代碼
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) {
    int error2 = 0;
    if(std::abs(x0-x1)<std::abs(y0-y1)) { // “陡峭”線
        if (y0>y1) {
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        int dx = x1 - x0;
        int dy = y1 - y0;
        int dError2 = std::abs(dx) * 2;
        int x = x0;
        for (int y = y0; y <= y1; y++) {
            image.set(x, y, color);
            error2 += dError2;
            if (error2>dy) {
                x += (x1>x0?1:-1);
                error2 -= dy * 2;
            }
        }
    }else { // “平緩”線
        if (x0>x1) {
            std::swap(x0, x1);
            std::swap(y0, y1);
        }
        int dx = x1 - x0;
        int dy = y1 - y0;
        int dError2 = std::abs(dy) * 2;
        int y = y0;
        for (int x = x0; x <= x1; x++) {
            image.set(x, y, color);
            error2 += dError2;
            if (error2>dx) {
                y += (y1>y0?1:-1);
                error2 -= dx*2;
            }
        }
    }
}
```

>沒有優化用時：2.98s
>
>第一次優化用時：2.96s
>
>合併分支用時：3.22s
>
>第二次優化用時：2.96s

優化程度也較爲有限了，原因是在浮點數化整的過程中增加了計算的次數，與浮點數的計算壓力相抵消了。

## 1.2 三維畫線

在前面的內容中，我們完成了Line()函數的編寫。具體內容是給定屏幕座標上的兩個點就可以在屏幕中繪製線段。

### 第一關：加載.obj

首先，我們創建model類作爲物體對象。我們在model加載的.obj文件裏可能會有如下內容：

```.obj
v 1.0 2.0 3.0
```

v表示3D座標，後面通常是三個浮點數，分別對應空間中的x, y, z。上面例子代表一個頂點，其座標爲 `(1.0, 2.0, 3.0)`。

當定義一個面（`f`）時，你引用的是先前定義的頂點（`v`）的索引。

```.obj
f 1 2 3
f 1/4/1 2/5/2 3/6/3
```

上面兩行都表示一個面，

- 第一行表示三個頂點的索引
- 第二行表示頂點/紋理座標/法線的索引

在這裏我提供一個簡單的 .obj 文件解析器 model.cpp 。你可以在此處找到當前項目[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/1.2_Wireframe_rendering)。以下是你可能用到的model類的信息：

- 模型面數量：`i<model->nfaces()`
- 獲取第n個面的三個頂點索引：`model->face(n)`
- 通過索引獲取頂點三維座標：`model->vert()`

> 本項目使用的.obj文件的所有頂點數據已做歸一化，也就是說v後面的三個數字都是在[-1, 1]之間。

### 第二關：繪製

在這裏我們僅僅考慮三維頂點中的(x, y)，不考慮深度值。最終在main.cpp中通過model解析出來的頂點座標繪製出所有線框即可。

```c++
for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
    for (int j=0; j<3; j++) { 
        Vec3f v0 = model->vert(face[j]); 
        Vec3f v1 = model->vert(face[(j+1)%3]); 
        int x0 = (v0.x+1.)*width/2.; 
        int y0 = (v0.y+1.)*height/2.; 
        int x1 = (v1.x+1.)*width/2.; 
        int y1 = (v1.y+1.)*height/2.; 
        line(x0, y0, x1, y1, image, blue); 
    } 
}
```

這段代碼對所有的面進行迭代，將每個面的三條邊都進行繪製。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230831145239967.png" alt="image-20230831145239967" />

### 第三關：優化

將不必要的計算設置爲const，避免重複分配釋放內存。

```c++
const float halfWidth = screenWidth / 2.0f;
const float halfHeight = screenHeight / 2.0f;

int nfaces = model->nfaces();
for (int i = 0; i < nfaces; ++i) {
    const std::vector<int>& face = model->face(i);
    Vec3f verts[3];
    
    for (int j = 0; j < 3; ++j) {
        verts[j] = model->vert(face[j]);
    }

    for (int j = 0; j < 3; ++j) {
        const Vec3f& v0 = verts[j];
        const Vec3f& v1 = verts[(j + 1) % 3];
        
        int x0 = (v0.x + 1.0f) * halfWidth;
        int y0 = (v0.y + 1.0f) * halfHeight;
        int x1 = (v1.x + 1.0f) * halfWidth;
        int y1 = (v1.y + 1.0f) * halfHeight;
        
        line(x0, y0, x1, y1, image, blue);
    }
}

```

## 2.1 三角形光柵化

接下來，繪製完整的三角形，不光是一個個三角形線框，更是要一個實心的三角形！爲什麼是三角形而不是其他形狀比如四邊形？因爲三角形可以任意組合成爲所有其他的形狀。基本上，在OpenGL中絕大多數都是三角形，因此我們的渲染器暫時無需考慮其他的東西了。

當繪製完一個實心的三角形後，完整渲染一個模型也就不算難事了。

在Games101的作業中，我們使用了AABB包圍盒與判斷點是否在三角形內的方法對三角形光柵化。你完全可以用自己的算法繪製三角形，在本文中，我們使用割半法處理。

### 第一關：線框三角形

利用上一章節完成的line()函數，進一步將其包裝成繪製三角形線框的triangleLine()函數。

```c++
void triangleLine(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color){
    line(v0.u, v0.v, v1.u, v1.v, image, color);
    line(v0.u, v0.v, v2.u, v2.v, image, color);
    line(v1.u, v1.v, v2.u, v2.v, image, color);
}
...
triangleLine(Vec2i(0,0),Vec2i(25,25),Vec2i(50,0),image,red);
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230831145749059.png" alt="image-20230831145749059" style="zoom:200%;" />

### 第二關：請你自己畫實心的三角形

這一部分最好由你自己花費大約一個小時完成。一個好的三角形光柵化算法應該是簡潔且高效的。你目前的項目大概是這樣的：[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/2.1_Filling_triangles)。

【此處省略一小時】

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230831213131128.png" alt="image-20230831213131128" style="zoom:200%;" />

### 第三關：掃描線算法

當你完成了你的算法之後，不妨來看看其他人是怎麼做的。爲了光柵化一個實心三角形，一種非常常見的方法是使用掃描線算法：

1. 按 `v`（或 `y`）座標對三角形的三個頂點進行排序，使得 `v0` 是最低的，`v2` 是最高的。
2. 對於三角形的每一行（從 `v0.v` 到 `v2.v`），確定該行與三角形的兩邊的交點，並繪製一條從左交點到右交點的線。

```c++
void triangleRaster(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color) {
    if (v0.v > v1.v) std::swap(v0, v1);
    if (v0.v > v2.v) std::swap(v0, v2);
    if (v1.v > v2.v) std::swap(v1, v2);

    // Helper function to compute the intersection of the line and a scanline
    auto interpolate = [](int y, Vec2i v1, Vec2i v2) -> int {
        if (v1.v == v2.v) return v1.u;
        return v1.u + (v2.u - v1.u) * (y - v1.v) / (v2.v - v1.v);
    };

    for (int y = v0.v; y <= v2.v; y++) {
        // Intersect triangle sides with scanline
        int xa = interpolate(y, v0, v2); // Intersection with line v0-v2
        int xb = (y < v1.v) ? interpolate(y, v0, v1) : interpolate(y, v1, v2); // Depending on current half

        if (xa > xb) std::swap(xa, xb);

        // Draw horizontal line
        for (int x = xa; x <= xb; x++) {
            image.set(x, y, color);
        }
    }
}
```

### 第四關：包圍盒逐點掃描

介紹另一個非常有名的方法，包圍盒掃描方法。將需要光柵化的三角形框上一個矩形的包圍盒子內，在這個包圍盒子內逐個像素判斷該像素是否在三角形內。如果在三角形內，則繪製對應的像素；如果在三角形外，則略過。僞代碼如下：

```c++
triangle(vec2 points[3]) { 
    vec2 bbox[2] = find_bounding_box(points); 
    for (each pixel in the bounding box) { 
        if (inside(points, pixel)) { 
            put_pixel(pixel); 
        } 
    } 
}
```

想要實現這個方法，主要需要解決兩個問題：找到包圍盒、判斷某個像素點是否在三角形內。

第一個問題很好解決，找到三角形的三個點中最小和最大的兩個分量兩兩組合。

第二個問題似乎有些棘手。我們需要學習什麼是重心座標 （[barycentric coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system) ）。

### 第五關：重心座標

利用重心座標，可以判斷給定某個點與三角形之間的位置關係。

給定一個三角形ABC和任意一個點P $(x,y)$ ，這個點的座標都可以用點ABC線性表示。不理解也無所謂，簡單理解就是一個點P和三角形三點的關係可以用三個數字來表示，像下面公式這樣：
$$
P = (1-u-v)A+uB+vC
$$
我們把上面的式子解開，得到關於 $\overrightarrow{AB},\overrightarrow{AC}和\overrightarrow{AP}$的關係：
$$
P=A+u \overrightarrow{A B}+v \overrightarrow{A C}
$$
然後將點P挪到同一邊，得到下面的式子：
$$
u \overrightarrow{A B}+v \overrightarrow{A C}+\overrightarrow{P A}=\overrightarrow{0}
$$
然後將上面的向量分爲x分量與y分量，寫成兩個等式。接下來用矩陣表示他們：
$$
\left\{\begin{aligned}
{\left[\begin{array}{lll}
u & v & 1
\end{array}\right]\left[\begin{array}{l}
\overrightarrow{A B}_x \\
\overrightarrow{A C}_x \\
\overrightarrow{P A}_x
\end{array}\right]=0 } \\
{\left[\begin{array}{lll}
u & v & 1
\end{array}\right]\left[\begin{array}{l}
\overrightarrow{A B}_y \\
\overrightarrow{A C}_y \\
\overrightarrow{P A}_y
\end{array}\right]=0 }
\end{aligned}\right.
$$
兩個向量點積是0，說明兩個向量垂直。右邊這倆向量都與 $[u v 1]$ ，說明他們的叉積就是$k[u v 1]$ ，因此輕輕鬆鬆解出uv。

梳理一下，當務之急是判斷給定的一個點與一個三角形的關係。直接給出結論，如果點在三角形內部，則這三個係數都屬於（0,1）之間。直接給出光柵化一個三角形的代碼：

```c++
Vec3f barycentric(Vec2i v0, Vec2i v1, Vec2i v2, Vec2i pixel){
    // v0, v1, v2 correspond to ABC
    Vec3f u = Vec3f(v1.x-v0.x,// AB_x
                    v2.x-v0.x,// AC_x
                    v0.x-pixel.x)// PA_x
              ^
              Vec3f(v1.y-v0.y,
                    v2.y-v0.y,
                    v0.y-pixel.y);
    if (std::abs(u.z)<1) return Vec3f(-1,1,1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z);
}
// 重心座標的方法 - 光柵化三角形
void triangleRaster(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color){
    // Find The Bounding Box
    Vec2i* pts[] = {&v0, &v1, &v2};// Pack
    Vec2i boundingBoxMin(image.get_width()-1,  image.get_height()-1);
    Vec2i boundingBoxMax(0, 0);
    Vec2i clamp(image.get_width()-1, image.get_height()-1);
    for (int i=0; i<3; i++) {
        boundingBoxMin.x = std::max(0, std::min(boundingBoxMin.x, pts[i]->x));
        boundingBoxMin.y = std::max(0, std::min(boundingBoxMin.y, pts[i]->y));

        boundingBoxMax.x = std::min(clamp.x, std::max(boundingBoxMax.x, pts[i]->x));
        boundingBoxMax.y = std::min(clamp.y, std::max(boundingBoxMax.y, pts[i]->y));
    }

    // For Loop To Iterate Over All Pixels Within The Bounding Box
    Vec2i pixel;
    for (pixel.x = boundingBoxMin.x; pixel.x <= boundingBoxMax.x; pixel.x++) {
        for (pixel.y = boundingBoxMin.y; pixel.y <= boundingBoxMax.y; pixel.y++) {
            Vec3f bc = barycentric(v0, v1, v2, pixel);
            if (bc.x<0 || bc.y<0 || bc.z<0 ) continue;
            image.set(pixel.x, pixel.y, color);
        }
    }
}
```

barycentric()函數可能比較難理解，可以暫時拋棄研究其數學原理。並且上面這段代碼是經過優化的，如果希望瞭解其原理可以看我這一篇文章：[鏈接🔗](https://remoooo.com/cg/835.html)。

```c++
const int screenWidth  = 250;
const int screenHeight = 250;
...
triangleRaster(Vec2i(10,10), Vec2i(100, 30), Vec2i(190, 160),image,red);
```

![image-20230904005910991](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904005910991.png)

你可以在下面的鏈接中找到當前項目的代碼：[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/2.1.1_barycentric_coordinates)。

## 2.2 平面着色Flat shading render

在「1.2 三維畫線」中繪製了模型的線框，即空三角形模型。在「2.1 三角形光柵化」中，介紹了兩種方法繪製一個“實心”的三角形。現在，我們將使用“平面着色”來渲染小人模型，其中平面着色使用隨機的RGB數值。

### 第一關：回顧

首先將加載模型的相關代碼準備好：

```c++
#include <vector>
#include <cmath>
#include "tgaimage.h"
#include "geometry.h"
#include "model.h"

...
Model *model = NULL;
const int screenWidth  = 800;
const int screenHeight = 800;
...
    
// 光柵化三角形的代碼
void triangleRaster(Vec2i v0, Vec2i v1, Vec2i v2, TGAImage &image, TGAColor color){
	...
}

int main(int argc, char** argv) {
    const float halfWidth = screenWidth / 2.0f;
    const float halfHeight = screenHeight / 2.0f;
    TGAImage image(screenWidth, screenHeight, TGAImage::RGB);
    model = new Model("../object/african_head.obj");

	...// 在此處編寫接下來的代碼

    image.flip_vertically();
    image.write_tga_file("output.tga");
    delete model;
    return 0;
}
```

### 第二關：繪製隨機的顏色

下面是遍歷獲得模型的每一個需要繪製的三角形的代碼：

```c++
for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
	...
}
```

當我們獲得了所有的面，在每一趟遍歷中，將`face`的三個點取出來並轉換到屏幕座標上，最後傳給三角形光柵化函數：

```c++
for (int j=0; j<3; j++) {
    Vec3f world_coords = model->vert(face[j]); 
    screen_coords[j] = Vec2i((world_coords.x+1.)*width/2., (world_coords.y+1.)*height/2.); 
}
triangleRaster(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(rand()%255, rand()%255, rand()%255, 255));
```

![image-20230904134928856](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904134928856.png)

### 第三關：根據光線傳播繪製顏色

剛纔的隨機顏色遠遠滿足不了我們，現在我們根據光線與三角形的法線方向繪製不同的灰度。什麼意思呢？看下面這張圖，當物體表面的法線方向與光線方向垂直，物體接受到了最多的光；隨着法線與光線方向的夾角越來越大，收到光的照射也會越來越少。當法線與光線方向垂直的時候，表面就接收不到光線了。

![image-20230904135449781](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904135449781.png)

將這個特性添加到光柵化渲染器中。

```c++
Vec3f light_dir(0,0,-1); // define light_dir

for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
    Vec2i screen_coords[3]; 
    Vec3f world_coords[3]; 
    for (int j=0; j<3; j++) { 
        Vec3f v = model->vert(face[j]); 
        screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.); 
        world_coords[j]  = v; 
    } 
    Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]); 
    n.normalize(); 
    float intensity = n*light_dir; 
    if (intensity>0) { 
        triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(intensity*255, intensity*255, intensity*255, 255)); 
    } 
}
```

上面代碼需要注意的點：

- 三角形法線`n`的計算
- 判斷點積正負

`intensity`小於等於0的意思是這個面（三角形）背對着光線，攝像機肯定看不到，不需要繪製。

![image-20230904141708453](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904141708453.png)

注意到嘴巴的地方有些問題，本應在嘴脣後面的嘴巴內部區域（像口腔這樣的空腔）卻被畫在嘴脣的上方或前面。這表明我們對不可見三角形的處理方式不夠精確或不夠規範。“dirty clipping”方法只適用於凸形狀。對於凹形狀或其他複雜的形狀，該方法可能會導致錯誤。在下一章節中我們使用 z-buffer 解決這個瑕疵（渲染錯誤）。

這裏給出當前步驟的代碼[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/2.2_Flat_shading_render)。

## 3.1 表面剔除

上一章的末尾我們發現嘴巴部分的渲染出現了錯誤。本章先介紹畫家算法（Painters' Algorithm），隨後引出 Z-Buffer ，插值計算出需渲染的像素的深度值。

### 第一關：畫家算法（Painters' Algorithm）

這個算法很直接，將物體按其到觀察者的距離排序，然後從遠到近的順序繪製，這樣近處的物體自然會覆蓋掉遠處的物體。

但是仔細想就會發現一個問題，當物體相互阻擋時算法就會出錯。也就是說，畫家算法無法處理相互重疊的多邊形。

![image-20230904144410471](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904144410471.png)

### 第二關：瞭解z-buffer

如果畫家算法行不通，應該怎麼解決物體相互重疊的問題呢？我們初始化一張表，長寬與屏幕像素匹配，且每個像素大小初始化爲無限遠。每一個像素存儲一個深度值。當要渲染一個三角形的一個像素時，先比較當前欲渲染的像素位置與表中對應的深度值，如果當前欲渲染的像素深度比較淺，說明欲渲染的像素更靠近屏幕，因此渲染。

而這張表，我們稱之爲：Z-Buffer。

### 第三關：創建Z-Buffer

理論上說創建的這個 Z-Buffer 是一個二維的數組，例如：

```c++
float **zbuffer = new float*[screenWidth];
for (int i = 0; i < screenWidth; i++) {
    zbuffer[i] = new float[screenHeight];
}
...
// 釋放內存
for (int i = 0; i < screenWidth; i++) {
    delete[] zbuffer[i];
}
delete[] zbuffer;
```

但是，我認爲這太醜陋了，不符合我的審美。我的做法是將二維數組打包變成一個一維的數組：

```c++
int *zBuffer = new int[screenWidth*screenHeight];
```

最基本的數據結構，取用的時候只需要：

```c++
int idx = x + y*screenWidth;
int x = idx % screenWidth;
int y = idx / screenWidth;
```

初始化zBuffer可以用一行代碼解決，將其全部設置爲負無窮：

```c++
for (int i=screenWidth*screenHeight; i--; zBuffer[i] = -std::numeric_limits<float>::max());
```

### 第四關：整理當前代碼

要給當前的`triangleRaster()`函數新增 Z-Buffer 功能。

我們給`pixel`增加一個維度用於存儲深度值。另外，由於深度是float類型，如果沿用之前的函數可能會出現問題，原因是之前傳入的頂點都是經過取捨得到的整數且不包含深度信息。而且需要注意整數座標下的深度值往往不等於取捨之前的深度值，這個精度的損失帶來的問題是在複雜精細且深度值波動很大的位置會出現渲染錯誤。但是目前可以直接忽略，等到後面進行超採樣、抗鋸齒或者其他需要考慮像素內部細節的技術時再展開講解。

因此，爲了後期拓展的方便，我們將之前涉及pixel的Vec2i代碼換爲Vec3f類型，並且每一個點都增加一個維度用於存儲深度值。

```c++
Vec3f barycentric(Vec3f A, Vec3f B, Vec3f C, Vec3f P) {
    Vec3f s[2];
    for (int i=2; i--; ) {
        s[i][0] = C[i]-A[i];
        s[i][1] = B[i]-A[i];
        s[i][2] = A[i]-P[i];
    }
    Vec3f u = cross(s[0], s[1]);
    if (std::abs(u[2])>1e-2)
        return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z);
    return Vec3f(-1,1,1);
}
// 重心座標的方法 - 光柵化三角形
void triangleRaster(Vec3f v0, Vec3f v1, Vec3f v2, float *zBuffer, TGAImage &image, TGAColor color){
    Vec3f* pts[] = {&v0, &v1, &v2};// Pack
    // Find The Bounding Box
    Vec2f boundingBoxMin( std::numeric_limits<float>::max(),  std::numeric_limits<float>::max());
    Vec2f boundingBoxMax(-std::numeric_limits<float>::max(), -std::numeric_limits<float>::max());
    Vec2f clamp(image.get_width()-1, image.get_height()-1);
    for (int i=0; i<3; i++) {
        boundingBoxMin.x = std::max(0.f, std::min(boundingBoxMin.x, pts[i]->x));
        boundingBoxMin.y = std::max(0.f, std::min(boundingBoxMin.y, pts[i]->y));
        boundingBoxMax.x = std::min(clamp.x, std::max(boundingBoxMax.x, pts[i]->x));
        boundingBoxMax.y = std::min(clamp.y, std::max(boundingBoxMax.y, pts[i]->y));
    }

    // For Loop To Iterate Over All Pixels Within The Bounding Box
    Vec3f pixel;// 將深度值打包到pixel的z分量上
    for (pixel.x = boundingBoxMin.x; pixel.x <= boundingBoxMax.x; pixel.x++) {
        for (pixel.y = boundingBoxMin.y; pixel.y <= boundingBoxMax.y; pixel.y++) {
            Vec3f bc = barycentric(v0, v1, v2, pixel);// Screen Space
            if (bc.x<0 || bc.y<0 || bc.z<0 ) continue;
            // HIGHLIGHT: Finished The Z-Buffer
            //image.set(pixel.x, pixel.y, color);
            pixel.z = 0;
            pixel.z = bc.x*v0.z+bc.y+v1.z+bc.z+v2.z;// 通過重心座標插值計算當前Shading Point的深度值
            if(zBuffer[int(pixel.x+pixel.y*screenWidth)]<pixel.z) {
                zBuffer[int(pixel.x + pixel.y * screenWidth)] = pixel.z;
                image.set(pixel.x, pixel.y,color);
            }
        }
    }
}
```

將世界座標轉化到屏幕座標的函數打包：

```c++
Vec3f world2screen(Vec3f v) {
    return Vec3f(int((v.x+1.)*width/2.+.5), int((v.y+1.)*height/2.+.5), v.z);
}
```

另外，對tgaimage、model和geometry做了一些修改，主要是優化了一些細節。具體項目請查看當前項目分支[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/3.1_Z-buffer)。

![image-20230904191612606](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230904191612606.png)

## 3.2 上貼圖

啥是貼圖呢？就是類似這種奇奇怪怪的圖片。

![image-20230905174124334](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230905174124334.png)

目前我們已經完成了三角形的重心座標插值得出了三角形內某點的深度值。接下來我們還可以用插值操作計算對應的紋理座標。

本章基於「3.1 表面剔除」最後的項目完善，本章主要是c++ STL相關操作。

### 第一關：思路

請首先下載「3.1 表面剔除」最後的項目[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/3.1_Z-buffer)。

首先從硬盤中加載紋理貼圖，然後傳到三角形頂點處，通過對應的紋理座標從texture獲取顏色，最後插值得到各個像素的顏色。

另外，項目框架的代辦清單：

1. 增加model模塊中對vt標籤的解析
2. 完善model模塊中對f標籤的解析，具體是獲取紋理座標索引
3. 完善geometry模塊的操作符，具體是實現Vec<Dom, f>與float相乘等操作

### 第二關：加載紋理文件

從硬盤中加載紋理texture，用TGAImage存儲。

```c++
TGAImage texture;
if(texture.read_tga_file("../object/african_head_diffuse.tga")){
    std::cout << "Image successfully loaded!" << std::endl;
    // 可以做一些圖像處理
} else {
    std::cerr << "Error loading the image." << std::endl;
}
```

### 第三關：獲取紋理座標

在 model.h 中，在class Model上方創建一個Face結構體用於存儲解析後obj中的f標籤。f標籤有三個值，這裏只存儲前兩個。f標籤的三個值分別是頂點索引/紋理索引/法線索引，等後面用到了法線座標再拓展即可。

```c++
struct Face {
    std::vector<int> vertexIndices;
    std::vector<int> texcoordIndices;
    ...
};
```

然後將model的模版私有屬性：

```c++
std::vector< std::vector<int> > faces_;
```

改爲：

```c++
std::vector<Face> faces_;
```

同時也修改 model.cpp 下獲取 face 的函數：

```c++
Face Model::face(int idx) {
    return faces_[idx];
}
```

實際解析時的函數：

```c++
else if (!line.compare(0, 2, "f ")) {
//            std::vector<int> f;
//            int itrash, idx;
//            iss >> trash;
//            while (iss >> idx >> trash >> itrash >> trash >> itrash) {
//                idx--; // in wavefront obj all indices start at 1, not zero
//                f.push_back(idx);
//            }
//            faces_.push_back(f);
            Face face;
            int itrash, idx, texIdx;
            iss >> trash;
            while (iss >> idx >> trash >> texIdx >> trash >> itrash) {
                idx--; // in wavefront obj all indices start at 1, not zero
                texIdx--; // similarly for texture indices
                face.vertexIndices.push_back(idx);
                face.texcoordIndices.push_back(texIdx);
            }
            faces_.push_back(face);
        }
```

接下來解析紋理座標索引texcoords_。

```c++
// model.h
...
class Model {
private:
	...
    std::vector<Vec2f> texcoords_;
public:
	...
    Vec2f& getTexCoord(int index);
};
...
```

```c++
// model.cpp
...
Model::Model(const char *filename) : verts_(), faces_(), texcoords_(){
    ...
        else if (!line.compare(0, 3, "vt ")) {
            iss >> trash >> trash;
            Vec2f tc;
            for (int i = 0; i < 2; i++) iss >> tc[i];
            texcoords_.push_back(tc);
        }
    ...
}
...    
Vec2f& Model::getTexCoord(int index) {
    return texcoords_[index];
}
```

最後就可以通過對應的索引得到紋理座標了。

```c++
tex_coords[j] = model->getTexCoord(face.texcoordIndices[j]);
```

### 第四關：通過紋理座標uv獲取對應顏色

獲得了紋理座標後就可以用texture.get(x_pos, y_pos)獲取圖片（貼圖/紋理）的對應像素。注意最後TGAColor使用的是BGRA通道，而不是RGBA通道。

```c++
TGAColor getTextureColor(TGAImage &texture, float u, float v) {
    // 紋理座標限制在(0, 1)
    u = std::max(0.0f, std::min(1.0f, u));
    v = std::max(0.0f, std::min(1.0f, v));
    // 將u, v座標乘以紋理的寬度和高度，以獲取紋理中的像素位置
    int x = u * texture.get_width();
    int y = v * texture.get_height();
    // 從紋理中獲取顏色
    TGAColor color = texture.get(x, y);
    // tga使用的是BGRA通道
    return TGAColor(color[2],color[1],color[0], 255);
}
```

### 第五關：在光柵化三角形函數中增加貼貼圖的功能

增加了四個傳參，分別是三個三角形的紋理座標與紋理。實現細節直接看代碼比較直接。

```c++
// 帶貼圖 - 光柵化三角形
void triangleRasterWithTexture(Vec3f v0, Vec3f v1, Vec3f v2,
                               Vec2f vt0, Vec2f vt1, Vec2f vt2,// 紋理貼圖
                               float *zBuffer, TGAImage &image,
                               TGAImage &texture){
	...
    // Find The Bounding Box
	...
        
    // For Loop To Iterate Over All Pixels Within The Bounding Box
    Vec3f pixel;// 將深度值打包到pixel的z分量上
    for (pixel.x = boundingBoxMin.x; pixel.x <= boundingBoxMax.x; pixel.x++) {
        for (pixel.y = boundingBoxMin.y; pixel.y <= boundingBoxMax.y; pixel.y++) {
            Vec3f bc = barycentric(v0, v1, v2, pixel);// Screen Space
            if (bc.x<0 || bc.y<0 || bc.z<0 ) continue;
            // HIGHLIGHT: Finished The Z-Buffer
            pixel.z = 0;
            pixel.z = bc.x*v0.z+bc.y+v1.z+bc.z*v2.z;
            Vec2f uv = bc.x*vt0+bc.y*vt1+bc.z*vt2;
            if(zBuffer[int(pixel.x+pixel.y*screenWidth)]<pixel.z) {
                zBuffer[int(pixel.x + pixel.y * screenWidth)] = pixel.z;
                image.set(pixel.x, pixel.y,getTextureColor(texture, uv.x, 1-uv.y));
            }
        }
    }
}
```

在上面的代碼中，你可能會發現乘號竟然報錯了，這個問題在下一關馬上得到解決。最終在 main() 函數中這樣調用：

```c++
// main.cpp
...
for (int i=0; i<model->nfaces(); i++) {
    Face face = model->face(i);
    Vec3f screen_coords[3], world_coords[3];
    Vec2f tex_coords[3];
    for (int j=0; j<3; j++) {
        world_coords[j]  = model->vert(face.vertexIndices[j]);
        screen_coords[j] = world2screen(world_coords[j]);
        tex_coords[j] = model->getTexCoord(face.texcoordIndices[j]);
    }
    triangleRasterWithTexture(screen_coords[0], screen_coords[1], screen_coords[2],
                              tex_coords[0],tex_coords[1],tex_coords[2],
                              zBuffer, image, texture);
}
...
```

![image-20230905084358296](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230905084358296.png)

### 第六關：爲模板函數添加更多重載符號操作

在寫紋理座標的時候，我們會用到一些操作比如說 Vec2i 類型與 float 浮點數相乘和相除。將下面的代碼添加到 geometry.h 的中間部分：

```c++
...
    
template <typename T> vec<3,T> cross(vec<3,T> v1, vec<3,T> v2) {
    return vec<3,T>(v1.y*v2.z - v1.z*v2.y, v1.z*v2.x - v1.x*v2.z, v1.x*v2.y - v1.y*v2.x);
}

// -------------添加內容-------------
template<size_t DIM, typename T> vec<DIM, T> operator*(const T& scalar, const vec<DIM, T>& v) {
    vec<DIM, T> result;
    for (size_t i = 0; i < DIM; i++) {
        result[i] = scalar * v[i];
    }
    return result;
}

template<size_t DIM, typename T> vec<DIM, T> operator*(const vec<DIM, T>& v, const T& scalar) {
    vec<DIM, T> result;
    for (size_t i = 0; i < DIM; i++) {
        result[i] = v[i] * scalar;
    }
    return result;
}

template<size_t DIM, typename T> vec<DIM, T> operator/(const vec<DIM, T>& v, const T& scalar) {
    vec<DIM, T> result;
    for (size_t i = 0; i < DIM; i++) {
        result[i] = v[i] / scalar;
    }
    return result;
}

// -------------添加內容結束-------------

template <size_t DIM, typename T> std::ostream& operator<<(std::ostream& out, vec<DIM,T>& v) {
    for(unsigned int i=0; i<DIM; i++) {
        out << v[i] << " " ;
    }
    return out ;
}

...
```

這樣就完全沒問題了，大功告成。當然你也可以在這個[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/3.2_Diffuse_texture)中找到完整的代碼。

## 4.1 透視視角

上文的內容全部都是正交視角下的渲染，這顯然算不上酷，因爲我們僅僅是將z軸“拍扁”了。這一章節的目標是學習繪製透視視角。

><img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/3aTJFQLzfWZuDOB.png" alt="image-20230409155021065" style="zoom: 33%;" />
>
>https://stackoverflow.com/questions/36573283/from-perspective-picture-to-orthographic-picture

### 第一關：線性變換

縮放可以表示爲：
$$
\operatorname{scale}\left(s_x, s_y\right)=\left[\begin{array}{cc}
s_x & 0 \\
0 & s_y
\end{array}\right] .
$$
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/pGTvkjmfCMlwRDs.png" alt="image-20230408154330557" style="zoom:50%;" />

拉伸可以表示爲：
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
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/UIO1DCYtucQTKbg.png" alt="image-20230408154937046" style="zoom:50%;" />

旋轉可以表示爲：
$$
\mathbf{R}_\theta=\left[\begin{array}{cc}
\cos \theta & -\sin \theta \\
\sin \theta & \cos \theta
\end{array}\right]
$$
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/MsPI1HRNmzxd6gS.png" alt="image-20230408155212728" style="zoom:50%;" />

### 第二關：齊次座標 Homogeneous coordinates

爲什麼要引入齊次座標呢？因爲想要表示一個二維變換的平移並不能僅僅使用一個2x2的矩陣。平移並不在這個二維矩陣的線性空間中。因此，我們拓展一個維度幫助我們表示平移。

在計算機圖形學中我們使用齊次座標（Homogeneous Coord）。比如說一個二維的$(x, y)$使用平移矩陣變換到$(x', y')$：
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
這樣，我們就可以通過 $t_x, t_y$ 做平移變換，簡直太聰明瞭。

在常規的笛卡爾座標中，很難從數學表示上區分一個點和一個向量，因爲它們都可能使用相同的形式如 vec2(x,y)。但在齊次座標中，通過最後一個座標值（這裏的z）可以明確區分它們。當z=0時，它是一個向量；當z≠0時，它是一個點。較爲數學一點的表示方法：
$$
k\left[\begin{array}{l}
x \\
y \\
1
\end{array}\right], k \neq 0
$$
上面公式中，無論 $k$ 取多少，都表示同一個點。再舉個例子：
$$
\left[\begin{array}{c}
x \\
y \\
1
\end{array}\right] \equiv\left[\begin{array}{c}
2 x \\
2 y \\
2
\end{array}\right] \equiv\left[\begin{array}{c}
514 x \\
514 y \\
514
\end{array}\right] \equiv\left[\begin{array}{c}
114 x \\
114 y \\
114
\end{array}\right]
$$
齊次座標是一個大大的好啊，當你進行數學操作時，結果的類型（向量或點）是明確的：

- **向量 + 向量 = 向量**：兩個向量相加的結果仍然是一個向量。
- **向量 - 向量 = 向量**：兩個向量相減的結果仍然是一個向量。
- **點 + 向量 = 點**：一個點和一個向量相加的結果是一個新的點。
- **點 + 點 = ？？**：兩個點座標的中點。

這使得數學操作更加直觀和有意義。

> 一段來自屏幕外的聲音🔊：齊次座標最下面那行有啥用？？這個問題非常關鍵。

$$
\left[\begin{array}{lll}
a & b & m \\
c & d & n \\
p & q & 1
\end{array}\right]
$$

家喻戶曉的，$\left[\begin{array}{ll}
a & b \\
c & d 
\end{array}\right]$可以實現縮放，$\left[\begin{array}{l}
m \\
n \\
1
\end{array}\right]$ 可以實現平移。

但是，這個 $\left[\begin{array}{ll}
p & q 
\end{array}\right]$ 能幹嘛？

變換矩陣不做其他線性變換，僅僅將pq隨便設爲一個數：
$$
\left[\begin{array}{lll}
1 & 0 & 0 \\
0 & 1 & 0 \\
2 & 0 & 1
\end{array}\right]\left[\begin{array}{l}
x \\
y \\
1
\end{array}\right]=\left[\begin{array}{c}
x \\
y \\
2 x+1
\end{array}\right] \equiv\left[\begin{array}{c}
\frac{x}{2 x+1} \\
\frac{y}{2 x+1} \\
1
\end{array}\right]
$$
我們發現，這個變換有點奇怪。隨着(x,y)越來越大，這個“縮放因子”就會越來越小。

> 有沒有一種可能，這個就是近大遠小？

![image-20230905160328502](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230905160328502.png)

沒錯，這就是一種透視的現象。至此，上面齊次座標矩陣的最後一朵烏雲已經攻破。 $\left[\begin{array}{ll}
p & q 
\end{array}\right]$ 就是用來做透視變換的。

隨着最後一朵烏雲散去，必然會迎來更多的烏雲。新的烏雲，名字叫做三維。

### 第三關：三維世界

上文所述都是二維下的，現在進入三維的世界。三維的齊次座標自然就是用四維的矩陣表示。

縮放：
$$
\mathbf{S}\left(s_x, s_y, s_z\right)=\left(\begin{array}{cccc}
s_x & 0 & 0 & 0 \\
0 & s_y & 0 & 0 \\
0 & 0 & s_z & 0 \\
0 & 0 & 0 & 1
\end{array}\right)\\
$$
平移：
$$
\mathbf{T}\left(t_x, t_y, t_z\right)=\left(\begin{array}{cccc}
1 & 0 & 0 & t_x \\
0 & 1 & 0 & t_y \\
0 & 0 & 1 & t_z \\
0 & 0 & 0 & 1
\end{array}\right)
$$
繞x,z,y軸旋轉：
$$
\left\{\begin{array}{rl}
\mathbf{R}_x(\alpha)=\left(\begin{array}{cccc}
1 & 0 & 0 & 0 \\
0 & \cos \alpha & -\sin \alpha & 0 \\
0 & \sin \alpha & \cos \alpha & 0 \\
0 & 0 & 0 & 1
\end{array}\right)\\
\mathbf{R}_z(\alpha)=\left(\begin{array}{cccc}
\cos \alpha & -\sin \alpha & 0 & 0 \\
\sin \alpha & \cos \alpha & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{array}\right)\\
\mathbf{R}_y(\alpha)=\left(\begin{array}{cccc}
\cos \alpha & 0 & \sin \alpha & 0 \\
0 & 1 & 0 & 0 \\
-\sin \alpha & 0 & \cos \alpha & 0 \\
0 & 0 & 0 & 1
\end{array}\right)
\end{array}\right.
$$
透視：
$$
\mathbf{P}\left(r\right)=\left(\begin{array}{cccc}
1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & r & 1
\end{array}\right)\\
$$
爲什麼只有z方向上纔有r？因爲我們默認攝像機擺在z軸，物體隨着z軸透視縮放的。

現在，將一個三維座標通過透視縮放，得到：
$$
\left[\begin{array}{llll}
1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & r & 1
\end{array}\right]\left[\begin{array}{l}
x \\
y \\
z \\
1
\end{array}\right]=\left[\begin{array}{c}
x \\
y \\
z \\
1+z r
\end{array}\right] \equiv\left[\begin{array}{c}
\frac{x}{1+z r} \\
\frac{y}{1+z r} \\
\frac{z}{1+z r} \\
1
\end{array}\right]
$$
![image-20230905173407444](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230905173407444.png)

上圖中，橫軸向左是z的正方向，縱軸向上是y的正方向。

根據相似三角形法則，y1/By=(c-z1)/c，最後得到：
$$
r=- \frac{1}{c}
$$
因此得到透視矩陣：
$$
\left[\begin{array}{cccc}
1 & 0 & 0 & 0 \\
0 & 1 & 0 & 0 \\
0 & 0 & 1 & 0 \\
0 & 0 & -\frac{1}{c} & 1
\end{array}\right]
$$
大家可能發現，如果有接觸過圖形學的朋友們可能會對之前的學習產生懷疑。爲什麼這裏頂點變換的透視變換矩陣和其他教材都不一樣呢？比方說虎書上的透視矩陣是這樣的：
$$
\left[\begin{array}{cccc}
\frac{1}{\text { aspect } \times \tan \left(\frac{f o v y}{2}\right)} & 0 & 0 & 0 \\
0 & \frac{1}{\tan \left(\frac{\text { fovy }}{2}\right)} & 0 & 0 \\
0 & 0 & -\frac{\text { far }+ \text { near }}{\text { far }- \text { near }} & -\frac{2 \times \text { far } \times \text { near }}{\text { far }- \text { near }} \\
0 & 0 & -1 & 0
\end{array}\right]
$$

- fovy：垂直視場角, 通常表示爲度數。 

- aspect : 寬高比, 即視口寬度除以視口高度。 

- near : 近裁剪面的距離。 

- far: 遠裁剪面的距離。

所以我們剛纔推導出來的矩陣並不是常見的透視投影矩陣，但是他確實表達了投影的思想，因此我們暫時用着。

> 來自屏幕外的聲音🔊：停停停，理論說了這麼多，能不能搞點實踐的！

### 第四關：具體代碼實現

在上一關中，我們得到了不那麼正規但是能用的透視矩陣，現在要做的就是將世界座標的頂點轉換到齊次座標，然後乘上透視矩陣、視口矩陣。視口矩陣其實就是用一個簡潔的矩陣把下面歸一化設備座標 NDC [-1,1]轉換到了屏幕空間[0,width]。看下面這一段代碼就是那個被ViewPort矩陣淘汰的傢伙：

```c++
Vec3f world2screen(Vec3f v) {
    return Vec3f(int((v.x+1.)*screenWidth/2.+.5), int((v.y+1.)*screenHeight/2.+.5), v.z);
}
```

接下來，把頂點座標乘上我們下面兩個矩陣（順序要注意）：

```c++
//初始化透視矩陣
Matrix Projection = Matrix::identity(4);
//初始化視角矩陣
Matrix ViewPort   = viewport(width/2, height/2, width/2, height/2);
//投影矩陣[3][2]=-1/c，c爲相機z座標
Projection[3][2] = -1.f/camera.z;
...
screen_coords[j] = m2v(ViewPort * Projection * v2m((world_coords[j])));
```

>一段來自屏幕外的聲音🔊：等等等等，v2m和m2v是什麼？viewport()具體實現方法是什麼？

v2m是將向量變成矩陣（齊次座標），m2v反之。

```c++
Vec3f m2v(Matrix m){
    return Vec3f(m[0][0]/m[3][0], m[1][0]/m[3][0], m[2][0]/m[3][0]);
}

Matrix v2m(Vec3f v) {
    Matrix m(4, 1);
    m[0][0] = v.x;
    m[1][0] = v.y;
    m[2][0] = v.z;
    m[3][0] = 1.f;
    return m;
}

Matrix viewport(int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;

    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m;
}
```

然後還需要完善geometry的模塊，在geometry.h中添加如下代碼：

```c++
//////////////////////////////////////////////////////////////////////////////////////////////

const int DEFAULT_ALLOC=4;

class Matrix {
    std::vector<std::vector<float> > m;
    int rows, cols;
public:
    Matrix(int r=DEFAULT_ALLOC, int c=DEFAULT_ALLOC);
    inline int nrows();
    inline int ncols();

    static Matrix identity(int dimensions);
    std::vector<float>& operator[](const int i);
    Matrix operator*(const Matrix& a);
    Matrix transpose();
    Matrix inverse();

    friend std::ostream& operator<<(std::ostream& s, Matrix& m);
};

/////////////////////////////////////////////////////////////////////////////////////////////
...
// typedef mat<4,4,float> Matrix; 
```

然後添加文件 geometry.cpp 

```c++
//
// Created by remoooo on 2023/9/6.
//

#include <vector>
#include <cassert>
#include <cmath>
#include <iostream>
#include "geometry.h"

Matrix::Matrix(int r, int c) : m(std::vector<std::vector<float> >(r, std::vector<float>(c, 0.f))), rows(r), cols(c) { }

int Matrix::nrows() {
    return rows;
}

int Matrix::ncols() {
    return cols;
}

Matrix Matrix::identity(int dimensions) {
    Matrix E(dimensions, dimensions);
    for (int i=0; i<dimensions; i++) {
        for (int j=0; j<dimensions; j++) {
            E[i][j] = (i==j ? 1.f : 0.f);
        }
    }
    return E;
}

std::vector<float>& Matrix::operator[](const int i) {
    assert(i>=0 && i<rows);
    return m[i];
}

Matrix Matrix::operator*(const Matrix& a) {
    assert(cols == a.rows);
    Matrix result(rows, a.cols);
    for (int i=0; i<rows; i++) {
        for (int j=0; j<a.cols; j++) {
            result.m[i][j] = 0.f;
            for (int k=0; k<cols; k++) {
                result.m[i][j] += m[i][k]*a.m[k][j];
            }
        }
    }
    return result;
}

Matrix Matrix::transpose() {
    Matrix result(cols, rows);
    for(int i=0; i<rows; i++)
        for(int j=0; j<cols; j++)
            result[j][i] = m[i][j];
    return result;
}

Matrix Matrix::inverse() {
    assert(rows==cols);
    // augmenting the square matrix with the identity matrix of the same dimensions a => [ai]
    Matrix result(rows, cols*2);
    for(int i=0; i<rows; i++)
        for(int j=0; j<cols; j++)
            result[i][j] = m[i][j];
    for(int i=0; i<rows; i++)
        result[i][i+cols] = 1;
    // first pass
    for (int i=0; i<rows-1; i++) {
        // normalize the first row
        for(int j=result.cols-1; j>=0; j--)
            result[i][j] /= result[i][i];
        for (int k=i+1; k<rows; k++) {
            float coeff = result[k][i];
            for (int j=0; j<result.cols; j++) {
                result[k][j] -= result[i][j]*coeff;
            }
        }
    }
    // normalize the last row
    for(int j=result.cols-1; j>=rows-1; j--)
        result[rows-1][j] /= result[rows-1][rows-1];
    // second pass
    for (int i=rows-1; i>0; i--) {
        for (int k=i-1; k>=0; k--) {
            float coeff = result[k][i];
            for (int j=0; j<result.cols; j++) {
                result[k][j] -= result[i][j]*coeff;
            }
        }
    }
    // cut the identity matrix back
    Matrix truncate(rows, cols);
    for(int i=0; i<rows; i++)
        for(int j=0; j<cols; j++)
            truncate[i][j] = result[i][j+cols];
    return truncate;
}

std::ostream& operator<<(std::ostream& s, Matrix& m) {
    for (int i=0; i<m.nrows(); i++)  {
        for (int j=0; j<m.ncols(); j++) {
            s << m[i][j];
            if (j<m.ncols()-1) s << "\t";
        }
        s << "\n";
    }
    return s;
}
```

接下來，渲染器啓動！

![image-20230906112817473](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230906112817473.png)

> 來自甲方的聲音🔊：效果非常好，下次不要做了。

看得出來，畫面出現了一點問題。但是值得注意的是，頂點的位置已經基本正確了。但是貼圖出現了錯誤。

藉此機會，調整一下貼圖加載的邏輯。我們原先在main函數粗暴加載，現在我們將物體對應的貼圖當作model對象的一個屬性，自動讀取。在model.h中加入字段：

```c++
TGAImage diffusemap_;
```

構造函數就可以根據文件名字存入對應的貼圖了：

```c++
load_texture(filename, "_diffuse.tga", diffusemap_);
```

然後通過以下函數得到對應的uv座標：

```c++
Vec2i Model::uv(int iface, int nvert) {
    int idx = faces_[iface][nvert][1];
    return Vec2i(uv_[idx].x*diffusemap_.get_width(), uv_[idx].y*diffusemap_.get_height());
}
```

改動部分比較多直接閱讀項目吧，項目[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/4.1_MVP_matrix)在這裏，我們在「4.2 代碼分析」中詳細討論整個項目，力求搞懂每一行代碼與設計思路，尤其是C++ STL細節。下面是一個最終結果：

![image-20230906183206037](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230906183206037.png)

## 4.2 項目代碼分析

目前的代碼[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/4.1_MVP_matrix)有較大的改動，但是技術原理是不變的。本章節可以選擇性閱讀，也可以直接跳到「5.1 移動攝像機」。

項目結構：

```tree
├── object
│   ├── african_head.obj
│   ├── african_head_diffuse.tga
├── CMakeLists.txt
├── geometry.cpp
├── geometry.h
├── main.cpp
├── model.cpp
├── model.h
├── tgaimage.cpp
└── tgaimage.h
```

### 第一關：model類

> model.h

```c++
class Model {
private:
    std::vector<Vec3f> verts_; // 模型的頂點
    std::vector<std::vector<Vec3i> > faces_; // this Vec3i means vertex/uv/normal
    std::vector<Vec3f> norms_; // 存儲模型的法線
    std::vector<Vec2f> uv_; // 存儲模型的 UV 紋理座標
    TGAImage diffusemap_; // 模型的漫反射紋理圖像
    // load_texture() 在加載模型的時候會用到，用於加載紋理。
    void load_texture(std::string filename, const char *suffix, TGAImage &img);
public:
    Model(const char *filename); // 構造函數，從給定文件名加載模型
    ~Model(); // 析構函數，用於釋放模型所佔用的資源
    int nverts(); // 返回模型的頂點數量
    int nfaces(); // 返回模型的面數量
    Vec3f vert(int i); // 返回指定索引的頂點
    Vec2i uv(int iface, int nvert); // 返回指定面和指定頂點的 UV 座標
    TGAColor diffuse(Vec2i uv); // 根據給定的 UV 座標，從紋理圖中獲取顏色
    std::vector<int> face(int idx); // 返回指定索引的面信息（可能是頂點/紋理座標/法線的索引）
};
```

> Model.cpp

```c++
/*
 * 構造函數 Model::Model(const char *filename)
 * - 構造函數使用了初始化列表來對 verts_, faces_, norms_ 和 uv_ 進行初始化。
 *
 */
Model::Model(const char *filename) : verts_(), faces_(), norms_(), uv_() {
    std::ifstream in;
    in.open (filename, std::ifstream::in);
    if (in.fail()) return;
    std::string line;
    /* 循環讀取並解析每一行內容。
     * 根據行的開頭字符來決定行的類型（例如頂點、法線、紋理座標或面）。
     * 根據這些信息更新相應的成員變量。
     */
    while (!in.eof()) {
        std::getline(in, line);
        std::istringstream iss(line.c_str());
        char trash;
        if (!line.compare(0, 2, "v ")) {
            iss >> trash;
            Vec3f v;
            for (int i=0;i<3;i++) iss >> v[i];
            verts_.push_back(v);
        } else if (!line.compare(0, 3, "vn ")) {
            iss >> trash >> trash;
            Vec3f n;
            for (int i=0;i<3;i++) iss >> n[i];
            norms_.push_back(n);
        } else if (!line.compare(0, 3, "vt ")) {
            iss >> trash >> trash;
            Vec2f uv;
            for (int i=0;i<2;i++) iss >> uv[i];
            uv_.push_back(uv);
        }  else if (!line.compare(0, 2, "f ")) {
            std::vector<Vec3i> f;
            Vec3i tmp;
            iss >> trash;
            while (iss >> tmp[0] >> trash >> tmp[1] >> trash >> tmp[2]) {
                for (int i=0; i<3; i++) tmp[i]--; // in wavefront obj all indices start at 1, not zero
                f.push_back(tmp);
            }
            faces_.push_back(f);
        }
    }
    std::cerr << "# v# " << verts_.size() << " f# "  << faces_.size() << " vt# " << uv_.size() << " vn# " << norms_.size() << std::endl;
    load_texture(filename, "_diffuse.tga", diffusemap_);// 調用 load_texture 函數加載相應的紋理文件
}

Model::~Model() {
}

// 返回模型的頂點數量
int Model::nverts() {
    return (int)verts_.size();
}
// 返回模型的面數量。
int Model::nfaces() {
    return (int)faces_.size();
}

// 接收一個面的索引並返回這個面的所有頂點/紋理/法線座標
std::vector<int> Model::face(int idx) {
    std::vector<int> face;
    for (int i=0; i<(int)faces_[idx].size(); i++) face.push_back(faces_[idx][i][0]);
    return face;
}
// 返回指定索引的頂點。
Vec3f Model::vert(int i) {
    return verts_[i];
}
// 加載紋理。
void Model::load_texture(std::string filename, const char *suffix, TGAImage &img) {
    std::string texfile(filename);
    size_t dot = texfile.find_last_of(".");
    if (dot!=std::string::npos) {
        texfile = texfile.substr(0,dot) + std::string(suffix);
        std::cerr << "texture file " << texfile << " loading " << (img.read_tga_file(texfile.c_str()) ? "ok" : "failed") << std::endl;
        img.flip_vertically();
    }
}
// 返回給定 UV 座標的漫反射顏色。
TGAColor Model::diffuse(Vec2i uv) {
    return diffusemap_.get(uv.x, uv.y);
}
// 返回指定面和頂點的 UV 座標。
Vec2i Model::uv(int iface, int nvert) {
    int idx = faces_[iface][nvert][1];
    return Vec2i(uv_[idx].x*diffusemap_.get_width(), uv_[idx].y*diffusemap_.get_height());
}
```

### 第二關：geometry

> geometry.h中，分爲兩個部分：模版向量類，矩陣類

```c++
/* --向量類定義-- */
// t -> 任意類型的數據，比如說 int float double等等
template <class t> struct Vec2 {
    t x, y;// 創建了兩個t類型的數據成員
    Vec2<t>() : x(t()), y(t()) {} // 使用類型t的默認構造函數來初始化x和y。
    Vec2<t>(t _x, t _y) : x(_x), y(_y) {} // 接受兩個參數，初始化x,y
//    Vec2<t>(const Vec2<t> &v) : x(t()), y(t()) { *this = v; } // 模板類的拷貝構造函數
    Vec2<t>(const Vec2<t> &v) : x(v.x), y(v.y) {} // 我認爲上面的代碼不太好，改了一下
    Vec2<t> & operator =(const Vec2<t> &v) { // 重載了等號，改變符號左邊對象的數值
        if (this != &v) {
            x = v.x;
            y = v.y;
        }
        return *this;
    }
    Vec2<t> operator +(const Vec2<t> &V) const { return Vec2<t>(x+V.x, y+V.y); }
    Vec2<t> operator -(const Vec2<t> &V) const { return Vec2<t>(x-V.x, y-V.y); }
    Vec2<t> operator *(float f)          const { return Vec2<t>(x*f, y*f); }
    // 重載[]符號，這裏官方寫錯了，if(x<=0)是錯誤的。
    t& operator[](const int i) { if (i<=0) return x; else return y; }
    // 重載輸出流
    template <class > friend std::ostream& operator<<(std::ostream& s, Vec2<t>& v);
};

template <class t> struct Vec3 {
    t x, y, z;
    Vec3<t>() : x(t()), y(t()), z(t()) { }
    Vec3<t>(t _x, t _y, t _z) : x(_x), y(_y), z(_z) {}
    template <class u> Vec3<t>(const Vec3<u> &v);
    Vec3<t>(const Vec3<t> &v) : x(t()), y(t()), z(t()) { *this = v; }
    Vec3<t> & operator =(const Vec3<t> &v) {
        if (this != &v) {
            x = v.x;
            y = v.y;
            z = v.z;
        }
        return *this;
    }
    Vec3<t> operator ^(const Vec3<t> &v) const { return Vec3<t>(y*v.z-z*v.y, z*v.x-x*v.z, x*v.y-y*v.x); }
    Vec3<t> operator +(const Vec3<t> &v) const { return Vec3<t>(x+v.x, y+v.y, z+v.z); }
    Vec3<t> operator -(const Vec3<t> &v) const { return Vec3<t>(x-v.x, y-v.y, z-v.z); }
    Vec3<t> operator *(float f)          const { return Vec3<t>(x*f, y*f, z*f); }
    t       operator *(const Vec3<t> &v) const { return x*v.x + y*v.y + z*v.z; }
    float norm () const { return std::sqrt(x*x+y*y+z*z); }
    Vec3<t> & normalize(t l=1) { *this = (*this)*(l/norm()); return *this; }
    t& operator[](const int i) { if (i<=0) return x; else if (i==1) return y; else return z; }
    template <class > friend std::ostream& operator<<(std::ostream& s, Vec3<t>& v);
};

// 爲常用類型提供了類型別名
typedef Vec2<float> Vec2f;
typedef Vec2<int>   Vec2i;
typedef Vec3<float> Vec3f;
typedef Vec3<int>   Vec3i;

/* 特化構造函數 */
// Vec3<int> <- Vec3<float>。浮點數值 -> 整數值
template <> template <> Vec3<int>::Vec3(const Vec3<float> &v);
// Vec3<float> <- Vec3<int>。整數值 -> 浮點數值
template <> template <> Vec3<float>::Vec3(const Vec3<int> &v);

// 使得Vec2對象可以被輸出到輸出流
template <class t> std::ostream& operator<<(std::ostream& s, Vec2<t>& v) {
    s << "(" << v.x << ", " << v.y << ")\n";
    return s;
}
// 使得Vec3對象可以被輸出到輸出流
template <class t> std::ostream& operator<<(std::ostream& s, Vec3<t>& v) {
    s << "(" << v.x << ", " << v.y << ", " << v.z << ")\n";
    return s;
}

//////////////////////////////////////////////////////////////////////////////////////////////
/*
 * Matrix類代表了一個浮點數矩陣，其中的數據存儲在std::vector。
 *
 */

const int DEFAULT_ALLOC=4;

class Matrix {
    // 這是一個二維vector，它存儲了矩陣的所有元素。
    std::vector<std::vector<float> > m;
    int rows, cols;
public:
    // 構造函數 可以接受行數和列數作爲參數。如果沒有手動提供參數，則會使用DEFAULT_ALLOC（默認值爲4）來初始化。
    Matrix(int r=DEFAULT_ALLOC, int c=DEFAULT_ALLOC);
    // 這個inline語法比較適合用於短小且經常被調用的函數。比如 inline int add(int a, int b){return a+b;}
    inline int nrows();
    inline int ncols();

    static Matrix identity(int dimensions); // 返回給定維度的單位矩陣
    std::vector<float>& operator[](const int i); // 這是一個重載的[]操作符，使得你可以使用像mat[i]這樣的語法來直接訪問矩陣的行。
    Matrix operator*(const Matrix& a); // 重載*操作符，以支持矩陣乘法。
    Matrix transpose(); // 返回矩陣的轉置
    Matrix inverse(); // 返回矩陣的逆

    friend std::ostream& operator<<(std::ostream& s, Matrix& m); // 支持流傳輸
};

/////////////////////////////////////////////////////////////////////////////////////////////

```

> geometry.cpp 大同小異，不做筆記了

注意到代碼中用到了`remplate <> template <>`進行模版特化，這種用法我比較少見到，也看不懂。

假設現在有一個模版類`Vec2`，有一個函數模版`printType`：

```c++
template <typename T>
class TemplateClass {
public:
    template <typename U>
    void memberFunction() {
        std::cout << "General memberFunction\n";
    }
};
```

現在，要爲`Vec2<float>`特化`printType`方法，讓他在`int`的時候打印`int`的專屬信息。

```c++
template <>
class Vec2<float> {
public:
    float x, y;
    Vec2(float x = 0, float y = 0) : x(x), y(y) {}

    template <typename U>
    void printType();

    template <>
    void printType<int>() {
        std::cout << "Specialized Vec2<float> with int" << std::endl;
    }
};
```

測試：

```c++
int main() {
    Vec2<double> vecDouble;
    vecDouble.printType<int>();  // 輸出: Vec2<double> with type int

    Vec2<float> vecFloat;
    vecFloat.printType<int>();   // 輸出: Specialized Vec2<float> with int
    vecFloat.printType<double>();  // 輸出: Vec2<float> with type double
}
```

### 第三關：main

```c++
#include <vector>
#include <cmath>
#include <limits>
#include "tgaimage.h"
#include "model.h"
#include "geometry.h"

const int width  = 800;
const int height = 800;
const int depth  = 255;

Model *model = NULL;
int *zBuffer = NULL;
Vec3f light_dir(0,0,-1);
Vec3f camera(0,0,3);

// 4d-->3d
Vec3f m2v(Matrix m) {
    return Vec3f(m[0][0]/m[3][0], m[1][0]/m[3][0], m[2][0]/m[3][0]);
}

// 3d-->4d
Matrix v2m(Vec3f v) {
    Matrix m(4, 1);
    m[0][0] = v.x;
    m[1][0] = v.y;
    m[2][0] = v.z;
    m[3][0] = 1.f;
    return m;
}

// 視角矩陣
Matrix viewport(int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;

    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m;
}

Vec3f barycentric(Vec3i *pts, Vec3i P) {
    Vec3f u =
            Vec3f(pts[2].x-pts[0].x, pts[1].x-pts[0].x, pts[0].x-P.x)^
            Vec3f(pts[2].y-pts[0].y, pts[1].y-pts[0].y, pts[0].y-P.y)
    ;
    if (std::abs(u.z)<1) return Vec3f(-1,1,1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z);
}
void triangle(Vec3i t0, Vec3i t1, Vec3i t2,
              Vec2i uv0, Vec2i uv1, Vec2i uv2,
              TGAImage &image, float intensity, int *zbuffer) {
    // Compute bounding box of the triangle
    Vec3i bboxmin(image.get_width() - 1, image.get_height() - 1, 0);
    Vec3i bboxmax(0, 0, 0);
    Vec3i clamp(image.get_width() - 1, image.get_height() - 1, 0);
    bboxmin.x = std::max(0, std::min(bboxmin.x, std::min(t0.x, std::min(t1.x, t2.x))));
    bboxmin.y = std::max(0, std::min(bboxmin.y, std::min(t0.y, std::min(t1.y, t2.y))));
    bboxmax.x = std::min(clamp.x, std::max(t0.x, std::max(t1.x, t2.x)));
    bboxmax.y = std::min(clamp.y, std::max(t0.y, std::max(t1.y, t2.y)));
    Vec3i pts[3] = {t0, t1, t2};
    Vec3i P;
    for (P.x = bboxmin.x; P.x <= bboxmax.x; P.x++) {
        for (P.y = bboxmin.y; P.y <= bboxmax.y; P.y++) {
            Vec3f bc_screen = barycentric(pts, P);
            if (bc_screen.x < 0 || bc_screen.y < 0 || bc_screen.z < 0) continue;  // P is outside of triangle
            P.z = bc_screen.x * t0.z + bc_screen.y * t1.z + bc_screen.z * t2.z;
            Vec2i uvP;
            uvP.x = bc_screen.x * uv0.x + bc_screen.y * uv1.x + bc_screen.z * uv2.x;
            uvP.y = bc_screen.x * uv0.y + bc_screen.y * uv1.y + bc_screen.z * uv2.y;
            int idx = P.x + P.y * image.get_width();
            if (zbuffer[idx] < P.z) {
                zbuffer[idx] = P.z;
                TGAColor color = model->diffuse(uvP);
                image.set(P.x, P.y,
                          TGAColor(color.r * intensity, color.g * intensity, color.b * intensity));
            }
        }
    }
}

int main(int argc, char** argv) {
    model = new Model("../object/african_head.obj");
    zBuffer = new int[width*height];

    for (int i=width*height; i--; zBuffer[i] = -std::numeric_limits<float>::max());

    { // draw the model
        Matrix Projection = Matrix::identity(4);
        Matrix ViewPort   = viewport(width/8, height/8, width*3/4, height*3/4);
        Projection[3][2] = -1.f/camera.z;

        TGAImage image(width, height, TGAImage::RGB);
        for (int i=0; i<model->nfaces(); i++) {
            std::vector<int> face = model->face(i);
            Vec3i screen_coords[3];
            Vec3f world_coords[3];
            for (int j=0; j<3; j++) {
                Vec3f v = model->vert(face[j]);
                // 視角矩陣*投影矩陣*座標
                screen_coords[j] =  m2v(ViewPort*Projection*v2m(v));
                world_coords[j]  = v;
            }
            // 計算法向量並且標準化
            Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]).normalize();
            // 計算光照
            float intensity = n*light_dir;
            if (intensity>0) {
                Vec2i uv[3];
                for (int k=0; k<3; k++) {
                    uv[k] = model->uv(i, k);
                }
                // 繪製三角形
                triangle(screen_coords[0], screen_coords[1], screen_coords[2],
                         uv[0], uv[1], uv[2], image, intensity, zBuffer);
            }
        }

        image.flip_vertically();
        image.write_tga_file("output.tga");
    }

    { // 輸出zbuffer
        TGAImage zbimage(width, height, TGAImage::GRAYSCALE);
        for (int i=0; i<width; i++) {
            for (int j=0; j<height; j++) {
                zbimage.set(i, j, TGAColor(zBuffer[i+j*width], 1));
            }
        }
        zbimage.flip_vertically();
        zbimage.write_tga_file("zbuffer.tga");
    }
    delete model;
    delete [] zBuffer;
    return 0;
}
```

## 5.1 移動攝像機

能看到這個地方的朋友簡直太牛逼了，這一章就稍微有點簡單了。

在初中我們就學過，物體的運動是相對的。

攝像機向左移動，其實就是物體向右移動。我們可以讓攝像機保持不動，讓物體運動（平移）。

旋轉其實也是一樣的，假設攝像機繞着垂直的y軸順時針旋轉了45度，相當於物體逆時針旋轉了45度。

圖形學中我們知道，一個變換就代表了一個矩陣。並且在線性代數中我們知道，一個矩陣A乘上另一個矩陣B之後，想要變回原來的矩陣A，只需要再乘上矩陣B的逆。也就是說，要反變換，只需要乘上對應變換的逆即可。

### 第一關：定義攝像機

我們想，需要定義什麼量纔可以確保攝像機在同一時間拍攝的畫面是一模一樣的？（這裏只考慮攝像機的位置、方向，攝像機鏡頭、焦距、ISO等信息默認相等。）

很顯然可以想到的是攝像機的位置，我們將其定義爲`cameraPos`。

並且拍攝的方向也需要定義，我們就叫他`gazeDirection`。

大家應該都玩過Rainbow Six圍攻或者是PUBG吧，他們都有一個共同的特點：可以歪頭射擊。即使是在同一個攝像機位置，同一個注視方向，也不能確保拍攝畫面的唯一性。因此，我們定義一個「向上向量」，即`viewUp`。

拓展一下，「向上向量」的很多好處：

1. 當使用歐拉角定義攝像機旋轉時，萬向節鎖是一個常見問題，它會導致攝像機失去一個自由度的旋轉。使用向上向量和觀察向量可以防止這個問題，因爲它們定義了一個明確的攝像機方向和旋轉。
2. 有了向上向量，我們可以方便地使用叉積計算攝像機的右向量，這對於某些計算和操作非常有用。

ok，完事之後，我們開始寫代碼。

### 第二關：相機代碼

在上一關我們得到了一個相機的基本定義：位置、看向的方向以及向上向量。

但是「看向的方向」這個量不太直接，我們想要更直接的也就是攝像機想要看向哪裏，於是相機的代碼中，定義相機的三要素就是：位置（`cameraPos`），看向的位置（`lookAt`）以及向上向量（`viewUp`）。

閱讀下面代碼之前需要知道我們注視的方向是 `-z` 軸。

```c++
Matrix lookAt(Vec3f cameraPos, Vec3f lookAt, Vec3f viewUp){
    //計算出z，根據z和up算出x，再算出y
    Vec3f gazeDirection = (cameraPos - lookAt).normalize(); // -z 軸
    Vec3f horizon = (viewUp ^ gazeDirection).normalize(); // 與相機水平的軸
    Vec3f vertical = (gazeDirection ^ horizon).normalize(); //

    Matrix rotation = Matrix::identity(4);
    Matrix translation = Matrix::identity(4);
    for (int i = 0; i < 3; i++) {
        rotation[i][3] = - lookAt[i];
        rotation[0][i] = horizon[i];
        rotation[1][i] = vertical[i];
        rotation[2][i] = gazeDirection[i];
    }
    //這樣乘法的效果是先平移物體，再旋轉
    Matrix res = rotation*translation;
    return res;
}
```

![image-20230907165837308](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230907165837308.png)

## 6.1 優化/重寫代碼

首先繼續優化 geometry 類，相當於重寫一個自己的向量類。接着將當前main函數的一系列關於光柵化三角形的代碼整合到新的類中。

由於本章的每一關內容都很多，尤其是重寫模版向量類，因此我將內容較多的關卡提高了標題層次。



## 6.2 重寫模版向量類

### 第一關：需求分析

不懂c++模版類的讀者可以閱讀 **附件1** ，不看也行，但是確保你對C++的模版特性有所瞭解。

本文我一步一步帶大家實現以下功能：

1. **向量 (vec)**
   - 有通用的模板定義和2D、3D的特化版本。
   - 通用構造函數，將每個元素初始化爲T類型的默認值。
   - 2D和3D向量的構造函數可以接受特定的初始化值。
   - 提供了索引運算符來獲取或設置特定元素的值。
   - 3D向量有`norm`函數，返回向量的模長。
   - 3D向量有`normalize`函數，可以規範化向量。
   - 重載了輸出運算符，方便向量的打印。
   - 運算符重載：向量的點乘、加法、減法、標量乘法和標量除法。
   - `embed`和`proj`函數用於擴展或投影向量到不同維度。
   - 3D向量之間的外積運算。
2. **矩陣 (mat)**
   - 可以獲取或設置矩陣的行。
   - 獲取矩陣的某一列。
   - 設置矩陣的某一列。
   - 獲取單位矩陣。
   - 計算矩陣的行列式。
   - 獲取矩陣的子矩陣。
   - 計算矩陣的餘子式。
   - 計算伴隨矩陣。
   - 計算逆矩陣的轉置。
   - 運算符重載：矩陣和向量的乘法、兩個矩陣的乘法、矩陣的標量除法。
   - 重載了輸出運算符，方便矩陣的打印。
3. **其他功能**
   - 使用typedef定義了常用的類型，如`Vec2f`, `Vec3i`, `Matrix`等。
   - 在geometry.cpp中，提供了從3D和2D的float向量到int向量的轉換，以及相反的轉換。



### 第二關：實現Vec2模版以及四個算數符

這一關我們構建Vec2i和Vec2f類，以及實現他們的加、減、點積和叉積操作。

我這裏直接創建了一個新的cpp項目，名爲MyMathLib。在項目中，創建 `geometry.h` 頭文件和 `geometry.cpp` 源代碼文件。**提醒一下**，其實在寫類模版的時候儘量都把內容寫在頭文件裏即可，這裏只是暫時分開寫，我們馬上就會發現這種寫法的維護難度很大。

在頭文件中定義 Vec2 模版類，讓他既支持整形，也支持浮點數。

```c++
// .h file
#ifndef MYMATHLIB_GEOMETRY_H
#define MYMATHLIB_GEOMETRY_H

// geometry.h
#pragma once

#include <iostream>

template <typename T>
struct Vec2 {
    T x, y;

    Vec2();
    Vec2(T x, T y);

    Vec2<T> operator+(const Vec2<T>& v) const;
    Vec2<T> operator-(const Vec2<T>& v) const;
    T dot(const Vec2<T>& v) const;
    T cross(const Vec2<T>& v) const;
};

typedef Vec2<float> Vec2f;
typedef Vec2<int>   Vec2i;



#endif //MYMATHLIB_GEOMETRY_H
```

源代碼文件實現構造函數、加法、減法、點積和叉積，然後在最後實現外部模板實例化。

**注意**：一般情況下，我們都直接把所有的實現（即函數體）都放在頭文件中，這裏只是稍微拓展一下可以使用**外部模板實例化**將類的模版的實現放在.cpp中。

**首先，我們需要理解C++中的模板是什麼。**模板不是實際的函數或類，而是編譯器使用的藍圖，用於生成函數或類的特定版本。這就是爲什麼我們通常會看到模板的定義直接在頭文件中：當模板在某個源文件中使用時，編譯器需要看到完整的模板定義，以便爲特定的類型生成正確的代碼。

**爲什麼要實現外部模板實例化？**模板的定義通常直接出現在頭文件中。但有時，爲了組織或其他原因，會把模板類的定義從其聲明中分離出來（就像常規的非模板類那樣），我目前也是這樣做的。但這樣做引發了一個問題，當鏈接器嘗試鏈接對象文件時，如果它沒有爲特定的模板類型實例找到定義，就會出錯。這是因爲編譯器只爲那些它確實看到的模板類型生成代碼。

對於模板類，如果模板類的所有成員函數都在類聲明中定義（即在頭文件中定義），那麼當模板類用於特定類型時，編譯器可以立即爲該類型生成模板類的實例。但是，如果模板類的某些成員函數在類聲明之外定義（例如，在`.cpp`文件中），那麼你可能需要使用外部模板實例化來確保爲所需的類型生成正確的模板實例。

```c++
// geometry.cpp
#include "geometry.h"

// 構造函數
template <typename T>
Vec2<T>::Vec2() : x(0), y(0) {}

template <typename T>
Vec2<T>::Vec2(T x, T y) : x(x), y(y) {}

// 加法
template <typename T>
Vec2<T> Vec2<T>::operator+(const Vec2<T>& v) const {
    return Vec2<T>(x + v.x, y + v.y);
}

// 減法
template <typename T>
Vec2<T> Vec2<T>::operator-(const Vec2<T>& v) const {
    return Vec2<T>(x - v.x, y - v.y);
}

// 點積
template <typename T>
T Vec2<T>::dot(const Vec2<T>& v) const {
    return x * v.x + y * v.y;
}

// 叉積
template <typename T>
T Vec2<T>::cross(const Vec2<T>& v) const {
    return x * v.y - y * v.x;
}

template class Vec2<int>;
template class Vec2<float>;
template class Vec2<double>;
```

調用測試一下。

```c++
// main.cpp
#include "geometry.h"
#include <iostream>

int main() {
    // 使用浮點數向量
    Vec2f v1(1.0f, 2.0f);
    Vec2f v2(2.0f, 3.0f);

    Vec2f sum = v1 + v2;
    std::cout << "v1 + v2 = (" << sum.x << ", " << sum.y << ")\n";

    float dotProduct = v1.dot(v2);
    std::cout << "v1 . v2 = " << dotProduct << "\n";

    float crossProduct = v1.cross(v2);
    std::cout << "v1 x v2 = " << crossProduct << "\n";

    // 使用整數向量
    Vec2i v3(1, 2);
    Vec2i v4(2, 3);

    Vec2i sumInt = v3 + v4;
    std::cout << "v3 + v4 = (" << sumInt.x << ", " << sumInt.y << ")\n";

    int dotProductInt = v3.dot(v4);
    std::cout << "v3 . v4 = " << dotProductInt << "\n";

    int crossProductInt = v3.cross(v4);
    std::cout << "v3 x v4 = " << crossProductInt << "\n";

    return 0;
}

```

結果：

>v1 + v2 = (3, 5)
>v1 . v2 = 8
>v1 x v2 = -1
>v3 + v4 = (3, 5)
>v3 . v4 = 8
>v3 x v4 = -1

### 第三關：實現Vec3模版以及四個算數符

Vec3其實和Vec2基本一致，只需修改一下計算代碼即可，這裏就不一一展示了。大部分就是將Vec2改成Vec3，計算時增加一個維度的考慮，比方說叉積。

```c++
template <typename T>
Vec3<T> Vec3<T>::cross(const Vec3<T> &v) const {
    return Vec3<T>(
            y * v.z - z * v.y,
            z * v.x - x * v.z,
            x * v.y - y * v.x
    );
}
```

### 第四關：用模版構建不同大小的向量

如果我還想添加Vec4，豈不是又要寫一大堆，不簡潔！對於多種不同大小的向量進行明確實例化是非常繁瑣的。

這裏我是用的是 `...` 摺疊表達式（C++17）遞歸構造。目前頭文件大致結構是這樣的：

```c++
template <typename T, int N>
struct Vec {
    T values[N];

    // 構造函數
    Vec() = default; // 默認構造函數
    template<typename... Args> Vec(Args... args);
    
    // ... 操作聲明（例如加減點積等） ...
};

// ... 實現 ...

// 爲 Vec2、Vec3、Vec4 等提供類型別名

```

這裏說一下構造函數，如果是隱式構造的，那麼默認會使用Vec()。爲了代碼健壯性，其實可以在帶可變長參數的構造函數前加 `explicit` 關鍵詞。

```c++
template<typename... Args> explicit Vec(Args... args);
```

讀者可能感覺到了，我在這裏直接將 .cpp 的操作挪過來頭文件裏邊來實現了，這是因爲如果這個時候要分離寫的話，代碼冗餘量會很大。因此我們全部寫在頭文件裏面，省事優雅。下面是當前完整的 geometry.h 代碼。

```c++
#ifndef MYMATHLIB_GEOMETRY_H
#define MYMATHLIB_GEOMETRY_H

// geometry.h
#pragma once

#include <iostream>

template <typename T, int N>
struct Vec {
    T values[N];

    // 構造函數
    Vec() = default; // 默認構造函數
    template<typename... Args> explicit Vec(Args... args);

    void print() const;
    T& operator[](int index);
    const T& operator[](int index) const;

    Vec<T, N> operator+(const Vec<T, N>& other) const;
    Vec<T, N> operator-(const Vec<T, N>& other) const;
    T dot(const Vec<T, N>& other) const;

    // 叉積僅對3D向量有效
    template<int M = N>
    typename std::enable_if<M == 3, Vec<T, 3>>::type cross(const Vec<T, 3>& other) const;
};

template <typename T, int N>
template<typename... Args>
Vec<T, N>::Vec(Args... args) : values{args...} {
    static_assert(sizeof...(args) == N, "Wrong number of arguments");
}

// 打印Vec
template <typename T, int N>
void Vec<T, N>::print() const {
    std::cout << "(";
    for(int i = 0; i < N; i++) {
        std::cout << values[i] << (i < N - 1 ? ", " : ")\n");
    }
}

// 非常量的Vec訪問，可以稱爲左值
template <typename T, int N>
T& Vec<T, N>::operator[](int index) {
    return values[index];
}

// 常量Vec訪問
template <typename T, int N>
const T& Vec<T, N>::operator[](int index) const {
    return values[index];
}

// 實現加法運算
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator+(const Vec<T, N>& other) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] + other[i];
    }
    return result;
}

// 實現減法運算
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator-(const Vec<T, N>& other) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] - other[i];
    }
    return result;
}

// 實現點積運算
template <typename T, int N>
T Vec<T, N>::dot(const Vec<T, N>& other) const {
    T sum = 0;
    for(int i = 0; i < N; i++) {
        sum += values[i] * other[i];
    }
    return sum;
}

// 實現了Vec3的叉積運算
template <typename T, int N>
template<int M>
typename std::enable_if<M == 3, Vec<T, 3>>::type Vec<T, N>::cross(const Vec<T, 3>& other) const {
    return Vec<T, 3>(
            values[1] * other[2] - values[2] * other[1],
            values[2] * other[0] - values[0] * other[2],
            values[0] * other[1] - values[1] * other[0]
    );
}

// 爲 Vec2、Vec3、Vec4 等提供類型別名
using Vec2i = Vec<int, 2>;
using Vec3i = Vec<int, 3>;
using Vec4i = Vec<int, 4>;

using Vec2f = Vec<float, 2>;
using Vec3f = Vec<float, 3>;
using Vec4f = Vec<float, 4>;

using Vec2d = Vec<double, 2>;
using Vec3d = Vec<double, 3>;
using Vec4d = Vec<double, 4>;


#endif //MYMATHLIB_GEOMETRY_H

```

測試：

```c++
int main() {
    // 使用整數向量測試
    Vec3i v3(1, 2, 3);
    Vec3i v4(2, 3, 4);

    Vec3i sumInt = v3 + v4;
    sumInt.print();

    int dotProductInt = v3.dot(v4);
    std::cout << "v3 \\dot v4 = " << dotProductInt << "\n";

    Vec3i crossProduct = v3.cross(v4);
    std::cout << "v3 x v4 = "; crossProduct.print();
    
    return 0;
}
```

>(3, 5, 7)
>v3 \dot v4 = 20
>v3 x v4 = (-1, 2, -1)



### 第五關：進一步完善向量功能

先總結一下目前完成的內容：

- 通過兩個模版參數 T 和 N，定義了一個向量。
- 有默認構造函數與可變參數的構造函數。
- 功能有：打印向量、通過索引訪問向量元素、向量加法、向量減法、向量點積和三維向量叉積。
- 提供了大量常用的向量別名。

目前來說已經基本可以用了，但是還有很多需要完善，我們繼續看需要完成的內容！

1. 增加標量與向量的乘/除法
2. 計算向量的模
3. 向量單位化
4. 重載輸出運算符

另外，可以在聲明運算操作中使用 `[[nodiscard]]` 標籤，提醒編譯器注意檢查返回值是否得到使用，然後使用該庫的用戶就可以在編輯器中得到提醒，例如下面。

```c++
[[nodiscard]] Vec<T, N> normalize() const;// 將向量單位化
```

當前功能的聲明：

```c++
[[nodiscard]] Vec<T, N> operator*(T scalar) const;// 向量與常數乘法
[[nodiscard]] Vec<T, N> operator/(T scalar) const;// 向量與常數除法
[[nodiscard]] double magnitude() const;// 向量模長
[[nodiscard]] Vec<T, N> normalize() const;// 向量單位化
// 流傳輸功能
template <typename U, int M>
friend std::ostream& operator<<(std::ostream& os, const Vec<U, N>& vec);
```

對應的實現：

```c++
...
// 流傳輸功能
template <typename U, int M>
std::ostream& operator<<(std::ostream& os, const Vec<U, M>& vec) {
    os << "(";
    for(int i = 0; i < M; i++) {
        os << vec[i];
        if (i < M - 1) {
            os << ", ";
        }
    }
    os << ")";
    return os;
}

// 實現標量與向量的乘
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator*(T scalar) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] * scalar;
    }
    return result;
}

// 實現標量與向量的除法
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator/(T scalar) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] / scalar;
    }
    return result;
}

// 實現求模長
template <typename T, int N>
double Vec<T, N>::magnitude() const {
    T sum = 0;
    for(int i = 0; i < N; i++) {
        sum += values[i] * values[i];
    }
    return std::sqrt(sum);
}

// 單位化向量
template <typename T, int N>
Vec<T, N> Vec<T, N>::normalize() const {
    T mag = magnitude();
    if(mag == 0) {
        // 不能單位化一個零向量，此處可以拋出異常或返回原向量
        // 爲簡化，此處返回原向量
        return *this;
    }
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] / mag;
    }
    return result;
}
```

可以在這裏[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/Custom_Math_Lib)中獲取當前的向量庫代碼。

### 第六關：構建矩陣

有了上面構建向量模版的經驗，我們可以照葫蘆畫瓢寫出一個矩陣模版。矩陣的構造、訪問元素、加法、乘法等操作都一一實現即可。

這裏讀者應該給自己幾個小時，獨立寫出代碼。

當我作爲庫的使用者創建一個矩陣時，我會想這樣創建：

```c++
Matrix<int, 2, 2> mat = {
    {1, 2},
    {3, 4}
};
```

構造函數可以使用兩層嵌套的 `std::initializer_list`。其中，`std::initializer_list` 是一個C++11中引入的模板類，它表示編譯時確定的值列表。先遍歷行，再遍歷列。

```c++
Matrix(const std::initializer_list<std::initializer_list<T>>& list) {
    int r = 0;
    for (const auto& rowList : list) {
        int c = 0;
        for (const auto& val : rowList) {
            values[r][c] = val;
            c++;
        }
        r++;
    }
}
// BTW：奇技淫巧壓縮代碼
Matrix(const std::initializer_list< std::initializer_list<T> >& list) {
    T* target = &values[0][0];
    for (const auto& rowList : list) {
        target = std::copy(rowList.begin(), rowList.end(), target);
    }
}
```

這裏重點說一下矩陣的乘法。

```c++
template <typename T, int Rows, int Cols>
struct Matrix {
    T values[Rows][Cols];
    
	...
        
	// 矩陣與矩陣的乘法
    template<int NewCols>
    Matrix<T, Rows, NewCols> operator*(const Matrix<T, Cols, NewCols>& other) const {
        Matrix<T, Rows, NewCols> result;
        for (int i = 0; i < Rows; i++) {
            for (int j = 0; j < NewCols; j++) {
                T sum = 0;
                for (int k = 0; k < Cols; k++) {
                    sum += values[i][k] * other(k, j);
                }
                result(i, j) = sum;
            }
        }
        return result;
    }
};
```

一個 `Rows x Cols` 的矩陣A和一個 `Cols x NewCols` 的矩陣B相乘，那麼結果將是一個 `Rows x NewCols` 的矩陣。舉個例子：

```c++
Matrix<int, 3, 2> matA = { /* 初始化 */ };
Matrix<int, 2, 4> matB = { /* 初始化 */ };
Matrix<int, 3, 4> result = matA * matB;  // 這裏的乘法使用的就是上述函數，NewCols 在這裏爲4
```

下面是其他的一些操作。

```c++
// 打印矩陣函數
void print() const {
    for (int i = 0; i < Rows; i++) {
        for (int j = 0; j < Cols; j++) {
            std::cout << values[i][j];
            if (j < Cols - 1) {
                std::cout << "\t";  // 在列之間添加製表符，以美化輸出
            }
        }
        std::cout << std::endl;  // 打印換行，進入下一行
    }
}

// 訪問矩陣元素
T& operator()(int row, int col) {
    return values[row][col];
}
const T& operator()(int row, int col) const {
    return values[row][col];
}

// 矩陣加法
Matrix operator+(const Matrix& other) const {
    Matrix result;
    for (int i = 0; i < Rows; i++) {
        for (int j = 0; j < Cols; j++) {
            result(i, j) = values[i][j] + other(i, j);
        }
    }
    return result;
}

// 矩陣與標量的乘法
Matrix operator*(T scalar) const {
    Matrix result;
    for (int i = 0; i < Rows; i++) {
        for (int j = 0; j < Cols; j++) {
            result(i, j) = values[i][j] * scalar;
        }
    }
    return result;
}
```

總結一下目前的工作：

- 構建了矩陣類的模版結構
- 提供默認和可傳參的構造函數
- 打印矩陣函數
- 訪問矩陣操作符()
- 矩陣加法+
- 矩陣與標量的乘法*
- 矩陣與矩陣的乘法*

添加了完整的註釋供大家參考，可以在這個[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/Custom_Math_Lib_v2)中找到當前的數學庫。

### 第七關：繼續完善矩陣庫

- 添加單位矩陣identity

```c++
// 添加單位矩陣功能
static Matrix<T, Rows, Cols> identity();
...
// 實現
template <typename T, int Rows, int Cols>
Matrix<T, Rows, Cols> Matrix<T, Rows, Cols>::identity() {
    static_assert(Rows == Cols, "Identity matrix can only be created for square matrices.");
    Matrix<T, Rows, Cols> mat = {}; // Initialize all elements to zero
    for (int i = 0; i < Rows; ++i) {
        mat(i, i) = 1;
    }
    return mat;
}
```

- 插入列

```c++
void set_col(size_t idx, const Vec<T, Rows>& v) const;
...
template<typename T, int Rows, int Cols>
void Matrix<T, Rows, Cols>::set_col(size_t idx, const Vec<T, Rows>& v) const{
    assert(idx < Cols);// 編譯器先判斷需要設置的行是否合理
    for (int i = 0; i < Rows; i++) {
        values[i][idx] = v[i];
    }
}
```

- 插入列

```c++
void set_row(size_t idx, const Vec<T, Rows>& v);
...
// 添加插入行向量到矩陣的功能
template<typename T, int Rows, int Cols>
void Matrix<T, Rows, Cols>::set_row(size_t idx, const Vec<T, Rows>& v){
    assert(idx < Cols);// 編譯器先判斷需要設置的行是否合理
    for (size_t j = 0; j < Rows; j++) {
        values[idx][j] = v[j];
    }
}
```

測試：

```c++
Matrix4f m = Matrix4f::identity();
const Vec4f vec4F(2,2,2,2);
m.set_col(1,vec4F);
m.set_row(3, vec4F);
m.print();
return 0;
```

輸出：

>1	2	0	0
>0	2	0	0
>0	2	1	0
>2	2	2	2

- 重載 `[][]` 

現在我如果要取用 Matrix 的 mat 對象的數值，我們是這樣的

```c++
mat.values[][]
```

但是我想直接

```c++
mat[][]
```

此時我們需要使用代理對象的設計模式：

```c++
template <typename T, int Rows, int Cols>
struct Matrix {
    T values[Rows][Cols];

    // ... 其他成員函數和數據 ...

    // 代理對象
    struct RowProxy {
        T* row;
        T& operator[](int col) {
            return row[col];
        }
    };

    RowProxy operator[](int row) {
        return RowProxy{values[row]};
    }

    RowProxy operator[](int row) const {
        return RowProxy{values[row]};
    }
};
```







## 6.3 整合光柵化代碼

瀏覽我們目前的main函數，既有矩陣變換函數，也有視角變換函數，還有三角形重心座標光柵化三角形的函數，更有視角變換矩陣等，真的有些亂。我們將這些方法打包到一個新的類裏面，這個類稱爲：our_gl。

### 特別節目1之：main代碼之旅

最終main函數如下：

```c++
#include <vector>
#include <iostream>

#include "tgaimage.h"
#include "model.h"
#include "geometry.h"
#include "our_gl.h"

Model *model     = NULL;
const int width  = 800;
const int height = 800;

Vec3f light_dir(1,1,1);
Vec3f       eye(0,-1,3);
Vec3f    center(0,0,0);
Vec3f        up(0,1,0);

struct GouraudShader : public IShader {
    Vec3f varying_intensity; // written by vertex shader, read by fragment shader

    virtual Vec4f vertex(int iface, int nthvert) {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;     // transform it to screen coordinates
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir); // get diffuse lighting intensity
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        float intensity = varying_intensity*bar;   // interpolate intensity for the current pixel
        color = TGAColor(255, 255, 255)*intensity; // well duh
        return false;                              // no, we do not discard this pixel
    }
};

int main(int argc, char** argv) {

    model = new Model("../object/african_head/african_head.obj");

    lookat(eye, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(-1.f/(eye-center).norm());
    light_dir.normalize();

    TGAImage image  (width, height, TGAImage::RGB);
    TGAImage zbuffer(width, height, TGAImage::GRAYSCALE);

    GouraudShader shader;
    for (int i=0; i<model->nfaces(); i++) {
        Vec4f screen_coords[3];
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, zbuffer);
    }

    image.  flip_vertically(); // to place the origin in the bottom left corner of the image
    zbuffer.flip_vertically();
    image.  write_tga_file("output.tga");
    zbuffer.write_tga_file("zbuffer.tga");

    delete model;
    return 0;
}
```

![image-20230921171158181](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921171158181.png)

其中，頂點着色和片元着色是可編程的。可以參考當前的項目代碼[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/6_1_Our_GL)。

🎬 **開場白**

接下來開始解讀這個main函數。首先，代碼導入了一堆頭文件，爲了讓我們的程序能夠處理3D模型、向量計算和圖像生成。

```c++
#include <vector>
#include <iostream>
#include "tgaimage.h"
#include "model.h"
#include "geometry.h"
#include "our_gl.h"
```

🌍 **全局變量來啦**

接着，全局變量閃亮登場！有了寬度、高度、光照方向、觀察點等等，這簡直是個小型的“宇宙”。

```c++
Model *model     = NULL;
const int width  = 800;
const int height = 800;
Vec3f light_dir(1,1,1);
Vec3f       eye(0,-1,3);
Vec3f    center(0,0,0);
Vec3f        up(0,1,0);
```

🎭 **GouraudShader 誕生**

然後，我們有一個名爲 `GouraudShader` 的類，這傢伙是渲染的明星！它的職責是處理頂點和片段（像素）。

```c++
struct GouraudShader : public IShader {
    // ... 
}
```

🎸 **主舞臺 main 函數**

最後，`main()` 函數，這是我們的主舞臺。所有的預設、加載、渲染都在這裏完成。

```c++
int main(int argc, char** argv) {
    //...
}

```

🎥 **Action！動作！**

1. **加載模型**: `new Model("../object/african_head/african_head.obj");` 這裏，我們召喚了一個來自非洲的神祕頭顱！
2. **視角設置**: 使用 `lookat`, `viewport`, 和 `projection` 函數，我們調整了觀察點、視口和投影。這些都是電影導演級別的設置！
3. **初始化畫布**: `TGAImage image (width, height, TGAImage::RGB);` 這裏我們預備了一張畫布，準備大展身手！
4. **渲染循環**: 嗯，這裏有一個循環，負責畫出那個非洲頭顱。用了 `GouraudShader`，它會逐個面片地渲染模型。
5. **圖片翻轉和保存**: 最後，不要忘了翻轉圖像，並保存爲 `.tga` 格式。現在你就可以拿這個圖跟朋友炫耀了！

### 特別節目2之：細說GouraudShader

這個角色是渲染的靈魂，讓我們細緻入微地來看一下它的表演。

**🕺GouraudShader的組成**

- **變量：varying_intensity**

這個變量是一個3D向量（Vec3f類型），用來存儲每個頂點的光照強度。這裏的“varying”意味着這個變量會在頂點着色器和片段着色器之間“變化”（實際上是插值）。

- **方法：vertex**

這個函數負責處理每個頂點。它做了以下幾件事：

1. **獲取模型頂點**: `Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert));` - 從3D模型中提取出一個頂點。
2. **座標轉換**: `gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;` - 使用各種矩陣變換將這個頂點從模型空間轉換到屏幕空間。
3. **光照計算**: `varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir);` - 根據光照方向計算這個頂點的光照強度。

- **方法：fragment**

這個函數負責處理每個像素（片段）：

1. **插值計算**: `float intensity = varying_intensity*bar;` - 這裏使用bar來進行插值，得到當前像素的光照強度。
2. **顏色設置**: `color = TGAColor(255, 255, 255)*intensity;` - 根據光照強度設置像素的顏色。
3. **像素保留**: `return false;` - 表示這個像素不會被丟棄，將出現在最終的圖像中。

**🎭角色分析**

這個`GouraudShader`類扮演了一個全能藝人的角色：

- **化妝師**：通過`vertex`函數，對每個頂點進行"化妝"，也就是座標變換和光照計算。
- **導演**：通過`fragment`函數，決定哪些像素應該用什麼顏色來"演繹"，以及哪些像素應該被"剪輯"掉（這裏沒有剪輯，所有像素都保留）。
- **燈光師**：通過計算光照強度，控制場景的"明暗"，使得模型更加逼真。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921174128141.png" alt="image-20230921174128141" style="zoom:50%;" />

### 特別節目3之：開始繪畫-片元着色器

OK，我們重新化妝一下，修改片元着色器（fragment）。

```c++
float intensity = varying_intensity*bar;// 通過插值計算得到當前像素的光照強度。
if (intensity>.85) intensity = 1;
else if (intensity>.60) intensity = .80;
else if (intensity>.45) intensity = .60;
else if (intensity>.30) intensity = .45;
else if (intensity>.15) intensity = .30;
else intensity = 0;
color = TGAColor(155, 155, 0)*intensity;
return false;
```

根據光照強度對像素進行了“分級”。每個級別都有一個特定的光照強度，就像你在照片編輯軟件裏手動設置不同級別的亮度。

這裏將顏色設置爲一個黃色（155, 155, 0），然後用上面的 `intensity` 來調節這個顏色。結果是一種不同深淺的黃色。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921174422284.png" alt="image-20230921174422284" style="zoom:50%;" />

着色器代碼就像藝術家的調色板，你永遠不知道接下來會畫出什麼樣的圖像！以下是一些有趣的着色器代碼片段給大夥參考參考：

#### 🌈 彩虹着色器

```c++
float t = varying_intensity*bar;
color = TGAColor(
    128 + 127 * std::sin(t),
    128 + 127 * std::sin(t + 2.f/3.f * 3.14159f),
    128 + 127 * std::sin(t + 4.f/3.f * 3.14159f)
);
return false;
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193615375.png" alt="image-20230921193615375" style="zoom:50%;" />

#### 📺 模擬老電視效果

```c++
float t = varying_intensity*bar;
float noise = rand() % 100 / 100.0;
if (noise > 0.9) {
    color = TGAColor(255, 255, 255);
} else if (noise < 0.1) {
    color = TGAColor(0, 0, 0);
} else {
    color = TGAColor(155, 155, 155) * t;
}
return false;
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193810437.png" alt="image-20230921193810437" style="zoom:50%;" />

#### 🔥 火焰效果

```c++
float intensity = varying_intensity * bar;
color = TGAColor(
        255 * intensity,
        (int)(160 * std::sqrt(intensity)),
        0
);
return false;
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193918227.png" alt="image-20230921193918227" style="zoom:50%;" />

#### 🌌 星空效果

```c++
float noise = rand() % 100 / 100.0;
if (noise > 0.98) {
    color = TGAColor(255, 255, 255);
} else {
    color = TGAColor(0, 0, 0);
}
return false;
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193958547.png" alt="image-20230921193958547" style="zoom:50%;" />





## 6.4 升級Shader-支持UV紋理

在上一節我們把玩了`GouraudShader`，現在我介紹一個更強的選手，`Shader`，他不僅能夠處理光照，還支持紋理貼圖！

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921212841026.png" alt="image-20230921212841026" style="zoom:50%;" />

```c++
struct Shader : public IShader {
    Vec3f          varying_intensity; // written by vertex shader, read by fragment shader
    mat<2,3,float> varying_uv;        // same as above

    Vec4f vertex(int iface, int nthvert) override {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir); // get diffuse lighting intensity
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }

    bool fragment(Vec3f bar, TGAColor &color) override {
        float intensity = varying_intensity*bar;   // interpolate intensity for the current pixel
        Vec2f uv = varying_uv*bar;                 // interpolate uv for the current pixel
        color = model->diffuse(uv)*intensity;      // well duh
        return false;                              // no, we do not discard this pixel
    }
};
```

### 🎬 **Shader類的角色列表**

1. **varying_intensity**：這個角色沒變，依然是頂點着色器計算出的光照強度。
2. **varying_uv**：新角色登場！這傢伙用來存儲每個頂點的紋理座標（u,v）。

### 🎭 **vertex函數：多面手**

這個函數的流程與之前的大同小異，但多了一個關鍵步驟：

```c++
varying_uv.set_col(nthvert, model->uv(iface, nthvert));
```

這裏，它從模型中獲取每個頂點的紋理座標（UV座標）並存儲下來。這些座標將被用於後面的片段着色器中進行紋理貼圖。

### 🌈 **fragment函數：藝術家**

在這個函數里，除了處理光照之外，我們還添加了紋理：

```c++
float intensity = varying_intensity*bar; // 同樣是計算當前像素的光照強度
Vec2f uv = varying_uv*bar;               // 新功能：計算當前像素的紋理座標
```

這裏，`bar`用來進行插值，得到當前像素點的光照強度和紋理座標。

```c++
color = model->diffuse(uv)*intensity; // 結合紋理和光照來計算最終顏色
```

接着，最精彩的部分來了。我們將之前計算的 `intensity` 和從 `uv` 貼圖中獲取的顏色相乘，得到的就是一個非常真實的顏色了。

### 🎨 **那麼，這個Shader類都能做什麼？**

1. **紋理貼圖**：它能給3D模型穿上“衣服”，使模型看起來更逼真。
2. **漫反射光照**：它依然做好了基礎的光照工作，讓模型不會看起來像個平面。
3. **代碼複用**：由於這個類繼承了`IShader`，你可以很容易地在不同的渲染任務中複用這段代碼。



## 6.5 學習法線貼圖

法線貼圖是一種用於3D計算機圖形的技術，用於使3D模型的表面看起來更加詳細，而無需使用更多的多邊形。

簡而言之，你使用一張紋理來存儲有關如何微妙調整模型表面上的法線向量的信息，從而改變光與模型表面的相互作用方式。

### 第一關：紋理

用於法線貼圖的紋理通常看起來像一種奇怪、抽象的藍色混合物。每個像素的RGB值代表一個3D向量的X、Y、Z分量，這將用於在光照計算期間調整3D模型的表面法線。這與僅依賴模型幾何形狀計算得出的法線（每個頂點的法線）有所不同。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921214038676.png" alt="image-20230921214038676" style="zoom:50%;" />

### 第二關：全局座標系與Darboux座標系

- **全局座標系**：在這裏，法線是在全局（世界座標）系統中表示的。

- **Darboux（切線空間）座標系**：在這裏，法線是相對於對象本身的表面表示的。Z向量垂直於物體的表面，而X和Y與物體的表面相切。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921222119442.png" alt="image-20230921222119442" style="zoom:50%;" />

通常，Darboux座標系中的紋理看起來更加“有機”或“彎曲”，因爲它是相對於物體表面的。全局座標系中的紋理可能看起來更“統一”或“筆直”。因此，Darboux座標系（切線空間）通常被認爲更好，原因是它是對象相對的，使其在複雜的3D場景中具有更高的靈活性和通用性。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921222407082.png" alt="image-20230921222407082" style="zoom:50%;" />

```c++
struct Shader : public IShader {
    mat<2,3,float> varying_uv;  // same as above
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
   }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec2f uv = varying_uv*bar;                 // interpolate uv for the current pixel
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize();
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize();
        float intensity = std::max(0.f, n*l);
        color = model->diffuse(uv)*intensity;      // well duh
        return false;                              // no, we do not discard this pixel
    }
};
[...]
    Shader shader;
    shader.uniform_M   =  Projection*ModelView;
    shader.uniform_MIT = (Projection*ModelView).invert_transpose();
    for (int i=0; i<model->nfaces(); i++) {
        Vec4f screen_coords[3];
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, zbuffer);
    }
```

### 第三關：經常見到的Uniform

首先，“Uniform”是GLSL（圖形庫着色器語言）中的一個保留關鍵字。這個關鍵字讓你能夠將常量（不會在渲染過程中改變的值）傳遞到着色器中。這裏我們的渲染器也保留GLSL的名字。

就像給了演員一個劇本，告訴他們："別亂改，按這個演！"

### 第四關：光照計算

在上面這段代碼中，光照強度的計算與之前基本相同，但有一個例外：它不是從每個頂點插值得到法線向量，而是從法線貼圖紋理中獲取這些信息。

換句話說，以前是依賴演員的自然演技（頂點法線），現在我們有了特效化妝師（法線貼圖紋理）來讓演員更出彩！

簡而言之，上面這段代碼就是將業餘戲劇社升級到好萊塢級別的製作！

## 6.6 實現Phong模型

Phong模型包含了三項**漫反射 (Diffuse Reflection)**、**鏡面反射 (Specular Reflection)**和**環境反射 (Ambient Reflection)**。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/agQhoxeqEs9IRpB.png" alt="image-20230525172542652" style="zoom:50%;" />
$$
\begin{aligned}
L & =L_a+L_d+L_s \\
& =k_a I_a+k_d\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{l})+k_s\left(I / r^2\right) \max (0, \mathbf{n} \cdot \mathbf{h})^p
\end{aligned}
$$
<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921234503261.png" alt="image-20230921234503261" style="zoom:50%;" />

```c++
struct Blinn_Phong_Shader : public IShader {
    mat<2,3,float> varying_uv;  // same as above
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()

    Vec4f vertex(int iface, int nthvert) override {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }
    bool fragment(Vec3f bar, TGAColor &color) override {
        Vec2f uv = varying_uv*bar;
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize(); // normal
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize(); // light direction
        Vec3f v = Vec3f(0, 0, -1); // simplified view direction
        Vec3f h = (l + v).normalize(); // halfway vector

        float spec = pow(std::max(0.f, n*h), model->specular(uv));
        float diff = std::max(0.f, n*l);
        TGAColor c = model->diffuse(uv);
        color = c;
        for (int i=0; i<3; i++) color[i] = std::min<float>(5 + c[i]*(diff + .6*spec), 255);
        return false;
    }
};
```

接下來，我們將會討論陰影。

## 7.1 陰影

需要注意的是，這裏我們談論的是硬陰影，軟陰影的實現又是另外一回事了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922113248506.png" alt="image-20230922113248506" style="zoom: 33%;" /><img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922113300470.png" alt="image-20230922113300470" style="zoom: 33%;" />

上面兩張圖片就是本章要實現的內容。讀者可能會想，右邊的圖片拍攝角度是不是有問題。實際上，右邊這張圖的拍攝位置是光源所在的位置。至於爲什麼，我們就在本章詳細探討。你可能會看到上圖有一些瑕疵，這正是 [Z-fighting](https://en.wikipedia.org/wiki/Z-fighting) 現象。

### 第一關：目前的問題

回到我們上一章完成的進度。根據我們的常識，在光線照射不到的地方（圖中高亮人物脖子的一側），應該與能照射到的地方有比較明顯的光照分界。目前我們的渲染器輸出的效果是左圖，而正常來說應該像右邊的圖片那樣。

<img src="/Users/remooo/Library/Application%20Support/typora-user-images/image-20230922151625557.png" alt="image-20230922151625557" style="zoom: 33%;" /><img src="/Users/remooo/Library/Application%20Support/typora-user-images/image-20230922152552309.png" alt="image-20230922152552309" style="zoom: 33%;" />

爲了解決這個問題，就要搬出圖形學大名鼎鼎的 two-pass 方法（two-pass rendering）了。這個方法基本思想是先從光源處渲染一副有深度信息的圖片，這張照片記錄了從光源視角看到的深度信息。接下來再從主相機視角渲染圖像，通過上一 Pass 的深度信息判斷當前的渲染像素點時候直接被光照射。

### 第二關：第一趟渲染-從光源出發

```c++
{ // rendering the shadow buffer
    TGAImage depth(width, height, TGAImage::RGB);
    lookat(light_dir, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(0);

    DepthShader depthshader;
    Vec4f screen_coords[3];
    for (int i=0; i<model->nfaces(); i++) {
        for (int j=0; j<3; j++) {
            screen_coords[j] = depthshader.vertex(i, j);
        }
        triangle(screen_coords, depthshader, depth, shadowbuffer);
    }
    depth.flip_vertically(); // to place the origin in the bottom left corner of the image
    depth.write_tga_file("depth.tga");
}
```

<img src="/Users/remooo/Library/Application%20Support/typora-user-images/image-20230922163751926.png" alt="image-20230922163751926" style="zoom:50%;" />

### 第三關：第二趟渲染-從主相機出發

```c++
{ // rendering the frame buffer
    TGAImage frame(width, height, TGAImage::RGB);
    lookat(eye, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(-1.f/(eye-center).norm());

    Shader shader(ModelView, (Projection*ModelView).invert_transpose(), M*(Viewport*Projection*ModelView).invert());
    Vec4f screen_coords[3];
    for (int i=0; i<model->nfaces(); i++) {
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, frame, zbuffer);
    }
    frame.flip_vertically(); // to place the origin in the bottom left corner of the image
    frame.write_tga_file("framebuffer.tga");
}
```

效果非常好～該有的陰影都有了。但是我們注意這隻怪物的手，陰影非常奇怪。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922163743152.png" alt="image-20230922163743152" style="zoom:50%;" />

奇怪的手臂陰影，這種現象我們稱爲陰影痤瘡（Shadow Acne）。當渲染的物體與其陰影深度映射幾乎重合時，可能會出現陰影斑點或噪聲。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922164007209.png" alt="image-20230922164007209" style="zoom:50%;" />

怎麼解決呢？可以考慮提高“閾值”，讓物體沒那麼容易被相鄰的部位遮擋住自己。具體到寫代碼上，就是稍微減小對應點的深度值。一點點小小的魔法，就可以解決這個煩人的問題了。

```c++
float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]+43.34); // magic coeff to avoid z-fighting
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922170825404.png" alt="image-20230922170825404" style="zoom:50%;" />

項目的代碼可以還是繼續提供給讀者們解讀，[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/7_1_Shadow)。

由於我比較懶，不想寫太詳細解讀了，讀者自己應該可以讀懂。

## 8.1 環境光遮蔽 - 模擬全局光照效果

上一講我們實現了 Phong 光照模型，他的組成有三項分別是環境光、高光以及漫反射。還講了一種渲染策略，Two-Pass渲染。這裏，我們介紹一種新的全局光照模擬技術，環境光遮蔽（Ambient Occlusion, AO）。

但是，Phong模型只考慮了物體與特定光源之間的直接互動。在物體的小凹陷或接近的物體之間的接觸區域，經常會出現微小的陰影。這些陰影往往與任何特定的光源無關，而是由於環境光被周圍的幾何體部分遮擋所造成的。Phong模型無法捕獲這種效果，而AO可以。

巧合的是，AO並不直接依賴於場景中的光源位置或屬性。這使得它可以與任何光照模型（如Phong模型）結合使用，併爲渲染效果增添額外的真實感。

### 特別節目：知識脈絡

讀者讀到這裏可能會有很多疑惑，對這些名詞概念的層級把握不清楚，這裏給讀者梳理一下。

1. **光照模型**
   - **Phong模型**
     - 環境光
     - 漫反射光
     - 鏡面反射光
   - **Blinn-Phong模型**
   - Lambert模型
   - Cook-Torrance模型
   - Oren-Nayar模型
2. **全局光照模擬技術**
   - **環境光遮蔽 (AO)**
   - 光線追蹤 (Ray Tracing)
   - 光子映射 (Photon Mapping)
   - 輻射度緩存 (Radiance Caching)
   - Final Gathering
3. **渲染策略**
   - **Two-Pass渲染**
     - Two-Pass陰影映射
   - 多Pass渲染
   - 延遲渲染 (Deferred Rendering)
   - 前向渲染 (Forward Rendering)
4. **後處理效果**
   - 色調映射 (Tone Mapping)
   - 抗鋸齒技術 (如 MSAA, FXAA, TAA)
   - 深度模糊 (Depth of Field)
   - 動態模糊 (Motion Blur)
5. **紋理技術**
   - 傳統紋理映射
   - 法線貼圖 (Normal Mapping)
   - 拋物線映射 (Parallax Mapping)
   - 物理基礎渲染 (Physically-Based Rendering, PBR) 的材質紋理（如 Albedo, Roughness, Metallic）



### 第一關：啥是AO？如何結合Phong使用？

環境光遮蔽的基本思想是評估一個給定的表面點在多大程度上被其周圍的幾何體遮擋。一個被其他物體嚴重遮擋的點會接收到更少的環境光，因此看起來會更暗。

當你在場景中使用Phong模型和環境光遮蔽時，通常的方法是先計算Phong模型的環境反射組成部分，然後使用環境光遮蔽來調整這個值。具體來說，你會將環境光遮蔽值乘以Phong模型的環境光分量，從而在需要的地方減少環境光。

計算方式有很多，最簡單的是屏幕空間技術（如SSAO，Screen Space Ambient Occlusion）。但是在介紹這個方法之前，我們不妨先自行思考一下我們如何實現。

### 第二關：做夢

想象一下你正在拍攝一個物體，而這個物體上方有一個半透明的傘，傘的下半部分可以發出均勻的光。現在，爲了知道物體的哪些部分更容易被這個光照亮，你決定在傘的內側隨機選擇一些點，然後看看從這些點發出的光線能不能照到物體。

它採用一種“暴力”的方法：隨機選擇很多點，並從每一個點觀察物體。

爲了記錄物體上哪些部分被光照到了，我們用一個圖片來記錄。每一次從傘的一個點看物體，都會產生一個新的圖片。

最後，我們把所有的圖片混合在一起，得到一個平均的圖片。這個圖片會告訴我們，物體的哪些部分通常更容易被光照到。

但是，這種方法也有缺點。比如，如果物體的兩個手臂在最終的圖片中使用了相同的位置，那麼這兩個手臂上的光就會被計算兩次，這會導致最終的效果不準確。

### 第三關：屏幕空間環境遮擋 (SSAO) 

全局照明非常昂貴，需要爲很多點計算可見性。爲了找到一個在計算時間和渲染質量之間的平衡，我們嘗試使用SSAO。

在這裏我們將SSAO用作一個單獨的效果，只計算環境遮擋而不計算其他光照。

- ZShader

這個着色器主要用於渲染z-buffer，只關心深度，不關心顏色。

```c++
struct ZShader : public IShader {
    mat<4,3,float> varying_tri;

    Vec4f vertex(int iface, int nthvert) override {
        Vec4f gl_Vertex = Projection*ModelView*embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, gl_Vertex);
        return gl_Vertex;
    }

    bool fragment(Vec3f gl_FragCoord, Vec3f bar, TGAColor &color) override {
        color = TGAColor(0, 0, 0);
        return false;
    }
};
```

- Max_elevation_angle

估算一個像素點與其周圍環境的最大仰角，這是評估遮擋程度的關鍵函數。

```c++
float max_elevation_angle(float *zbuffer, Vec2f p, Vec2f dir) {
    float maxangle = 0;
    for (float t=0.; t<1000.; t+=1.) {
        Vec2f cur = p + dir*t;
        if (cur.x>=width || cur.y>=height || cur.x<0 || cur.y<0) return maxangle;
        float distance = (p-cur).norm();
        if (distance < 1.f) continue;
        float elevation = zbuffer[int(cur.x)+int(cur.y)*width]-zbuffer[int(p.x)+int(p.y)*width];
        maxangle = std::max(maxangle, atanf(elevation/distance));
    }
    return maxangle;
}
```

- 環境光遮蔽的計算

對於每個像素，使用8個方向的射線來評估其環境遮擋程度。

```c++
for (int x=0; x<width; x++) {
    for (int y=0; y<height; y++) {
        if (zbuffer[x+y*width] < -1e5) continue;
        float total = 0;
        for (float a=0; a<M_PI*2-1e-4; a += M_PI/4) {
            total += M_PI/2 - max_elevation_angle(zbuffer, Vec2f(x, y), Vec2f(cos(a), sin(a)));
        }
        total /= (M_PI/2)*8;
        total = pow(total, 100.f);
        frame.set(x, y, TGAColor(total*255, total*255, total*255));
    }
}
```

項目完整代碼，[鏈接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/8_1_AO)。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230925172916074.png" alt="image-20230925172916074" style="zoom:50%;" />







## 附錄1. c++模版類 - 從入門到入土

第一關：爲什麼需要模版類？

第二關：「函數模版」

第三關：「類模版」

第四關：「多模板參數」與「非類型參數」

第五關：「模板特化」

第六關：「類型推斷」

​	1. auto & decltype 2. 模板中的基本類型推斷3. 自動構造模版類型4. 尾返回類型

第七關：「變量模板」

第八關：「模板類型別名」

第九關：模板的SFINAE原則

第十關：模板與友元

第十一關：摺疊表達式

第十二關：模板概念 - C++20

第十三關： `std::enable_if` 和 SFINAE

第十四關：類模板偏特化

第十五關：`constexpr` 和模板

第十六關：模板中的嵌套類型

第十七關：模板參數包與展開

第十八關：Lambda 表達式與模板

第十九關：模板遞歸

第二十關：帶有模板的繼承

### 第一關：爲什麼需要模版類？

在沒有模板之前，如果你想爲不同的數據類型編寫相同的功能，你可能需要爲每種數據類型寫一個函數或類。這會導致大量的重複代碼。

> 用專業的話來說就是，函數模板和類模板在 C++ 中是用來支持泛型編程的工具。泛型編程是一種編寫與類型無關的代碼的方法。這就意味着，通過使用模板，你可以創建一個能夠適應任何數據類型的函數或類，而不需要爲每種數據類型都重新編寫代碼。

例如一個函數，它的任務是交換兩個整數的值。後來，你又想交換兩個浮點數。沒有模板，你可能需要爲每種數據類型編寫單獨的函數。

### 第二關：「函數模版」

解決上面提到的問題，非常簡單。

```c++
// 模版函數
template <typename T>
void swap(T &a, T &b) {
    T temp = a;
    a = b;
    b = temp;
}
// 調用方法
int x = 5, y = 10;
swap(x, y);

double m = 5.5, n = 10.5;
swap(m, n);
```

`template <typename T>` 聲明瞭一個模板函數。此處的 `T` 可以被認爲是一個佔位符，它在編譯時會被實際的數據類型替換。

### 第三關：「類模版」

類模版跟函數模版差不多。下面的例子是一個用於存儲任意類型的數組的類。

```c++
// 模版類
template <typename T>
class Array {
private:
    T *data;
    int size;
public:
    Array(int s) : size(s) {
        data = new T[size];
    }
    ~Array() {
        delete[] data;
    }
    T& operator[](int index) { // 實現索引獲取元素
        return data[index];
    }
};
// 如何調用？
Array<int> intArray(10);
Array<double> doubleArray(10);
```

### 第四關：「多模板參數」與「非類型參數」

可以爲一個模板定義多個參數。同時，參數可以是上面所說的 `typename T` 非類型參數，也可以是類型參數，像下面代碼中的 `int SIZE` 。

```c++
// 多模板參數、非類型參數
template <typename T, int SIZE>
class FixedArray {
private:
    T data[SIZE];
public:
    T& operator[](int index) {
        return data[index];
    }
}
// 使用方式
FixedArray<int, 10> intArray;
```

### 第五關：「模板特化」

有時候，希望某個模板對某個特定類型有一個不同的實現。這時你可以使用模板特化。假如現在有下面的模版。

```c++
template <typename T>
class Printer {
public:
    void print(T value) {
        std::cout << "General print: " << value << std::endl;
    }
};
```

但我希望對於 `int` 類型有一個特殊的輸出。

```c++
template <>
class Printer<int> {
public:
    void print(int value) {
        std::cout << "Special print for int: " << value << std::endl;
    }
};
```

### 第六關：「類型推斷」

#### 1. auto & decltype

在 C++11 中引入了很多特性，其中一個與類型推斷相關的特性是“auto”關鍵字。除了剛纔說的“auto”，C++11還引入了“decltype”關鍵詞，可以判斷一個表達式的類型。

```c++
auto x = 42;    // x 的類型被推斷爲 int
auto y = 3.14;  // y 的類型被推斷爲 double

int num = 5;
decltype(num) y = 10;  // y 的類型被推斷爲 int
```

#### 2. 模板中的基本類型推斷

此外，函數模板的類型推斷在 C++ 中已經存在了一段時間，但 C++11 增強了這一特性。函數模板可以自動推斷類型參數。

```c++
template <typename T>
void show(T value) {
    std::cout << value << std::endl;
}

// 調用
show(5);        // 5
show(3.14);     // 3.14
```

#### 3. 自動構造模版類型

在 C++17 之後，類型推斷就更加強大了。在 C++17 之前，類模板的類型參數不能自動推斷。但是從 C++17 開始，我們可以通過模板參數的自動類型推斷來構造類模板的對象。

```c++
template <typename T>
class MyClass {
    T data;
public:
    MyClass(T d) : data(d) {}
    void display() {
        std::cout << data << std::endl;
    }
};

int main() {
    // C++17 之前的方式
    MyClass<int> obj1(10);
    obj1.display();

    // C++17 之後的方式
    MyClass obj2(10);    // 自動推斷爲 MyClass<int>
    obj2.display();
}
```

#### 4. 尾返回類型

C++11 引入了尾返回類型，使得函數的返回類型可以基於其參數進行推斷，這對於模板特別有用。下面代碼的 `->` 用於指定函數的尾返回類型。此時，`auto` 告訴編譯器函數返回類型將由其後的表達式來決定，也就是剛剛說的 `->` 。

```c++
template <typename T1, typename T2>
auto add(T1 x, T2 y) -> decltype(x + y) {
    return x + y;
}

int main() {
    auto result = add(5, 3.14);  // 結果的類型推斷爲 double
    std::cout << result << std::endl;
}
```

### 第七關：「變量模板」

C++14 引入了變量模板，它允許你爲模板定義靜態數據成員。它與函數和類的模板類似，但是用於變量。

我們定義了一個名爲 `pi` 的變量模板，它爲每種類型 `T` 提供了 π 的近似值。你可以像使用其他模板那樣使用變量模板，但需要指定模板參數來獲取相應的變量實例。

```c++
template <typename T>
constexpr T pi = T(3.1415926535897932385);

int main() {
    std::cout << pi<int> << std::endl;        // 輸出 3
    std::cout << pi<double> << std::endl;     // 輸出 3.14159...
}
```

一般這個「變量模版」非常適用於那些需要爲不同類型提供不同值或配置的情況。同時使用的時候注意以下事項：

- 變量模板通常與 `constexpr` 一起使用，以確保它們在編譯時是常數。
- 變量模板的實例化方式與函數或類模板相似。當你第一次爲特定的類型使用變量模板時，編譯器將爲該類型創建一個實例。

###  第八關：「模板類型別名」

「模板類型別名」爲已存在的模板類型定義了一個新的、更簡短的名稱。

在 C++11 之前，如果你想爲複雜的模板類型創建別名，這往往是非常麻煩的。C++11 引入了 `using` 關鍵字來創建模板類型別名，這提供了一個更清晰、更簡潔的方式來定義這些別名。

這裏以 第三關 的例子說明創建別名的最簡單實踐。

```c++
template <typename T>
using MyArray = Array<T>;
```

這裏再舉一個簡單、常用的例子爲常見的向量類型提供別名。

```c++
using Vec3f = Vec3<float>;
using Vec3d = Vec3<double>;
using Vec4f = Vec4<float>;
using Vec4d = Vec4<double>;

Vec3f position;
```

值得注意的是，你也可以用old school的方法，即typedef。上下兩段代碼是完全一致的。

```c++
typedef Vec2<float> Vec2f;
typedef Vec2<int>   Vec2i;
typedef Vec3<float> Vec3f;
typedef Vec3<int>   Vec3i;
```

他們的區別在於，`typedef` 使用舊的 C/C++ 語法，而`using` 是 C++11 引入的新語法，用於定義類型別名。對於簡單的類型別名，這兩種方法之間的差異可能不明顯。但是，當涉及到更復雜的類型，如函數指針或模板類型，`using` 的語法往往更爲簡潔和直觀。

> 這裏拓展一下，`using` 和 `typedef`  兩者一個主要的區別是，`using`可以爲模板提供別名。
>
> ```c++
> template <typename T>
> using Vec2Ptr = Vec2<T>*;
> ```

### 第九關：模板的SFINAE原則

SFINAE 原則是 C++ 模板中的一個特性。SFINAE是“Substitution Failure Is Not An Error”（替換失敗不是錯誤）的縮寫。當試圖用給定的模板參數替換模板時，如果發生錯誤，則該特殊化不被考慮。

想象一下你正在爲一個魔法展示準備一套卡片。每張卡片上都有一個指令，例如“變成兔子”或“飛起來”。但有一張卡片的指令是“讓豬飛起來”。顯然，這是一個不可能的任務。

在通常情況下，魔術師會看到這張卡片並說：“這個指令有問題，展示失敗了！”。但在 SFINAE 的世界裏，魔術師會說：“好吧，這張卡片不工作，讓我試試下一張”。

換句話說，SFINAE 就像是編譯器的一個內置魔術師。當你嘗試用一個不合適的類型進行模板替換時，而不是直接報錯，編譯器會悄悄地“忽略”那個模板，並嘗試其他的選項。

直到沒有選項合適（**No matching**）或者很多合適選項（**Ambiguous**），編譯器就會報出錯誤。

一個簡單的場景：我們希望寫一個函數 `printValue`，該函數可以打印整數或字符串。但是，如果我們嘗試使用其他類型，這個函數就不應該存在。

```c++
#include <iostream>
#include <type_traits>

// 1. 對於整數類型
template <typename T>
typename std::enable_if<std::is_integral<T>::value>::type
printValue(const T& val) {
    std::cout << "Integer: " << val << std::endl;
}

// 2. 對於字符串類型
template <typename T>
typename std::enable_if<std::is_same<T, std::string>::value>::type
printValue(const T& val) {
    std::cout << "String: " << val << std::endl;
}

int main() {
    printValue(42);           // 輸出: Integer: 42
    printValue(std::string("Hello")); // 輸出: String: Hello
    
    // printValue(3.14);      // 這一行會引起編譯錯誤，因爲沒有適合double類型的printValue版本
    return 0;
}
```

這一長串代碼確實有點醜陋了，我們將代碼拆開詳細看看。

```c++
// 節選自上面的代碼
template <typename T>
typename std::enable_if<std::is_integral<T>::value>::type
printValue(const T& val) {
    std::cout << "Integer: " << val << std::endl;
}
```

1. **模板聲明**:

   ```cpp
   template <typename T>
   ```

   聲明瞭一個模板函數，其中 `T` 是一個待定的類型。你可以爲 `T` 提供任何類型，比如 `int`、`double`、`std::string` 等，但是函數的實際行爲取決於你提供的類型。

2. **返回類型**:

   ```cpp
   typename std::enable_if<std::is_integral<T>::value>::type
   ```

   這段代碼使用了兩個主要的模板工具：`std::enable_if` 和 `std::is_integral`。

   - `std::is_integral<T>::value` 是一個類型特性，檢查 `T` 是否是整數類型。如果是，它返回 `true`；否則返回 `false`。

   - `std::enable_if` 是一個模板，它有一個嵌套的 `type` 成員，但這個成員只在給定的布爾表達式爲 `true` 時存在。在這裏，它檢查前面的 `std::is_integral<T>::value` 是否爲 `true`。

   結合起來，這意味着：

   - 如果 `T` 是整數類型，函數的返回類型將是 `void`（因爲 `std::enable_if` 的默認類型是 `void`）。
   - 如果 `T` 不是整數類型，由於 `type` 成員不存在，SFINAE 將阻止此函數模板被實例化，因此該版本的 `printValue` 函數將不可用。

如果我想讓當函數傳入int類型時輸出double類型，可以這樣做：

```c++
template <typename T>
typename std::enable_if<std::is_same<T, int>::value, double>::type
printValue(const T& val) {
    std::cout << "Integer: " << val << std::endl;
    return static_cast<double>(val);
}

int main() {
    double result = printValue(42);
    std::cout << "Returned value: " << result << std::endl;
    // print 42.0
}
```

關鍵部分是 `typename std::enable_if<std::is_same<T, int>::value, double>::type`，這會檢查 `T` 是否與 `int` 相同。如果是，它將產生類型 `double`。如果不是，該版本的 `printValue` 函數將由於 SFINAE 而不被考慮。

有朋友可能會說，爲什麼不用多態呢？寫這坨代碼實在是太難看了，我用多態寫那叫一個簡潔：

```c++
double printValue(int val) {
    std::cout << "Integer: " << val << std::endl;
    return static_cast<double>(val);
}

void printValue(double val) {
    std::cout << "Double: " << val << std::endl;
}
```

以下是一些常見的解釋：

1. **泛型編程**: 使用模板，你可以爲各種類型編寫通用的代碼，而不僅僅是那些你預先知道的類型。
2. **類型約束**: 通過 SFINAE 和其他模板技巧，你可以對哪些類型可以用於你的泛型代碼施加更精細的約束。例如，你可能想要一個函數，它只接受具有某些成員函數的對象。
3. **編譯時優化**: 由於模板在編譯時實例化，編譯器可以爲每個特定的類型生成優化過的代碼，這可能會導致更高的執行效率。
4. **靈活性**: 模板提供了更多的靈活性，例如模板元編程、模板特化等，允許更復雜和高效的編程技術。
5. **類型透明性**: 當使用模板時，原始類型信息在使用模板函數或類的地方保持不變。這與多態不同，其中類型信息可能會丟失，特別是在使用繼承和虛函數時。

隨着進一步學習以及項目的接觸，我們可以更加體會到這種編程方式的優缺點。

### 第十關：模板與友元

模板類或函數可以聲明爲另一個類或函數的友元。

```c++
template <typename T>
class Container {
private:
    T data;
public:
    Container(T d) : data(d) {}
    template <typename U>
    friend bool operator==(const Container<U>&, const Container<U>&);
};

template <typename T>
bool operator==(const Container<T>& lhs, const Container<T>& rhs) {
    return lhs.data == rhs.data;
}
```

### 第十一關：摺疊表達式

C++17中的摺疊表達式可以簡化某些變長模板參數的操作。

例如，要計算所有給定參數的總和：

```c++
template<typename... Args>
auto sum(Args... args) {
    return (... + args);
}
```

### 第十二關：模板概念(Concepts) - C++20

C++20引入了模板的概念，允許你爲模板參數指定更明確的約束。只有滿足給定概念的類型纔可以作爲`print`函數的參數。

比如說，

```c++
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template <Arithmetic T>
T add(T a, T b) {
    return a + b;
}
```

這裏是其他的一些特性：

- `std::is_integral<T>`：檢查 `T` 是否是整數類型。
- `std::is_floating_point<T>`：檢查 `T` 是否是浮點數類型。
- `std::is_array<T>`：檢查 `T` 是否是數組。
- `std::is_pointer<T>`：檢查 `T` 是否是指針。
- `std::is_reference<T>`：檢查 `T` 是否是引用。
- `std::is_class<T>`：檢查 `T` 是否是類或結構體類型。
- `std::is_function<T>`：檢查 `T` 是否是函數。

另外還可以通過其他方法檢查“一個類型是否可以被輸出流輸出”。也就是在下面代碼中，我們定義了一個`Printable`的conecpt，要滿足這個概念，類型 `T` 必須滿足 `requires` 表達式中的要求。

```c++
template <typename T>
concept Printable = requires(T t) {
    { std::cout << t } -> std::same_as<std::ostream&>;
};

template <Printable T>
void print(T value) {
    std::cout << value;
}
```

其中， `requires` 表達式是與**概念** (concepts) 相關的一種新特性，用於描述一個類型必須滿足的要求。

```c++
requires ( 參數 ) { 要求列表 }
```

在這裏，我們要求類型 `T` 必須支持一個操作，即：當你嘗試將 `t` 輸出到 `std::cout` 時，結果的類型必須是 `std::ostream&`。在 `requires` 表達式中，`->` 符號被用於指定一個表達式的預期返回類型。

另外注意，`requires` 表達式是在編譯階段處理的。

###  第十三關： `std::enable_if` 和 SFINAE

上面我們已經有所提及，當我們希望根據某種條件來決定是否生成模板函數或類時，`std::enable_if`非常有用。

例如，假設你有一個函數，你只希望當傳入的類型是整數時，它才存在：

```c++
template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
functionOnlyForIntegers(T value) {
    return value * 2;
}
```

### 第十四關：類模板偏特化

現在我們從頭開始梳理一遍類模板。假設我們有以下基本模板。

```c++
template <typename T1, typename T2>
class MyPair {
    T1 first;
    T2 second;
    // ... 其他成員函數 ...
};
```

接下來，對類模板偏特化。假設我們想爲第二個模板參數是指針類型的所有情況提供特化。這裏的"偏"意味着我們不是爲兩個特定的類型提供特化，而是隻爲一個類型（這裏是 T2）提供。

```c++
template <typename T1, typename T2>
class MyPair<T1, T2*> {
    T1 first;
    T2* second;
    // ... 其他成員函數 ...
};
```

需要注意的是，函數模板不支持偏特化，但可以通過重載來達到類似的效果。

### 第十五關：`constexpr` 和模板

`constexpr` 是 C++11 引入的關鍵字，它用於聲明常量表達式，這些表達式在編譯時就可以計算出結果。使用`constexpr`與模板一起可以在編譯時生成高效的代碼。譬如下面的例子。

```c++
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}
constexpr int value = factorial(5);  // 120
```

那麼結合 `constexpr` 和模板的例子是啥樣的？當 `constexpr` 與模板結合使用時，你可以爲各種類型創建編譯時函數或實體，它們將針對給定的類型進行優化，並在編譯時生成結果。

```c++
template <typename T>
constexpr T square(const T& value) {
    return value * value;
}

constexpr int int_val = square(5);        // 25
constexpr double double_val = square(5.0); // 25.0
```

兩者結合的優勢很大，我這裏列出兩點：

- **性能**：結合 `constexpr` 和模板，生成的代碼是在編譯時優化的，這可以消除運行時計算的需要，從而提高性能。
- **泛型**：模板使你可以爲多種類型編寫代碼，而 `constexpr` 確保了對每種類型的高效實現。

### 第十六關：模板中的嵌套類型

一個模板可以在其內部定義另一個模板類：

```c++
template <typename T>
class Outer {
    T data;
public:
    template <typename U>
    class Inner {
        U innerData;
    };
};
```

接下來，讓我們給 `Outer` 和 `Inner` 類添加一些成員函數，使它們更具功能性。

```c++
template <typename T>
class Outer {
    T data;
public:
    Outer(T d) : data(d) {}

    T getOuterData() const { return data; }

    template <typename U>
    class Inner {
        U innerData;
    public:
        Inner(U d) : innerData(d) {}

        U getInnerData() const { return innerData; }
    };
};
```

使用示例：

```c++
Outer<int> outerInstance(10);
std::cout << "Outer data: " << outerInstance.getOuterData() << std::endl; 
// Outputs: Outer data: 10

Outer<int>::Inner<double> innerInstance(5.5);
std::cout << "Inner data: " << innerInstance.getInnerData() << std::endl; 
// Outputs: Inner data: 5.5
```

進一步添加功能，在 `Outer` 類中定義一個函數，該函數接受一個 `Inner` 對象並與之交互。

```c++
template <typename T>
class Outer {
    T data;
public:
    Outer(T d) : data(d) {}

    T getOuterData() const { return data; }
    
	// ---- template ----
    template <typename U>
    class Inner {
        U innerData;
    public:
        Inner(U d) : innerData(d) {}

        U getInnerData() const { return innerData; }
    };
    // ---- -------- ----
    
    template <typename U>
    void printCombinedData(const Inner<U>& inner) {
        std::cout << "Combined data: " << data << " and " << inner.getInnerData() << std::endl;
    }
};

// 使用：
Outer<int> outerInstance(10);
Outer<int>::Inner<double> innerInstance(5.5);
outerInstance.printCombinedData(innerInstance);  // Outputs: Combined data: 10 and 5.5
```

總之需要知道，外部類完全可以訪問其內部類及其成員，但它需要擁有內部類的對象實例才能訪問內部類的非靜態成員。

### 第十七關：模板參數包與展開

當使用變長模板參數時，你可以使用模板參數包。使用`...`修飾的參數被稱爲參數包。

```c++
template <typename... Args>
void printValues(Args... args) {
    (std::cout << ... << args);  // 展開參數
}

int main() {
    printValues(1, 2, 3, "hello", 'c');
    //Same As ： std::cout << 1 << 2 << 3 << "hello" << 'c';
}
```

如果要用多態來實現上面的效果，將會變得比較複雜。需要爲每一種要輸出的類型創建一個公共的基類並實現虛函數。然後爲每種具體的類型實現一個子類。下面是用多態來實現的，可以看出模版參數包的優越性了吧。

```c++
#include <iostream>
#include <vector>

class Printable {
public:
    virtual ~Printable() {}
    virtual void print() const = 0;
};

class PrintInt : public Printable {
    int value;
public:
    PrintInt(int v) : value(v) {}
    void print() const override {
        std::cout << value;
    }
};

class PrintString : public Printable {
    std::string value;
public:
    PrintString(const std::string& v) : value(v) {}
    void print() const override {
        std::cout << value;
    }
};

class PrintChar : public Printable {
    char value;
public:
    PrintChar(char v) : value(v) {}
    void print() const override {
        std::cout << value;
    }
};

void printValues(const std::vector<Printable*>& values) {
    for (const auto& val : values) {
        val->print();
    }
}

int main() {
    std::vector<Printable*> values = {new PrintInt(1), new PrintInt(2), new PrintInt(3), new PrintString("hello"), new PrintChar('c')};
    printValues(values);

    // Cleaning up
    for (auto ptr : values) {
        delete ptr;
    }
}
```

還記得 十一關 講解的摺疊表達式嗎？摺疊表達式是 C++17 引入的，是一種新的、更簡潔的方式來展開參數包，並對其應用特定的運算。在 C++17 之前，當需要在模板中使用參數包的時候，通常需要使用某種機制對其進行展開。在 C++11 和 C++14 中，展開參數包通常涉及到遞歸的模板技巧。例如，

```c++
template <typename T>
void printValues(T value) {
    std::cout << value << std::endl;
}
template <typename First, typename... Rest>
void printValues(First first, Rest... rest) {
    std::cout << first << ", ";
    printValues(rest...);  // 展開剩餘的參數
}
// 使用
int main() {
    printValues(1, 2, 3);        // 輸出: 1, 2, 3
    printValues("a", "b", "c");  // 輸出: a, b, c
    return 0;
}
```

而使用了摺疊表達式，就不用涉及遞歸輸出了，上下兩則代碼完全一致。

``` c++
template <typename... Args>
void printValues(Args... args) {
    (std::cout << ... << args);
}
// 使用
int main() {
    printValues(1, 2, 3, "hello", 'c');  // 輸出：123helloc
    return 0;
}
```

### 第十八關：Lambda 表達式與模板

```c++
auto lambda = []<typename T>(T value) { return value * 2; };
auto result = lambda(5);  // result爲10
```

進一步添加“概念”，以確保類型是可計算的。這裏直接使用了`std::is_arithmetic_v`。

```c++
#include <iostream>
#include <type_traits>

int main() {
    auto genericLambda = [](auto x) {
        static_assert(std::is_arithmetic_v<decltype(x)>, "Type must be arithmetic!");
        return x * x;
    };

    std::cout << genericLambda(5) << std::endl;    // 輸出：25
    std::cout << genericLambda(5.5) << std::endl;  // 輸出：30.25

    // genericLambda("hello"); // 編譯錯誤：Type must be arithmetic!
}
```

### 第十九關：模板遞歸

模板遞歸是一種非常強大的技巧，但也需要謹慎使用，因爲它可能導致編譯時間增加和代碼膨脹。

在前面我們已經見識到了模版的強大。例如，計算階乘或斐波那契數列，直接在編譯期間就可以完成計算，減少運行時的計算量。

```c++
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}
constexpr int value = factorial(5);  // 120
```

### 第二十關：帶有模板的繼承

類模板可以繼承自其他類模板。下面是一個最簡單的例子，我們逐漸完善他。

```c++
template <typename T>
class Base {};

template <typename T>
class Derived : public Base<T> {};
```

#### 1. 模版基類

可以創建一個模板基類，使得不同的子類可以以不同的方式特化或使用這個基類。

```c++
template <typename T>
class Base {
public:
    T value;
    Base(T val) : value(val) {}
    void show() { std::cout << value << std::endl; }
};

class Derived : public Base<int> {
public:
    Derived(int v) : Base(v) {}
    void display() { std::cout << "Derived: " << value << std::endl; }
};

int main() {
    Derived d(10);
    d.show();
    d.display();
}
```

#### 2. 模版子類

可以使子類是模板，而基類不是。這樣，就可以爲基類定義一組行爲，而子類則爲這些行爲提供具體的實現。

```c++
class Base {
public:
    virtual void show() const = 0;
};

template <typename T>
class Derived : public Base {
    T value;
public:
    Derived(T v) : value(v) {}
    void show() const override {
        std::cout << "Value: " << value << std::endl;
    }
};

int main() {
    Derived<int> d1(5);
    Derived<double> d2(3.14);
    d1.show();
    d2.show();
}
```



#### 3. 在模板類中繼承模板基類

子類和基類都可以是模板，這樣你可以創建高度靈活和可重用的設計。

```c++
template <typename T>
class Base {
public:
    T value;
    Base(T val) : value(val) {}
    virtual void show() const {
        std::cout << "Base: " << value << std::endl;
    }
};

template <typename T>
class Derived : public Base<T> {
public:
    Derived(T v) : Base<T>(v) {}
    void show() const override {
        std::cout << "Derived: " << this->value << std::endl;
    }
};

int main() {
    Derived<int> d(10);
    d.show();
}
```

### 第二十一關：`std::type_trait`的工具集

`<type_traits>`頭文件提供了一組用於類型檢查和修改的模板，可以在編譯時獲取和操作類型的信息。

```c++
static_assert(std::is_same<std::remove_const<const int>::type, int>::value);
```

以下是 `std::type_traits` 中一些常用的工具：

1. **基礎類型檢查**:
    - `std::is_integral<T>`: 檢查T是否是一個整數類型。
    - `std::is_floating_point<T>`: 檢查T是否是一個浮點類型。
    - `std::is_arithmetic<T>`: 檢查T是否是算術類型（整數或浮點數）。
    - `std::is_pointer<T>`: 檢查T是否是指針。
    - `std::is_reference<T>`: 檢查T是否是引用。
    - `std::is_array<T>`: 檢查T是否是數組。
    - `std::is_enum<T>`: 檢查T是否是枚舉類型。

2. **類型關係檢查**:
    - `std::is_same<T, U>`: 檢查兩個類型是否完全相同。
    - `std::is_base_of<Base, Derived>`: 檢查`Base`是否是`Derived`的基類。
    - `std::is_convertible<T, U>`: 檢查類型T是否可以被隱式轉換爲U。

3. **類型修改器**:
    - `std::remove_reference<T>`: 去除引用，得到裸類型。
    - `std::add_pointer<T>`: 爲類型T添加一個指針。
    - `std::remove_pointer<T>`: 去除指針。
    - `std::remove_const<T>`: 去除常量限定符。
    - `std::add_const<T>`: 添加常量限定符。

4. **其他**:
    - `std::underlying_type<T>`: 對於枚舉類型T，得到對應的底層類型。
    - `std::result_of<F(Args...)>`: 對於函數類型F，返回它使用參數`Args...`調用時的返回類型。

5. **輔助類型**:
    - 對於上述的每個特性檢查，都有一個對應的`_v`後綴的變量模板，如`std::is_integral_v<T>`，它直接返回bool值，這使得代碼更簡潔。

```c++
static_assert(std::is_same<std::remove_const<const int>::type, int>::value);
static_assert(std::is_integral_v<int>);
```

### 第二十二關：模板與動態多態性

儘管模板提供了一種靜態多態性形式，但它們也可以與虛函數和動態多態性結合使用。

```c++
#include <iostream>
#include <vector>

class Base {
public:
    virtual void print() const {
        std::cout << "Base class." << std::endl;
    }

    virtual ~Base() {}
};

class Derived1 : public Base {
public:
    void print() const override {
        std::cout << "Derived1 class." << std::endl;
    }
};

class Derived2 : public Base {
public:
    void print() const override {
        std::cout << "Derived2 class." << std::endl;
    }
};

template <typename T>
class Container {
private:
    std::vector<T*> elements;

public:
    void add(T* elem) {
        elements.push_back(elem);
    }

    void printAll() const {
        for (auto& elem : elements) {
            elem->print();
        }
    }
};

int main() {
    Container<Base> cont;
    Derived1 d1;
    Derived2 d2;

    cont.add(&d1);
    cont.add(&d2);

    cont.printAll();  // Outputs: Derived1 class. Derived2 class.

    return 0;
}

```















----

## 備註/聲明

1. 本文使用的模型數據由 [Vidar Rapp](https://se.linkedin.com/in/vidarrapp) 提供。
2. 本文框架基於https://github.com/ssloy/tinyrenderer。