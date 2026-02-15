# Shading from environment lighting
Rendering Equation:

我们暂时不考虑visibility项,那么rendering equation就只是brdf项和lighting项相乘再积分。
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/8b0d1035-ee4e-48e3-906c-034ab553dd45" />

games101里有讲过用蒙特卡洛sample的方法求解，这种方法的缺点是有些费时

接下来我们提出新的方法分析：
## Lighting 项
brdf又分为两种情况:

1.brdf为glossy时,覆盖在球面上的范围很小,也就是small support(积分域).

2.brdf为diffuse时,它会覆盖整个半球的区域,但是是smooth的,也就是值的变化不大,就算加上cos也是相对平滑的.

因此可以单独拆出rendering equation的光照项

<img width="888" height="123" alt="image" src="https://github.com/user-attachments/assets/7a2ab0c0-4222-4574-9d51-d0d00e34e417" />

把light项拆分出来,然后将brdf范围内的lighting积分起来并进行normalize,其实就是将IBL这张图给模糊了.模糊就是在任何一点上取周围一片范围求出范围内的平均值并将平均值写回这个点上

而这些模糊过的图是我们在进行rendering之前生成的,也就是pre-filtering,Prefiltering就是提前把滤波环境光生成，提前将不同卷积核的滤波核的环境光生成一系列模糊过的图（和mipmap相似），当我们需要时进行查询即可,其他尺寸的图则可以经过这些已生成的通过三线性插值得到.

<img width="831" height="249" alt="image" src="https://github.com/user-attachments/assets/c42f0977-fb4e-407e-9570-7d37c73432df" />

左图为brdf求shading point值时,我们要以一定立体角的范围内进行采样再加权平均从而求出shading pointd的值.

右图为我们从镜面反射方向望去在pre-filtering的图上进行查询,由于图上任何一点都是周围范围内的加权平均值,因此在镜面反射方向上进行一次查询就等价于左图的操作,并且不需要采样,速度更快.

## BRDF项
拆出来了lightning项后我们接下来算关于brdf项的积分

$\int_{\Omega^+} f_r(p, \omega_i, \omega_o)\cos\theta_i \, d\omega_i$

我们仍然可以用预计算来解决后半部分积分采样的问题，但这样参数很多，存储量爆炸

在microfacet brdf中中，考虑的是菲涅尔项、阴影项以及法线项，由于此时暂时不考虑阴影，此处需要关注的是Fresnel term和distribution of normals。

Frenel term可以近似成一个基础反射率R0和入射角度的指数函数

法线分布函数是一个一维的分布，其中有两个变量，一个变量定义是diffuse还是gloosy，另一个 是half vector和法线中间的夹角，可以近似成入射角度相关的数，这样就变成了3维的预计算。

(在这里做了一个近似，入射角=反射角=half vector和n的夹角，不是很明白为什么能这么近似)

至此我们有了三个变量:基础反射率r0,roughness 和角度

接下来我们还要想办法减少参数量

菲涅尔项的Schlick近似公式为：

$F = F_0 + (1 - F_0)(1 - (v \cdot h))^5$

brdf还可写作

$$
f_r = \frac{f_r}{F} \cdot F
$$

把它带入Brdf项积分化简有：

$\int_{\Omega^+} f_r(p,\omega_i \omega_o)\cos\theta_i\ d\omega_i \approx R_0 \int_{\Omega^+} \frac{f_r}{F}(1-(1-\cos\theta_i)^5)\cos\theta_i\ d\omega_i + \int_{\Omega^+} \frac{f_r}{F}(1-\cos\theta_i)^5 \cos\theta_i\ d\omega_i$

基础反射R0被拆出积分式，需要预计算的两个量就只有roughness 和角度 
### 拆分后的物理意义

经过 Schlick Fresnel 分离后，积分结果可以写成：

$$
\text{Result} = F_0 \cdot A + B
$$

其中：

- $F_0$：材质的正视角反射率（与材质颜色有关）
- $A, B$：两个与材质颜色无关的项


#### A 和 B 的依赖关系

$A$ 和 $B$ 只与以下几何参数有关：

- **roughness**（表面粗糙度）
- **view angle**（通常表示为 $N \cdot V$）

它们 **不依赖于材质颜色或金属度**。



这意味着：

- Fresnel 中的材质信息（$F_0$）被单独提取出来
- BRDF 积分中剩余部分只与几何关系有关
- 因此可以对 $A$ 和 $B$ 进行预计算


### 预计算（Precomputation）

可以离线生成一个 **二维查找表（2D LUT）**：

| 输入参数 | 含义 |
|---|---|
| roughness | 表面粗糙度 |
| $N \cdot V$ | 视角与法线的夹角 |

| 输出 |
|---|
| $A$ |
| $B$ |

运行时只需：

$$
\text{Specular} = F_0 \cdot A + B
$$

即可快速得到镜面反射结果。

## PRT(Precomputed Radiance Transfer)
<img width="904" height="405" alt="image" src="https://github.com/user-attachments/assets/70bc8998-f891-457d-8a46-5292649a0ad3" />
PRT 的关键观察：

如果物体不动，那么
可见性 + 几何 + BRDF 可以提前算好

PRT的基本思想:

我们把rendering equation分为两部分,lighting 和 light transport.
<img width="695" height="251" alt="image" src="https://github.com/user-attachments/assets/7fae1979-21fb-4c17-ad57-d65821d60b91" />

假设在渲染时场景中只有lighting项会发生变化(旋转,更换光照等),由于lighting是一个球面函数,因此可以用基函数来表示,在预计算阶段计算出lighting.

而light transport(visibility和brdf)是不变的(因为摄像机不变，物体不运动),因此相当于对任一shading point来说,light transport项固定的,可以认为是shading point自己的性质,light transport总体来说还是一个球面函数,因此也可以写成基函数形式,是可以预计算出的.

我们分为两种情况,diffuse和glossy:

### Diffuse:
<img width="947" height="454" alt="image" src="https://github.com/user-attachments/assets/b51528df-3aa2-478e-9c35-7eef06db5b13" />

由于在diffuse情况下,brdf几乎是一个常数,因此我们把brdf提到外面.

由于lighting项可以写成基函数的形式,因此我们求和式把其代入积分中,对于任何一个积分来说,在Bi的限制下,li此对积分来说是常数,可以提出来.

对于积分中的部分来说,Bi是基函数,v和cos项在一起就是light transport吗,那就是light transport乘一个基函数，这就成了lighting transport投影到一个基函数的系数，代入就能进行预计算了。


所以对于任何一个shading point我们去算他的shading 和 shadow,只需要计算一个点乘就可以了,十分方便,

这个方法的局限性是

1.物体和摄像机不能动

2.对于预计算的光源我们把它投影到sh上,如果光源发生了旋转,那就相当于换了个光源

第二个问题由于sh函数的旋转不变性可以完美的解决.

旋转光照 = 旋转SH的基函数

但任何一个SH基函数旋转后都可以被同阶的SH基函数线性组合表示出来

因此,我们根据这个性质,还是可以立刻得出旋转后的sh基函数新的线性组合.

PRT预计算了两部分,一部分是lighting项,一部分是除了lighting项以外的部分,称其为light transports项.

1.brdf是diffuse相当于是一个常数,因此可以提到外面.

2.由于环境光照是一个球面函数,因此可以用sh基函数的线性组合表示,把其代入,由于li相对于积分是一个常数,可以提出.

3.剩下的light transport项也可以看作一个球面函数,现如今积分里剩基函数和球面函数,相当于函数投影到基函数求系数,因此变为了一个系数.


### Glossy：
Diffuse和glossy的区别在于,diffuse的brdf是一个常数,而glossy的brdf是一个4维的brdf(2维的输入方向,2维的输出方向).

因此计算glossy的rendering equation比diffuse多了一个out lighting的两个维度

light transport即使投影到了i方向上的基函数,所得到的仍然是一个关于O的函数而不是系数.

<img width="877" height="599" alt="image" src="https://github.com/user-attachments/assets/49800380-6001-4818-8d69-b433a91b1830" />

我们将4D的函数投影在2D上之后,虽然得到的是一个关于O的函数,但是现在这个函数也只是关于O了,因此我们在O的方向上将其投影到SH基函数上.

因此,light transport上就不再认为得到的是向量了,而是一个矩阵,也就是对于任意一个O都会得到一串VECTOR,最后把所有不同O得到的VECTOR摆在一起,自然而然就形成了一个矩阵.

显然这样的话将会产生巨大的存储.

这里看出来PRT Glossy比Diffuse 效率要差很多,而当Glossy非常高频的时候,也就是接近镜面反射的情况的时候, PRT就没有那么好用，我们虽然可以采用更高阶的SH来描述高频信息,但是使用SH甚至远不如直接采样方便。

# Homework

<img width="452" height="97" alt="image" src="https://github.com/user-attachments/assets/86b0bb20-fa2b-4859-9ed1-5b96ef021e62" />

对任意定义在球面上的函数，把它与每个球谐基函数做球面积分（内积），得到对应的系数，这样就完成了球谐展开。

在实际中通常通过采样和加权求和进行数值近似。
从渲染方程出发：

Lo(p, ωo) = ∫_{Ω+} Li(p, ωi) * fr(p, ωi, ωo) * cosθi * V(p, ωi) dωi

PRT 的做法是把积分里的两个部分分别预计算并投影到一组基函数上，然后在运行时做点积。

--------------------------------------------------

1. 光照项（light.txt）

对入射光 Li(p, ωi) 在整个球面上做积分：

Li_coeff ≈ Σ_pixels Li(p, ωi) * Δωi

其中：
Δωi = CalcArea(...)  
用于近似立体角 dωi

也就是用 cubemap 像素来近似：

∫ Li(p, ωi) dωi

这一部分只与环境贴图有关，不依赖模型。

--------------------------------------------------

2. 传输项（transport.txt）

把几何相关部分作为一个函数：

T(p, ωi) = fr(p, ωi, ωo) * cosθi * V(p, ωi)

对于 Lambert：

fr = 1 / π

所以：

T(p, ωi) = max(0, n · ωi) / π * V(p, ωi)

代码中通过随机采样方向来近似积分：

Ti_coeff ≈ (4π / N) * Σ_k T(p, ωk)

这一部分依赖：
- 法线方向
- 遮挡（是否相交）

对于多次bounce的情况，作业里简化为无损耗的全反射，因此最后结果只要直接加上之前hit point的球谐函数的系数就可

--------------------------------------------------

3. 运行时

最终颜色为两者的内积：

Lo(p, ωo) ≈ Light_coeff · Transport_coeff

等价于对原始渲染方程的快速近似。

## 一些数学原理
### delta area的计算
参考https://www.rorydriscoll.com/2012/01/15/cubemap-texel-solid-angle/
### 纹理上的uv映射到球坐标phi theta
#### 1. α、β 的作用

```cpp
double alpha = (t + rng(gen)) / sample_side;
double beta  = (p + rng(gen)) / sample_side;
```

这一步将离散网格加随机抖动转换为连续随机变量：

$$
\alpha, \beta \sim U(0, 1)
$$

即：

$$
(\alpha, \beta) \in [0,1]^2
$$

接下来需要将二维均匀分布映射到单位球面。

---

#### 2. 方位角 $\phi$

```cpp
double phi = 2.0 * M_PI * beta;
```

因为：

$$
\phi \in [0, 2\pi)
$$

若 $\beta$ 均匀分布，则：

$$
\phi = 2\pi \beta
$$

可以保证经度方向均匀。

---

#### 3. 极角 $\theta$

很多人会误写为：

$$
\theta = \pi \alpha
$$

但这是错误的，因为球面面积对 $\theta$ 不是线性的。

---

##### 3.1 球面面积元素

单位球面的面积元素为：

$$
dA = \sin\theta \, d\theta \, d\phi
$$

说明：

- 赤道附近面积大  
- 两极附近面积小  

如果 $\theta$ 均匀采样，点会在两极聚集。

---

##### 3.2 正确方法：令 $\cos\theta$ 均匀

令：

$$
u = \cos\theta
$$

则：

$$
du = -\sin\theta \, d\theta
$$

代入面积元素：

$$
dA = \sin\theta \, d\theta \, d\phi = d\phi \, du
$$

面积分布变为线性。

因此：

$$
u \sim U(-1, 1)
$$

再设：

$$
\theta = \arccos(u)
$$

即可得到球面均匀分布。

---
