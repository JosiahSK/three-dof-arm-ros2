# 3-DOF Robotic Arm with Gripper - ROS 2 Jazzy & Gazebo Harmonic

[![ROS 2 Jazzy](https://img.shields.io/badge/ROS2-Jazzy-blue)](https://docs.ros.org/en/jazzy/)
[![Gazebo Harmonic](https://img.shields.io/badge/Gazebo-Harmonic-orange)](https://gazebosim.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A **complete, production-ready** ROS 2 workspace for simulating and controlling a 3-DOF robotic manipulator arm with a 1-DOF prismatic gripper.

---

## Features

- **3 Revolute Joints** (joint1, joint2, joint3) - full arm motion
- **1 Prismatic Gripper Joint** - open/close control
- **GUI Slider Control** via joint_state_publisher_gui
- **RViz2 Visualization** with pre-loaded config
- **Gazebo Harmonic Simulation** with physics, gravity, and collisions
- **ros2_control** with JointStateBroadcaster + JointTrajectoryController
- **Sample Trajectory Publisher** script (bonus)

---

## Tech Stack

| Tool | Version |
|------|---------|
| ROS 2 | Jazzy |
| Gazebo | Harmonic |
| ros2_control | Latest |
| xacro | Latest |
| RViz2 | Latest |

---

## Repository Structure

```
three-dof-arm-ros2/
