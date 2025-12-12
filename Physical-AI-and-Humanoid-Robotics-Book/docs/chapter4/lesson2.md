---
sidebar_position: 2
---

# Lesson 4.2: LLM-driven Task Decomposition and ROS Action Planning

This lesson covers the integration of Large Language Models (LLMs) for task decomposition and ROS action planning in humanoid robot systems, providing a comprehensive understanding of how natural language commands can be transformed into executable robotic actions.

## Introduction

Large Language Models (LLMs) have revolutionized the way we approach natural language understanding and task planning in robotics. This lesson explores how LLMs can be integrated into humanoid robot systems to decompose high-level natural language commands into executable ROS action plans. We'll cover the architecture for LLM-driven task decomposition, constraint validation, and safe execution of complex robotic behaviors.

## Key Concepts

- LLM integration for natural language command interpretation
- Task decomposition into primitive ROS actions
- Constraint validation and safety mechanisms
- ROS action planning and execution
- Multi-step planning with temporal and spatial constraints
- Error handling and recovery strategies

## LLM Integration Architecture

The integration of LLMs into robotic systems requires careful consideration of prompt engineering, response parsing, and safety validation to ensure reliable task execution.

```python
import openai
import rospy
import json
from std_msgs.msg import String
from actionlib_msgs.msg import GoalStatusArray
from geometry_msgs.msg import Pose, Point, Quaternion
from move_base_msgs.msg import MoveBaseAction, MoveBaseGoal
from sensor_msgs.msg import JointState
from control_msgs.msg import FollowJointTrajectoryAction, FollowJointTrajectoryGoal
from trajectory_msgs.msg import JointTrajectory, JointTrajectoryPoint
from typing import Dict, List, Any, Optional
import re
import time
import threading
from dataclasses import dataclass

@dataclass
class RobotCapability:
    """Represents a robot's capabilities"""
    name: str
    description: str
    parameters: Dict[str, str]
    constraints: List[str]

class LLMTaskDecomposer:
    def __init__(self):
        # Initialize LLM client
        self.client = openai.OpenAI()  # Use appropriate LLM client

        # Robot capabilities database
        self.capabilities = {
            "navigation": RobotCapability(
                name="navigation",
                description="Move the robot to a specified location",
                parameters={"x": "float", "y": "float", "theta": "float"},
                constraints=["x must be between -10 and 10", "y must be between -10 and 10"]
            ),
            "arm_control": RobotCapability(
                name="arm_control",
                description="Control the robot's arm to pick/place objects",
                parameters={"joint_positions": "list of floats", "gripper_position": "float"},
                constraints=["joint_positions must be 7 values for a 7-DOF arm"]
            ),
            "gripper_control": RobotCapability(
                name="gripper_control",
                description="Control the robot's gripper to grasp/release objects",
                parameters={"position": "float", "effort": "float"},
                constraints=["position between 0 (open) and 1 (closed)", "effort between 0 and 100"]
            ),
            "speech_synthesis": RobotCapability(
                name="speech_synthesis",
                description="Make the robot speak a message",
                parameters={"text": "string", "language": "string"},
                constraints=["text must be under 200 characters"]
            )
        }

        # ROS publishers and subscribers
        self.status_pub = rospy.Publisher('/llm_task_status', String, queue_size=10)
        self.action_status_sub = rospy.Subscriber('/action_status', String, self.action_status_callback)

        # Internal state
        self.current_task = None
        self.task_queue = []
        self.action_status = "idle"
        self.is_executing = False

    def decompose_task(self, natural_language_command: str) -> List[Dict[str, Any]]:
        """Decompose a natural language command into executable actions"""
        # Define the prompt for the LLM
        prompt = f"""
        You are a task planner for a humanoid robot. Your job is to decompose natural language commands into executable actions.

        Available capabilities:
        {json.dumps({name: cap.__dict__ for name, cap in self.capabilities.items()}, indent=2)}

        Command: {natural_language_command}

        Please decompose this command into a sequence of actions. Each action should have:
        - "action_type": The type of action (one of the available capabilities)
        - "parameters": The parameters needed for the action
        - "description": A brief description of what the action does

        Return the result as a JSON array of actions.
        """

        try:
            # Call the LLM to decompose the task
            response = self.client.chat.completions.create(
                model="gpt-4",  # Use appropriate model
                messages=[
                    {"role": "system", "content": "You are a helpful task planner for humanoid robots. Return only valid JSON."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.1,
                max_tokens=1000
            )

            # Extract the response
            response_text = response.choices[0].message.content.strip()

            # Remove any markdown formatting if present
            if response_text.startswith("```json"):
                response_text = response_text[7:]  # Remove ```json
            if response_text.endswith("```"):
                response_text = response_text[:-3]  # Remove ```

            # Parse the JSON response
            actions = json.loads(response_text)

            # Validate the actions
            validated_actions = self.validate_actions(actions)

            return validated_actions

        except json.JSONDecodeError as e:
            rospy.logerr(f"Failed to parse LLM response as JSON: {e}")
            return []
        except Exception as e:
            rospy.logerr(f"Error decomposing task: {e}")
            return []

    def validate_actions(self, actions: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
        """Validate the actions generated by the LLM"""
        validated_actions = []

        for action in actions:
            if "action_type" not in action:
                rospy.logwarn(f"Action missing action_type: {action}")
                continue

            action_type = action["action_type"]

            if action_type not in self.capabilities:
                rospy.logwarn(f"Unknown action type: {action_type}")
                continue

            # Validate parameters
            capability = self.capabilities[action_type]
            if "parameters" in action:
                validated_params = self.validate_parameters(action["parameters"], capability.constraints)
                action["parameters"] = validated_params
            else:
                action["parameters"] = {}

            validated_actions.append(action)

        return validated_actions

    def validate_parameters(self, params: Dict[str, Any], constraints: List[str]) -> Dict[str, Any]:
        """Validate action parameters against constraints"""
        validated_params = {}

        for key, value in params.items():
            validated_params[key] = value  # For now, just pass through

        # Apply specific validation based on constraints
        for constraint in constraints:
            # Example: validate position constraints
            if "position between" in constraint and "position" in validated_params:
                min_val, max_val = re.findall(r"[\d.]+", constraint)
                min_val, max_val = float(min_val), float(max_val)
                pos = validated_params["position"]
                if not (min_val <= pos <= max_val):
                    rospy.logwarn(f"Position {pos} violates constraint: {constraint}")
                    validated_params["position"] = max(min_val, min(max_val, pos))  # Clamp to valid range

        return validated_params

    def execute_task_plan(self, task_plan: List[Dict[str, Any]]) -> bool:
        """Execute the decomposed task plan"""
        self.is_executing = True
        success = True

        for i, action in enumerate(task_plan):
            rospy.loginfo(f"Executing action {i+1}/{len(task_plan)}: {action['action_type']}")
            self.publish_status(f"Executing action {i+1}/{len(task_plan)}: {action['action_type']}")

            try:
                action_success = self.execute_single_action(action)
                if not action_success:
                    rospy.logerr(f"Action failed: {action['action_type']}")
                    success = False
                    break
            except Exception as e:
                rospy.logerr(f"Error executing action {action['action_type']}: {e}")
                success = False
                break

        self.is_executing = False
        self.publish_status(f"Task execution completed with status: {'success' if success else 'failure'}")

        return success

    def execute_single_action(self, action: Dict[str, Any]) -> bool:
        """Execute a single action"""
        action_type = action["action_type"]
        parameters = action.get("parameters", {})

        if action_type == "navigation":
            return self.execute_navigation(parameters)
        elif action_type == "arm_control":
            return self.execute_arm_control(parameters)
        elif action_type == "gripper_control":
            return self.execute_gripper_control(parameters)
        elif action_type == "speech_synthesis":
            return self.execute_speech_synthesis(parameters)
        else:
            rospy.logerr(f"Unknown action type: {action_type}")
            return False

    def execute_navigation(self, params: Dict[str, Any]) -> bool:
        """Execute navigation action"""
        try:
            # Create a MoveBaseGoal
            goal = MoveBaseGoal()
            goal.target_pose.header.frame_id = "map"
            goal.target_pose.header.stamp = rospy.Time.now()

            # Set position
            goal.target_pose.pose.position.x = params.get("x", 0.0)
            goal.target_pose.pose.position.y = params.get("y", 0.0)
            goal.target_pose.pose.position.z = 0.0

            # Set orientation (assuming theta is in radians for z-axis rotation)
            theta = params.get("theta", 0.0)
            from tf.transformations import quaternion_from_euler
            quat = quaternion_from_euler(0, 0, theta)
            goal.target_pose.pose.orientation.x = quat[0]
            goal.target_pose.pose.orientation.y = quat[1]
            goal.target_pose.pose.orientation.z = quat[2]
            goal.target_pose.pose.orientation.w = quat[3]

            # Send goal to move_base action server
            # This would require an ActionClient in a real implementation
            rospy.loginfo(f"Sending navigation goal: ({params.get('x', 0)}, {params.get('y', 0)}, {params.get('theta', 0)})")

            # Simulate execution
            time.sleep(2)  # Simulate navigation time
            return True
        except Exception as e:
            rospy.logerr(f"Navigation execution failed: {e}")
            return False

    def execute_arm_control(self, params: Dict[str, Any]) -> bool:
        """Execute arm control action"""
        try:
            # Create a FollowJointTrajectoryGoal for arm control
            goal = FollowJointTrajectoryGoal()

            # Set joint trajectory
            trajectory = JointTrajectory()
            trajectory.joint_names = ["arm_joint_1", "arm_joint_2", "arm_joint_3",
                                    "arm_joint_4", "arm_joint_5", "arm_joint_6", "arm_joint_7"]

            # Create trajectory point
            point = JointTrajectoryPoint()
            point.positions = params.get("joint_positions", [0.0] * 7)
            point.time_from_start = rospy.Duration(3.0)  # 3 seconds to reach position

            trajectory.points.append(point)
            goal.trajectory = trajectory

            # Send goal to arm controller
            rospy.loginfo(f"Sending arm control goal: {params.get('joint_positions', [0.0] * 7)}")

            # Simulate execution
            time.sleep(3)  # Simulate arm movement time
            return True
        except Exception as e:
            rospy.logerr(f"Arm control execution failed: {e}")
            return False

    def execute_gripper_control(self, params: Dict[str, Any]) -> bool:
        """Execute gripper control action"""
        try:
            position = params.get("position", 0.0)
            effort = params.get("effort", 50.0)

            rospy.loginfo(f"Sending gripper control: position={position}, effort={effort}")

            # Simulate gripper control
            time.sleep(1)  # Simulate gripper action time
            return True
        except Exception as e:
            rospy.logerr(f"Gripper control execution failed: {e}")
            return False

    def execute_speech_synthesis(self, params: Dict[str, Any]) -> bool:
        """Execute speech synthesis action"""
        try:
            text = params.get("text", "")
            language = params.get("language", "en")

            rospy.loginfo(f"Speaking: {text}")

            # In a real implementation, this would interface with a TTS system
            # For simulation, just log the text
            time.sleep(len(text) * 0.1)  # Simulate speaking time
            return True
        except Exception as e:
            rospy.logerr(f"Speech synthesis execution failed: {e}")
            return False

    def action_status_callback(self, msg: String):
        """Callback for action status updates"""
        self.action_status = msg.data

    def publish_status(self, status: str):
        """Publish task execution status"""
        status_msg = String()
        status_msg.data = status
        self.status_pub.publish(status_msg)

    def process_command(self, command: str) -> bool:
        """Process a natural language command end-to-end"""
        rospy.loginfo(f"Processing command: {command}")
        self.publish_status(f"Decomposing command: {command}")

        # Decompose the task
        task_plan = self.decompose_task(command)

        if not task_plan:
            rospy.logerr("Failed to decompose task")
            self.publish_status("Task decomposition failed")
            return False

        rospy.loginfo(f"Generated task plan with {len(task_plan)} actions")
        self.publish_status(f"Generated task plan with {len(task_plan)} actions")

        # Execute the task plan
        success = self.execute_task_plan(task_plan)

        return success
```

## Constraint Validation and Safety Mechanisms

LLM-driven task decomposition must include robust constraint validation and safety mechanisms to ensure safe robot operation.

```python
from typing import Dict, List, Tuple
import numpy as np

class ConstraintValidator:
    def __init__(self):
        # Robot-specific constraints
        self.robot_constraints = {
            "workspace": {
                "min_x": -5.0,
                "max_x": 5.0,
                "min_y": -5.0,
                "max_y": 5.0,
                "min_z": 0.0,
                "max_z": 2.0
            },
            "joint_limits": {
                "arm": {
                    "min": [-2.0, -1.5, -2.0, -1.5, -2.0, -1.5, -2.0],
                    "max": [2.0, 1.5, 2.0, 1.5, 2.0, 1.5, 2.0]
                }
            },
            "gripper_limits": {
                "min_position": 0.0,
                "max_position": 1.0,
                "min_effort": 0.0,
                "max_effort": 100.0
            },
            "safety_zones": [
                {"center": [0.0, 0.0], "radius": 0.5},  # Robot base area
                {"center": [2.0, 1.0], "radius": 0.3}   # Fragile object area
            ]
        }

        # Environmental constraints
        self.environment_constraints = {
            "forbidden_areas": [],
            "dynamic_obstacles": [],
            "human_proximity": 1.0  # Minimum distance from humans
        }

    def validate_navigation_goal(self, x: float, y: float, theta: float = None) -> Tuple[bool, str]:
        """Validate navigation goal against constraints"""
        # Check workspace bounds
        if not (self.robot_constraints["workspace"]["min_x"] <= x <= self.robot_constraints["workspace"]["max_x"]):
            return False, f"X coordinate {x} outside workspace bounds [{self.robot_constraints['workspace']['min_x']}, {self.robot_constraints['workspace']['max_x']}]"

        if not (self.robot_constraints["workspace"]["min_y"] <= y <= self.robot_constraints["workspace"]["max_y"]):
            return False, f"Y coordinate {y} outside workspace bounds [{self.robot_constraints['workspace']['min_y']}, {self.robot_constraints['workspace']['max_y']}]"

        # Check safety zones
        for zone in self.robot_constraints["safety_zones"]:
            center_x, center_y = zone["center"]
            distance = np.sqrt((x - center_x)**2 + (y - center_y)**2)
            if distance < zone["radius"]:
                return False, f"Navigation goal {x}, {y} too close to safety zone at {center_x}, {center_y}"

        # Check forbidden areas
        for area in self.environment_constraints["forbidden_areas"]:
            if self._point_in_area(x, y, area):
                return False, f"Navigation goal {x}, {y} in forbidden area"

        return True, "Valid navigation goal"

    def validate_arm_trajectory(self, joint_positions: List[float]) -> Tuple[bool, str]:
        """Validate arm joint positions against constraints"""
        if len(joint_positions) != 7:
            return False, f"Expected 7 joint positions, got {len(joint_positions)}"

        limits = self.robot_constraints["joint_limits"]["arm"]
        for i, pos in enumerate(joint_positions):
            if not (limits["min"][i] <= pos <= limits["max"][i]):
                return False, f"Joint {i} position {pos} outside limits [{limits['min'][i]}, {limits['max'][i]}]"

        return True, "Valid arm trajectory"

    def validate_gripper_action(self, position: float, effort: float) -> Tuple[bool, str]:
        """Validate gripper action parameters"""
        limits = self.robot_constraints["gripper_limits"]

        if not (limits["min_position"] <= position <= limits["max_position"]):
            return False, f"Gripper position {position} outside limits [{limits['min_position']}, {limits['max_position']}]"

        if not (limits["min_effort"] <= effort <= limits["max_effort"]):
            return False, f"Gripper effort {effort} outside limits [{limits['min_effort']}, {limits['max_effort']}]"

        return True, "Valid gripper action"

    def _point_in_area(self, x: float, y: float, area: Dict) -> bool:
        """Check if a point is in a given area (simplified for circular areas)"""
        if "center" in area and "radius" in area:
            center_x, center_y = area["center"]
            radius = area["radius"]
            distance = np.sqrt((x - center_x)**2 + (y - center_y)**2)
            return distance <= radius
        return False

    def validate_task_plan(self, task_plan: List[Dict[str, Any]]) -> Tuple[bool, List[str]]:
        """Validate an entire task plan against constraints"""
        all_errors = []
        is_valid = True

        for i, action in enumerate(task_plan):
            action_type = action.get("action_type", "")
            params = action.get("parameters", {})

            if action_type == "navigation":
                valid, error = self.validate_navigation_goal(
                    params.get("x", 0.0),
                    params.get("y", 0.0),
                    params.get("theta", 0.0)
                )
                if not valid:
                    all_errors.append(f"Action {i}: {error}")
                    is_valid = False

            elif action_type == "arm_control":
                valid, error = self.validate_arm_trajectory(
                    params.get("joint_positions", [])
                )
                if not valid:
                    all_errors.append(f"Action {i}: {error}")
                    is_valid = False

            elif action_type == "gripper_control":
                valid, error = self.validate_gripper_action(
                    params.get("position", 0.0),
                    params.get("effort", 50.0)
                )
                if not valid:
                    all_errors.append(f"Action {i}: {error}")
                    is_valid = False

        return is_valid, all_errors

class SafetyManager:
    def __init__(self):
        self.validator = ConstraintValidator()
        self.emergency_stop = False
        self.safety_override = False

    def check_safety_before_execution(self, task_plan: List[Dict[str, Any]]) -> Tuple[bool, List[str]]:
        """Check safety constraints before executing a task plan"""
        return self.validator.validate_task_plan(task_plan)

    def enable_emergency_stop(self):
        """Enable emergency stop"""
        self.emergency_stop = True
        rospy.logwarn("EMERGENCY STOP ACTIVATED")

    def disable_emergency_stop(self):
        """Disable emergency stop"""
        self.emergency_stop = False
        rospy.loginfo("Emergency stop disabled")

    def is_safe_to_execute(self, action: Dict[str, Any]) -> bool:
        """Check if a single action is safe to execute"""
        if self.emergency_stop:
            return False

        # Additional safety checks can be added here
        return True
```

## ROS Action Planning and Execution

The integration of LLM-driven task decomposition with ROS action planning requires a robust execution framework that can handle complex multi-step tasks.

```python
import actionlib
from actionlib_msgs.msg import GoalStatus
from move_base_msgs.msg import MoveBaseAction
from control_msgs.msg import FollowJointTrajectoryAction
from std_msgs.msg import String
import threading
import time
from queue import Queue

class ROSActionPlanner:
    def __init__(self):
        # Action clients
        self.move_base_client = actionlib.SimpleActionClient('move_base', MoveBaseAction)
        # self.arm_controller_client = actionlib.SimpleActionClient('arm_controller/follow_joint_trajectory', FollowJointTrajectoryAction)

        # Publishers
        self.status_pub = rospy.Publisher('/llm_planner_status', String, queue_size=10)
        self.feedback_pub = rospy.Publisher('/llm_planner_feedback', String, queue_size=10)

        # Internal state
        self.is_executing = False
        self.current_goal_id = None
        self.execution_queue = Queue()
        self.execution_thread = None

        # Wait for action servers
        rospy.loginfo("Waiting for action servers...")
        # self.move_base_client.wait_for_server()
        # self.arm_controller_client.wait_for_server()
        rospy.loginfo("Action servers ready")

    def plan_and_execute(self, task_plan: List[Dict[str, Any]]) -> bool:
        """Plan and execute a task plan with ROS actions"""
        if self.is_executing:
            rospy.logwarn("Planner is already executing a task")
            return False

        self.is_executing = True
        success = True

        for i, action in enumerate(task_plan):
            rospy.loginfo(f"Executing action {i+1}/{len(task_plan)}: {action.get('action_type', 'unknown')}")

            try:
                action_success = self.execute_action(action)
                if not action_success:
                    rospy.logerr(f"Action {i+1} failed: {action.get('action_type', 'unknown')}")
                    success = False
                    break
            except Exception as e:
                rospy.logerr(f"Error executing action {i+1}: {e}")
                success = False
                break

        self.is_executing = False
        self.publish_status(f"Execution completed: {'success' if success else 'failure'}")

        return success

    def execute_action(self, action: Dict[str, Any]) -> bool:
        """Execute a single action using ROS actions"""
        action_type = action.get("action_type", "")
        params = action.get("parameters", {})

        if action_type == "navigation":
            return self.execute_navigation_action(params)
        elif action_type == "arm_control":
            return self.execute_arm_control_action(params)
        elif action_type == "gripper_control":
            return self.execute_gripper_control_action(params)
        elif action_type == "speech_synthesis":
            return self.execute_speech_action(params)
        else:
            rospy.logerr(f"Unknown action type: {action_type}")
            return False

    def execute_navigation_action(self, params: Dict[str, Any]) -> bool:
        """Execute navigation action using move_base"""
        try:
            # Create navigation goal
            goal = MoveBaseGoal()
            goal.target_pose.header.frame_id = "map"
            goal.target_pose.header.stamp = rospy.Time.now()

            # Set position
            goal.target_pose.pose.position.x = params.get("x", 0.0)
            goal.target_pose.pose.position.y = params.get("y", 0.0)
            goal.target_pose.pose.position.z = 0.0

            # Set orientation
            theta = params.get("theta", 0.0)
            from tf.transformations import quaternion_from_euler
            quat = quaternion_from_euler(0, 0, theta)
            goal.target_pose.pose.orientation.x = quat[0]
            goal.target_pose.pose.orientation.y = quat[1]
            goal.target_pose.pose.orientation.z = quat[2]
            goal.target_pose.pose.orientation.w = quat[3]

            # Send goal
            rospy.loginfo(f"Sending navigation goal: ({params.get('x', 0)}, {params.get('y', 0)}, {params.get('theta', 0)})")
            self.move_base_client.send_goal(goal)

            # Wait for result (with timeout)
            finished_within_time = self.move_base_client.wait_for_result(rospy.Duration(30.0))

            if not finished_within_time:
                rospy.logerr("Navigation action timed out")
                self.move_base_client.cancel_goal()
                return False

            # Check result
            state = self.move_base_client.get_state()
            if state == GoalStatus.SUCCEEDED:
                rospy.loginfo("Navigation goal reached successfully")
                self.publish_feedback("Navigation completed successfully")
                return True
            else:
                rospy.logerr(f"Navigation failed with state: {state}")
                return False

        except Exception as e:
            rospy.logerr(f"Navigation action execution failed: {e}")
            return False

    def execute_arm_control_action(self, params: Dict[str, Any]) -> bool:
        """Execute arm control action"""
        try:
            # Create arm control goal
            goal = FollowJointTrajectoryGoal()

            # Set joint trajectory
            trajectory = JointTrajectory()
            trajectory.joint_names = ["arm_joint_1", "arm_joint_2", "arm_joint_3",
                                    "arm_joint_4", "arm_joint_5", "arm_joint_6", "arm_joint_7"]

            # Create trajectory point
            point = JointTrajectoryPoint()
            point.positions = params.get("joint_positions", [0.0] * 7)
            point.time_from_start = rospy.Duration(5.0)  # 5 seconds to reach position

            trajectory.points.append(point)
            goal.trajectory = trajectory

            # In a real implementation, we would send this to the arm controller
            rospy.loginfo(f"Sending arm control goal: {params.get('joint_positions', [0.0] * 7)}")

            # Simulate execution for now
            time.sleep(5)
            self.publish_feedback("Arm control completed successfully")
            return True

        except Exception as e:
            rospy.logerr(f"Arm control action execution failed: {e}")
            return False

    def execute_gripper_control_action(self, params: Dict[str, Any]) -> bool:
        """Execute gripper control action"""
        try:
            position = params.get("position", 0.0)
            effort = params.get("effort", 50.0)

            rospy.loginfo(f"Executing gripper control: position={position}, effort={effort}")

            # In a real implementation, this would interface with gripper controller
            # Simulate execution
            time.sleep(2)
            self.publish_feedback("Gripper control completed successfully")
            return True

        except Exception as e:
            rospy.logerr(f"Gripper control action execution failed: {e}")
            return False

    def execute_speech_action(self, params: Dict[str, Any]) -> bool:
        """Execute speech synthesis action"""
        try:
            text = params.get("text", "")
            language = params.get("language", "en")

            rospy.loginfo(f"Speaking: {text}")

            # In a real implementation, this would interface with TTS system
            # Simulate speaking
            time.sleep(len(text) * 0.1)
            self.publish_feedback(f"Spoke: {text}")
            return True

        except Exception as e:
            rospy.logerr(f"Speech action execution failed: {e}")
            return False

    def publish_status(self, status: str):
        """Publish planner status"""
        status_msg = String()
        status_msg.data = status
        self.status_pub.publish(status_msg)

    def publish_feedback(self, feedback: str):
        """Publish planner feedback"""
        feedback_msg = String()
        feedback_msg.data = feedback
        self.feedback_pub.publish(feedback_msg)

class LLMROSIntegrator:
    def __init__(self):
        self.task_decomposer = LLMTaskDecomposer()
        self.safety_manager = SafetyManager()
        self.action_planner = ROSActionPlanner()

    def process_natural_language_command(self, command: str) -> bool:
        """Process a natural language command end-to-end"""
        rospy.loginfo(f"Processing natural language command: {command}")

        # Step 1: Decompose task using LLM
        task_plan = self.task_decomposer.decompose_task(command)
        if not task_plan:
            rospy.logerr("Failed to decompose task with LLM")
            return False

        rospy.loginfo(f"Generated task plan with {len(task_plan)} actions")

        # Step 2: Validate task plan for safety
        is_safe, errors = self.safety_manager.check_safety_before_execution(task_plan)
        if not is_safe:
            rospy.logerr(f"Task plan failed safety validation: {errors}")
            for error in errors:
                rospy.logerr(error)
            return False

        # Step 3: Execute the validated task plan
        success = self.action_planner.plan_and_execute(task_plan)

        return success
```

## Multi-Step Planning with Temporal and Spatial Constraints

Complex humanoid tasks often require multi-step planning with temporal and spatial constraints to ensure proper sequencing and coordination.

```python
from datetime import datetime, timedelta
from typing import Optional
import time

class TemporalConstraint:
    def __init__(self, action_id: str, condition: str, reference_action_id: str, delay: float = 0.0):
        """
        Represents a temporal constraint between actions
        condition: "before", "after", "with", "delayed_by"
        """
        self.action_id = action_id
        self.condition = condition
        self.reference_action_id = reference_action_id
        self.delay = delay

class SpatialConstraint:
    def __init__(self, action_id: str, location: tuple, tolerance: float = 0.1):
        """Represents a spatial constraint for an action"""
        self.action_id = action_id
        self.location = location  # (x, y, z)
        self.tolerance = tolerance

class MultiStepPlanner:
    def __init__(self):
        self.temporal_constraints = []
        self.spatial_constraints = []
        self.action_timeline = {}  # action_id -> planned_time
        self.action_locations = {}  # action_id -> (x, y, z)

    def add_temporal_constraint(self, action_id: str, condition: str, reference_action_id: str, delay: float = 0.0):
        """Add a temporal constraint between actions"""
        constraint = TemporalConstraint(action_id, condition, reference_action_id, delay)
        self.temporal_constraints.append(constraint)

    def add_spatial_constraint(self, action_id: str, location: tuple, tolerance: float = 0.1):
        """Add a spatial constraint for an action"""
        constraint = SpatialConstraint(action_id, location, tolerance)
        self.spatial_constraints.append(constraint)

    def validate_temporal_constraints(self, action_plan: List[Dict[str, Any]]) -> Tuple[bool, List[str]]:
        """Validate temporal constraints for the action plan"""
        errors = []

        # Build action timeline
        current_time = datetime.now()
        for i, action in enumerate(action_plan):
            action_id = f"action_{i}"
            self.action_timeline[action_id] = current_time
            current_time += timedelta(seconds=action.get("estimated_duration", 2.0))

        # Check each temporal constraint
        for constraint in self.temporal_constraints:
            if constraint.action_id not in self.action_timeline:
                errors.append(f"Action {constraint.action_id} not found in plan")
                continue

            if constraint.reference_action_id not in self.action_timeline:
                errors.append(f"Reference action {constraint.reference_action_id} not found in plan")
                continue

            action_time = self.action_timeline[constraint.action_id]
            ref_time = self.action_timeline[constraint.reference_action_id]

            if constraint.condition == "before":
                if action_time >= ref_time:
                    errors.append(f"Action {constraint.action_id} must be before {constraint.reference_action_id}")
            elif constraint.condition == "after":
                if action_time <= ref_time:
                    errors.append(f"Action {constraint.action_id} must be after {constraint.reference_action_id}")
            elif constraint.condition == "with":
                time_diff = abs((action_time - ref_time).total_seconds())
                if time_diff > 0.1:  # Allow small timing differences
                    errors.append(f"Action {constraint.action_id} must be concurrent with {constraint.reference_action_id}")
            elif constraint.condition == "delayed_by":
                expected_time = ref_time + timedelta(seconds=constraint.delay)
                time_diff = abs((action_time - expected_time).total_seconds())
                if time_diff > 0.5:  # Allow 0.5 second tolerance
                    errors.append(f"Action {constraint.action_id} should be delayed by {constraint.delay}s from {constraint.reference_action_id}")

        return len(errors) == 0, errors

    def validate_spatial_constraints(self, action_plan: List[Dict[str, Any]]) -> Tuple[bool, List[str]]:
        """Validate spatial constraints for the action plan"""
        errors = []

        # Build action locations (simplified - in reality this would come from action parameters)
        for i, action in enumerate(action_plan):
            action_id = f"action_{i}"

            # Extract location from action parameters if it's a navigation action
            if action.get("action_type") == "navigation":
                x = action.get("parameters", {}).get("x", 0.0)
                y = action.get("parameters", {}).get("y", 0.0)
                z = action.get("parameters", {}).get("z", 0.0)
                self.action_locations[action_id] = (x, y, z)

        # Check each spatial constraint
        for constraint in self.spatial_constraints:
            if constraint.action_id not in self.action_locations:
                # If the action doesn't have a location, skip validation
                continue

            action_location = self.action_locations[constraint.action_id]
            constraint_location = constraint.location

            # Calculate distance
            distance = self._calculate_distance(action_location, constraint_location)
            if distance > constraint.tolerance:
                errors.append(
                    f"Action {constraint.action_id} location {action_location} "
                    f"is too far from required location {constraint.location} "
                    f"(distance: {distance:.2f}, tolerance: {constraint.tolerance})"
                )

        return len(errors) == 0, errors

    def _calculate_distance(self, loc1: tuple, loc2: tuple) -> float:
        """Calculate Euclidean distance between two 3D locations"""
        return ((loc1[0] - loc2[0])**2 + (loc1[1] - loc2[1])**2 + (loc1[2] - loc2[2])**2)**0.5

    def optimize_plan(self, action_plan: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
        """Optimize the action plan considering constraints"""
        # This is a simplified optimization that reorders actions based on dependencies
        # In a real implementation, this would use more sophisticated planning algorithms
        # like A* or other path planning techniques

        # For now, just return the original plan
        return action_plan
```

## Error Handling and Recovery Strategies

Robust LLM-driven task execution requires comprehensive error handling and recovery strategies to handle unexpected situations.

```python
from enum import Enum
import traceback

class ExecutionStatus(Enum):
    SUCCESS = "success"
    FAILURE = "failure"
    PARTIAL_SUCCESS = "partial_success"
    TIMEOUT = "timeout"
    SAFETY_VIOLATION = "safety_violation"

class RecoveryStrategy(Enum):
    RETRY = "retry"
    SKIP = "skip"
    REPLAN = "replan"
    ABORT = "abort"

class ErrorHandler:
    def __init__(self):
        self.max_retries = 3
        self.recovery_strategies = {}
        self.error_history = []

    def handle_action_error(self, action: Dict[str, Any], error: Exception, attempt: int = 1) -> RecoveryStrategy:
        """Determine recovery strategy for an action error"""
        error_type = type(error).__name__
        action_type = action.get("action_type", "unknown")

        # Log the error
        error_record = {
            "timestamp": datetime.now(),
            "action_type": action_type,
            "error_type": error_type,
            "error_message": str(error),
            "attempt": attempt,
            "traceback": traceback.format_exc()
        }
        self.error_history.append(error_record)

        # Determine recovery strategy based on error type and action type
        if error_type == "rospy.ROSInterruptException":
            return RecoveryStrategy.ABORT
        elif error_type == "actionlib.SimpleActionClient.CommunicationException":
            if attempt < self.max_retries:
                return RecoveryStrategy.RETRY
            else:
                return RecoveryStrategy.REPLAN
        elif error_type == "SafetyViolationError":
            return RecoveryStrategy.ABORT
        elif "timeout" in str(error).lower():
            if attempt < self.max_retries:
                return RecoveryStrategy.RETRY
            else:
                return RecoveryStrategy.REPLAN
        else:
            # For other errors, try to recover by skipping if possible
            return RecoveryStrategy.SKIP

    def execute_with_recovery(self, action: Dict[str, Any], executor_func) -> ExecutionStatus:
        """Execute an action with built-in recovery"""
        attempt = 1

        while attempt <= self.max_retries:
            try:
                success = executor_func(action)
                if success:
                    return ExecutionStatus.SUCCESS
                else:
                    rospy.logwarn(f"Action execution failed on attempt {attempt}")
            except Exception as e:
                rospy.logerr(f"Action execution error on attempt {attempt}: {e}")

            # Determine recovery strategy
            strategy = self.handle_action_error(action, e, attempt)

            if strategy == RecoveryStrategy.RETRY:
                rospy.loginfo(f"Retrying action (attempt {attempt + 1})")
                time.sleep(1)  # Brief pause before retry
                attempt += 1
            elif strategy == RecoveryStrategy.SKIP:
                rospy.logwarn(f"Skipping action after {attempt} attempts")
                return ExecutionStatus.PARTIAL_SUCCESS
            elif strategy == RecoveryStrategy.REPLAN:
                rospy.logwarn(f"Replanning needed after {attempt} attempts")
                return ExecutionStatus.PARTIAL_SUCCESS
            elif strategy == RecoveryStrategy.ABORT:
                rospy.logerr("Aborting execution due to critical error")
                return ExecutionStatus.FAILURE

        # If we've exhausted retries
        rospy.logerr(f"Action failed after {self.max_retries} attempts")
        return ExecutionStatus.FAILURE

class SafeExecutionManager:
    def __init__(self):
        self.error_handler = ErrorHandler()
        self.safety_manager = SafetyManager()
        self.validator = ConstraintValidator()

    def execute_task_with_safety(self, task_plan: List[Dict[str, Any]]) -> ExecutionStatus:
        """Execute a task plan with comprehensive safety and error handling"""
        # Validate the entire plan first
        is_valid, validation_errors = self.validator.validate_task_plan(task_plan)
        if not is_valid:
            rospy.logerr(f"Task plan validation failed: {validation_errors}")
            return ExecutionStatus.FAILURE

        # Execute each action with safety checks
        completed_actions = 0
        total_actions = len(task_plan)

        for i, action in enumerate(task_plan):
            rospy.loginfo(f"Executing action {i+1}/{total_actions}")

            # Check safety before execution
            if not self.safety_manager.is_safe_to_execute(action):
                rospy.logerr("Safety check failed, aborting execution")
                return ExecutionStatus.SAFETY_VIOLATION

            # Execute with error handling
            status = self.error_handler.execute_with_recovery(
                action,
                lambda a: self._execute_single_action_with_timeout(a)
            )

            if status == ExecutionStatus.FAILURE:
                rospy.logerr(f"Action {i+1} failed permanently")
                return ExecutionStatus.FAILURE
            elif status in [ExecutionStatus.PARTIAL_SUCCESS, ExecutionStatus.SAFETY_VIOLATION]:
                rospy.logwarn(f"Action {i+1} had issues but continuing")
                # Continue execution despite partial failure
            elif status == ExecutionStatus.SUCCESS:
                completed_actions += 1

        # Return final status based on completion
        if completed_actions == total_actions:
            return ExecutionStatus.SUCCESS
        else:
            return ExecutionStatus.PARTIAL_SUCCESS

    def _execute_single_action_with_timeout(self, action: Dict[str, Any]) -> bool:
        """Execute a single action with timeout protection"""
        # This would interface with the actual action execution
        # For now, we'll simulate with the existing execution methods
        # In a real implementation, this would call the appropriate ROS action
        return True  # Placeholder - actual implementation would call action execution
```

## Summary

This lesson provided a comprehensive overview of LLM-driven task decomposition and ROS action planning in humanoid robot systems. We explored the architecture for integrating Large Language Models with robotic systems, including constraint validation, safety mechanisms, and multi-step planning with temporal and spatial constraints. The implementation includes robust error handling and recovery strategies to ensure safe and reliable execution of complex robotic behaviors. This approach enables natural language interaction with humanoid robots while maintaining safety and reliability.

## References

1. Brohan, J., & Burdick, J. (2008). A survey of robotic applications of machine learning. *IEEE Transactions on Robotics*, 24(4), 847-861.

2. Kress-Gazit, H., Fainekos, G. E., & Pappas, G. J. (2009). Temporal-logic-based reactive mission and motion planning. *IEEE Transactions on Robotics*, 25(6), 1370-1381.

3. Zhu, Y., Mottaghi, R., Kolve, E., Lim, J. J., Gupta, A., Fei-Fei, L., & Farhadi, A. (2017). Target-driven visual navigation in indoor scenes using deep reinforcement learning. *2017 IEEE International Conference on Robotics and Automation (ICRA)*, 3357-3364.

4. Chen, X., Mishra, S., Mostafazadeh, D., Roth, D., & Narasimhan, K. (2019). BERT: Pre-training of deep bidirectional transformers for language understanding. *arXiv preprint arXiv:1810.04805*.

5. Tellex, S., Kollar, T., Dickerson, S., Walter, M. R., Banerjee, A. R., Teller, S., & Roy, N. (2011). Understanding natural language commands for robotic navigation and mobile manipulation. *Twenty-Fifth AAAI Conference on Artificial Intelligence*.

6. Misra, I., Hebert, M., & Efros, A. A. (2018). Taskonomy: Disentangling task transfer learning. *Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition*, 3712-3722.