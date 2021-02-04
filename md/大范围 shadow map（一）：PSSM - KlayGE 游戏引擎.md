> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.klayge.org](http://www.klayge.org/2013/05/06/%E5%A4%A7%E8%8C%83%E5%9B%B4shadow-map%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9Apssm/)

大家一定很熟悉大场景的阴影渲染容易出现的锯齿问题，我这里就不废话了。这个问题常用的解决方式是 Cascaded Shadow Map（CSM），用一系列同样大小的 shadow map，每个管视锥的一个范围。现在大部分引擎也都支持 CSM，各种资料也很齐全，所以我这里不详细阐述原理了。想了解细节的可以看[原论文](http://dl.acm.org/citation.cfm?id=1128975&dl=ACM&coll=DL&CFID=213883521&CFTOKEN=15717279)，或者 [GPU Gems 3 的文章](http://http.developer.nvidia.com/GPUGems3/gpugems3_ch10.html)。

CSM 还是 PSSM
-----------

一个常见的疑问是为什么有的叫它 CSM，有的叫他 PSSM（Parallel-Split Shadow Map）。实际上原论文是把它叫做 PSSM，工业界实现的时候选择了个更广泛的名字 CSM。从名字本身来说，只要是一系列层级关系的 shadow map，就能叫 CSM，而只有平行切分视锥的才叫做 PSSM。换句话说，CSM 是个种类，PSSM 是具体方法。

PSSM
----

如前面所说，一般实现的 CSM 都是用的 PSSM。算法分为几个步骤：

1.  把视锥根据 log 的距离分布平行切成几个区域。
2.  计算每个区域的 scale 和 bias，组成 crop matrix 乘到 projection matrix 之后。
3.  对每一区域都生成一张 shadow map。
4.  在使用的时候，根据像素在视空间的位置确定自己是在哪个区域，以选择 shadow map。

除了步骤 2，其他都没什么特别的。步骤 2 的本质就是通过放缩和平移，把 shadow map“集中” 到看得见（视锥内）的区域。实现本身不难，这里就提几个容易遇到的小陷阱。

### 光锥的建立

在切分视锥之前，需要先建立 light 的 “视锥”（也就是光锥），用来包裹视锥。在不同的区域，这个光锥会用不同的 crop matrix 变换，以最大化 shadow map 使用率。由于 CSM 要处理的光源是阳光这样的非局部平行光，不像 spot light 那样可以直接给出光锥。在实践中，一个比较好的确定光锥 view matrix 的方法是：

1.  用视锥的中心为视点、光的方向为视线方向，视锥的 up 为上方向，算出一个初始的 view matrix
2.  把视锥的八个角点变换到 1 的 view space 内，算个 AABB
3.  把视点沿着光的方向移到 AABB 之外，重新建立 view matrix

这样得到的 view 能包裹住整个视锥，却又不会有太多多余的面积。projection matrix 是个平行投影，宽、高、远近平面也可以从上面的 2 里得到的 AABB 计算出来。

有了光锥的 view 和 projection，就能根据 PSSM 的算法来产生和使用 shadow map 了。

### 关闭 depth clip

第一次这么尝试之后，PSSM 的阴影在大部分时候都挺好，但在视线接近光的方向、以及视线逆着光的方向的时候，部分阴影会突然消失。查了很多地方都没有头绪，最后发现情况发生在，一个区域的物体不会投到另一个区域，因为在产生另一个区域的 shadow map 时，别的区域的物体已经被远近平面切掉了。解决方法是关闭掉远近平面的裁剪。在 D3D10+，这是通过关闭 depth clip 来做到，在 OpenGL，这是通过打开 depth clamp 来做到。类似的名字，相反的操作，要小心区分。

### shadow map 的产生

KlayGE 中的 shadow map 都是通过渲染到 depth texture，再做一次 depth to linear 的操作，把非线性的 depth 转换成线性的、view space 的 depth，同时计算 VSM 需要的 depth 平方。但原先的线性化公式仅仅适用于透视投影，平行投影需要另一个公式。于是就等于需要把 depth to linear、VSM 等好多个 shader 都加上支持平行投影的。与其如此，不如简化成在生成 shadow map 的时候就直接渲染到 VSM，把 view space depth 存出去，不经过线性化。但不排除以后找到更好的方法之后，恢复到原先方法的可能。

### 光锥裁剪

在产生每一个区域的 shadow map 是，按说可以根据 view * projection * crop 后的 matrix 做一次光锥裁剪，去掉不在光锥内的物体。但目前我的实现里，如果打开了这个，就会多裁一些物体，还没找到原因。

结果
--

解决或者绕开这几个坑之后，PSSM 就可以顺利使用了。分成 4 个区域的结果如下。残余的一些锯齿来自于 deferred shadowing，不是 PSSM 造成的。

[![](http://www.klayge.org/wp/wp-content/uploads/2013/05/PSSM-300x175.jpg)](http://www.klayge.org/wp/wp-content/uploads/2013/05/PSSM.jpg)

如果不分区域，结果就是这样：

[![](http://www.klayge.org/wp/wp-content/uploads/2013/05/PSSM_1_level-300x175.jpg)](http://www.klayge.org/wp/wp-content/uploads/2013/05/PSSM_1_level.jpg)

对 PSSM 做一些分析，可以发现它有两个算法上的缺点。第一个是，PSSM 对阴影效果影响很大的地方是光锥的集中程度。如果光锥需要包裹整个视锥，那么即便视锥的近平面上有面墙挡住了整个视野，光锥仍然不能集中到可见的 pixel 上。有一个优化方法是根据视锥内物体的 AABB 来提高光锥的集中效率，但如果遇上前面说的那面墙，情况并不会得到改善。另一个缺点是，决定区域划分的时候有一个参数，通过对它的调整可以改善近距离区域或者远距离区域的阴影质量，但不能两全其美。

下一篇打算介绍一下新的 CSM 方法：SDSM，能根据可见 pixel 来动态分配区域划分，完全解决掉这两个问题。