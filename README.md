# Real-Time Global Illumination
## Reflective Shadow Map(RSM)
games101中我们使用蒙特卡洛方法来sample光线的方法来计算全局光照

现在我们尽量避免复杂的sample计算

那么我们需要解决

### 1：哪些表面会被直接照到？
这里可以使用shadow map，shadow map上每一个像素可以看成是一个小surface patch，并且假设所有的次级光源都是diffuse(这里是假设reflector，没有要求receiver也是diffuse)。

这样的话故outgoing radiance在所有方向上都是uniform的,这样不管从camera看过去还是从点p看过去所得到的结果是一样的。


### 2：如何计算每个surface patch对着色点p的贡献？

games101里学过我们在计算某一点的光照的时候可以通过和光源换底来sample

<img width="391" height="325" alt="image" src="https://github.com/user-attachments/assets/30a408c5-3e58-409d-b393-2e911ba6fa36" />

所以代入可以写成

<img width="855" height="197" alt="image" src="https://github.com/user-attachments/assets/8baab9d0-e7d6-4b9a-b7ed-eadc6f4c02a8" />

而对于从q点到p点的radiance有

<img width="836" height="253" alt="image" src="https://github.com/user-attachments/assets/a0f6a072-50dc-4912-964d-246f6aed0b02" />

对于每个次级光源点来说,由于我们假设它的brdf是diffuse的,因此次级光源的fr积分后是个常数,此时我们把Li代入到式子中会发现dA刚好会被抵消.

<img width="548" height="88" alt="image" src="https://github.com/user-attachments/assets/96ef5ffb-9222-4433-96df-b79a002631d1" />

注意到这里没办法解决Visibility的问题，因此直接把它丢掉

而计算所有的像素也过于慢了，因此为了加速这一过程,我们认为在shadow map中着色点
的位置和间接光源
的距离可以近似为它们在世界空间中的距离。所以我们认为，对着色点
影响大的间接光源在shadow map中一定也是接近的。

## Light Propagation Volumes （LPV）
思路： 在3D空间中去传播光线,从而利用它做出间接光照从而实现GI

如果我们能获得任何一个Shading point上来自四周的radiance的话,就可以立刻得到其间接光照.

因此我们将场景划分为若干个3D网格,每个网格叫做Voxel,在计算完直接光照后,将接受到直接光照的表面看作间接光照在场景中传播的起点.

### 步骤
1.Generation

与RSM一样,首先通过Shadow Map找出接受直接光照的表面或物体

2.Injection

预先把场景划分为若干个3D网格

把虚拟光源注入到其对应的格子内

一个格子内可能包含许多不同朝向的虚拟光源,把格子内所有虚拟光源的不同朝向的radiance算出来并sum up从而得到一个往四面八方的radiance
由于是在空间上的分布,也就可以看作是球面函数,自然可以用SH来表示(工业界用两阶SH就可以表示各个方向上的radiance初始值)

3.Propagation

由于是3D网格,因此可以向六个面进行传播(上下左右前后),由于radiance是沿直线传播的,我们认为radiance是从网格中心往不同方向进行传播的

每个格子计算收到的radiance,并用SH表示
迭代四五次之后,场景中各voxel的radiance趋于稳定

4.Rendering

对于任意的shading point，找到他所在的网格
获得所在网格中所有方向的Radicae；
渲染。

LPV也有自己的问题，格子划分得不够精细的时候它也会漏光
<img width="371" height="293" alt="image" src="https://github.com/user-attachments/assets/07502b89-285a-423b-a7dc-bf8c6e9dd69a" />

## Voxel Global Illumination （VXGI）
和LPV的区别主要是：

1. VXGL建立出一个Hierachical树形结构的体素

2. 在LPV中,我们将受到直接光照的点注入到场景划分的Voxel之后进行传播,只需要传播一次就可以知道场景中任何一个shading point收到间接光照的radiance.

在VXGI中第二趟我们从camera出发,就像有一个Camera Ray打到每一个pixel上,根据pixel上代表的物体材质做出不同的操作,如果是glossy则打出一个锥形区域,diffuse则打出若干个锥形区域,打出的锥形区域与场景中一些已经存在的voxel相交,这些voxel对于Shading point的贡献可以算出来,也就是我们要对每一个shading point都做一个cone tracing


Pass1：Light pass

<img width="816" height="639" alt="image" src="https://github.com/user-attachments/assets/6c8e51bb-454f-4648-b107-07ca1d89e5c8" />

记录直接光源从哪些范围来（绿色部分），记录各个反射表面的法线（橙色部分），通过输入方向和法线范围两个信息然后通过表面的材质，来准确的算出出射的分布，这样就比LPV认为格子表面是diffuse再用SH来压缩的方法要准确

Pass 2: Camera pass

对于Glossy的表面，向反射方向追踪出一个cone区域，基于追踪出的圆锥面的大小，对格子的层级进行查询

对于diffuse的情况来说,通常考虑成若干圆锥，忽略圆锥Tracing时的重叠和空隙

## Screen Space Ambient Occlusion（SSAO）
<img width="852" height="511" alt="image" src="https://github.com/user-attachments/assets/d6cf77d1-89aa-4173-ace4-705a6f9cfa80" />

橙色方框中积分的结果，由于我们假设了所有方向的间接光照是一个常数，架设了物体是Diffuse的,因此BRDF也是常数.因此最终结果就是漫反射系数×间接光照强度Li,由于也间接光照强度Li你可以随便指定,漫反射系数也是你可以随便指定的,因此就是橙色部分的结果是你自己来定义的一个数。

AO就是: shading point的加权平均visibility * 一个你自己定义的颜色

SSAO简单理解：
间接光照是常数—Li为常数

BRDF是Diffuse的—BRDF（fr）是常数ρ/π

因此这两项可以直接从积分里面拿出来，最后Rendering Equation就成了算Visibility的积分。

那么接下来我们只需要去求得visibility部分就可以了,也就是算加权平均的visibility,在世界空间下可以通过ray tarcing做，但是在屏幕空间下面我们该怎么做？

<img width="940" height="351" alt="image" src="https://github.com/user-attachments/assets/70e4d0e9-7c49-4fd9-a851-6b13211e4a02" />

方法是在shading point附近的半径为R的球面里随机sample一些点，然后通过深度判断从camera是否能看到，从而估算被遮挡的比例

## HBAO

和SSAO的主要区别是有了法线方向以后可以在半球上采样

## Screen space Direction Occlusion （SSDO）
SSAO的重要假设是间接光照处处相等都是一个常数，SSDO里去除掉这个假设

SSDO会通过入射光线是否打到物体来计算入射光的大小(类似path tracing的想法)

<img width="868" height="523" alt="image" src="https://github.com/user-attachments/assets/85675a24-2a49-48b2-ba06-c5b20164ae86" />

SSAO和SSDO的想法是相反的

SSAO的想法是打出光线如何碰到物体说明这根光线被挡到，因此接收不到光

SSDO的想法是打出光纤如果碰到物体说明能接收到该物体作为次级光源发出的光，碰不到物体才没有光

因为对于AO我们假设间接光照是从比较远的地方来的，在DO中,我们认为红色框里接收的是直接光照,而黄色框里才是接收到的间接光照.因为红色框里的光线打不到用来反射的面，因此这些方向上就不会有间接光照，黄色框里的光线能打到物体上，P点接收到的是来自红色框的直接光照+黄色框里的间接光照,也就是假设间接光照是从比较近的反射物来的。

<img width="891" height="527" alt="image" src="https://github.com/user-attachments/assets/bc1bcb18-c372-4f9d-b841-7fdb5d4db121" />

SSDO的逻辑是这里A/B/D这三个点的深度比从camera看去的最小深度深,也就是说PA,PB,PD方向会被物体挡住,因此会为P点提供间接光照。这样的逻辑当然会产生一些问题。

SSDO的缺点除了依赖camera view计算光照以外，还有就是它只能解决一个很小范围内的全局光照

## Screen space Reflection（SSR）
基础的SSR算法：镜面反射

对于任何一个像素:

知道shading point的观察方向后,可以得出其反射方向

从Shading point点沿着反射方向延长找到与屏幕的壳的交点

将交点的颜色作为反射的颜色记录到shading point。

### 怎么求反射光与场景的相交?

Linear Raymarch


<img width="934" height="418" alt="image" src="https://github.com/user-attachments/assets/c27f21d1-e85d-49ab-ac1a-238c92588960" />

我们是为了找到反射光与场景“壳”的交点:

沿着反射方向以一个固定的步长逐步前进,并将每次停止时的深度与壳的深度进行比较,如果浅于壳,则继续前进,比壳深,则停止求交,也就是我们用深度来进行可见性判断
质量取决于步长的大小，步长小越精准，同时计算量也越大，因此步长太大太小都不行，在没有SDF的情况下，步长只能是一个定值。

由于步长是由我们来决定的,太长太短都有其各自的问题,因此我们引入另一种动态决定步长的方法:
Hierachical ray trace 其思路有些像计网里的TCP拥塞控制

为了这个我们需要做一个准备工作,把场景的深度图,做一个mip-map,但这个高一级的mipmap记录的是四个像素中深度的最小值

那么如果一根光线与mip-map中的上层结点不相交,那他肯定也不会与这个结点的子节点相交.

至此我们完成了屏幕空间光线追踪的部分,但是我们还没完成如何计算shading.

这部分与路径追踪的方法完全相同，仅仅是把光线与场景求交变成了光线与“壳”求交，因此路径追踪的算法在这里是可以直接使用的。

对于任何一个shading point，看到的radicance就是对半球进行积分,如果是specular的物体,那么相当于光线打到物体的哪里,就用它所发出的radiance就可以.

如果是glossy情况下,同样的用蒙特卡洛多采样几根光线,不管怎么所打到的物体反射过来的radiance,一定就是shading point点接收到的incident radiance.

这里我们同样需要假设反射物/次级光源 是Diffuse的情况,地板之类的接收物可以是任何物体.

