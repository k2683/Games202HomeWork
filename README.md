# Real-Time Global Illumination

games101中我们使用蒙特卡洛方法来sample光线的方法来计算全局光照

现在我们尽量避免复杂的sample计算

那么我们需要解决

## 1：哪些表面会被直接照到？
这里可以使用shadow map，shadow map上每一个像素可以看成是一个小surface patch，并且假设所有的次级光源都是diffuse(这里是假设reflector，没有要求receiver也是diffuse)。

这样的话故outgoing radiance在所有方向上都是uniform的,这样不管从camera看过去还是从点p看过去所得到的结果是一样的。


## 2：如何计算每个surface patch对着色点p的贡献？

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
