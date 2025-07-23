---
title: ROS2学习:使用launch启动脚本
date: 2025-07-23 16:26:23
tags: ros2
categories: ROS2学习
comments: true
---

## 使用launch启动多个节点
ROS2可用XML，YAML，PYTHON编写启动脚本，但PYTHON使用更加广泛，故主要总结PYTHON编写脚本的方法。

### 1. 启动命令

>> ros2 [package名(文件夹名)] [launch脚本名]

例子：

>> ros2 launch fishbot_discription gazebo_display.launch.py

### 2.脚本编写

launch脚本位置一般在功能包的launch目录下(需自己新建)

使用python库: *launch*、*launch_ros*

```
import launch
import launch_ros
```

launch工具在工作时会在文件中搜索*generate_launch_description*的函数来获取启动内容的描述，故第一步编写*generate_launch_description*函数

```
def generate_launch_description():
    #创建launch_ros.actions.Node类对象，包装各个节点
    action_node_patrol_client = launch_ros.actions.Node(
        package='demo_cpp_service',
        executable='patrol_client',
        output='log',
    )
    action_node_turtlesim_node = launch_ros.actions.Node(
        package='turtlesim',
        executable='turtlesim_node',
        output='both',
    )
    #创建描述对象launch.LaunchDescription
     launch_description = launch.LaunchDescription([
        action_node_patrol_client,
        action_node_turtlesim_node
    ])
    return launch_description
```

总结：首先对各个节点创建*launch_ros.actions.Node*类对象，对象传入参数及其含义：

| 参数 | 作用 |
|:-:|:-:|
|package|指定功能包名称|
|executable|功能包对应可执行文件(即创建的节点名)|
|output|输出位置，screen：屏幕，log：日志，both：兼有|

最后将节点启动对象合成数组传给*launch.LaunchDescription*并返回。


### 3.传递参数

对于需传入参数的节点，需用*launch.actions.DeclareLaunchArgument*进行声明，并在*launch_ros.actions.Node*类对象的*parameter*参数使用*launch.substitutions.LaunchConfiguration*传递节点的参数。

```
def generate_launch_description():
    #参数声明
    action_declare_arg_max_speed = launch.actions.DeclareLaunchArgument('launch_max_speed',default_value='2.0')
    #传入Node对象
    action_node_turtle_control = launch_ros.actions.Node(
        package='demo_cpp_service',
        executable='turtle_control',
        # 使用 launch 中参数launch_max_speed 值替换max_speed值
        parameters=[{'max_speed':launch.substitutions.LaunchConfiguration('launch_max_speed',default='2.0')}],
        output='screen',
    )
    #创建描述对象launch.LaunchDescription，需包含参数声明
     launch_description = launch.LaunchDescription([
        action_declare_arg_max_speed,
        action_node_turtle_control
    ])
    return launch_description
```

### 4.launch组件总结

+ **动作:** 主要模块*launch_ros.actions*、*launch.actions*
|常见动作|功能|
|:-:|:-:|
|IncludeLaunchDescription动作|包含其他launch文件|
|ExecuteProcess动作|执行指定命令行命令|
|LogInfo动作|输出日志|
```
import launch
import launch_ros
from ament_index_python.packages import get_package_share_directory
def generate_launch_description():
    #IncludeLaunchDescription动作包括其他launch文件
    #利用launch.launch_description_sources.PythonLaunchDescriptionSource对象传入路径
    #常与get_package_share_directory连用获取指定launch文件的路径
    action_include_launch = launch.actions.IncludeLaunchDescription(launch.launch_description_sources.PythonLaunchDescriptionSource([get_package_share_directory("turtlesiml"),"/launch","/multisim.launch.py"]))
    #ExecuteProcess动作执行命令行
    #注意各个命令间的空格
    action_executeprocess = launch.actions.ExecuteProcess(
        cmd=['ros2 ','service ','call ','/turtlesiml/spawn ','turtlesim/srv/Spawn ','{x:1, y:1}']
    )
    #利用LogInfo动作输出日志
    action_log_info = launch.actions.LogInfo(msg='使用launch来调用服务生成海龟')

    #利用定时器动作实现依次启动日志输出和进程执行,并使用GroupAction封装成组合
    action_group = launch.actions.GroupAction([
        launch.actions.TimerAction(period=2.0,actions=[action_log_info]),
        launch.actions.TimerAction(period=3.0,actions=[action_executeprocess]),
    ])
    #合成启动描述
    launch_description = launch.LaunchDescription([action_include_launch,action_group])
    return launch_description
```

+ **替换:**上述对节点传递参数即为使用launch的参数*替换*节点的参数值

+ **条件:**决定哪些动作启动和哪些动作不启动
主要库:launch.condition

例子：
```
import launch
import launch_ros
from launch.conditions import IfCondition

def generate_launch_description():
    #声明参数，是否创建新的海龟
    declare_spawn_turtle = launch.actions.DeclareLaunchArgument(
        'spawn_turtle', default_value='false',description='是否生成新的海龟'
    )
    spawn_turtle = launch.substitutions.LaunchConfiguration("spawn_turtle")

    action_turtlesim = launch_ros.actions.Node(
        package="turtlesim", executable= "turtlesim_node", output="screen"
    )

    #给日志输出和服务调用添加条件
    action_executeprocess = launch.actions.ExecuteProcess(
        condition=IfCondition(spawn_turtle),
        cmd=['ros2 ','service ','call ','/spawn ','turtlesim/srv/Spawn ','{x:1,y:1}']

    )
    action_log_info = launch.actions.LogInfo(IfCondition(spawn_turtle),msg="使用executeprocess来调用服务生成海龟")

    #利用定时器依次启动
    action_group = launch.actions.GroupAction([
        launch.actions.TimerAction(period=2.0,actions=[action_log_info])
        launch.actions.TimerAction(period=3.0,actions=
        [action_executeprocess])
    ])
    #合成启动描述并返回
    launch_description = launch.LaunchDescription([
        declare_spawn_turtle,
        action_turtlesim,
        action_group
    ])
    return launch_description