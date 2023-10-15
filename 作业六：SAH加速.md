这篇文章主要讲解表面积启发式(Surface Area Heuristic, SAH)策略在构建和遍历边界体层次结构（Bounding Volume Hierarchy, BVH）中的应用。文章的流程如下：

先编写一个轮子：包围盒体积光线求交算法。然后遍历场景中的所有包围盒。然后讲解传统BVH的构建，最后引出本文重点：SAH。

<!--more-->

## 使用包围盒体积优化光线求交

框架中的包围盒仅用了两个三维坐标来表示，分别是 `pMin` 和 `pMax` 。

这两个点在对角上，正好与6个面相接。因此，两个面两个面组合，用简化公式（二维）即可计算与平面的相交时间 `t` 。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/7pUKwzZrCNgMiYA.png" alt="image-20230613191117933" style="zoom: 67%;" />

简化公式如下：
$$
t_x=\frac{\mathbf{p}^{\prime}{ }_x-\mathbf{o}_x}{\mathbf{d}_x}\\
t_y=\frac{\mathbf{p}^{\prime}{ }_y-\mathbf{o}_y}{\mathbf{d}_y}\\
t_z=\frac{\mathbf{p}^{\prime}{ }_z-\mathbf{o}_z}{\mathbf{d}_z}
$$
首先计算射线在每个方向（x, y, z）上与边界盒最小点和最大点的交点所对应的参数值 `t1, t2`。

```c++
Vector3f t1 = (pMin - ray.origin) * invDir;
Vector3f t2 = (pMax - ray.origin) * invDir;
```

然后从 `t1, t2` 的各个分量中找出最大和最小的，然后分别写到 `tMax, tMin` 。

```c++
Vector3f tMin = Vector3f::Min(t1, t2);
Vector3f tMax = Vector3f::Max(t1, t2);
```

我们知道，进入AABB的时间要找最大，出去AABB的时间要找最小的。

```c++
tEnter = std::max(tMin.x, std::max(tMin.y, tMin.z));
tExit = std::min(tMax.x, std::min(tMax.y, tMax.z));
```

并且，射线是否与边界盒相交，需要满足 `t_enter < t_exit && t_exit >= 0` 。

也就是说，在这一段程序里我们已知包围体积AABB的两个顶点坐标  `pMin` 和 `pMax` ，并且传入了一条光线 `ray` 和光线的方向向量的倒数 `invDir` 。最终即可判定光线是否会穿过AABB体积。

## 应用BVH加速模型

我们可以直接使用刚才我们构建了的 `IntersectP` 函数，判断光线与场景中所有体积盒子的求交情况。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/RUXSv1tAydMkDiN.png" alt="image-20230614154411468" style="zoom:50%;" />

按照惯例，先看BVH.hpp中对于 `BVHBuildNode` 的的构造：

```c++
struct BVHBuildNode {
    Bounds3 bounds;
    BVHBuildNode *left;
    BVHBuildNode *right;
    Object* object;
    ......
```

一个BVH节点包含了包围盒体积的信息 `bounds` ，节点的左右孩子，还有当前包围盒体积里面的物体 `object` 。

函数传入的参数：

- `BVHBuildNode* node` ：当前BVH节点
- `const Ray& ray` ：当前需要判断的光线

我们现在要通过遍历边界体层次结构（Bounding Volume Hierarchy，简称BVH）来找到射线与三维对象（可能是三角形或其他形状）的交点。

- 首先判断当前节点 `node` 是否为空，并且判断传入的光线 `ray` 与当前节点的边界盒（即BVH节点的边界）是否有交点。如果节点是空且没有交点，那么函数直接返回一个默认的`Intersection`实例。这里的 `Intersection` 对象没有包含任何实际的交点信息。

- 否则，当前节点则是**叶子节点**或者**非终端节点的所有孩子都不可能与光线有交点**。那么进入下面的流程。
  - 如果是叶子节点，那么函数会直接计算射线与该节点所包含的实际几何对象的交点，然后返回结果。在这种情况下，`node->object->getIntersection(ray)`会调用光线-三角形求交方法对当前节点下的所有物体求交。
  - 如果不是叶子节点，那么函数会递归地计算射线与该节点的左子节点和右子节点的交点，返回两个值： `hit1, hit2` 。
- 然后，函数会返回离射线起点更近的那个交点作为结果。（ `hit1.distance < hit2.distance`）

```c++
Intersection BVHAccel::getIntersection(BVHBuildNode* node, const Ray& ray) const{
    Intersection intersection;
    if (node == nullptr || !node->bounds.IntersectP(ray, ray.direction_inv, {0,0,0}))
        return intersection;
    if (node->left == nullptr && node->right == nullptr)
        return node->object->getIntersection(ray);
    Intersection hit1 = getIntersection(node->left, ray);
    Intersection hit2 = getIntersection(node->right, ray);
    return  hit1.distance < hit2.distance ? hit1 : hit2;
}
```

## BVH的包围盒划分过程

在深入SAH之前，我们先了解如何使用传统的方法构建BVH，以及这种方法的优势与不足，借此引出SAH策略。

在我的[另一篇文章](https://remoooo.com/cg/9-1.html#ci_title27)中从宏观的角度简单地概括了BVH的构建流程。

这里从代码的角度讲解：

1. 在BVH.cpp的 `BVHAccel` 函数中创建了 `BVH` 数据结构的根节点 `root` 。

   并且函数会递归地构建一整颗 `BVH` 树。

   ```c++
   root = recursiveBuild(primitives);
   ```

2. 在 `recursiveBuild` 函数中，首先会创建一个新的`BVHBuildNode`对象，并计算给定的所有物体的联合边界（Union Bound），即包含**所有**物体的最小边界盒。

   1. 如果该边界盒的对象（object）数量为1，则创建一个叶子节点，其中包含这个唯一的物体 `object` 和它的边界 `bounds` ，然后返回这个节点。

      ```c++
      if (objects.size() == 1) {
          // Create leaf _BVHBuildNode_
          node->bounds = objects[0]->getBounds();
          node->object = objects[0];
          node->left = nullptr;
          node->right = nullptr;
          return node;
      }
      ```

   2. 如果该边界盒的对象（object）数量为2，则创建两个叶子节点，其中包含这个唯一的物体 `object` 和它的边界 `bounds` ，然后返回这个节点。

      ```c++
      else if (objects.size() == 2) {
          node->left = recursiveBuild(std::vector{objects[0]});
          node->right = recursiveBuild(std::vector{objects[1]});
          node->bounds = Union(node->left->bounds, node->right->bounds);
          return node;
      }
      ```

   3. 如果边界盒的对象数量大于两个，则继续递归。具体递归方式如下：

3. 首先计算所有物体质心（中心点）的联合边界。

   ```c++
   for (int i = 0; i < objects.size(); ++i)
       centroidBounds = Union(centroidBounds, objects[i]->getBounds().Centroid());
   ```

4. 然后找到这个边界在哪个维度（x, y, 或 z）上最大。

   ```c++
   int dim = centroidBounds.maxExtent();
   ```

5. 在此基础上，根据这个维度的质心坐标对所有物体进行**排序**。

   ```c++
   switch (dim) {
       case 0: ...
       case 1: ...
       case 2: ...
   }
   ```

6. 在排序后的物体列表中**找到中间位置**，并将列表分成两半。然后，对每一半的物体递归地调用这个函数，分别构建左子节点和右子节点。

   ```c++
   auto middling = objects.begin() + (objects.size() / 2);
   ...
   auto leftshapes = std::vector<Object*>(beginning, middling);
   auto rightshapes = std::vector<Object*>(middling, ending);
   ```

7. 最后，计算左子节点和右子节点的联合边界，即包含这两个子节点的最小边界盒，然后返回这个节点。

   ```c++
   node->bounds = Union(node->left->bounds, node->right->bounds);
   ```

除此之外还有一些技术细节需要补充：

- Union函数的实现：传入两个边界盒（或一个盒子和一个点），返回一个能包围他们的最小的新的边界盒。

- 每一个物体都有一个包围盒属性，可以调用每个物体的`getBounds()`函数来获取。构建BVH结构的过程就是创建一个层次结构的包围盒，其中每个节点的包围盒包含其所有子节点的包围盒。
- 框架中BVH.cpp文件的第85行 `assert(objects.size() == (leftshapes.size() + rightshapes.size()));` ，用于验证程序的某个条件是否为真。如果该条件为假，`assert`会发送一个错误消息，并终止程序的执行。这里实际作用是检查`objects`的大小是否等于`leftshapes`和`rightshapes`的大小之和。也就是说，检查原始的物体集合`objects`是否被正确地分割为两个子集`leftshapes`和`rightshapes`。
- 在最大的维度对物体进行排序排序是通过`std::sort`函数完成。这个函数传入 `objects.begin()` 和 `objects.end()` ，表示整个 `objects` 列表，和一个比较函数。比较函数是一个lambda表达式，它接受两个物体，然后返回一个布尔值，表示第一个物体是否应该在第二个物体之前。第三个参数是告诉 `sort` 函数如何进行排序，具体实现原理自行查阅百度。

## 使用SAH表面积启发式策略构建和遍历BVH

### 为什么会有SAH

- SAH可以使得光线-物体相交测试的预期次数最小。

Bounding Volume Hierarchy (BVH) 是一种常见的几何体组织和索引技术，被广泛应用于光线追踪等需要大量物体相交测试的图形应用中。但是构建BVH的方式对效率的影响非常的大，尤其是复杂的场景。因此，为了改进BVH的效率，研究人员提出了各种方法优化BVH，其中一种方法就是Surface Area Heuristic（SAH）。

### SAH是啥

- SAH是一种基于空间的划分。

首先先讲解闫老师提供的资料。[链接🔗](http://15462.courses.cs.cmu.edu/fall2015/lecture/acceleration/slide_024)。

### 概率估计

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/BNehmqbdRp1FYgV.png" alt="slide_024" style="zoom: 67%;" />

如果对象A位于对象B内部，那么打中对象B的任意射线也打中对象A的概率可以近似为它们的表面积的比值，即  $\frac{S_a}{S_b}$。

这是基于两个**假设**：

- 射线的入射方向是均匀分布的，即所有方向都同样可能
- 射线不会被遮挡

这个原理在SAH中被用来估计射线和各个包围盒相交的概率。通过比较不同分割方式的预期成本（即射线和包围盒相交的概率乘以相交测试的成本），我们就可以选择最优的分割方式高效地构建BVH以提高光线追踪的效率。

### 实现BVH树的划分

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/KnNEGZQPrtYL1MH.png" alt="slide_025" style="zoom: 67%;" />

在实现BVH树的划分时，通常会选择轴对齐的空间划分。对于给定的轴，如上图所示的一根横向的轴，我们在该轴上的划分平面。

**对于有N个物体的节点，为什么有2N-2个可能的划分位置呢？**

这是因为，对于每一个物体，我们都可以选择在它的质心的左侧或右侧作为划分平面，这就有2个可能的位置，换句话说，就是每一个物体都有一个 `x_min` 和 `x_max` 。但是，对于最左边和最右边的物体，我们不能选择在它们外侧的位置作为划分平面，因为这样会导致一个子包围盒为空。所以，对于最左边和最右边的物体，我们只能选择在它们的内侧作为划分平面。因此，总的可能的划分位置就是2N-2。上面图片已经非常清楚了。

尽管有这么多可能的划分位置，但在实际实现时，我们不需要真的尝试所有的划分位置。因为，对于大多数情况，我们可以通过一些启发式的方法（如SAH）来快速找到一个近似最优的划分位置。

### 分桶方法高效分区

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/fgW7qARB9QnSbXu.png" alt="slide_026" style="zoom:67%;" />

这里使用的是基于分桶（bucketing）的方法。

代码段会对每个轴 (x, y, z) 进行操作。

- 对于每个轴，首先初始化一个桶（bucket）：创建一个大小为B的桶数组。B通常比较小，例如小于32。

- 然后计算每一个物体p的质心（centroid），看看这个物体落在哪个桶里。

  - 将物体p的包围盒与桶b的包围盒做并集操作，也就是**扩展桶b的包围盒**，使其能够包含物体p。

  - 增加桶b中的物体计数。

- 对于每个可能的划分平面（总共有B-1个），使用**表面积启发式（SAH）公式评估**其成本。

- 执行成本最低的划分（如果找不到有效的划分，就将当前节点设为叶子节点）。

原本需要对所有物体的每一种可能划分进行评估，现在只需要对B-1个划分进行评估。因此，**分桶方法**可以在构建BVH时，有效地降低计算复杂度，提高算法的效率。

### 花费模型

回到最上面的ppt。

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/BNehmqbdRp1FYgV-20230627155427350.png" alt="slide_024" style="zoom: 67%;" />

SAH定义了一个花费模型，用于估算射线-对象相交的计算成本。这个模型假设，当射线穿过一个节点（或空间区域）时，将产生一定的花费（表示为 $C$ ）。这个花费与光线穿过子节点的概率 $p_x$ 成正比。另外，也需要考虑额外的花费（表示为 $C_{trav}$ ），比如计算射线与包围盒的相交等。

**划分策略**：SAH的目标是找到一种空间划分方式，使得整体花费最小。

假设有A个物体被划分到x子节点，B个物体被划分到y子节点，且假设穿过子节点的概率p与该节点的包围盒大小成正比。那么，空间划分的总花费C可以近似为：
$$
C=C_{\text {trav }}+\frac{S_A}{S_N} N_A C_{\text {isect }}+\frac{S_B}{S_N} N_B C_{\text {isect }}
$$
其中， $S_A$ 、 $S_B$ 分别表示x、y子节点的表面积， $S_N$ 表示整个节点的表面积， $N_A$ 、$N_B$ 分别表示x、y子节点中的物体数量， $C_{\text {isect }}$ 表示射线与物体相交的计算成本。

**空间划分**：为了实现上述的优化目标，利用**桶分类**即可加速分区。

### 实际代码

```c++
BVHBuildNode *BVHAccel::recursiveBuild(std::vector<Object *> objects) {
    BVHBuildNode *node = new BVHBuildNode();

    // Compute bounds of all primitives in BVH node
    Bounds3 bounds;
    for (int i = 0; i < objects.size(); ++i)
        bounds = Union(bounds, objects[i]->getBounds());
    if (objects.size() == 1) {
        // Create leaf _BVHBuildNode_
        node->bounds = objects[0]->getBounds();
        node->object = objects[0];
        node->left = nullptr;
        node->right = nullptr;
        return node;
    } else if (objects.size() == 2) {
        node->left = recursiveBuild(std::vector{objects[0]});
        node->right = recursiveBuild(std::vector{objects[1]});

        node->bounds = Union(node->left->bounds, node->right->bounds);
        return node;
    } else {
        Bounds3 centroidBounds;
        for (int i = 0; i < objects.size(); ++i)
            centroidBounds =
                    Union(centroidBounds, objects[i]->getBounds().Centroid());
        int maxExtentDimension = centroidBounds.maxExtent();
        switch (SplitMethod::SAH) {
            case SplitMethod::NAIVE: {

                switch (maxExtentDimension) {
                    case 0:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().x <
                                   f2->getBounds().Centroid().x;
                        });
                        break;
                    case 1:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().y <
                                   f2->getBounds().Centroid().y;
                        });
                        break;
                    case 2:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().z <
                                   f2->getBounds().Centroid().z;
                        });
                        break;
                }

                auto beginning = objects.begin();
                auto middling = objects.begin() + (objects.size() / 2);
                auto ending = objects.end();

                auto leftshapes = std::vector<Object *>(beginning, middling);
                auto rightshapes = std::vector<Object *>(middling, ending);

                assert(objects.size() == (leftshapes.size() + rightshapes.size()));

                node->left = recursiveBuild(leftshapes);
                node->right = recursiveBuild(rightshapes);

                node->bounds = Union(node->left->bounds, node->right->bounds);
                break;
            }
            case SplitMethod::SAH: {
                float totalSurfaceArea = centroidBounds.SurfaceArea();
                int bucketSize = 10;
                int optimalSplitIndex = 0;
                float minCost = std::numeric_limits<float>::infinity(); //最小花费

                // Sort objects based on their centroid along the max extent dimension
                switch (maxExtentDimension){
                    case 0:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().x <
                                   f2->getBounds().Centroid().x;
                        });
                        break;
                    case 1:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().y <
                                   f2->getBounds().Centroid().y;
                        });
                        break;
                    case 2:
                        std::sort(objects.begin(), objects.end(), [](auto f1, auto f2) {
                            return f1->getBounds().Centroid().z <
                                   f2->getBounds().Centroid().z;
                        });
                        break;
                }
                
                // Find the split that minimizes the SAH cost
                for (int i = 1; i < bucketSize; i++) {
                    auto beginning = objects.begin();
                    auto middling = objects.begin() + (objects.size() * i / bucketSize);
                    auto ending = objects.end();
                    auto leftshapes = std::vector<Object *>(beginning, middling);
                    auto rightshapes = std::vector<Object *>(middling, ending);
                    //求左右包围盒:
                    Bounds3 leftBounds, rightBounds;
                    for (int k = 0; k < leftshapes.size(); ++k)
                        leftBounds = Union(leftBounds, leftshapes[k]->getBounds().Centroid());
                    for (int k = 0; k < rightshapes.size(); ++k)
                        rightBounds = Union(rightBounds, rightshapes[k]->getBounds().Centroid());
                    float SA = leftBounds.SurfaceArea(); //SA
                    float SB = rightBounds.SurfaceArea(); //SB
                    float cost = 0.125 + (leftshapes.size() * SA + rightshapes.size() * SB) / totalSurfaceArea; //计算花费
                    if (cost < minCost){//如果花费更小，记录当前坐标值
                        minCost = cost;
                        optimalSplitIndex = i;
                    }
                }
                //找到optimalSplitIndex后的操作等同于BVH
                auto beginning = objects.begin();
                auto middling = objects.begin() + (objects.size() * optimalSplitIndex / bucketSize);//划分点选为当前最优桶的位置
                auto ending = objects.end();
                auto leftshapes = std::vector<Object *>(beginning, middling); //数组切分
                auto rightshapes = std::vector<Object *>(middling, ending);

                assert(objects.size() == (leftshapes.size() + rightshapes.size()));

                node->left = recursiveBuild(leftshapes); //左右开始递归
                node->right = recursiveBuild(rightshapes);
                node->bounds = Union(node->left->bounds, node->right->bounds);//返回pMin pMax构成大包围盒

            }
        }
    }
    return node;
}
```

## 附录 1.

### 糟糕的情况



<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/ESKmJdiG93VHgPD.png" alt="slide_027" style="zoom:67%;" />

- 左边的情况：所有物体都有相同的质心。这种情况可能会导致树不平衡，因为所有的基元最终会在同一个分区中。在最坏的情况下，这可能导致树更像**链表**而不是平衡二叉树。
- 右边的情况：所有的物体都具有相同的边界框AABB。这种情况可能会导致低效的遍历，因为与边界框相交的光线可能会访问两个分区，而不管实际与基元本身相交。

### 原始划分（基于边界体层次结构）vs. 空间划分（如网格、KD树）

<img src="https://regz-1258735137.cos.ap-guangzhou.myqcloud.com/remo_t/hPFn7qQAw9VTipo.png" alt="slide_028" style="zoom:67%;" />

- **原始划分**（例如：基于边界体的层次结构，BVH）：这种方式容易导致AABB重叠。但BVH的优势是，更容易构建，因为它们只需要基于原始数据进行划分，而不是基于空间。在某些情况下，BVH可能比其他类型的加速结构具有更好的性能。

- **空间划分**（例如：网格，K-D树）：这种方式将空间划分为不重叠的区域（原始数据可能存在于多个空间区域中）。空间划分通常可以提供高效的射线追踪性能，尤其是对于静态场景。然而，如果场景中的对象移动，这种类型的加速结构可能需要完全重建，这会导致效率下降。

## Reference

1. Fundamentals of Computer Graphics 4th
2. GAMES101 Lingqi Yan
3. https://zhuanlan.zhihu.com/p/475966001
4. http://15462.courses.cs.cmu.edu/fall2015/lecture/acceleration/slide_024
5. https://cs184.eecs.berkeley.edu/sp23/lecture/9-90/intro-to-ray-tracing-and-acceler
