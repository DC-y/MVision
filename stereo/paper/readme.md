# 双目立体匹配等算法论文
[kitti双目算法评测](http://www.cvlibs.net/datasets/kitti/eval_scene_flow.php?benchmark=stereo)

## 1. CSCA  2014 多尺度代价聚合
[论文： Cross-Scale Cost Aggregation for Stereo Matching](https://arxiv.org/pdf/1403.0316.pdf)

[代码](https://github.com/Ewenwan/CrossScaleStereo)

[参考博客](https://blog.csdn.net/wsj998689aa/article/details/44411215)

### 立体匹配最基本的步骤
    1）代价计算。
       计算左图一个像素和右图一个像素之间的代价。

    2）代价聚合。
       一般基于点之间的匹配很容易受噪声的影响，往往真实匹配的像素的代价并不是最低。
       所以有必要在点的周围建立一个window，让像素块和像素块之间进行比较，这样肯定靠谱些。
       代价聚合往往是局部算法或者半全局算法才会使用，
       全局算法抛弃了window，采用基于全图信息的方式建立能量函数。

    3）深度赋值。
       这一步可以区分局部算法与全局算法，局部算法直接优化 代价聚合模型。
       而全局算法，要建立一个 能量函数，能量函数的数据项往往就是代价聚合公式，例如 DoubleBP。
       输出的是一个粗略的视差图。

    4）结果优化。对上一步得到的粗估计的视差图进行精确计算，策略有很多，
    例如plane fitting，BP，动态规划等。

    可以看作为一种全局算法框架，通过融合现有的局部算法，大幅的提高了算法效果。

### 论文贡献
    第一，设计了一种一般化的代价聚合模型，可将现有算法作为其特例。
    
    第二，考虑到了多尺度交互（multi-scale interaction），
         形式化为正则化项，应用于代价聚合（costaggregation）。
    
    第三，提出一种框架，可以融合现有多种立体匹配算法。

    CSCA利用了多尺度信息，多尺度从何而来？
    其实说到底，就是简单的对图像进行高斯下采样，得到的多幅成对图像（一般是5对），就代表了多尺度信息。
    为什么作者会这么提，作者也是从生物学的角度来启发，他说人类就是这么一个由粗到精的观察习惯（coarse-to-line）。
    生物学好奇妙！

    该文献生成的稠密的视差图，基本方法也是逐像素的（pixelwise），
    分别对每个像素计算视差值，并没有采用惯用的图像分割预处理手段，
    如此看来运算量还是比较可观的。
    
### 算法流程
![](https://github.com/Ewenwan/MVision/blob/master/stereo/paper/img/csca.png)

    1. 对左右两幅图像进行高斯下采样，得到多尺度图像。

    2. 计算匹配代价，这个是基于当前像素点对的，通常代价计算这一步并不重要，
       主要方法有CEN（Census,周围框,对窗口中每一个像素与中心像素进行比较，大于中心像素即为0，否则为1。从而得到一个二进制系列），
       CG（Census + TAD(三通道像素差值均值) + 梯度差值(灰度像素梯度差值)），
       GRD(TAD + 梯度差值)等几种，多种代价之间加权叠加

       得到的结果是一个三维矩阵
       左图 长*宽*视差搜索范围
       值为每一个匹配点对的匹配代价

    3. 代价聚合,一个领域内的代价合并（聚合相当于代价滤波）
       BF（bilateral filter，双边滤波器 ），
       GF（guided image filter，引导滤波器），
       NL（Non-Local，非局部，最小生成树），
       ST（Segment-Tree，分割树，图论分割树） 等，

       基于滤波器的方法是设定一个窗口，在窗口内进行代价聚合。
       双边滤波器就是对窗口内像素进行距离加权和亮度加权。 

       后两种都抛弃了固定窗口模式，
       基于NL的代价聚合是使用最小生成树代替窗口。
       基于ST的代价聚合是使用分割块代替窗口。

       分割树ST代价聚合方法：
            a. 初始化一个图G（V，E）
               每个像素就是顶点V，边(E)是两个像素点对之间三通道像素差值绝对值中最大的那个，作为边的权值。
```c

float CColorWeight::GetWeight(int x0, int y0, int x1, int y1) const {// 两个点对
//得到三通道两个像素值差值最大的那个通道的差值的绝对值,作为边测权重
return (float)std::max(
    std::max( abs(imgPtr(y0,x0)[0] - imgPtr(y1,x1)[0]), abs(imgPtr(y0,x0)[1] - imgPtr(y1,x1)[1]) ), 
    abs(imgPtr(y0,x0)[2] - imgPtr(y1,x1)[2])
    );


```
            b. 利用边的权值对原图像进行分割，构建分割树。
               先对边按权值大小进行升序排列
               判断边权值是否大于阈值，大于则不在同一个分割块，否则就在同一个分割块
            c. 对整颗树的所有节点计算其父节点和子节点，并进行排序。
            d. 从叶节点到根节点，然后再从根节点到叶节点进行代价聚合。

    4. 多尺度代价聚合
       五层金字塔，利用上述方法对每一层计算代价-代价聚合
       添加正则化项的方式，考了到了多尺度之间的关系

       保持不同尺度下的，同一像素的代价一致，如果正则化因子越大，
       说明同一像素不同尺度之间的一致性约束越强，这会加强对低纹理区域的视差估计
       （低纹理区域视差难以估计，需要加强约束才行），
       但是副作用是使得其他区域的视差值估计不够精确。
       不同尺度之间最小代价用减法来体现，
       L2范式正则化相比较于L1正则化而言，对变量的平滑性约束更加明显。

    5. 求最优视差值
       赢者通吃：winner-takes-all
       将每一个视差值代入多尺度代价聚合公式，选择最小的那个代价对应的视差值为当前像素视差值。

    6. 视差求精
        1：加权中值滤波后，进行左右一致性检测; 
           进行不可靠点检测，找左右最近的有效点，选视差值较小的那个; 
           然后对不可靠点进行加权中值滤波; 
        2：分割后，进行左右一致性检测; 
           进行不可靠点检测，找左右最近的有效点，选视差值较小的那个; 
           然后进行对不可靠点进行加权中值滤波; 
           然后求分割块的平面参数，对分割块内的每个像素进行视差求精。
### 速度
    在“CEN+ST+WM”组合下，
    在640p的图像上，运行时间需要6.9s，
    在320p的图像上，运行时间为2.1s，
    在160p上，需要0.43s。
           
###  总结
    1）CSCA是一个优秀的立体匹配算法，它的性价比，综合来说是比较高的，并且CSCA只是一个框架，
       言外之意，我们可以根据框架的思想自己创建新的算法，说不定能够获取更好的性能。
    2）我认为CSCA是一个多尺度的局部算法，还不应该归类为全局算法的类别，这种多尺度思想，
       我想在今后的工作中会有越来越多的研究人员继续深入研究。
       
## 2. NLCA 算法 全局最小生成树代价聚合 (变形树局部框)
[论文：A Non-Local Cost Aggregation Method for Stereo Matching](http://fcv2011.ulsan.ac.kr/files/announcement/592/A%20Non-Local%20Aggregation%20Method%20Stereo%20Matching.pdf)

[代码](https://github.com/Ewenwan/NLCA-fo-stereo-matching/tree/master)

[参考](https://blog.csdn.net/wsj998689aa/article/details/45041547)

### 关键点
    明确抛弃了support window（支持窗口）的算法思想，
    指出support window在视察估计上普遍具有陷入局部最优的缺陷，
    创新性的提出了基于全局最小生成树进行代价聚合的想法，我觉得这个想法非常厉害，全局算法早就有了，
    但是往往是基于复杂的优化算法，重心放在了视察粗估计和迭代求精两步，十分耗时，而最小生成树，
    可以天然地建立像素之间的关系，像素之间的关系一目了然，
    可以大幅减少代价聚合的计算时间，文献表述为线性搜索时间。
    计算聚合代价的过程不需要迭代，算法时间复杂度小，符合实际应用的需求，
    所以NL算法已经获得了不错的引用率。
    在后续算法中得到了很多改进，一些好的立体匹配算法，如CSCA，ST等都是基于NL进行了改进，
    以下部分着重说说我对NL核心部分，最小生成树(minimum spanning tree，MST)的理解。
    
    避免了局部最优解，找到了全局最优解。
    其实MST就是一种贪心算法，每一步选择都是选取当前候选中最优的一个选择。
    
    从一个顶点开始，选取一个最小边对应的顶点
    从已经选取的顶点对应的连接顶点中选取最小的边对应的顶点。。。

[最小生成树](https://blog.csdn.net/qq_35644234/article/details/59106779)

> prim（普里姆）算法：

[代码](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/MST/MST_Prim_Matrix.cpp)

原地图：

![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/1.png)

修建步骤（从一点向外扩展，每次都选最小权重对应的边）：

![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/2.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/3.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/4.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/5.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/6.png)

> kruskal（克鲁斯卡尔）算法:

[代码](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/MST/MST_kruskal.cpp)

修建步骤（每次选出最小的边，不产生环，直到所有的顶点都在树上了）：

![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/7.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/8.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/9.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/10.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/11.png)
![](https://github.com/Ewenwan/ShiYanLou/blob/master/Algorithm/profect/img/12.png)

    因为支持窗口的办法，本质上只考虑了窗口内像素对中心像素的影响，
    窗口之外的像素的影响彻底忽略，其实想想看，这样做也没有什么不妥，
    但是它并不适用一些场合，比如文献列举的图像，带纯色边框的一些图像。
    
    左上角的图像就是原始灰度图像，这个时候我们就会发现，
    这幅图像中像素与像素之间的关系用支持窗口来处理明显不灵，
    比如说周围框状区域的任何一个像素，肯定与框状区域内部的像素的深度信息一致，
    而与中间区域的像素不同。或者说，如果单考虑颜色信息，红框内的像素关系最大，
    如何表征这样的关系就是一个问题。很遗憾，我们不能事先提取出这样的区域
    ，因为图像分割真的很耗时，并且不稳定，这就是作者的牛逼之处，
    他想到了MST可以表示这种像素关系，
    于是采用像素之间颜色信息作为“边权值”，进一步构建MST。
    
    MST指的是最小生成数，全称是最小权重生成树。
    它以全图的像素作为节点，构建过程中不断删除权值较大的边。
    注意，是全图所有的像素，然后采用kruskal（克鲁斯卡尔）算法或prim（普里姆）算法进行计算。
    这样便得到了全图像素之间的关系。
    然后基于这层关系，构建代价聚合，这便是文章标题Non-Local Cost Aggregation的由来。
    
    
## 3. ST（Segment-Tree，分割树，图论分割树）
[代码](https://github.com/Ewenwan/STCostAggregation)

[论文](https://www.cv-foundation.org/openaccess/content_cvpr_2013/papers/Mei_Segment-Tree_Based_Cost_2013_CVPR_paper.pdf)

    ST（segment tree）确实是基于NLCA的改进版本，
    本文的算法思想是：
        基于图像分割，
        采用和NLCA同样的方法对每个分割求取子树，
        然后根据贪心算法，将各个分割对应的子树进行合并，
        算法核心是其复杂的合并流程。
    但是这个分割不是简单的图像分割，其还是利用了最小生成树（MST）的思想，
    对图像进行处理，在分割完毕的同时，每个分割的MST树结构也就出来了。
    然后将每个子树视为一个个节点，在节点的基础上继续做一个MST，
    因此作者号称ST是 分层MST，还是比较贴切的！

    算法是基于NLCA的，那么ST和NLCA比较起来好在哪里？
    作者给出的解释是，NLCA只在一张图上面做一个MST，
    并且edge的权重只是简单的灰度差的衍生值，这点不够科学，
    比如说，当遇到纹理丰富的区域时，这种区域会导致MST的构造出现错误
    ，其实想想看的确是这样，如果MST构造的不好，自然会导致视差值估计不准确。
    而ST考虑了一个分层的MST，有点“由粗到精”的意思在里面。
    
    ST是NLCA的扩展，是一种非传统的全局算法，
    与NLCA唯一的区别就在于ST在创建MST的时候，引入了一个判断条件，使其可以考虑到图像的区域信息。
    这点很新颖，说明作者阅读了大量的文章，并且组合能力惊人，组合了MST和基于图表示的图像分割方法。
    它的运行时间虽然比NLCA要大一些，但是相比较全局算法，速度已经很可以了。
    但是ST也有一些缺陷，首先，算法对图像区域信息的考虑并不是很严谨，
    对整幅图像用一个相同的判断条件进行分割，分割效果不会好到哪里去，
    并且基于实际数据实测，会发现视差图总是出现“白洞”，说明该处的视差值取最大，
    这也是区域信息引入的不好导致的。虽然在细节上略胜于NLCA，但是算法耗时也有所增加。
    上述原因也是本文引用率不佳的原因。
      

    



