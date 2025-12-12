---
sidebar_position: 3
---

# Lesson 1.3: Real-Time Joint Control and `rclpy` Bridging

This lesson covers real-time joint control and bridging Python code with ROS controllers using `rclpy`, providing a comprehensive approach to controlling humanoid robot joints with precise timing requirements.

## Introduction

Real-time joint control is essential for humanoid robot operation, requiring precise timing and deterministic behavior to maintain stability and safety. This lesson explores how to implement robust control loops using `rclpy` to interface with ROS 2, with special attention to the timing and safety requirements of humanoid robots.

## Key Concepts

- Real-time constraints and message timing
- `rclpy` for Python → ROS controller bridging
- Control loop implementation with timing precision
- Joint command execution with safety monitoring
- Integration with hardware abstraction layers

## Advanced Control Loop Implementation

Real-time control for humanoid robots requires more sophisticated control loops that handle timing precision, safety monitoring, and proper error handling:

```python
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, DurabilityPolicy
from sensor_msgs.msg import JointState
from control_msgs.msg import JointTrajectoryControllerState
from trajectory_msgs.msg import JointTrajectory, JointTrajectoryPoint
from builtin_interfaces.msg import Duration
from std_msgs.msg import Header
import numpy as np
import time
from collections import deque
import threading

class HumanoidJointController(Node):
    """
    Advanced humanoid joint controller with real-time capabilities
    """
    def __init__(self):
        super().__init__('humanoid_joint_controller')

        # Define humanoid joint names
        self.joint_names = [
            'left_hip_joint', 'left_knee_joint', 'left_ankle_joint',
            'right_hip_joint', 'right_knee_joint', 'right_ankle_joint',
            'left_shoulder_joint', 'left_elbow_joint', 'left_wrist_joint',
            'right_shoulder_joint', 'right_elbow_joint', 'right_wrist_joint'
        ]

        # Initialize joint states
        self.current_positions = np.zeros(len(self.joint_names))
        self.current_velocities = np.zeros(len(self.joint_names))
        self.current_efforts = np.zeros(len(self.joint_names))
        self.desired_positions = np.zeros(len(self.joint_names))
        self.desired_velocities = np.zeros(len(self.joint_names))
        self.desired_efforts = np.zeros(len(self.joint_names))

        # Control parameters
        self.control_rate = 100  # Hz
        self.dt = 1.0 / self.control_rate  # Control loop time step
        self.last_control_time = self.get_clock().now()

        # Safety parameters
        self.max_velocity = 5.0  # rad/s
        self.max_effort = 100.0  # Nm
        self.joint_limits = self._define_joint_limits()

        # Timing monitoring
        self.control_loop_times = deque(maxlen=100)  # Store last 100 loop times
        self.target_loop_time = 0.01  # 10ms target for 100Hz

        # QoS for critical control messages
        control_qos = QoSProfile(
            depth=1,
            reliability=ReliabilityPolicy.RELIABLE,
            durability=DurabilityPolicy.VOLATILE
        )

        # Publishers
        self.joint_state_pub = self.create_publisher(JointState, 'joint_states', control_qos)
        self.controller_state_pub = self.create_publisher(
            JointTrajectoryControllerState,
            'controller_state',
            control_qos
        )

        # Subscribers
        self.joint_command_sub = self.create_subscription(
            JointState,
            'joint_commands',
            self.joint_command_callback,
            control_qos
        )

        # Create timer for control loop
        self.control_timer = self.create_timer(
            self.dt,
            self.control_loop,
            clock=self.get_clock()
        )

        # Initialize control loop timing
        self.get_logger().info(f'Humanoid joint controller initialized at {self.control_rate}Hz')

    def _define_joint_limits(self):
        """
        Define joint limits for humanoid robot joints
        """
        limits = {}
        for i, name in enumerate(self.joint_names):
            if 'hip' in name:
                limits[name] = {'min': -1.57, 'max': 1.57}  # Hip joints
            elif 'knee' in name:
                limits[name] = {'min': 0, 'max': 2.36}      # Knee joints (flex only)
            elif 'ankle' in name:
                limits[name] = {'min': -0.79, 'max': 0.79}  # Ankle joints
            elif 'shoulder' in name:
                limits[name] = {'min': -1.57, 'max': 1.57}  # Shoulder joints
            elif 'elbow' in name:
                if 'left' in name:
                    limits[name] = {'min': -2.36, 'max': 0}  # Left elbow (flex only)
                else:
                    limits[name] = {'min': 0, 'max': 2.36}   # Right elbow (flex only)
            elif 'wrist' in name:
                limits[name] = {'min': -1.57, 'max': 1.57}  # Wrist joints
        return limits

    def joint_command_callback(self, msg):
        """
        Handle incoming joint commands
        """
        for i, name in enumerate(msg.name):
            try:
                idx = self.joint_names.index(name)
                if len(msg.position) > i:
                    # Apply joint limits
                    pos = np.clip(
                        msg.position[i],
                        self.joint_limits[name]['min'],
                        self.joint_limits[name]['max']
                    )
                    self.desired_positions[idx] = pos

                if len(msg.velocity) > i:
                    vel = np.clip(msg.velocity[i], -self.max_velocity, self.max_velocity)
                    self.desired_velocities[idx] = vel

                if len(msg.effort) > i:
                    eff = np.clip(msg.effort[i], -self.max_effort, self.max_effort)
                    self.desired_efforts[idx] = eff

            except ValueError:
                self.get_logger().warn(f'Unknown joint: {name}')

    def control_loop(self):
        """
        Main control loop running at specified frequency
        """
        start_time = time.time()

        # Get current time for timing calculations
        current_time = self.get_clock().now()
        dt_actual = (current_time - self.last_control_time).nanoseconds / 1e9
        self.last_control_time = current_time

        # Execute control algorithm
        self.execute_control_algorithm(dt_actual)

        # Publish joint states
        self.publish_joint_states(current_time)

        # Publish controller state
        self.publish_controller_state(current_time)

        # Monitor timing performance
        loop_time = time.time() - start_time
        self.control_loop_times.append(loop_time)

        # Log timing warnings if needed
        if loop_time > self.target_loop_time * 1.5:  # 50% over target
            self.get_logger().warn(
                f'Control loop exceeded timing: {loop_time:.4f}s vs {self.target_loop_time:.4f}s'
            )

        # Safety checks
        self.perform_safety_checks()

    def execute_control_algorithm(self, dt):
        """
        Execute the control algorithm for all joints
        This is where PID control or other control strategies would be implemented
        """
        # Simple proportional control for demonstration
        kp = 10.0  # Proportional gain
        kv = 2.0   # Velocity damping gain

        for i in range(len(self.joint_names)):
            # Calculate error
            error = self.desired_positions[i] - self.current_positions[i]

            # Simple PD control
            control_effort = kp * error - kv * self.current_velocities[i]

            # Apply effort limits
            control_effort = np.clip(control_effort, -self.max_effort, self.max_effort)

            # Update current state (in simulation, this would come from the physics engine)
            self.current_velocities[i] = self.desired_velocities[i] + control_effort * dt
            self.current_positions[i] += self.current_velocities[i] * dt
            self.current_efforts[i] = control_effort

    def publish_joint_states(self, timestamp):
        """
        Publish current joint states
        """
        msg = JointState()
        msg.header = Header()
        msg.header.stamp = timestamp.to_msg()
        msg.header.frame_id = 'base_link'
        msg.name = self.joint_names
        msg.position = self.current_positions.tolist()
        msg.velocity = self.current_velocities.tolist()
        msg.effort = self.current_efforts.tolist()

        self.joint_state_pub.publish(msg)

    def publish_controller_state(self, timestamp):
        """
        Publish detailed controller state
        """
        msg = JointTrajectoryControllerState()
        msg.header = Header()
        msg.header.stamp = timestamp.to_msg()
        msg.joint_names = self.joint_names
        msg.desired.positions = self.desired_positions.tolist()
        msg.desired.velocities = self.desired_velocities.tolist()
        msg.desired.accelerations = [0.0] * len(self.joint_names)  # Simplified
        msg.actual.positions = self.current_positions.tolist()
        msg.actual.velocities = self.current_velocities.tolist()
        msg.error.positions = (self.desired_positions - self.current_positions).tolist()
        msg.error.velocities = (self.desired_velocities - self.current_velocities).tolist()

        self.controller_state_pub.publish(msg)

    def perform_safety_checks(self):
        """
        Perform safety checks during control loop
        """
        # Check for excessive joint velocities
        if np.any(np.abs(self.current_velocities) > self.max_velocity * 1.1):
            self.get_logger().error('Excessive joint velocity detected - triggering safety stop')
            self.safety_stop()

        # Check for joint limit violations
        for i, name in enumerate(self.joint_names):
            if (self.current_positions[i] < self.joint_limits[name]['min'] or
                self.current_positions[i] > self.joint_limits[name]['max']):
                self.get_logger().error(f'Joint limit violation: {name} = {self.current_positions[i]}')
                self.safety_stop()
                break

    def safety_stop(self):
        """
        Emergency safety stop procedure
        """
        self.get_logger().warn('Safety stop activated - zeroing all joint commands')
        self.desired_positions = np.zeros(len(self.joint_names))
        self.desired_velocities = np.zeros(len(self.joint_names))
        self.desired_efforts = np.zeros(len(self.joint_names))

    def get_average_loop_time(self):
        """
        Get average control loop execution time
        """
        if len(self.control_loop_times) > 0:
            return sum(self.control_loop_times) / len(self.control_loop_times)
        return 0.0

def main(args=None):
    rclpy.init(args=args)

    controller = HumanoidJointController()

    try:
        rclpy.spin(controller)
    except KeyboardInterrupt:
        controller.get_logger().info('Interrupted, shutting down...')
    finally:
        controller.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## PID Control Implementation

For more sophisticated control, a PID controller can be implemented:

```python
class PIDController:
    """
    PID controller for humanoid joint control
    """
    def __init__(self, kp=1.0, ki=0.0, kd=0.0, dt=0.01):
        self.kp = kp
        self.ki = ki
        self.kd = kd
        self.dt = dt

        self.prev_error = 0.0
        self.integral = 0.0
        self.derivative = 0.0

    def compute(self, setpoint, measurement):
        """
        Compute PID output
        """
        error = setpoint - measurement

        # Proportional term
        p_term = self.kp * error

        # Integral term
        self.integral += error * self.dt
        i_term = self.ki * self.integral

        # Derivative term
        self.derivative = (error - self.prev_error) / self.dt
        d_term = self.kd * self.derivative

        # Store current error for next iteration
        self.prev_error = error

        # Compute output
        output = p_term + i_term + d_term

        return output

# Example usage in the joint controller
class AdvancedJointController(Node):
    def __init__(self):
        super().__init__('advanced_joint_controller')

        # Initialize PID controllers for each joint
        self.pid_controllers = []
        for _ in range(12):  # 12 joints for example humanoid
            self.pid_controllers.append(PIDController(kp=10.0, ki=0.1, kd=0.5, dt=0.01))

        # Other initialization code...
```

## Real-Time Considerations for Humanoid Control

Real-time control of humanoid joints requires special attention to:

### Timing Precision
- Control loops typically run at 100-1000Hz for stable control
- Deterministic timing is crucial for stability
- Jitter in control timing can cause instability

### Safety Systems
- Joint limit monitoring
- Velocity and effort limits
- Emergency stop procedures
- Balance and stability monitoring

### Performance Optimization
- Minimize computation in control loops
- Use efficient data structures
- Avoid memory allocation in loops
- Consider real-time kernel for critical applications

## Integration with Hardware Abstraction

For real hardware, controllers integrate with hardware abstraction layers:

```python
class HardwareInterface:
    """
    Interface to physical hardware
    """
    def __init__(self):
        # Initialize hardware communication (CAN, EtherCAT, etc.)
        pass

    def read_joint_states(self):
        """
        Read current joint positions, velocities, and efforts from hardware
        """
        # Implementation depends on hardware interface
        pass

    def write_joint_commands(self, commands):
        """
        Send joint commands to hardware
        """
        # Implementation depends on hardware interface
        pass

class HardwareJointController(HumanoidJointController):
    def __init__(self):
        super().__init__()
        self.hardware_interface = HardwareInterface()

    def control_loop(self):
        # Read current state from hardware
        hw_state = self.hardware_interface.read_joint_states()
        self.current_positions = hw_state.positions
        self.current_velocities = hw_state.velocities
        self.current_efforts = hw_state.efforts

        # Execute control algorithm
        self.execute_control_algorithm(self.dt)

        # Send commands to hardware
        self.hardware_interface.write_joint_commands(self.desired_positions)

        # Continue with publishing and safety checks...
```

## Summary

This lesson provided a comprehensive overview of real-time joint control for humanoid robots, including advanced control loop implementations, safety considerations, and integration with hardware abstraction layers. We explored the critical timing requirements for humanoid control, safety systems, and performance optimization techniques. The next chapters will build on these concepts to integrate joint control with perception, navigation, and cognition systems.

## References

1. Quigley, M., Gerkey, B., & Smart, W. D. (2009). Programming with ROS. *Proceedings of the 2009 IEEE/RSJ International Conference on Intelligent Robots and Systems*, 4729-4735. https://doi.org/10.1109/IROS.2009.5354432
2. Vijayakumar, S., D'Souza, A., & Schaal, S. (2005). Incremental on-line learning: A framework and application to data-driven control. *Proceedings of the 2005 IEEE/RSJ International Conference on Intelligent Robots and Systems*, 3312-3317. https://doi.org/10.1109/IROS.2005.1545326
3. Sciavicco, L., & Siciliano, B. (2012). *Modelling and control of robot manipulators* (2nd ed.). Springer Science & Business Media. https://doi.org/10.1007/978-1-4471-0449-0