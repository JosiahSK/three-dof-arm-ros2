# 3-DOF Robotic Arm — Beginner Setup Guide (No Git Clone)

This guide builds the **3-DOF Robotic Arm with Gripper** project from scratch on **Ubuntu 22.04**, using **ROS 2 Humble** and **Gazebo Fortress** — without cloning the repo. Every command is explained *before* you run it, and related commands are grouped into single copy-paste blocks so you're never jumping around.

> 🚀 **In a hurry?** Skip straight to [Option 0: One-Shot Script](#option-0-one-shot-script-fastest) and paste 3 lines instead of following the whole guide.

> ℹ️ **Why Fortress, not Harmonic?** ROS 2 Humble's officially supported Gazebo version (via `apt`) is **Gazebo Fortress**, not Harmonic — Harmonic is what ROS 2 Jazzy pairs with. If a tutorial elsewhere shows "Harmonic" commands, that's for Jazzy, not this guide. The plugin names used in the code below (`gz_ros2_control`, `libgz_ros2_control-system.so`) are identical either way, so nothing else in the project changes.

## Table of Contents
1. [Option 0: One-Shot Script (fastest)](#option-0-one-shot-script-fastest)
2. [How This Guide Works](#how-this-guide-works)
3. [Step 1 — Install Everything](#step-1--install-everything)
4. [Step 2 — Create the Workspace Folders](#step-2--create-the-workspace-folders)
5. [Step 3 — Build the `my_robot_description` Package](#step-3--build-the-my_robot_description-package)
6. [Step 4 — Build the `my_robot_bringup` Package](#step-4--build-the-my_robot_bringup-package)
7. [Step 5 — Compile the Workspace](#step-5--compile-the-workspace)
8. [Step 6 — Run It](#step-6--run-it)
9. [Understanding the Robot Model](#understanding-the-robot-model)
10. [Troubleshooting](#troubleshooting)

---

## Option 0: One-Shot Script (fastest)

If you just want the whole thing built with minimal fuss, download and run `setup_arm.sh` — it does every step below automatically, with comments inside explaining each part as it runs.

```bash
curl -O https://raw.githubusercontent.com/JosiahSK/three-dof-arm-ros2/humble/setup_arm.sh
chmod +x setup_arm.sh
./setup_arm.sh
```

That's it — skip to [Step 6 — Run It](#step-6--run-it) once it finishes. If you'd rather understand *what* you're building as you go, keep reading from Step 1 instead.

---

## How This Guide Works

- Every code block below is **safe to paste as one chunk** — you don't need to run lines individually.
- Blocks that start with `cat > path/to/file << 'EOF' ... EOF` **create a file** — everything between the two `EOF` markers becomes that file's content. This is how we write project files without cloning or using a text editor.
- Read the paragraph *above* each block before pasting — it tells you what you're about to create and why.

---

## Step 1 — Install Everything

We need five groups of software:
| Group | What it's for |
|---|---|
| `ros-humble-desktop` | The ROS 2 Humble framework itself |
| `ros-*-ros-gz*` | Lets ROS 2 talk to the Gazebo Fortress physics simulator (Humble’s default) |
| `ros-*-ros2-control*` | The framework that actually drives the arm's joints |
| `ros-*-xacro`, `joint-state-publisher-gui` | Robot-description tooling + manual joint sliders |
| `colcon`, `rosdep` | The tools used to build and dependency-check the workspace |

Paste this whole block — it installs everything and only needs to run once:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y ros-humble-desktop

sudo apt install -y \
  ros-humble-ros-gz \
  ros-humble-ros-gz-sim \
  ros-humble-ros-gz-bridge \
  ros-humble-gz-ros2-control \
  ros-humble-ros2-control \
  ros-humble-ros2-controllers \
  ros-humble-joint-state-broadcaster \
  ros-humble-joint-trajectory-controller \
  ros-humble-xacro \
  ros-humble-joint-state-publisher-gui \
  python3-colcon-common-extensions \
  python3-rosdep

# rosdep only needs to be initialized once per machine — safe to ignore
# a "already initialized" message here.
sudo rosdep init 2>/dev/null || true
rosdep update

# Load ROS 2 commands into this terminal (do this in every NEW terminal too).
source /opt/ros/humble/setup.bash
```

---

## Step 2 — Create the Workspace Folders

A ROS 2 **workspace** is just a folder (`~/ros2_ws`) containing **packages**. This project has two packages:
- `my_robot_description` — the robot's physical shape (URDF/xacro) + controller settings
- `my_robot_bringup` — the launch files that start RViz or Gazebo

```bash
mkdir -p ~/ros2_ws/src/my_robot_description/urdf
mkdir -p ~/ros2_ws/src/my_robot_description/config
mkdir -p ~/ros2_ws/src/my_robot_description/meshes
mkdir -p ~/ros2_ws/src/my_robot_bringup/launch
mkdir -p ~/ros2_ws/src/my_robot_bringup/rviz
mkdir -p ~/ros2_ws/src/my_robot_bringup/scripts
```

---

## Step 3 — Build the `my_robot_description` Package

This single block creates all 7 files for this package at once: the package metadata, build instructions, the robot's 3D shape, its motor/controller wiring, its Gazebo plugin hookup, and its controller settings.

```bash
# package.xml — tells ROS 2 this package's name and what it depends on.
cat > ~/ros2_ws/src/my_robot_description/package.xml << 'EOF'
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>my_robot_description</name>
    <version>0.0.0</version>
    <description>URDF description package for a 3-DOF robotic arm with gripper</description>
    <maintainer email="user@todo.todo">user</maintainer>
    <license>MIT</license>
    <buildtool_depend>ament_cmake</buildtool_depend>
    <exec_depend>robot_state_publisher</exec_depend>
    <exec_depend>joint_state_publisher_gui</exec_depend>
    <exec_depend>xacro</exec_depend>
    <exec_depend>ros2_control</exec_depend>
    <exec_depend>ros2_controllers</exec_depend>
    <exec_depend>gz_ros2_control</exec_depend>
    <exec_depend>ros_gz_bridge</exec_depend>
    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
EOF

# CMakeLists.txt — tells the build tool to copy urdf/ and config/ into the
# installed package so launch files can find them later.
cat > ~/ros2_ws/src/my_robot_description/CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.8)
project(my_robot_description)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()
find_package(ament_cmake REQUIRED)
install(
  DIRECTORY urdf config
  DESTINATION share/${PROJECT_NAME}
)
ament_package()
EOF

# arm.urdf.xacro — the "table of contents" file that pulls the next three
# files together into one robot.
cat > ~/ros2_ws/src/my_robot_description/urdf/arm.urdf.xacro << 'EOF'
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">
  <xacro:include filename="$(find my_robot_description)/urdf/arm_core.xacro" />
  <xacro:include filename="$(find my_robot_description)/urdf/arm.ros2_control.xacro" />
  <xacro:include filename="$(find my_robot_description)/urdf/arm.gazebo.xacro" />
</robot>
EOF

# arm_core.xacro — the actual 3D shape: 5 rigid links joined by 3 rotating
# joints (like shoulder, elbow, wrist) plus 1 sliding gripper joint.
cat > ~/ros2_ws/src/my_robot_description/urdf/arm_core.xacro << 'EOF'
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">

  <material name="blue"><color rgba="0.2 0.2 0.8 1"/></material>
  <material name="gray"><color rgba="0.5 0.5 0.5 1"/></material>
  <material name="red"><color rgba="0.8 0.2 0.2 1"/></material>

  <link name="base_link">
    <visual><geometry><cylinder radius="0.1" length="0.05"/></geometry><material name="gray"/></visual>
    <collision><geometry><cylinder radius="0.1" length="0.05"/></geometry></collision>
    <inertial><mass value="2.0"/><origin xyz="0 0 0"/><inertia ixx="0.0054" ixy="0.0" ixz="0.0" iyy="0.0054" iyz="0.0" izz="0.01"/></inertial>
  </link>

  <joint name="joint1" type="revolute">
    <parent link="base_link"/><child link="link1"/>
    <origin xyz="0 0 0.025" rpy="0 0 0"/><axis xyz="0 0 1"/>
    <limit lower="-3.14" upper="3.14" effort="100.0" velocity="1.0"/>
  </joint>

  <link name="link1">
    <visual><origin xyz="0 0 0.15" rpy="0 0 0"/><geometry><cylinder radius="0.05" length="0.3"/></geometry><material name="blue"/></visual>
    <collision><origin xyz="0 0 0.15" rpy="0 0 0"/><geometry><cylinder radius="0.05" length="0.3"/></geometry></collision>
    <inertial><mass value="1.0"/><origin xyz="0 0 0.15" rpy="0 0 0"/><inertia ixx="0.008" ixy="0.0" ixz="0.0" iyy="0.008" iyz="0.0" izz="0.001"/></inertial>
  </link>

  <joint name="joint2" type="revolute">
    <parent link="link1"/><child link="link2"/>
    <origin xyz="0 0 0.3" rpy="0 0 0"/><axis xyz="0 1 0"/>
    <limit lower="-1.57" upper="1.57" effort="100.0" velocity="1.0"/>
  </joint>

  <link name="link2">
    <visual><origin xyz="0 0 0.15" rpy="0 0 0"/><geometry><cylinder radius="0.04" length="0.3"/></geometry><material name="blue"/></visual>
    <collision><origin xyz="0 0 0.15" rpy="0 0 0"/><geometry><cylinder radius="0.04" length="0.3"/></geometry></collision>
    <inertial><mass value="0.8"/><origin xyz="0 0 0.15" rpy="0 0 0"/><inertia ixx="0.006" ixy="0.0" ixz="0.0" iyy="0.006" iyz="0.0" izz="0.0006"/></inertial>
  </link>

  <joint name="joint3" type="revolute">
    <parent link="link2"/><child link="link3"/>
    <origin xyz="0 0 0.3" rpy="0 0 0"/><axis xyz="0 1 0"/>
    <limit lower="-1.57" upper="1.57" effort="100.0" velocity="1.0"/>
  </joint>

  <link name="link3">
    <visual><origin xyz="0 0 0.05" rpy="0 0 0"/><geometry><box size="0.05 0.05 0.1"/></geometry><material name="gray"/></visual>
    <collision><origin xyz="0 0 0.05" rpy="0 0 0"/><geometry><box size="0.05 0.05 0.1"/></geometry></collision>
    <inertial><mass value="0.5"/><origin xyz="0 0 0.05" rpy="0 0 0"/><inertia ixx="0.0005" ixy="0.0" ixz="0.0" iyy="0.0005" iyz="0.0" izz="0.0002"/></inertial>
  </link>

  <joint name="gripper_joint" type="prismatic">
    <parent link="link3"/><child link="gripper_link"/>
    <origin xyz="0 0 0.1" rpy="0 0 0"/><axis xyz="1 0 0"/>
    <limit lower="-0.02" upper="0.02" effort="50.0" velocity="0.5"/>
  </joint>

  <link name="gripper_link">
    <visual><origin xyz="0 0 0.025" rpy="0 0 0"/><geometry><box size="0.08 0.02 0.05"/></geometry><material name="red"/></visual>
    <collision><origin xyz="0 0 0.025" rpy="0 0 0"/><geometry><box size="0.08 0.02 0.05"/></geometry></collision>
    <inertial><mass value="0.2"/><origin xyz="0 0 0.025" rpy="0 0 0"/><inertia ixx="0.00005" ixy="0.0" ixz="0.0" iyy="0.0001" iyz="0.0" izz="0.0001"/></inertial>
  </link>

</robot>
EOF

# arm.ros2_control.xacro — tells ROS 2 HOW to command each joint (position
# control) and what feedback (position/velocity) it reads back.
cat > ~/ros2_ws/src/my_robot_description/urdf/arm.ros2_control.xacro << 'EOF'
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
  <ros2_control name="GazeboSimSystem" type="system">
    <hardware>
      <plugin>gz_ros2_control/GazeboSimSystem</plugin>
    </hardware>
    <joint name="joint1">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
    <joint name="joint2">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
    <joint name="joint3">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
    <joint name="gripper_joint">
      <command_interface name="position"/>
      <state_interface name="position"/>
      <state_interface name="velocity"/>
    </joint>
  </ros2_control>
</robot>
EOF

# arm.gazebo.xacro — plugs the robot into Gazebo Fortress and gives each
# part a color so it doesn't render as plain white.
cat > ~/ros2_ws/src/my_robot_description/urdf/arm.gazebo.xacro << 'EOF'
<robot xmlns:xacro="http://www.ros.org/wiki/xacro">
  <gazebo>
    <plugin name="gz_ros2_control::GazeboSimROS2ControlPlugin" filename="libgz_ros2_control-system.so">
      <parameters>$(find my_robot_description)/config/controller.yaml</parameters>
    </plugin>
  </gazebo>
  <gazebo reference="base_link"><material>Gazebo/Gray</material></gazebo>
  <gazebo reference="link1"><material>Gazebo/Blue</material></gazebo>
  <gazebo reference="link2"><material>Gazebo/Blue</material></gazebo>
  <gazebo reference="link3"><material>Gazebo/Gray</material></gazebo>
  <gazebo reference="gripper_link"><material>Gazebo/Red</material></gazebo>
</robot>
EOF

# controller.yaml — configures which controllers run and which joints
# they manage: a state broadcaster (reports positions) and a trajectory
# controller (moves joint1/2/3 + the gripper).
cat > ~/ros2_ws/src/my_robot_description/config/controller.yaml << 'EOF'
controller_manager:
  ros__parameters:
    update_rate: 10
    joint_state_broadcaster:
      type: joint_state_broadcaster/JointStateBroadcaster
    arm_controller:
      type: joint_trajectory_controller/JointTrajectoryController

arm_controller:
  ros__parameters:
    joints:
      - joint1
      - joint2
      - joint3
      - gripper_joint
    command_interfaces:
      - position
    state_interfaces:
      - position
      - velocity
EOF

# Placeholder note — this project uses simple built-in shapes, not custom
# 3D models, so this folder stays empty for now.
cat > ~/ros2_ws/src/my_robot_description/meshes/README.md << 'EOF'
# Meshes
Place robot mesh files here (STL, DAE, OBJ) if you add custom 3D models later.
EOF
```

---

## Step 4 — Build the `my_robot_bringup` Package

This block creates the 6 files that start everything up: package metadata, build instructions, two launch modes (visualize-only vs. full simulation), a saved RViz camera view, and a test script that moves the arm automatically.

```bash
# package.xml — depends on the description package above, plus RViz and
# Gazebo bridging tools.
cat > ~/ros2_ws/src/my_robot_bringup/package.xml << 'EOF'
<?xml version="1.0"?>
<?xml-model href="http://download.ros.org/schema/package_format3.xsd" schematypens="http://www.w3.org/2001/XMLSchema"?>
<package format="3">
    <name>my_robot_bringup</name>
    <version>0.0.0</version>
    <description>Bringup launch files for the 3-DOF robotic arm</description>
    <maintainer email="user@todo.todo">user</maintainer>
    <license>MIT</license>
    <buildtool_depend>ament_cmake</buildtool_depend>
    <exec_depend>my_robot_description</exec_depend>
    <exec_depend>robot_state_publisher</exec_depend>
    <exec_depend>joint_state_publisher_gui</exec_depend>
    <exec_depend>rviz2</exec_depend>
    <exec_depend>ros_gz_sim</exec_depend>
    <exec_depend>ros_gz_bridge</exec_depend>
    <export>
        <build_type>ament_cmake</build_type>
    </export>
</package>
EOF

cat > ~/ros2_ws/src/my_robot_bringup/CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.8)
project(my_robot_bringup)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()
find_package(ament_cmake REQUIRED)
install(
  DIRECTORY launch
  DESTINATION share/${PROJECT_NAME}
)
ament_package()
EOF

# display.launch.py — LIGHTWEIGHT MODE: no physics, no Gazebo. Just RViz
# plus manual joint sliders. Use this first to sanity-check the model.
cat > ~/ros2_ws/src/my_robot_bringup/launch/display.launch.py << 'EOF'
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.substitutions import Command, LaunchConfiguration
from launch_ros.actions import Node

def generate_launch_description():
    pkg_description = get_package_share_directory('my_robot_description')
    pkg_bringup = get_package_share_directory('my_robot_bringup')

    default_model_path = os.path.join(pkg_description, 'urdf', 'arm.urdf.xacro')
    default_rviz_path = os.path.join(pkg_bringup, 'rviz', 'display.rviz')

    robot_state_publisher_node = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        parameters=[{
            'robot_description': Command(['xacro ', LaunchConfiguration('model')])
        }]
    )

    joint_state_publisher_gui_node = Node(
        package='joint_state_publisher_gui',
        executable='joint_state_publisher_gui',
        name='joint_state_publisher_gui'
    )

    rviz_node = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', LaunchConfiguration('rvizconfig')],
    )

    return LaunchDescription([
        DeclareLaunchArgument(name='model', default_value=default_model_path,
                               description='Absolute path to robot URDF/xacro file'),
        DeclareLaunchArgument(name='rvizconfig', default_value=default_rviz_path,
                               description='Absolute path to RViz config file'),
        robot_state_publisher_node,
        joint_state_publisher_gui_node,
        rviz_node
    ])
EOF

# gazebo.launch.py — FULL SIMULATION MODE: boots Gazebo Fortress, spawns
# the robot into an empty world, and auto-starts every controller.
cat > ~/ros2_ws/src/my_robot_bringup/launch/gazebo.launch.py << 'EOF'
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import Command, FindExecutable, PathJoinSubstitution
from launch_ros.actions import Node
from launch_ros.substitutions import FindPackageShare

def generate_launch_description():
    pkg_share = get_package_share_directory('my_robot_description')

    robot_description_content = Command([
        PathJoinSubstitution([FindExecutable(name='xacro')]),
        ' ',
        PathJoinSubstitution([pkg_share, 'urdf', 'arm.urdf.xacro']),
    ])
    robot_description = {'robot_description': robot_description_content}

    robot_state_publisher_node = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        output='both',
        parameters=[robot_description, {'use_sim_time': True}]
    )

    gazebo = IncludeLaunchDescription(
        PythonLaunchDescriptionSource([
            PathJoinSubstitution([FindPackageShare('ros_gz_sim'), 'launch', 'gz_sim.launch.py'])
        ]),
        launch_arguments={'gz_args': '-r empty.sdf'}.items()
    )

    gz_spawn_entity = Node(
        package='ros_gz_sim',
        executable='create',
        output='screen',
        arguments=['-topic', 'robot_description', '-name', 'my_robot', '-allow_renaming', 'true'],
    )

    bridge = Node(
        package='ros_gz_bridge',
        executable='parameter_bridge',
        arguments=['/clock@rosgraph_msgs/msg/Clock[gz.msgs.Clock'],
        output='screen'
    )

    joint_state_broadcaster_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['joint_state_broadcaster', '--controller-manager', '/controller_manager'],
    )

    arm_controller_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['arm_controller', '-c', '/controller_manager'],
    )

    gripper_controller_spawner = Node(
        package='controller_manager',
        executable='spawner',
        arguments=['gripper_controller', '-c', '/controller_manager'],
    )

    return LaunchDescription([
        robot_state_publisher_node,
        gazebo,
        bridge,
        gz_spawn_entity,
        joint_state_broadcaster_spawner,
        arm_controller_spawner,
        gripper_controller_spawner,
    ])
EOF

# display.rviz — a saved camera angle so RViz opens already looking at the
# robot, instead of a blank gray screen.
cat > ~/ros2_ws/src/my_robot_bringup/rviz/display.rviz << 'EOF'
Panels:
  - Class: rviz_common/Displays
    Help Height: 78
    Name: Displays
    Property Tree Widget:
      Expanded:
        - /Global Options1
        - /RobotModel1
      Splitter Ratio: 0.5
    Tree Height: 549
Visualization Manager:
  Class: ""
  Displays:
    - Alpha: 0.5
      Cell Size: 1
      Class: rviz_default_plugins/Grid
      Color: 160; 160; 164
      Enabled: true
      Line Style:
        Line Width: 0.03
        Value: Lines
      Name: Grid
      Normal Cell Count: 0
      Offset:
        X: 0
        Y: 0
        Z: 0
      Plane: XY
      Plane Cell Count: 10
      Reference Frame: <Fixed Frame>
      Value: true
    - Alpha: 1
      Class: rviz_default_plugins/RobotModel
      Collision Enabled: false
      Description Source: Topic
      Description Topic:
        Depth: 5
        Durability Policy: Volatile
        History Policy: Keep Last
        Reliability Policy: Reliable
        Value: /robot_description
      Enabled: true
      Name: RobotModel
      TF Prefix: ""
      Update Interval: 0
      Value: true
      Visual Enabled: true
  Enabled: true
  Global Options:
    Background Color: 48; 48; 48
    Fixed Frame: base_link
    Frame Rate: 30
  Name: root
  Tools:
    - Class: rviz_default_plugins/Interact
      Hide Inactive Objects: true
    - Class: rviz_default_plugins/MoveCamera
    - Class: rviz_default_plugins/Select
  Value: true
  Views:
    Current:
      Class: rviz_default_plugins/Orbit
      Distance: 2.0
      Enable Suspend: false
      Focal Point:
        X: 0
        Y: 0
        Z: 0.5
      Name: Current View
      Pitch: 0.5
      Target Frame: <Fixed Frame>
      Value: Orbit (rviz)
      Yaw: 0.78
Window Geometry:
  Displays:
    collapsed: false
    Height: 800
    Width: 1200
    X: 100
    Y: 100
EOF

# sample_trajectory_publisher.py — a test script that alternates the arm
# between two poses every 3 seconds, to confirm the controllers work.
cat > ~/ros2_ws/src/my_robot_bringup/scripts/sample_trajectory_publisher.py << 'EOF'
#!/usr/bin/env python3
"""
sample_trajectory_publisher.py
Publishes sample JointTrajectory messages to test ros2_control.
Run AFTER gazebo.launch.py and controllers are active.
"""

import rclpy
from rclpy.node import Node
from trajectory_msgs.msg import JointTrajectory, JointTrajectoryPoint
from builtin_interfaces.msg import Duration
import math

class SampleTrajectoryPublisher(Node):

    def __init__(self):
        super().__init__('sample_trajectory_publisher')
        self.arm_pub = self.create_publisher(
            JointTrajectory, '/arm_controller/joint_trajectory', 10)
        self.gripper_pub = self.create_publisher(
            JointTrajectory, '/gripper_controller/joint_trajectory', 10)
        self.timer = self.create_timer(3.0, self.publish_trajectory)
        self.step = 0
        self.get_logger().info('SampleTrajectoryPublisher started!')

    def publish_trajectory(self):
        arm_msg = JointTrajectory()
        arm_msg.joint_names = ['joint1', 'joint2', 'joint3']
        gripper_msg = JointTrajectory()
        gripper_msg.joint_names = ['gripper_joint']
        point = JointTrajectoryPoint()
        gripper_point = JointTrajectoryPoint()
        if self.step % 2 == 0:
            point.positions = [0.0, -math.pi / 4, math.pi / 4]
            gripper_point.positions = [0.02]
            self.get_logger().info('Position A: arm forward, gripper open')
        else:
            point.positions = [math.pi / 2, -math.pi / 3, math.pi / 6]
            gripper_point.positions = [-0.02]
            self.get_logger().info('Position B: arm side, gripper closed')
        point.time_from_start = Duration(sec=2, nanosec=0)
        gripper_point.time_from_start = Duration(sec=2, nanosec=0)
        arm_msg.points = [point]
        gripper_msg.points = [gripper_point]
        self.arm_pub.publish(arm_msg)
        self.gripper_pub.publish(gripper_msg)
        self.step += 1

def main(args=None):
    rclpy.init(args=args)
    node = SampleTrajectoryPublisher()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
EOF

# The script above must be executable, or `ros2 run` will refuse to run it.
chmod +x ~/ros2_ws/src/my_robot_bringup/scripts/sample_trajectory_publisher.py
```

---

## Step 5 — Compile the Workspace

Now that every file exists, install any missing dependencies and compile:

```bash
cd ~/ros2_ws

# Scans package.xml files and installs anything still missing.
rosdep install --from-paths src --ignore-src -r -y

# Compiles both packages. --symlink-install means editing a Python/launch
# file later takes effect immediately without rebuilding.
colcon build --symlink-install

# Load the newly built packages into this terminal.
source install/setup.bash
```

💡 **Tip:** Add this line to your `~/.bashrc` so you never have to re-source manually:
```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
```

---

## Step 6 — Run It

### A. Visualize only (no physics) — start here
```bash
cd ~/ros2_ws && source install/setup.bash
ros2 launch my_robot_bringup display.launch.py
```
RViz2 and a slider GUI open. Drag the sliders to test each joint moves correctly.

### B. Full physics simulation in Gazebo
```bash
cd ~/ros2_ws && source install/setup.bash
ros2 launch my_robot_bringup gazebo.launch.py
```
Gazebo Fortress opens with the arm spawned and all controllers active.

### C. Auto-move test (run in a **second terminal**, while B is still running)
```bash
cd ~/ros2_ws && source install/setup.bash
ros2 run my_robot_bringup sample_trajectory_publisher.py
```
The arm alternates between two poses every 3 seconds — confirms everything is wired correctly.

---

## Understanding the Robot Model

| Part | Type | What it does |
|---|---|---|
| `base_link` | fixed | The mounting plate everything sits on |
| `joint1` | revolute (±180°) | Rotates the whole arm left/right around Z |
| `joint2` | revolute (±90°) | "Shoulder" — tilts the upper arm up/down |
| `joint3` | revolute (±90°) | "Elbow/wrist" — tilts the forearm up/down |
| `gripper_joint` | prismatic (±2cm) | Slides open/closed to grab things |

The xacro files are split by purpose so each stays simple: `arm_core.xacro` (shape), `arm.ros2_control.xacro` (motor wiring), `arm.gazebo.xacro` (simulator hookup) — all combined by `arm.urdf.xacro`.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Package 'my_robot_bringup' not found` | Workspace not sourced in this terminal | `source ~/ros2_ws/install/setup.bash` |
| Controller/plugin timeout in Gazebo | Missing `ros_gz`/`ros2_control` packages | `sudo apt install -y ros-humble-ros-gz-sim ros-humble-gz-ros2-control ros-humble-ros2-controllers` |
| `Permission denied` running the test script | Script isn't executable | `chmod +x ~/ros2_ws/src/my_robot_bringup/scripts/sample_trajectory_publisher.py` |
| Arm looks broken/white in RViz | Wrong Fixed Frame or no joint states | Set Fixed Frame to `base_link`; confirm `robot_state_publisher` is running |
| Empty Gazebo window, no robot | Spawn service timed out | Re-run `ros2 launch my_robot_bringup gazebo.launch.py` |
