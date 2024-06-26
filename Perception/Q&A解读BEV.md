# 道尽BEV   
### Q：BEV分类有几种？   
根据输入数据不同，分为3种：BEV camera，BEV LiDAR，BEV Fusion   
### Q：BEV Camera 分为几类，一般流程是什么？   
纯视觉感知可分为单目，双目，以及多相机这3种。以单目BEV方案为例：   
![untitled](https://github.com/zihaosoog/CV-LLM-for-beginner/assets/67272893/63fec883-8743-4e0d-939c-8d703cd9b188)

1. 2D特征提取   
2. 特征转换：**可有可无**，转换对于相机的内外参数需求也**可有可无**   
3. PV-BEV转换[perspective view (PV) to bird’s eye view(BEV)]   
4. 3D解码：输入2D或者3D特征，输出BEV图分割以及3D预测   
 --- 
   
单目BEV的2D-3D转换分为3种流派：2D-3D；3D-2D；Pure-network-based   
1. 2D-3D：在2D特征上预测深度信息，基于预测的深度将2D特征lift到3D特征【LSS】   
2. 3D-2D：利用逆透视映射算法，根据Camera内外参以及转换矩阵的数学公式推导，可以将2D和3D相应坐标位置对应起来（投影），接着通过2D投影区域特征的变换得到3D空间的Voxel特征   
3. Pure-network-based：减少几何投影的偏差，利用network学习隐式表达Camera投影关系   
   
### Q：BEV 一般分为哪几个范式/流派？   
有2种范式：   
> LSS-Based的方法，如：BEVDet, BEVDet4D, BEVFusion, FastBEV。   

1. Lift：提取2D feature，与此同时，提取Depth信息，这里将Depth信息比作一个在视锥射线上离散深度区间的分类概率问题，将2D feature和Depth信息外积得到point cloud   
2. Splat：根据Depth信息以及相机内外参，将2D坐标映射到车身坐标系下的3D坐标，将3D坐标每个点分配给最近的pillar【dx,dy,无限高的长柱子】，每个pillar的特征就是匹配到的点的特征sum-pooling，得到BEV features   
3. BEV Encoder：对于BEV features进一步特征提取和多尺度融合   
4. Task Head：segmentation/3D object detection etc.   
   
> Transformer-Based的方法，如：DETR3D, BEVFormer, PETR, PETRv2, StreamPETR.   

1. 显式BEV Queries：大小 [H,W,C]，栅格化可学习参数，共H\*W个grid，每个grid的特征维度是C，表示语义信息   
2. Spatial cross-attention：将每个栅格特征*Qp*按z轴方向Lift，并按间隔设置N个参考点得到N个3D空间参考点；再根据相机内外参将这些3D空间参考点转换为各个视角相机下的2D图像坐标点(由于相机感知范围限制，每个3D空间参考点只有1-2个相机上能找到有限的转换投影)；利用deformable attention在这些2D参考点的基础上，在2D特征图上对周围局部的特征进行采样；将不同相机采样到的特征做加权求和，作为该栅格的BEV特征；   
   ![untitled-1](https://github.com/zihaosoog/CV-LLM-for-beginner/assets/67272893/7536663a-1fe2-45b3-8700-6a92e866f8a6) 
4. Temporal self-attention：对 grid query *Qp*，基于reference point *p *即当前位置坐标(x,y)，对 *cat(Q, B\_(t-1))* 进行 DCN*，*即在当前时刻的* BEV feature *以及上一时刻的 BEV\_(t-1) 两者的 *p *周围进行 DCN；   
   ![untitled-2](https://github.com/zihaosoog/CV-LLM-for-beginner/assets/67272893/47178130-c2be-45f1-acb8-33cb51949661) 
    Tips：基于reference point的offset和这个采样点的attention weight都由*Qp(z\_q)*经过线性变换得到；    
    ![untitled-3](https://github.com/zihaosoog/CV-LLM-for-beginner/assets/67272893/43dc101c-9e8e-42db-9a01-bdb3b2f0b9e0)

### Q：相机内参与外参分别是什么？作用是什么？涉及到的坐标系是什么？   
1. 相机内参：是指相机的内部参数，这些参数定义了相机的光学特性和传感器尺寸。内参主要包括焦距（f）、主点（principal point，即光心在图像平面上的投影位置）、以及畸变系数（distortion coefficients，描述镜头引起的图像畸变的参数）。   
2. 内参的作用：是将相机坐标系下的点坐标转换到像素坐标系中，也就是说，它描述了从三维空间中的点到二维图像平面的映射关系。   
3. 相机外参：是指相机的外部参数，这些参数描述了相机相对于某个参考坐标系（通常是世界坐标系）的位置和方向。外参包括旋转矩阵（rotation matrix）和平移向量（translation vector），它们定义了从世界坐标系到相机坐标系的变换。   
4. 外参的作用：是将世界坐标系下的点坐标转换到相机坐标系中，也就是说，它描述了从三维空间中的点到相机坐标系的映射关系。   
   
**注：相机坐标系→像素坐标系的转换是3D→2D的转换，两者之间是透视投影的关系。**   
