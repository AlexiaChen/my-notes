# ROS学习笔记


## 安装ROS

![[ROS的安装]]


## ROS基本使用

### 创建并初始化workspace

```bash
mkdir -p ~/catkin_ws/src 
cd ~/catkin_ws/src 
catkin_init_workspace
```

### 构建workspace中的工程

```bash
cd ~/catkin_ws/
catkin_make
source devel/setup.bash
echo $ROS_PACKAGE_PATH
```

编译排除workspace中的某个package

```bash
catkin_make -DCATKIN_BLACKLIST_PACKAGES="<package-name-A>;<package-name-B>"
```

### 创建功能包

类似于Rust中的cargo new出来的东西。一个工作空间可以有多个功能包。

```bash
catkin_create_pkg <package_name> [depend1] [depend2] [depend3]
```

创建的这个功能包的depend会反应在package.xml文件的配置里面。手动修改xml配置也是可以的。

在同一个工作空间下，不允许存在同名功能包，否则在编译时会报错。

### 工作空间的覆盖（Overlaying）

![[ROS工作空间截图.png]]

所有工作空间的路径会依次在ROS_PACKAGE_PATH环境变量中记录，当设置多个工作空间的环境变量后，新设置的路径在ROS_PACKAGE_PATH中会自动放置在最前端。在运行时，ROS会优先查找最前端的工作空间中是否存在指定的功能包，如果不存在，就顺序向后查找其他工作空间，直到最后一个工作空间为止。

查找功能包的工作空间路径:

```bash
sudo apt-get install ros-<ros-dist>-ros-tutorials
rospack find <package_name>
```

roscpp_tutorials是ros-tutorials中的一个功能包，此时该功能包存放于ROS的默认工作空间下。一般通过apt安装的都是存放在默认的工作空间下面。

```bash
 cd ~/catkin_ws/src
 git clone git://github.com/ros/ros_tutorials.git
cd ~/catkin_ws 
catkin_make 
source ./devel/setup.bash
```

以上的例子就是，直接clone其他repo的功能包，都自己的工作空间下编译，并设置环境变量。这样的一个效果就是，自己的工作空间的环境变量在$ROS_PACKAGE_PATH中，被设置到默认工作空间的路径之前了。这样就用本地自己自定义的功能包替换了apt安装的功能包了。

### ROS开发环境

[IDEs - ROS Wiki](http://wiki.ros.org/IDEs)


- RoboWare ，  感觉看起来像基于VSCode的UI开发的一个定制化IDE。[TonyRobotics/RoboWare-Studio: An IDE environment for ROS development. (github.com)](https://github.com/tonyrobotics/roboware-studio)
-  VSCode 安装ROS插件


### 乌龟(turtle)样例Demo学习

安装样例

```bash
sudo apt-get install ros-<ros-dist>-turtlesim
```

运行

```bash
roscore
rosrun turtlesim turtlesim_node # run sim node
rosrun turtlesim turtle_teleop_key # 运行键盘控制节点
```

![[turtle-demo.png]]

查看样例的计算图

```bash
rqt_graph
```

![[rqt_grahp_turtle.png]]

查看当前系统所有topic

```bash
$ rostopic list
/rosout  
/rosout_agg  
/turtle1/cmd_vel  
/turtle1/color_sensor  
/turtle1/pose
```

查看某一个topic，topic发布的数据结构类型是什么之类的

```bash
mathxh@MathxH:~$ rostopic info /turtle1/cmd_vel  
Type: geometry_msgs/Twist  
  
Publishers:  
* /teleop_turtle (http://MathxH:45293/)  
  
Subscribers:  
* /turtlesim (http://MathxH:39567/)
```



#### 发布订阅模型

 先创建功能包

```bash
cd ~/catkin_ws/src
catkin_create_pkg learning_communication std_msgs rospy roscpp
```
 publisher代码大概长这样:

![[publisher的cpp代码]]

subscriber的代码大概这样:

![[subscriber的cpp代码]]

上面的发布订阅模型的cmake大概是这样，大部分是生成的，我们只需要修修改改:

```cmake
include_directories(include ${catkin_INCLUDE_DIRS})

add_executable(talker src/talker.cpp)
target_link_libraries(talker ${catkin_LIBRARIES})
# add_dependencies(talker ${PROJECT_NAME}_generate_messages_cpp)

add_executable(listener src/listener.cpp)
target_link_libraries(listener ${catkin_LIBRARIES})
# add_dependencies(listener ${PROJECT_NAME}_generate_messages_cpp)
```

我们可以看到，ROS的一个功能包下面是可以生成多个binary的。

启动发布订阅的样例

```bash
roscore
rosrun <package-name> talker  # 启动publichser
rosrun <package-name> listener # 启动listener
```

```txt
mathxh@MathxH:~$ rostopic list  
/chatter  
/rosout  
/rosout_agg  
mathxh@MathxH:~$ rostopic info /chatter  
Type: std_msgs/String  
  
Publishers:  
* /talker (http://MathxH:39491/)  
  
Subscribers:  
* /listener (http://MathxH:39285/)  
  
  
mathxh@MathxH:~$ rqt_graph
```

![[graph-pubsub.png]]

发布订阅还可以发布自定义的消息的数据结构，也就是除了ROS标准库提供的消息数据结构以外的数据。

在以上例程中，chatter话题的消息类型是ROS中预定义的String。在ROS的元功能包common_msgs中提供了许多不同消息类型的功能包，如std_msgs（标准数据类型）、geometry_msgs（几何学数据类型）、sensor_msgs（传感器数据类型）等。这些功能包中提供了大量常用的消息类型，可以满足一般场景下的常用消息。

对于自定义消息结构

需要在功能包的msg文件夹(与src目录平级)下面加入 Person.msg：

```protobuf
string name 
uint8 sex 
uint8 age
```

因为是自定义消息，所以需要在package.xml中加入message相关的build和runtime

```xml
<build_depend>message_generation</build_depend>
<exec_depend>message_runtime</exec_depend> 
```

以前的tag叫run_depend, 现在才叫exc_depend。

然后在功能包的CMakeLists.txt中加入:

```cmake
find_package(catkin REQUIRED COMPONENTS
    geometry_msgs
    roscpp
    rospy
    std_msgs
    message_generation
)

catkin_package(
    ……
    CATKIN_DEPENDS geometry_msgs roscpp rospy std_msgs message_runtime
    ……)

add_message_files(
    FILES
    Person.msg
)
generate_messages(
    DEPENDENCIES
    std_msgs
)

```

然后编译，编译完成后, 可以通过以下命令查看自定义的消息

```bash
rosmsg show Person
```

publisher自定义消息的代码:

```cpp
#include <sstream>

#include "ros/ros.h"

#include "std_msgs/String.h"

#include "learning_communication/Person.h" // 实际上生成的.h文件是在工作空间devel文件夹的include文件夹下面

  

int main(int argc, char** argv)

{

    ros::init(argc, argv, "talker");

    ros::NodeHandle n;

    ros::Publisher chatter_pub = n.advertise<std_msgs::String>("chatter", 1000);

    ros::Publisher chatter_pub_cust = n.advertise<learning_communication::Person>("chatter_cust", 1000);

    ros::Rate loop_rate(10);

  

    int count = 0;

  

    while(ros::ok())

    {

        std_msgs::String msg;

  

        std::stringstream ss;

        ss << "hello world " << count;

        msg.data = ss.str();

  

        ROS_INFO("%s", msg.data.c_str());

  

        chatter_pub.publish(msg);

  

        learning_communication::Person person;

        person.name = "John";

        person.sex = 1;

        person.age = 12;

  

        chatter_pub_cust.publish(person);

  

        ROS_INFO("cust Person name %s", person.name.c_str());

  

        ros::spinOnce();

  

        loop_rate.sleep();

  

        ++count;

    }

}
```

subscriber的自定义消息代码:


```cpp
#include "ros/ros.h"

#include "std_msgs/String.h"

#include "learning_communication/Person.h"

  

void chatterCallback(const std_msgs::String::ConstPtr& msg)

{

    ROS_INFO("I heard: [%s]", msg->data.c_str());

}

  

void chatterCustCallback(const learning_communication::Person::ConstPtr& msg)

{

    ROS_INFO("I heard Person name: [%s]", msg->name.c_str());

    ROS_INFO("I heard Person sex: %d", msg->sex);

    ROS_INFO("I heard Person age: %d", msg->age);

}

  

int main(int argc, char **argv)

{

    ros::init(argc, argv, "listener");

    ros::NodeHandle n;

    ros::Subscriber sub = n.subscribe("chatter", 1000, chatterCallback);

    ros::Subscriber sub_cust = n.subscribe("chatter_cust", 1000, chatterCustCallback);

    ros::spin();

    return 0;

}
```

经过以上测试，成功。

![[pusub_cust_msg_type.png]]

```txt
mathxh@MathxH:~$ rostopic info /chatter_cust  
Type: learning_communication/Person  
  
Publishers:  
* /talker (http://MathxH:36987/)  
  
Subscribers:  
* /listener (http://MathxH:41311/)
```

[ROS中自定义消息的发布和订阅_大星小辰的博客-CSDN博客_ros发布自定义结构体](https://blog.csdn.net/qq_28306361/article/details/85243353)

#### Service同步调用模型

发布订阅是异步的，Service是同步的，可以理解为RPC调用，有调用就会有应答(response)。
也就是返回值这些。

在turtle样例的运行状态下，运行以下命令可以看到当前系统中的service接口列表:

```bash
mathxh@MathxH:~$ rosservice list  
/clear  
/kill  
/reset  
/rosout/get_loggers  
/rosout/set_logger_level  
/spawn  
/teleop_turtle/get_loggers  
/teleop_turtle/set_logger_level  
/turtle1/set_pen  
/turtle1/teleport_absolute  
/turtle1/teleport_relative  
/turtlesim/get_loggers  
/turtlesim/set_logger_level
mathxh@MathxH:~$ rosservice info /spawn  
Node: /turtlesim  
URI: rosrpc://MathxH:49453  
Type: turtlesim/Spawn  
Args: x y theta name
```

用命令行去调用显式出来的service的URL接口，比如`/spawn`就是这样一个接口。
```bash
 rosservice call /spawn "x: 8.0 y: 8.0 theta: 0.0 name: 'turtle2'"
```
以上命令会报错，因为双引号里面的字符串是yaml格式，所以需要输入双引号后按tab建补全，不然格式有问题。

在创建好turtle2后，就可以看到，新增了一些turtle2的topic,并且可以通过命令向特定的topic上publish数据:

```bash
mathxh@MathxH:~$ rostopic list  
/rosout  
/rosout_agg  
/turtle1/cmd_vel  
/turtle1/color_sensor  
/turtle1/pose  
/turtle2/cmd_vel  
/turtle2/color_sensor  
/turtle2/pose  
/turtle3/cmd_vel  
/turtle3/color_sensor  
/turtle3/pose
mathxh@MathxH:~$ rostopic pub /turtle2/cmd_vel geometry_msgs/Twist "linear:  
x: 4.0  
y: 3.0  
z: 0.0  
angular:  
x: 2.0  
y: 0.0  
z: 0.0"  
publishing and latching message. Press ctrl-C to terminate
```

以上通过该topic，直接就可以让turtle2运动了，只要设置了线速度和角速度。至于参数，也是类似rosservice那样，通过Tab按键补全。

对于自定义的Service数据，在srv文件夹底下添加AddTwoInts.srv：

```protobuf
int64 a 
int64 b 
--- 
int64 sum
```

很显然，是求2数之和，`---`  分割开的就是request和reponse的参数字段。package.xml中也是要添加message的依赖，参考前面发布订阅的自定义消息那节。

cmake中是:

```cmake
find_package(catkin REQUIRED COMPONENTS
    geometry_msgs
    roscpp
    rospy
    std_msgs
    message_generation
)
add_service_files(
    FILES
    AddTwoInts.srv
)

```

在功能包的src下面创建server.cpp 和client的cpp

server:

![[service中的server代码]]

client:

![[service中的client代码]]

相应cmakelists:

```cmake
add_executable(server src/server.cpp) 
target_link_libraries(server ${catkin_LIBRARIES}) 
add_dependencies(server ${PROJECT_NAME}_gencpp) 
add_executable(client src/client.cpp) 
target_link_libraries(client ${catkin_LIBRARIES}) 
add_dependencies(client ${PROJECT_NAME}_gencpp)
```

运行:

```bash
roscore
rosrun <package-name> server
rosrun <package-name> client 3 5
```

```bash
mathxh@MathxH:~/catkin_ws$ rosrun learning_communication client 2 3  
[ INFO] [1654648725.484207400]: Sum: 5  
mathxh@MathxH:~/catkin_ws$ rosservice list  
/add_two_ints_server/get_loggers  
/add_two_ints_server/set_logger_level  
/rosout/get_loggers  
/rosout/set_logger_level  
/two_sum  
mathxh@MathxH:~/catkin_ws$ rosservice info /two_sum  
Node: /add_two_ints_server  
URI: rosrpc://MathxH:39039  
Type: learning_communication/TwoSum  
Args: a b   
mathxh@MathxH:~/catkin_ws$ rosservice call /two_sum "a: 2  
b: 6"  
sum: 8
```


注意，还有一种不常用的通信机制，Action。[浅析ROS及其通讯机制（Topic/Service/Action）_光头明明的博客-CSDN博客_ros通信机制](https://blog.csdn.net/harrycomeon/article/details/106556154)

Action是异步RPC，Service是同步RPC可以这么理解。

#### ROS的命名空间

相比之前通过rqt_graph看当前系统的计算图，里面的节点，话题，Service统称为计算图源。其命名方式采用灵活的分层结构，便于在复杂的系统中集成和复用。类似于URL资源定位符号。


计算图源命名是ROS封装的一种重要机制。每个资源都定义在一个命名空间内，该命名空间内还可以创建更多资源。但是处于不同命名空间内的资源不仅可以在所处命名空间内使用，还可以在全局范围内访问，这与C++中的private有所不同。这种命名机制可以有效避免不同命名空间内的命名冲突。

计算图源的命名可以分为以下四种:

- 基础名称 例如  base
- 全局名称  例如 /global/name
- 相对名称 例如 relative/name
- 私有名称 例如 ~private/name

基础名称就是资源本身，上面的name都是基础名称，首字符是左斜杠（/）的名称是全局名称，由左斜杠分开一系列命名空间，示例中的命名空间为global。全局名称之所以称为全局，是因为它的解析度最高，可以在全局范围内直接访问。但是在系统中全局名称越少越好，因为过多的全局名称会直接影响功能包的可移植性。应该使用相对名称。

当前终端使用 `export ROS_NAMESPACE=default-namespace` 来设置默认名字空间。

![[namespace-parse.png]]


命名还可以重映射

所有ROS节点内的资源名称都可以在节点启动时进行重映射。ROS这一强大的特性甚至可以支持我们同时打开多个相同的节点，而不会发生命名冲突。

`name:=new_name`

采用以上的语法来映射。

![[remap-namespace.png]]


### launch启动文件

目前为止，我们启动ROS的节点都是一个终端开启一个节点，这样终端越来越多，比较麻烦，所以launch就是为了解决这个问题的。

```xml
<launch> 
  <node pkg="package-name" name="node-name" type="executable-name" output="screen"/> 
  <node pkg="package-name" name="node-name" type="executable-name"/> 
  </launch>
```

以上就是一个launch的示例。pkg定义节点所在的功能包名称，type定义节点的可执行文件名称，这两个属性等同于在终端中使用rosrun命令执行节点时的输入参数。name属性用来定义节点运行的名称，将覆盖节点中init(argc, argv, "node-name")赋予节点的名称。

- output这里是设置日志打印到终端屏幕，

- respawn="true"：复位属性，该节点停止时，会自动重启，默认为false。
- required="true"：必要节点，当该节点终止时，launch文件中的其他节点也被终止。
- ns="namespace"：命名空间，为节点内的相对名称添加命名空间前缀。
- args="arguments"：节点需要的输入参数。

以上这些属性，都是node标签的内部属性。

以下有一个较为复杂的案例:

```xml
<launch>
  <node pkg="mrobot_bringup" type="mrobot_bringup" name="mrobot_bringup" output= "screen" />
  <arg name="urdf_file" default="$(find xacro)/xacro --inorder '$(find mrobot_ description)/urdf/mrobot_with_rplidar.urdf.xacro'" />
  <param name="robot_description" command="$(arg urdf_file)" />
  <node name="joint_state_publisher" pkg="joint_state_publisher" type="joint_state_publisher" />
  <node pkg="robot_state_publisher" type="robot_state_publisher" name="state_publisher">
    <param name="publish_frequency" type="double" value="5.0" />
  </node>
  <node name="base2laser" pkg="tf" type="static_transform_publisher" args="0 0 0 0 0 0 1 /base_link /laser 50"/>
  <node pkg="robot_pose_ekf" type="robot_pose_ekf" name="robot_pose_ekf">
    <remap from="robot_pose_ekf/odom_combined" to="odom_combined"/>
    <param name="freq" value="10.0"/>
    <param name="sensor_timeout" value="1.0"/>
    <param name="publish_tf" value="true"/>
    <param name="odom_used" value="true"/>
    <param name="imu_used" value="false"/>
    <param name="vo_used" value="false"/>
    <param name="output_frame" value="odom"/>
  </node>
  <include file="$(find mrobot_bringup)/launch/rplidar.launch" />
</launch>

```

parameter是ROS系统运行中的参数，存储在参数服务器中。在launch文件中通过<param>元素加载parameter；launch文件执行后，parameter就加载到ROS的参数服务器上了。每个活跃的节点都可以通过ros：：param：：get()接口来获取parameter的值

argument是另外一个概念，类似于launch文件内部的局部变量，仅限于launch文件使用，便于launch文件的重构，与ROS节点内部的实现没有关系。


[roslaunch/XML - ROS Wiki](http://wiki.ros.org/roslaunch/XML)
[ROS创建launch文件_冰激凌啊的博客-CSDN博客_ros创建launch文件](https://blog.csdn.net/gyxx1998/article/details/122245125)
[ros中的launch文件_X uuuer.的博客-CSDN博客_roslaunch](https://blog.csdn.net/weixin_45868890/article/details/123615432)


launch里面也可以通过remap标签对计算源的一些service topic的URL进行重映射，以实现复用，比如某个节点发布的topic的URL与我们自己订阅的不同，我们就可以remap到我们的订阅话题的URL上以实现兼容。

launch中还可以嵌套，include其他的launch，以实现模块化和可维护性。


### TF坐标转换

[ROS中TF(坐标系转换)原理与使用 - 古月居 (guyuehome.com)](https://guyuehome.com/35673)

以上链接要与古月（胡春旭）写的《ROS机器人开发实践》的TF坐标变换那章结合一起看，效果最佳。

注意重点:

> 1.  source、target frame是在**进行坐标变换**时的概念，source是坐标变换的源坐标系，target是目标坐标系。这个时候，这个变换代表的是**坐标变换**。
> 2.  parent、child frame是在**描述坐标系变换**时的概念，parent是原坐标系，child是变换后的坐标系，这个时候这个变换**描述的是坐标系变换**，也是child坐标系在parent坐标系下的描述。
> 3.  a frame到b frame的坐标系变换（frame transform），也表示了b frame在a frame的描述，也代表了把一个点在b frame里坐标变换成在a frame里坐标的**坐标变换**。
> 4.  从parent到child的坐标系变换（frame transform）等同于把一个点从child坐标系向parent坐标系的**坐标变换**，等于child坐标系在parent frame坐标系的姿态描述。


以下是微信上找的相关文章:

-  https://mp.weixin.qq.com/s/urfVkw57E5EqU6-dD2uDLg
-  https://mp.weixin.qq.com/s/zRIjVh8IaMnyhBMQ-6RrqA
-  https://mp.weixin.qq.com/s/K2EMKAzjymLHaH5yuExGgw

其他的一些资源：

- [ROS：小车线速度、角速度、转弯半径之间的关系_-点灯-的博客-CSDN博客_转弯半径与速度](https://blog.csdn.net/qq_14977553/article/details/108523001)

- [TF坐标变换学习整理_尼古拉斯王的博客-CSDN博客_tf变换](https://blog.csdn.net/qq_41906863/article/details/103773620)

- [ROS之tf空间坐标变换浅析_小菜虎的博客-CSDN博客](https://blog.csdn.net/u014587147/article/details/76377024)
- [ROS之tf空间坐标变换浅析 （二）_小菜虎的博客-CSDN博客](https://blog.csdn.net/u014587147/article/details/76400880)

官方的TF的wiki [tf - ROS Wiki](http://wiki.ros.org/tf), 据说TF过时了，要用TF2 [tf2 - ROS Wiki](http://wiki.ros.org/tf2)

[tf/Debugging tools - ROS Wiki](http://wiki.ros.org/tf/Debugging%20tools)

打印TF树中所有坐标系(frame)的发布状态:

```bash
rosrun tf tf_monitor
```

查看两个坐标系之间的发布状态:

```bash
rosrun tf tf_monitor <source_frame> <target_target>
```

tf_echo工具的功能是查看指定坐标系之间的变换关系。

```bash
rosrun tf tf_echo <source_frame> <target_frame>
```

生成整个TF树的状况，结果为PDF文件

```bash
 rosrun tf view_frames
```

#### 通过turtle来学习TF

```bash
sudo apt install ros-<ros-dist>-turtle-tf
roslaunch turtle_tf turtle_tf_demo.launch
rosrun turtlesim turtle_teleop_key
```

乌龟仿真器打开后会出现两只小乌龟，并且下方的小乌龟会自动向中心位置的小乌龟移动。   你键盘控制到哪里，它就跟随到哪里。然而，我的noetic在WSL2的ubuntu20.04上没有两个小乌龟，不知道怎么回事。

```cpp
#include <ros/ros.h>
#include <tf/transform_broadcaster.h>
#include <turtlesim/Pose.h>
std::string turtle_name;
void poseCallback(const turtlesim::PoseConstPtr& msg)
{
  // TF广播器
  static tf::TransformBroadcaster br;
  // 根据乌龟当前的位姿, 设置相对于世界坐标系的坐标变换
  tf::Transform transform;
  transform.setOrigin( tf::Vector3(msg->x, msg->y, 0.0) );
  tf::Quaternion q;
  q.setRPY(0, 0, msg->theta);
  transform.setRotation(q);
  // 发布坐标变换
  br.sendTransform(tf::StampedTransform(transform, ros::Time::now(), "world", turtle_name));
}
int main(int argc, char** argv)
{
  // 初始化节点
  ros::init(argc, argv, "my_tf_broadcaster");
  if (argc != 2)
  {
    ROS_ERROR("need turtle name as argument"); 
    return -1;
  };
  turtle_name = argv[1];
  // 订阅乌龟的pose信息
  ros::NodeHandle node;
  ros::Subscriber sub = node.subscribe(turtle_name+"/pose", 10, &poseCallback);
  ros::spin();
  return 0;
};

```


代码中的Quaternion 类型就是四元数了，setRPY就是设置欧拉角了，R是roll意为翻滚，p为pitch意为俯仰，y为yaw意为航向。

具体解释看[简单理解3D系统中pitch/yaw/roll 的基本概念 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/450246573)

[欧拉角介绍Yaw,Pitch,Roll (360doc.com)](http://www.360doc.com/content/17/1103/17/2036337_700628160.shtml)

欧拉角表示一个三维的旋转，看文章，roll是绕Z轴旋转，pitch是绕X轴，yaw是绕Y轴旋转，欧拉角就是这三个RPY的组合了，也就是说，任意的三维旋转都可以通过这三个旋转先后顺序旋转得到。


但是好像在ROS中的这个setRPY的说明上，

```cpp
  /**@brief Set the quaternion using fixed axis RPY

   * @param roll Angle around X

   * @param pitch Angle around Y

   * @param yaw Angle around Z*/
```

跟以上链接中提到的不一样，roll居然是X，Pitch是Y，yaw是Z。估计是坐标系的建立方向不一样。然后Direct3D也不一样，欧拉角的XYZ的旋转顺序是不一样的。因为以上代码是二维平面的turtle，所以没有roll pitch，这个光大脑想象就可以想出来，只有yaw这个航向。

欧拉角一般只是为了方便给人用，方便理解，表示一个3维物体的朝向，但是最终到计算机图形里，就是线性代数，这个旋转也可以用矩阵表示，三个旋转分量，就是三次矩阵变换，数学上就是三个矩阵相乘(三个3\*3的矩阵)，如果理解矩阵是线性变换就好理解了，也就是做了三次变换。

注意，先进行那个轴的旋转对结果是有影响的。所以，一般引擎(平台)都会规定自己的旋转顺序。比如ROS，D3D的肯定不一样。说句大白话就是：同一旋转，欧拉角表示不唯一。所以在不同平台下，你得注意一下XYZ的旋转顺序，也就是RPY的顺序，不能搞错了。这里的setRPY是先roll后pitch再yaw。

同时还要注意，一般pitch这个值不能超过90度，这与万向锁有关: [欧拉角与万向锁的理解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/346718090)

[欧拉角与万向锁—深度解读万向锁_euphorias的博客-CSDN博客_欧拉角万向锁](https://blog.csdn.net/euphorias/article/details/123612227)

pitch限制在 [-90, 90] roll和yaw [-180, 180] 以内。其实这里也可以想象出来，比如控制roll的是Y轴，pitch是X轴，yaw是Z轴，如果pitch了90度，那么朝向就与Z轴一致了(不管方向)，那么roll的时候，就是原本旋转Y轴的roll，这时候旋转Z轴也可以了，这样就错了，两个轴重合了，失去了一个轴的自由度，相当于一个轴就失效了，被“锁”了，可以这么理解。


再回顾下上面的TF代码表示rotation哪里，四元数类型的变量setRPY，也就是相当于设置一个欧拉角，欧拉角是表示三维旋转，其实四元数也是表示三维旋转，欧拉角和四元数是可以互相转换的，还有旋转矩阵也是，它们的之间都可以互转:

![[3d-rotation.png]]



[三维旋转：欧拉角、四元数、旋转矩阵、轴角之间的转换 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/45404840)

[四元数和旋转(Quaternion & rotation) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/78987582)

[四元数表示旋转的理解_小方爱自律的博客-CSDN博客_四元数旋转](https://blog.csdn.net/qq_28773183/article/details/80083607)

[如何形象地理解四元数？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/23005815)

tf listener:

```cpp
#include <ros/ros.h>
#include <tf/transform_listener.h>
#include <geometry_msgs/Twist.h>
#include <turtlesim/Spawn.h>
int main(int argc, char** argv)
{
  // 初始化节点
  ros::init(argc, argv, "my_tf_listener");
  ros::NodeHandle node;
  // 通过服务调用, 产生第二只乌龟turtle2
  ros::service::waitForService("spawn");
  ros::ServiceClient add_turtle =
  node.serviceClient<turtlesim::Spawn>("spawn");
  turtlesim::Spawn srv;
  add_turtle.call(srv);
  // 定义turtle2的速度控制发布器
  ros::Publisher turtle_vel =
  node.advertise<geometry_msgs::Twist>("turtle2/cmd_vel", 10);
  // TF监听器
  tf::TransformListener listener;
  ros::Rate rate(10.0);
  while (node.ok())
  {
    tf::StampedTransform transform;
    try
    {
      // 查找turtle2与turtle1的坐标变换
      listener.waitForTransform("/turtle2", "/turtle1", ros::Time(0), ros:: Duration(3.0));
      listener.lookupTransform("/turtle2", "/turtle1", ros::Time(0), transform);
    }
    catch (tf::TransformException &ex) 
    {
      ROS_ERROR("%s",ex.what());
      ros::Duration(1.0).sleep();
      continue;
    }
    // 根据turtle1和turtle2之间的坐标变换, 计算turtle2需要运动的线速度和角速度
    // 并发布速度控制指令, 使turtle2向turtle1移动
    geometry_msgs::Twist vel_msg;
    vel_msg.angular.z = 4.0 * atan2(transform.getOrigin().y(),
                                    transform.getOrigin().x());
    vel_msg.linear.x = 0.5 * sqrt(pow(transform.getOrigin().x(), 2) +
                                  pow(transform.getOrigin().y(), 2));
    turtle_vel.publish(vel_msg);
    rate.sleep();
  }
  return 0;
};

```

```cmake
find_package(catkin REQUIRED COMPONENTS

  roscpp

  rospy

  std_msgs

  tf

)

# 这个cakin_package不能少

catkin_package(

    INCLUDE_DIRS include

    LIBRARIES learning_tf

    CATKIN_DEPENDS roscpp rospy std_msgs

    DEPENDS system_lib

)

add_executable(turtle_tf_broadcaster src/turtle_tf_broadcaster.cpp)

add_executable(turtle_tf_listener src/turtle_tf_listener.cpp)

 target_link_libraries(turtle_tf_broadcaster

   ${catkin_LIBRARIES}

 )

 target_link_libraries(turtle_tf_listener

   ${catkin_LIBRARIES}

 )

```

```xml
<launch>
  <!-- 海龟仿真器 -->
  <node pkg="turtlesim" type="turtlesim_node" name="sim"/>
  <!-- 键盘控制 -->
  <node pkg="turtlesim" type="turtle_teleop_key" name="teleop" output="screen"/>
  <!-- 两只海龟的TF广播 -->
  <node pkg="learning_tf" type="turtle_tf_broadcaster"
        args="/turtle1" name="turtle1_tf_broadcaster" />
  <node pkg="learning_tf" type="turtle_tf_broadcaster"
        args="/turtle2" name="turtle2_tf_broadcaster" />
  <!-- 监听TF广播, 并且控制turtle2移动 -->
  <node pkg="learning_tf" type="turtle_tf_listener"
        name="listener" />
</launch>

```

最终实现了小乌龟跟随运动，传入到poseCallback这个函数里面的小乌龟pose值是X Y, Z， 只是目前是二维平面，所以只需要X Y值。  以小乌龟为原点，X是前方，Y是左方，Z是上方，也就是朝屏幕外面，这里使用右手定义坐标。





#ros : ROS相关
#AI : 人工智能相关













