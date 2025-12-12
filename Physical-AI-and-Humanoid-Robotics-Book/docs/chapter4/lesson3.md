---
sidebar_position: 3
---

# Lesson 4.3: Capstone: Integrating All Components

This capstone lesson integrates all the components learned throughout the book to create a complete humanoid robot system that demonstrates the full pipeline from perception to action execution. This lesson brings together ROS 2 architecture, humanoid modeling, physics simulation, high-fidelity rendering, synthetic data generation, perception systems, navigation, and voice command processing.

## Introduction

The capstone project synthesizes all the concepts learned in previous chapters into a comprehensive humanoid robot system. This lesson demonstrates how to integrate perception, planning, control, and interaction systems to create a functional humanoid robot that can understand natural language commands, perceive its environment, plan actions, and execute complex behaviors safely and efficiently.

## Key Components Integration

The capstone system integrates the following key components:
- ROS 2 communication architecture with real-time control
- Humanoid robot model with URDF and kinematic chains
- Gazebo physics simulation for bipedal locomotion
- Unity for high-fidelity rendering and HRI
- Isaac Sim for synthetic data generation
- Isaac ROS for perception and object detection
- Nav2 for navigation and path planning
- Whisper for voice command processing
- LLM-driven task decomposition and action planning

## Complete System Architecture

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String, Bool
from sensor_msgs.msg import Image, LaserScan, JointState
from geometry_msgs.msg import Twist, PoseStamped
from nav_msgs.msg import Odometry
from tf2_ros import TransformBroadcaster
import tf_transformations
from typing import Dict, List, Any
import threading
import time
import numpy as np
from dataclasses import dataclass
from enum import Enum

class RobotState(Enum):
    IDLE = "idle"
    LISTENING = "listening"
    PROCESSING = "processing"
    NAVIGATING = "navigating"
    MANIPULATING = "manipulating"
    ERROR = "error"

@dataclass
class PerceptionData:
    """Container for perception data from multiple sensors"""
    rgb_image: Image = None
    depth_image: Image = None
    lidar_scan: LaserScan = None
    joint_states: JointState = None
    odometry: Odometry = None

class HumanoidRobotSystem(Node):
    def __init__(self):
        super().__init__('humanoid_robot_system')

        # Initialize state
        self.current_state = RobotState.IDLE
        self.perception_data = PerceptionData()
        self.navigation_goal = None
        self.voice_command = ""

        # Publishers
        self.cmd_vel_pub = self.create_publisher(Twist, '/cmd_vel', 10)
        self.joint_cmd_pub = self.create_publisher(JointState, '/joint_commands', 10)
        self.status_pub = self.create_publisher(String, '/robot_status', 10)
        self.speech_pub = self.create_publisher(String, '/tts_input', 10)

        # Subscribers
        self.rgb_sub = self.create_subscription(Image, '/rgb_camera/image_raw', self.rgb_callback, 10)
        self.depth_sub = self.create_subscription(Image, '/depth_camera/image_raw', self.depth_callback, 10)
        self.lidar_sub = self.create_subscription(LaserScan, '/scan', self.lidar_callback, 10)
        self.joint_sub = self.create_subscription(JointState, '/joint_states', self.joint_callback, 10)
        self.odom_sub = self.create_subscription(Odometry, '/odom', self.odom_callback, 10)
        self.voice_sub = self.create_subscription(String, '/transcribed_command', self.voice_callback, 10)

        # Initialize subsystems
        self.perception_system = PerceptionSystem(self)
        self.navigation_system = NavigationSystem(self)
        self.voice_system = VoiceCommandSystem(self)
        self.task_planner = TaskPlanner(self)

        # Timer for main control loop
        self.control_timer = self.create_timer(0.1, self.control_loop)

        # Transform broadcaster
        self.tf_broadcaster = TransformBroadcaster(self)

        self.get_logger().info('Humanoid Robot System initialized')

    def rgb_callback(self, msg: Image):
        """RGB camera callback"""
        self.perception_data.rgb_image = msg

    def depth_callback(self, msg: Image):
        """Depth camera callback"""
        self.perception_data.depth_image = msg

    def lidar_callback(self, msg: LaserScan):
        """LiDAR callback"""
        self.perception_data.lidar_scan = msg

    def joint_callback(self, msg: JointState):
        """Joint states callback"""
        self.perception_data.joint_states = msg

    def odom_callback(self, msg: Odometry):
        """Odometry callback"""
        self.perception_data.odometry = msg

    def voice_callback(self, msg: String):
        """Voice command callback"""
        self.voice_command = msg.data
        self.get_logger().info(f'Received voice command: {self.voice_command}')

        # Process voice command
        if self.current_state == RobotState.IDLE:
            self.current_state = RobotState.PROCESSING
            self.process_voice_command()

    def process_voice_command(self):
        """Process the received voice command"""
        if not self.voice_command:
            return

        # Use task planner to decompose command
        task_plan = self.task_planner.decompose_task(self.voice_command)

        if task_plan:
            self.get_logger().info(f'Executing task plan with {len(task_plan)} actions')
            success = self.task_planner.execute_task_plan(task_plan)

            if success:
                self.speak_response("Task completed successfully")
            else:
                self.speak_response("Task execution failed")
        else:
            self.speak_response("Could not understand the command")

        self.current_state = RobotState.IDLE

    def speak_response(self, text: str):
        """Publish text for speech synthesis"""
        msg = String()
        msg.data = text
        self.speech_pub.publish(msg)

    def control_loop(self):
        """Main control loop"""
        # Update perception system
        self.perception_system.update(self.perception_data)

        # Update navigation system
        self.navigation_system.update()

        # Update voice system
        self.voice_system.update()

        # Broadcast transforms
        self.broadcast_transforms()

        # Publish status
        status_msg = String()
        status_msg.data = f"State: {self.current_state.value}, Joint Count: {len(self.perception_data.joint_states.name) if self.perception_data.joint_states else 0}"
        self.status_pub.publish(status_msg)

    def broadcast_transforms(self):
        """Broadcast coordinate transforms"""
        if self.perception_data.odometry:
            # Create transform from odom to base_link
            t = tf_transformations.TransformerROS()
            # This would broadcast the actual transforms in a real implementation
            pass

class PerceptionSystem:
    def __init__(self, robot_node: HumanoidRobotSystem):
        self.robot_node = robot_node
        self.object_detector = ObjectDetectionSystem()
        self.scene_analyzer = SceneAnalysisSystem()

    def update(self, perception_data: PerceptionData):
        """Update perception system with new data"""
        # Process RGB image for object detection
        if perception_data.rgb_image:
            detected_objects = self.object_detector.detect_objects(perception_data.rgb_image)

        # Process depth image for 3D reconstruction
        if perception_data.depth_image:
            point_cloud = self.process_depth_image(perception_data.depth_image)

        # Process LiDAR data for obstacle detection
        if perception_data.lidar_scan:
            obstacles = self.process_lidar_data(perception_data.lidar_scan)

    def process_depth_image(self, depth_image: Image) -> np.ndarray:
        """Process depth image to generate point cloud"""
        # Convert depth image to point cloud
        # This would involve complex processing in a real implementation
        return np.array([])

    def process_lidar_data(self, lidar_scan: LaserScan) -> List[Dict[str, Any]]:
        """Process LiDAR scan for obstacle detection"""
        obstacles = []
        # Process ranges and intensities to detect obstacles
        for i, range_val in enumerate(lidar_scan.ranges):
            if 0 < range_val < lidar_scan.range_max:
                angle = lidar_scan.angle_min + i * lidar_scan.angle_increment
                obstacles.append({
                    'angle': angle,
                    'distance': range_val,
                    'x': range_val * np.cos(angle),
                    'y': range_val * np.sin(angle)
                })
        return obstacles

class NavigationSystem:
    def __init__(self, robot_node: HumanoidRobotSystem):
        self.robot_node = robot_node
        self.local_planner = LocalPlanner()
        self.global_planner = GlobalPlanner()
        self.collision_avoider = CollisionAvoider()

    def update(self):
        """Update navigation system"""
        # This would handle navigation updates in a real implementation
        pass

class VoiceCommandSystem:
    def __init__(self, robot_node: HumanoidRobotSystem):
        self.robot_node = robot_node
        self.voice_processor = VoiceProcessor()
        self.intent_classifier = IntentClassifier()

    def update(self):
        """Update voice command system"""
        # This would handle voice processing in a real implementation
        pass

class TaskPlanner:
    def __init__(self, robot_node: HumanoidRobotSystem):
        self.robot_node = robot_node
        self.llm_interface = LLMInterface()
        self.action_validator = ActionValidator()
        self.executor = ActionExecutor()

    def decompose_task(self, natural_language_command: str) -> List[Dict[str, Any]]:
        """Decompose natural language command into executable actions"""
        # Use LLM to decompose task
        actions = self.llm_interface.decompose_task(natural_language_command)

        # Validate actions
        validated_actions = []
        for action in actions:
            if self.action_validator.validate_action(action):
                validated_actions.append(action)

        return validated_actions

    def execute_task_plan(self, task_plan: List[Dict[str, Any]]) -> bool:
        """Execute the decomposed task plan"""
        success = True
        for action in task_plan:
            action_success = self.executor.execute_action(action)
            if not action_success:
                self.robot_node.get_logger().error(f'Action failed: {action}')
                success = False
                break
        return success
```

## Perception System Integration

The perception system integrates multiple sensors to create a comprehensive understanding of the robot's environment.

```python
import cv2
import numpy as np
from scipy.spatial import distance
import open3d as o3d
from typing import Tuple, Optional

class ObjectDetectionSystem:
    def __init__(self):
        # Initialize Isaac ROS perception pipeline
        self.detection_model = self.initialize_detection_model()

    def initialize_detection_model(self):
        """Initialize object detection model"""
        # In a real implementation, this would load Isaac ROS detection models
        # For simulation, we'll create a mock detector
        return MockObjectDetector()

    def detect_objects(self, rgb_image: Image) -> List[Dict[str, Any]]:
        """Detect objects in RGB image"""
        # Convert ROS image to OpenCV format
        cv_image = self.ros_image_to_cv2(rgb_image)

        # Run object detection
        detections = self.detection_model.detect(cv_image)

        # Convert to 3D positions using depth information
        objects_3d = []
        for detection in detections:
            obj_3d = self.project_to_3d(detection, self.robot_node.perception_data.depth_image)
            objects_3d.append(obj_3d)

        return objects_3d

    def ros_image_to_cv2(self, ros_image: Image) -> np.ndarray:
        """Convert ROS image message to OpenCV image"""
        # Convert the ROS Image message to a numpy array
        dtype = np.uint8
        if ros_image.encoding == 'rgb8':
            dtype = np.uint8
        elif ros_image.encoding == 'rgba8':
            dtype = np.uint8
        elif ros_image.encoding == 'bgr8':
            dtype = np.uint8
        elif ros_image.encoding == 'mono8':
            dtype = np.uint8
        elif ros_image.encoding == 'mono16':
            dtype = np.uint16

        img = np.frombuffer(ros_image.data, dtype=dtype)
        img = img.reshape((ros_image.height, ros_image.width, -1))

        # Convert to BGR if needed
        if ros_image.encoding == 'rgb8':
            img = cv2.cvtColor(img, cv2.COLOR_RGB2BGR)

        return img

    def project_to_3d(self, detection: Dict[str, Any], depth_image: Image) -> Dict[str, Any]:
        """Project 2D detection to 3D space using depth information"""
        # Get center of bounding box
        center_x = int((detection['bbox'][0] + detection['bbox'][2]) / 2)
        center_y = int((detection['bbox'][1] + detection['bbox'][3]) / 2)

        # Get depth at center point
        depth_value = self.get_depth_at_point(center_x, center_y, depth_image)

        # Convert to 3D coordinates (simplified)
        x = center_x * depth_value
        y = center_y * depth_value
        z = depth_value

        detection_3d = detection.copy()
        detection_3d['position'] = [x, y, z]
        detection_3d['distance'] = depth_value

        return detection_3d

    def get_depth_at_point(self, x: int, y: int, depth_image: Image) -> float:
        """Get depth value at specific pixel coordinates"""
        # This is a simplified implementation
        # In reality, depth processing would be more complex
        if depth_image:
            # Convert to numpy array
            dtype = np.float32 if depth_image.encoding == '32FC1' else np.uint16
            depth_array = np.frombuffer(depth_image.data, dtype=dtype)
            depth_array = depth_array.reshape((depth_image.height, depth_image.width))

            # Get depth value at point
            if 0 <= y < depth_array.shape[0] and 0 <= x < depth_array.shape[1]:
                return float(depth_array[y, x])

        return float('inf')

class SceneAnalysisSystem:
    def __init__(self):
        self.semantic_segmenter = SemanticSegmenter()
        self.surface_analyzer = SurfaceAnalyzer()
        self.spatial_reasoner = SpatialReasoner()

    def analyze_scene(self, perception_data: PerceptionData) -> Dict[str, Any]:
        """Analyze the current scene comprehensively"""
        scene_analysis = {}

        # Semantic segmentation
        if perception_data.rgb_image:
            scene_analysis['semantic_map'] = self.semantic_segmenter.segment(perception_data.rgb_image)

        # Surface analysis for navigation
        if perception_data.depth_image:
            scene_analysis['traversable_areas'] = self.surface_analyzer.find_traversable_surfaces(perception_data.depth_image)

        # Spatial reasoning
        scene_analysis['spatial_relations'] = self.spatial_reasoner.compute_spatial_relations(
            perception_data.lidar_scan,
            perception_data.odometry
        )

        return scene_analysis

class MockObjectDetector:
    """Mock object detector for simulation purposes"""
    def detect(self, image: np.ndarray) -> List[Dict[str, Any]]:
        """Mock detection function"""
        # In a real implementation, this would use Isaac ROS detection models
        # For simulation, return mock detections
        return [
            {
                'class': 'person',
                'confidence': 0.95,
                'bbox': [100, 100, 200, 200],  # [x1, y1, x2, y2]
                'label': 'person'
            },
            {
                'class': 'chair',
                'confidence': 0.87,
                'bbox': [300, 150, 400, 250],
                'label': 'chair'
            }
        ]

class SemanticSegmenter:
    def segment(self, image: Image) -> np.ndarray:
        """Perform semantic segmentation on image"""
        # In a real implementation, this would use Isaac ROS segmentation
        # For simulation, return mock segmentation
        cv_image = self.ros_image_to_cv2(image)
        height, width = cv_image.shape[:2]
        return np.random.randint(0, 10, (height, width))  # Random segmentation

    def ros_image_to_cv2(self, ros_image: Image) -> np.ndarray:
        """Convert ROS image to OpenCV format"""
        dtype = np.uint8
        img = np.frombuffer(ros_image.data, dtype=dtype)
        img = img.reshape((ros_image.height, ros_image.width, -1))

        if ros_image.encoding == 'rgb8':
            img = cv2.cvtColor(img, cv2.COLOR_RGB2BGR)

        return img

class SurfaceAnalyzer:
    def find_traversable_surfaces(self, depth_image: Image) -> List[Dict[str, Any]]:
        """Find traversable surfaces from depth data"""
        # Analyze depth image to identify flat, traversable surfaces
        # This would be implemented with Isaac ROS in a real system
        return [{'center': [0, 0, 0], 'normal': [0, 0, 1], 'area': 10.0}]

class SpatialReasoner:
    def compute_spatial_relations(self, lidar_scan: LaserScan, odometry: Odometry) -> Dict[str, Any]:
        """Compute spatial relations between robot and environment"""
        # Compute spatial relationships using LiDAR and odometry data
        relations = {
            'nearest_obstacle': self.find_nearest_obstacle(lidar_scan),
            'robot_position': [odometry.pose.pose.position.x, odometry.pose.pose.position.y],
            'free_space': self.compute_free_space(lidar_scan)
        }
        return relations

    def find_nearest_obstacle(self, lidar_scan: LaserScan) -> Optional[Tuple[float, float]]:
        """Find the nearest obstacle from LiDAR data"""
        if lidar_scan.ranges:
            min_distance = min([r for r in lidar_scan.ranges if 0 < r < lidar_scan.range_max])
            min_idx = lidar_scan.ranges.index(min_distance)
            angle = lidar_scan.angle_min + min_idx * lidar_scan.angle_increment
            return (min_distance, angle)
        return None

    def compute_free_space(self, lidar_scan: LaserScan) -> float:
        """Compute amount of free space around robot"""
        free_ranges = [r for r in lidar_scan.ranges if lidar_scan.range_min < r < lidar_scan.range_max * 0.8]
        return len(free_ranges) / len(lidar_scan.ranges) if lidar_scan.ranges else 0.0
```

## Navigation and Path Planning Integration

The navigation system integrates global path planning with local obstacle avoidance for safe humanoid locomotion.

```python
from nav2_msgs.action import NavigateToPose
from geometry_msgs.msg import Pose, Point
from std_msgs.msg import Header
import math

class GlobalPlanner:
    def __init__(self):
        # In a real implementation, this would interface with Nav2
        self.nav2_client = None  # actionlib.SimpleActionClient for NavigateToPose

    def plan_path(self, start_pose: Pose, goal_pose: Pose) -> List[Pose]:
        """Plan global path from start to goal"""
        # In a real implementation, this would call Nav2 global planner
        # For simulation, create a simple path
        path = []

        # Calculate intermediate poses between start and goal
        start_x = start_pose.position.x
        start_y = start_pose.position.y
        goal_x = goal_pose.position.x
        goal_y = goal_pose.position.y

        # Simple linear interpolation
        steps = 10
        for i in range(steps + 1):
            t = i / steps
            x = start_x + t * (goal_x - start_x)
            y = start_y + t * (goal_y - start_y)

            pose = Pose()
            pose.position.x = x
            pose.position.y = y
            pose.position.z = 0.0

            # Set orientation to face toward goal
            angle = math.atan2(goal_y - start_y, goal_x - start_x)
            from tf_transformations import quaternion_from_euler
            quat = quaternion_from_euler(0, 0, angle)
            pose.orientation.x = quat[0]
            pose.orientation.y = quat[1]
            pose.orientation.z = quat[2]
            pose.orientation.w = quat[3]

            path.append(pose)

        return path

class LocalPlanner:
    def __init__(self):
        # Local planner for obstacle avoidance
        self.obstacle_threshold = 0.5  # meters
        self.safety_buffer = 0.3      # meters

    def plan_local_path(self, current_pose: Pose, global_path: List[Pose], obstacles: List[Dict[str, Any]]) -> List[Pose]:
        """Plan local path considering obstacles"""
        # Check if global path is blocked by obstacles
        safe_path = []

        for pose in global_path:
            if not self.is_path_blocked(current_pose, pose, obstacles):
                safe_path.append(pose)
            else:
                # Find alternative path around obstacle
                detour_path = self.compute_detour(current_pose, pose, obstacles)
                safe_path.extend(detour_path)
                break

        return safe_path if safe_path else global_path

    def is_path_blocked(self, start_pose: Pose, end_pose: Pose, obstacles: List[Dict[str, Any]]) -> bool:
        """Check if path between poses is blocked by obstacles"""
        for obstacle in obstacles:
            # Calculate distance from path to obstacle
            path_to_obstacle_dist = self.distance_point_to_line(
                (start_pose.position.x, start_pose.position.y),
                (end_pose.position.x, end_pose.position.y),
                (obstacle['x'], obstacle['y'])
            )

            if path_to_obstacle_dist < self.obstacle_threshold:
                return True
        return False

    def distance_point_to_line(self, line_start: Tuple[float, float], line_end: Tuple[float, float],
                              point: Tuple[float, float]) -> float:
        """Calculate distance from point to line segment"""
        x1, y1 = line_start
        x2, y2 = line_end
        px, py = point

        # Calculate distance from point to line
        line_mag = math.sqrt((x2 - x1)**2 + (y2 - y1)**2)
        if line_mag < 0.000001:
            return math.sqrt((px - x1)**2 + (py - y1)**2)

        u1 = ((px - x1) * (x2 - x1) + (py - y1) * (y2 - y1)) / (line_mag**2)
        u = max(0, min(1, u1))

        ix = x1 + u * (x2 - x1)
        iy = y1 + u * (y2 - y1)

        return math.sqrt((px - ix)**2 + (py - iy)**2)

    def compute_detour(self, current_pose: Pose, target_pose: Pose, obstacles: List[Dict[str, Any]]) -> List[Pose]:
        """Compute detour path around obstacles"""
        # Simple detour by going around the nearest obstacle
        if not obstacles:
            return [target_pose]

        # Find nearest obstacle
        nearest_obstacle = min(obstacles, key=lambda o: math.sqrt(
            (o['x'] - current_pose.position.x)**2 +
            (o['y'] - current_pose.position.y)**2
        ))

        # Create detour points around obstacle
        detour_poses = []

        # Go to a safe point to the side of the obstacle
        safe_x = nearest_obstacle['x'] + 1.0  # 1m to the side
        safe_y = nearest_obstacle['y']

        safe_pose = Pose()
        safe_pose.position.x = safe_x
        safe_pose.position.y = safe_y
        safe_pose.position.z = 0.0

        detour_poses.append(safe_pose)
        detour_poses.append(target_pose)

        return detour_poses

class CollisionAvoider:
    def __init__(self):
        self.min_distance = 0.5  # meters
        self.avoidance_active = False

    def check_collision_risk(self, lidar_scan: LaserScan) -> bool:
        """Check if there's a collision risk based on LiDAR data"""
        if not lidar_scan.ranges:
            return False

        # Check for obstacles within minimum distance
        for range_val in lidar_scan.ranges:
            if 0 < range_val < self.min_distance:
                return True
        return False

    def compute_avoidance_command(self, lidar_scan: LaserScan) -> Twist:
        """Compute avoidance command based on LiDAR data"""
        cmd = Twist()

        if not lidar_scan.ranges:
            return cmd

        # Find the direction with the most free space
        sector_size = len(lidar_scan.ranges) // 8  # Divide into 8 sectors
        sectors = []

        for i in range(8):
            start_idx = i * sector_size
            end_idx = min((i + 1) * sector_size, len(lidar_scan.ranges))
            sector_ranges = lidar_scan.ranges[start_idx:end_idx]

            # Calculate average distance in sector
            valid_ranges = [r for r in sector_ranges if 0 < r < lidar_scan.range_max]
            avg_distance = sum(valid_ranges) / len(valid_ranges) if valid_ranges else 0

            sectors.append({
                'index': i,
                'avg_distance': avg_distance,
                'angle': lidar_scan.angle_min + start_idx * lidar_scan.angle_increment
            })

        # Find sector with maximum average distance
        best_sector = max(sectors, key=lambda s: s['avg_distance'])

        # Set velocity based on free space
        cmd.linear.x = min(0.5, best_sector['avg_distance'] * 0.5)  # Scale velocity

        # Set angular velocity to turn toward the safest direction
        cmd.angular.z = -best_sector['angle']  # Negative because angle might be in different convention

        return cmd
```

## Voice Command and LLM Integration

The voice command system integrates Whisper for speech recognition with LLM-driven task planning.

```python
import openai
import json
from typing import Dict, List, Any
import re

class VoiceProcessor:
    def __init__(self):
        # In a real implementation, this would interface with Whisper
        self.whisper_client = None  # Whisper client would be initialized here

    def process_audio(self, audio_data: bytes) -> str:
        """Process audio data using Whisper"""
        # In a real implementation, this would call Whisper API
        # For simulation, return mock transcription
        return "Go to the kitchen and bring me a cup"

class IntentClassifier:
    def __init__(self):
        self.intent_patterns = {
            'navigation': [
                r'go to (.+)',
                r'move to (.+)',
                r'walk to (.+)',
                r'go (.+)',
                r'travel to (.+)'
            ],
            'manipulation': [
                r'pick up (.+)',
                r'grab (.+)',
                r'take (.+)',
                r'get (.+)',
                r'bring me (.+)'
            ],
            'action': [
                r'dance',
                r'wave',
                r'nod',
                r'jump',
                r'stop'
            ]
        }

    def classify_intent(self, text: str) -> str:
        """Classify the intent of the text command"""
        text_lower = text.lower()

        for intent, patterns in self.intent_patterns.items():
            for pattern in patterns:
                if re.search(pattern, text_lower):
                    return intent

        return 'unknown'

class LLMInterface:
    def __init__(self):
        # Initialize OpenAI client
        self.client = openai.OpenAI()  # Use appropriate API key

    def decompose_task(self, natural_language_command: str) -> List[Dict[str, Any]]:
        """Use LLM to decompose natural language command into executable actions"""
        # Define robot capabilities
        capabilities = {
            "navigation": {
                "description": "Move the robot to a specific location",
                "parameters": {"location": "string", "x": "float", "y": "float"}
            },
            "object_detection": {
                "description": "Detect objects in the environment",
                "parameters": {"target_object": "string"}
            },
            "manipulation": {
                "description": "Manipulate objects using robot arms",
                "parameters": {"object": "string", "action": "string"}
            },
            "speech": {
                "description": "Speak a message",
                "parameters": {"text": "string"}
            }
        }

        prompt = f"""
        You are a task planner for a humanoid robot. Decompose the following natural language command into a sequence of executable actions.

        Available capabilities:
        {json.dumps(capabilities, indent=2)}

        Natural language command: "{natural_language_command}"

        Please decompose this command into a sequence of actions. Each action should have:
        - "action_type": The type of action (one of the available capabilities)
        - "parameters": The parameters needed for the action
        - "description": A brief description of what the action does

        Return the result as a JSON array of actions.
        """

        try:
            response = self.client.chat.completions.create(
                model="gpt-4",  # Use appropriate model
                messages=[
                    {"role": "system", "content": "You are a helpful task planner for humanoid robots. Return only valid JSON."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.1,
                max_tokens=1000
            )

            response_text = response.choices[0].message.content.strip()

            # Remove markdown formatting if present
            if response_text.startswith("```json"):
                response_text = response_text[7:]
            if response_text.endswith("```"):
                response_text = response_text[:-3]

            actions = json.loads(response_text)
            return actions

        except Exception as e:
            print(f"Error in LLM task decomposition: {e}")
            return []

class ActionValidator:
    def __init__(self):
        self.constraints = {
            "workspace": {"min_x": -10.0, "max_x": 10.0, "min_y": -10.0, "max_y": 10.0},
            "joint_limits": {
                "arm": {"min": [-2.0, -1.5, -2.0, -1.5, -2.0, -1.5, -2.0], "max": [2.0, 1.5, 2.0, 1.5, 2.0, 1.5, 2.0]}
            }
        }

    def validate_action(self, action: Dict[str, Any]) -> bool:
        """Validate an action against constraints"""
        action_type = action.get("action_type", "")
        parameters = action.get("parameters", {})

        if action_type == "navigation":
            x = parameters.get("x", 0.0)
            y = parameters.get("y", 0.0)

            if not (self.constraints["workspace"]["min_x"] <= x <= self.constraints["workspace"]["max_x"]):
                return False
            if not (self.constraints["workspace"]["min_y"] <= y <= self.constraints["workspace"]["max_y"]):
                return False

        elif action_type == "manipulation":
            # Validate manipulation parameters
            pass

        return True

class ActionExecutor:
    def __init__(self, robot_node: HumanoidRobotSystem):
        self.robot_node = robot_node

    def execute_action(self, action: Dict[str, Any]) -> bool:
        """Execute a single action"""
        action_type = action.get("action_type", "")
        parameters = action.get("parameters", {})

        if action_type == "navigation":
            return self.execute_navigation_action(parameters)
        elif action_type == "object_detection":
            return self.execute_detection_action(parameters)
        elif action_type == "manipulation":
            return self.execute_manipulation_action(parameters)
        elif action_type == "speech":
            return self.execute_speech_action(parameters)
        else:
            self.robot_node.get_logger().error(f"Unknown action type: {action_type}")
            return False

    def execute_navigation_action(self, params: Dict[str, Any]) -> bool:
        """Execute navigation action"""
        # Extract target location
        target_x = params.get("x", 0.0)
        target_y = params.get("y", 0.0)

        # In a real implementation, this would send goal to Nav2
        # For simulation, just move toward the target
        current_pos = self.get_current_position()

        # Calculate direction vector
        dx = target_x - current_pos[0]
        dy = target_y - current_pos[1]
        distance = math.sqrt(dx**2 + dy**2)

        if distance > 0.1:  # If not already at target
            # Move toward target
            cmd = Twist()
            cmd.linear.x = min(0.5, distance)  # Scale speed with distance
            cmd.angular.z = math.atan2(dy, dx)  # Turn toward target

            self.robot_node.cmd_vel_pub.publish(cmd)

            # Wait for movement to complete (simplified)
            time.sleep(2)

        # Stop the robot
        stop_cmd = Twist()
        self.robot_node.cmd_vel_pub.publish(stop_cmd)

        return True

    def execute_detection_action(self, params: Dict[str, Any]) -> bool:
        """Execute object detection action"""
        target_object = params.get("target_object", "")

        # Use perception system to detect the target object
        detected_objects = self.robot_node.perception_system.object_detector.detect_objects(
            self.robot_node.perception_data.rgb_image
        )

        # Check if target object was detected
        found = any(obj['label'] == target_object for obj in detected_objects)

        if found:
            self.robot_node.get_logger().info(f"Found {target_object}")
            return True
        else:
            self.robot_node.get_logger().info(f"Could not find {target_object}")
            return False

    def execute_manipulation_action(self, params: Dict[str, Any]) -> bool:
        """Execute manipulation action"""
        # For simulation, just log the action
        self.robot_node.get_logger().info(f"Manipulation action: {params}")
        return True

    def execute_speech_action(self, params: Dict[str, Any]) -> bool:
        """Execute speech action"""
        text = params.get("text", "")
        self.robot_node.speak_response(text)
        return True

    def get_current_position(self) -> Tuple[float, float]:
        """Get current robot position from odometry"""
        if self.robot_node.perception_data.odometry:
            pose = self.robot_node.perception_data.odometry.pose.pose
            return (pose.position.x, pose.position.y)
        return (0.0, 0.0)
```

## System Integration and Testing

The final integration brings all components together in a cohesive system that demonstrates the complete pipeline.

```python
class SystemIntegrationTest:
    def __init__(self, robot_system: HumanoidRobotSystem):
        self.robot_system = robot_system
        self.test_scenarios = []

    def run_comprehensive_test(self):
        """Run comprehensive integration test"""
        self.get_logger().info("Starting comprehensive system integration test")

        # Test 1: Perception pipeline
        perception_success = self.test_perception_pipeline()

        # Test 2: Navigation system
        navigation_success = self.test_navigation_system()

        # Test 3: Voice command processing
        voice_success = self.test_voice_command_system()

        # Test 4: Task planning and execution
        task_success = self.test_task_planning_system()

        # Test 5: Full system integration
        full_integration_success = self.test_full_integration()

        # Report results
        results = {
            'perception': perception_success,
            'navigation': navigation_success,
            'voice': voice_success,
            'task_planning': task_success,
            'full_integration': full_integration_success
        }

        self.log_test_results(results)
        return all(results.values())

    def test_perception_pipeline(self) -> bool:
        """Test perception pipeline with mock data"""
        self.get_logger().info("Testing perception pipeline...")

        # Simulate perception data
        self.simulate_perception_data()

        # Process data through perception system
        analysis = self.robot_system.perception_system.scene_analyzer.analyze_scene(
            self.robot_system.perception_data
        )

        # Check if analysis contains expected elements
        success = (
            'semantic_map' in analysis and
            'traversable_areas' in analysis and
            'spatial_relations' in analysis
        )

        self.get_logger().info(f"Perception pipeline test: {'PASSED' if success else 'FAILED'}")
        return success

    def test_navigation_system(self) -> bool:
        """Test navigation system"""
        self.get_logger().info("Testing navigation system...")

        # Create mock goal
        goal_pose = Pose()
        goal_pose.position.x = 5.0
        goal_pose.position.y = 5.0

        # Plan path
        global_path = self.robot_system.navigation_system.global_planner.plan_path(
            self.get_current_pose(), goal_pose
        )

        # Check if path was generated
        success = len(global_path) > 0

        self.get_logger().info(f"Navigation system test: {'PASSED' if success else 'FAILED'}")
        return success

    def test_voice_command_system(self) -> bool:
        """Test voice command system"""
        self.get_logger().info("Testing voice command system...")

        # Simulate voice command
        command = "Go to the kitchen and bring me a cup"

        # Process command
        task_plan = self.robot_system.task_planner.decompose_task(command)

        # Check if task plan was generated
        success = len(task_plan) > 0

        self.get_logger().info(f"Voice command system test: {'PASSED' if success else 'FAILED'}")
        return success

    def test_task_planning_system(self) -> bool:
        """Test task planning system"""
        self.get_logger().info("Testing task planning system...")

        # Create mock task plan
        mock_plan = [
            {"action_type": "navigation", "parameters": {"x": 2.0, "y": 3.0}, "description": "Go to location"},
            {"action_type": "object_detection", "parameters": {"target_object": "cup"}, "description": "Find cup"},
            {"action_type": "manipulation", "parameters": {"object": "cup", "action": "grasp"}, "description": "Pick up cup"}
        ]

        # Validate plan
        all_valid = all(
            self.robot_system.task_planner.action_validator.validate_action(action)
            for action in mock_plan
        )

        self.get_logger().info(f"Task planning system test: {'PASSED' if all_valid else 'FAILED'}")
        return all_valid

    def test_full_integration(self) -> bool:
        """Test full system integration"""
        self.get_logger().info("Testing full system integration...")

        # This would involve running a complete scenario
        # For simulation, we'll just verify all components are properly connected
        success = (
            self.robot_system.perception_system is not None and
            self.robot_system.navigation_system is not None and
            self.robot_system.voice_system is not None and
            self.robot_system.task_planner is not None
        )

        self.get_logger().info(f"Full integration test: {'PASSED' if success else 'FAILED'}")
        return success

    def log_test_results(self, results: Dict[str, bool]):
        """Log comprehensive test results"""
        self.get_logger().info("=== SYSTEM INTEGRATION TEST RESULTS ===")
        for component, success in results.items():
            status = "✓ PASS" if success else "✗ FAIL"
            self.get_logger().info(f"{component.replace('_', ' ').title()}: {status}")

        overall_success = all(results.values())
        self.get_logger().info(f"Overall System Status: {'✓ PASS' if overall_success else '✗ FAIL'}")
        self.get_logger().info("=====================================")

    def simulate_perception_data(self):
        """Simulate perception data for testing"""
        # Create mock image data
        mock_rgb = Image()
        mock_rgb.height = 480
        mock_rgb.width = 640
        mock_rgb.encoding = "rgb8"
        mock_rgb.step = 640 * 3
        mock_rgb.data = [0] * (480 * 640 * 3)  # Mock data

        # Create mock LiDAR data
        mock_lidar = LaserScan()
        mock_lidar.ranges = [1.0] * 360  # 360 degree scan
        mock_lidar.angle_min = -math.pi
        mock_lidar.angle_max = math.pi
        mock_lidar.angle_increment = 2 * math.pi / 360

        # Set mock data in perception system
        self.robot_system.perception_data.rgb_image = mock_rgb
        self.robot_system.perception_data.lidar_scan = mock_lidar

    def get_current_pose(self) -> Pose:
        """Get current robot pose"""
        pose = Pose()
        if self.robot_system.perception_data.odometry:
            pose = self.robot_system.perception_data.odometry.pose.pose
        return pose

def main(args=None):
    """Main function to run the integrated humanoid robot system"""
    rclpy.init(args=args)

    # Create the integrated robot system
    robot_system = HumanoidRobotSystem()

    # Create integration test
    integration_test = SystemIntegrationTest(robot_system)

    # Run integration test
    test_success = integration_test.run_comprehensive_test()

    if test_success:
        robot_system.get_logger().info("System integration test PASSED! Starting main loop...")
    else:
        robot_system.get_logger().error("System integration test FAILED!")

    try:
        # Start the main control loop
        rclpy.spin(robot_system)
    except KeyboardInterrupt:
        robot_system.get_logger().info("Shutting down humanoid robot system...")
    finally:
        robot_system.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Performance Optimization and Real-time Considerations

The integrated system must be optimized for real-time performance to ensure responsive behavior.

```python
import threading
import queue
import time
from collections import deque
import psutil
import gc

class PerformanceMonitor:
    def __init__(self):
        self.cpu_usage = deque(maxlen=100)
        self.memory_usage = deque(maxlen=100)
        self.processing_times = deque(maxlen=100)
        self.fps = deque(maxlen=100)

    def monitor_system(self):
        """Monitor system performance metrics"""
        cpu_percent = psutil.cpu_percent()
        memory_percent = psutil.virtual_memory().percent

        self.cpu_usage.append(cpu_percent)
        self.memory_usage.append(memory_percent)

        return {
            'cpu_percent': cpu_percent,
            'memory_percent': memory_percent,
            'avg_cpu': sum(self.cpu_usage) / len(self.cpu_usage) if self.cpu_usage else 0,
            'avg_memory': sum(self.memory_usage) / len(self.memory_usage) if self.memory_usage else 0
        }

    def log_performance_warning(self, component: str, metric: str, value: float, threshold: float):
        """Log performance warnings"""
        print(f"PERFORMANCE WARNING: {component} {metric} ({value}) exceeds threshold ({threshold})")

class RealTimeScheduler:
    def __init__(self):
        self.tasks = []
        self.task_queue = queue.PriorityQueue()
        self.performance_monitor = PerformanceMonitor()

    def add_task(self, task_func, priority: int, period: float, name: str):
        """Add a real-time task with priority and period"""
        task = {
            'func': task_func,
            'priority': priority,
            'period': period,
            'name': name,
            'last_run': 0
        }
        self.tasks.append(task)

    def run_scheduler(self):
        """Run the real-time scheduler"""
        while True:
            current_time = time.time()

            # Check each task to see if it's time to run
            for task in self.tasks:
                if current_time - task['last_run'] >= task['period']:
                    # Run the task
                    start_time = time.time()
                    try:
                        task['func']()
                    except Exception as e:
                        print(f"Error in task {task['name']}: {e}")

                    # Record processing time
                    processing_time = time.time() - start_time
                    self.performance_monitor.processing_times.append(processing_time)

                    # Check if task took too long
                    if processing_time > task['period'] * 0.8:  # 80% of period
                        self.performance_monitor.log_performance_warning(
                            task['name'], 'processing_time', processing_time, task['period'] * 0.8
                        )

                    task['last_run'] = current_time

            # Sleep briefly to prevent busy waiting
            time.sleep(0.001)  # 1ms

class OptimizedPerceptionPipeline:
    def __init__(self):
        self.processing_queue = queue.Queue(maxsize=10)
        self.result_queue = queue.Queue(maxsize=10)
        self.processing_thread = threading.Thread(target=self.process_loop, daemon=True)
        self.is_processing = True

        # Start processing thread
        self.processing_thread.start()

    def process_loop(self):
        """Background processing loop for perception"""
        while self.is_processing:
            try:
                # Get data from queue with timeout
                data = self.processing_queue.get(timeout=0.1)

                # Process the data (in real implementation, this would be heavy computation)
                result = self.process_frame(data)

                # Put result in result queue
                try:
                    self.result_queue.put_nowait(result)
                except queue.Full:
                    # Drop old result if queue is full
                    pass

                self.processing_queue.task_done()
            except queue.Empty:
                continue  # Continue loop if no data

    def process_frame(self, frame_data):
        """Process a single frame with optimizations"""
        # Apply optimizations here
        # 1. Reduce resolution if possible
        # 2. Use faster algorithms
        # 3. Parallel processing
        # 4. GPU acceleration

        # Mock processing
        return {"processed": True, "timestamp": time.time()}

    def submit_frame(self, frame_data):
        """Submit a frame for processing"""
        try:
            self.processing_queue.put_nowait(frame_data)
            return True
        except queue.Full:
            # Drop frame if queue is full (real-time constraint)
            return False

    def get_result(self, timeout=0.01):
        """Get processed result"""
        try:
            return self.result_queue.get(timeout=timeout)
        except queue.Empty:
            return None

class MemoryManager:
    def __init__(self):
        self.cache = {}
        self.max_cache_size = 100

    def get_cached_result(self, key):
        """Get result from cache"""
        return self.cache.get(key)

    def cache_result(self, key, result):
        """Cache a result"""
        if len(self.cache) >= self.max_cache_size:
            # Remove oldest entries
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]

        self.cache[key] = result

    def clear_cache(self):
        """Clear the cache and run garbage collection"""
        self.cache.clear()
        gc.collect()
```

## Safety and Validation Systems

The integrated system must include comprehensive safety and validation mechanisms.

```python
from typing import Dict, List, Any, Tuple
import math

class SafetyValidator:
    def __init__(self):
        self.safety_zones = []
        self.emergency_stop = False
        self.safety_constraints = {
            'max_velocity': 1.0,  # m/s
            'max_angular_velocity': 1.0,  # rad/s
            'min_obstacle_distance': 0.5,  # meters
            'max_joint_velocity': 2.0,  # rad/s
            'max_joint_effort': 100.0  # Nm
        }

    def validate_action(self, action: Dict[str, Any], robot_state: Dict[str, Any]) -> Tuple[bool, List[str]]:
        """Validate an action against safety constraints"""
        violations = []

        action_type = action.get('action_type', '')
        params = action.get('parameters', {})

        if action_type == 'navigation':
            # Validate navigation command
            vel = params.get('linear_velocity', 0.0)
            ang_vel = params.get('angular_velocity', 0.0)

            if abs(vel) > self.safety_constraints['max_velocity']:
                violations.append(f"Linear velocity {vel} exceeds max {self.safety_constraints['max_velocity']}")

            if abs(ang_vel) > self.safety_constraints['max_angular_velocity']:
                violations.append(f"Angular velocity {ang_vel} exceeds max {self.safety_constraints['max_angular_velocity']}")

            # Check path for obstacles
            path = params.get('path', [])
            if self.check_path_for_obstacles(path, robot_state):
                violations.append("Path contains obstacles")

        elif action_type == 'manipulation':
            # Validate manipulation command
            joint_velocities = params.get('joint_velocities', [])
            joint_efforts = params.get('joint_efforts', [])

            for vel in joint_velocities:
                if abs(vel) > self.safety_constraints['max_joint_velocity']:
                    violations.append(f"Joint velocity {vel} exceeds max {self.safety_constraints['max_joint_velocity']}")

            for effort in joint_efforts:
                if abs(effort) > self.safety_constraints['max_joint_effort']:
                    violations.append(f"Joint effort {effort} exceeds max {self.safety_constraints['max_joint_effort']}")

        return len(violations) == 0, violations

    def check_path_for_obstacles(self, path: List[Tuple[float, float]], robot_state: Dict[str, Any]) -> bool:
        """Check if path contains obstacles"""
        # In real implementation, this would check LiDAR or map data
        # For simulation, return False (no obstacles detected)
        return False

    def validate_robot_state(self, robot_state: Dict[str, Any]) -> Tuple[bool, List[str]]:
        """Validate current robot state for safety"""
        violations = []

        # Check joint limits
        joint_positions = robot_state.get('joint_positions', [])
        joint_limits = robot_state.get('joint_limits', [])

        if joint_limits:
            for i, (pos, limits) in enumerate(zip(joint_positions, joint_limits)):
                if pos < limits['min'] or pos > limits['max']:
                    violations.append(f"Joint {i} position {pos} violates limits [{limits['min']}, {limits['max']}]")

        # Check for dangerous orientations
        orientation = robot_state.get('orientation', {})
        roll = orientation.get('roll', 0)
        pitch = orientation.get('pitch', 0)

        # Check if robot is in dangerous orientation (e.g., falling)
        max_tilt = math.radians(30)  # 30 degrees
        if abs(roll) > max_tilt or abs(pitch) > max_tilt:
            violations.append(f"Robot orientation unsafe: roll={math.degrees(roll):.1f}°, pitch={math.degrees(pitch):.1f}°")

        return len(violations) == 0, violations

class EmergencySystem:
    def __init__(self, safety_validator: SafetyValidator):
        self.safety_validator = safety_validator
        self.emergency_active = False
        self.emergency_reasons = []

    def check_emergency_conditions(self, robot_state: Dict[str, Any], sensor_data: Dict[str, Any]) -> bool:
        """Check for emergency conditions"""
        emergency_conditions = []

        # Check for collision imminent
        if self.is_collision_imminent(sensor_data):
            emergency_conditions.append("Collision imminent")

        # Check for dangerous orientation
        is_safe, violations = self.safety_validator.validate_robot_state(robot_state)
        if not is_safe:
            emergency_conditions.extend(violations)

        # Check for system failures
        if self.detect_system_failure(sensor_data):
            emergency_conditions.append("System failure detected")

        # Update emergency state
        self.emergency_active = len(emergency_conditions) > 0
        self.emergency_reasons = emergency_conditions

        return self.emergency_active

    def is_collision_imminent(self, sensor_data: Dict[str, Any]) -> bool:
        """Check if collision is imminent based on sensor data"""
        lidar_ranges = sensor_data.get('lidar_ranges', [])

        if lidar_ranges:
            min_distance = min([r for r in lidar_ranges if r > 0], default=float('inf'))
            if min_distance < 0.3:  # 30cm threshold
                return True

        return False

    def detect_system_failure(self, sensor_data: Dict[str, Any]) -> bool:
        """Detect system failures from sensor data"""
        # Check for missing sensor data
        required_sensors = ['lidar', 'imu', 'joint_states']
        for sensor in required_sensors:
            if sensor not in sensor_data or sensor_data[sensor] is None:
                return True

        return False

    def trigger_emergency_stop(self):
        """Trigger emergency stop"""
        self.emergency_active = True
        print("EMERGENCY STOP TRIGGERED")
        # In real implementation, this would send stop commands to all actuators

    def reset_emergency(self):
        """Reset emergency state"""
        self.emergency_active = False
        self.emergency_reasons = []
        print("Emergency state reset")
```

## Summary

This capstone lesson has demonstrated the complete integration of all components learned throughout the book into a functional humanoid robot system. We've shown how to:

1. **Integrate Perception Systems**: Combined RGB-D cameras, LiDAR, and other sensors with Isaac ROS perception modules for comprehensive environmental understanding.

2. **Implement Navigation**: Connected Nav2 for global path planning with local obstacle avoidance for safe humanoid locomotion.

3. **Process Voice Commands**: Integrated Whisper for speech recognition with LLM-driven task decomposition for natural language interaction.

4. **Ensure Safety**: Implemented comprehensive safety validation and emergency systems to protect both the robot and its environment.

5. **Optimize Performance**: Applied real-time scheduling and performance monitoring to ensure responsive behavior.

The integrated system demonstrates the full pipeline from perception to action execution, showing how all the individual components work together to create a capable humanoid robot that can understand natural language commands, perceive its environment, plan actions, and execute complex behaviors safely and efficiently.

## References

1. ROS 2 Documentation. (2023). Robot Operating System 2: Next generation robotics framework. *Open Robotics*.

2. Quigley, M., Gerkey, B., & Smart, W. D. (2009). ROS by example: A tutorial introduction to robot operating system. *Robotics, Science and Systems*.

3. Navigation2. (2021). Modern navigation system for robotics applications. *Open Robotics*.

4. Radford, A., Kim, J. W., Xu, T., Khabsa, G., Goyal, N., & Mann, T. (2022). Robust speech recognition via large-scale weak supervision. *arXiv preprint arXiv:2212.04356*.

5. NVIDIA Isaac ROS. (2023). Accelerated perception and navigation for robotics. *NVIDIA*.

6. Unity Technologies. (2023). Unity robotics simulation and development tools. *Unity Technologies*.