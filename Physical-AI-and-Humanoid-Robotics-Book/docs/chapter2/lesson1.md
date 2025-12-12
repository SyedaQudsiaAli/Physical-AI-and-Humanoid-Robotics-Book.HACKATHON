---
sidebar_position: 1
---

# Lesson 2.1: Gazebo Physics Engine Fundamentals and Bipedal Balance

This lesson covers the fundamentals of the Gazebo physics engine and its application to bipedal balance simulation for humanoid robots, providing a comprehensive understanding of physics-based simulation for humanoid robotics.

## Introduction

Gazebo is a powerful physics simulation engine that enables the creation of complex robotic scenarios with realistic physics. For humanoid robots, Gazebo provides essential capabilities for simulating bipedal balance, joint dynamics, and environmental interactions. This lesson explores the core concepts of Gazebo physics simulation and its application to humanoid robotics, with particular focus on bipedal balance challenges.

## Key Concepts

- Gazebo physics engine fundamentals and configuration
- Bipedal balance simulation and stability analysis
- Physics parameters for humanoid models and joints
- Integration with ROS 2 for real-time control and feedback
- Collision detection and contact modeling for humanoid robots

## Gazebo Physics Engine Architecture

Gazebo uses physics engines such as ODE (Open Dynamics Engine), Bullet, or DART to simulate realistic physics. For humanoid robots, the physics simulation must accurately model:

- Joint dynamics and constraints
- Collision detection and response
- Contact forces and friction
- Center of mass calculations
- Inertial properties

### Physics Engine Configuration

```xml
<!-- Gazebo world configuration with physics parameters -->
<sdf version='1.7'>
  <world name='humanoid_world'>
    <!-- Physics engine configuration -->
    <physics type='ode'>
      <max_step_size>0.001</max_step_size>
      <real_time_factor>1.0</real_time_factor>
      <real_time_update_rate>1000</real_time_update_rate>
      <gravity>0 0 -9.8</gravity>
      <ode>
        <solver>
          <type>quick</type>
          <iters>10</iters>
          <sor>1.3</sor>
        </solver>
        <constraints>
          <cfm>0.0</cfm>
          <erp>0.2</erp>
          <contact_max_correcting_vel>100.0</contact_max_correcting_vel>
          <contact_surface_layer>0.001</contact_surface_layer>
        </constraints>
      </ode>
    </physics>

    <!-- Include ground plane -->
    <include>
      <uri>model://ground_plane</uri>
    </include>

    <!-- Include sky -->
    <include>
      <uri>model://sun</uri>
    </include>
  </world>
</sdf>
```

## Bipedal Balance Simulation

Bipedal balance for humanoid robots is one of the most challenging aspects of robotics simulation. Key considerations include:

### Center of Mass (CoM) Management
The center of mass must remain within the support polygon defined by the feet for stable standing.

```xml
<!-- Example humanoid model with CoM considerations -->
<robot name="humanoid_with_balance">
  <!-- Torso with low center of mass -->
  <link name="torso">
    <inertial>
      <mass value="8.0" />
      <origin xyz="0 0 -0.1" />
      <inertia ixx="0.2" ixy="0" ixz="0" iyy="0.2" iyz="0" izz="0.1" />
    </inertial>
    <visual>
      <origin xyz="0 0 0" rpy="0 0 0" />
      <geometry>
        <box size="0.2 0.2 0.6" />
      </geometry>
    </visual>
    <collision>
      <origin xyz="0 0 0" rpy="0 0 0" />
      <geometry>
        <box size="0.2 0.2 0.6" />
      </geometry>
    </collision>
  </link>
</robot>
```

### Support Polygon and Zero Moment Point (ZMP)
The Zero Moment Point is crucial for bipedal stability analysis in simulation:

```python
import numpy as np

def calculate_zmp_forces(center_of_pressure_x, center_of_pressure_y,
                        total_force, robot_height):
    """
    Calculate Zero Moment Point (ZMP) for bipedal balance
    """
    # ZMP calculation based on center of pressure and total force
    zmp_x = center_of_pressure_x - (robot_height / 9.81) * (total_force[1] / total_force[2])
    zmp_y = center_of_pressure_y + (robot_height / 9.81) * (total_force[0] / total_force[2])

    return zmp_x, zmp_y

def check_balance_stability(zmp_x, zmp_y, support_polygon):
    """
    Check if ZMP is within the support polygon for stability
    """
    # Check if ZMP is within the convex hull of support points
    # This is a simplified check - in practice, more complex geometry is used
    min_x, max_x = support_polygon['x_range']
    min_y, max_y = support_polygon['y_range']

    return min_x <= zmp_x <= max_x and min_y <= zmp_y <= max_y
```

## Gazebo Plugin Configuration for Humanoid Robots

Gazebo plugins enable integration with ROS 2 and provide specialized functionality for humanoid robots:

```xml
<!-- Gazebo plugin configuration for humanoid robot -->
<robot name="humanoid_with_plugins" xmlns:xacro="http://www.ros.org/wiki/xacro">
  <!-- Joint state publisher plugin -->
  <gazebo>
    <plugin name="joint_state_publisher" filename="libgazebo_ros_joint_state_publisher.so">
      <robotNamespace>/humanoid</robotNamespace>
      <jointName>left_hip_joint, left_knee_joint, left_ankle_joint,
                right_hip_joint, right_knee_joint, right_ankle_joint,
                left_shoulder_joint, left_elbow_joint, left_wrist_joint,
                right_shoulder_joint, right_elbow_joint, right_wrist_joint</jointName>
      <updateRate>100</updateRate>
    </plugin>
  </gazebo>

  <!-- Joint trajectory controller plugin -->
  <gazebo>
    <plugin name="joint_trajectory_controller" filename="libgazebo_ros_joint_trajectory.so">
      <robotNamespace>/humanoid</robotNamespace>
      <topicName>joint_trajectory</topicName>
      <serviceName>joint_trajectory_service</serviceName>
    </plugin>
  </gazebo>

  <!-- Contact sensor plugin for foot contact detection -->
  <gazebo reference="left_foot">
    <sensor name="left_foot_contact" type="contact">
      <always_on>true</always_on>
      <update_rate>100</update_rate>
      <contact>
        <collision>left_foot_collision</collision>
      </contact>
      <plugin name="left_foot_contact_plugin" filename="libgazebo_ros_bumper.so">
        <robotNamespace>/humanoid</robotNamespace>
        <topicName>left_foot_contact</topicName>
      </plugin>
    </sensor>
  </gazebo>

  <!-- IMU sensor plugin for balance feedback -->
  <gazebo reference="torso">
    <sensor name="torso_imu" type="imu">
      <always_on>true</always_on>
      <update_rate>100</update_rate>
      <imu>
        <angular_velocity>
          <x>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>2e-4</stddev>
            </noise>
          </x>
          <y>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>2e-4</stddev>
            </noise>
          </y>
          <z>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>2e-4</stddev>
            </noise>
          </z>
        </angular_velocity>
        <linear_acceleration>
          <x>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>1.7e-2</stddev>
            </noise>
          </x>
          <y>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>1.7e-2</stddev>
            </noise>
          </y>
          <z>
            <noise type="gaussian">
              <mean>0.0</mean>
              <stddev>1.7e-2</stddev>
            </noise>
          </z>
        </linear_acceleration>
      </imu>
      <plugin name="torso_imu_plugin" filename="libgazebo_ros_imu.so">
        <robotNamespace>/humanoid</robotNamespace>
        <topicName>imu/data</topicName>
        <serviceName>imu/service</serviceName>
      </plugin>
    </sensor>
  </gazebo>
</robot>
```

## Balance Control Strategies in Simulation

Implementing balance control in simulation requires understanding the physics and implementing appropriate control strategies:

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Imu, JointState
from geometry_msgs.msg import Vector3
import numpy as np
from tf2_ros import TransformListener, Buffer
import math

class BalanceController(Node):
    """
    Balance controller for humanoid robot in Gazebo simulation
    """
    def __init__(self):
        super().__init__('balance_controller')

        # Subscribers
        self.imu_sub = self.create_subscription(
            Imu, '/humanoid/imu/data', self.imu_callback, 10)
        self.joint_state_sub = self.create_subscription(
            JointState, '/humanoid/joint_states', self.joint_state_callback, 10)

        # Publishers
        self.joint_cmd_pub = self.create_publisher(
            JointState, '/humanoid/joint_commands', 10)

        # Balance control parameters
        self.roll_pid = PIDController(kp=2.0, ki=0.1, kd=0.5)
        self.pitch_pid = PIDController(kp=2.0, ki=0.1, kd=0.5)
        self.control_rate = 100  # Hz
        self.timer = self.create_timer(1.0/self.control_rate, self.balance_control_loop)

        # State variables
        self.current_imu = Imu()
        self.current_joint_state = JointState()
        self.balance_enabled = True

        self.get_logger().info('Balance controller initialized')

    def imu_callback(self, msg):
        """Handle IMU data for balance feedback"""
        self.current_imu = msg

    def joint_state_callback(self, msg):
        """Handle joint state feedback"""
        self.current_joint_state = msg

    def balance_control_loop(self):
        """Main balance control loop"""
        if not self.balance_enabled:
            return

        # Extract roll and pitch from IMU quaternion
        roll, pitch, yaw = self.quaternion_to_euler(
            self.current_imu.orientation.x,
            self.current_imu.orientation.y,
            self.current_imu.orientation.z,
            self.current_imu.orientation.w
        )

        # Calculate desired joint adjustments for balance
        desired_roll = 0.0  # Target is upright
        desired_pitch = 0.0  # Target is upright

        # Compute balance corrections using PID
        roll_correction = self.roll_pid.compute(desired_roll, roll)
        pitch_correction = self.pitch_pid.compute(desired_pitch, pitch)

        # Generate joint commands for balance
        balance_commands = self.compute_balance_joints(roll_correction, pitch_correction)

        # Publish balance commands
        self.publish_balance_commands(balance_commands)

    def quaternion_to_euler(self, x, y, z, w):
        """Convert quaternion to Euler angles"""
        # Roll (x-axis rotation)
        sinr_cosp = 2 * (w * x + y * z)
        cosr_cosp = 1 - 2 * (x * x + y * y)
        roll = math.atan2(sinr_cosp, cosr_cosp)

        # Pitch (y-axis rotation)
        sinp = 2 * (w * y - z * x)
        if abs(sinp) >= 1:
            pitch = math.copysign(math.pi / 2, sinp)  # Use 90 degrees if out of range
        else:
            pitch = math.asin(sinp)

        # Yaw (z-axis rotation)
        siny_cosp = 2 * (w * z + x * y)
        cosy_cosp = 1 - 2 * (y * y + z * z)
        yaw = math.atan2(siny_cosp, cosy_cosp)

        return roll, pitch, yaw

    def compute_balance_joints(self, roll_correction, pitch_correction):
        """Compute joint adjustments for balance"""
        # Simple balance strategy: adjust ankle joints to counteract tilt
        commands = JointState()
        commands.name = ['left_ankle_joint', 'right_ankle_joint',
                        'left_hip_joint', 'right_hip_joint']
        commands.position = [0.0] * len(commands.name)

        # Ankle adjustments for roll balance
        commands.position[0] = roll_correction * 0.5  # Left ankle
        commands.position[1] = -roll_correction * 0.5  # Right ankle

        # Hip adjustments for pitch balance
        commands.position[2] = pitch_correction * 0.3  # Left hip
        commands.position[3] = pitch_correction * 0.3  # Right hip

        return commands

    def publish_balance_commands(self, commands):
        """Publish balance control commands"""
        commands.header.stamp = self.get_clock().now().to_msg()
        self.joint_cmd_pub.publish(commands)

class PIDController:
    """Simple PID controller for balance control"""
    def __init__(self, kp=1.0, ki=0.0, kd=0.0):
        self.kp = kp
        self.ki = ki
        self.kd = kd
        self.prev_error = 0.0
        self.integral = 0.0

    def compute(self, setpoint, measurement):
        error = setpoint - measurement
        self.integral += error * 0.01  # dt = 0.01s
        derivative = (error - self.prev_error) / 0.01
        output = self.kp * error + self.ki * self.integral + self.kd * derivative
        self.prev_error = error
        return output

def main(args=None):
    rclpy.init(args=args)
    balance_controller = BalanceController()

    try:
        rclpy.spin(balance_controller)
    except KeyboardInterrupt:
        balance_controller.get_logger().info('Balance controller shutting down...')
    finally:
        balance_controller.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Physics Parameters for Humanoid Models

Proper physics parameters are crucial for realistic humanoid simulation:

### Friction Parameters
```xml
<!-- Friction parameters for humanoid feet -->
<gazebo reference="left_foot">
  <collision>
    <surface>
      <friction>
        <ode>
          <mu>1.0</mu>  <!-- Static friction coefficient -->
          <mu2>1.0</mu2>  <!-- Secondary friction coefficient -->
          <slip1>0.0</slip1>  <!-- Primary slip coefficient -->
          <slip2>0.0</slip2>  <!-- Secondary slip coefficient -->
        </ode>
      </friction>
    </surface>
  </collision>
</gazebo>
```

### Joint Dynamics
```xml
<!-- Joint dynamics for humanoid joints -->
<joint name="left_knee_joint" type="revolute">
  <parent link="left_thigh"/>
  <child link="left_shin"/>
  <origin xyz="0 0 -0.4" rpy="0 0 0"/>
  <axis xyz="0 0 1"/>
  <limit lower="0" upper="2.36" effort="30" velocity="2"/>
  <dynamics damping="0.1" friction="0.01"/>
</joint>

<gazebo reference="left_knee_joint">
  <implicitSpringDamper>1</implicitSpringDamper>
  <provideFeedback>1</provideFeedback>
  <axis>
    <dynamics>
      <damping>0.1</damping>
      <friction>0.01</friction>
      <spring_reference>0</spring_reference>
      <spring_stiffness>100000</spring_stiffness>
    </dynamics>
  </axis>
</gazebo>
```

## Integration with ROS 2 Control Systems

Integrating Gazebo with ROS 2 control systems enables real-time control of humanoid robots:

```xml
<!-- ROS 2 control interface for Gazebo -->
<gazebo>
  <plugin name="gazebo_ros_control" filename="libgazebo_ros_control.so">
    <robotNamespace>/humanoid</robotNamespace>
    <robotSimType>gazebo_ros_control/DefaultRobotHWSim</robotSimType>
    <legacyModeNS>true</legacyModeNS>
  </plugin>
</gazebo>
```

## Summary

This lesson provided a comprehensive overview of Gazebo physics engine fundamentals and their application to bipedal balance simulation for humanoid robots. We explored physics engine configuration, balance control strategies, plugin integration, and the critical parameters needed for realistic humanoid simulation. The next lessons will build on these concepts to explore Unity integration, sensor simulation, and advanced control strategies.

## References

1. Koeneke, J., Reina, G., & Santos, J. (2014). Gazebo: A 3D multi-robot simulator. *Journal of Advanced Robotics*, 28(16), 1109-1110. https://doi.org/10.1080/01691864.2014.947902
2. Tedrake, R., & Manchester, I. R. (2011). LQR kernels for optimal control of bilinear systems. *Proceedings of the 50th IEEE Conference on Decision and Control*, 4385-4390. https://doi.org/10.1109/CDC.2011.6160624
3. Kajita, S., Kanehiro, F., Kaneko, K., Fujiwara, K., Harada, K., Yokoi, K., & Hirukawa, H. (2003). Biped walking pattern generation by using preview control of zero-moment point. *Proceedings of the 2003 IEEE International Conference on Robotics and Automation*, 1649-1655. https://doi.org/10.1109/ROBOT.2003.1241826