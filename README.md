# Apollo 2.0阅读笔记

本文档主要介绍Apollo 2.0软件平台结构，由于专业方向为视觉感知，以及未来从事方向为路径规划，因此阅读笔记主要包含对应的两部分。文档的目录结构为：

<!-- 添加隐藏链接，便于后续小结返回 -->
<a name="目录头"></a>
- [1. Apollo 2.0 软件结构简介](#总体软件结构)
	- [1.1 软件结构图](#软件结构图)
	- [1.2 感知模块: Perception](#感知模块)
	- [1.3 预测模块: Prediction](#预测模块)
	- [1.4 路由模块: Routing](#路由模块)
	- [1.5 规划模块: Planning](#规划模块)
	- [1.6 控制模块: Control](#控制模块)
	- [1.7 控制区域网络模块: CanBus](#控制区域网络模块)
	- [1.8 高清地图模块: HD-Map](#高清地图模块)
	- [1.9 定位模块: Localization](#定位模块)
- [2. 感知模块笔记](#感知模块详解)
	- [2.1 代码层次结构图](#代码层次结构)
	- [2.2 障碍物感知: 3D Obstacles Perception](#障碍物感知)
	- [2.3 信号灯感知: Traffic Light Perception](#信号灯感知)

## <a name="总体软件结构">1. Apollo 2.0总体软件结构简介</a>
本章主要介绍Apollo 2.0的软件结构，粗略的解释总体组成模块以及每个模块的功能，代码请参考([Apollo 2.0 Github](https://github.com/ApolloAuto/apollo)), 软件框架请参考([Apollo 2.0软件架构](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/Apollo_2.0_Software_Architecture.md))。

### <a name="软件结构图">1.1 软件结构图</a>

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/Apollo_2_0_Software_Arch.png)

上图每个模块都以独立的ROS node运行，相互之间的消息传递依赖ROS的消息发布与订阅机制。消息订阅(subsrcibe)等同于数据输入(data input)，而消息发布(publish)等同于数据输出(data output)。

### <a name="感知模块">1.2 感知模块: Perception</a>

感知模块主要功能是识别周围环境树木，人，路面，交通灯等信息，为后续路径规划，控制等做辅助。感知模块主要包含两个重要的子模块：

- 障碍物检测(3D obstacles perception).
- 交通信号灯检测(Traffic light perception).

感知模块输入来源于汽车物理感知设备，主要包含激光雷达点云数据，视觉摄像头数据。同时，交通信号灯检测也依赖于其他模块提供的数据，包含定位模块数据，高清地图数据(信号灯实时检测是不可行的或者说是不合理不必要的，原因在于太过耗费计算资源，实际只需要定位与高精度地图模块提供相应的位置信息，指示感知模块在什么时间段与路面段做信号灯检测)，E.g.路口需要启动信号灯检测，中间路段只需要障碍物检测，不存在信号灯。

#### <a name="感知模块输入">感知模块输入数据</a>

- Point cloud data/点云数据 (ROS topic /apollo/sensor/velodyne64/comensator/pointCloud2)
- Radar data/雷达数据 (ROS topic /apollo/sensor/conti_radar)
- Image data/图像数据 (ROS topic /apollo/sensor/camera/traffic/image_long & /apollo/sensor/camera/traffic/image_short)
- Coordinate frame transformation information over time/坐标系转换数据 (ROS topic /tf)
- HDMap/高清地图
- Extrinsic parameters of LiDAR sensor calibration/激光雷达传感器矫正外参 (ROS topic /tf_static)
- Extrinsic parameters of radar sensor calibration/雷达传感器矫正外参 (来自外部YAML文件)
- Extrinsic and Intrinsic parameters of all camera calibration/所有相机/长焦+短焦相机的内参和外参 (来自外部YAML文件)
- Velocity of host vehicle/主车体速度 (ROS topic /apollo/localization/pos)

#### <a name="感知模块输出">感知模块输出数据</a>

- 3D障碍物跟踪的方向，速度以及树，人，车辆等分类信息
- 交通信号灯的标定框以及信号灯状态信息

### <a name="预测模块">1.3预测模块: Prediction</a>

预测模块主要功能为从感知模块获取障碍物信息，包括位置，方向，速度，加速度等。最终预测障碍物未来时刻的运动轨迹。当感知模块发布障碍物信息时，自动触发预测模块。

#### <a name="预测模块输入">预测模块输入数据</a>

- Obstacles from perception module/感知模块障碍物信息
- Localization from localization module/定位模块信息

#### <a name="预测模块输出">预测模块输出数据</a>

- Obstacles additionally with predicted trajectories/障碍物预测轨迹

#### <a name="预测模块函数">预测模块函数与功能</a>

- Contrainer/容器
Container存储来自ROS订阅的输入数据，目前版本支持存储感知障碍物(perception onstacles)，车辆定位(vehicle localization)，车辆轨迹规划(vehicle planning)

- Evaluator/评估器
Evaluator对视野中的每个障碍物独立评估器路径与速度。评估器通过使用保存在prediction/data下的模型数据来对每个车道序列进行路径评估，输出车辆在每个车道的概率。

- Predictor/预测器
Predictor最终生成障碍物的预测轨迹，目前支持的预测器有：
	- Lane sequence: 障碍物沿着车道线运动
	- Move sequence: 障碍物遵循汽车动态模式，沿车道线运动(没翻译好，不准确)
	- Free movement: 障碍物自由运动
	- Regional movement: 障碍物在一定区域内运动

### <a name="路由模块">1.4 路由模块: Routing</a>

路由模块根据请求(起始地与目的地信息)生成高层的导航信息，通过一系列的车道线与车道信息，指示车辆如何抵达目的地

#### <a name="路由模块输入">路由模块输入数据</a>

- Map data/地图数据
- Routing request/路由请求，包含起始地与目的地位置信息。

#### <a name="路由模块输出">路由模块输出数据</a>

- 路由导航信息

### <a name="规划模块">1.5 规划模块: Planning</a>

规划模块为自动驾驶车辆规划时空轨迹。在Apollo 2.0中，规划模块使用多个信息源来规划安全，无碰撞的运动轨迹，所以规划模块几乎可以与其他任何模块交互(localization, vehicle status (position, velocity, acceleration, chassis), map, routing, perception and prediction)。

(1). 初始化时，规划模块使用预测模块输出信息。预测模块输出包含了原始障碍物信息及其运动轨迹信息。为什么规划模块不使用感知模块Perception/3D_obstacles输出信息？因为规划模块订阅的topic为交通信号灯Perception/traffic_light topic，而没有订阅Perception/3D_obstacles topic，所以不能直接使用。

(2). 紧接着，规划模块使用路由模块输出信息。在某些特定情况下，如果不能按照路由模块导航路线行驶，规划模块可以发送请求并触发路由模块进行新的计算，生成新的导航信息。E.g. 路上发生交通事故，需要重新导航新的路径。

(3). 最终，规划模块需要了解定位信息，以及当前车辆状态信息。规划模块有特定的函数进行一定频率的触发。

支持两类主要的规划器Planner：
- RTK replay planner(since Apollo 1.0)
- EM planner (since Apollo 1.5)

#### <a name="规划模块输入">规划模块输入数据</a>

- RTK replay planner
	- Localization/定位信息
	- Recorded RTK trajectory/记录的RTK轨迹(put into the folder modules/planning/data, and change the gflag file name in planning/common/planning_gflags)
- EM planner
	- Localization/定位信息
	- Perception/感知模块信息 traffic light
	- Prediction/预测模块信息
	- HD Map/高清地图信息
	- Routing/路由导航信息

#### <a name="规划模块输出">规划模块输出数据</a>

- 安全合理的运动轨迹信息，供控制模块执行

### <a name="控制模块">1.6 控制模块: Control</a>

控制模块通过生成诸如节流阀，制动器，转向的控制指令给CanBus模块，执行来自规划模块的时空轨迹命令。

#### <a name="控制模块输入">控制模块输入数据</a>

- Planning trajectory/规划路径
- Car status/车辆状态
- Localization/定位信息
- Dreamview AUTO mode change request/模式切换请求

#### <a name="控制模块输出">控制模块输出数据</a>

- control command (steering, throttle, brake) to canbus/具体的控制指令

### <a name="控制区域网络模块">1.7  控制区域网络模块:  CanBus</a>

类似于下位机，控制区域网络模块讲来自于控制模块的具体控制指令传递给汽车硬件，同时向上层软件反馈车辆硬件信息。

#### <a name="控制区域网络模块输入">控制区域网络模块输入数据</a>

- Control command/控制指令

#### <a name="控制区域网络模块输出">控制区域网络模块输出数据</a>

- Chassis status/机箱状态
- Chassis detail status/机箱具体状态

### <a name="高清地图模块">1. 8 高清地图模块: HD Map</a>

高清地图模块类似于一个库，不像其他模块使用发布与订阅机制，他频繁的被调用用来检索相关道路的结构信息。

### <a name="定位模块">1.9 定位模块: Localization</a>

定位模块从不同的信息源，如GPS(经纬度)，LiDAR(距离)和IMU(加速度等)来评估车辆当前的信息。常用的定位技术有两类：

- 基于RTK(Real-Time Kinematic, 实时动态载波相位差分技术)定位。由OnTimer函数以一定频率触发定位。
- 多传感器融合(Multiple Sensor Fusion, MSF)定位。由一系列传感器触发函数触发。

#### <a name="定位模块输入">定位模块输入数据</a>

- RTK-base method.
	- GPS - Global Position System/全球定位系统
	- IMU - Interial Measurement Unit/惯性测量单元
- MSF method.
	- GPS - Global Position System/全球定位系统
	- IMU - Interial Measurement Unit/惯性测量单元
	- LiDAR - Light Detection And Ranging Sensor/光检测和测距传感器，激光雷达

[返回目录](#目录头)

## <a name="感知模块详解">2. 感知模块笔记</a>

本章节主要详细的解析Apollo 2.0感知模块Perception代码与功能结构，相关资料请参考([Perception: 3D Obstacles](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/3d_obstacle_perception.md)和[Percepton: Traffic Light](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/traffic_light.md))

### <a name="代码层次结构">2.1 代码层次结构图</a>

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_software_arch.png)

感知模块框架本质是一个DAG有向图，该图由3类基本元素组成，包括：子节点Sub Node，边Edge和共享数据Share Data。框架中的每一个功能都以子节点SubNode的形式存在，以线程形式运行；子节点之间的共享数据ShareData沿着边Edge有向流动，从产生子节点流向需求子节点。上图中第二行分别初始化共享数据，子节点以及DAG，最终DAG运行执行感知功能。

#### <a name="共享数据初始化">共享数据初始化</a>

```
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  AdapterManager::Init(FLAGS_perception_adapter_config_filename);

  RegistAllOnboardClass();
  ...
}
void Perception::RegistAllOnboardClass() {
  /// regist sharedata
  RegisterFactoryLidarObjectData();
  RegisterFactoryRadarObjectData();
  traffic_light::RegisterFactoryTLPreprocessingData();
  ...
}
```
共享数据包含3类，分别为：
- LiDARObjectData/激光测距仪数据，用于障碍物感知/3D Obstacle Perception
- RadarObjectData/雷达数据，用于障碍物感知/3D Obstacle Perception
- TLPreprocessingData/交通灯预处理数据，用于信号灯感知/Traffic Light Perception

以LiDARObjectData注册初始化为例，共享数据初始化分两步：

(1) 对应共享数据容器类注册，注册LidarObjectData
```
/// file in apollo/modules/perception/obstacle/onboard/object_share_data.h
#define OBJECT_SHARED_DATA(data_name)                        \
  class data_name : public CommonSharedData<SensorObjects> { \
   public:                                                   \
    data_name() : CommonSharedData<SensorObjects>() {}       \
    virtual ~data_name() {}                                  \
    std::string name() const override {                      \
      return #data_name;                                     \
    }                                                        \
    ...														 \
  }

OBJECT_SHARED_DATA(LidarObjectData);
...
```
该过程通过宏注册对应的共享数据类，继承CommonSharedData基础类，其中CommonSharedData类包含的元素如下

| 名称 | 返回 | 备注 |
| ---- | ---- | ---- |
| Init() | bool | 初始化标记 |
| name() | bool | 共享数据名称 |
| Reset() | void | 清空所有共享数据，当重置DAGStreaming时，ShareDataManager调用 |
| RemoveStaleData() | void | 清空共享数据map中过时数据，当前时刻-数据时间戳大于人工设定的共享数据过期时间，则清空 |
| Add(const std::string &key, const SharedDataPtr<M> &data) | bool | 根据键-值对添加新的共享数据，key为字符串类型 |
| Add(const CommonSharedDataKey &key, const SharedDataPtr<M> &data) | bool | 根据键-值对添加新的共享数据 ，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Get(const std::string &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，key为字符串类型 |
| Get(const CommonSharedDataKey &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Pop(const std::string &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，并从map中删除，key为字符串类型 |
| Pop(const CommonSharedDataKey &key, SharedDataPtr<M> \*data) | bool | 由key获取共享数据，存储进data，并从map中删除，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Remove(const std::string &key) | bool | 根据key删除共享数据，key为字符串类型 |
| Remove(const CommonSharedDataKey &key) | bool | 根据key删除共享数据，key为CommonSharedDataKey类型(包含时间戳与设备id) |
| Size() | unsigned | 共享数据类map中存储的数据量 |
| GetStat() | CommonSharedDataStat | 返回类操作记录，增加数据次数，删除数据次数，获取数据次数 |
| std::map<std::string, SharedDataPtr<M>> SharedDataMap/data_map_ | -- | 共享数据存储容器map |
| std::map<std::string, uint64_t> DataAddedTimeMap/data_added_time_map_ | -- | map中数据加入的时间戳，配合用于删除过时数据 |
| CommonSharedDataStat stat_ | -- | 类操作记录: 增加数据次数，删除数据次数，获取数据次数 |

由上表可知，第一步注册对应的LidarObjectData主要的工作是创建一个Lidar数据的容器类，用以数据的存储，删除与查询。数据以一定格式(ShareData子类)存储在map中，每个数据标有时间戳和设备id，并定时清理旧数据。

(2) 创建共享数据容器类实例与保存类函数，方便实例化LidarObjectData与保存

```
/// file in apollo/modules/perception/obstacle/onboard/object_share_data.h
#define OBJECT_SHARED_DATA(data_name)                        \
  class data_name : public CommonSharedData<SensorObjects> { \
   public:                                                   \
    data_name() : CommonSharedData<SensorObjects>() {}       \
    virtual ~data_name() {}                                  \
    std::string name() const override {                      \
      return #data_name;                                     \
    }                                                        \
    ...                                                      \
  }
REGISTER_SHAREDDATA(LidarObjectData);
...

/// file in apollo/modules/perception/onboard/shared_data.h
#define REGISTER_SHAREDDATA(name) REGISTER_CLASS(SharedData, name)

/// file in apollo/modules/perception/lib/base/registerer.h
typedef std::map<std::string, ObjectFactory \*> FactoryMap;
typedef std::map<std::string, FactoryMap> BaseClassMap;
BaseClassMap &GlobalFactoryMap();

#define REGISTER_CLASS(clazz, name)                                           \
  class ObjectFactory##name : public apollo::perception::ObjectFactory {      \
   public:                                                                    \
    virtual ~ObjectFactory##name() {}                                         \
    virtual perception::Any NewInstance() {                                   \
      return perception::Any(new name());                                     \
    }                                                                         \
  };                                                                          \
  inline void RegisterFactory##name() {                                       \
    perception::FactoryMap &map = perception::GlobalFactoryMap()[#clazz];     \
    if (map.find(#name) == map.end()) map[#name] = new ObjectFactory##name(); \
  }
```

总结可知REGISTER_SHAREDDATA宏实际是创建共享数据容器类实例化与保存函数，通过调用该宏生成的函数可以方便的实例化对应的容易类并添加至全局工厂管理类，方便管理所有共享数据实例。E.g. 当在perception.cc的RegistAllOnboardClass中调用RegisterFactoryLidarObjectData()时，实际是实例化对应的容器类LidarObjectData，最终存储进GlobalFactoryMap中，存储的形式为：GlobalFactory[SharedData][LidarObjectData]两级存储。

#### <a name="子节点初始化">子节点SubNode初始化</a>
```
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  AdapterManager::Init(FLAGS_perception_adapter_config_filename);

  RegistAllOnboardClass();
  ...
}
void Perception::RegistAllOnboardClass() {
  ...
  /// regist subnode
  RegisterFactoryLidarProcessSubnode();
  RegisterFactoryRadarProcessSubnode();
  RegisterFactoryFusionSubnode();
  traffic_light::RegisterFactoryTLPreprocessorSubnode();
  traffic_light::RegisterFactoryTLProcSubnode();
}
```

子节点SubNode是程序中的某个功能块，每个子节点都是一个线程，在感知模块中，一共存在5个子节点：
- LidarProcessSubnode/激光测距处理子节点，用于障碍物感知/3D Obstacle Perception
- RadarProcessSubnode/雷达处理子节点，用于障碍物感知/3D Obstacle Perception
- FusionSubnode/融合模式处理子节点，用于障碍物感知/3D Obstacle Perception
- TLPreprocessSubnode/信号灯预处理子节点，用于信号灯感知/Traffic Light Perception
- TLProcessSubnode/信号灯处理子节点，用于信号灯感知/Traffic Light Perception

以TLPreprocessSubnode注册初始化为例，子节点初始化
```
/// file in apollo/modules/perception/onboard/tl_preprocessor_subnode.h
class TLProcSubnode : public Subnode {
 public:
  ...
 protected:
  ...
 private:
 ...
};
REGISTER_SUBNODE(TLPreprocessorSubnode);

/// file in apollo/modules/perception/onboard/subnode.h
#define REGISTER_SUBNODE(name) REGISTER_CLASS(Subnode, name)

/// file in apollo/modules/perception/lib/base/registerer.h
typedef std::map<std::string, ObjectFactory \*> FactoryMap;
typedef std::map<std::string, FactoryMap> BaseClassMap;
BaseClassMap &GlobalFactoryMap();

#define REGISTER_CLASS(clazz, name)                                           \
  class ObjectFactory##name : public apollo::perception::ObjectFactory {      \
   public:                                                                    \
    virtual ~ObjectFactory##name() {}                                         \
    virtual perception::Any NewInstance() {                                   \
      return perception::Any(new name());                                     \
    }                                                                         \
  };                                                                          \
  inline void RegisterFactory##name() {                                       \
    perception::FactoryMap &map = perception::GlobalFactoryMap()[#clazz];     \
    if (map.find(#name) == map.end()) map[#name] = new ObjectFactory##name(); \
  }
```

与前小节类似，REGISTER_SUBNODE宏作用是生成对应的Subnode类，同时创建该Subnode类的实例化与保存函数，通过调用RegisterFactoryTLPreprocessorSubnode可以方便的实例化该子节点类，并保存到全局工厂管理类中，存储的形式为：GlobalFactory[Subnode][TLPreprocessorSubnode]两级存储。

TLPreprocessorSubnode继承了Subnode类，可以进一步分析Subnode基类，该类主要包含的元素如下

| 名称 | 返回 | 备注 |
| ---- | ---- | ---- |
| InitInternal() | bool | 内部初始化函数，初始化边界映射函数，共享数据，信号灯预处理器，高清地图等。 |
| InitSharedData() | bool | 初始化共享数据类TLPreprocessingData，在InitInternal函数中被调用 |
| InitPreprocessor() | bool | 初始化预处理器(实际做预处理工作类)，在InitInternal函数中被调用 |
| InitHdmap() | bool | 初始化高清地图类，在InitInternal函数中被调用 |
| AddDataAndPublishEvent(const std::shared_ptr<ImageLights> &data,const CameraId &camera_id, double timestamp) | bool | 将数据存储进共享数据实例的map中，并交由EventManager发布消息信息，在SubCameraImage函数中被调用 |
| SubLongFocusCamera(const sensor_msgs::Image &msg) | void | 选择长焦摄像头回调函数，在InitInternal函数中被调用注册回调函数 |
| SubShortFocusCamera(const sensor_msgs::Image &msg) | void | 选择短焦摄像头回调函数，在InitInternal函数中被调用注册回调函数 |
| SubCameraImage(boost::shared_ptr<const sensor_msgs::Image> msg, CameraId camera_id) | void | 主体函数，获取图像，相机选择，图像处理等 |
| CameraSelection(double ts) | void | 相机选择，在SubCameraImage函数中被调用 |
| VerifyLightsProjection(std::shared_ptr<ImageLights> image_lights) | bool | 验证信号灯映射，在SubCameraImage函数中被调用 |
| GetSignals(double ts, CarPose \*pose, std::vector<apollo::hdmap::Signal> \*signals) | bool |获取汽车姿态与图像信息，，在SubCameraImage函数中被调用 |
| GetCarPose(const double ts, CarPose \*pose) | bool | 获取汽车姿态信息，在GetSignals函数被调用 |
| TLPreprocessor preprocessor_ | -- | 预处理器 |
| TLPreprocessingData \*preprocessing_data_ | -- | 预处理共享数据容器类指针 |
| HDMapInput \*hd_map_ | -- | 高清地图类指针 |
| last_signals_ts_  | -- | 上一次调用GetSignals的时间戳，若相隔太短，直接用上次的信息 |
| std::vector<apollo::hdmap::Signal> last_signals_ | -- | 上一次的信息 |
| last_query_tf_ts_ | -- | 上一次调用CameraSelection时间戳，若相隔太短，则直接调用该步骤，同时跳过GetSignals和CacheLightsProjections |
| last_proc_image_ts_ | -- | 上一次图像处理时间戳，若相隔太短，掉过后续图像选择步骤 |

从TLPreprocessorSubnode成员函数可以看到，子节点类主要为ROS消息发布与订阅机制(或者是定时触发机制)完善回调函数，在回调函数中执行相应的功能，5类子节点都具有相同的类形式，但功能不同。具体的功能在下小节描述。

#### <a name="有向图初始化">有向图DAG初始化</a>

DAG初始化过程主要是构建子节点SubNode，边Edge和共享数据ShareData的一个有向图，关于有向图的三部分内容，程序从config文件读入
```
/// file in apollo/modules/perception/perception.cc
Status Perception::Init() {
  ...
  // work_root: modules/perception
  // dag_config_path: ./conf/dag_streaming.config
  const std::string dag_config_path = apollo::common::util::GetAbsolutePath(
      FLAGS_work_root, FLAGS_dag_config_path);
  ...
}

/// file in apollo/modules/perception/conf/dag_streaming.config
# Define all nodes in DAG streaming.
subnode_config {
    # 64-Lidar Input nodes.
    subnodes {
        id: 1
        name: "LidarProcessSubnode"
        reserve: "device_id:velodyne64;"
        type: SUBNODE_IN
    }
    ...
}

# Define all edges linked nodes.
edge_config {
    # 64-Lidar LidarProcessSubnode -> FusionSubnode
    edges {
        id: 101
        from_node: 1
        to_node: 31
        events {
            id: 1001
            name: "lidar_fusion"
        }
    }
    ...
}

data_config {
    datas {
        id: 1
        name: "LidarObjectData"
    }
 	...
}
```

(1) 在DAG三部分初始化过程中，子节点SubNode已经在上小节中完成初始化，5个子节点分别开启5个线程，每个线程设置对应的回调函数，当有输入的时候(收到ROS订阅的消息topic或者定时触发)，自动触发子节点功能。在代码中子节点的初始化工作包含SubNode配置记录(节点id，入度id，出度id)，子节点实例化。上小节"子节点SubNode初始化"讲述了子节点SubNode初始化仅仅创建实例化的函数，调用该函数可以实例化子节点，并保存在GlobalFactory[Subnode][TLPreprocessorSubnode]两级存储中。本节是真正的实例化对象。

```
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::InitSubnodes(const DAGConfig& dag_config) {
  const DAGConfig::SubnodeConfig& subnode_config = dag_config.subnode_config();
  const DAGConfig::EdgeConfig& edge_config = dag_config.edge_config();
  
  map<SubnodeID, DAGConfig::Subnode> subnode_config_map;
  map<SubnodeID, vector<EventID>> subnode_sub_events_map;
  map<SubnodeID, vector<EventID>> subnode_pub_events_map;
   
  for (auto& subnode_proto : subnode_config.subnodes()) {
    std::pair<map<SubnodeID, DAGConfig::Subnode>::iterator, bool> result =
        subnode_config_map.insert(std::make_pair(subnode_proto.id(), subnode_proto));
    ...
  }

  for (auto& edge_proto : edge_config.edges()) {
    SubnodeID from = edge_proto.from_node();
    SubnodeID to = edge_proto.to_node();
    ...
    for (auto& event_proto : edge_proto.events()) {
      subnode_pub_events_map[from].push_back(event_proto.id());
      subnode_sub_events_map[to].push_back(event_proto.id());
    }
  }

  // Generate Subnode instance.
  for (auto pair : subnode_config_map) {
    const DAGConfig::Subnode& subnode_config = pair.second;
    const SubnodeID subnode_id = pair.first;
    Subnode* inst = SubnodeRegisterer::GetInstanceByName(subnode_config.name());
    ...
    bool result = inst->Init(
        subnode_config, &event_manager_, &shared_data_manager_,
        subnode_sub_events_map[subnode_id], subnode_pub_events_map[subnode_id]);
    ...
    subnode_map_.emplace(subnode_id, std::unique_ptr<Subnode>(inst));
  }
  ...
}

/// file in apollo/modules/perception/lib/base/registerer.h
#define REGISTER_REGISTERER(base_class)                               \
  class base_class##Registerer {                                      \
   public:                                                            \
    static base_class *GetInstanceByName(const ::std::string &name) { \
      FactoryMap &map = perception::GlobalFactoryMap()[#base_class];  \
      FactoryMap::iterator iter = map.find(name);                     \
      Any object = iter->second->NewInstance();                       \
      return *(object.AnyCast<base_class *>());                       \
    }                                                                 \
};
```

(2) 在DAG初始化过程中，边Edge初始化需要依赖EvenManager完成，其实本质上Edge是数据的流动过程，也就是消息的订阅与发布过程，所以Apollo中使用EvenManager统一管理各个topic消息的订阅与发布

```
/// file in apollo/modules/perception/onboard/dag_streaming.cc
bool DAGStreaming::Init(const string& dag_config_path) {
  ...
  if (!event_manager_.Init(dag_config.edge_config())) {
    AERROR << "failed to Init EventManager. file: " << dag_config_path;
    return false;
  }
  ...
}

/// file in aploo/modules/perception/onboard/event_manager.cc 
bool EventManager::Init(const DAGConfig::EdgeConfig &edge_config) {
  ...
  for (const DAGConfig::Edge &edge : edge_config.edges()) {
    for (const DAGConfig::Event event_pb : edge.events()) {
      if (event_queue_map_.find(event_pb.id()) != event_queue_map_.end()) {
        AERROR << "duplicate event id in config. id: " << event_pb.id();
        return false;
      }

      event_queue_map_[event_pb.id()].reset(
          new EventQueue(FLAGS_max_event_queue_size));

      EventMeta event_meta;
      event_meta.event_id = event_pb.id();
      event_meta.name = event_pb.name();
      event_meta.from_node = edge.from_node();
      event_meta.to_node = edge.to_node();
      event_meta_map_.emplace(event_pb.id(), event_meta);
      AINFO << "load EventMeta: " << event_meta.to_string();
    }
  }
  ...
}

/// file in apollo/modules/perception/onboard/dag_streaming.h
class EventManager {
  ...
private:
  using EventQueue = FixedSizeConQueue<Event>;
  using EventQueueMap = std::unordered_map<EventID, std::unique_ptr<EventQueue>>;
  using EventMetaMap = std::unordered_map<EventID, EventMeta>;

  EventQueueMap event_queue_map_;
  EventMetaMap event_meta_map_;
};
```
由代码分析可知，EvenManager类包含两个主要的成员变量，分别保存<事件id，消息队列>的event_queue_map_，以及保存<事件id，事件信息>的event_meta_map_。一个事件的成分包含id和name。一个完成的Edge总体保存了事件信息(id，name)，入度节点(from_node)，出度节点(to_node)

(3) 最后的共享数据初始化依赖


### <a name="障碍物感知">2.2 障碍物感知: 3D Obstacles Perception</a>

### <a name="信号灯感知">2.3 信号灯感知: Traffic Light Perception</a>
Status Perception::Init() {
  AdapterManager::Init(FLAGS_perception_adapter_config_filename);

  RegistAllOnboardClass();
  /// init config manager
  ConfigManager* config_manager = ConfigManager::instance();
  if (!config_manager->Init()) {
    AERROR << "failed to Init ConfigManager";
    return Status(ErrorCode::PERCEPTION_ERROR, "failed to Init ConfigManager.");
  }
  AINFO << "Init config manager successfully, work_root: "
        << config_manager->work_root();

  const std::string dag_config_path = apollo::common::util::GetAbsolutePath(
      FLAGS_work_root, FLAGS_dag_config_path);

  if (!dag_streaming_.Init(dag_config_path)) {
    AERROR << "failed to Init DAGStreaming. dag_config_path:"
           << dag_config_path;
    return Status(ErrorCode::PERCEPTION_ERROR, "failed to Init DAGStreaming.");
  }
  callback_thread_num_ = 5;

  return Status::OK();
}

[返回目录](#目录头)