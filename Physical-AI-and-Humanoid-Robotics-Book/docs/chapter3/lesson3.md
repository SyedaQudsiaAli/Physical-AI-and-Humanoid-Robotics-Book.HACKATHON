---
sidebar_position: 3
---

# Lesson 3.3: Nav2: Global Path Planning and Footstep Planning for Bipeds

This lesson covers Nav2 for global path planning and specialized footstep planning for bipedal humanoid robots, providing a comprehensive understanding of navigation systems tailored to humanoid locomotion constraints.

## Introduction

Navigation is a critical capability for humanoid robots, requiring specialized path planning that accounts for bipedal locomotion constraints, balance requirements, and anthropomorphic movement patterns. This lesson explores Nav2 integration with humanoid-specific modifications, including footstep planning, dynamic path adjustment for balance, and specialized navigation behaviors for bipedal locomotion. We'll examine how to configure Nav2 for humanoid robots while considering their unique kinematic and dynamic constraints.

## Key Concepts

- Nav2 for global path planning with humanoid-specific constraints
- Footstep planning algorithms for bipedal locomotion
- Dynamic path adjustment for balance maintenance
- Navigation in humanoid-specific environments
- Integration with ROS 2 navigation stack for bipedal robots
- Stability-aware path planning and execution
- Humanoid kinematic constraints in navigation

## Advanced Nav2 Configuration for Humanoid Robots

Configuring Nav2 for humanoid robots requires careful consideration of their unique locomotion characteristics:

```yaml
# Nav2 configuration for humanoid robot
bt_navigator:
  ros__parameters:
    use_sim_time: False
    global_frame: map
    robot_base_frame: base_link
    odom_topic: /odom
    bt_loop_duration: 10
    default_server_timeout: 20
    enable_groot_monitoring: True
    groot_zmq_publisher_port: 1666
    groot_zmq_server_port: 1667
    # Use customized behavior tree for humanoid navigation
    default_nav_to_pose_bt_xml: "humanoid_nav_to_pose.xml"
    default_nav_through_poses_bt_xml: "humanoid_nav_through_poses.xml"
    plugin_lib_names:
    - nav2_compute_path_to_pose_action_bt_node
    - nav2_compute_path_through_poses_action_bt_node
    - nav2_smooth_path_action_bt_node
    - nav2_follow_path_action_bt_node
    - nav2_spin_action_bt_node
    - nav2_wait_action_bt_node
    - nav2_assisted_teleop_action_bt_node
    - nav2_back_up_action_bt_node
    - nav2_drive_on_heading_bt_node
    - nav2_clear_costmap_service_bt_node
    - nav2_is_stuck_condition_bt_node
    - nav2_have_reached_goal_condition_bt_node
    - nav2_have_passed_goal_condition_bt_node
    - nav2_is_path_valid_condition_bt_node
    - nav2_initial_pose_received_condition_bt_node
    - nav2_reinitialize_global_localization_service_bt_node
    - nav2_rate_controller_bt_node
    - nav2_distance_controller_bt_node
    - nav2_speed_controller_bt_node
    - nav2_truncate_path_action_bt_node
    - nav2_truncate_path_local_action_bt_node
    - nav2_goal_updater_node_bt_node
    - nav2_recovery_node_bt_node
    - nav2_pipeline_sequence_bt_node
    - nav2_round_robin_node_bt_node
    - nav2_transformer_node_bt_node
    - nav2_get_path_on_approach_adder_node_bt_node
    - nav2_modify_path_for_planning_scene_node_bt_node
    - nav2_modify_path_for_humanoid_locomotion_node_bt_node  # Custom humanoid-specific node

controller_server:
  ros__parameters:
    use_sim_time: False
    controller_frequency: 20.0
    min_x_velocity_threshold: 0.001
    min_y_velocity_threshold: 0.5
    min_theta_velocity_threshold: 0.001
    # Humanoid-specific controller parameters
    max_linear_velocity: 0.8    # Slower for stability
    max_angular_velocity: 0.5
    min_linear_velocity: 0.1    # Minimum for smooth walking
    min_angular_velocity: 0.1
    # Footstep planning integration
    use_footstep_planning: True
    step_duration: 0.8          # Time per step
    step_length: 0.3            # Typical humanoid step length
    step_width: 0.2             # Distance between feet
    controller_plugins: ["FollowPath"]

    # Humanoid-specific FollowPath controller
    FollowPath:
      plugin: "nav2_mppi::HumanoidLocalPlanner"  # Custom humanoid local planner
      lookahead_dist: 0.6
      lookahead_time: 1.0
      transform_tolerance: 0.1
      min_vel_x: 0.1
      max_vel_x: 0.8
      max_vel_theta: 0.5
      min_vel_theta: 0.1
      min_speed_xy: 0.1
      max_speed_xy: 0.8
      max_acc_xy: 0.5
      max_decel_xy: 0.8
      max_acc_theta: 0.5
      max_decel_theta: 0.8
      speed_computed_xyz_threshold: 0.01
      yaw_goal_tolerance: 0.1
      xy_goal_tolerance: 0.2
      stateful: True
      oscillation_timeout: 0.0
      oscillation_distance: 0.0
      # Balance-aware parameters
      balance_margin: 0.15      # Extra margin for stability
      center_of_mass_offset: 0.85  # Height of COM relative to feet

local_costmap:
  local_costmap:
    ros__parameters:
      update_frequency: 5.0
      publish_frequency: 2.0
      global_frame: odom
      robot_base_frame: base_link
      use_sim_time: False
      rolling_window: true
      width: 6
      height: 6
      resolution: 0.05
      # Humanoid-specific inflation
      inflation_radius: 0.6     # Larger for humanoid footprint
      cost_scaling_factor: 3.0
      footprint: "[[-0.3, -0.2], [-0.3, 0.2], [0.3, 0.2], [0.3, -0.2]]"  # Larger humanoid footprint
      plugins: ["obstacle_layer", "voxel_layer", "inflation_layer"]
      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        cost_scaling_factor: 3.0
        inflation_radius: 0.6
      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: True
        observation_sources: scan
        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: True
          marking: True
          data_type: "LaserScan"
          raytrace_range: 3.0
          obstacle_range: 2.5
          inflate_obstacles: True
      voxel_layer:
        plugin: "nav2_costmap_2d::VoxelLayer"
        enabled: True
        publish_voxel_map: False
        origin_z: 0.0
        z_resolution: 0.2
        z_voxels: 8
        max_obstacle_height: 2.0
        mark_threshold: 0
        observation_sources: pointcloud
        pointcloud:
          topic: /pointcloud
          max_obstacle_height: 2.0
          clearing: True
          marking: True
          data_type: "PointCloud2"
          min_obstacle_height: 0.0
          obstacle_range: 2.5
          raytrace_range: 3.0

global_costmap:
  global_costmap:
    ros__parameters:
      update_frequency: 1.0
      publish_frequency: 1.0
      global_frame: map
      robot_base_frame: base_link
      use_sim_time: False
      robot_radius: 0.4  # Humanoid radius
      resolution: 0.05
      track_unknown_space: true
      plugins: ["static_layer", "obstacle_layer", "inflation_layer"]
      obstacle_layer:
        plugin: "nav2_costmap_2d::ObstacleLayer"
        enabled: True
        observation_sources: scan
        scan:
          topic: /scan
          max_obstacle_height: 2.0
          clearing: True
          marking: True
          data_type: "LaserScan"
          raytrace_range: 5.0
          obstacle_range: 4.0
          track_unknown_space: true
          footprint_clearing_enabled: true
      static_layer:
        plugin: "nav2_costmap_2d::StaticLayer"
        map_subscribe_transient_local: True
      inflation_layer:
        plugin: "nav2_costmap_2d::InflationLayer"
        cost_scaling_factor: 3.0
        inflation_radius: 1.0
```

## Advanced Humanoid Navigation Node with Footstep Planning

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
from nav2_msgs.action import NavigateToPose, NavigateThroughPoses
from geometry_msgs.msg import PoseStamped, Point, Quaternion
from sensor_msgs.msg import LaserScan, PointCloud2
from tf2_ros import TransformListener, Buffer
from visualization_msgs.msg import Marker, MarkerArray
import numpy as np
import math
from scipy.spatial.transform import Rotation as R
import threading
from collections import deque
import time

class HumanoidNav2Node(Node):
    """
    Advanced Nav2 node for humanoid robots with footstep planning
    """
    def __init__(self):
        super().__init__('humanoid_nav2_node')

        # Initialize TF2
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

        # Action clients
        self.nav_to_pose_client = ActionClient(
            self, NavigateToPose, 'navigate_to_pose')

        self.nav_through_poses_client = ActionClient(
            self, NavigateThroughPoses, 'navigate_through_poses')

        # Publishers
        self.footstep_plan_pub = self.create_publisher(
            MarkerArray, '/humanoid/footsteps', 10)
        self.balance_marker_pub = self.create_publisher(
            Marker, '/humanoid/balance_center', 10)

        # Subscribers
        qos_profile = QoSProfile(
            depth=10,
            reliability=ReliabilityPolicy.BEST_EFFORT,
            history=HistoryPolicy.KEEP_LAST
        )
        self.laser_sub = self.create_subscription(
            LaserScan, '/scan', self.laser_callback, qos_profile)
        self.pc_sub = self.create_subscription(
            PointCloud2, '/pointcloud', self.pointcloud_callback, qos_profile)

        # Humanoid-specific parameters
        self.step_length = 0.3  # meters
        self.step_width = 0.2   # meters
        self.step_duration = 0.8  # seconds
        self.balance_margin = 0.15  # meters
        self.com_height = 0.85  # Center of mass height
        self.max_step_height = 0.1  # Maximum step over obstacles

        # Navigation state
        self.current_pose = None
        self.navigation_active = False
        self.footstep_queue = deque()
        self.support_polygon = None
        self.balance_center = None

        # Footstep planning parameters
        self.left_foot_pose = None
        self.right_foot_pose = None
        self.current_support_foot = 'left'  # Which foot is currently supporting weight

        # Initialize foot poses
        self.initialize_foot_poses()

        self.get_logger().info('Humanoid Nav2 Node initialized with footstep planning')

    def initialize_foot_poses(self):
        """
        Initialize foot poses based on current robot pose
        """
        # Initially, both feet are aligned with robot base
        if self.current_pose:
            # Set initial foot positions relative to robot base
            self.left_foot_pose = self.offset_pose(self.current_pose, 0.0, self.step_width/2, 0.0)
            self.right_foot_pose = self.offset_pose(self.current_pose, 0.0, -self.step_width/2, 0.0)
        else:
            # Default poses
            self.left_foot_pose = PoseStamped()
            self.right_foot_pose = PoseStamped()

    def offset_pose(self, base_pose, x_offset, y_offset, z_offset):
        """
        Create a pose offset from base pose
        """
        offset_pose = PoseStamped()
        offset_pose.header = base_pose.header
        offset_pose.pose.position.x = base_pose.pose.position.x + x_offset
        offset_pose.pose.position.y = base_pose.pose.position.y + y_offset
        offset_pose.pose.position.z = base_pose.pose.position.z + z_offset
        offset_pose.pose.orientation = base_pose.pose.orientation
        return offset_pose

    def laser_callback(self, msg):
        """
        Handle laser scan data for navigation
        """
        # Process laser data for obstacle detection
        # This would be used for dynamic path adjustment
        pass

    def pointcloud_callback(self, msg):
        """
        Handle point cloud data for 3D navigation
        """
        # Process 3D point cloud for terrain analysis
        # This would be used for footstep planning on uneven terrain
        pass

    def navigate_to_pose(self, target_x, target_y, target_yaw):
        """
        Navigate humanoid robot to target pose with footstep planning
        """
        if not self.nav_to_pose_client.wait_for_server(timeout_sec=1.0):
            self.get_logger().error('NavigateToPose action server not available')
            return False

        # Create navigation goal
        goal_msg = NavigateToPose.Goal()
        goal_msg.pose.header.frame_id = 'map'
        goal_msg.pose.pose.position.x = target_x
        goal_msg.pose.pose.position.y = target_y
        goal_msg.pose.pose.position.z = 0.0

        # Convert yaw to quaternion
        q = R.from_euler('z', target_yaw).as_quat()
        goal_msg.pose.pose.orientation.x = q[0]
        goal_msg.pose.pose.orientation.y = q[1]
        goal_msg.pose.pose.orientation.z = q[2]
        goal_msg.pose.pose.orientation.w = q[3]

        # Send navigation goal
        self.navigation_active = True
        goal_future = self.nav_to_pose_client.send_goal_async(
            goal_msg,
            feedback_callback=self.navigation_feedback_callback)

        goal_future.add_done_callback(self.navigation_goal_response_callback)

        return True

    def navigate_through_poses(self, poses):
        """
        Navigate humanoid robot through a series of poses with footstep planning
        """
        if not self.nav_through_poses_client.wait_for_server(timeout_sec=1.0):
            self.get_logger().error('NavigateThroughPoses action server not available')
            return False

        # Create navigation goal
        goal_msg = NavigateThroughPoses.Goal()
        goal_msg.poses = poses
        goal_msg.behavior_tree = ''  # Use default BT

        # Send navigation goal
        self.navigation_active = True
        goal_future = self.nav_through_poses_client.send_goal_async(
            goal_msg,
            feedback_callback=self.navigation_feedback_callback)

        goal_future.add_done_callback(self.navigation_goal_response_callback)

        return True

    def navigation_feedback_callback(self, feedback_msg):
        """
        Handle navigation feedback
        """
        if hasattr(feedback_msg, 'current_pose'):
            self.current_pose = feedback_msg.current_pose

        if hasattr(feedback_msg, 'distance_remaining'):
            self.get_logger().info(
                f'Navigation progress: {feedback_msg.distance_remaining:.2f}m remaining')

        # Update footstep plan based on current progress
        self.update_footstep_plan()

    def navigation_goal_response_callback(self, future):
        """
        Handle navigation goal response
        """
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().info('Navigation goal rejected')
            self.navigation_active = False
            return

        self.get_logger().info('Navigation goal accepted')
        result_future = goal_handle.get_result_async()
        result_future.add_done_callback(self.navigation_result_callback)

    def navigation_result_callback(self, future):
        """
        Handle navigation result
        """
        result = future.result().result
        status = future.result().status

        self.navigation_active = False

        if status == 4:  # STATUS_SUCCEEDED
            self.get_logger().info('Navigation completed successfully')
        else:
            self.get_logger().info(f'Navigation failed with status: {status}')

        # Clear footstep queue
        self.footstep_queue.clear()

    def update_footstep_plan(self):
        """
        Update footstep plan based on current navigation state
        """
        if not self.current_pose or not self.navigation_active:
            return

        # Calculate footstep plan based on current pose and navigation path
        planned_steps = self.plan_footsteps()

        # Publish footsteps for visualization
        self.publish_footsteps(planned_steps)

        # Update support polygon and balance center
        self.update_balance_metrics()

    def plan_footsteps(self):
        """
        Plan footsteps for humanoid locomotion
        """
        if not self.current_pose:
            return []

        # Simplified footstep planning algorithm
        # In a real implementation, this would use more sophisticated planning
        footsteps = []

        # Calculate desired next step based on navigation direction
        if self.current_support_foot == 'left':
            # Plan right foot step
            next_step = self.calculate_next_step('right')
            self.current_support_foot = 'right'
        else:
            # Plan left foot step
            next_step = self.calculate_next_step('left')
            self.current_support_foot = 'left'

        if next_step:
            footsteps.append(next_step)

        # Plan additional steps if needed
        for i in range(1, 5):  # Plan next 4 steps
            if self.current_support_foot == 'left':
                next_step = self.calculate_next_step('right')
                self.current_support_foot = 'right'
            else:
                next_step = self.calculate_next_step('left')
                self.current_support_foot = 'left'

            if next_step:
                # Add small offset for stability
                next_step.pose.position.x += np.random.uniform(-0.02, 0.02)
                next_step.pose.position.y += np.random.uniform(-0.02, 0.02)
                footsteps.append(next_step)

        return footsteps

    def calculate_next_step(self, foot_type):
        """
        Calculate the next step for the specified foot
        """
        if not self.current_pose:
            return None

        # Calculate step based on current navigation direction
        # This is a simplified approach - in reality, this would involve
        # more complex gait planning algorithms
        next_step = PoseStamped()
        next_step.header.frame_id = 'map'
        next_step.header.stamp = self.get_clock().now().to_msg()

        # For now, just advance in the navigation direction
        # with appropriate foot placement
        nav_direction = self.estimate_navigation_direction()

        step_offset = self.step_length
        lateral_offset = self.step_width / 2.0

        if foot_type == 'left':
            # Offset to the left
            next_step.pose.position.x = self.current_pose.pose.position.x + nav_direction[0] * step_offset
            next_step.pose.position.y = self.current_pose.pose.position.y + nav_direction[1] * step_offset + lateral_offset
        else:
            # Offset to the right
            next_step.pose.position.x = self.current_pose.pose.position.x + nav_direction[0] * step_offset
            next_step.pose.position.y = self.current_pose.pose.position.y + nav_direction[1] * step_offset - lateral_offset

        # Maintain same orientation as current pose
        next_step.pose.orientation = self.current_pose.pose.orientation

        return next_step

    def estimate_navigation_direction(self):
        """
        Estimate the current navigation direction
        """
        # This would normally come from the path planner
        # For now, we'll use a simple heuristic
        if self.current_pose:
            # In a real implementation, this would be derived from the planned path
            # For demonstration, assume forward movement
            return [1.0, 0.0]  # Moving forward
        return [0.0, 0.0]

    def publish_footsteps(self, footsteps):
        """
        Publish footsteps for visualization
        """
        marker_array = MarkerArray()

        for i, step in enumerate(footsteps):
            marker = Marker()
            marker.header = step.header
            marker.ns = 'footsteps'
            marker.id = i
            marker.type = Marker.CYLINDER
            marker.action = Marker.ADD

            marker.pose = step.pose
            marker.scale.x = 0.15  # Foot length
            marker.scale.y = 0.1   # Foot width
            marker.scale.z = 0.02  # Foot height

            # Color based on foot type
            if i % 2 == 0:  # Left foot
                marker.color.r = 1.0
                marker.color.g = 0.0
                marker.color.b = 0.0
            else:  # Right foot
                marker.color.r = 0.0
                marker.color.g = 0.0
                marker.color.b = 1.0
            marker.color.a = 0.8

            marker_array.markers.append(marker)

        self.footstep_plan_pub.publish(marker_array)

    def update_balance_metrics(self):
        """
        Update support polygon and balance center
        """
        if not self.left_foot_pose or not self.right_foot_pose:
            return

        # Calculate support polygon (convex hull of feet positions)
        left_pos = self.left_foot_pose.pose.position
        right_pos = self.right_foot_pose.pose.position

        # For biped, support polygon is line segment between feet
        self.support_polygon = [
            [left_pos.x, left_pos.y],
            [right_pos.x, right_pos.y]
        ]

        # Calculate balance center (midpoint between feet)
        self.balance_center = Point()
        self.balance_center.x = (left_pos.x + right_pos.x) / 2.0
        self.balance_center.y = (left_pos.y + right_pos.y) / 2.0
        self.balance_center.z = min(left_pos.z, right_pos.z)  # Ground level

        # Publish balance center for visualization
        self.publish_balance_center()

    def publish_balance_center(self):
        """
        Publish balance center for visualization
        """
        if not self.balance_center:
            return

        marker = Marker()
        marker.header.frame_id = 'map'
        marker.header.stamp = self.get_clock().now().to_msg()
        marker.ns = 'balance'
        marker.id = 0
        marker.type = Marker.SPHERE
        marker.action = Marker.ADD

        marker.pose.position = self.balance_center
        marker.pose.orientation.w = 1.0
        marker.scale.x = 0.05
        marker.scale.y = 0.05
        marker.scale.z = 0.05

        marker.color.r = 0.0
        marker.color.g = 1.0
        marker.color.b = 0.0
        marker.color.a = 0.9

        self.balance_marker_pub.publish(marker)

    def check_balance_feasibility(self, proposed_pose):
        """
        Check if proposed pose maintains balance
        """
        if not self.support_polygon or not self.balance_center:
            return True  # Can't evaluate, assume feasible

        # Calculate new projected center of mass
        # This is a simplified check - in reality, this would involve
        # complex dynamics and stability analysis
        com_projected = [
            proposed_pose.pose.position.x,
            proposed_pose.pose.position.y
        ]

        # Check if COM projection is within balance margin of support polygon
        # For now, just check distance to balance center
        distance_to_balance = math.sqrt(
            (com_projected[0] - self.balance_center.x)**2 +
            (com_projected[1] - self.balance_center.y)**2
        )

        return distance_to_balance <= self.balance_margin

def main(args=None):
    rclpy.init(args=args)
    nav_node = HumanoidNav2Node()

    try:
        rclpy.spin(nav_node)
    except KeyboardInterrupt:
        nav_node.get_logger().info('Humanoid Nav2 Node shutting down...')
    finally:
        nav_node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Footstep Planning Algorithms for Humanoid Robots

Advanced footstep planning algorithms specifically designed for bipedal locomotion:

```python
import numpy as np
from scipy.spatial.distance import cdist
from scipy.optimize import minimize
import matplotlib.pyplot as plt
from dataclasses import dataclass
from typing import List, Tuple
import math

@dataclass
class FootPose:
    """
    Data class for foot pose representation
    """
    x: float
    y: float
    theta: float  # Orientation in radians
    step_type: str  # 'left', 'right', or 'support'

@dataclass
class StepConstraint:
    """
    Constraint for footstep planning
    """
    min_distance: float = 0.1
    max_distance: float = 0.5
    max_rotation: float = 0.5  # radians
    balance_margin: float = 0.15

class FootstepPlanner:
    """
    Advanced footstep planner for humanoid robots
    """
    def __init__(self, step_length=0.3, step_width=0.2, com_height=0.85):
        self.step_length = step_length
        self.step_width = step_width
        self.com_height = com_height

        # Initialize with neutral stance
        self.left_foot = FootPose(0.0, step_width/2, 0.0, 'left')
        self.right_foot = FootPose(0.0, -step_width/2, 0.0, 'right')
        self.support_foot = 'left'  # Which foot supports weight

        self.constraints = StepConstraint()
        self.trajectory = []

    def plan_steps_to_target(self, target_x, target_y, target_theta=0.0, num_steps=10):
        """
        Plan footstep sequence to reach target position
        """
        steps = []

        # Calculate required steps based on distance
        dist_to_target = math.sqrt((target_x - self.left_foot.x)**2 + (target_y - self.left_foot.y)**2)
        required_steps = max(1, int(dist_to_target / self.step_length) + 1)

        # Generate intermediate waypoints
        for i in range(min(required_steps, num_steps)):
            # Interpolate toward target
            ratio = (i + 1) / min(required_steps, num_steps)
            waypoint_x = self.left_foot.x + (target_x - self.left_foot.x) * ratio
            waypoint_y = self.left_foot.y + (target_y - self.left_foot.y) * ratio
            waypoint_theta = self.left_foot.theta + (target_theta - self.left_foot.theta) * ratio

            # Generate footstep based on current support foot
            if self.support_foot == 'left':
                next_step = self.generate_right_step(waypoint_x, waypoint_y, waypoint_theta)
                self.support_foot = 'right'
            else:
                next_step = self.generate_left_step(waypoint_x, waypoint_y, waypoint_theta)
                self.support_foot = 'left'

            # Validate step
            if self.validate_step(next_step):
                steps.append(next_step)
                # Update current foot positions
                if next_step.step_type == 'left':
                    self.left_foot = next_step
                else:
                    self.right_foot = next_step
            else:
                # If step is invalid, try alternative placement
                alternative_step = self.generate_alternative_step(
                    next_step, self.support_foot)
                if self.validate_step(alternative_step):
                    steps.append(alternative_step)
                    if alternative_step.step_type == 'left':
                        self.left_foot = alternative_step
                    else:
                        self.right_foot = alternative_step
                else:
                    # If still invalid, skip this step and continue
                    continue

        return steps

    def generate_left_step(self, target_x, target_y, target_theta):
        """
        Generate next left foot step
        """
        # Calculate offset from current right foot position
        dx = target_x - self.right_foot.x
        dy = target_y - self.right_foot.y

        # Normalize direction vector
        dist = math.sqrt(dx*dx + dy*dy)
        if dist > 0:
            dx /= dist
            dy /= dist

        # Calculate new position with step length and width considerations
        new_x = self.right_foot.x + dx * self.step_length
        new_y = self.right_foot.y + dy * self.step_length + self.step_width/2  # Offset to left
        new_theta = target_theta  # Match target orientation

        return FootPose(new_x, new_y, new_theta, 'left')

    def generate_right_step(self, target_x, target_y, target_theta):
        """
        Generate next right foot step
        """
        # Calculate offset from current left foot position
        dx = target_x - self.left_foot.x
        dy = target_y - self.left_foot.y

        # Normalize direction vector
        dist = math.sqrt(dx*dx + dy*dy)
        if dist > 0:
            dx /= dist
            dy /= dist

        # Calculate new position with step length and width considerations
        new_x = self.left_foot.x + dx * self.step_length
        new_y = self.left_foot.y + dy * self.step_length - self.step_width/2  # Offset to right
        new_theta = target_theta  # Match target orientation

        return FootPose(new_x, new_y, new_theta, 'right')

    def generate_alternative_step(self, original_step, support_foot):
        """
        Generate alternative footstep if original is invalid
        """
        if support_foot == 'left':
            # Generate right foot alternative
            dx = original_step.x - self.left_foot.x
            dy = original_step.y - self.left_foot.y
            dist = math.sqrt(dx*dx + dy*dy)

            if dist > self.constraints.max_distance:
                # Scale down to maximum allowed distance
                scale = self.constraints.max_distance / dist
                new_x = self.left_foot.x + dx * scale
                new_y = self.left_foot.y + dy * scale - self.step_width/2
            else:
                new_x = original_step.x
                new_y = original_step.y - 0.05  # Small adjustment

            return FootPose(new_x, new_y, original_step.theta, 'right')
        else:
            # Generate left foot alternative
            dx = original_step.x - self.right_foot.x
            dy = original_step.y - self.right_foot.y
            dist = math.sqrt(dx*dx + dy*dy)

            if dist > self.constraints.max_distance:
                # Scale down to maximum allowed distance
                scale = self.constraints.max_distance / dist
                new_x = self.right_foot.x + dx * scale
                new_y = self.right_foot.y + dy * scale + self.step_width/2
            else:
                new_x = original_step.x
                new_y = original_step.y + 0.05  # Small adjustment

            return FootPose(new_x, new_y, original_step.theta, 'left')

    def validate_step(self, step):
        """
        Validate if step meets all constraints
        """
        # Check if step is within distance limits
        if step.step_type == 'left':
            support_pos = [self.right_foot.x, self.right_foot.y]
        else:
            support_pos = [self.left_foot.x, self.left_foot.y]

        step_pos = [step.x, step.y]
        distance = math.sqrt(sum((a-b)**2 for a, b in zip(step_pos, support_pos)))

        if distance < self.constraints.min_distance or distance > self.constraints.max_distance:
            return False

        # Check rotation constraint
        if step.step_type == 'left':
            rotation_diff = abs(step.theta - self.left_foot.theta)
        else:
            rotation_diff = abs(step.theta - self.right_foot.theta)

        if rotation_diff > self.constraints.max_rotation:
            return False

        # Check balance constraint (simplified)
        if not self.check_balance_constraint(step):
            return False

        return True

    def check_balance_constraint(self, step):
        """
        Check if step maintains balance
        """
        # Calculate support polygon vertices
        if step.step_type == 'left':
            # New left foot, right foot is support
            support_polygon = [
                [self.right_foot.x, self.right_foot.y],
                [step.x, step.y]
            ]
        else:
            # New right foot, left foot is support
            support_polygon = [
                [self.left_foot.x, self.left_foot.y],
                [step.x, step.y]
            ]

        # Calculate approximate COM projection
        # This is a simplified balance check
        com_x = (support_polygon[0][0] + support_polygon[1][0]) / 2.0
        com_y = (support_polygon[0][1] + support_polygon[1][1]) / 2.0

        # Check if COM is within balance margin of support polygon
        # For a line segment (biped stance), check distance to line
        dist_to_line = self.distance_to_line_segment(
            com_x, com_y, support_polygon[0], support_polygon[1])

        return dist_to_line <= self.constraints.balance_margin

    def distance_to_line_segment(self, px, py, line_start, line_end):
        """
        Calculate distance from point to line segment
        """
        x1, y1 = line_start
        x2, y2 = line_end

        # Vector from line_start to line_end
        dx = x2 - x1
        dy = y2 - y1

        # Length squared of the line segment
        length_sq = dx*dx + dy*dy

        if length_sq == 0:
            # Line segment is actually a point
            return math.sqrt((px - x1)**2 + (py - y1)**2)

        # Parameter of closest point on the line
        t = max(0, min(1, ((px - x1) * dx + (py - y1) * dy) / length_sq))

        # Coordinates of closest point
        closest_x = x1 + t * dx
        closest_y = y1 + t * dy

        # Distance to closest point
        return math.sqrt((px - closest_x)**2 + (py - closest_y)**2)

    def optimize_step_sequence(self, steps):
        """
        Optimize step sequence for smoother motion
        """
        # This would implement optimization algorithms to smooth the step sequence
        # For now, return steps as-is
        return steps

    def visualize_plan(self, steps):
        """
        Visualize the planned footstep sequence
        """
        fig, ax = plt.subplots(figsize=(10, 6))

        # Plot footsteps
        left_xs, left_ys = [], []
        right_xs, right_ys = [], []

        for step in steps:
            if step.step_type == 'left':
                left_xs.append(step.x)
                left_ys.append(step.y)
            else:
                right_xs.append(step.x)
                right_ys.append(step.y)

        ax.scatter(left_xs, left_ys, c='red', label='Left Foot', s=100, alpha=0.7)
        ax.scatter(right_xs, right_ys, c='blue', label='Right Foot', s=100, alpha=0.7)

        # Draw step sequence
        all_steps = [(step.x, step.y, step.step_type) for step in steps]
        for i in range(len(all_steps)-1):
            x1, y1, _ = all_steps[i]
            x2, y2, _ = all_steps[i+1]
            ax.plot([x1, x2], [y1, y2], 'k--', alpha=0.3)

        ax.set_xlabel('X (m)')
        ax.set_ylabel('Y (m)')
        ax.set_title('Footstep Plan Visualization')
        ax.legend()
        ax.grid(True, alpha=0.3)
        ax.axis('equal')

        plt.tight_layout()
        plt.show()

def example_footstep_planning():
    """
    Example of using the footstep planner
    """
    planner = FootstepPlanner(step_length=0.3, step_width=0.2, com_height=0.85)

    # Plan steps to a target position
    target_x, target_y = 2.0, 1.0
    steps = planner.plan_steps_to_target(target_x, target_y, num_steps=10)

    print(f"Planned {len(steps)} footsteps to reach ({target_x}, {target_y})")

    for i, step in enumerate(steps):
        print(f"Step {i+1}: {step.step_type} foot at ({step.x:.2f}, {step.y:.2f}), θ={step.theta:.2f}")

    # Visualize the plan (optional)
    # planner.visualize_plan(steps)

if __name__ == "__main__":
    example_footstep_planning()
```

## Integration with Navigation Stack and Path Planning

Integrating humanoid-specific navigation with the broader ROS 2 navigation stack:

```python
import rclpy
from rclpy.node import Node
from nav2_msgs.action import ComputePathToPose
from nav2_msgs.srv import GetCostmap
from geometry_msgs.msg import PoseStamped, Point
from nav_msgs.msg import Path
from visualization_msgs.msg import MarkerArray
from std_msgs.msg import Float32
import numpy as np
from scipy.spatial.distance import cdist
from tf2_ros import TransformListener, Buffer
import threading

class HumanoidPathIntegrator(Node):
    """
    Integrator for humanoid-specific path planning with Nav2
    """
    def __init__(self):
        super().__init__('humanoid_path_integrator')

        # TF2 for coordinate transforms
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

        # Action and service clients
        self.path_client = ActionClient(self, ComputePathToPose, 'compute_path_to_pose')
        self.costmap_client = self.create_client(GetCostmap, 'costmap/get_costmap')

        # Publishers
        self.optimized_path_pub = self.create_publisher(Path, '/humanoid/optimized_path', 10)
        self.balance_risk_pub = self.create_publisher(Float32, '/humanoid/balance_risk', 10)
        self.stability_markers_pub = self.create_publisher(MarkerArray, '/humanoid/stability_markers', 10)

        # Parameters for humanoid-specific path optimization
        self.com_height = 0.85  # Center of mass height
        self.step_length = 0.3
        self.step_width = 0.2
        self.balance_margin = 0.15
        self.max_slope = 0.3  # Maximum traversable slope (rise/run)

        # Threading for path optimization
        self.path_optimization_lock = threading.Lock()

        self.get_logger().info('Humanoid Path Integrator initialized')

    def request_path(self, start_pose, goal_pose):
        """
        Request path from Nav2 and optimize for humanoid constraints
        """
        if not self.path_client.wait_for_server(timeout_sec=1.0):
            self.get_logger().error('ComputePathToPose action server not available')
            return None

        # Create path request
        goal_msg = ComputePathToPose.Goal()
        goal_msg.start = start_pose
        goal_msg.goal = goal_pose
        goal_msg.planner_id = ''  # Use default planner

        # Send request
        future = self.path_client.send_goal_async(goal_msg)
        future.add_done_callback(lambda f: self.path_response_callback(f, start_pose, goal_pose))

        return future

    def path_response_callback(self, future, start_pose, goal_pose):
        """
        Handle path response and optimize for humanoid
        """
        goal_handle = future.result()
        if not goal_handle.accepted:
            self.get_logger().info('Path planning request rejected')
            return

        result_future = goal_handle.get_result_async()
        result_future.add_done_callback(
            lambda f: self.process_path_result(f, start_pose, goal_pose))

    def process_path_result(self, future, start_pose, goal_pose):
        """
        Process the computed path and optimize for humanoid
        """
        result = future.result().result

        if result is None or result.path.poses is None:
            self.get_logger().error('Invalid path result')
            return

        # Optimize path for humanoid constraints
        with self.path_optimization_lock:
            optimized_path = self.optimize_path_for_humanoid(result.path)

        # Publish optimized path
        self.optimized_path_pub.publish(optimized_path)

        # Evaluate balance risk along path
        balance_risk = self.evaluate_balance_risk(optimized_path)
        risk_msg = Float32()
        risk_msg.data = balance_risk
        self.balance_risk_pub.publish(risk_msg)

        # Publish stability markers
        self.publish_stability_markers(optimized_path)

    def optimize_path_for_humanoid(self, original_path):
        """
        Optimize path for humanoid-specific constraints
        """
        if not original_path.poses:
            return original_path

        optimized_poses = []

        # Process each pose in the path
        for i, pose in enumerate(original_path.poses):
            # Check for terrain constraints (slope, obstacles, etc.)
            if self.is_pose_traversable(pose):
                # Adjust pose for humanoid kinematics if needed
                adjusted_pose = self.adjust_pose_for_humanoid(pose)

                # Add to optimized path
                optimized_poses.append(adjusted_pose)

        # Create new path message
        optimized_path = Path()
        optimized_path.header = original_path.header
        optimized_path.poses = optimized_poses

        # Smooth the path for humanoid gait
        smoothed_path = self.smooth_path_for_gait(optimized_path)

        return smoothed_path

    def is_pose_traversable(self, pose):
        """
        Check if a pose is traversable by humanoid robot
        """
        # Check for excessive slope
        slope_ok = self.check_terrain_slope(pose)

        # Check for obstacles in humanoid-sized footprint
        obstacle_free = self.check_footprint_clearance(pose)

        # Check for step height constraints
        step_height_ok = self.check_step_height(pose)

        return slope_ok and obstacle_free and step_height_ok

    def check_terrain_slope(self, pose):
        """
        Check if terrain slope is within humanoid limits
        """
        # In a real implementation, this would query elevation data
        # For simulation, assume flat terrain is OK
        return True

    def check_footprint_clearance(self, pose):
        """
        Check if humanoid-sized footprint is clear of obstacles
        """
        # In a real implementation, this would check costmap with humanoid footprint
        # For simulation, assume clearance is OK
        return True

    def check_step_height(self, pose):
        """
        Check if step height is within humanoid limits
        """
        # In a real implementation, this would check elevation differences
        # For simulation, assume step height is OK
        return True

    def adjust_pose_for_humanoid(self, pose):
        """
        Adjust pose for humanoid kinematic constraints
        """
        # In a real implementation, this might adjust orientation
        # for better balance or step placement
        adjusted_pose = PoseStamped()
        adjusted_pose.header = pose.header
        adjusted_pose.pose = pose.pose

        # Ensure orientation is appropriate for humanoid locomotion
        # (e.g., facing direction of movement)
        self.ensure_appropriate_orientation(adjusted_pose)

        return adjusted_pose

    def ensure_appropriate_orientation(self, pose):
        """
        Ensure pose orientation is appropriate for humanoid movement
        """
        # In a real implementation, this would adjust orientation
        # to face along the path direction
        pass

    def smooth_path_for_gait(self, path):
        """
        Smooth path for natural humanoid gait
        """
        if len(path.poses) < 3:
            return path

        # Convert poses to numpy arrays for processing
        points = np.array([[p.pose.position.x, p.pose.position.y]
                          for p in path.poses])

        # Apply smoothing algorithm (e.g., cubic spline)
        smoothed_points = self.apply_path_smoothing(points)

        # Create new path with smoothed points
        smoothed_path = Path()
        smoothed_path.header = path.header

        for i, point in enumerate(smoothed_points):
            pose_stamped = PoseStamped()
            pose_stamped.header = path.header
            pose_stamped.pose.position.x = point[0]
            pose_stamped.pose.position.y = point[1]
            pose_stamped.pose.position.z = path.poses[i].pose.position.z

            # Interpolate orientation
            if i > 0:
                dx = point[0] - smoothed_points[i-1][0]
                dy = point[1] - smoothed_points[i-1][1]
                yaw = math.atan2(dy, dx)

                q = R.from_euler('z', yaw).as_quat()
                pose_stamped.pose.orientation.x = q[0]
                pose_stamped.pose.orientation.y = q[1]
                pose_stamped.pose.orientation.z = q[2]
                pose_stamped.pose.orientation.w = q[3]
            else:
                # Keep original orientation for first point
                pose_stamped.pose.orientation = path.poses[0].pose.orientation

            smoothed_path.poses.append(pose_stamped)

        return smoothed_path

    def apply_path_smoothing(self, points):
        """
        Apply path smoothing algorithm
        """
        if len(points) < 3:
            return points

        # Simple Gaussian smoothing
        smoothed = np.zeros_like(points)
        window_size = 3
        half_window = window_size // 2

        for i in range(len(points)):
            start_idx = max(0, i - half_window)
            end_idx = min(len(points), i + half_window + 1)

            smoothed[i] = np.mean(points[start_idx:end_idx], axis=0)

        return smoothed

    def evaluate_balance_risk(self, path):
        """
        Evaluate balance risk along the path
        """
        if len(path.poses) < 2:
            return 0.0

        total_risk = 0.0
        num_segments = len(path.poses) - 1

        for i in range(num_segments):
            pose1 = path.poses[i]
            pose2 = path.poses[i+1]

            # Calculate direction vector
            dx = pose2.pose.position.x - pose1.pose.position.x
            dy = pose2.pose.position.y - pose1.pose.position.y
            distance = math.sqrt(dx*dx + dy*dy)

            if distance > 0:
                # Normalize direction
                dx /= distance
                dy /= distance

                # Calculate risk based on path curvature and step length
                risk = self.calculate_segment_risk(pose1, pose2, dx, dy)
                total_risk += risk

        avg_risk = total_risk / num_segments if num_segments > 0 else 0.0
        return min(avg_risk, 1.0)  # Clamp to [0, 1]

    def calculate_segment_risk(self, pose1, pose2, dx, dy):
        """
        Calculate risk for a path segment
        """
        # Factors affecting balance risk:
        # - Sharp turns
        # - Steep slopes
        # - Rough terrain
        # - Obstacle proximity

        risk = 0.0

        # Check for sharp turns (high curvature)
        if len(self.previous_orientations) >= 2:
            prev_yaw = self.orientation_to_yaw(self.previous_orientations[-2])
            curr_yaw = self.orientation_to_yaw(pose1.pose.orientation)
            next_yaw = self.orientation_to_yaw(pose2.pose.orientation)

            # Calculate change in heading
            turn1 = abs(self.normalize_angle(curr_yaw - prev_yaw))
            turn2 = abs(self.normalize_angle(next_yaw - curr_yaw))

            avg_turn = (turn1 + turn2) / 2.0
            risk += min(avg_turn / (math.pi/4), 1.0) * 0.3  # Max 30% risk from turns

        # Add other risk factors as needed

        return risk

    def orientation_to_yaw(self, orientation):
        """
        Convert quaternion orientation to yaw angle
        """
        siny_cosp = 2 * (orientation.w * orientation.z + orientation.x * orientation.y)
        cosy_cosp = 1 - 2 * (orientation.y * orientation.y + orientation.z * orientation.z)
        return math.atan2(siny_cosp, cosy_cosp)

    def normalize_angle(self, angle):
        """
        Normalize angle to [-π, π] range
        """
        while angle > math.pi:
            angle -= 2 * math.pi
        while angle < -math.pi:
            angle += 2 * math.pi
        return angle

    def publish_stability_markers(self, path):
        """
        Publish stability markers for path visualization
        """
        marker_array = MarkerArray()

        for i, pose in enumerate(path.poses):
            # Create stability indicator marker
            marker = Marker()
            marker.header.frame_id = path.header.frame_id
            marker.header.stamp = self.get_clock().now().to_msg()
            marker.ns = 'stability'
            marker.id = i
            marker.type = Marker.SPHERE
            marker.action = Marker.ADD

            marker.pose = pose.pose
            marker.scale.x = 0.1
            marker.scale.y = 0.1
            marker.scale.z = 0.1

            # Color based on stability (green=stable, red=unstable)
            stability_score = self.assess_local_stability(pose)
            marker.color.r = 1.0 - stability_score  # Red decreases with stability
            marker.color.g = stability_score      # Green increases with stability
            marker.color.b = 0.0
            marker.color.a = 0.7

            marker_array.markers.append(marker)

        self.stability_markers_pub.publish(marker_array)

    def assess_local_stability(self, pose):
        """
        Assess local stability at a given pose
        """
        # In a real implementation, this would check local terrain properties
        # For simulation, return a dummy stability score
        return 0.8  # Assume 80% stability

def main(args=None):
    rclpy.init(args=args)
    integrator = HumanoidPathIntegrator()

    try:
        rclpy.spin(integrator)
    except KeyboardInterrupt:
        integrator.get_logger().info('Humanoid Path Integrator shutting down...')
    finally:
        integrator.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Summary

This lesson provided a comprehensive overview of Nav2 integration for humanoid robots, including global path planning with humanoid-specific constraints and specialized footstep planning algorithms. We explored advanced Nav2 configuration for bipedal locomotion, implemented sophisticated footstep planning with balance considerations, and detailed integration with the broader navigation stack. The approach considers humanoid kinematic constraints, balance requirements, and gait-specific path optimization to enable safe and stable navigation for bipedal robots.

## References

1. Navigation Working Group. (2021). Nav2: The Navigation Stack for Autonomy. *ROS Wiki*. https://navigation.ros.org/
2. Kuffner, J., & LaValle, S. M. (2000). RRT-connect: An efficient approach to single-query path planning. *Proceedings of the IEEE International Conference on Robotics and Automation*, 995-1001. https://doi.org/10.1109/ROBOT.2000.844730
3. Wieber, P. B. (2006). Pattern generators with sensory feedback for the control of quadruped and biped walking. *Proceedings of the 2006 IEEE International Conference on Robotics and Automation*, 2573-2578. https://doi.org/10.1109/ROBOT.2006.1642025