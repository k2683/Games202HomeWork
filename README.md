# Real-Time Global Illumination

games101中我们使用蒙特卡洛方法来sample光线的方法来计算全局光照

现在我们尽量避免复杂的sample计算

那么我们需要解决

## 1：哪些表面会被直接照到？
这里可以使用shadow map，shadow map上每一个像素可以看成是一个小surface patch，并且假设所有的次级光源都是diffuse(这里是假设reflector，没有要求receiver也是diffuse)。

这样的话故outgoing radiance在所有方向上都是uniform的,这样不管从camera看过去还是从点p看过去所得到的结果是一样的。


## 2：如何计算每个surface patch对着色点p的贡献？


