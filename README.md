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
