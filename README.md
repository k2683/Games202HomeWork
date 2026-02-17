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


