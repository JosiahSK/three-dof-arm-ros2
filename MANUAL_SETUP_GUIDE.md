# 3-DOF Robotic Arm — Manual Setup Guide (No Git Clone)

This guide provides step-by-step instructions to create, build, and run the **3-DOF Robotic Arm with Gripper** project on **Ubuntu 24.04 LTS** using **ROS 2 Jazzy** and **Gazebo Harmonic** entirely from scratch, without cloning the Git repository.

---

## Table of Contents

- [1. Prerequisites \& Installation](#1-prerequisites--installation)
- [2. Workspace \& Package Structure Setup](#2-workspace--package-structure-setup)
- [3. Project Source Files Recreation](#3-project-source-files-recreation)
  - [3.1 `my_robot_description` Package](#31-my_robot_description-package)
    - [3.1.1 `src/my_robot_description/package.xml`](#311-srcmy_robot_descriptionpackagexml)
    - [3.1.2 `src/my_robot_description/CMakeLists.txt`](#312-srcmy_robot_descriptioncmakelists-txt)
    - [3.1.3 `src/my_robot_description/urdf/arm.urdf.xacro`](#313-srcmy_robot_descriptionurdfarmurdfxacro)
    - [3.1.4 `src/my_robot_description/urdf/arm_core.xacro`](#314-srcmy_robot_descriptionurdfarm_corexacro)
    - [3.1.5 `src/my_robot_description/urdf/arm.ros2_control.xacro`](#315-srcmy_robot_descriptionurdfarmros2_controlxacro)
    - [3.1.6 `src/my_robot_description/urdf/arm.gazebo.xacro`](#316-srcmy_robot_descriptionurdfarmgazeboxacro)
    - [3.1.7 `src/my_robot_description/config/controller.yaml`](#317-srcmy_robot_descriptionconfigcontrolleryaml)
    - [3.1.8 `src/my_robot_description/meshes/README.md`](#318-srcmy_robot_descriptionmeshesreadmemd)
  - [3.2 `my_robot_bringup` Package](#32-my_robot_bringup-package)
    - [3.2.1 `src/my_robot_bringup/package.xml`](#321-srcmy_robot_bringuppackagexml)
    - [3.2.2 `src/my_robot_bringup/CMakeLists.txt`](#322-srcmy_robot_bringupcmakelists-txt)
    - [3.2.3 `src/my_robot_bringup/launch/display.launch.py`](#323-srcmy_robot_bringuplaunchdisplaylaunchpy)
    - [3.2.4 `src/my_robot_bringup/launch/gazebo.launch.py`](#324-srcmy_robot_bringuplaunchgazebolaunchpy)
    - [3.2.5 `src/my_robot_bringup/rviz/display.rviz`](#325-srcmy_robot_bringuprvizdisplayrviz)
    - [3.2.6 `src/my_robot_bringup/scripts/sample_trajectory_publisher.py`](#326-srcmy_robot_bringupscriptssample_trajectory_publisherpy)
- [4. URDF / Xacro Robot Model Architecture](#4-urdf--xacro-robot-model-architecture)
- [5. Launch Files \& Execution Flow](#5-launch-files--execution-flow)
- [6. Full Build Sequence](#6-full-build-sequence)
- [7. Launching and Testing the Simulation](#7-launching-and-testing-the-simulation)
- [8. Troubleshooting Guide](#8-troubleshooting-guide)

---

## 1. Prerequisites & Installation

Ensure you are running **Ubuntu 24.04 LTS (Noble Numbat)**. Run the following terminal commands to install **ROS 2 Jazzy**, **Gazebo Harmonic**, `ros2_control`, `ros2_controllers`, `xacro`, and `joint_state_publisher_gui`:

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Install ROS 2 Jazzy Desktop (if not already installed)
sudo apt install -y ros-jazzy-desktop

# 3. Install Gazebo Harmonic and ROS 2 Integration packages
sudo apt install -y \
  ros-jazzy-ros-gz \
  ros-jazzy-ros-gz-sim \
  ros-jazzy-ros-gz-bridge \
  ros-jazzy-gz-ros2-control

# 4. Install ros2_control framework and controllers
sudo apt install -y \
  ros-jazzy-ros2-control \
  ros-jazzy-ros2-controllers \
  ros-jazzy-joint-state-broadcaster \
  ros-jazzy-joint-trajectory-controller

# 5. Install GUI tools, Xacro parser, and build utilities
sudo apt install -y \
  ros-jazzy-xacro \
  ros-jazzy-joint-state-publisher-gui \
  python3-colcon-common-extensions \
  python3-rosdep

# 6. Initialize rosdep (if not done previously)
sudo rosdep init 2>/dev/null || true
rosdep update
```

---

## 2. Workspace & Package Structure Setup

Create a dedicated ROS 2 workspace named `ros2_ws` and construct the exact folder hierarchy matching the project repository:

```bash
# Create workspace directory structure
mkdir -p ~/ros2_ws/src/my_robot_description/urdf
mkdir -p ~/ros2_ws/src/my_robot_description/config
mkdir -p ~/ros2_ws/src/my_robot_description/meshes
mkdir -p ~/ros2_ws/src/my_robot_bringup/launch
mkdir -p ~/ros2_ws/src/my_robot_bringup/rviz
mkdir -p ~/ros2_ws/src/my_robot_bringup/scripts
```

---

## 3. Project Source Files Recreation

Execute the shell commands below to automatically generate each source file in its correct relative path.

### 3.1 `my_robot_description` Package

#### 3.1.1 `src/my_robot_description/package.xml`

```bash
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
```

> **Description**: The `package.xml` defines metadata for the `my_robot_description` ROS 2 package, specifying dependencies such as `robot_state_publisher`, `xacro`, `ros2_control`, `gz_ros2_control`, and `ros2_controllers` necessary for parsing the robot description and interacting with Gazebo Harmonic.

---

#### 3.1.2 `src/my_robot_description/CMakeLists.txt`

```bash
cat > ~/ros2_ws/src/my_robot_description/CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.8)
project(my_robot_description)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)

# Install URDF, config, and mesh directories
install(
  DIRECTORY urdf config
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
EOF
```

> **Description**: The `CMakeLists.txt` configures `ament_cmake` build rules to install the `urdf/` and `config/` directories into the ROS 2 share directory (`share/my_robot_description`), allowing launch files and nodes to resolve robot models and configuration files dynamically at runtime.

---

#### 3.1.3 `src/my_robot_description/urdf/arm.urdf.xacro`

```bash
cat > ~/ros2_ws/src/my_robot_description/urdf/arm.urdf.xacro << 'EOF'
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="my_robot">

  <!-- Core robot links and joints -->
  <xacro:include filename="$(find my_robot_description)/urdf/arm_core.xacro" />

  <!-- ros2_control hardware interface block -->
  <xacro:include filename="$(find my_robot_description)/urdf/arm.ros2_control.xacro" />

  <!-- Gazebo Harmonic plugin block -->
  <xacro:include filename="$(find my_robot_description)/urdf/arm.gazebo.xacro" />

</robot>
EOF
```

> **Description**: This top-level Xacro file acts as the primary entry point for the robot description, combining the mechanical link/joint definitions (`arm_core.xacro`), the hardware interface setup (`arm.ros2_control.xacro`), and Gazebo Harmonic simulation plugins (`arm.gazebo.xacro`) into a single modular robot model named `my_robot`.

---

#### 3.1.4 `src/my_robot_description/urdf/arm_core.xacro`

```bash
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
```

> **Description**: `arm_core.xacro` defines the fundamental kinematics, dynamics, geometry, and visual colors for the manipulator arm. It specifies 5 rigid body links (`base_link`, `link1`, `link2`, `link3`, `gripper_link`) connected by 3 revolute joints (`joint1`, `joint2`, `joint3`) and 1 prismatic end-effector joint (`gripper_joint`), complete with inertial mass matrices for physics simulation.

---

#### 3.1.5 `src/my_robot_description/urdf/arm.ros2_control.xacro`

```bash
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
```

> **Description**: This file configures the `ros2_control` tag, specifying `gz_ros2_control/GazeboSimSystem` as the hardware plugin interface. It defines position command interfaces and position/velocity feedback state interfaces for all 4 joints (`joint1`, `joint2`, `joint3`, and `gripper_joint`).

---

#### 3.1.6 `src/my_robot_description/urdf/arm.gazebo.xacro`

```bash
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
```

> **Description**: This file integrates the `gz_ros2_control` system plugin into Gazebo Harmonic (`libgz_ros2_control-system.so`) and loads the controller parameter YAML file. Additionally, it assigns material colors (`Gazebo/Blue`, `Gazebo/Gray`, `Gazebo/Red`) for visual rendering inside Gazebo.

---

#### 3.1.7 `src/my_robot_description/config/controller.yaml`

```bash
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
```

> **Description**: The controller configuration YAML file parameterizes the `controller_manager`. It spawns a `joint_state_broadcaster` to publish active joint states and a `joint_trajectory_controller` (`arm_controller`) to control joint position trajectories for `joint1`, `joint2`, `joint3`, and `gripper_joint`.

---

#### 3.1.8 `src/my_robot_description/meshes/README.md`

```bash
cat > ~/ros2_ws/src/my_robot_description/meshes/README.md << 'EOF'
# Meshes

Place robot mesh files here (STL, DAE, OBJ).
Link links to this directory in URDF.
EOF
```

> **Description**: A placeholder documentation file explaining where custom 3D mesh files (STL, DAE, OBJ) should be stored if custom CAD geometries are used instead of primitive geometric shapes.

---

### 3.2 `my_robot_bringup` Package

#### 3.2.1 `src/my_robot_bringup/package.xml`

```bash
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
```

> **Description**: Defines metadata and runtime dependencies for `my_robot_bringup`, depending on `my_robot_description`, `rviz2`, `ros_gz_sim`, `ros_gz_bridge`, and `joint_state_publisher_gui` to orchestrate visualization and Gazebo simulation launches.

---

#### 3.2.2 `src/my_robot_bringup/CMakeLists.txt`

```bash
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
```

> **Description**: Configures `my_robot_bringup` build instructions to install the `launch/` directory into `share/my_robot_bringup`, making launch scripts accessible via `ros2 launch my_robot_bringup <script.launch.py>`.

---

#### 3.2.3 `src/my_robot_bringup/launch/display.launch.py`

```bash
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
        DeclareLaunchArgument(
            name='model',
            default_value=default_model_path,
            description='Absolute path to robot URDF/xacro file'
        ),
        DeclareLaunchArgument(
            name='rvizconfig',
            default_value=default_rviz_path,
            description='Absolute path to RViz config file'
        ),
        robot_state_publisher_node,
        joint_state_publisher_gui_node,
        rviz_node
    ])
EOF
```

> **Description**: Launch script for lightweight offline visualization in RViz2 without running physics simulation. It executes `robot_state_publisher` to parse the URDF/Xacro, launches `joint_state_publisher_gui` for interactive slider-based joint manipulation, and opens RViz2 with pre-configured camera and TF settings.

---

#### 3.2.4 `src/my_robot_bringup/launch/gazebo.launch.py`

```bash
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
            PathJoinSubstitution([
                FindPackageShare('ros_gz_sim'),
                'launch',
                'gz_sim.launch.py'
            ])
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
```

> **Description**: Main simulation launch file. It starts `robot_state_publisher`, boots Gazebo Harmonic (`ros_gz_sim`), spawns the robot into an empty simulation world, establishes a ROS-Gazebo clock bridge (`ros_gz_bridge`), and automatically spawns the `joint_state_broadcaster`, `arm_controller`, and `gripper_controller`.

---

#### 3.2.5 `src/my_robot_bringup/rviz/display.rviz`

```bash
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
```

> **Description**: The RViz2 display configuration file sets `base_link` as the fixed reference frame, adds a ground reference grid, displays the 3D robot model from topic `/robot_description`, and sets up an isometric orbital camera angle for optimal viewing.

---

#### 3.2.6 `src/my_robot_bringup/scripts/sample_trajectory_publisher.py`

```bash
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

# Grant executable permission to the script
chmod +x ~/ros2_ws/src/my_robot_bringup/scripts/sample_trajectory_publisher.py
```

> **Description**: A Python ROS 2 node that periodically publishes joint trajectory commands (`trajectory_msgs/msg/JointTrajectory`) every 3 seconds to `/arm_controller/joint_trajectory` and `/gripper_controller/joint_trajectory`, alternating between two target arm configurations to verify dynamic motion control in simulation.

---

## 4. URDF / Xacro Robot Model Architecture

The robot model uses a clean, modular XML Macro (`xacro`) structure:

- **Root File (`arm.urdf.xacro`)**: Combines core geometry (`arm_core.xacro`), hardware interfaces (`arm.ros2_control.xacro`), and simulation plugins (`arm.gazebo.xacro`) into a clean unified representation using `<xacro:include>` directives.
- **Kinematic Chain (`arm_core.xacro`)**:
  - `base_link`: Cylindrical ground mounting plate (radius `0.1m`, height `0.05m`).
  - `joint1` (Revolute): Rotates around Z-axis (`[0, 0, 1]`) between `-180°` (`-3.14 rad`) and `+180°` (`+3.14 rad`). Connects `base_link` to `link1`.
  - `link1`: Vertical arm link (height `0.3m`, radius `0.05m`).
  - `joint2` (Revolute): Pitch joint rotating around Y-axis (`[0, 1, 0]`) between `-90°` (`-1.57 rad`) and `+90°` (`+1.57 rad`). Connects `link1` to `link2`.
  - `link2`: Forearm link (length `0.3m`, radius `0.04m`).
  - `joint3` (Revolute): Wrist pitch joint rotating around Y-axis (`[0, 1, 0]`) between `-90°` (`-1.57 rad`) and `+90°` (`+1.57 rad`). Connects `link2` to `link3`.
  - `link3`: Wrist box block (dimensions `0.05 x 0.05 x 0.1m`).
  - `gripper_joint` (Prismatic): Linear motion along X-axis (`[1, 0, 0]`) ranging from `-0.02m` to `+0.02m`. Connects `link3` to `gripper_link`.
  - `gripper_link`: Rectangular end-effector tool.

---

## 5. Launch Files & Execution Flow

1. **`display.launch.py` (Visualization Mode)**:
   - Used for quick mechanical checks without loading heavy physics engine overhead.
   - Starts `robot_state_publisher` to calculate transformation matrices.
   - Starts `joint_state_publisher_gui` offering manual sliders to test joint limits.
   - Launches RViz2 with pre-configured camera angle.

2. **`gazebo.launch.py` (Full Physics Simulation Mode)**:
   - Starts `robot_state_publisher` in simulation time mode (`use_sim_time: True`).
   - Launches Gazebo Harmonic (`gz_sim`) in an empty world (`empty.sdf`).
   - Spawns the robot model into Gazebo using the `create` node.
   - Launches `ros_gz_bridge` to synchronize ROS and Gazebo simulation clocks.
   - Automatically loads and activates `joint_state_broadcaster`, `arm_controller`, and `gripper_controller` via `controller_manager/spawner`.

---

## 6. Full Build Sequence

Run the following commands from your terminal to install any missing workspace dependencies, compile the ROS 2 packages, and source the overlay environment:

```bash
# 1. Navigate to workspace root
cd ~/ros2_ws

# 2. Automatically resolve missing package dependencies
rosdep install --from-paths src --ignore-src -r -y

# 3. Build workspace packages using symlink installation
colcon build --symlink-install

# 4. Source the workspace overlay environment
source install/setup.bash
```

> [!TIP]
> Add `source ~/ros2_ws/install/setup.bash` to your `~/.bashrc` file to automatically source this workspace whenever you open a new terminal window.

---

## 7. Launching and Testing the Simulation

Follow these step-by-step commands to visualize and run the arm simulation:

### Option A: Offline Joint Slider Testing in RViz2

Open a terminal and run:

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 launch my_robot_bringup display.launch.py
```
*An RViz2 window and a GUI window with joint position sliders will open. Move the sliders to test joint rotations.*

---

### Option B: Gazebo Harmonic Physics Simulation

Open a new terminal and run:

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 launch my_robot_bringup gazebo.launch.py
```
*Gazebo Harmonic will launch with the 3-DOF arm spawned at origin, and controllers will initialize automatically.*

---

### Option C: Automated Trajectory Control Test

With `gazebo.launch.py` running in your first terminal, open a **second terminal** and run the test script:

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run my_robot_bringup sample_trajectory_publisher.py
```
*The script will publish joint trajectory commands every 3 seconds, causing the robot arm to alternate between Position A and Position B in Gazebo.*

---

## 8. Troubleshooting Guide

| Issue / Symptom | Probable Cause | Recommended Fix |
|---|---|---|
| `Package 'my_robot_bringup' not found` | Workspace environment not sourced in current terminal. | Run `source ~/ros2_ws/install/setup.bash` in the terminal before running `ros2 launch`. |
| `gz_ros2_control` plugin failure or controller timeout | Missing `ros_gz` or `ros2_control` packages for Jazzy. | Run `sudo apt install -y ros-jazzy-ros-gz-sim ros-jazzy-gz-ros2-control ros-jazzy-ros2-controllers`. |
| `Permission denied` when running `sample_trajectory_publisher.py` | Python script lacks execute permissions. | Run `chmod +x ~/ros2_ws/src/my_robot_bringup/scripts/sample_trajectory_publisher.py`. |
| Robot arm appears white or broken in RViz2 | Fixed Frame is set incorrectly or missing joint states. | Ensure Fixed Frame in RViz is set to `base_link` and `robot_state_publisher` is active. |
| Empty Gazebo window with no robot model | `ros_gz_sim` spawn service failed or timed out during launch. | Verify Gazebo started properly, or run `ros2 launch my_robot_bringup gazebo.launch.py` again. |
