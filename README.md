# Real-Time Physically-Based Materials
PBR在概念上来说指的是基于物理的渲染，它不只限制在材质上,但是在RTR中我们提到PBR指的就是PBR材质

## 实时渲染中的材质分类

我们将其分为两类:

1. 基于物体表面上定义的材质（绝大多数材质）

基于物体表面又分为两种:

Microfacet models微表面模型（有时候并不是完全基于物理的）

Disney Principled BRDF计算量比较轻量级，因此虽然产生初衷是为了能够用于离线渲染,但也可以运用在实时渲染中，这套材质的种类多效果也很不错,但也不是PBR，是基于artist的角度来考虑的。

2. 基于体积上定义的材质: 光线会进入到云,烟,雾,皮肤,头发等体积里的材质


## Microfacet models微表面模型
<img width="838" height="486" alt="image" src="https://github.com/user-attachments/assets/23491b48-ccbb-4ad4-8e9d-e343c1312575" />

### F项：菲涅尔项，表示观察角度与反射的关系(从一个角度看去会有多少的能量被反射)

有多少能量被反射取决于入射光的角度，当入射方向接近grazing angle掠射角度的时候，光线是被反射的最多的,也就是当你的入射方向与法线几乎垂直时候,反射的radiance是最多的.

平时会用一个简单的近似
<img width="552" height="223" alt="image" src="https://github.com/user-attachments/assets/4bb5a766-727d-4563-946d-1b16b7659d42" />

### D项：微表面的法线分布

决定这一项的是不同微表面朝向的法线分布；

当朝向比较集中的时候会得到Glossy的结果,如果朝向特别集中指向时认为是specular的.

我们有有很多不同的模型来描述法线分布

我们主要讲Beckmann和GGX这两个NDF模型:

#### Beckmann

<img width="899" height="452" alt="image" src="https://github.com/user-attachments/assets/f92e8d78-e0ff-4f80-92b8-fc44130c6b37" />

其目的为了描述法线分布，因此肯定是一个关于法线方向h的函数，而h是半球上的任意一个方向，然后描述这一方向对应的值是多少，这就是NDF。

举个例子，给定向量h，如果我们的微平面中有35%与向量hℎ取向一致，则法线分布函数或者说NDF将会返回0.35。

我们来深入理解一下这个函数:

这个函数可以描述不同粗糙程度的表面,不同粗糙程度的意思是NDF中lobe是集中在一个点上,还是分布的比较开.

我们想一下高斯函数中用 α
 来控制胖瘦,也就是标准差,同样对应到D(h)中

#### GGX模型

Beckmann模型的NDF曲线与GGX模型的NDF曲线相比有一个明显的特点:

Long tail 长尾性质:

会很快衰减，但是衰减到一定程度的时候衰减速度会变慢,可以看到即使到了grazing angle(90度)时仍不为0。

这会带来两个好处:

1. Beckmann的高光会逐渐消失,而GGX的高光会减少而不会消失,这就意味着高光的周围我们看到一种光晕的现象.

2.GGX除了高光部分,其余部分会像Diffuse的感觉.

### G项：Shadiowing-Masking

解决的问题就是微表面之间的自遮挡问题,尤其是在角度接近grazing angle时.

<img width="736" height="230" alt="image" src="https://github.com/user-attachments/assets/854803f7-3249-496b-b69a-2059927922bb" />

如图,由于在微观上有不同的微表面,因此虚线部分本该入射的光被遮挡了,由于光线时可逆的,因此不只是看过去被遮挡,往外看时也被遮挡.

分为两种情况:

左边这中从light出发发生的微表面遮挡现象叫做Shadiowing

右边这种从eye出发发生的微表面遮挡现象被称为Masking；

#### 常用的The Smith Shadowing-Masking项：

在两种不同NDF下预测出的G项有一些细微差别但其实相差不大垂直入射的时候Shadowing-Masking不起作用，在grazing angle时,G项变得非常小接近于0,从而我们解决了分子除以分母导致函数结果过大,整体发白的问题.
<img width="892" height="553" alt="image" src="https://github.com/user-attachments/assets/cb5b8dcf-1f68-4240-acc3-480d3b1d92b4" />

### 但是在我们正确考虑了F项,G项,NDF项仍然有一些问题:

没有考虑光线在表面上的多次弹射，只考虑了微表面遮挡的情况，当粗糙度越来越大的时候，能量是不守恒的,因此导致了粗糙度增大引起了能量损失这一现象.

因此我们需要把丢失的能量补回去.

在离线渲染中,我们去考虑多次bounce,在微表面做一个类似光线追踪的东西,结果准确但速度很慢,更不要提在RTR中了.

在RTR中有自己的方法去补足丢失的能量

其核心思路是将反射光看作两种情况:

1. 当不被遮挡的时候，这些光就会被看到；

2. 当反射光被微表面遮挡的时候，认为这些被挡住的光将进行后续的弹射,直到能被看到

通过basic ieda从而有了工业界中的处理方法:

#### The Kulla-Conty Approximation

这种方法是通过经验去补全多次反射丢失的能量,其实是创建一个模拟多次反射表面反射的附加BRDF波瓣->fms，利用这个BRDF算出消失的能量作为能量补偿项（Energy Compensation Term）,那么我需要考虑两件事:

1. 在反射时有多少能量丢失了？
2. 最后反射出的能量有多少?

首先我们先来算最后反射出了多少的能量:

<img width="706" height="118" alt="image" src="https://github.com/user-attachments/assets/97071c59-d6b0-4f2c-9c67-419a6a56c1a1" />

我们认为任何方向入射的Radiance是1,也就是rendering equation中的Lighting项是1（因为是1所以式子中没有出现）
同样假设BRDF是各向同性的，也就是与i、o无关的；
因此最终积分的结果意义是,在uniform的lighting=1的情况下,在经历了1 bounce之后射出的总能量 E(uo).

那么有多少能量被遮挡就是 1-E_{u_0}

不同方向积分出来的值是不同的，因此不同观察方向损失的能量1-E(μ0)我们可以求出来了,那么只需要加上这一部分能量就解决问题了.

由于BRDF的可逆性，因此除了考虑o方向,还要考虑i方向（也就是要考虑入射丢失的和出射丢失的）,同时要乘一个归一化的量c（可以是常量或函数）得出一个brdf也就是：

<img width="373" height="81" alt="image" src="https://github.com/user-attachments/assets/3a85914e-8956-4f98-a245-76c0e14d59eb" />
通过这个brdf积分后得到的结果要等于消失的能量->1 - E(uo)

之所以这么做是因为简单,我们保留下需要求出来的积分值1 - E(uo),并且考虑由于brdf可逆性的另外半边的1-E(ui),剩下的部分我们写成一个常数c不去管他,这个C是可以求出来的

<img width="852" height="541" alt="image" src="https://github.com/user-attachments/assets/c6b9bbec-a5b3-45db-b2e4-b1f90b73b36f" />

它的原理是希望设计一种可以交换输入输出方向的brdf->fms,使得它的积分结果正是我们所失去的能量,因此用失去的+射出的能量达到能量守恒.

#### 当单次反射的BRDF是有颜色情况怎么办？

有颜色就意味着，物体发生了能量吸收的情况，也就是有额外能量损失，也就是单次反射的积分结果不是1.
因此我们可以先考虑没有颜色的情况，然后再去考虑他的颜色是什么。
Favg->平均菲涅尔项:

不管入射角多大，平均每次反射会有多少能量反射

因此平均菲涅尔项的计算就是计算在所有角度下，菲涅尔项的平均值。

<img width="957" height="190" alt="image" src="https://github.com/user-attachments/assets/ca0633d7-8e21-4809-92e3-9048ece9ed29" />

<img width="1000" height="666" alt="image" src="https://github.com/user-attachments/assets/27576b1f-6dc6-48a1-8eb1-9c75b086c061" />


## Linearly Transformed Cosines(LTC->线性变换的余弦)
TC是为了解决microfacet models的shading问题,但是它有一些限制:

1. 主要是针对GGX模型,对于其他模型原理也同样适用.

2. 做的是不考虑shadow的shading

3. 光源是多边形光源,且发出的radiance时uniform的.

