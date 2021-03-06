# 基于激光雷达的无特征目标跟随
## Object Tracking based on Laser Scaner

---

## 项目介绍

基于激光雷达的无特征目标跟随

### 背景

目前在 **自主移动机器人** 领域， **激光雷达**（激光扫描仪）是广泛应用在轮式机器人最基本的传感器之一，激光扫描仪凭借其稳定可靠的数据成为机器人导航避障领域的研究热门，广泛地被应用在机器人建图导航、激光SLAM、定点定位、点云重构等领域，各类应用层出不穷。

![Cleaner Robot](https://d3uepj124s5rcx.cloudfront.net/items/0L3H2k3v1C2C0T0a2W3v/Image%202017-02-12%20at%205.37.31%20PM.png)

扫地机作为一个成熟的机器人产品，目前已经在中高价位产品线中广泛地配备了激光雷达用作最基本的传感器，导航避障领域技术的应用很大程度完善了产品的表现，丰富了产品的功能。但是，激光扫描仪作为一个数据量较大的传感器，其提供的信息量远比导航避障需要的丰富。在满足机器人导航避障与自主定位的前提下，富余的信息为进一步应用提供了可能。

### 概况

基于激光雷达的建图导航、定点定位在自主移动机器人领域基本上算是“标配”，但是相对于环境更为复杂一些的服务机器人来讲，激光数据还有更多的用途。

![Robot of Shanghai University](https://cl.ly/1O2z1f0c1S1s/Image%202017-02-12%20at%206.18.06%20PM.png)

在 **RoboCup@Home** 比赛中，比赛场地为一个模拟的家庭环境，其中一个环节要求机器人跟随使用者，中途有其他行人干扰，同时还需避开场景中随机障碍物。

对于特定人的跟随，识别定位通常有三种方案：

> - 利用摄像头识别、匹配图像流
> - 利用Kinect等传感器SDK提供的接口
> - 利用激光雷达识别

![USB Cam Kibect](https://d3uepj124s5rcx.cloudfront.net/items/3p2i3i263Q361r1r1G2s/1_%E5%89%AF%E6%9C%AC.jpg)

**这三种** 方法在完成对特定人的定位后通常采用人与机器人的相对位置做PID控制，PID算法的输出很大程度上会受噪声、精度、计算速度的影响，甚至差分轮或全向轮的底盘差异也会很大程度影响控制系统的设计，PID的位置控制系统流程大致分为：

1. 传感器的数据采集与处理，找出人的位置
2. 根据人与机器人的相对位置，计算底盘速度的PID参数
3. 底盘执行运动指令

其难点 **一方面** 在于PID参数的整定，考虑到机器人本身的质量包括电机等执行器的差异，利用PID达到理想效果的难度较大；**另一方面** 在于PID太过依赖于输入信号的质量，对不确定性容忍度较低。以上识别定位的三种方案均有一定程度上较为致命的缺点。

**方法一**，摄像头特征识别：

> 1. 由于OpenCV中大多数特征点匹配的算法对图像质量要求较高，大多数低成本非全局快门相机在机器人运动过程中会产生模糊或者撕裂的图像，造成很大的噪声或者丢帧。
2. 特征匹配或着机器学习的方法因其巨大的计算量很难做到实时处理，严重的甚至影响离散系统采样率，干扰后面的PID系统。
3. 太过依赖人的衣服图像特征，对穿纯色或者纯纹理衣服的人无效。
4. 识别到的人缺乏距离信息，仅靠方向很难做控制，而且跟随过程缓慢，人必须离的很近，影响效果。

**方法二**，Kinect自带SDK识别人体：

> 1. Kinect自带的SDK可以很方便地识别人体，但是前提条件是使用Windows，这对很多开源用户和ROS的使用者来说很矛盾。
2. Kinect识别人体的算法不开源并且有一定地计算量，速度方面会稍慢一些，对于运动的机器人会比较迟钝，容易产生较大误差。
3. 而且Kinect视野有限，易受太阳光干扰，性价比、实用性一般。

**方法三**，激光雷达：

> 1. 激光的数据优点在于精度高、稳定性好、不容易受干扰，但是数据量相对来说小了很多，造成了一定程度的信息匮乏，分不清人与类人物体。
2. ROS官网中有一个People-Perception的功能包，其中包含了激光雷达扫描双腿对人定位的节点，在获得人的位置后，位置信息直接送入PID节点进行运动控制，实机效果很一般，比赛时很容易受到干扰。

三种对人进行定位的方法都有一些不足，形象地说，机器人用相对位置PID跟随人的过程可以被比喻为一个“不好用的人体遥控器”，我们主要从改进控制方式入手，优化激光找人的算法。

### 设想

三种对人进行定位的方法中，激光雷达的方法较为稳定、适用范围广、计算量小，较为适合自主研发的服务机器人。我们设想利用激光雷达作为数据来源，对在激光数据中找人的算法进行改进，并且将原来直接利用相对位置PID做控制的方法换为路径点导航，并且引入图优化及卡尔曼滤波等方法提升系统对不确定性的容忍度，进而优化整体控制系统的运行效果。

设想的系统主要分为 **三大部分**：

> 1. **数据处理：** 激光雷达的数据采集与处理，分析获取人的位置等信息。
> 2. **运动控制：** 根据人的轨迹与机器人轨迹进行路径点导航，优化运动控制效果。
> 3. **上层应用：** 一定程度的人机交互。

其中核心的部分是激光雷达的数据采集与处理，单独依靠激光数据在不依赖任何库的情况下实现目标的识别、锁定与追踪。

---

## 激光雷达数据序列的分析与处理

### 激光数据序列

ROS 中对 **sensor_msgs/LaserScan** 类型的消息定义如下：

```cpp
std_msgs/Header header
	uint32 seq
	time stamp
	string frame_id
float32 angle_min
float32 angle_max
float32 angle_increment
float32 time_increment
float32 scan_time
float32 range_min
float32 range_max
float32[] ranges
float32[] intensities
```

简单来说其中每一帧激光数据就是一个依次包含距离数据的数组，假设激光雷达扫描范围为360°，分辨率为1°，那么每一帧则包含序号由1至360的360个点，每个点存有该方向物体的距离信息。

### 激光数据序列分析

假设距离最大值为255，最小值为0，距离大于255的点取255，那么，当机器人周围没有物体时，当周围有不同尺寸物体时激光数据序列如图。

![无物体](https://d3uepj124s5rcx.cloudfront.net/items/3N223U0U2q1B1R2P2l17/1_1.png)

>无物体时，激光数据序列表现为一条平滑的直线

![不同尺寸的物体](https://d3uepj124s5rcx.cloudfront.net/items/3N0O3x1S0T2D392e3j3F/1_2.png)

>如图所示，上半部分为模拟出的形状、宽度随机的物体，下半部分红点代表差分的数据，绿线代表平滑后的原始数据，通过算法可以分析出物体的位置、宽度等信息，如蓝色的线条所示

由于激光雷达每帧**采样的连续性**，测得表面连续的物体的点也在每一帧数据中呈**连续分布**，那么利用这个特性，对每帧数据队列**求差分**即可获得数据队列中**突变的点**，即物体的**左右边缘**。

![物体检测算法](https://d3uepj124s5rcx.cloudfront.net/items/0N3d233b2k471s3i3K0K/1_3.png)

> 通过左右差分求得突变的点，找到物体边缘，如图，上半部分为随机分布的物体，金字塔的分布用于模拟锁定的目标背后的干扰，假设人背后有干扰，从结果可看出算法可以不受影响，当干扰在人的前方时，此时会利用前一帧的位置信息判断后一帧目标位置分布的可能性，进而锁定目标，后文会详细说明

以上的差分方法获取的物体信息包括物体位置、物体距离、左右边缘位置等，通过进一步分析可以获得的信息还有物体宽度，然后根据宽度可将物体分类为手臂、腰、墙等等类别。此时，根据不同的方案滤出需要的物体，**People-Perception**  的定位方案选择双腿，我们选择腰和手臂。

ROS官网中的 **People-Perception** 功能包利用这个方法扫描人腿：
- **边缘连续物体** 的宽度利用 **左右边缘** 做差求得。
- 用先前设定的符合腿宽度的 **阈值滤除** 不符合条件的物体。
- 剩下的物体中判断两两间的距离，符合正常人两腿宽度的物体标记为人，取两腿坐标的均值为人的位置。

这样做的缺点在于：

- 人在运动过程中左右腿的速度不均匀，会影响估算人中心点算法的稳定性。
- 人在运动过程中左右腿的遮挡引起较大偏差或者误识别。
- 和人腿宽度接近的物体太多，识别率偏低。

我们的改进方案：

- 改变激光高度，改为扫描人的 **腰部**。
- 利用寻找腰部再于附近寻找手臂来确定要跟随的人。
- 利用左右臂的一定程度的手势来组合出几个简单的人机 **交互动作**。

找到人或者疑似人的物体的位置后，此时人与环境中的一些其他物体不能被有效地区别，干扰依旧存在。于是进一步在被标记为人的物体附近寻找是否有手臂，这样做一方面可以区分人与干扰物，另一方面也可以在好几个人中锁定需要跟随的人，即通过人机交互锁定目标。

流程如下：

> 1. 扫描物体
> 2. 滤出符合人的特征的物体
> 3. 通过交互锁定跟随对象
> 4. 跟随
> 5. 是否跟丢？没有跟丢继续跟随
> 6. 跟丢的话重新扫描，再次通过交互锁定跟随对象
> 7. 跟随

### 目标锁定

由于物理世界中位置和速度连续、不会突变，当第一帧锁定目标后，下一帧目标位置的 **可能性分布** 一定在前一帧目标位置的附近，这样的“先验”数据极大地提高了下一帧数据搜索成功的概率，使得目标追踪系统输出更快地收敛到真实目标位置。

设想一个实际场景，当机器人前方站有一个人A时，机器人扫描得到A的位置，然后A通过动作交互使得机器人锁定A，开始跟随。此时，在锁定A的同时，B从机器人与A之间穿插而过，那么有极大的可能会出现机器人被B带走的情况，此时目标追踪系统被输入信号干扰。比赛时围观的人群、障碍物、信号的中断等均会造成类似的干扰，这是因为算法在全局搜索每一帧数据序列，容易出现 **输出发散** 的情况。

前文提到的三种方法中摄像头得到的信息 **维度不足**，Kinect的信息 **过于冗余** 且对计算机造成 **过大压力**，而激光扫腿 **易受干扰** 锁定目标能力有限，激光扫描腰部可以提高定位的成功率和准确性，并且简单的可能性分布可以很大程度上滤除无关的干扰。

在假设的场景中，横穿的B直接会造成传感器失去信号然后锁定其他物品，但是如果引入前提条件，A在以已知速度（速度通过前帧可以测得）于机器人前方离机器人远去，那么下一帧A的位置最有可能在机器人前方更远一些的位置，那么此时横向而来的B挡住A时机器人也大致可以估计出A最有可能的位置，此时目标的追踪就是一个二维平面点的 **轨迹平滑与预测** 问题。

### 抗干扰与目标锁定

根据前一帧的信息预测下一帧的可能性就是一个简单的 **时域滤波器**，此时每一帧的迭代都会刷新下一帧的 **可能性分布**，使得机器人对下一帧人的位置有一个较为准确的预测，利用先验数据做预测是卡尔曼滤波器的思想。

![模拟目标锁定](https://cl.ly/1p1s3c3j0515/tracking_and_lost.gif)

> 利用可能性分布进行目标的锁定，模拟目标缓慢移动时算法锁定追随的目标，最后模拟目标丢失，即目标的运动超出可能性预测的范围，导致目标丢失，此时算法开始全局搜索，寻找新的目标。动图需在网页查看

可以较为准确地获取追随对象的位置时，追随对象的路径点就可以进一步与机器人自主移动起来，形成 **地图与路径**，从而将激光跟随的功能和机器人避障导航结合起来，形成一整套系统。

在整合的系统中，激光数据处理节点与路径规划节点并行运行，搜索被追随者与路径点导航同时进行。
其中被追随者的轨迹点用作机器人运动的路径点，当机器人的路径点存在干扰时，用卡尔曼滤波器的 **平滑功能** 优化路径；当被追随者被障碍物遮挡时或因其他原因造成信号丢失，卡尔曼滤波器的 **预测功能** 依然会工作，为机器人提供一条较为可靠的路径，至此即便机器人跟丢，机器人也可以根据预测数据走到跟丢时离人最近的点进行重新搜索，此时被跟随着只需做一下动作，机器人会立刻重新锁定然后继续跟随。

![卡尔曼滤波](http://images.cnblogs.com/cnblogs_com/zjuhjm/441744/r_kalman.png)

> 卡尔曼滤波器可以在充满噪声的测量信号中对真实值做出较准的估计，提供平滑和预测的功能

---

## 利用地图与路径进行间接运动控制

### 从目标锁定到路径队列

运动控制即根据人的位置控制机器人的运动，前文提到的PID是利用人与机器人的相对位置直接进行PID运算，人靠左机器人向左转，人靠右机器人向右转，人的距离远机器人前进，人的距离近机器人后退，这样的直接控制好比一个“**人体遥控器**”。

如果利用手柄对机器人进行控制，机器人的行为完全依赖于人给的信号，同样，相对位置PID很难称之为机器人自主导航。从比赛情况来看，**相对位置PID** 存在有很多不足：

- 过于依赖人的行为，对于不熟悉机器人的用户操控困难
- 机器人的运动情况需要人的控制，无法与地图联系起来实现自主导航，容易发生碰撞
- PID参数整定困难，运动控制的效果不理想

### 改进后的间接运动控制

改进后的 **间接运动控制** 即 **路径点导航**，其最主要的特征就是将机器人的控制从人工改为自主，所以称之为“**自主导航**”，简单来说就是机器人不跟人走而是跟着人的轨迹走，在追随过程中：

1. 激光数据节点不断更新人的位置，并以一定速率在地图上更新下一个路径点，同时利用概率分布来预测、优化路径点，为运动控制模块提供任务队列
2. 运动控制模块根据机器人当前位置与下一个路径点的误差进行PID控制，以平滑的速度和最优的轨迹运动到下一个路径点，在此过程中PID算法不受任何激光输入信号的干扰，实现 **运动控制和数据处理的分离**
3. 运动控制模块的设想与 **ROS-Navigition** 一致，可以进一步与已知地图导航、无碰撞规划等功能结合起来，形成整体系统

![路径点的平滑优化1](https://d3uepj124s5rcx.cloudfront.net/items/3M2C3o0o1P3c3X2s3l1G/kalman_filter-1.png)

> 绿点代表采样点的真实值，绿线为采样点间的插值，用于记录采样点轨迹，‘+’点代表添加随机噪声后的观测值，红线为卡尔曼滤波器输入观测值后给出的估计值

![路径点的平滑优化2](https://d3uepj124s5rcx.cloudfront.net/items/1G0e3Q1h0J2N1a3n3G20/kalman_filter.png)

> 上图算法用于路径点的平滑优化，卡尔曼滤波器使用通常情况下的默认参数。当人的移动速度较为缓慢，激光定位算法的误差不大时，滤波效果比较好，此处两张图的测量误差呈半径一米的随机分布，可以看出滤波有一定效果

---

## 简单的人机交互

激光数据只有一个维度，扫描仪通过旋转可以给出二维平面点的位置，但是对于激光来说人的特征就仅仅是一个类似圆柱的物体，相比二维图像信息、三维点云信息，激光数据的信息量显得略微匮乏，在这样的情况下，无法很明确地将人从场景中区分出来。对于目标跟随系统，如果指定了 **初始目标**，则如前文所述，目标的追踪问题可以以各种方式解决，但是由于激光数据本身特性所限，初始目标的确认有一些困难。

对于实际应用，可以利用 **给定信号** 指定初始目标，通过激光数据的分析找出一些简单的特征，从而达到一种交互的功能。

由于激光数据是一个水平的截面，人被扫到的形状包含腰和两臂，在数据分析节点进一步进行人周围物体的分析，利用前文求差分的方法可以区分出人的两臂，将左臂开合、右臂开合、双臂开合等动作指定为几个交互动作，并行运行一个人机交互的监视节点即可获取一些简单的交互指令。

我们将锁定初始目标的动作设定为“**双臂开合**”，前文所述“滤出符合人的特征的物体”、“通过交互锁定跟随对象”的实现就是依靠此动作，在追随过程中，假设目标追踪的方法失效，那么使用者依然可以利用交互的方法使得系统重新锁定目标，由此 **提高系统的鲁棒性**。

---

## 实验

### 实验代码说明

实验代码包含两部分：

1. 比赛使用的ROS包，用于RoboCup@Home项目的Follow环节，使用C++实现
	- user_tracker  *目标追随主程序*
	- user_tracker_msgs  *目标追随定义的消息格式*
	- robot_fake  *用于离线调试算法的机器人模拟器*
	- headless_move_base  *适配于上海大学机器人全向轮底盘的运动控制节点*
	- bringup  *启动Follow的Launch文件*

	- *代码全部开源，放置于GitHub不定期进行维护，最新更改请于GitHub查看*
	`https://github.com/yonhdee/RoboCup-Home-Follow`

2. 研究算法使用的Python脚本
	- filter/
		- butterworth.py  *巴特沃斯低通滤波器*
		- kalman.py  *卡尔曼滤波器*
		- smooth.py  *简单的低通平滑滤波器*
	- tracking_simulator.py *鼠标点击模拟激光数据进行跟随*
	- tracking_simulator_analyse.py *鼠标点击模拟激光数据，绘制数据进行算法分析*
	- tracking_kalman.py *用于优化、预测平面轨迹的卡尔曼滤波器，用鼠标轨迹模拟坐标输入进行分析*

	- *代码全部开源，放置于GitHub不定期进行维护，最新更改请于GitHub查看*
	`https://github.com/yonhdee/object_tracking_simulator_python`


### RoboCup@Home 比赛项目的 Follow ROS包

![Follow ROS包](https://d3uepj124s5rcx.cloudfront.net/items/3m320R2w3U090n3S2J3u/Screenshot%20from%202017-02-12%2015-01-07.png)

Follow 的ROS包不依赖任何库，但运行需要ROS的基本环境。

- 编译

	```bash
	cd workspace
	catkin_make --pkg user_tracker_msgs
	source devel/setup.bash
	catkin_make
	```

- 运行

	运行一个仿真实例：

	```bash
	roslaunch bringup follow_simulation.launch
	roslaunch bringup rviz.launch
	```

	说明：

	- 程序启动后在`follow_simulation.launch`终端内按`R`键，给定被跟随者的位置
	- 在`follow_simulation.launch`终端内按`S、W、A、D`键会改变模拟器中激光状态，模拟运动的人的位置。*按W使人由静止变为移动，6秒后开始跟随*
	- 在`follow_simulation.launch`终端内按`R`键，重置人的位置
	- 在`follow_simulation.launch`终端内按`N`键，清除人
	- 机器人底盘的数据由 robot_fake 节点模拟，人的距离过远或者人突然以很快的速度跑开会造成机器人跟丢，此时机器人会发出 “Wait me” 的声音，人走回原位即可重新锁定。

> - 运行截图1
> ![运行截图1](https://d3uepj124s5rcx.cloudfront.net/items/3p2m0H330B1o06043x0B/Image%202017-02-12%20at%207.40.39%20PM.png)
> - 运行截图2
> ![运行截图2](https://d3uepj124s5rcx.cloudfront.net/items/1t3w3j3V330P0z1B0C0b/Image%202017-02-12%20at%207.41.13%20PM.png)
> - 运行截图3
> ![运行截图3](https://d3uepj124s5rcx.cloudfront.net/items/2o0B453B143B1o3w0901/Image%202017-02-12%20at%207.41.27%20PM.png)
> - 运行截图4
> ![运行截图4](https://d3uepj124s5rcx.cloudfront.net/items/1x2G1R2J2O1u0F2U0d2x/Image%202017-02-12%20at%207.48.02%20PM.png)

### 算法研究使用的Python脚本文件

研究算法使用的Python脚本如下：

- tracking_simulator.py *鼠标点击模拟激光数据进行跟随*
- tracking_simulator_analyse.py *鼠标点击模拟激光数据，绘制数据进行算法分析*
- tracking_kalman.py *用于优化、预测平面轨迹的卡尔曼滤波器，用鼠标轨迹模拟坐标输入进行分析*
- filter/
	- butterworth.py  *巴特沃斯低通滤波器*
	- kalman.py  *卡尔曼滤波器*
	- smooth.py  *简单的低通平滑滤波器*

脚本运行环境配置：

```bash
sudo apt insall pip # python-pip
sudo pip install pip --upgrade
sudo pip install numpy scipy sympy pillow matplotlib --upgrade
sudo pip install pykalman # Packages for Engineering
```

**tracking_simulator.py：** 用Python写的算法模拟器

![模拟目标锁定](https://cl.ly/1p1s3c3j0515/tracking_and_lost.gif)

> 利用可能性分布进行目标的锁定，模拟目标缓慢移动时算法锁定追随的目标，最后模拟目标丢失，即目标的运动超出可能性预测的范围，导致目标丢失，此时算法开始全局搜索，寻找新的目标。动图需在网页查看

界面分为上下两部分，上半部分模拟激光雷达的裸数据，下半部分显示算法数据。用鼠标左键点击输入，模拟激光数据供跟随算法使用，点击画布灰色部分可以清空画布。

下半部分红色圆点代表差分数据，用于进行物体检测以及特征分析，绿色线条代表激光数据的低通平滑效果，‘\*’形线代表锁定人之后下一帧可能性区域的蒙版，下一帧如果人的位置没有突变，算法在此区域局部搜索，从而实现‘锁定人’的效果。

如果下一帧仍可以锁定人，根据人的位置更新可能性区域蒙版，如果下一帧在较远的位置点击左键，算法判定跟丢，可能性区域蒙版扩展为全局，算法在下一帧实行全局搜索，重新锁定目标。

**tracking_simulator_analyse.py：** 用Python写的算法模拟器，用于分析数据

![不同尺寸的物体](https://d3uepj124s5rcx.cloudfront.net/items/3N0O3x1S0T2D392e3j3F/1_2.png)

界面分为上下两部分，上半部分模拟激光雷达的裸数据，下半部分显示算法数据。

- 鼠标左键单击：清空画布，添加模拟人的数据
- 鼠标右键单击：不清空画布，用右键点击绘制任意情景的数据
- 鼠标中键单击：对原始数据进行平滑滤波
- 点击画布灰色部分可以清空画布

下半部分显示算法数据，模拟激光数据供跟随算法使用，点击画布灰色部分可以清空画布。
红色圆点代表差分数据，用于进行物体检测以及特征分析，绿色线条代表激光数据的低通平滑效果，‘\*’形线代表分析出的‘物体蒙版’，值为‘1’的部分代表有边缘连续的物体，线条宽度代表物体宽度，用于对物体进行分析、分类。

**tracking_kalman.py：** 鼠标模拟输入，卡尔曼滤波效果

![ 卡尔曼滤波](https://d3uepj124s5rcx.cloudfront.net/items/3M2C3o0o1P3c3X2s3l1G/kalman_filter-1.png)

用鼠标在画布走过的轨迹模拟输入二维的点序列，对每次采样后的点坐标添加随机噪声，再对存在噪声的信号进行卡尔曼滤波。

界面模拟 30m*30m 的平面，鼠标移动到画布内程序开始定时采样，记录鼠标的轨迹，并添加半径为1m的随机噪声作为干扰。
缓慢移动鼠标，绿点代表采样点的真实值，绿线为采样点间的插值，用于记录采样点轨迹，‘+’点代表添加随机噪声后的观测值，红线为卡尔曼滤波器输入观测值后给出的估计值。

**filter/：** 滤波器：一些常用滤波器的Demo

- butterworth.py  *巴特沃斯低通滤波器*
- kalman.py  *卡尔曼滤波器*
- smooth.py  *利用门信号卷积实现简单的低通平滑滤波器*

---

