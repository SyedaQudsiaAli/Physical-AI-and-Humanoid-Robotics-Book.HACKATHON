---
sidebar_position: 2
---

# Lesson 1.2: URDF and Kinematic Modeling for Humanoid Joints

This lesson covers URDF (Unified Robot Description Format) and kinematic modeling for humanoid joints, providing a comprehensive approach to representing humanoid robots in ROS 2.

## Introduction

URDF (Unified Robot Description Format) is the standard XML format for representing robot models in ROS. For humanoid robots, proper URDF modeling is crucial for accurate simulation, control, and kinematic analysis. This lesson explores the detailed structure of URDF files for humanoid robots, including links, joints, inertial properties, and kinematic chains.

## Key Concepts

- URDF for humanoid modeling: links, joints, inertia, kinematics
- Joint types and constraints for humanoid robots
- Kinematic chains and forward/inverse kinematics
- Collision and visual properties
- Inertial parameters for dynamic simulation
- Transmission elements for actuator modeling

## Detailed URDF Structure for Humanoid Robots

A humanoid robot URDF model consists of multiple interconnected links and joints that represent the physical structure of the robot:

```xml
<?xml version="1.0"?>
<robot name="simple_humanoid" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <!-- Base link - represents the main body/torso -->
  <link name="base_link">
    <inertial>
      <mass value="10.0" />
      <origin xyz="0 0 0.5" />
      <inertia ixx="1.0" ixy="0" ixz="0" iyy="1.0" iyz="0" izz="1.0" />
    </inertial>
    <visual>
      <origin xyz="0 0 0.5" rpy="0 0 0" />
      <geometry>
        <box size="0.3 0.3 1.0" />
      </geometry>
      <material name="light_grey">
        <color rgba="0.7 0.7 0.7 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 0.5" rpy="0 0 0" />
      <geometry>
        <box size="0.3 0.3 1.0" />
      </geometry>
    </collision>
  </link>

  <!-- Head link -->
  <link name="head">
    <inertial>
      <mass value="2.0" />
      <origin xyz="0 0 0" />
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01" />
    </inertial>
    <visual>
      <origin xyz="0 0 0" rpy="0 0 0" />
      <geometry>
        <sphere radius="0.15" />
      </geometry>
      <material name="white">
        <color rgba="1.0 1.0 1.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 0" rpy="0 0 0" />
      <geometry>
        <sphere radius="0.15" />
      </geometry>
    </collision>
  </link>

  <!-- Neck joint -->
  <joint name="neck_joint" type="revolute">
    <parent link="base_link" />
    <child link="head" />
    <origin xyz="0 0 1.0" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-0.5" upper="0.5" effort="10" velocity="2" />
  </joint>

  <!-- Left arm - shoulder -->
  <link name="left_shoulder">
    <inertial>
      <mass value="1.0" />
      <origin xyz="0 0 -0.1" />
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.05" />
      </geometry>
      <material name="blue">
        <color rgba="0.0 0.0 1.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.05" />
      </geometry>
    </collision>
  </link>

  <joint name="left_shoulder_joint" type="revolute">
    <parent link="base_link" />
    <child link="left_shoulder" />
    <origin xyz="0.15 0 0.8" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-1.57" upper="1.57" effort="20" velocity="2" />
  </joint>

  <!-- Left arm - upper arm -->
  <link name="left_upper_arm">
    <inertial>
      <mass value="1.0" />
      <origin xyz="0 0 -0.15" />
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.04" />
      </geometry>
      <material name="blue">
        <color rgba="0.0 0.0 1.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.04" />
      </geometry>
    </collision>
  </link>

  <joint name="left_elbow_joint" type="revolute">
    <parent link="left_shoulder" />
    <child link="left_upper_arm" />
    <origin xyz="0 0 -0.2" rpy="0 0 0" />
    <axis xyz="0 0 1" />
    <limit lower="-2.36" upper="0" effort="15" velocity="2" />
  </joint>

  <!-- Left arm - lower arm -->
  <link name="left_lower_arm">
    <inertial>
      <mass value="0.8" />
      <origin xyz="0 0 -0.15" />
      <inertia ixx="0.008" ixy="0" ixz="0" iyy="0.008" iyz="0" izz="0.008" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.035" />
      </geometry>
      <material name="blue">
        <color rgba="0.0 0.0 1.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.035" />
      </geometry>
    </collision>
  </link>

  <joint name="left_wrist_joint" type="revolute">
    <parent link="left_upper_arm" />
    <child link="left_lower_arm" />
    <origin xyz="0 0 -0.3" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-1.57" upper="1.57" effort="10" velocity="2" />
  </joint>

  <!-- Left hand -->
  <link name="left_hand">
    <inertial>
      <mass value="0.3" />
      <origin xyz="0 0 -0.05" />
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.05" rpy="0 0 0" />
      <geometry>
        <box size="0.1 0.08 0.1" />
      </geometry>
      <material name="light_grey">
        <color rgba="0.7 0.7 0.7 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.05" rpy="0 0 0" />
      <geometry>
        <box size="0.1 0.08 0.1" />
      </geometry>
    </collision>
  </link>

  <joint name="left_hand_joint" type="fixed">
    <parent link="left_lower_arm" />
    <child link="left_hand" />
    <origin xyz="0 0 -0.1" rpy="0 0 0" />
  </joint>

  <!-- Similar structure for right arm -->
  <link name="right_shoulder">
    <inertial>
      <mass value="1.0" />
      <origin xyz="0 0 -0.1" />
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.05" />
      </geometry>
      <material name="red">
        <color rgba="1.0 0.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.05" />
      </geometry>
    </collision>
  </link>

  <joint name="right_shoulder_joint" type="revolute">
    <parent link="base_link" />
    <child link="right_shoulder" />
    <origin xyz="-0.15 0 0.8" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-1.57" upper="1.57" effort="20" velocity="2" />
  </joint>

  <link name="right_upper_arm">
    <inertial>
      <mass value="1.0" />
      <origin xyz="0 0 -0.15" />
      <inertia ixx="0.01" ixy="0" ixz="0" iyy="0.01" iyz="0" izz="0.01" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.04" />
      </geometry>
      <material name="red">
        <color rgba="1.0 0.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.04" />
      </geometry>
    </collision>
  </link>

  <joint name="right_elbow_joint" type="revolute">
    <parent link="right_shoulder" />
    <child link="right_upper_arm" />
    <origin xyz="0 0 -0.2" rpy="0 0 0" />
    <axis xyz="0 0 1" />
    <limit lower="0" upper="2.36" effort="15" velocity="2" />
  </joint>

  <link name="right_lower_arm">
    <inertial>
      <mass value="0.8" />
      <origin xyz="0 0 -0.15" />
      <inertia ixx="0.008" ixy="0" ixz="0" iyy="0.008" iyz="0" izz="0.008" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.035" />
      </geometry>
      <material name="red">
        <color rgba="1.0 0.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.15" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.3" radius="0.035" />
      </geometry>
    </collision>
  </link>

  <joint name="right_wrist_joint" type="revolute">
    <parent link="right_upper_arm" />
    <child link="right_lower_arm" />
    <origin xyz="0 0 -0.3" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-1.57" upper="1.57" effort="10" velocity="2" />
  </joint>

  <link name="right_hand">
    <inertial>
      <mass value="0.3" />
      <origin xyz="0 0 -0.05" />
      <inertia ixx="0.001" ixy="0" ixz="0" iyy="0.001" iyz="0" izz="0.001" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.05" rpy="0 0 0" />
      <geometry>
        <box size="0.1 0.08 0.1" />
      </geometry>
      <material name="light_grey">
        <color rgba="0.7 0.7 0.7 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.05" rpy="0 0 0" />
      <geometry>
        <box size="0.1 0.08 0.1" />
      </geometry>
    </collision>
  </link>

  <joint name="right_hand_joint" type="fixed">
    <parent link="right_lower_arm" />
    <child link="right_hand" />
    <origin xyz="0 0 -0.1" rpy="0 0 0" />
  </joint>

  <!-- Left leg - hip -->
  <link name="left_hip">
    <inertial>
      <mass value="1.5" />
      <origin xyz="0 0 -0.1" />
      <inertia ixx="0.02" ixy="0" ixz="0" iyy="0.02" iyz="0" izz="0.02" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.06" />
      </geometry>
      <material name="green">
        <color rgba="0.0 1.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.06" />
      </geometry>
    </collision>
  </link>

  <joint name="left_hip_joint" type="revolute">
    <parent link="base_link" />
    <child link="left_hip" />
    <origin xyz="0.08 0 0" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-0.79" upper="0.79" effort="30" velocity="2" />
  </joint>

  <!-- Left leg - thigh -->
  <link name="left_thigh">
    <inertial>
      <mass value="2.0" />
      <origin xyz="0 0 -0.2" />
      <inertia ixx="0.04" ixy="0" ixz="0" iyy="0.04" iyz="0" izz="0.04" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.055" />
      </geometry>
      <material name="green">
        <color rgba="0.0 1.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.055" />
      </geometry>
    </collision>
  </link>

  <joint name="left_knee_joint" type="revolute">
    <parent link="left_hip" />
    <child link="left_thigh" />
    <origin xyz="0 0 -0.2" rpy="0 0 0" />
    <axis xyz="0 0 1" />
    <limit lower="0" upper="2.36" effort="30" velocity="2" />
  </joint>

  <!-- Left leg - shin -->
  <link name="left_shin">
    <inertial>
      <mass value="1.5" />
      <origin xyz="0 0 -0.2" />
      <inertia ixx="0.03" ixy="0" ixz="0" iyy="0.03" iyz="0" izz="0.03" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.05" />
      </geometry>
      <material name="green">
        <color rgba="0.0 1.0 0.0 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.05" />
      </geometry>
    </collision>
  </link>

  <joint name="left_ankle_joint" type="revolute">
    <parent link="left_thigh" />
    <child link="left_shin" />
    <origin xyz="0 0 -0.4" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-0.79" upper="0.79" effort="20" velocity="2" />
  </joint>

  <!-- Left foot -->
  <link name="left_foot">
    <inertial>
      <mass value="0.8" />
      <origin xyz="0.05 0 -0.02" />
      <inertia ixx="0.005" ixy="0" ixz="0" iyy="0.005" iyz="0" izz="0.005" />
    </inertial>
    <visual>
      <origin xyz="0.05 0 -0.02" rpy="0 0 0" />
      <geometry>
        <box size="0.18 0.1 0.04" />
      </geometry>
      <material name="dark_grey">
        <color rgba="0.3 0.3 0.3 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0.05 0 -0.02" rpy="0 0 0" />
      <geometry>
        <box size="0.18 0.1 0.04" />
      </geometry>
    </collision>
  </link>

  <joint name="left_foot_joint" type="fixed">
    <parent link="left_shin" />
    <child link="left_foot" />
    <origin xyz="0 0 -0.4" rpy="0 0 0" />
  </joint>

  <!-- Similar structure for right leg -->
  <link name="right_hip">
    <inertial>
      <mass value="1.5" />
      <origin xyz="0 0 -0.1" />
      <inertia ixx="0.02" ixy="0" ixz="0" iyy="0.02" iyz="0" izz="0.02" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.06" />
      </geometry>
      <material name="purple">
        <color rgba="0.5 0.0 0.5 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.1" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.2" radius="0.06" />
      </geometry>
    </collision>
  </link>

  <joint name="right_hip_joint" type="revolute">
    <parent link="base_link" />
    <child link="right_hip" />
    <origin xyz="-0.08 0 0" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-0.79" upper="0.79" effort="30" velocity="2" />
  </joint>

  <link name="right_thigh">
    <inertial>
      <mass value="2.0" />
      <origin xyz="0 0 -0.2" />
      <inertia ixx="0.04" ixy="0" ixz="0" iyy="0.04" iyz="0" izz="0.04" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.055" />
      </geometry>
      <material name="purple">
        <color rgba="0.5 0.0 0.5 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.055" />
      </geometry>
    </collision>
  </link>

  <joint name="right_knee_joint" type="revolute">
    <parent link="right_hip" />
    <child link="right_thigh" />
    <origin xyz="0 0 -0.2" rpy="0 0 0" />
    <axis xyz="0 0 1" />
    <limit lower="0" upper="2.36" effort="30" velocity="2" />
  </joint>

  <link name="right_shin">
    <inertial>
      <mass value="1.5" />
      <origin xyz="0 0 -0.2" />
      <inertia ixx="0.03" ixy="0" ixz="0" iyy="0.03" iyz="0" izz="0.03" />
    </inertial>
    <visual>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.05" />
      </geometry>
      <material name="purple">
        <color rgba="0.5 0.0 0.5 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0 0 -0.2" rpy="0 0 0" />
      <geometry>
        <cylinder length="0.4" radius="0.05" />
      </geometry>
    </collision>
  </link>

  <joint name="right_ankle_joint" type="revolute">
    <parent link="right_thigh" />
    <child link="right_shin" />
    <origin xyz="0 0 -0.4" rpy="0 0 0" />
    <axis xyz="0 1 0" />
    <limit lower="-0.79" upper="0.79" effort="20" velocity="2" />
  </joint>

  <link name="right_foot">
    <inertial>
      <mass value="0.8" />
      <origin xyz="0.05 0 -0.02" />
      <inertia ixx="0.005" ixy="0" ixz="0" iyy="0.005" iyz="0" izz="0.005" />
    </inertial>
    <visual>
      <origin xyz="0.05 0 -0.02" rpy="0 0 0" />
      <geometry>
        <box size="0.18 0.1 0.04" />
      </geometry>
      <material name="dark_grey">
        <color rgba="0.3 0.3 0.3 1.0" />
      </material>
    </visual>
    <collision>
      <origin xyz="0.05 0 -0.02" rpy="0 0 0" />
      <geometry>
        <box size="0.18 0.1 0.04" />
      </geometry>
    </collision>
  </link>

  <joint name="right_foot_joint" type="fixed">
    <parent link="right_shin" />
    <child link="right_foot" />
    <origin xyz="0 0 -0.4" rpy="0 0 0" />
  </joint>

  <!-- Gazebo plugin for physics simulation -->
  <gazebo reference="base_link">
    <material>Gazebo/Grey</material>
  </gazebo>

  <gazebo>
    <plugin name="humanoid_controller" filename="libgazebo_ros_joint_state_publisher.so">
      <robotNamespace>/humanoid</robotNamespace>
      <jointName>neck_joint, left_shoulder_joint, left_elbow_joint, left_wrist_joint, right_shoulder_joint, right_elbow_joint, right_wrist_joint, left_hip_joint, left_knee_joint, left_ankle_joint, right_hip_joint, right_knee_joint, right_ankle_joint</jointName>
    </plugin>
  </gazebo>

</robot>
```

## Kinematic Modeling for Humanoid Robots

Kinematic modeling involves understanding the relationship between joint angles and the position/orientation of the robot's end effectors. For humanoid robots, this is particularly important for:

- Forward kinematics: computing end-effector position from joint angles
- Inverse kinematics: computing joint angles to achieve desired end-effector position
- Balance and stability analysis
- Motion planning

## Humanoid-Specific Considerations

Humanoid robots have specific requirements for joint modeling:

- **Multiple degrees of freedom**: Each limb requires multiple joints to achieve human-like motion
- **Balance and stability**: The model must support center of mass calculations for stable walking
- **Coordination between multiple joints**: Arms and legs must work together for complex tasks
- **Anthropomorphic proportions**: Joint limits and ranges should reflect human-like capabilities
- **Bipedal locomotion**: Leg joints must support walking, standing, and balance control

## Inertial Properties and Dynamic Simulation

Accurate inertial properties are crucial for realistic simulation:

- Mass values should reflect the physical robot
- Inertia tensors must be calculated based on the shape and mass distribution
- Center of mass locations affect balance and stability
- Proper inertial properties enable accurate physics simulation in Gazebo

## Xacro for Complex Humanoid Models

For complex humanoid models, Xacro (XML Macros) can simplify the URDF definition:

```xml
<?xml version="1.0"?>
<robot xmlns:xacro="http://www.ros.org/wiki/xacro" name="humanoid_with_xacro">

  <!-- Define properties -->
  <xacro:property name="M_PI" value="3.1415926535897931" />
  <xacro:property name="base_mass" value="10.0" />
  <xacro:property name="arm_mass" value="1.0" />
  <xacro:property name="leg_mass" value="2.0" />

  <!-- Macro for a generic joint -->
  <xacro:macro name="generic_joint" params="name type parent child origin_xyz axis_xyz lower upper effort velocity">
    <joint name="${name}" type="${type}">
      <parent link="${parent}" />
      <child link="${child}" />
      <origin xyz="${origin_xyz}" rpy="0 0 0" />
      <axis xyz="${axis_xyz}" />
      <limit lower="${lower}" upper="${upper}" effort="${effort}" velocity="${velocity}" />
    </joint>
  </xacro:macro>

  <!-- Use the macro to define joints -->
  <xacro:generic_joint
    name="left_shoulder_joint"
    type="revolute"
    parent="base_link"
    child="left_shoulder"
    origin_xyz="0.15 0 0.8"
    axis_xyz="0 1 0"
    lower="-1.57"
    upper="1.57"
    effort="20"
    velocity="2" />

</robot>
```

## Summary

This lesson provided a comprehensive overview of URDF for humanoid modeling, including detailed examples of links, joints, and kinematic chains. We explored the specific requirements for humanoid robots, including multiple degrees of freedom, balance considerations, and proper inertial properties. The next lesson will cover real-time joint control and the integration of Python code with ROS controllers.

## References

1. ROS. (n.d.). *URDF: Unified Robot Description Format*. http://wiki.ros.org/urdf
2. Chitta, S., Sucan, I., & Cousins, S. (2012). MoveLib: Motion planning library. *IEEE Robotics & Automation Magazine*, 19(4), 18-31. https://doi.org/10.1109/MRA.2012.2205352
3. Featherstone, R. (2008). *Rigid body dynamics algorithms*. Springer Science & Business Media. https://doi.org/10.1007/978-1-4899-7560-7