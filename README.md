# CourseNotes
## Shadow Mapping
### 原理
在光源处放置一个摄像头渲染一个texture记录场景中物体的深度

在主渲染流程中将shading point转换到光源空间里和深度图比较就能判断该点是否被遮挡
### 存在的问题
自遮挡

从光源射到物体上时会默认该像素点里的点的深度都为采样点深度，因此可能会遮挡住一整个像素的所在的平面“背后”的所有物体

<img width="460" height="302" alt="image" src="https://github.com/user-attachments/assets/ddf6b677-1fdf-498a-8628-b5c5740014e2" />

锯齿
## 一些数学工具
实时渲染里有近似公式：
$\int_{\Omega} f(x) g(x)\ dx \approx \frac{\int_{\Omega} f(x)\, dx}{\int_{\Omega} dx} \cdot \int_{\Omega} g(x)\ dx$
这个公式里的约等成立需要以下两个条件:

1.g(x)积分的support较小。这里的support我们可以暂时理解为积分域。

2.f(x)/g(x)在积分域上足够光滑（变化不大）。

那么这个时候我们的渲染方程
$L_o(\mathbf{p}  \omega_o) = \int_{\Omega^+} L_i(\mathbf{p}  \omega_i)\  f_r(\mathbf{p}  \omega_i  \omega_o)\  \cos \theta_i\, V(\mathbf{p}, \omega_i)\  d\omega_i$
可以被近似为
$L_o(\mathbf{p}  \omega_o) \approx \frac{\int_{\Omega^+} V(\mathbf{p}  \omega_i)\, d\omega_i}{\int_{\Omega^+} d\omega_i} \cdot \int_{\Omega^+} L_i(\mathbf{p}, \omega_i) f_r(\mathbf{p}  \omega_i  \omega_o) \cos \theta_i\ d\omega_i$

它表示的意义就是,我们计算每个点的shading，然后去乘这个点的visibality得到的就是最后的渲染结果。

那么什么时候这个约等式比较正确呢？

a) 我们要控制积分域足够小，也就是说我们只有一个点光源或者方向光源。

b) 我们要保证shading部分足够光滑，也就是说brdf的部分变化足够小，那么这个brdf部分是diffuse的。

c) 我们还要保证光源各处的radience变化也不大，类似于一个面光源。


## PCF
### 原理
Shadow Mapping只能产生非0即1的硬阴影

PCF通过filter shading point附近的一片区域来计算shading point在阴影里的权重，作为乘以阴影颜色的系数
## PCSS
### 原理
PCF的filter的范围是手动选择且固定的

但实际的情况是物体离平面越近阴影越硬，物体离平面越远阴影越软

<img width="686" height="906" alt="image" src="https://github.com/user-attachments/assets/0076a416-ea08-4012-b87e-ee040314fe0e" />

PCSS的三个步骤
第一步：取Shadow Map上一定范围内的可遮挡深度值的平均值average blocker depth作为最终的 blocker depth。（注：若某深度不会产生遮挡，该深度值不参与计算）

第二步：Penumbra estimation 半影区域计算。根据第一步得到的 blocker depth，计算出半影区域的大小，进而得到 filter size 的大小

第三步：执行PCF。根据第二步得到的 filter size 执行PCF

问题是步骤1里的取一定范围的遮挡深度，这里的范围要怎么选呢

一种方法是hw3里的给定一个常数

第二种，是通过计算得到一个范围大小:

我们计算shadow map的时候在光源处设置过相机，如图所示，我们把shadow map放在由相机看向场景形成的视锥中的近截面上,然后将光源shading point相连，在shadow map上截出来的面就是要查询计算平均遮挡距离的部分.这部分的深度求一个均值，就是Blocker到光源的平均遮挡距离。

离光源越远，遮挡物也会更多,所以需要在Shadow map上的一个小区域内查找blocker.

离光源越近，遮挡物会少,所以需要在Shadow map上的一个大区域内查找blocker.
<img width="687" height="503" alt="image" src="https://github.com/user-attachments/assets/d2f919aa-b9a7-4ca3-a268-c1c617e67587" />
## Variance soft shadow mapping
在做PCF的时候我们要判断邻域的depth和当前depth的大小来得到平均的visualibility，这个过程比较慢，我们可以假设所有的分布是一个正态分布

因此想要知道邻域有多少depth大于自己<=>求出范围内有百分之多少的像素比它浅

这样的话我们只要有均值和方差就可以了

这样的话就只需要用SAT存储depth和depth^2的图就可以快速计算$\mathrm{Var}(X) = \mathbb{E}[X^2] - \mathbb{E}[X]^2$

得到期望和方差之后,根据切比雪夫不等式近似得到一个depth大于shading point点深度的面积（把不等号近似为约等，甚至不用假设高斯分布）,也就是求出了未遮挡Shading point的概率,从而可以求出一个在1-0之间的visilibity.

这样就结算了PCSS第三步里的采样问题

但PCSS的第一步仍需采样一个范围来求平均值depth，VSSM继续来解决这个问题

已知非遮挡像素占的比例 * 非遮挡物的平均深度 + 遮挡像素占的比例 * 遮挡物的平均深度 = 总区域内的平均深度

$\frac{N_1}{N} Z_{\text{unocc}} + \frac{N_2}{N} Z_{\text{occ}} = Z_{\text{avg}}$

总区域内的平均深度我们用 mipmap 或者 SAT 去求，然后用 shadow map 和 square-depth map 方差，最后根据切比雪夫不等式近似求出 $\frac{N_1}{N}$ 和 $\frac{N_2}{N}$。

$$
\frac{N_1}{N} = P(x > t)
$$

$$
\frac{N_2}{N} = 1 - P(x > t)
$$

此时公式中的 $Z_{\text{unocc}}$ 和 $Z_{\text{occ}}$ 仍然是未知的，我们做一个大胆的假设：认为非遮挡物的平均深度等于 shading point 的深度。至此我们只剩下 $Z_{\text{occ}}$ 的深度，将所有值代入即可求出遮挡物的平均深度 $Z_{\text{occ}}$。但是当接收平面是曲面或者与光源不平行时，这个假设会出现问题。

# Homework
## hw1
<img width="891" height="1032" alt="image" src="https://github.com/user-attachments/assets/b1c739b4-5df6-4618-9277-822248efa11f" />
<img width="1378" height="996" alt="image" src="https://github.com/user-attachments/assets/dd3179ab-e42b-4817-aa5b-87f937ddb7d8" />
trick：

在作业里有尝试根据根据shading point的法向与光源方向的夹角调整bias的值。当目标点法向与光源方向夹角很小（接近平行）时，令bias尽可能小。当目标点法向与光源方向夹角很大（接近垂直）时，令bias尽可能大。
## hw2

<img width="1227" height="1070" alt="image" src="https://github.com/user-attachments/assets/7cb0fc7e-dcfc-4858-8b3d-a9ab3047b97c" />

## hw3
<img width="846" height="1155" alt="image" src="https://github.com/user-attachments/assets/4fb4cc49-d133-4eb6-98a1-4f05b2e6ba00" />

