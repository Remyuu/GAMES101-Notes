- 作業框架「上下顛倒」的解決辦法

  - $\mathbf{M}_{\mathrm{persp}}$ 改為  $\mathbf{M}_{\text {OpenGL }}$

- GAMES101 Assignment2題解概要：

  1. 三角形栅格化算法

  2. 测试点是否在三角形内

  3. z-buffer 算法

  4. super-sampling 处理 Anti-aliasing。即SSAA（超采样抗锯齿算法）。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/tbBxkiwj2cfaSJd.png" alt="image-20230527130132554" style="zoom:20%;" />

本文將詳細講解代碼所有細節。


<!--more-->

这个作业二不是特别简单，我大概弄了一整天吧。。。

----

## 題目

> 在上次作业中，虽然我们在屏幕上画出一个线框三角形，但这看起来并不是 那么的有趣。所以这一次我们继续推进一步——在屏幕上画出一个实心三角形， 换言之，栅格化一个三角形。上一次作业中，在视口变化之后，我们调用了函数 rasterize_wireframe(const Triangle& t)。但这一次，你需要自己填写并调用 函数 rasterize_triangle(const Triangle& t)。
>
> **该函数的内部工作流程如下:**
>
> 1. 创建三角形的 2 维 bounding box。
> 2. 遍历此 bounding box 内的所有像素(使用其整数索引)。然后，使用像素中 心的屏幕空间坐标来检查中心点是否在三角形内。
> 3. 如果在内部，则将其位置处的插值深度值 (interpolated depth value) 与深度 缓冲区 (depth buffer) 中的相应值进行比较。
> 4. 如果当前点更靠近相机，请设置像素颜色并更新深度缓冲区 (depth buffer)。
>
> **你需要修改的函数如下:**
>
> - rasterize_triangle(): 执行三角形栅格化算法
>
> - static bool insideTriangle(): 测试点是否在三角形内。你可以修改此函 数的定义，这意味着，你可以按照自己的方式更新返回类型或函数参数。
>
> 因为我们只知道三角形三个顶点处的深度值，所以对于三角形内部的像素， 我们需要用插值的方法得到其深度值。我们已经为你处理好了这一部分，因为有 关这方面的内容尚未在课程中涉及。插值的深度值被储存在变量 z_interpolated 中。
>
> **请注意我们是如何初始化 depth buffer 和注意 z values 的符号。为了方便 同学们写代码，我们将 z 进行了反转，保证都是正数，并且越大表示离视点越远。**
>
> 在此次作业中，你无需处理旋转变换，只需为模型变换返回一个单位矩阵。最 后，我们提供了两个 hard-coded 三角形来测试你的实现。
>
> 在你自己的计算机或虚拟机上下载并使用我们更新的框架代码。你会注意到， 在 main.cpp 下的 get_projection_matrix() 函数是空的。请复制粘贴你在第一 次作业中的实现来填充该函数。
>
> ```
> Eigen::Matrix4f get_projection_matrix(float eye_fov, float aspect_ratio, float zNear, float zFar)
> {
>     // TODO: Copy-paste your implementation from the previous assignment.
>     Eigen::Matrix4f projection = Eigen::Matrix4f::Identity();
>     Eigen::Matrix4f M = Eigen::Matrix4f ::Identity();
>     float fov = eye_fov*MY_PI/180;
>     float top = tan(fov) * zNear;
>     float bottom = -top;
>     float right = top * aspect_ratio;
>     float left = -right;
>     M <<   2*abs(zNear)/(right-left),0,(right+left)/(right-left),0,
>            0,2*abs(zNear)/(top-bottom),(top+bottom)/(top-bottom),0,
>            0,0,(abs(zNear)+abs(zFar))/(abs(zNear)-abs(zFar)),2*abs(zFar*zNear)/(abs(zNear)-abs(zFar)),
>            0,0,-1,0;
>     return M;
> }
> ```

這裡需要注意的是，在閆老師上課時講解的zFar和zNear都是負數，但是作業框架的zFar和zNear是正數。因此，推導的變換矩陣會不一樣。

在Fundamentals of Computer Graphics, 4th 的第153頁有如下描述：

> Many APIs such as *OpenGL* (Shreiner, Neider, Woo, & Davis, 2004) use the same canonical view volume as presented here. They also usually have the user specify the absolute values of n and f . The projection matrix for *OpenGL* is
> $$
> \mathbf{M}_{\text {OpenGL }}=\left[\begin{array}{cccc}
> \frac{2|n|}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
> 0 & \frac{2|n|}{t-b} & \frac{t+b}{t-b} & 0 \\
> 0 & 0 & \frac{|n|+|f|}{|n|-|f|} & \frac{2|f||n|}{|n|-|f|} \\
> 0 & 0 & -1 & 0
> \end{array}\right]
> $$
> Other APIs send n and f to 0 and 1, respectively. Blinn (J. Blinn, 1996) recom- mends making the canonical view volume $[0, 1]^3$ for efficiency. All such decisions will change the the projection matrix slightly.

也就是說，將
$$
\mathbf{M}_{\mathrm{persp}}=\left[\begin{array}{cccc}
\frac{2 n}{r-l} & 0 & \frac{l+r}{l-r} & 0 \\
0 & \frac{2 n}{t-b} & \frac{b+t}{b-t} & 0 \\
0 & 0 & \frac{f+n}{n-f} & \frac{2 f n}{f-n} \\
0 & 0 & 1 & 0
\end{array}\right]
$$
改為
$$
\mathbf{M}_{\text {OpenGL }}=\left[\begin{array}{cccc}
\frac{2|n|}{r-l} & 0 & \frac{r+l}{r-l} & 0 \\
0 & \frac{2|n|}{t-b} & \frac{t+b}{t-b} & 0 \\
0 & 0 & \frac{|n|+|f|}{|n|-|f|} & \frac{2|f||n|}{|n|-|f|} \\
0 & 0 & -1 & 0
\end{array}\right]
$$



![image-20230525235944363](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/CnTOLiBcDfXA6Uu.png)

作业要求：

- [5 分] 正确地提交所有必须的文件，且代码能够编译运行。
- [20 分] 正确实现三角形栅格化算法。
- [10 分] 正确测试点是否在三角形内。
- [10 分] 正确实现 z-buffer 算法, 将三角形按顺序画在屏幕上。
- [提高项 5 分] 用 super-sampling 处理 Anti-aliasing : 你可能会注意 到，当我们放大图像时，图像边缘会有锯齿感。我们可以用 super-sampling 来解决这个问题，即对每个像素进行 2 * 2 采样，并比较前后的结果 (这里 并不需要考虑像素与像素间的样本复用)。需要注意的点有，对于像素内的每 一个样本都需要维护它自己的深度值，即每一个像素都需要维护一个 sample list。最后，如果你实现正确的话，你得到的三角形不应该有不正常的黑边。

----

## rasterize_triangle()函數

需要填充的函數：

```c++
//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t) {
    auto v = t.toVector4();
    
    // TODO : Find out the bounding box of current triangle.
    // iterate through the pixel and find if the current pixel is inside the triangle

    // If so, use the following code to get the interpolated z value.
    //auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
    //float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    //float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    //z_interpolated *= w_reciprocal;

    // TODO : set the current pixel (use the set_pixel function) to the color of the triangle (use getColor function) if it should be painted.


}
```

- 先找到包圍盒子bounding box
- 在bounding box中檢測像素是否在三角形內
- 如果在，則
  - 用重心座標**插值**得到当前三角形像素的z值
    - 判斷z值与depth buffer的关系
    - z<depth buffer
      - 設置當前像素顏色

```c++
//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t) {
    auto v = t.toVector4();
    
    // TODO : Find out the bounding box of current triangle.
    // iterate through the pixel and find if the current pixel is inside the triangle

    // If so, use the following code to get the interpolated z value.
    //auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
    //float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    //float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    //z_interpolated *= w_reciprocal;

    // TODO : set the current pixel (use the set_pixel function) to the color of the triangle (use getColor function) if it should be painted.

    // Finding out the bounding box for the current triangle
    float min_x = ceil(std::min({v[0].x(), v[1].x(), v[2].x()}));
    float max_x = ceil(std::max({v[0].x(), v[1].x(), v[2].x()}));
    float min_y = ceil(std::min({v[0].y(), v[1].y(), v[2].y()}));
    float max_y = ceil(std::max({v[0].y(), v[1].y(), v[2].y()}));

    // Iterating through each pixel in the bounding box
    for (float x = min_x; x <= max_x; x++) {
        for (float y = min_y; y <= max_y; y++) {

            // If the pixel is inside the triangle
            if (insideTriangle(x+0.5f, y+0.5f, t.v)) {
                auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
                float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
                float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
                z_interpolated *= w_reciprocal;
                if (z_interpolated < depth_buf[get_index(x, y)]) {
                    // Set the pixel color if it should be painted
                    set_pixel(Eigen::Vector3f(x, y, z_interpolated), t.getColor());
                    depth_buf[get_index(x, y)] = z_interpolated;
                }
            }
        }
    }
}
```

另外，对于作业框架给出的计算z轴插值算法，使用的是重心坐标法。详细内容请查看我的另一篇关于常见插值算法的文章。https://remoooo.com/cg/837.html



## insideTriangle函数

这个函数传入像素的坐标（屏幕坐标），以及三角形的三个顶点坐标`_v[0] _v[1] _v[2}`。

要求判断该像素是否在这个三个顶点围成的三角形中。

```c++
static bool insideTriangle(int x, int y, const Vector3f* _v)
{   
    // TODO : Implement this function to check if the point (x, y) is inside the triangle represented by _v[0], _v[1], _v[2]

    // Compute vectors
    Vector2f v0 = _v[2].head<2>() - _v[0].head<2>();
    Vector2f v1 = _v[1].head<2>() - _v[0].head<2>();
    Vector2f v2 = Vector2f(x, y) - _v[0].head<2>();

    // Compute dot products
    float dot00 = v0.dot(v0);
    float dot01 = v0.dot(v1);
    float dot02 = v0.dot(v2);
    float dot11 = v1.dot(v1);
    float dot12 = v1.dot(v2);

    // Compute barycentric coordinates
    float invDenom = 1.0 / (dot00 * dot11 - dot01 * dot01);
    float u = (dot11 * dot02 - dot01 * dot12) * invDenom;
    float v = (dot00 * dot12 - dot01 * dot02) * invDenom;

    // Check if point is in triangle
    return (u >= 0) && (v >= 0) && (u + v < 1);

}
```

我这里使用的算法是 **重心坐标插值（Barycentric Interpolation）** ，关于算法原理请浏览我另一篇文章。https://remoooo.com/cg/835.html

rasterizer.cpp代码：

```c++
// clang-format off
//
// Created by goksu on 4/6/19.
//

#include <algorithm>
#include <vector>
#include "rasterizer.hpp"
#include <opencv2/opencv.hpp>
#include <math.h>

rst::pos_buf_id rst::rasterizer::load_positions(const std::vector<Eigen::Vector3f> &positions)
{
    auto id = get_next_id();
    pos_buf.emplace(id, positions);

    return {id};
}

rst::ind_buf_id rst::rasterizer::load_indices(const std::vector<Eigen::Vector3i> &indices)
{
    auto id = get_next_id();
    ind_buf.emplace(id, indices);

    return {id};
}

rst::col_buf_id rst::rasterizer::load_colors(const std::vector<Eigen::Vector3f> &cols)
{
    auto id = get_next_id();
    col_buf.emplace(id, cols);

    return {id};
}

auto to_vec4(const Eigen::Vector3f& v3, float w = 1.0f)
{
    return Vector4f(v3.x(), v3.y(), v3.z(), w);
}


static bool insideTriangle(int x, int y, const Vector3f* _v)
{
    // TODO : Implement this function to check if the point (x, y) is inside the triangle represented by _v[0], _v[1], _v[2]

    // Compute vectors
    Vector2f v0 = _v[2].head<2>() - _v[0].head<2>();
    Vector2f v1 = _v[1].head<2>() - _v[0].head<2>();
    Vector2f v2 = Vector2f(x, y) - _v[0].head<2>();

    // Compute dot products
    float dot00 = v0.dot(v0);
    float dot01 = v0.dot(v1);
    float dot02 = v0.dot(v2);
    float dot11 = v1.dot(v1);
    float dot12 = v1.dot(v2);

    // Compute barycentric coordinates
    float invDenom = 1.0f / (dot00 * dot11 - dot01 * dot01);
    float u = (dot11 * dot02 - dot01 * dot12) * invDenom;
    float v = (dot00 * dot12 - dot01 * dot02) * invDenom;

    // Check if point is in triangle
    return (u >= 0) && (v >= 0) && (u + v < 1);

}

static std::tuple<float, float, float> computeBarycentric2D(float x, float y, const Vector3f* v)
{
    float c1 = (x*(v[1].y() - v[2].y()) + (v[2].x() - v[1].x())*y + v[1].x()*v[2].y() - v[2].x()*v[1].y()) / (v[0].x()*(v[1].y() - v[2].y()) + (v[2].x() - v[1].x())*v[0].y() + v[1].x()*v[2].y() - v[2].x()*v[1].y());
    float c2 = (x*(v[2].y() - v[0].y()) + (v[0].x() - v[2].x())*y + v[2].x()*v[0].y() - v[0].x()*v[2].y()) / (v[1].x()*(v[2].y() - v[0].y()) + (v[0].x() - v[2].x())*v[1].y() + v[2].x()*v[0].y() - v[0].x()*v[2].y());
    float c3 = (x*(v[0].y() - v[1].y()) + (v[1].x() - v[0].x())*y + v[0].x()*v[1].y() - v[1].x()*v[0].y()) / (v[2].x()*(v[0].y() - v[1].y()) + (v[1].x() - v[0].x())*v[2].y() + v[0].x()*v[1].y() - v[1].x()*v[0].y());
    return {c1,c2,c3};
}

void rst::rasterizer::draw(pos_buf_id pos_buffer, ind_buf_id ind_buffer, col_buf_id col_buffer, Primitive type)
{
    auto& buf = pos_buf[pos_buffer.pos_id];
    auto& ind = ind_buf[ind_buffer.ind_id];
    auto& col = col_buf[col_buffer.col_id];

    float f1 = (50 - 0.1) / 2.0;
    float f2 = (50 + 0.1) / 2.0;

    Eigen::Matrix4f mvp = projection * view * model;
    for (auto& i : ind)
    {
        Triangle t;
        Eigen::Vector4f v[] = {
                mvp * to_vec4(buf[i[0]], 1.0f),
                mvp * to_vec4(buf[i[1]], 1.0f),
                mvp * to_vec4(buf[i[2]], 1.0f)
        };
        //Homogeneous division
        for (auto& vec : v) {
            vec /= vec.w();
        }
        //Viewport transformation
        for (auto & vert : v)
        {
            vert.x() = 0.5*width*(vert.x()+1.0);
            vert.y() = 0.5*height*(vert.y()+1.0);
            vert.z() = vert.z() * f1 + f2;
        }

        for (int i = 0; i < 3; ++i)
        {
            t.setVertex(i, v[i].head<3>());
            t.setVertex(i, v[i].head<3>());
            t.setVertex(i, v[i].head<3>());
        }

        auto col_x = col[i[0]];
        auto col_y = col[i[1]];
        auto col_z = col[i[2]];

        t.setColor(0, col_x[0], col_x[1], col_x[2]);
        t.setColor(1, col_y[0], col_y[1], col_y[2]);
        t.setColor(2, col_z[0], col_z[1], col_z[2]);

        rasterize_triangle(t);
    }
}

//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t) {
    auto v = t.toVector4();
    
    // TODO : Find out the bounding box of current triangle.
    // iterate through the pixel and find if the current pixel is inside the triangle

    // If so, use the following code to get the interpolated z value.
    //auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
    //float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    //float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    //z_interpolated *= w_reciprocal;

    // TODO : set the current pixel (use the set_pixel function) to the color of the triangle (use getColor function) if it should be painted.

    // Finding out the bounding box for the current triangle
    float min_x = ceil(std::min({v[0].x(), v[1].x(), v[2].x()}));
    float max_x = ceil(std::max({v[0].x(), v[1].x(), v[2].x()}));
    float min_y = ceil(std::min({v[0].y(), v[1].y(), v[2].y()}));
    float max_y = ceil(std::max({v[0].y(), v[1].y(), v[2].y()}));

    // Iterating through each pixel in the bounding box
    for (float x = min_x; x <= max_x; x++) {
        for (float y = min_y; y <= max_y; y++) {

            // If the pixel is inside the triangle
            if (insideTriangle(x, y, t.v)) {
                auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
                float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
                float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
                z_interpolated *= w_reciprocal;
                if (z_interpolated < depth_buf[get_index(x, y)]) {
                    // Set the pixel color if it should be painted
                    set_pixel(Eigen::Vector3f(x, y, z_interpolated), t.getColor());
                    depth_buf[get_index(x, y)] = z_interpolated;
                }
            }
        }
    }
}

void rst::rasterizer::set_model(const Eigen::Matrix4f& m)
{
    model = m;
}

void rst::rasterizer::set_view(const Eigen::Matrix4f& v)
{
    view = v;
}

void rst::rasterizer::set_projection(const Eigen::Matrix4f& p)
{
    projection = p;
}

void rst::rasterizer::clear(rst::Buffers buff)
{
    if ((buff & rst::Buffers::Color) == rst::Buffers::Color)
    {
        std::fill(frame_buf.begin(), frame_buf.end(), Eigen::Vector3f{0, 0, 0});
    }
    if ((buff & rst::Buffers::Depth) == rst::Buffers::Depth)
    {
        std::fill(depth_buf.begin(), depth_buf.end(), std::numeric_limits<float>::infinity());
    }
}

rst::rasterizer::rasterizer(int w, int h) : width(w), height(h)
{
    frame_buf.resize(w * h);
    depth_buf.resize(w * h);
}

int rst::rasterizer::get_index(int x, int y)
{
    return (height-1-y)*width + x;
}

void rst::rasterizer::set_pixel(const Eigen::Vector3f& point, const Eigen::Vector3f& color)
{
    //old index: auto ind = point.y() + point.x() * width;
    auto ind = (height-1-point.y())*width + point.x();
    frame_buf[ind] = color;

}

// clang-format on
```

Rasteriser.hpp代码：

```c++
//
// Created by goksu on 4/6/19.
//

#pragma once

#include <eigen3/Eigen/Eigen>
#include <algorithm>
#include "global.hpp"
#include "Triangle.hpp"
using namespace Eigen;

namespace rst
{
    enum class Buffers
    {
        Color = 1,
        Depth = 2
    };

    inline Buffers operator|(Buffers a, Buffers b)
    {
        return Buffers((int)a | (int)b);
    }

    inline Buffers operator&(Buffers a, Buffers b)
    {
        return Buffers((int)a & (int)b);
    }

    enum class Primitive
    {
        Line,
        Triangle
    };

    /*
     * For the curious : The draw function takes two buffer id's as its arguments. These two structs
     * make sure that if you mix up with their orders, the compiler won't compile it.
     * Aka : Type safety
     * */
    struct pos_buf_id
    {
        int pos_id = 0;
    };

    struct ind_buf_id
    {
        int ind_id = 0;
    };

    struct col_buf_id
    {
        int col_id = 0;
    };

    class rasterizer
    {
    public:
        rasterizer(int w, int h);
        pos_buf_id load_positions(const std::vector<Eigen::Vector3f>& positions);
        ind_buf_id load_indices(const std::vector<Eigen::Vector3i>& indices);
        col_buf_id load_colors(const std::vector<Eigen::Vector3f>& colors);

        void set_model(const Eigen::Matrix4f& m);
        void set_view(const Eigen::Matrix4f& v);
        void set_projection(const Eigen::Matrix4f& p);

        void set_pixel(const Eigen::Vector3f& point, const Eigen::Vector3f& color);

        void clear(Buffers buff);

        void draw(pos_buf_id pos_buffer, ind_buf_id ind_buffer, col_buf_id col_buffer, Primitive type);

        std::vector<Eigen::Vector3f>& frame_buffer() { return frame_buf; }

    private:
        void draw_line(Eigen::Vector3f begin, Eigen::Vector3f end);

        void rasterize_triangle(const Triangle& t);

        // VERTEX SHADER -> MVP -> Clipping -> /.W -> VIEWPORT -> DRAWLINE/DRAWTRI -> FRAGSHADER

    private:
        Eigen::Matrix4f model;
        Eigen::Matrix4f view;
        Eigen::Matrix4f projection;

        std::map<int, std::vector<Eigen::Vector3f>> pos_buf;
        std::map<int, std::vector<Eigen::Vector3i>> ind_buf;
        std::map<int, std::vector<Eigen::Vector3f>> col_buf;

        std::vector<Eigen::Vector3f> frame_buf;

        std::vector<float> depth_buf;
        int get_index(int x, int y);

        int width, height;

        int next_id = 0;
        int get_next_id() { return next_id++; }
    };
}

```

Main.cpp代码：

```c++
// clang-format off
#include <iostream>
#include <opencv2/opencv.hpp>
#include "rasterizer.hpp"
#include "global.hpp"
#include "Triangle.hpp"

constexpr double MY_PI = 3.1415926;

Eigen::Matrix4f get_view_matrix(Eigen::Vector3f eye_pos)
{
    Eigen::Matrix4f view = Eigen::Matrix4f::Identity();

    Eigen::Matrix4f translate;
    translate << 1,0,0,-eye_pos[0],
                 0,1,0,-eye_pos[1],
                 0,0,1,-eye_pos[2],
                 0,0,0,1;

    view = translate*view;

    return view;
}

Eigen::Matrix4f get_model_matrix(float rotation_angle)
{
    Eigen::Matrix4f model = Eigen::Matrix4f::Identity();
    return model;
}

Eigen::Matrix4f get_projection_matrix(float eye_fov, float aspect_ratio, float zNear, float zFar)
{
    // TODO: Copy-paste your implementation from the previous assignment.
    Eigen::Matrix4f projection = Eigen::Matrix4f::Identity();
    Eigen::Matrix4f M = Eigen::Matrix4f ::Identity();
    float fov = eye_fov*MY_PI/180;
    float top = tan(fov) * zNear;
    float bottom = -top;
    float right = top * aspect_ratio;
    float left = -right;
    M <<   2*abs(zNear)/(right-left),0,(right+left)/(right-left),0,
           0,2*abs(zNear)/(top-bottom),(top+bottom)/(top-bottom),0,
           0,0,(abs(zNear)+abs(zFar))/(abs(zNear)-abs(zFar)),2*abs(zFar*zNear)/(abs(zNear)-abs(zFar)),
           0,0,-1,0;
    return M;
}

int main(int argc, const char** argv)
{
    float angle = 0;
    bool command_line = false;
    std::string filename = "output.png";

    if (argc == 2)
    {
        command_line = true;
        filename = std::string(argv[1]);
    }

    rst::rasterizer r(700, 700);

    Eigen::Vector3f eye_pos = {0,0,5};


    std::vector<Eigen::Vector3f> pos
            {
                    {2, 0, -2},
                    {0, 2, -2},
                    {-2, 0, -2},
                    {3.5, -1, -5},
                    {2.5, 1.5, -5},
                    {-1, 0.5, -5}
            };

    std::vector<Eigen::Vector3i> ind
            {
                    {0, 1, 2},
                    {3, 4, 5}
            };

    std::vector<Eigen::Vector3f> cols
            {
                    {217.0, 238.0, 185.0},
                    {217.0, 238.0, 185.0},
                    {217.0, 238.0, 185.0},
                    {185.0, 217.0, 238.0},
                    {185.0, 217.0, 238.0},
                    {185.0, 217.0, 238.0}
            };

    auto pos_id = r.load_positions(pos);
    auto ind_id = r.load_indices(ind);
    auto col_id = r.load_colors(cols);

    int key = 0;
    int frame_count = 0;

    if (command_line)
    {
        r.clear(rst::Buffers::Color | rst::Buffers::Depth);

        r.set_model(get_model_matrix(angle));
        r.set_view(get_view_matrix(eye_pos));
        r.set_projection(get_projection_matrix(45, 1, 0.1, 50));

        r.draw(pos_id, ind_id, col_id, rst::Primitive::Triangle);
        cv::Mat image(700, 700, CV_32FC3, r.frame_buffer().data());
        image.convertTo(image, CV_8UC3, 1.0f);
        cv::cvtColor(image, image, cv::COLOR_RGB2BGR);

        cv::imwrite(filename, image);

        return 0;
    }

    while(key != 27)
    {
        r.clear(rst::Buffers::Color | rst::Buffers::Depth);

        r.set_model(get_model_matrix(angle));
        r.set_view(get_view_matrix(eye_pos));
        r.set_projection(get_projection_matrix(45, 1, 0.1, 50));

        r.draw(pos_id, ind_id, col_id, rst::Primitive::Triangle);

        cv::Mat image(700, 700, CV_32FC3, r.frame_buffer().data());
        image.convertTo(image, CV_8UC3, 1.0f);
        cv::cvtColor(image, image, cv::COLOR_RGB2BGR);
        cv::imshow("image", image);
        key = cv::waitKey(10);

        std::cout << "frame count: " << frame_count++ << '\n';
    }

    return 0;
}
// clang-format on
```



----

## 框架研究

在编写SSAA抗锯齿之前，我们需要通读代码。

### main函数

我们先从main函数开始：

```c++
float angle = 0;
bool command_line = true;
std::string filename = "output.png";

if (argc == 2)
{
    command_line = true;
    filename = std::string(argv[1]);
}
```

**angle**定义了旋转角度，HW2中用不到，不需要理会。

**command_line**定义当前是否是命令行模式打开程序。

- 此处我们把**command_line**改为true，让程序输出图像文件。后续更好的观察SSAA的效果。

```c++
rst::rasterizer r(700, 700);
Eigen::Vector3f eye_pos = {0,0,5};
```

**r** 的两个参数是屏幕长宽，r代表了整个渲染器。

**eye_pos** 定义了眼睛的位置。

```c++
std::vector<Eigen::Vector3f> pos
{
        {2, 0, -2},
        {0, 2, -2},
        {-2, 0, -2},
        {3.5, -1, -5},
        {2.5, 1.5, -5},
        {-1, 0.5, -5}
};

std::vector<Eigen::Vector3i> ind
{
        {0, 1, 2},
        {3, 4, 5}
};

std::vector<Eigen::Vector3f> cols
{
        {217.0, 238.0, 185.0},
        {217.0, 238.0, 185.0},
        {217.0, 238.0, 185.0},
        {185.0, 217.0, 238.0},
        {185.0, 217.0, 238.0},
        {185.0, 217.0, 238.0}
};
```

pos是顶点坐标。

ind是三角形的顶点索引。比如说`{0, 1, 2}`的意思就是取pos数组中的0，1，2个元素。

cols表示顶点的颜色RGB数值。

```c++
auto pos_id = r.load_positions(pos);
auto ind_id = r.load_indices(ind);
auto col_id = r.load_colors(cols);

int key = 0;
int frame_count = 0;
```

和HW1一样，加载三角形数据到r渲染器中。

再初始化frame_count帧数计数器。

```c++
if (command_line){
    ...
    return 0;
}
```

如果command_line为true，则输出单张图片（第一帧）后退出程序。

否则进入while loop，每一次loop就是一帧。

```c++
while(key != 27){
	...
}
```

接下来是while循环里面的内容：

```c++
r.clear(rst::Buffers::Color | rst::Buffers::Depth);

r.set_model(get_model_matrix(angle));
r.set_view(get_view_matrix(eye_pos));
r.set_projection(get_projection_matrix(45, 1, 0.1, 50));

r.draw(pos_id, ind_id, col_id, rst::Primitive::Triangle);

cv::Mat image(700, 700, CV_32FC3, r.frame_buffer().data());
image.convertTo(image, CV_8UC3, 1.0f);
cv::cvtColor(image, image, cv::COLOR_RGB2BGR);
cv::imshow("image", image);
key = cv::waitKey(10);

std::cout << "frame count: " << frame_count++ << '\n';
```

开始渲染一张画面前，先调用渲染器的清除残留画面函数（或者初始化）。

然后执行MVP。

调用draw函数，并且告诉draw函数我们绘制的是三角形。

draw函数下面的都是OpenCV的函数，与本课程没有关系。

接下来重点探讨draw函数的实现。

跳转到draw函数位置。

### draw函数

- 简单地说，draw函数处理的是几何图形（在这种情况下是三角形）转换成像素的过程。

```c++
void rst::rasterizer::draw(pos_buf_id pos_buffer, ind_buf_id ind_buffer, col_buf_id col_buffer, Primitive type){
	...
    for (auto& i : ind){
        Triangle t;
        ...
        rasterize_triangle(t);
    }
}
```

我们发现最大的for循环 `for (auto& i : ind){...}` 的末尾都会调用一个`rasterize_triangle(t);` 函数，即光栅化三角形。

传入的参数 `t` 其实是一个三角形。关于三角形的数据结构先不用管。

也就是说，每一次 `for (auto& i : ind)` ，就相当于处理一个三角形。

具体的细节不讲太多，主要讲一下Viewport transformation中的z值的处理。

```c++
//Viewport transformation
for (auto & vert : v)
{
    vert.x() = 0.5*width*(vert.x()+1.0);
    vert.y() = 0.5*height*(vert.y()+1.0);
    vert.z() = vert.z() * f1 + f2;
}
```

`vert.z() = vert.z() * f1 + f2;`这一行代码的意思是：
在做完mvp和Homogeneous division之后，vert.z()被压缩到了-1到1的区间。
经过vert.z() * f1 + f2后，z值（深度值）从齐次坐标系转换到屏幕空间，并将其范围映射到0到1之间，以便后续的深度测试和深度写入。

最终将颜色放入t三角形，调用 `rasterize_triangle(t);` 进一步渲染三角形。

接下来讲解 `rasterize_triangle(t);` 函数。

### rasterize_triangle函数

```
void rst::rasterizer::rasterize_triangle(const Triangle& t)
```

这个函数传入一个三角形。三角形包含三个点和每个顶点的颜色、深度信息。

这里重点解说z-buffer的内容。

```c++
auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
z_interpolated *= w_reciprocal;
```

`computeBarycentric2D(x, y, t.v)`这行代码是在计算重心坐标，它返回一个三元组[alpha, beta, gamma]，这三个值分别表示像素点相对于三角形三个顶点的权重，而且这三个值的总和为1。

但是我们知道，重心坐标在经过MVP视角变换之后，会变化。所以我们需要做矫正。

然后，`w_reciprocal`的计算是在进行**透视校正**。因为我们在透视投影下，距离摄像机远的物体，它们在屏幕空间中的深度变化要小于距离摄像机近的物体。透视校正就是为了消除这个因透视造成的非线性变化。

接下来的`z_interpolated`就是在用**重心坐标对深度z进行插值**。注意这里的插值是在进行透视除法之前的深度，因此每个深度值还需要除以对应的齐次坐标w。

最后，用`w_reciprocal`对`z_interpolated`进行透视校正，得到的就是在屏幕空间中，当前像素点的正确深度值。这个值可以用于之后的深度测试，决定是否应该绘制这个像素。

作业框架大致流程就已经介绍完毕了，接下来讲解如何写SSAA。

## SSAA算法实现

要实现SSAA（Super Sampling Anti-Aliasing），需要对每个像素进行多次采样。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/tbBxkiwj2cfaSJd-20230719182402832.png" alt="image-20230527130132554" style="zoom:50%;" />

左：Default 右：SSAA

```c++
//Screen space rasterization
void rst::rasterizer::rasterize_triangle(const Triangle& t) {
    auto v = t.toVector4();

    // TODO : Find out the bounding box of current triangle.
    // iterate through the pixel and find if the current pixel is inside the triangle

    // If so, use the following code to get the interpolated z value.
    //auto[alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
    //float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
    //float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
    //z_interpolated *= w_reciprocal;

    // TODO : set the current pixel (use the set_pixel function) to the color of the triangle (use getColor function) if it should be painted.

    // Finding out the bounding box for the current triangle
//    float min_x = std::min({v[0].x(), v[1].x(), v[2].x()});
//    float max_x = std::max({v[0].x(), v[1].x(), v[2].x()});
//    float min_y = std::min({v[0].y(), v[1].y(), v[2].y()});
//    float max_y = std::max({v[0].y(), v[1].y(), v[2].y()});

    float min_x = std::floor(std::min({v[0].x(), v[1].x(), v[2].x()}));
    float max_x = std::ceil(std::max({v[0].x(), v[1].x(), v[2].x()}));
    float min_y = std::floor(std::min({v[0].y(), v[1].y(), v[2].y()}));
    float max_y = std::ceil(std::max({v[0].y(), v[1].y(), v[2].y()}));

    bool ssaa_mode = true;
    if(ssaa_mode){
        // Iterating through each pixel in the bounding box
        for (float x = min_x; x <= max_x; x++) {
            for (float y = min_y; y <= max_y; y++) {

                float min_depth = std::numeric_limits<float>::infinity();
                Eigen::Vector3f color(0, 0, 0);

                // Super Sampling at 2x2 grid within the pixel
                for (int dx = 0; dx < 2; ++dx) {
                    for (int dy = 0; dy < 2; ++dy) {
                        float sample_x = x + dx * 0.5f;
                        float sample_y = y + dy * 0.5f;

                        // If the sample point is inside the triangle
                        if (insideTriangle(sample_x, sample_y, t.v)) {
                            auto[alpha, beta, gamma] = computeBarycentric2D(sample_x, sample_y, t.v);
                            float w_reciprocal = 1.0/(alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
                            float z_interpolated = alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
                            z_interpolated *= w_reciprocal;

                            // Accumulate color and find minimum depth
                            color += t.getColor();
                            min_depth = std::min(min_depth, z_interpolated);
                        }
                    }
                }

                color /= 4.0f; // average the color

                if (min_depth < depth_buf[get_index(x, y)]) {
                    // Set the pixel color if it should be painted
                    set_pixel(Eigen::Vector3f(x, y, min_depth), color);
                    depth_buf[get_index(x, y)] = min_depth;
                }
            }
        }
        return;
    }else{
        // Iterating through each pixel in the bounding box
        for (float x = min_x; x <= max_x; x++) {
            for (float y = min_y; y <= max_y; y++) {
                // If the pixel is inside the triangle
                if (insideTriangle(x, y, t.v)) {
                    auto [alpha, beta, gamma] = computeBarycentric2D(x, y, t.v);
                    float w_reciprocal = 1.0 / (alpha / v[0].w() + beta / v[1].w() + gamma / v[2].w());
                    float z_interpolated =
                            alpha * v[0].z() / v[0].w() + beta * v[1].z() / v[1].w() + gamma * v[2].z() / v[2].w();
                    z_interpolated *= w_reciprocal;
                    if (z_interpolated < depth_buf[get_index(x, y)]) {
                        // Set the pixel color if it should be painted
                        set_pixel(Eigen::Vector3f(x, y, z_interpolated), t.getColor());
                        depth_buf[get_index(x, y)] = z_interpolated;
                    }
                }
            }
        }
    }
}
```
