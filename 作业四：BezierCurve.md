使用De Casteljau的算法来绘制由四个控制点表示的Bezier曲线，改进部分实现了贝塞尔曲线的抗锯齿。抗锯齿是通过在像素之间进行线性插值来改善图像的视觉效果。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/g9nRCkNoHGpUw3q.png" alt="三个代码的不同效果对比" style="zoom:10%;" />

<!--more-->

# HW4:绘制贝塞尔曲线 + 线性插值反走样

## 题目

### 基础部分

- 实现 De Casteljau 算法来绘制由 4 个控制点表示的 Bézier 曲线。

  - **bezier**:该函数实现绘制 Bézier 曲线的功能。它使用一个控制点序列和一个 OpenCV::Mat 对象作为输入，没有返回值。它会使 t 在 0 到 1 的范围内进行迭代，并在每次迭代中使 t 增加一个微小值。对于每个需要计算的 t，将调用另一个函数 recursive_bezier，然后该函数将返回在 Bézier 曲线上 t 处的点。最后，将返回的点绘制在 OpenCV ::Mat 对象上。

  - **recursive_bezier**:该函数使用一个控制点序列和一个浮点数 t 作为输入， 实现 de Casteljau 算法来返回 Bézier 曲线上对应点的坐标。

#### 完善以下代码

 ```c++
 cv::Point2f recursive_bezier(const std::vector<cv::Point2f> &control_points, float t) 
 {
     // TODO: Implement de Casteljau's algorithm
     return cv::Point2f();
 }
 
 void bezier(const std::vector<cv::Point2f> &control_points, cv::Mat &window) 
 {
     // TODO: Iterate through all t = 0 to t = 1 with small steps, and call de Casteljau's 
     // recursive Bezier algorithm.
 }
 ```

### 提高部分

- 实现对 Bézier 曲线的反走样。(对于一个曲线上的点，不只把它对应于一个像素，你需要根据到像素中心的距离来考虑与它相邻的像素的颜色。)

## 原理

#### De Casteljau算法

De Casteljau 算法说明如下：

1. 考虑一个 p0, p1, ... pn 为控制点序列的 Bézier 曲线。首先，将相邻的点连接 起来以形成线段。
2. 用 t : (1 − t) 的比例细分每个线段，并找到该分割点。
3. 得到的分割点作为新的控制点序列，新序列的长度会减少一。
4. 如果序列只包含一个点，则返回该点并终止。否则，使用新的控制点序列并 转到步骤 1。

使用 [0,1] 中的多个不同的 t 来执行上述算法，你就能得到相应的 Bézier 曲线。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/vVhaEScqslN4A7m.png" alt="image-20230611142920304" style="zoom:50%;" />



## 步骤

### 基础部分

首先写recursive_bezier函数。这个函数的作用是，传入一系列控制点集合和一个比例系数t。

输出这一系列点在t值下的Bezier曲线的坐标。

如果控制点集合只剩下一个点，那么这个点就是我们要求的坐标。

否则，递归计算。每一次递归，传入的控制点数量-1。

```c++
cv::Point2f recursive_bezier(const std::vector<cv::Point2f> &control_points, float t) 
{
    // TODO: Implement de Casteljau's algorithm

    // if there's only one control point left, return it
    if(control_points.size() == 1)
        return control_points[0];

    std::vector<cv::Point2f> new_points;

    // Compute points for next recursive step, between every pair of control points
    for(size_t i = 0; i < control_points.size()-1; i++){
        float x = (1 - t) * control_points[i].x + t * control_points[i+1].x;
        float y = (1 - t) * control_points[i].y + t * control_points[i+1].y;
        new_points.push_back(cv::Point2f(x, y));
    }

    // recursively continue on new set of points
    return recursive_bezier(new_points, t);
}
```

接下来，bezier函数就简单了，只需参考框架中给出的naive_bezier函数就行了。

```c++
void bezier(const std::vector<cv::Point2f> &control_points, cv::Mat &window) 
{
    // TODO: Iterate through all t = 0 to t = 1 with small steps, and call de Casteljau's 
    // recursive Bezier algorithm.
    const float step = 0.001f;
    for(float t = 0; t <= 1; t += step)
    {
        cv::Point2f point = recursive_bezier(control_points, t);
        window.at<cv::Vec3b>(point) = cv::Vec3b(0, 255, 0);
    }
}
```

关于 `window.at<cv::Vec3b>(point) = cv::Vec3b(0, 255, 0);` 这行代码的解释：

`window` 是图像矩阵，`point` 是你要设置的像素的位置。

整行代码的作用是将图像 `window` 在 `point` 位置的像素设置为绿色。



效果：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/4qrsdLAmCZfNOQe.png" alt="image-20230611145138155" style="zoom:50%;" />

混合输出的效果：

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/C8r7zq9caTSRVFX.png" alt="image-20230611145058801" style="zoom:50%;" />

### 提高部分

为了对比反走样的效果，在main函数中手动传入四个点的坐标。

```c++
......
cv::setMouseCallback("Bezier Curve", mouse_handler, nullptr);

control_points.emplace_back(10, 10);
control_points.emplace_back(690, 10);
control_points.emplace_back(10, 690);
control_points.emplace_back(690, 690);

int key = -1;
...
```



接下来编写上色函数。

```c++
void addColor(cv::Mat &img, cv::Point2f point, cv::Vec3b color) {
    cv::Point2f p1(point.x, point.y);
    cv::Point2f p2(point.x + 1, point.y);
    cv::Point2f p3(point.x, point.y + 1);
    cv::Point2f p4(point.x + 1, point.y + 1);

    float dx = point.x - p1.x;
    float dy = point.y - p1.y;

    img.at<cv::Vec3b>(p1) += color * ((1-dx) * (1-dy));
    img.at<cv::Vec3b>(p2) += color * (dx * (1-dy));
    img.at<cv::Vec3b>(p3) += color * ((1-dx) * dy);
    img.at<cv::Vec3b>(p4) += color * (dx * dy);
}
```

最后将bezier重新打包一下：

```c++
void bezier_sa(const std::vector<cv::Point2f> &control_points, cv::Mat &window)
{
    const double step = 0.001f;

    for(double t = 0; t <= 1; t += step)
    {
        cv::Point2f point = recursive_bezier(control_points, t);
        addColor(window, point, cv::Vec3b(0, 255, 0));
    }
}
```



为了获得更好的效果，这里尝试了几种不同的去像素点坐标的做法，请自行比对。

左：

```c++
void addColor(cv::Mat &img, cv::Point2f point, cv::Vec3b color) {
    cv::Point2f p1(point.x, point.y);
    cv::Point2f p2(point.x + 1, point.y);
    cv::Point2f p3(point.x, point.y + 1);
    cv::Point2f p4(point.x + 1, point.y + 1);

	......
}
```

中：

```c++
void addColor(cv::Mat &img, cv::Point2f point, cv::Vec3b color) {
    cv::Point p1(point.x, point.y);
    cv::Point p2(point.x + 1, point.y);
    cv::Point p3(point.x, point.y + 1);
    cv::Point p4(point.x + 1, point.y + 1);

	......
}
```

右：

```c++
void addColor(cv::Mat &img, cv::Point2f point, cv::Vec3b color) {
    cv::Point2f center(std::round(point.x), std::round(point.y));
    cv::Point2f p1(center.x - 1, center.y - 1);
    cv::Point2f p2(center.x, center.y - 1);
    cv::Point2f p3(center.x - 1, center.y);
    cv::Point2f p4(center.x, center.y);

	......
}
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/g9nRCkNoHGpUw3q-20230627155930526.png" alt="三个代码的不同效果对比" style="zoom:67%;" />

显然，Point必须要是浮点类型的数值。其次，不将点做一次round+0.5效果更好（更少的锯齿感、边缘平滑）。

值得一说的是，不使用反走样的图像和使用Point而不是Point2f反走样的图像效果是一样的。
