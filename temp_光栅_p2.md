## 6.4 升级Shader-支持UV纹理

在上一节我们把玩了`GouraudShader`，现在我介绍一个更强的选手，`Shader`，他不仅能够处理光照，还支持纹理贴图！

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

### 🎬 **Shader类的角色列表**

1. **varying_intensity**：这个角色没变，依然是顶点着色器计算出的光照强度。
2. **varying_uv**：新角色登场！这家伙用来存储每个顶点的纹理坐标（u,v）。

### 🎭 **vertex函数：多面手**

这个函数的流程与之前的大同小异，但多了一个关键步骤：

```c++
varying_uv.set_col(nthvert, model->uv(iface, nthvert));
```

这里，它从模型中获取每个顶点的纹理坐标（UV坐标）并存储下来。这些坐标将被用于后面的片段着色器中进行纹理贴图。

### 🌈 **fragment函数：艺术家**

在这个函数里，除了处理光照之外，我们还添加了纹理：

```c++
float intensity = varying_intensity*bar; // 同样是计算当前像素的光照强度
Vec2f uv = varying_uv*bar;               // 新功能：计算当前像素的纹理坐标
```

这里，`bar`用来进行插值，得到当前像素点的光照强度和纹理坐标。

```c++
color = model->diffuse(uv)*intensity; // 结合纹理和光照来计算最终颜色
```

接着，最精彩的部分来了。我们将之前计算的 `intensity` 和从 `uv` 贴图中获取的颜色相乘，得到的就是一个非常真实的颜色了。

### 🎨 **那么，这个Shader类都能做什么？**

1. **纹理贴图**：它能给3D模型穿上“衣服”，使模型看起来更逼真。
2. **漫反射光照**：它依然做好了基础的光照工作，让模型不会看起来像个平面。
3. **代码复用**：由于这个类继承了`IShader`，你可以很容易地在不同的渲染任务中复用这段代码。



## 6.5 学习法线贴图

法线贴图是一种用于3D计算机图形的技术，用于使3D模型的表面看起来更加详细，而无需使用更多的多边形。

简而言之，你使用一张纹理来存储有关如何微妙调整模型表面上的法线向量的信息，从而改变光与模型表面的相互作用方式。

### 第一关：纹理

用于法线贴图的纹理通常看起来像一种奇怪、抽象的蓝色混合物。每个像素的RGB值代表一个3D向量的X、Y、Z分量，这将用于在光照计算期间调整3D模型的表面法线。这与仅依赖模型几何形状计算得出的法线（每个顶点的法线）有所不同。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921214038676.png" alt="image-20230921214038676" style="zoom:50%;" />

### 第二关：全局坐标系与Darboux坐标系

- **全局坐标系**：在这里，法线是在全局（世界坐标）系统中表示的。

- **Darboux（切线空间）坐标系**：在这里，法线是相对于对象本身的表面表示的。Z向量垂直于物体的表面，而X和Y与物体的表面相切。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230921222119442.png" alt="image-20230921222119442" style="zoom:50%;" />

通常，Darboux坐标系中的纹理看起来更加“有机”或“弯曲”，因为它是相对于物体表面的。全局坐标系中的纹理可能看起来更“统一”或“笔直”。因此，Darboux坐标系（切线空间）通常被认为更好，原因是它是对象相对的，使其在复杂的3D场景中具有更高的灵活性和通用性。

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

### 第三关：经常见到的Uniform

首先，“Uniform”是GLSL（图形库着色器语言）中的一个保留关键字。这个关键字让你能够将常量（不会在渲染过程中改变的值）传递到着色器中。这里我们的渲染器也保留GLSL的名字。

就像给了演员一个剧本，告诉他们："别乱改，按这个演！"

### 第四关：光照计算

在上面这段代码中，光照强度的计算与之前基本相同，但有一个例外：它不是从每个顶点插值得到法线向量，而是从法线贴图纹理中获取这些信息。

换句话说，以前是依赖演员的自然演技（顶点法线），现在我们有了特效化妆师（法线贴图纹理）来让演员更出彩！

简而言之，上面这段代码就是将业余戏剧社升级到好莱坞级别的制作！

## 6.6 实现Phong模型

Phong模型包含了三项**漫反射 (Diffuse Reflection)**、**镜面反射 (Specular Reflection)**和**环境反射 (Ambient Reflection)**。

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

接下来，我们将会讨论阴影。

## 7.1 阴影

需要注意的是，这里我们谈论的是硬阴影，软阴影的实现又是另外一回事了。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922113248506.png" alt="image-20230922113248506" style="zoom: 33%;" /><img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922113300470.png" alt="image-20230922113300470" style="zoom: 33%;" />

上面两张图片就是本章要实现的内容。读者可能会想，右边的图片拍摄角度是不是有问题。实际上，右边这张图的拍摄位置是光源所在的位置。至于为什么，我们就在本章详细探讨。你可能会看到上图有一些瑕疵，这正是 [Z-fighting](https://en.wikipedia.org/wiki/Z-fighting) 现象。

### 第一关：目前的问题

回到我们上一章完成的进度。根据我们的常识，在光线照射不到的地方（图中高亮人物脖子的一侧），应该与能照射到的地方有比较明显的光照分界。目前我们的渲染器输出的效果是左图，而正常来说应该像右边的图片那样。

<img src="/Users/remooo/Library/Application%20Support/typora-user-images/image-20230922151625557.png" alt="image-20230922151625557" style="zoom: 33%;" /><img src="/Users/remooo/Library/Application%20Support/typora-user-images/image-20230922152552309.png" alt="image-20230922152552309" style="zoom: 33%;" />

为了解决这个问题，就要搬出图形学大名鼎鼎的 two-pass 方法（two-pass rendering）了。这个方法基本思想是先从光源处渲染一副有深度信息的图片，这张照片记录了从光源视角看到的深度信息。接下来再从主相机视角渲染图像，通过上一 Pass 的深度信息判断当前的渲染像素点时候直接被光照射。

### 第二关：第一趟渲染-从光源出发

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

### 第三关：第二趟渲染-从主相机出发

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

效果非常好～该有的阴影都有了。但是我们注意这只怪物的手，阴影非常奇怪。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922163743152.png" alt="image-20230922163743152" style="zoom:50%;" />

奇怪的手臂阴影，这种现象我们称为阴影痤疮（Shadow Acne）。当渲染的物体与其阴影深度映射几乎重合时，可能会出现阴影斑点或噪声。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922164007209.png" alt="image-20230922164007209" style="zoom:50%;" />

怎么解决呢？可以考虑提高“阈值”，让物体没那么容易被相邻的部位遮挡住自己。具体到写代码上，就是稍微减小对应点的深度值。一点点小小的魔法，就可以解决这个烦人的问题了。

```c++
float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]+43.34); // magic coeff to avoid z-fighting
```

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/image-20230922170825404.png" alt="image-20230922170825404" style="zoom:50%;" />

项目的代码可以还是继续提供给读者们解读，[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/7_1_Shadow)。

由于我比较懒，不想写太详细解读了，读者自己应该可以读懂。

## 8.1 环境光遮蔽 - 模拟全局光照效果

上一讲我们实现了 Phong 光照模型，他的组成有三项分别是环境光、高光以及漫反射。还讲了一种渲染策略，Two-Pass渲染。这里，我们介绍一种新的全局光照模拟技术，环境光遮蔽（Ambient Occlusion, AO）。

但是，Phong模型只考虑了物体与特定光源之间的直接互动。在物体的小凹陷或接近的物体之间的接触区域，经常会出现微小的阴影。这些阴影往往与任何特定的光源无关，而是由于环境光被周围的几何体部分遮挡所造成的。Phong模型无法捕获这种效果，而AO可以。

巧合的是，AO并不直接依赖于场景中的光源位置或属性。这使得它可以与任何光照模型（如Phong模型）结合使用，并为渲染效果增添额外的真实感。

### 特别节目：知识脉络

读者读到这里可能会有很多疑惑，对这些名词概念的层级把握不清楚，这里给读者梳理一下。

1. **光照模型**
   - **Phong模型**
     - 环境光
     - 漫反射光
     - 镜面反射光
   - **Blinn-Phong模型**
   - Lambert模型
   - Cook-Torrance模型
   - Oren-Nayar模型
2. **全局光照模拟技术**
   - **环境光遮蔽 (AO)**
   - 光线追踪 (Ray Tracing)
   - 光子映射 (Photon Mapping)
   - 辐射度缓存 (Radiance Caching)
   - Final Gathering
3. **渲染策略**
   - **Two-Pass渲染**
     - Two-Pass阴影映射
   - 多Pass渲染
   - 延迟渲染 (Deferred Rendering)
   - 前向渲染 (Forward Rendering)
4. **后处理效果**
   - 色调映射 (Tone Mapping)
   - 抗锯齿技术 (如 MSAA, FXAA, TAA)
   - 深度模糊 (Depth of Field)
   - 动态模糊 (Motion Blur)
5. **纹理技术**
   - 传统纹理映射
   - 法线贴图 (Normal Mapping)
   - 抛物线映射 (Parallax Mapping)
   - 物理基础渲染 (Physically-Based Rendering, PBR) 的材质纹理（如 Albedo, Roughness, Metallic）



### 第一关：啥是AO？如何结合Phong使用？

环境光遮蔽的基本思想是评估一个给定的表面点在多大程度上被其周围的几何体遮挡。一个被其他物体严重遮挡的点会接收到更少的环境光，因此看起来会更暗。

当你在场景中使用Phong模型和环境光遮蔽时，通常的方法是先计算Phong模型的环境反射组成部分，然后使用环境光遮蔽来调整这个值。具体来说，你会将环境光遮蔽值乘以Phong模型的环境光分量，从而在需要的地方减少环境光。

计算方式有很多，最简单的是屏幕空间技术（如SSAO，Screen Space Ambient Occlusion）。但是在介绍这个方法之前，我们不妨先自行思考一下我们如何实现。

### 第二关：做梦

想象一下你正在拍摄一个物体，而这个物体上方有一个半透明的伞，伞的下半部分可以发出均匀的光。现在，为了知道物体的哪些部分更容易被这个光照亮，你决定在伞的内侧随机选择一些点，然后看看从这些点发出的光线能不能照到物体。

它采用一种“暴力”的方法：随机选择很多点，并从每一个点观察物体。

为了记录物体上哪些部分被光照到了，我们用一个图片来记录。每一次从伞的一个点看物体，都会产生一个新的图片。

最后，我们把所有的图片混合在一起，得到一个平均的图片。这个图片会告诉我们，物体的哪些部分通常更容易被光照到。

但是，这种方法也有缺点。比如，如果物体的两个手臂在最终的图片中使用了相同的位置，那么这两个手臂上的光就会被计算两次，这会导致最终的效果不准确。

### 第三关：屏幕空间环境遮挡 (SSAO) 

全局照明非常昂贵，需要为很多点计算可见性。为了找到一个在计算时间和渲染质量之间的平衡，我们尝试使用SSAO。

在这里我们将SSAO用作一个单独的效果，只计算环境遮挡而不计算其他光照。

- ZShader

这个着色器主要用于渲染z-buffer，只关心深度，不关心颜色。

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

估算一个像素点与其周围环境的最大仰角，这是评估遮挡程度的关键函数。

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

- 环境光遮蔽的计算

对于每个像素，使用8个方向的射线来评估其环境遮挡程度。

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

项目完整代码，[链接🔗](https://github.com/Remyuu/Tiny-Renderer/tree/8_1_AO)。

![image-20230925172916074](/Users/remooo/Library/Application%20Support/typora-user-images/image-20230925172916074.png)