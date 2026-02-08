# Games202HomeWork&Notes
## Shadow Mapping
### 原理
在光源处放置一个摄像头渲染一个texture记录场景中物体的深度

在主渲染流程中将shading point转换到光源空间里和深度图比较就能判断该点是否被遮挡
### 存在的问题
自遮挡

从光源射到物体上时会默认该像素点里的点的深度都为采样点深度，因此可能会遮挡住一整个像素的所在的平面“背后”的所有物体

<img width="460" height="302" alt="image" src="https://github.com/user-attachments/assets/ddf6b677-1fdf-498a-8628-b5c5740014e2" />

trick：

在作业里有尝试根据根据shading point的法向与光源方向的夹角调整bias的值。当目标点法向与光源方向夹角很小（接近平行）时，令bias尽可能小。当目标点法向与光源方向夹角很大（接近垂直）时，令bias尽可能大。
### 效果
<img width="891" height="1032" alt="image" src="https://github.com/user-attachments/assets/b1c739b4-5df6-4618-9277-822248efa11f" />
<img width="1378" height="996" alt="image" src="https://github.com/user-attachments/assets/dd3179ab-e42b-4817-aa5b-87f937ddb7d8" />

## PCF
### 原理
Shadow Mapping只能产生非0即1的硬阴影

PCF通过filter shading point附近的一片区域来计算shading point在阴影里的权重，作为乘以阴影颜色的系数
### 效果
<img width="1227" height="1070" alt="image" src="https://github.com/user-attachments/assets/7cb0fc7e-dcfc-4858-8b3d-a9ab3047b97c" />
## PCSS
### 原理
PCF的filter的范围是手动选择且固定的

但实际的情况是物体离平面越近阴影越硬，物体离平面越远阴影越软

<img width="686" height="906" alt="image" src="https://github.com/user-attachments/assets/0076a416-ea08-4012-b87e-ee040314fe0e" />

PCSS的三个步骤
第一步：取Shadow Map上一定范围内的可遮挡深度值的平均值average blocker depth作为最终的 blocker depth。（注：若某深度不会产生遮挡，该深度值不参与计算）

第二步：Penumbra estimation 半影区域计算。根据第一步得到的 blocker depth，计算出半影区域的大小，进而得到 filter size 的大小

第三步：执行PCF。根据第二步得到的 filter size 执行PCF
### 效果
<img width="846" height="1155" alt="image" src="https://github.com/user-attachments/assets/4fb4cc49-d133-4eb6-98a1-4f05b2e6ba00" />
