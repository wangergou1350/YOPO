# YOPO ROS2 迁移指南

## 目录
1. [迁移概述](#迁移概述)
2. [ROS1 与 ROS2 主要差异](#ros1-与-ros2-主要差异)
3. [环境准备](#环境准备)
4. [代码迁移步骤](#代码迁移步骤)
5. [包结构调整](#包结构调整)
6. [编译系统迁移](#编译系统迁移)
7. [Python代码迁移](#python代码迁移)
8. [C++代码迁移](#c代码迁移)
9. [Launch文件迁移](#launch文件迁移)
10. [消息和服务迁移](#消息和服务迁移)
11. [参数服务器迁移](#参数服务器迁移)
12. [迁移检查清单](#迁移检查清单)

---

## 迁移概述

### 为什么迁移到 ROS2？

- **更好的实时性能**: DDS通信中间件提供更低延迟
- **更强的类型安全**: 强类型接口和更好的错误检查
- **跨平台支持**: Windows, Linux, macOS
- **安全性**: 内置的安全通信机制
- **生命周期管理**: 更好的节点状态管理
- **长期支持**: ROS1 已停止维护，ROS2 是未来趋势

### 迁移难度评估

| 组件 | 难度 | 工作量 | 主要改动 |
|------|------|--------|----------|
| **Simulator** | 🔴 高 | 3-5天 | CUDA代码、CMake、消息类型 |
| **Controller** | 🟡 中 | 2-3天 | CMake、launch文件、话题重映射 |
| **YOPO** | 🟢 低 | 1-2天 | Python API、话题订阅/发布 |

---

## ROS1 与 ROS2 主要差异

### 1. 通信机制
```
ROS1: TCPROS/UDPROS (自定义协议)
ROS2: DDS (Data Distribution Service)
```

### 2. 节点管理
```python
# ROS1
import rospy
rospy.init_node('my_node')

# ROS2
import rclpy
from rclpy.node import Node
class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
```

### 3. 话题发布/订阅
```python
# ROS1
pub = rospy.Publisher('/topic', String, queue_size=10)
sub = rospy.Subscriber('/topic', String, callback)

# ROS2
pub = self.create_publisher(String, '/topic', 10)
sub = self.create_subscription(String, '/topic', callback, 10)
```

### 4. 参数
```python
# ROS1
rospy.get_param('/param_name', default_value)

# ROS2
self.declare_parameter('param_name', default_value)
value = self.get_parameter('param_name').value
```

### 5. Launch 文件
```xml
<!-- ROS1: XML -->
<launch>
    <node pkg="package" type="executable" name="node_name"/>
</launch>
```

```python
# ROS2: Python
from launch import LaunchDescription
from launch_ros.actions import Node

def generate_launch_description():
    return LaunchDescription([
        Node(package='package', executable='executable', name='node_name')
    ])
```

### 6. 编译系统
```
ROS1: catkin_make / catkin build (CMake)
ROS2: colcon build (ament_cmake / ament_python)
```

---

## 环境准备

### 系统要求
- **操作系统**: Ubuntu 22.04 (ROS2 Humble) 或 Ubuntu 24.04 (ROS2 Jazzy)
- **ROS2 版本**: Humble Hawksbill (LTS) 推荐
- **Python**: 3.10+
- **CUDA**: 11.8+ (用于 Simulator)
- **Conda**: Anaconda/Miniconda (与ROS2共存需要注意)

### 安装 ROS2 Humble

```bash
# 设置 locale
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 添加 ROS2 仓库
sudo apt install software-properties-common
sudo add-apt-repository universe
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 安装 ROS2
sudo apt update
sudo apt install ros-humble-desktop
sudo apt install ros-dev-tools

# 安装 colcon
sudo apt install python3-colcon-common-extensions
```

### 安装依赖

```bash
# ROS2 相关
sudo apt install ros-humble-cv-bridge
sudo apt install ros-humble-image-transport
sudo apt install ros-humble-vision-opencv
sudo apt install ros-humble-pcl-ros
sudo apt install ros-humble-pcl-conversions

# 其他依赖
sudo apt install libyaml-cpp-dev
sudo apt install libeigen3-dev
```

### Python 虚拟环境

```bash
# 创建 ROS2 兼容的虚拟环境
conda create --name yopo_ros2 python=3.10
conda activate yopo_ros2

# 安装 Python 依赖
pip install torch torchvision
pip install numpy scipy opencv-python
pip install pyyaml
pip install transforms3d  # ROS2 推荐使用而非 scipy.spatial.transform
```

---

## 代码迁移步骤

### 整体迁移流程

```
1. 创建 ROS2 工作空间
   ↓
2. 迁移消息定义（quadrotor_msgs, mavros_msgs）
   ↓
3. 迁移 Controller 包（C++ 代码）
   ↓
4. 迁移 Simulator 包（C++/CUDA 代码）
   ↓
5. 迁移 YOPO 包（Python 代码）
   ↓
6. 测试和调试
```

### 步骤 1: 创建 ROS2 工作空间

```bash
# 创建工作空间
mkdir -p ~/yopo_ros2_ws/src
cd ~/yopo_ros2_ws/src

# 从 ROS1 项目复制代码
cp -r /path/to/YOPO-YOPO-Simple/* .

# 重组目录结构
mkdir -p yopo_msgs
mkdir -p yopo_simulator
mkdir -p yopo_controller
mkdir -p yopo_planner
```

### 步骤 2: 创建包结构

ROS2 使用不同的包结构，每个包需要 `package.xml` 和 `CMakeLists.txt`（C++）或 `setup.py`（Python）。

---

## 包结构调整

### ROS1 vs ROS2 包结构

#### ROS1 结构:
```
Controller/
  src/
    so3_control/
      package.xml
      CMakeLists.txt
      src/
    so3_quadrotor_simulator/
      package.xml
      CMakeLists.txt
      src/
```

#### ROS2 结构:
```
yopo_ros2_ws/
  src/
    yopo_msgs/              # 消息定义
      msg/
      package.xml
      CMakeLists.txt
    yopo_controller/        # 控制器
      include/
      src/
      launch/
      config/
      package.xml
      CMakeLists.txt
    yopo_simulator/         # 仿真器
      include/
      src/
      launch/
      config/
      package.xml
      CMakeLists.txt
    yopo_planner/           # Python 规划器
      yopo_planner/
      resource/
      test/
      setup.py
      package.xml
      setup.cfg
```

---

## 编译系统迁移

### 1. CMakeLists.txt 迁移 (C++ 包)

#### ROS1 版本:
```cmake
cmake_minimum_required(VERSION 2.8.3)
project(so3_quadrotor_simulator)

find_package(catkin REQUIRED COMPONENTS
  roscpp
  geometry_msgs
  nav_msgs
  sensor_msgs
)

catkin_package()

add_executable(quadrotor_simulator_so3 src/quadrotor_simulator_so3.cpp)
target_link_libraries(quadrotor_simulator_so3 ${catkin_LIBRARIES})
```

#### ROS2 版本:
```cmake
cmake_minimum_required(VERSION 3.8)
project(yopo_controller)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# 查找依赖
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(yopo_msgs REQUIRED)

# 添加可执行文件
add_executable(quadrotor_simulator_so3 src/quadrotor_simulator_so3.cpp)
ament_target_dependencies(quadrotor_simulator_so3
  rclcpp
  geometry_msgs
  nav_msgs
  sensor_msgs
  yopo_msgs
)

# 安装
install(TARGETS
  quadrotor_simulator_so3
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY
  launch
  config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

### 2. package.xml 迁移

#### ROS1 版本:
```xml
<?xml version="1.0"?>
<package format="2">
  <name>so3_quadrotor_simulator</name>
  <version>0.0.1</version>
  <description>SO3 Quadrotor Simulator</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>MIT</license>

  <buildtool_depend>catkin</buildtool_depend>
  <depend>roscpp</depend>
  <depend>nav_msgs</depend>
</package>
```

#### ROS2 版本:
```xml
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
  <name>yopo_controller</name>
  <version>0.0.1</version>
  <description>YOPO Quadrotor Controller and Simulator</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>MIT</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <depend>rclcpp</depend>
  <depend>geometry_msgs</depend>
  <depend>nav_msgs</depend>
  <depend>sensor_msgs</depend>
  <depend>yopo_msgs</depend>

  <test_depend>ament_lint_auto</test_depend>
  <test_depend>ament_lint_common</test_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

### 3. CUDA CMakeLists.txt 迁移 (Simulator)

#### ROS2 CUDA 支持:
```cmake
cmake_minimum_required(VERSION 3.8)
project(yopo_simulator CUDA CXX)

# CUDA 设置
enable_language(CUDA)
set(CMAKE_CUDA_STANDARD 14)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

# 查找依赖
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(PCL REQUIRED)
find_package(OpenCV REQUIRED)
find_package(yaml-cpp REQUIRED)

# CUDA 架构
set(CMAKE_CUDA_ARCHITECTURES 86)  # RTX 30系列

# 添加 CUDA 可执行文件
add_executable(sensor_simulator_cuda
  src/sensor_simulator.cu
  src/sensor_simulator.cpp
  src/maps.cpp
  src/perlinnoise.cpp
  src/test_simulator_cuda.cpp
)

set_target_properties(sensor_simulator_cuda PROPERTIES
  CUDA_SEPARABLE_COMPILATION ON
)

ament_target_dependencies(sensor_simulator_cuda
  rclcpp
  sensor_msgs
  nav_msgs
  cv_bridge
)

target_link_libraries(sensor_simulator_cuda
  ${PCL_LIBRARIES}
  ${OpenCV_LIBS}
  yaml-cpp
)

# 安装
install(TARGETS
  sensor_simulator_cuda
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY
  launch config pointcloud
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

---

## Python代码迁移

### YOPO Planner 迁移 (test_yopo_ros.py)

#### 1. 节点初始化

**ROS1:**
```python
import rospy

rospy.init_node('yopo_net', anonymous=False)
rospy.spin()
```

**ROS2:**
```python
import rclpy
from rclpy.node import Node

class YopoNet(Node):
    def __init__(self, config, weight):
        super().__init__('yopo_net')
        # 初始化代码...
        
def main():
    rclpy.init()
    node = YopoNet(settings, weight)
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

#### 2. 发布者和订阅者

**ROS1:**
```python
from nav_msgs.msg import Odometry
from sensor_msgs.msg import Image

self.odom_sub = rospy.Subscriber('/sim/odom', Odometry, 
                                  self.callback_odometry, 
                                  queue_size=1, tcp_nodelay=True)
self.depth_sub = rospy.Subscriber('/depth_image', Image, 
                                   self.callback_depth, 
                                   queue_size=1, tcp_nodelay=True)
self.ctrl_pub = rospy.Publisher('/so3_control/pos_cmd', 
                                 PositionCommand, queue_size=1)
```

**ROS2:**
```python
from nav_msgs.msg import Odometry
from sensor_msgs.msg import Image
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy

# 创建 QoS 配置（类似 tcp_nodelay）
qos = QoSProfile(
    reliability=ReliabilityPolicy.BEST_EFFORT,
    history=HistoryPolicy.KEEP_LAST,
    depth=1
)

self.odom_sub = self.create_subscription(
    Odometry, '/sim/odom', self.callback_odometry, qos)
self.depth_sub = self.create_subscription(
    Image, '/depth_image', self.callback_depth, qos)
self.ctrl_pub = self.create_publisher(
    PositionCommand, '/so3_control/pos_cmd', 1)
```

#### 3. 定时器

**ROS1:**
```python
self.timer_ctrl = rospy.Timer(rospy.Duration(self.ctrl_dt), self.control_pub)
```

**ROS2:**
```python
self.timer_ctrl = self.create_timer(self.ctrl_dt, self.control_pub)
```

#### 4. 时间处理

**ROS1:**
```python
rospy.sleep(1.0)
now = rospy.Time.now()
```

**ROS2:**
```python
import time
time.sleep(1.0)
now = self.get_clock().now()
```

#### 5. 参数获取

**ROS1:**
```python
goal = rospy.get_param('~goal', [50, 0, 2])
```

**ROS2:**
```python
self.declare_parameter('goal', [50.0, 0.0, 2.0])
goal = self.get_parameter('goal').value
```

#### 6. 日志输出

**ROS1:**
```python
rospy.loginfo("YOPO Net Node Ready!")
rospy.logwarn("Warning message")
rospy.logerr("Error message")
```

**ROS2:**
```python
self.get_logger().info("YOPO Net Node Ready!")
self.get_logger().warn("Warning message")
self.get_logger().error("Error message")
```

### 完整的 ROS2 YOPO Planner 示例

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
from nav_msgs.msg import Odometry
from geometry_msgs.msg import PoseStamped
from sensor_msgs.msg import PointCloud2, Image
from cv_bridge import CvBridge
import numpy as np
import torch

class YopoNet(Node):
    def __init__(self, config, weight):
        super().__init__('yopo_net')
        
        # 声明参数
        self.declare_parameter('goal', [50.0, 0.0, 2.0])
        self.declare_parameter('velocity', 6.0)
        self.declare_parameter('pitch_angle_deg', 0.0)
        
        # 读取参数
        self.goal = np.array(self.get_parameter('goal').value)
        self.velocity = self.get_parameter('velocity').value
        
        # 加载网络
        self.policy = self.load_network(weight)
        
        # QoS 配置
        sensor_qos = QoSProfile(
            reliability=ReliabilityPolicy.BEST_EFFORT,
            history=HistoryPolicy.KEEP_LAST,
            depth=1
        )
        
        # 创建发布者
        self.ctrl_pub = self.create_publisher(
            PositionCommand, '/so3_control/pos_cmd', 10)
        self.best_traj_pub = self.create_publisher(
            PointCloud2, '/yopo_net/best_traj_visual', 10)
        
        # 创建订阅者
        self.odom_sub = self.create_subscription(
            Odometry, '/sim/odom', self.callback_odometry, sensor_qos)
        self.depth_sub = self.create_subscription(
            Image, '/depth_image', self.callback_depth, sensor_qos)
        self.goal_sub = self.create_subscription(
            PoseStamped, '/goal_pose', self.callback_set_goal, 10)
        
        # 创建定时器
        self.ctrl_dt = 0.02
        self.timer = self.create_timer(self.ctrl_dt, self.control_pub)
        
        self.get_logger().info("YOPO Net Node Ready!")
    
    def callback_odometry(self, msg):
        self.odom = msg
        # 处理里程计...
    
    def callback_depth(self, msg):
        # 处理深度图像...
        pass
    
    def callback_set_goal(self, msg):
        self.goal = np.array([
            msg.pose.position.x,
            msg.pose.position.y,
            msg.pose.position.z
        ])
        self.get_logger().info(f'New goal: {self.goal}')
    
    def control_pub(self):
        # 发布控制指令...
        pass

def main(args=None):
    rclpy.init(args=args)
    
    # 加载配置
    settings = {
        'goal': [50, 0, 2],
        'velocity': 6.0,
    }
    weight = 'path/to/weights.pth'
    
    node = YopoNet(settings, weight)
    
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Python 包结构 (setup.py)

```python
from setuptools import setup
import os
from glob import glob

package_name = 'yopo_planner'

setup(
    name=package_name,
    version='0.0.1',
    packages=[package_name],
    data_files=[
        ('share/ament_index/resource_index/packages',
            ['resource/' + package_name]),
        ('share/' + package_name, ['package.xml']),
        (os.path.join('share', package_name, 'launch'), glob('launch/*.py')),
        (os.path.join('share', package_name, 'config'), glob('config/*.yaml')),
    ],
    install_requires=['setuptools'],
    zip_safe=True,
    maintainer='your_name',
    maintainer_email='your_email@example.com',
    description='YOPO Planner for ROS2',
    license='MIT',
    tests_require=['pytest'],
    entry_points={
        'console_scripts': [
            'yopo_planner = yopo_planner.yopo_ros2:main',
        ],
    },
)
```

---

## C++代码迁移

### Controller 节点迁移

#### ROS1 代码:
```cpp
#include <ros/ros.h>
#include <nav_msgs/Odometry.h>

int main(int argc, char** argv) {
    ros::init(argc, argv, "quadrotor_simulator");
    ros::NodeHandle nh;
    
    ros::Publisher odom_pub = nh.advertise<nav_msgs::Odometry>("/sim/odom", 10);
    ros::Subscriber cmd_sub = nh.subscribe("/so3_cmd", 10, cmdCallback);
    
    ros::Rate rate(100);
    while(ros::ok()) {
        // 仿真循环
        ros::spinOnce();
        rate.sleep();
    }
    return 0;
}
```

#### ROS2 代码:
```cpp
#include <rclcpp/rclcpp.hpp>
#include <nav_msgs/msg/odometry.hpp>

class QuadrotorSimulator : public rclcpp::Node {
public:
    QuadrotorSimulator() : Node("quadrotor_simulator") {
        // 声明参数
        this->declare_parameter("init_x", 0.0);
        this->declare_parameter("init_y", 0.0);
        this->declare_parameter("init_z", 2.0);
        
        // 读取参数
        double init_x = this->get_parameter("init_x").as_double();
        
        // 创建发布者
        odom_pub_ = this->create_publisher<nav_msgs::msg::Odometry>(
            "/sim/odom", 10);
        
        // 创建订阅者
        cmd_sub_ = this->create_subscription<geometry_msgs::msg::Twist>(
            "/so3_cmd", 10,
            std::bind(&QuadrotorSimulator::cmdCallback, this, std::placeholders::_1));
        
        // 创建定时器
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(10),
            std::bind(&QuadrotorSimulator::timerCallback, this));
        
        RCLCPP_INFO(this->get_logger(), "Quadrotor Simulator Ready!");
    }

private:
    void cmdCallback(const geometry_msgs::msg::Twist::SharedPtr msg) {
        // 处理命令
    }
    
    void timerCallback() {
        // 定时器回调
        auto odom_msg = nav_msgs::msg::Odometry();
        odom_msg.header.stamp = this->now();
        odom_msg.header.frame_id = "world";
        odom_pub_->publish(odom_msg);
    }
    
    rclcpp::Publisher<nav_msgs::msg::Odometry>::SharedPtr odom_pub_;
    rclcpp::Subscription<geometry_msgs::msg::Twist>::SharedPtr cmd_sub_;
    rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char** argv) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<QuadrotorSimulator>();
    rclcpp::spin(node);
    rclcpp::shutdown();
    return 0;
}
```

### Simulator CUDA 代码迁移

主要变化在于 ROS 接口，CUDA 核心代码保持不变：

```cpp
#include <rclcpp/rclcpp.hpp>
#include <sensor_msgs/msg/image.hpp>
#include <sensor_msgs/msg/point_cloud2.hpp>
#include <nav_msgs/msg/odometry.hpp>
#include <cv_bridge/cv_bridge.h>

class SensorSimulator : public rclcpp::Node {
public:
    SensorSimulator() : Node("sensor_simulator_cuda") {
        // 加载配置文件
        loadConfig();
        
        // 创建发布者
        image_pub_ = this->create_publisher<sensor_msgs::msg::Image>(
            depth_topic_, 10);
        pcl_pub_ = this->create_publisher<sensor_msgs::msg::PointCloud2>(
            lidar_topic_, 10);
        map_pub_ = this->create_publisher<sensor_msgs::msg::PointCloud2>(
            "mock_map", rclcpp::QoS(1).transient_local());
        
        // 创建订阅者
        auto qos = rclcpp::QoS(rclcpp::KeepLast(1)).best_effort();
        odom_sub_ = this->create_subscription<nav_msgs::msg::Odometry>(
            odom_topic_, qos,
            std::bind(&SensorSimulator::odomCallback, this, std::placeholders::_1));
        
        // 创建定时器
        map_timer_ = this->create_wall_timer(
            std::chrono::seconds(1),
            std::bind(&SensorSimulator::publishMap, this));
        
        RCLCPP_INFO(this->get_logger(), "Sensor Simulator Ready!");
    }

private:
    void odomCallback(const nav_msgs::msg::Odometry::SharedPtr msg) {
        // 更新位姿
        updatePose(msg);
        
        // 渲染深度图
        if (render_depth_) {
            auto depth_image = renderDepth();
            auto img_msg = cv_bridge::CvImage(
                msg->header, "32FC1", depth_image).toImageMsg();
            image_pub_->publish(*img_msg);
        }
        
        // 渲染点云
        if (render_lidar_) {
            auto pcl_msg = renderLidar();
            pcl_msg.header = msg->header;
            pcl_pub_->publish(pcl_msg);
        }
    }
    
    void publishMap() {
        map_pub_->publish(map_msg_);
    }
    
    rclcpp::Publisher<sensor_msgs::msg::Image>::SharedPtr image_pub_;
    rclcpp::Publisher<sensor_msgs::msg::PointCloud2>::SharedPtr pcl_pub_;
    rclcpp::Publisher<sensor_msgs::msg::PointCloud2>::SharedPtr map_pub_;
    rclcpp::Subscription<nav_msgs::msg::Odometry>::SharedPtr odom_sub_;
    rclcpp::TimerBase::SharedPtr map_timer_;
};
```

---

## Launch文件迁移

### ROS1 Launch (XML)

```xml
<launch>
    <arg name="init_x" value="0.0"/>
    <arg name="init_y" value="0.0"/>
    <arg name="init_z" value="2.0"/>

    <node pkg="so3_quadrotor_simulator" type="quadrotor_simulator_so3" 
          name="quadrotor_simulator_so3" output="screen">
        <param name="rate/odom" value="100.0"/>
        <param name="simulator/init_state_x" value="$(arg init_x)"/>
        <remap from="~odom" to="/sim/odom"/>
    </node>
</launch>
```

### ROS2 Launch (Python)

```python
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    # 声明启动参数
    init_x_arg = DeclareLaunchArgument(
        'init_x', default_value='0.0',
        description='Initial X position'
    )
    init_y_arg = DeclareLaunchArgument(
        'init_y', default_value='0.0',
        description='Initial Y position'
    )
    init_z_arg = DeclareLaunchArgument(
        'init_z', default_value='2.0',
        description='Initial Z position'
    )
    
    # 定义节点
    quadrotor_simulator = Node(
        package='yopo_controller',
        executable='quadrotor_simulator_so3',
        name='quadrotor_simulator_so3',
        output='screen',
        parameters=[{
            'rate_odom': 100.0,
            'init_x': LaunchConfiguration('init_x'),
            'init_y': LaunchConfiguration('init_y'),
            'init_z': LaunchConfiguration('init_z'),
        }],
        remappings=[
            ('odom', '/sim/odom'),
        ]
    )
    
    controller_node = Node(
        package='yopo_controller',
        executable='network_control_node',
        name='network_controller_node',
        output='screen',
        parameters=[{
            'is_simulation': True,
            'use_disturbance_observer': True,
            'hover_thrust': 0.375,
        }],
        remappings=[
            ('odom', '/sim/odom'),
            ('position_cmd', '/so3_control/pos_cmd'),
        ]
    )
    
    return LaunchDescription([
        init_x_arg,
        init_y_arg,
        init_z_arg,
        quadrotor_simulator,
        controller_node,
    ])
```

### 完整系统启动文件

```python
# yopo_system.launch.py
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from ament_index_python.packages import get_package_share_directory
import os

def generate_launch_description():
    # 获取包路径
    controller_dir = get_package_share_directory('yopo_controller')
    simulator_dir = get_package_share_directory('yopo_simulator')
    planner_dir = get_package_share_directory('yopo_planner')
    
    # 控制器启动文件
    controller_launch = IncludeLaunchDescription(
        PythonLaunchDescriptionSource(
            os.path.join(controller_dir, 'launch', 'simulator_attitude_control.launch.py')
        )
    )
    
    # 仿真器节点
    simulator_node = Node(
        package='yopo_simulator',
        executable='sensor_simulator_cuda',
        name='sensor_simulator',
        output='screen',
        parameters=[
            os.path.join(simulator_dir, 'config', 'config.yaml')
        ]
    )
    
    # YOPO 规划器节点
    planner_node = Node(
        package='yopo_planner',
        executable='yopo_planner',
        name='yopo_planner',
        output='screen',
        parameters=[
            os.path.join(planner_dir, 'config', 'traj_opt.yaml')
        ]
    )
    
    # RViz
    rviz_config = os.path.join(planner_dir, 'config', 'yopo.rviz')
    rviz_node = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        arguments=['-d', rviz_config]
    )
    
    return LaunchDescription([
        controller_launch,
        simulator_node,
        planner_node,
        rviz_node,
    ])
```

---

## 消息和服务迁移

### 自定义消息定义

在 ROS2 中，消息需要在独立的包中定义。

#### 包结构:
```
yopo_msgs/
  msg/
    PositionCommand.msg
    SO3Command.msg
  CMakeLists.txt
  package.xml
```

#### CMakeLists.txt:
```cmake
cmake_minimum_required(VERSION 3.8)
project(yopo_msgs)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)

# 定义消息
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/PositionCommand.msg"
  "msg/SO3Command.msg"
  DEPENDENCIES std_msgs geometry_msgs
)

ament_package()
```

#### package.xml:
```xml
<?xml version="1.0"?>
<package format="3">
  <name>yopo_msgs</name>
  <version>0.0.1</version>
  <description>YOPO custom messages</description>
  <maintainer email="user@todo.todo">user</maintainer>
  <license>MIT</license>

  <buildtool_depend>ament_cmake</buildtool_depend>
  <build_depend>rosidl_default_generators</build_depend>
  <exec_depend>rosidl_default_runtime</exec_depend>
  <member_of_group>rosidl_interface_packages</member_of_group>

  <depend>std_msgs</depend>
  <depend>geometry_msgs</depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

#### PositionCommand.msg:
```
std_msgs/Header header
geometry_msgs/Point position
geometry_msgs/Vector3 velocity
geometry_msgs/Vector3 acceleration
float64 yaw
float64 yaw_dot
float64[3] kx
float64[3] kv
```

---

## 参数服务器迁移

### ROS1 参数加载 (rosparam)

```xml
<!-- ROS1 -->
<rosparam command="load" file="$(find package)/config/params.yaml"/>
```

### ROS2 参数加载

#### 方法 1: Launch 文件中加载

```python
# ROS2
from launch_ros.actions import Node

node = Node(
    package='yopo_planner',
    executable='yopo_planner',
    parameters=['/path/to/config/traj_opt.yaml']
)
```

#### 方法 2: 命令行加载

```bash
ros2 run yopo_planner yopo_planner --ros-args --params-file config/traj_opt.yaml
```

#### 方法 3: 代码中加载

```python
import yaml
from rclpy.node import Node

class MyNode(Node):
    def __init__(self):
        super().__init__('my_node')
        
        # 加载 YAML 文件
        with open('config.yaml', 'r') as f:
            config = yaml.safe_load(f)
        
        # 声明参数
        for key, value in config.items():
            self.declare_parameter(key, value)
```

---

## 迁移检查清单

### 代码迁移

- [ ] **包结构重组**
  - [ ] 创建 ROS2 工作空间
  - [ ] 重新组织包结构
  - [ ] 创建 package.xml (format 3)
  - [ ] 创建 CMakeLists.txt (ament_cmake)

- [ ] **消息定义**
  - [ ] 创建 yopo_msgs 包
  - [ ] 迁移 PositionCommand.msg
  - [ ] 迁移 SO3Command.msg
  - [ ] 更新消息依赖

- [ ] **C++ 代码**
  - [ ] 替换 `ros/ros.h` → `rclcpp/rclcpp.hpp`
  - [ ] 更新节点初始化
  - [ ] 更新发布者/订阅者
  - [ ] 更新定时器
  - [ ] 更新参数读取
  - [ ] 更新日志宏

- [ ] **Python 代码**
  - [ ] 替换 `rospy` → `rclpy`
  - [ ] 创建节点类（继承 Node）
  - [ ] 更新发布者/订阅者
  - [ ] 更新定时器
  - [ ] 更新参数读取
  - [ ] 创建 setup.py

- [ ] **Launch 文件**
  - [ ] XML → Python
  - [ ] 更新节点声明
  - [ ] 更新参数传递
  - [ ] 更新话题重映射

- [ ] **编译系统**
  - [ ] catkin → colcon
  - [ ] 更新 CMakeLists.txt
  - [ ] 更新 package.xml
  - [ ] 测试编译

### 功能测试

- [ ] **基础功能**
  - [ ] 节点正常启动
  - [ ] 话题正常发布/订阅
  - [ ] 参数正常读取
  - [ ] 服务正常调用

- [ ] **Controller**
  - [ ] 动力学仿真正常
  - [ ] SO3 控制器正常
  - [ ] /sim/odom 发布正常
  - [ ] 接收 /so3_control/pos_cmd 正常

- [ ] **Simulator**
  - [ ] CUDA 编译成功
  - [ ] 深度图渲染正常
  - [ ] 点云渲染正常
  - [ ] 性能满足要求

- [ ] **YOPO Planner**
  - [ ] 网络推理正常
  - [ ] 轨迹规划正常
  - [ ] 控制指令发布正常
  - [ ] 可视化正常

- [ ] **系统集成**
  - [ ] 三个模块协同工作
  - [ ] 闭环控制正常
  - [ ] RViz 可视化正常
  - [ ] 性能满足要求

### 性能优化

- [ ] **通信优化**
  - [ ] QoS 配置优化
  - [ ] 零拷贝传输（intra-process）
  - [ ] 话题优先级设置

- [ ] **实时性优化**
  - [ ] 使用 ROS2 实时执行器
  - [ ] 内存锁定
  - [ ] CPU 亲和性设置

---

## 编译和运行

### 编译

```bash
cd ~/yopo_ros2_ws
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release

# 仅编译特定包
colcon build --packages-select yopo_msgs yopo_controller

# 并行编译
colcon build --parallel-workers 4
```

### 运行

```bash
# Source 环境
source ~/yopo_ros2_ws/install/setup.bash

# 启动完整系统
ros2 launch yopo_planner yopo_system.launch.py

# 单独启动节点
ros2 run yopo_controller quadrotor_simulator_so3
ros2 run yopo_simulator sensor_simulator_cuda
ros2 run yopo_planner yopo_planner
```

### 常用命令

```bash
# 查看节点
ros2 node list

# 查看话题
ros2 topic list
ros2 topic echo /sim/odom
ros2 topic hz /depth_image

# 查看参数
ros2 param list
ros2 param get /yopo_planner velocity

# 设置参数
ros2 param set /yopo_planner velocity 8.0

# 录制数据
ros2 bag record -a

# 回放数据
ros2 bag play <bag_file>
```

---

## 已知问题和解决方案

### 1. Conda 环境冲突

**问题**: Conda 和 ROS2 的 Python 路径冲突

**解决**:
```bash
# 创建干净的环境
conda create --name yopo_ros2 python=3.10
conda activate yopo_ros2

# 确保使用系统的 ROS2 Python
export PYTHONPATH=/opt/ros/humble/lib/python3.10/site-packages:$PYTHONPATH
```

### 2. CUDA 版本不兼容

**问题**: PyTorch CUDA 版本与系统 CUDA 不一致

**解决**:
```bash
# 安装匹配的 PyTorch 版本
pip install torch==2.0.0+cu118 torchvision==0.15.0+cu118 \
    --index-url https://download.pytorch.org/whl/cu118
```

### 3. cv_bridge 找不到

**问题**: Python 找不到 cv_bridge

**解决**:
```bash
sudo apt install ros-humble-cv-bridge
```

### 4. 消息类型不匹配

**问题**: ROS1 和 ROS2 的标准消息有差异

**解决**: 检查消息定义，必要时创建转换函数

---

## 性能对比

| 指标 | ROS1 | ROS2 | 提升 |
|------|------|------|------|
| 话题延迟 | 2-5 ms | 0.5-2 ms | 2-4x |
| 吞吐量 | ~100 MB/s | ~500 MB/s | 5x |
| CPU 占用 | 基准 | -20% | 更低 |
| 内存占用 | 基准 | -10% | 更低 |

---

## 下一步

完成迁移后，建议：

1. **性能调优**: 使用 ROS2 的实时特性
2. **安全加固**: 启用 DDS 安全
3. **分布式部署**: 利用 ROS2 的 DDS 多播
4. **生命周期管理**: 使用托管节点
5. **组件化**: 使用 ROS2 组件实现零拷贝

---

## 参考资源

- [ROS2 官方文档](https://docs.ros.org/en/humble/)
- [ROS1 到 ROS2 迁移指南](https://docs.ros.org/en/humble/How-To-Guides/Migrating-from-ROS1.html)
- [Colcon 文档](https://colcon.readthedocs.io/)
- [rclcpp API](https://docs.ros2.org/latest/api/rclcpp/)
- [rclpy API](https://docs.ros2.org/latest/api/rclpy/)
