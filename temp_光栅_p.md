## 6.2 重写模版向量类

### 第一关：需求分析

不懂c++模版类的读者可以阅读 **附件1** ，不看也行，但是确保你对C++的模版特性有所了解。

本文我一步一步带大家实现以下功能：

1. **向量 (vec)**
   - 有通用的模板定义和2D、3D的特化版本。
   - 通用构造函数，将每个元素初始化为T类型的默认值。
   - 2D和3D向量的构造函数可以接受特定的初始化值。
   - 提供了索引运算符来获取或设置特定元素的值。
   - 3D向量有`norm`函数，返回向量的模长。
   - 3D向量有`normalize`函数，可以规范化向量。
   - 重载了输出运算符，方便向量的打印。
   - 运算符重载：向量的点乘、加法、减法、标量乘法和标量除法。
   - `embed`和`proj`函数用于扩展或投影向量到不同维度。
   - 3D向量之间的外积运算。
2. **矩阵 (mat)**
   - 可以获取或设置矩阵的行。
   - 获取矩阵的某一列。
   - 设置矩阵的某一列。
   - 获取单位矩阵。
   - 计算矩阵的行列式。
   - 获取矩阵的子矩阵。
   - 计算矩阵的余子式。
   - 计算伴随矩阵。
   - 计算逆矩阵的转置。
   - 运算符重载：矩阵和向量的乘法、两个矩阵的乘法、矩阵的标量除法。
   - 重载了输出运算符，方便矩阵的打印。
3. **其他功能**
   - 使用typedef定义了常用的类型，如`Vec2f`, `Vec3i`, `Matrix`等。
   - 在geometry.cpp中，提供了从3D和2D的float向量到int向量的转换，以及相反的转换。



### 第二关：实现Vec2模版以及四个算数符

这一关我们构建Vec2i和Vec2f类，以及实现他们的加、减、点积和叉积操作。

我这里直接创建了一个新的cpp项目，名为MyMathLib。在项目中，创建 `geometry.h` 头文件和 `geometry.cpp` 源代码文件。**提醒一下**，其实在写类模版的时候尽量都把内容写在头文件里即可，这里只是暂时分开写，我们马上就会发现这种写法的维护难度很大。

在头文件中定义 Vec2 模版类，让他既支持整形，也支持浮点数。

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

源代码文件实现构造函数、加法、减法、点积和叉积，然后在最后实现外部模板实例化。

**注意**：一般情况下，我们都直接把所有的实现（即函数体）都放在头文件中，这里只是稍微拓展一下可以使用**外部模板实例化**将类的模版的实现放在.cpp中。

**首先，我们需要理解C++中的模板是什么。**模板不是实际的函数或类，而是编译器使用的蓝图，用于生成函数或类的特定版本。这就是为什么我们通常会看到模板的定义直接在头文件中：当模板在某个源文件中使用时，编译器需要看到完整的模板定义，以便为特定的类型生成正确的代码。

**为什么要实现外部模板实例化？**模板的定义通常直接出现在头文件中。但有时，为了组织或其他原因，会把模板类的定义从其声明中分离出来（就像常规的非模板类那样），我目前也是这样做的。但这样做引发了一个问题，当链接器尝试链接对象文件时，如果它没有为特定的模板类型实例找到定义，就会出错。这是因为编译器只为那些它确实看到的模板类型生成代码。

对于模板类，如果模板类的所有成员函数都在类声明中定义（即在头文件中定义），那么当模板类用于特定类型时，编译器可以立即为该类型生成模板类的实例。但是，如果模板类的某些成员函数在类声明之外定义（例如，在`.cpp`文件中），那么你可能需要使用外部模板实例化来确保为所需的类型生成正确的模板实例。

```c++
// geometry.cpp
#include "geometry.h"

// 构造函数
template <typename T>
Vec2<T>::Vec2() : x(0), y(0) {}

template <typename T>
Vec2<T>::Vec2(T x, T y) : x(x), y(y) {}

// 加法
template <typename T>
Vec2<T> Vec2<T>::operator+(const Vec2<T>& v) const {
    return Vec2<T>(x + v.x, y + v.y);
}

// 减法
template <typename T>
Vec2<T> Vec2<T>::operator-(const Vec2<T>& v) const {
    return Vec2<T>(x - v.x, y - v.y);
}

// 点积
template <typename T>
T Vec2<T>::dot(const Vec2<T>& v) const {
    return x * v.x + y * v.y;
}

// 叉积
template <typename T>
T Vec2<T>::cross(const Vec2<T>& v) const {
    return x * v.y - y * v.x;
}

template class Vec2<int>;
template class Vec2<float>;
template class Vec2<double>;
```

调用测试一下。

```c++
// main.cpp
#include "geometry.h"
#include <iostream>

int main() {
    // 使用浮点数向量
    Vec2f v1(1.0f, 2.0f);
    Vec2f v2(2.0f, 3.0f);

    Vec2f sum = v1 + v2;
    std::cout << "v1 + v2 = (" << sum.x << ", " << sum.y << ")\n";

    float dotProduct = v1.dot(v2);
    std::cout << "v1 . v2 = " << dotProduct << "\n";

    float crossProduct = v1.cross(v2);
    std::cout << "v1 x v2 = " << crossProduct << "\n";

    // 使用整数向量
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

结果：

>v1 + v2 = (3, 5)
>v1 . v2 = 8
>v1 x v2 = -1
>v3 + v4 = (3, 5)
>v3 . v4 = 8
>v3 x v4 = -1

### 第三关：实现Vec3模版以及四个算数符

Vec3其实和Vec2基本一致，只需修改一下计算代码即可，这里就不一一展示了。大部分就是将Vec2改成Vec3，计算时增加一个维度的考虑，比方说叉积。

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

### 第四关：用模版构建不同大小的向量

如果我还想添加Vec4，岂不是又要写一大堆，不简洁！对于多种不同大小的向量进行明确实例化是非常繁琐的。

这里我是用的是 `...` 折叠表达式（C++17）递归构造。目前头文件大致结构是这样的：

```c++
template <typename T, int N>
struct Vec {
    T values[N];

    // 构造函数
    Vec() = default; // 默认构造函数
    template<typename... Args> Vec(Args... args);
    
    // ... 操作声明（例如加减点积等） ...
};

// ... 实现 ...

// 为 Vec2、Vec3、Vec4 等提供类型别名

```

这里说一下构造函数，如果是隐式构造的，那么默认会使用Vec()。为了代码健壮性，其实可以在带可变长参数的构造函数前加 `explicit` 关键词。

```c++
template<typename... Args> explicit Vec(Args... args);
```

读者可能感觉到了，我在这里直接将 .cpp 的操作挪过来头文件里边来实现了，这是因为如果这个时候要分离写的话，代码冗余量会很大。因此我们全部写在头文件里面，省事优雅。下面是当前完整的 geometry.h 代码。

```c++
#ifndef MYMATHLIB_GEOMETRY_H
#define MYMATHLIB_GEOMETRY_H

// geometry.h
#pragma once

#include <iostream>

template <typename T, int N>
struct Vec {
    T values[N];

    // 构造函数
    Vec() = default; // 默认构造函数
    template<typename... Args> explicit Vec(Args... args);

    void print() const;
    T& operator[](int index);
    const T& operator[](int index) const;

    Vec<T, N> operator+(const Vec<T, N>& other) const;
    Vec<T, N> operator-(const Vec<T, N>& other) const;
    T dot(const Vec<T, N>& other) const;

    // 叉积仅对3D向量有效
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

// 非常量的Vec访问，可以称为左值
template <typename T, int N>
T& Vec<T, N>::operator[](int index) {
    return values[index];
}

// 常量Vec访问
template <typename T, int N>
const T& Vec<T, N>::operator[](int index) const {
    return values[index];
}

// 实现加法运算
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator+(const Vec<T, N>& other) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] + other[i];
    }
    return result;
}

// 实现减法运算
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator-(const Vec<T, N>& other) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] - other[i];
    }
    return result;
}

// 实现点积运算
template <typename T, int N>
T Vec<T, N>::dot(const Vec<T, N>& other) const {
    T sum = 0;
    for(int i = 0; i < N; i++) {
        sum += values[i] * other[i];
    }
    return sum;
}

// 实现了Vec3的叉积运算
template <typename T, int N>
template<int M>
typename std::enable_if<M == 3, Vec<T, 3>>::type Vec<T, N>::cross(const Vec<T, 3>& other) const {
    return Vec<T, 3>(
            values[1] * other[2] - values[2] * other[1],
            values[2] * other[0] - values[0] * other[2],
            values[0] * other[1] - values[1] * other[0]
    );
}

// 为 Vec2、Vec3、Vec4 等提供类型别名
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

测试：

```c++
int main() {
    // 使用整数向量测试
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



### 第五关：进一步完善向量功能

先总结一下目前完成的内容：

- 通过两个模版参数 T 和 N，定义了一个向量。
- 有默认构造函数与可变参数的构造函数。
- 功能有：打印向量、通过索引访问向量元素、向量加法、向量减法、向量点积和三维向量叉积。
- 提供了大量常用的向量别名。

目前来说已经基本可以用了，但是还有很多需要完善，我们继续看需要完成的内容！

1. 增加标量与向量的乘/除法
2. 计算向量的模
3. 向量单位化
4. 重载输出运算符

另外，可以在声明运算操作中使用 `[[nodiscard]]` 标签，提醒编译器注意检查返回值是否得到使用，然后使用该库的用户就可以在编辑器中得到提醒，例如下面。

```c++
[[nodiscard]] Vec<T, N> normalize() const;// 将向量单位化
```

当前功能的声明：

```c++
[[nodiscard]] Vec<T, N> operator*(T scalar) const;// 向量与常数乘法
[[nodiscard]] Vec<T, N> operator/(T scalar) const;// 向量与常数除法
[[nodiscard]] double magnitude() const;// 向量模长
[[nodiscard]] Vec<T, N> normalize() const;// 向量单位化
// 流传输功能
template <typename U, int M>
friend std::ostream& operator<<(std::ostream& os, const Vec<U, N>& vec);
```

对应的实现：

```c++
...
// 流传输功能
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

// 实现标量与向量的乘
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator*(T scalar) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] * scalar;
    }
    return result;
}

// 实现标量与向量的除法
template <typename T, int N>
Vec<T, N> Vec<T, N>::operator/(T scalar) const {
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] / scalar;
    }
    return result;
}

// 实现求模长
template <typename T, int N>
double Vec<T, N>::magnitude() const {
    T sum = 0;
    for(int i = 0; i < N; i++) {
        sum += values[i] * values[i];
    }
    return std::sqrt(sum);
}

// 单位化向量
template <typename T, int N>
Vec<T, N> Vec<T, N>::normalize() const {
    T mag = magnitude();
    if(mag == 0) {
        // 不能单位化一个零向量，此处可以抛出异常或返回原向量
        // 为简化，此处返回原向量
        return *this;
    }
    Vec<T, N> result;
    for(int i = 0; i < N; i++) {
        result[i] = values[i] / mag;
    }
    return result;
}
```

可以在这里[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/Custom_Math_Lib)中获取当前的向量库代码。

### 第六关：构建矩阵

有了上面构建向量模版的经验，我们可以照葫芦画瓢写出一个矩阵模版。矩阵的构造、访问元素、加法、乘法等操作都一一实现即可。

这里读者应该给自己几个小时，独立写出代码。

当我作为库的使用者创建一个矩阵时，我会想这样创建：

```c++
Matrix<int, 2, 2> mat = {
    {1, 2},
    {3, 4}
};
```

构造函数可以使用两层嵌套的 `std::initializer_list`。其中，`std::initializer_list` 是一个C++11中引入的模板类，它表示编译时确定的值列表。先遍历行，再遍历列。

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
// BTW：奇技淫巧压缩代码
Matrix(const std::initializer_list< std::initializer_list<T> >& list) {
    T* target = &values[0][0];
    for (const auto& rowList : list) {
        target = std::copy(rowList.begin(), rowList.end(), target);
    }
}
```

这里重点说一下矩阵的乘法。

```c++
template <typename T, int Rows, int Cols>
struct Matrix {
    T values[Rows][Cols];
    
	...
        
	// 矩阵与矩阵的乘法
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

一个 `Rows x Cols` 的矩阵A和一个 `Cols x NewCols` 的矩阵B相乘，那么结果将是一个 `Rows x NewCols` 的矩阵。举个例子：

```c++
Matrix<int, 3, 2> matA = { /* 初始化 */ };
Matrix<int, 2, 4> matB = { /* 初始化 */ };
Matrix<int, 3, 4> result = matA * matB;  // 这里的乘法使用的就是上述函数，NewCols 在这里为4
```

下面是其他的一些操作。

```c++
// 打印矩阵函数
void print() const {
    for (int i = 0; i < Rows; i++) {
        for (int j = 0; j < Cols; j++) {
            std::cout << values[i][j];
            if (j < Cols - 1) {
                std::cout << "\t";  // 在列之间添加制表符，以美化输出
            }
        }
        std::cout << std::endl;  // 打印换行，进入下一行
    }
}

// 访问矩阵元素
T& operator()(int row, int col) {
    return values[row][col];
}
const T& operator()(int row, int col) const {
    return values[row][col];
}

// 矩阵加法
Matrix operator+(const Matrix& other) const {
    Matrix result;
    for (int i = 0; i < Rows; i++) {
        for (int j = 0; j < Cols; j++) {
            result(i, j) = values[i][j] + other(i, j);
        }
    }
    return result;
}

// 矩阵与标量的乘法
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

总结一下目前的工作：

- 构建了矩阵类的模版结构
- 提供默认和可传参的构造函数
- 打印矩阵函数
- 访问矩阵操作符()
- 矩阵加法+
- 矩阵与标量的乘法*
- 矩阵与矩阵的乘法*

添加了完整的注释供大家参考，可以在这个[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/Custom_Math_Lib_v2)中找到当前的数学库。

### 第七关：添加更多的功能

在 Vec 中，添加如下功能：

- Vec 可以通过已有的 `Vec_f` 或  `Vec_d` 对象构造 `Vec_i` ，且是四舍五入的。也可以相反。
- 通用的构造函数，可以接受不同类型的参数，最终转换为统一的类型

第一项功能：

```c++
/**
 * @brief 可以将已有的 `Vec_U` 对象构造 `Vec_T`。
 * 四舍五入的。
 *
 * @param Vec<U, N> other 待转换的向量
 * @return 新类型的向量。
 */
template <typename U>
explicit Vec<T, N>(const Vec<U, N>& other);

...// 其他代码
    
// 将已有的 `Vec_U` 对象构造 `Vec_T`
template <typename T, int N>
template <typename U>
Vec<T, N>::Vec(const Vec<U, N>& other) {
    for(int i = 0; i < N; ++i) {
        values[i] = static_cast<T>(std::round(other[i]));
    }
}
```

第二项功能：

```c++
/**
 * @brief 变参构造函数。
 * 允许通过提供具体的值来初始化向量。
 *
 * @tparam Args 变参类型。
 * @param args 用于初始化向量的值。
 */
template<typename... Args> explicit Vec(Args... args);

...// 其他代码
    
// 更通用的构造函数 比如你可以这样构造函数 Vec2f v(1,2);
template <typename T, int N>
template <typename... Args>
Vec<T, N>::Vec(Args... args) : values{static_cast<T>(args)...} {
    static_assert(sizeof...(args) == N, "Wrong number of arguments");
}
```

目前代码可以从[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/Custom_Math_Lib_v3)中下载。



## 6.3 整合光栅化代码

浏览我们目前的main函数，既有矩阵变换函数，也有视角变换函数，还有三角形重心坐标光栅化三角形的函数，更有视角变换矩阵等，真的有些乱。我们将这些方法打包到一个新的类里面，这个类称为：our_gl。

### 特别节目1之：main代码之旅

最终main函数如下：

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

其中，顶点着色和片元着色是可编程的。可以参考当前的项目代码[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/6_1_Our_GL)。

🎬 **开场白**

接下来开始解读这个main函数。首先，代码导入了一堆头文件，为了让我们的程序能够处理3D模型、向量计算和图像生成。

```c++
#include <vector>
#include <iostream>
#include "tgaimage.h"
#include "model.h"
#include "geometry.h"
#include "our_gl.h"
```

🌍 **全局变量来啦**

接着，全局变量闪亮登场！有了宽度、高度、光照方向、观察点等等，这简直是个小型的“宇宙”。

```c++
Model *model     = NULL;
const int width  = 800;
const int height = 800;
Vec3f light_dir(1,1,1);
Vec3f       eye(0,-1,3);
Vec3f    center(0,0,0);
Vec3f        up(0,1,0);
```

🎭 **GouraudShader 诞生**

然后，我们有一个名为 `GouraudShader` 的类，这家伙是渲染的明星！它的职责是处理顶点和片段（像素）。

```c++
struct GouraudShader : public IShader {
    // ... 
}
```

🎸 **主舞台 main 函数**

最后，`main()` 函数，这是我们的主舞台。所有的预设、加载、渲染都在这里完成。

```c++
int main(int argc, char** argv) {
    //...
}

```

🎥 **Action！动作！**

1. **加载模型**: `new Model("../object/african_head/african_head.obj");` 这里，我们召唤了一个来自非洲的神秘头颅！
2. **视角设置**: 使用 `lookat`, `viewport`, 和 `projection` 函数，我们调整了观察点、视口和投影。这些都是电影导演级别的设置！
3. **初始化画布**: `TGAImage image (width, height, TGAImage::RGB);` 这里我们预备了一张画布，准备大展身手！
4. **渲染循环**: 嗯，这里有一个循环，负责画出那个非洲头颅。用了 `GouraudShader`，它会逐个面片地渲染模型。
5. **图片翻转和保存**: 最后，不要忘了翻转图像，并保存为 `.tga` 格式。现在你就可以拿这个图跟朋友炫耀了！

### 特别节目2之：细说GouraudShader

这个角色是渲染的灵魂，让我们细致入微地来看一下它的表演。

**🕺GouraudShader的组成**

- **变量：varying_intensity**

这个变量是一个3D向量（Vec3f类型），用来存储每个顶点的光照强度。这里的“varying”意味着这个变量会在顶点着色器和片段着色器之间“变化”（实际上是插值）。

- **方法：vertex**

这个函数负责处理每个顶点。它做了以下几件事：

1. **获取模型顶点**: `Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert));` - 从3D模型中提取出一个顶点。
2. **坐标转换**: `gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;` - 使用各种矩阵变换将这个顶点从模型空间转换到屏幕空间。
3. **光照计算**: `varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir);` - 根据光照方向计算这个顶点的光照强度。

- **方法：fragment**

这个函数负责处理每个像素（片段）：

1. **插值计算**: `float intensity = varying_intensity*bar;` - 这里使用bar来进行插值，得到当前像素的光照强度。
2. **颜色设置**: `color = TGAColor(255, 255, 255)*intensity;` - 根据光照强度设置像素的颜色。
3. **像素保留**: `return false;` - 表示这个像素不会被丢弃，将出现在最终的图像中。

**🎭角色分析**

这个`GouraudShader`类扮演了一个全能艺人的角色：

- **化妆师**：通过`vertex`函数，对每个顶点进行"化妆"，也就是坐标变换和光照计算。
- **导演**：通过`fragment`函数，决定哪些像素应该用什么颜色来"演绎"，以及哪些像素应该被"剪辑"掉（这里没有剪辑，所有像素都保留）。
- **灯光师**：通过计算光照强度，控制场景的"明暗"，使得模型更加逼真。

![image-20230921174128141](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921174128141.png)

### 特别节目3之：开始绘画-片元着色器

OK，我们重新化妆一下，修改片元着色器（fragment）。

```c++
float intensity = varying_intensity*bar;// 通过插值计算得到当前像素的光照强度。
if (intensity>.85) intensity = 1;
else if (intensity>.60) intensity = .80;
else if (intensity>.45) intensity = .60;
else if (intensity>.30) intensity = .45;
else if (intensity>.15) intensity = .30;
else intensity = 0;
color = TGAColor(155, 155, 0)*intensity;
return false;
```

根据光照强度对像素进行了“分级”。每个级别都有一个特定的光照强度，就像你在照片编辑软件里手动设置不同级别的亮度。

这里将颜色设置为一个黄色（155, 155, 0），然后用上面的 `intensity` 来调节这个颜色。结果是一种不同深浅的黄色。

![image-20230921174422284](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921174422284.png)

着色器代码就像艺术家的调色板，你永远不知道接下来会画出什么样的图像！以下是一些有趣的着色器代码片段给大伙参考参考：

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

![image-20230921193615375](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193615375.png)

#### 📺 模拟老电视效果

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

![image-20230921193810437](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193810437.png)

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

![image-20230921193918227](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193918227.png)

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

![image-20230921193958547](https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921193958547.png)