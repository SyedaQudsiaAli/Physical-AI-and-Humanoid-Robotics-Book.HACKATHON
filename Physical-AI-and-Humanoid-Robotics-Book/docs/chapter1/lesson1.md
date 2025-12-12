---
sidebar_position: 1
---

# Lesson 1.1: ROS 2 Architecture and Communication Primitives

This lesson covers the fundamentals of ROS 2 architecture and communication primitives for humanoid control.

## Introduction

ROS 2 (Robot Operating System 2) provides the middleware foundation for robot control and communication. This lesson introduces the core concepts of ROS 2 architecture that are essential for humanoid robotics applications. Unlike ROS 1 which used a centralized master architecture, ROS 2 uses a distributed architecture based on Data Distribution Service (DDS) for robust, scalable robot communication.

## Key Concepts

- Nodes, topics, services, and actions
- Communication patterns in ROS 2
- Real-time constraints and message timing
- Integration with humanoid control systems
- Quality of Service (QoS) policies

## ROS 2 Communication Architecture

The ROS 2 architecture uses a distributed system of nodes that communicate through topics, services, and actions. Each communication pattern serves a specific purpose in humanoid robotics:

### Topics and Publishers/Subscribers
Topics provide asynchronous, one-way communication using a publish-subscribe pattern. This is ideal for sensor data streams and continuous control commands.

```python
# Example of a ROS 2 publisher node for humanoid joint control
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from std_msgs.msg import Header
import math

class HumanoidJointController(Node):
    def __init__(self):
        super().__init__('humanoid_joint_controller')
        self.publisher_ = self.create_publisher(JointState, 'joint_states', 10)

        # Timer for control loop at 100Hz
        self.timer_ = self.create_timer(0.01, self.control_loop)

        # Joint names for humanoid model
        self.joint_names = [
            'left_hip_joint', 'left_knee_joint', 'left_ankle_joint',
            'right_hip_joint', 'right_knee_joint', 'right_ankle_joint',
            'left_shoulder_joint', 'left_elbow_joint', 'left_wrist_joint',
            'right_shoulder_joint', 'right_elbow_joint', 'right_wrist_joint'
        ]

        # Initialize joint positions
        self.joint_positions = [0.0] * len(self.joint_names)

    def control_loop(self):
        """Main control loop running at 100Hz"""
        msg = JointState()
        msg.header = Header()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.header.frame_id = 'base_link'
        msg.name = self.joint_names
        msg.position = self.joint_positions
        msg.velocity = [0.0] * len(self.joint_names)
        msg.effort = [0.0] * len(self.joint_names)

        self.publisher_.publish(msg)

def main(args=None):
    rclpy.init(args=args)
    node = HumanoidJointController()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

### Services
Services provide synchronous, request-response communication. Useful for configuration commands or state queries.

```python
# Example service server for humanoid configuration
from example_interfaces.srv import SetBool
import rclpy
from rclpy.node import Node

class HumanoidConfigService(Node):
    def __init__(self):
        super().__init__('humanoid_config_service')
        self.srv = self.create_service(
            SetBool,
            'set_balance_mode',
            self.set_balance_mode_callback
        )
        self.balance_mode_enabled = False

    def set_balance_mode_callback(self, request, response):
        self.balance_mode_enabled = request.data
        response.success = True
        response.message = f'Balance mode set to: {self.balance_mode_enabled}'
        self.get_logger().info(response.message)
        return response
```

### Actions
Actions provide asynchronous, goal-oriented communication with feedback and result. Perfect for complex humanoid behaviors like walking or manipulation.

```python
# Example action server for humanoid walking
from rclpy.action import ActionServer
from rclpy.node import Node
from humanoid_msgs.action import WalkToPose  # hypothetical message type
from geometry_msgs.msg import Pose
import rclpy

class HumanoidWalkActionServer(Node):
    def __init__(self):
        super().__init__('humanoid_walk_action_server')
        self._action_server = ActionServer(
            self,
            WalkToPose,
            'walk_to_pose',
            self.execute_callback)

    def execute_callback(self, goal_handle):
        self.get_logger().info('Executing walk to pose action...')

        # Simulate walking process with feedback
        feedback_msg = WalkToPose.Feedback()

        # Walk to the target pose
        target_pose = goal_handle.request.target_pose
        current_pose = self.get_current_pose()

        # Simulate walking progress
        for i in range(0, 101, 10):
            feedback_msg.current_pose = current_pose
            feedback_msg.distance_remaining = self.calculate_distance(
                current_pose, target_pose)
            goal_handle.publish_feedback(feedback_msg)

            # Update progress
            current_pose = self.interpolate_pose(
                current_pose, target_pose, i/100.0)

            # Sleep to simulate walking
            rclpy.spin_once(self, timeout_sec=0.5)

        # Return result
        result = WalkToPose.Result()
        result.success = True
        result.final_pose = current_pose
        goal_handle.succeed()

        return result
```

## Quality of Service (QoS) Considerations

For humanoid robotics applications, QoS settings are crucial for reliable communication:

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy

# QoS for critical control messages
control_qos = QoSProfile(
    depth=1,
    reliability=ReliabilityPolicy.RELIABLE,
    durability=DurabilityPolicy.VOLATILE
)

# QoS for sensor data (may drop messages if behind)
sensor_qos = QoSProfile(
    depth=5,
    reliability=ReliabilityPolicy.BEST_EFFORT,
    durability=DurabilityPolicy.VOLATILE
)
```

## Real-time Considerations for Humanoid Control

Humanoid robots have strict timing requirements for stable control:

- Joint control loops typically run at 100-1000Hz
- Sensor data processing needs low latency
- Safety systems require immediate response
- Communication must be deterministic

## Summary

This lesson introduced the fundamental ROS 2 architecture concepts that form the basis of humanoid robot control systems. We covered the three main communication patterns (topics, services, actions), demonstrated their implementation with code examples, and discussed QoS considerations for humanoid applications. The next lessons will build on these concepts to create more sophisticated humanoid control systems.

## References

1. ROS 2 Documentation. (n.d.). *ROS 2 User Documentation*. https://docs.ros.org/en/humble/
2. Faconti, P., Merz, J., Timmerman, A., Krawczyk, M., & Vugts, R. (2018). Design and use of the ROS 2 logging system. *Proceedings of the 2018 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)*. https://doi.org/10.1109/IROS.2018.8593718
3. Intuitive Machines. (2021). *Real-time ROS 2 Systems: Performance and Determinism*. https://github.com/ros2/ros2/wiki/Performance-and-Real-time