本文梳理下`OpenMVS`的各个模块的算法步骤，不涉及具体细节。如有错误，欢迎评论指出。

---
# 介绍
> 算法的输入为一组相机姿态及稀疏的点云（由`SFM`算法获得，比如`Colmap`），输出为带纹理的三角网格。

# 一.稠密点云重建
>此模块用于获取所拍摄场景尽可能准确和完整的三维点云[1-2]。

具体步骤有：
## 立体匹配图像对选择
选择用于做立体匹配的图像，即为每个参考图像选择有效的邻域图像。分两步：1.根据共视点在图像上的夹角、分辨率、面积三个准则进行首轮选择 2.进一步地构建马尔科夫随机场（`MRF`）问题[8]，即经典的标注（`labeling`）问题，选择一个全局最优的领域图像。
## 深度图计算
对构建的图像对计算视差图，也即深度图。首先先利用稀疏的三维点+delaunay三角剖分做深度图的初始化，然后进一步采用`SGM`[6]或者`PatchMatch`[7]算法计算得到稠密的深度图，最后通过深度段（`depth segment`）过滤+孔洞插值+领域图与参考图深度一致性检查对深度图做初步优化。
## 深度图融合
将多张深度图即多个点云融合成单个场景点云。这里主要是想要去除冗余点和遮挡点。
# 二.网格重建
> 此模块用于从输入点云重建得到三维网格表面[3]。可参考笔者之前的博客[9]。

具体步骤有：
## 构键点云的四面体剖分
对输入点云进行四面体剖分，将表面提取问题转化为四面体的内外标注问题，内外分界的三角形面即为最终待重建的曲面。
## 构建图结构
同样地，标注问题可采用`MRF`来进行建模，把四面体当做图中的节点，四面体如果共享一个共同的三角形面即说明相邻可构建图节点的边。
## 利用可视信息对图中的边进行加权
利用相机-观测点所连成的光线，光线可触及的四面体为外，不可触及的四面体为内空间，据此对图中的边进行加权，进而可构建出能量方程来。
## 对弱支持面对应边的权重进行调整
对局部不存在或存在较少的点的曲面（`weakly Supported Surfaces`）的边权重进行调整，从而提高这种类型曲面的重建效果。
## 求解最小割
问题自此转化为求解`MRF`的最小割问题，最终图结构的不连通边得到了一组三角形面，即为最终的重建曲面。

# 三.网格优化
> 用于恢复网格的细节[4]。

具体步骤有：
预处理
用于优化提取的网格质量，涉及到网格精简（`decimate`、`simplify`），网格细分（`subdivide`）等处理算法。
顶点调整
基于光度一致性、拉普拉斯平滑、轮廓一致性等构建优化问题，采用梯度下降法求解，沿着法线方向迭代调整网格顶点位置。
# 四.网格贴图
>用于为网格着色[5]，这里采用的是`UV`贴图，而不是顶点贴图。

具体步骤有：
## 视图选择  
为每个网格面片（`face`）挑选最佳视图，构建`MRF`问题进行求解。
## 纹理块构建
将连在一起的且对应于同一个视图的`face`构建成一个块（`patch`）。
## 全局颜色校正
由于同一物体的同一位置在不同视角下的成像受到光线等因素的影响会呈现出不同的颜色，从而当左右两个`patch`所映射到的视角图像若不同会使得形成接缝，反应在最后的网格上就是颜色不连续。因此构建了一个优化问题，为`patch`接缝处的顶点计算一个颜色调整量。
## 局部颜色校正
采用泊松融合，对每个`patch`的颜色进一步进行单独优化，进一步减少颜色不均匀的问题。
## 纹理图生成
采用装箱算法，将纹理块打包在一张图上（`texture altas`），并让网格顶点对应到该图上。

# 五.参考资料
[1] PatchMatch: A Randomized Correspondence Algorithm for Structural Image EditingC. Barnes et al. 2009 \
[2] Memory Efficient Semi-Global MatchingH. Hirschmüller et al. 2012 \
[3] Exploiting Visibility Information in Surface Reconstruction to Preserve Weakly Supported SurfacesM. Jancosek et al. 2014 \
[4] High Accuracy and Visibility-Consistent Dense Multiview StereoHH. Vu et al. 2012 \
[5] Let There Be Color! - Large-Scale Texturing of 3D ReconstructionsM. Waechter et al. 2014 \
[6] semi-global matching 算法总结 \
[7] 【算法】PatchMatch立体匹配算法_原理解析_patchmatch算法-CSDN博客 \
[8] 6.1条件随机场在NLP中的应用_哔哩哔哩_bilibili \
[9] Hypochondria：点云表面提取算法之图割

---
### 欢迎关注我的微信公众号：Hypochondria 
![公众号二维码](https://github.com/user-attachments/assets/2f8e41fb-e990-4fef-825a-d1b86fba7c3f)

### 以及我的知乎主页：https://www.zhihu.com/people/yang-shi-lei-62-72
---
