---
sidebar_position: 4
---

# Full Pipeline Example: Autonomous Humanoid Workflow

This document provides a complete example of the full pipeline that integrates all concepts from the book into a single, functional workflow.

## Complete Pipeline Overview

The complete pipeline demonstrates the end-to-end workflow from voice command to robot action execution:

1. **Voice Input**: User speaks a command
2. **Transcription**: Whisper converts speech to text
3. **Planning**: LLM generates an action plan
4. **Validation**: Safety systems validate the plan
5. **Execution**: ROS 2 executes the plan in simulation
6. **Perception**: Robot senses and responds to the environment
7. **Navigation**: Robot moves to target locations
8. **Manipulation**: Robot performs requested actions

## Complete Implementation Example

```python
#!/usr/bin/env python3
"""
Complete pipeline example for the Autonomous Humanoid System
Integrates all four layers of humanoid intelligence
"""

import rclpy
from rclpy.node import Node
from std_msgs.msg import String
from geometry_msgs.msg import PoseStamped, Twist
from sensor_msgs.msg import LaserScan, Image, Imu
from std_srvs.srv import Trigger
import whisper
import threading
import time
import math

class AutonomousHumanoidPipeline(Node):
    def __init__(self):
        super().__init__('autonomous_humanoid_pipeline')

        # State management
        self.current_state = "IDLE"
        self.voice_command = ""
        self.action_plan = []
        self.current_step = 0

        # Initialize Whisper model
        self.whisper_model = whisper.load_model("base")

        # Publishers
        self.transcribed_pub = self.create_publisher(String, 'transcribed_command', 10)
        self.action_plan_pub = self.create_publisher(String, 'action_plan', 10)
        self.validated_plan_pub = self.create_publisher(String, 'validated_action_plan', 10)
        self.cmd_vel_pub = self.create_publisher(Twist, 'cmd_vel', 10)
        self.goal_pub = self.create_publisher(PoseStamped, 'goal_pose', 10)

        # Subscribers
        self.perception_sub = self.create_subscription(String, 'perception_data',
                                                      self.perception_callback, 10)
        self.nav_status_sub = self.create_subscription(String, 'navigation_status',
                                                      self.nav_status_callback, 10)

        # Services
        self.emergency_stop_client = self.create_client(Trigger, 'emergency_stop')

        # Timers
        self.pipeline_timer = self.create_timer(1.0, self.pipeline_step)

        # Start voice recognition
        self.start_voice_recognition()

    def start_voice_recognition(self):
        """Start voice recognition in a separate thread"""
        self.recognition_thread = threading.Thread(target=self.voice_recognition_loop)
        self.recognition_thread.daemon = True
        self.recognition_thread.start()

    def voice_recognition_loop(self):
        """Continuously listen for voice commands"""
        import speech_recognition as sr

        recognizer = sr.Recognizer()
        microphone = sr.Microphone()

        with microphone as source:
            recognizer.adjust_for_ambient_noise(source)

        while rclpy.ok():
            try:
                with microphone as source:
                    self.get_logger().info("Listening for voice command...")
                    audio = recognizer.listen(source, timeout=5.0)

                # Transcribe using Whisper
                result = self.whisper_model.transcribe(audio.get_wav_data())
                command = result["text"]

                if command.strip():
                    self.get_logger().info(f"Heard command: {command}")
                    self.voice_command = command

                    # Publish transcribed command
                    msg = String()
                    msg.data = command
                    self.transcribed_pub.publish(msg)

            except sr.WaitTimeoutError:
                pass  # Continue listening
            except Exception as e:
                self.get_logger().error(f"Voice recognition error: {e}")
                time.sleep(1)

    def generate_action_plan(self, command):
        """Generate an action plan using LLM simulation"""
        # In a real implementation, this would call an LLM API
        # For this example, we'll create a mock plan based on command keywords

        command_lower = command.lower()

        if "fetch" in command_lower or "get" in command_lower or "bring" in command_lower:
            # Fetch object command
            plan = [
                "NAVIGATE_TO_OBJECT_LOCATION",
                "DETECT_OBJECT_VISUALLY",
                "APPROACH_OBJECT",
                "GRASP_OBJECT",
                "NAVIGATE_TO_RETURN_LOCATION",
                "PLACE_OBJECT"
            ]
        elif "go to" in command_lower or "move to" in command_lower:
            # Navigation command
            plan = [
                "PARSE_TARGET_LOCATION",
                "NAVIGATE_TO_TARGET",
                "CONFIRM_ARRIVAL"
            ]
        elif "turn" in command_lower or "rotate" in command_lower:
            # Rotation command
            plan = [
                "DETERMINE_ROTATION_ANGLE",
                "EXECUTE_ROTATION",
                "STABILIZE_POSTURE"
            ]
        else:
            # Default plan
            plan = [
                "ACKNOWLEDGE_COMMAND",
                "DETERMINE_ACTION_TYPE",
                "GENERATE_DETAILED_PLAN"
            ]

        return plan

    def validate_action_plan(self, plan):
        """Validate the action plan for safety"""
        # Check each step for safety constraints
        for step in plan:
            if not self.is_step_safe(step):
                return False, f"Unsafe step detected: {step}"

        return True, "Plan is safe"

    def is_step_safe(self, step):
        """Check if individual step is safe"""
        # In a real implementation, this would check:
        # - Joint limits
        # - Collision avoidance
        # - Stability constraints
        # - Workspace bounds

        # For this example, assume most steps are safe
        unsafe_keywords = ["dangerous", "impossible", "break", "crash"]
        step_lower = step.lower()

        for keyword in unsafe_keywords:
            if keyword in step_lower:
                return False

        return True

    def execute_action_step(self, step):
        """Execute a single action step"""
        self.get_logger().info(f"Executing step: {step}")

        if step == "NAVIGATE_TO_OBJECT_LOCATION":
            self.navigate_to_object()
        elif step == "DETECT_OBJECT_VISUALLY":
            self.detect_object()
        elif step == "APPROACH_OBJECT":
            self.approach_object()
        elif step == "GRASP_OBJECT":
            self.grasp_object()
        elif step == "NAVIGATE_TO_RETURN_LOCATION":
            self.navigate_to_return()
        elif step == "PLACE_OBJECT":
            self.place_object()
        elif step == "PARSE_TARGET_LOCATION":
            self.parse_target_location()
        elif step == "NAVIGATE_TO_TARGET":
            self.navigate_to_target()
        elif step == "CONFIRM_ARRIVAL":
            self.confirm_arrival()
        elif step == "DETERMINE_ROTATION_ANGLE":
            self.determine_rotation_angle()
        elif step == "EXECUTE_ROTATION":
            self.execute_rotation()
        elif step == "STABILIZE_POSTURE":
            self.stabilize_posture()
        else:
            self.get_logger().warn(f"Unknown action step: {step}")

        # Add a small delay to allow for completion
        time.sleep(0.5)

    def navigate_to_object(self):
        """Navigate to object location"""
        goal = PoseStamped()
        goal.header.frame_id = "map"
        goal.pose.position.x = 1.0  # Example coordinates
        goal.pose.position.y = 2.0
        goal.pose.orientation.w = 1.0
        self.goal_pub.publish(goal)

    def detect_object(self):
        """Detect object visually"""
        # This would integrate with perception system
        self.get_logger().info("Detecting object...")

    def approach_object(self):
        """Approach the detected object"""
        twist = Twist()
        twist.linear.x = 0.2  # Move forward slowly
        self.cmd_vel_pub.publish(twist)

    def grasp_object(self):
        """Grasp the object"""
        # This would control the humanoid's hand
        self.get_logger().info("Grasping object...")

    def navigate_to_return(self):
        """Navigate back to starting position"""
        goal = PoseStamped()
        goal.header.frame_id = "map"
        goal.pose.position.x = 0.0
        goal.pose.position.y = 0.0
        goal.pose.orientation.w = 1.0
        self.goal_pub.publish(goal)

    def place_object(self):
        """Place the object down"""
        # This would control the humanoid's hand
        self.get_logger().info("Placing object...")

    def parse_target_location(self):
        """Parse target location from command"""
        self.get_logger().info("Parsing target location...")

    def navigate_to_target(self):
        """Navigate to target location"""
        self.get_logger().info("Navigating to target...")

    def confirm_arrival(self):
        """Confirm arrival at destination"""
        self.get_logger().info("Arrived at destination!")

    def determine_rotation_angle(self):
        """Determine rotation angle"""
        self.get_logger().info("Determining rotation angle...")

    def execute_rotation(self):
        """Execute rotation"""
        twist = Twist()
        twist.angular.z = 0.5  # Rotate at 0.5 rad/s
        self.cmd_vel_pub.publish(twist)

    def stabilize_posture(self):
        """Stabilize humanoid posture"""
        self.get_logger().info("Stabilizing posture...")

    def perception_callback(self, msg):
        """Handle perception data"""
        self.get_logger().info(f"Perception update: {msg.data}")

    def nav_status_callback(self, msg):
        """Handle navigation status"""
        self.get_logger().info(f"Navigation status: {msg.data}")

    def pipeline_step(self):
        """Main pipeline execution step"""
        if self.voice_command and self.current_state == "IDLE":
            self.get_logger().info(f"Processing command: {self.voice_command}")
            self.current_state = "PLANNING"

            # Generate action plan
            self.action_plan = self.generate_action_plan(self.voice_command)
            self.current_step = 0

            # Publish action plan
            plan_msg = String()
            plan_msg.data = str(self.action_plan)
            self.action_plan_pub.publish(plan_msg)

            self.get_logger().info(f"Generated plan: {self.action_plan}")

        elif self.current_state == "PLANNING":
            # Validate the plan
            is_safe, message = self.validate_action_plan(self.action_plan)

            if is_safe:
                self.get_logger().info("Plan is safe, starting execution")
                self.current_state = "EXECUTING"

                # Publish validated plan
                validated_msg = String()
                validated_msg.data = str(self.action_plan)
                self.validated_plan_pub.publish(validated_msg)
            else:
                self.get_logger().error(f"Plan validation failed: {message}")
                self.current_state = "IDLE"
                self.voice_command = ""

        elif self.current_state == "EXECUTING":
            if self.current_step < len(self.action_plan):
                # Execute current step
                current_action = self.action_plan[self.current_step]
                self.execute_action_step(current_action)

                self.current_step += 1
                self.get_logger().info(f"Completed step {self.current_step} of {len(self.action_plan)}")
            else:
                # All steps completed
                self.get_logger().info("All steps completed successfully!")
                self.current_state = "IDLE"
                self.voice_command = ""

def main(args=None):
    rclpy.init(args=args)

    pipeline_node = AutonomousHumanoidPipeline()

    try:
        rclpy.spin(pipeline_node)
    except KeyboardInterrupt:
        pass
    finally:
        pipeline_node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Pipeline Execution Flow

The complete pipeline follows this execution flow:

1. **Voice Recognition Loop**: Continuously listens for voice commands
2. **Transcription**: Uses Whisper to convert speech to text
3. **Plan Generation**: Creates an action plan based on the command
4. **Safety Validation**: Checks each step for safety constraints
5. **Step-by-Step Execution**: Executes each action in sequence
6. **State Management**: Tracks progress through the pipeline
7. **Feedback Loop**: Uses perception and navigation feedback

## Integration Points

The pipeline integrates all four layers:

- **ROS 2 Layer**: Provides communication infrastructure
- **Simulation Layer**: Gazebo provides physics and sensor simulation
- **Isaac Layer**: Perception and navigation capabilities
- **VLA Layer**: Voice processing and LLM planning

## Testing the Pipeline

To test the complete pipeline:

1. Launch the simulation environment
2. Start the autonomous humanoid pipeline node
3. Issue voice commands to the system
4. Observe the system processing the command through all layers
5. Verify safe execution of the requested actions

## Summary

This complete pipeline example demonstrates the integration of all concepts covered in the book, creating a functional autonomous humanoid system that can receive voice commands and execute them safely in simulation.

## References

1. Complete Autonomous Systems
2. Multi-layer Robotics Integration
3. Voice Command Processing Pipelines