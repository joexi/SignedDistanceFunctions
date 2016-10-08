# Volumetric Rendering: Signed Distance Functions
# 体积渲染：有向距离函数

This tutorial explains how to create complex 3D shapes inside volumetric shaders. **Signed Distance Functions** (often referred as **Fields**) are mathematical tools used to describe geometrical shapes such as sphere, boxes and tori. Compared to traditional 3D models made out of triangles, signed distance functions provide virtually **infinite resolution**, and are amenable to geometric manipulation. The following animation, from [formulanimation tutorial :: making a snail](https://www.youtube.com/watch?v=XuSnLbB1j6E), shows how a snail can be created using simpler shapes:

[](https://github.com/joexi/SignedDistanceFunctions/blob/master/a74cc1b8-890d-4ebd-b1a2-4837dda9c66f.png)

 本片教程介绍了如何在体积着色器中创建复杂的三维模型。**有向距离函数**（通常被称为**场**）是用来描述球形，盒子和环面的几何形状的数学工具。和传统的由三角形组成的3D模型相比，有向距离函数提供了几乎**无限的分辨率**，并且适合进行计划操作。下面的动画来自于[动画教程：制作一个蜗牛](https://www.youtube.com/watch?v=XuSnLbB1j6E)，展示了如何使用更简单的形状去创建一个蜗牛。

* Introduction
    简介
* Part 1. SDF Sphere
    SDF球形
* Part 2. Union and Intersection
    并集和交集
* Part 3. SDF Box
    SDF盒子
* Part 4. Shape Blending
    形状混合
* Part 5. Smooth Union
    平滑合并
* Part 6. SDF Algebra
    SDF代数
* Conclusion
    结论



You can find here all the other posts in this series:
你可以在这个列表中找到其他所有的帖子:
* Part 1. [Volumetric Rendering](http://www.alanzucconi.com/2016/07/01/volumetric-rendering/)
    体积渲染
* Part 2. [Raymarching](http://www.alanzucconi.com/2016/07/01/raymarching/)
    光路进行
* Part 3. [Surface Shading](http://www.alanzucconi.com/2016/07/01/surface-shading/)
    表面着色
* Part 4. **Signed Distance Fields**
    **有向距离场**
* Part 5. [Ambient Occlusion](http://www.alanzucconi.com/2016/07/01/ambient-occlusion/)
    环境遮挡
* Part 6. [Hard and Soft Shadows](http://www.alanzucconi.com/2016/07/01/shadow/)
    硬阴影和软阴影
* Part 7. [Volume Raycasting](http://www.alanzucconi.com/2016/07/01/volume/)
    体积光线投射

## Introdunction
## 介绍
The way most modern 3D engines – such as Unity – handle geometries is by using triangles. Every objects, no matter how complex, must be composed of those primitive triangles. Despite being the de-facto standard in computer graphics, there are objects which cannot be represented with triangles. Spheres, and all other curved geometries, are impossible to tessellate with flat entities. It is indeed true that we can approximate a sphere by covering its surface with lot of small triangles, but this comes at the cost of adding more primitives to draw.
大多数现代的3D引擎-例如Unity-都使用三角形来处理几何。每一个物体，无论多复杂，都是由原始的三角形所组成。尽管这在计算机图形学中是实际的标准，任然有对象不能用三角形表示。球形以及其他曲面几何无法被细分为平面实体。我们可以通过在表面覆盖大量的小三角形来得到一个近似的球体，但这会增加更多的绘制成本。

Alternative ways to represent geometries exist. One of this uses **signed distance functions**, which are mathematical descriptions of the objects we want to represent. When you replace the geometry of a sphere with its very equation, you have suddenly removed any approximation error from your 3D engine. You can think of signed distance fields as the SVG equivalent of triangles. You can scale up and zoom SDF geometries without ever losing detail. A sphere will always be smooth, regardless how close you are to its edges.

有用于表现几何形状的替代方法。其中一个就是使用**有向距离函数**，就是我们要代表的对象的数学描述。当你用一个球体的方程来取代它的几何形状时，你就会突然把所有的误差错误从3D引擎中移除。你可以把有向距离函数视作SVG等量的三角形。你可以缩放和变焦SDF的几何形状而不丢失它的细节。球体永远都是光滑的，无论你离他的边缘多么地近。

Signed distance functions are based on the idea that every primitive object must be represented with a function. It takes a 3D point as a parameter, and returns a value that indicates how distant that point is to the object surface.
有向距离函数是基于这种一种思想，即每一个原始对象都必须有一个对应的函数。它以3D坐标作为参数，并返回一个值，表明了这个点距离物体表面的距离。

## SDF Sphere
## SDF 球体
In the first post of this series, [Volumetric Rendering](http://www.alanzucconi.com/2016/07/01/volumetric-rendering/), we’ve seen a hit function that indicates if we are inside a sphere or not:
在这个系列的第一部分[体积渲染](http://www.alanzucconi.com/2016/07/01/volumetric-rendering/)中，我们看到了一个表明我们是否在球体内的碰撞函数：
``` c
bool sphereHit (float3 p)
{
    return distance(p,_Centre) < _Radius;
}
```
We can change this function so that it returns the distance from the sphere surface instead:
我们可以改变这个函数让他返回距离球体表面的距离
``` c
float sdf_sphere (float3 p, float3 c, float r)
{
    return distance(p,c) - r;
}
```
If sdf_spher returns a positive distance, we’re not hitting the sphere. A negative distance indicates that we are inside the sphere, while zero is reserved for the points of the space which actually make up the surface.
如果sdf_spher 返回了一个正的距离，那么我们就没有碰撞到球体。负责距离则表示我们是在球体内，而零保留的实际上就是表面上的点。
## Union and Intersection
## 并集和交集

The concept of signed distance function was briefly introduced in [Raymarching tutorial](http://www.alanzucconi.com/2016/07/01/raymarching/), where it guided the advancement of the camera rays into the material. There is another reason why SDFs are used. And it is because they are amenable to composition. Given the SDFs of two different spheres, how can we merge them into a single SDF?

[光线行进](http://www.alanzucconi.com/2016/07/01/raymarching/)指南在介绍摄像机射线到材质的进阶部分中简要的介绍了有向距离函数的概念。这也是另一个使用有向距离函数的理由。也是因为他们是适合进行组合的。给出了2个不同球形的SDFs，我们该如何将他们合并成一个呢？
 
We can think about this from the perspective of a camera rays, advancing into the material. At each step, the ray must find its closest obstacle. If there are two spheres, we should evaluate the distance from both and get the smallest. We don’t want to overshoot the sphere, so we must advance by the most conservative estimation.
我们可以从相机射线的角度来考虑这个问题。在每一步中，射线必须找到它最接近的障碍物。如果有两个球体，我们需要评估两者的距离得到最小值。我们不希望在球体上作超量的开销，所以我们必须作最保守的评估。

 
This toy example can be extended to any two SDFs. Taking the minimum value between them returns another SDF which corresponds to their union:
这个玩具的例子可以扩展到任何两个SDF，获取他们之间的最小值并返回另一个相当于他们并集的SDF：
``` c
float map (float3 p)
{
    return min
        (
            sdf_sphere(p, - float3 (1.5, 0, 0), 2), // Left sphere
            sdf_sphere(p, + float3 (1.5, 0, 0), 2)  // Right sphere
        );
}
```
The result can be seen in the following picture (which also features few other visual enhancements that will be discussed in the next post on [Ambient Occlusion](http://www.alanzucconi.com/2016/07/01/ambient-occlusion/)):
结果可以在下面的图片中看到（它还具有其他视觉增强功能，将在下一次关于[环境遮挡](http://www.alanzucconi.com/2016/07/01/ambient-occlusion/)的帖子里讨论）

[](https://github.com/joexi/SignedDistanceFunctions/blob/master/24ff4d34-856c-4ccb-a839-6cf05947c433.png)

With the same reasoning, it’s easy to see that taking the maximum value between two SDFs returns their intersection:
因为同样的原因，显而易见的获取两个SDF的最大值则返回了他们的交集。
``` c
float map (float3 p)
{
    return max
        (
            sdf_sphere(p, - float3 (1.5, 0, 0), 2), // Left sphere
            sdf_sphere(p, + float3 (1.5, 0, 0), 2)  // Right sphere
        );
}
```

[](https://github.com/joexi/SignedDistanceFunctions/blob/master/177f8918-4805-4149-9699-89914fa75d8c.png)

##  SDF Box
##  SDF 盒子
Many geometries can be constructed with what we already know. If we want to push out knowledge further, we need to introduce a new SDF primitive: [the half-space](https://en.wikipedia.org/wiki/Half-space_(geometry)). As the name suggests, it is nothing more than just a primitive that occupies half of the 3D space.
很多几何图形可以以我们已知的方式进行构建。如果我们想把知识更进一步，我们需要引入一个新的SDF原型：[半空间](https://en.wikipedia.org/wiki/Half-space_(geometry))。就像名字所显示的那样，它只是一个原始的占据了半个3D空间的东西。
``` c
// X Axis
d = + p.x - c.x; // Left half-space full
d = - p.x + c.x; // Right half-space full
 
// Y Axis
d = + p.y - c.y; // Left half-space full
d = - p.y + c.y; // Right half-space full
 
// Z Axis
d = + p.z - c.z; // Left half-space full
d = - p.z + c.z; // Right half-space full
```

The trick is to intersect six planes in order to create a box with the given size s, like shown in the animation below:
关键点是要和6个平面相交，以创建一个给定大小的盒子，就如下面动画所示：
``` c
float sdf_box (float3 p, float3 c, float3 s)
{
    float x = max
    (   p.x - _Centre.x - float3(s.x / 2., 0, 0),
        _Centre.x - p.x - float3(s.x / 2., 0, 0)
    );
 
    float y = max
    (   p.y - _Centre.y - float3(s.y / 2., 0, 0),
        _Centre.y - p.y - float3(s.y / 2., 0, 0)
    );
 
    float z = max
    (   p.z - _Centre.z - float3(s.z / 2., 0, 0),
        _Centre.z - p.z - float3(s.z / 2., 0, 0)
    );
 
    float d = x;
    d = max(d,y);
    d = max(d,z);
    return d;
}
```

[](https://github.com/joexi/SignedDistanceFunctions/blob/master/f28ffff7-4b77-4008-8579-0827b7bcc304.png)

There are more compact (yet less precise) ways to create a box, which take advantage of the symmetries around the centre:
有更简洁（但不够精确）的方法来创建一个盒子，利用了中心周围的对称性。
``` c
float vmax(float3 v)
{
    return max(max(v.x, v.y), v.z);
}
float sdf_boxcheap(float3 p, float3 c, float3 s)
{
    return vmax(abs(p-c) - s);
}
```

## Shape Blending
## 形状混合

If you are familiar with the concept of **alpha blending**, you will probably recognise the following piece of code:
如果你熟悉**alpha混合**的概念，你可能会认得下面的代码：
``` c
float sdf_blend(float d1, float d2, float a)
{
    return a * d1 + (1 - a) * d2;
}
```

It’s purpose is to create a blending between two values, d1 and d2 , controller by a value a (from zero to one). The exact same code use to blend colours can also be used to blend shapes. For instance, the following code blends a sphere into a cube:
这么做的目的是创建d1和d2两个值之间的混合，通过a的值（从0到1）进行控制。用于混合颜色的代码也可以用于混合形状。例如，下面的代码将一个球体混合到一个立方体中：

``` c
d = sdf_blend
(
    sfd_sphere(p, 0, r),
    sfd_box(p, 0, r),
    (_SinTime[3] + 1.) / 2.
);
```

## Smooth Union
## 光滑合并
In a previous section we’ve seen how two SDFs can be merged together using min. If it is true that SDF union is indeed effective, it is also true that its results is rather unnatural. Working with SDFs allows for many ways in which primitives can be blended together. One of this technique,** exponential smoothing** (link: [Smooth Minimum](http://www.iquilezles.org/www/articles/smin/smin.htm)), has been used extensively in the original animations  of this tutorial.
在上一章节中我们已经看到两个SDF可以通过取最小值的方式合并在一起。SDF的合集虽然确实是有效的，但它的结果会有一点不真实。SDF可以将原物体以多种方式混合在一起。其中一个技巧就是，**指数平滑**（链接：[最小光滑](http://www.iquilezles.org/www/articles/smin/smin.htm)）已经被广泛使用在本教程的原始动画中。
``` c
float sdf_smin(float a, float b, float k = 32)
{
    float res = exp(-k*a) + exp(-k*b);
    return -log(max(0.0001,res)) / k;
}
```

When two shapes are joined using this new operator, they merge softly, creating a gentle step that removes any sharp edge. In the following animation, you can see how the spheres merges together:
当两个形状以这个新的操作进行结合时，他们会平滑地合并，创建一个温和的步骤去移除所有锋利的边缘。在下面的动画中，你可以看到球体们是如何合并在一起的：

## SDF Algebra
## SDF 代数
As you can anticipate, all those SDF primitives and operators are part of a signed distance function algebra. Rotations, scaling, bending, twisting… all those operations can be performed with signed distance functions.
 可以预见的是，那些SDF元物体以及操作是有向距离函数代数的一部分。旋转，缩放，混合，扭曲...所有这些操作都可以被有向距离函数所表示。

In his article title [Modeling With Distance Functions](http://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm), [Íñigo Quílez](https://twitter.com/iquilezles) has worked on a vast collection of SDFs that can be used as primitive for the construction of more complex geometries. You can see some of them by clicking in the interactive **ShaderToy** below:
在他名为《[使用有向距离函数建模](http://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm)》的文章中，[Íñigo Quílez](https://twitter.com/iquilezles)曾致力于一个巨大的SDF集合，提供元物件用于构建更复杂的几何体。你们可以通过点击下面的可交互**ShaderToy**进行查看：

An even larger collection of primitives and operators is available in the library **hg_sdf** (link [here](http://mercury.sexy/hg_sdf/)) curated by the [MERCURY](https://twitter.com/mercury_labs) group. Despite being written in GLSL, the functions are easily portable to Unity’s Cg/HLSL.
一个更大的元物件和操作的集合在[MERCURY](https://twitter.com/mercury_labs)团队创建的**hg_sdf**（[链接](http://mercury.sexy/hg_sdf/)）库中。虽然是由GLSL写的，但是函数可以很方便的移植到Unity的Cg/HLSL中。

## Conclusion
## 结论
The number of transformations that can be performed with SDFs is virtually endless. This post provided just a quick introduction to the topic. If you really want to master volumetric rendering, improving your knowledge of SDFs is a good starting point.
可以通过SDF表达的转换数量几乎是无限的。这篇文章只是提供一个该主题的简介。如果你真的想掌握体积渲染，增加你对SDF的了解是一个很好的起点

## Other Resources
* Part 1. [Volumetric Rendering](http://www.alanzucconi.com/2016/07/01/volumetric-rendering/)
    体积渲染
* Part 2. [Raymarching](http://www.alanzucconi.com/2016/07/01/raymarching/)
    光路进行
* Part 3. [Surface Shading](http://www.alanzucconi.com/2016/07/01/surface-shading/)
    表面着色
* Part 4. **Signed Distance Fields**
    **有向距离场**
* Part 5. [Ambient Occlusion](http://www.alanzucconi.com/2016/07/01/ambient-occlusion/)
    环境遮挡
* Part 6. [Hard and Soft Shadows](http://www.alanzucconi.com/2016/07/01/shadow/)
    硬阴影和软阴影
* Part 7. [Volume Raycasting](http://www.alanzucconi.com/2016/07/01/volume/)
    体积光线投射
* [Raymarching Distance Fields](http://9bitscience.blogspot.co.uk/2013/07/raymarching-distance-fields_14.html)

* [Modeling with Signed Distance Fields](http://www.iquilezles.org/www/articles/sdfmodeling/sdfmodeling.htm)
