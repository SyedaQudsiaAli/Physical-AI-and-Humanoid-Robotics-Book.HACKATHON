---
sidebar_position: 2
---

# Capstone Build Guide: The Autonomous Humanoid

This step-by-step guide will help you build and integrate the complete autonomous humanoid system that combines all concepts from the previous chapters.

## Prerequisites

Before starting the capstone project, ensure you have:

- ROS 2 installed (Humble Hawksbill or later recommended)
- Gazebo Garden or compatible simulation environment
- NVIDIA Isaac Sim (if using Isaac components)
- Python 3.8+ with required packages
- Whisper model for voice processing
- Access to an LLM API or local model

## System Architecture

The complete autonomous humanoid system consists of four integrated layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE                           │
├─────────────────────────────────────────────────────────────┤
│  Voice Command → Whisper → LLM → Action Plan                │
├─────────────────────────────────────────────────────────────┤
│                    PLANNING LAYER                           │
│  Task Decomposition, Path Planning, Action Sequencing       │
├─────────────────────────────────────────────────────────────┤
│                   CONTROL LAYER                             │
│  ROS 2 Nodes, Actions, Services, Topics                     │
├─────────────────────────────────────────────────────────────┤
│                   PERCEPTION LAYER                          │
│  VSLAM, Object Detection, Sensor Fusion                     │
├─────────────────────────────────────────────────────────────┤
│                  SIMULATION LAYER                           │
│  Gazebo/Unity, Physics, Humanoid Model                      │
└─────────────────────────────────────────────────────────────┘
```

## Step 1: Environment Setup

### 1.1 Install ROS 2 and Dependencies

```bash
# Install ROS 2 Humble (Ubuntu/Debian)
sudo apt update
sudo apt install ros-humble-desktop
sudo apt install python3-rosdep python3-rosinstall python3-rosinstall-generator python3-wstool build-essential

# Source ROS 2 environment
source /opt/ros/humble/setup.bash
```

### 1.2 Set up Workspace

```bash
# Create workspace
mkdir -p ~/humanoid_ws/src
cd ~/humanoid_ws

# Build workspace
colcon build
source install/setup.bash
```

## Step 2: Implement Voice Command Processing

Create the voice command processing node:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
import speech_recognition as sr
import whisper
import threading

class VoiceCommandNode(Node):
    def __init__(self):
        super().__init__('voice_command_node')
        self.publisher_ = self.create_publisher(String, 'transcribed_command', 10)

        # Load Whisper model
        self.model = whisper.load_model("base")

        # Start voice recognition in a separate thread
        self.recognition_thread = threading.Thread(target=self.listen_for_commands)
        self.recognition_thread.daemon = True
        self.recognition_thread.start()

    def listen_for_commands(self):
        recognizer = sr.Recognizer()
        microphone = sr.Microphone()

        with microphone as source:
            recognizer.adjust_for_ambient_noise(source)

        while rclpy.ok():
            try:
                with microphone as source:
                    audio = recognizer.listen(source, timeout=5.0)

                # Process with Whisper
                result = self.model.transcribe(audio.get_wav_data())
                command = result["text"]

                # Publish the transcribed command
                msg = String()
                msg.data = command
                self.publisher_.publish(msg)
                self.get_logger().info(f'Transcribed: {command}')

            except sr.WaitTimeoutError:
                pass  # Continue listening
            except Exception as e:
                self.get_logger().error(f'Error in voice recognition: {e}')
```

## Step 3: Implement LLM Planning Node

Create the LLM planning node:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
import openai  # or your preferred LLM API

class LLMPlannerNode(Node):
    def __init__(self):
        super().__init__('llm_planner_node')

        # Subscribe to transcribed commands
        self.command_sub = self.create_subscription(
            String, 'transcribed_command', self.command_callback, 10)

        # Publish action plans
        self.plan_pub = self.create_publisher(String, 'action_plan', 10)

    def command_callback(self, msg):
        command = msg.data
        self.get_logger().info(f'Processing command: {command}')

        # Generate action plan
        plan = self.generate_action_plan(command)

        # Publish the plan
        plan_msg = String()
        plan_msg.data = plan
        self.plan_pub.publish(plan_msg)
        self.get_logger().info(f'Published plan: {plan}')

    def generate_action_plan(self, command):
        # Define the prompt for the LLM
        prompt = f"""
        You are a planning system for a humanoid robot. Convert the following command into a sequence of executable actions.

        Command: "{command}"

        Output a numbered sequence of actions that the humanoid robot should perform. Each action should be specific and executable.

        Example output format:
        1. Navigate to the kitchen area (x: 1.5, y: 2.0)
        2. Detect the red cup using object recognition
        3. Plan approach trajectory to the cup
        4. Execute grasping motion with right hand
        5. Lift the cup 10cm above the table
        6. Navigate to the counter
        7. Place the cup on the counter

        Be specific about locations, objects, and actions.
        """

        # Call the LLM (example with OpenAI)
        try:
            # response = openai.Completion.create(
            #     engine="text-davinci-003",
            #     prompt=prompt,
            #     max_tokens=300
            # )
            # return response.choices[0].text.strip()

            # For demonstration, return a mock plan
            return f"1. Mock action for: {command}"
        except Exception as e:
            self.get_logger().error(f'LLM call failed: {e}')
            return "Failed to generate plan"
```

## Step 4: Implement Safety Validator

Create the safety validation node:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
from std_srvs.srv import Trigger

class SafetyValidatorNode(Node):
    def __init__(self):
        super().__init__('safety_validator_node')

        # Subscribe to action plans
        self.plan_sub = self.create_subscription(
            String, 'action_plan', self.plan_callback, 10)

        # Publish validated plans
        self.validated_pub = self.create_publisher(
            String, 'validated_action_plan', 10)

        # Emergency stop service
        self.emergency_srv = self.create_service(
            Trigger, 'emergency_stop', self.emergency_stop_callback)

    def plan_callback(self, msg):
        plan = msg.data
        self.get_logger().info(f'Validating plan: {plan}')

        # Validate the plan
        is_safe = self.validate_plan(plan)

        if is_safe:
            self.get_logger().info('Plan passed safety validation')
            validated_msg = String()
            validated_msg.data = plan
            self.validated_pub.publish(validated_msg)
        else:
            self.get_logger().error('Plan failed safety validation')
            # Trigger emergency stop if needed
            # self.trigger_emergency_stop()

    def validate_plan(self, plan):
        # Implement safety checks
        # - Check joint limits
        # - Check collision avoidance
        # - Check stability constraints
        # - Check workspace bounds

        # For now, return True (implement actual checks in production)
        return True

    def emergency_stop_callback(self, request, response):
        self.get_logger().warn('EMERGENCY STOP ACTIVATED')
        # Implement emergency stop logic
        response.success = True
        response.message = 'Emergency stop executed'
        return response
```

## Step 5: Integration Launch File

Create a launch file to start the complete system:

```xml
<!-- humanoid_system.launch.py -->
from launch import LaunchDescription
from launch_ros.actions import Node
from launch.actions import DeclareLaunchArgument
from launch.substitutions import LaunchConfiguration

def generate_launch_description():
    return LaunchDescription([
        # Voice command node
        Node(
            package='humanoid_system',
            executable='voice_command_node',
            name='voice_command_node',
            output='screen'
        ),

        # LLM planner node
        Node(
            package='humanoid_system',
            executable='llm_planner_node',
            name='llm_planner_node',
            output='screen'
        ),

        # Safety validator node
        Node(
            package='humanoid_system',
            executable='safety_validator_node',
            name='safety_validator_node',
            output='screen'
        ),

        # Navigation node (Nav2 integration)
        Node(
            package='humanoid_system',
            executable='navigation_node',
            name='navigation_node',
            output='screen'
        ),

        # Perception node (Isaac ROS integration)
        Node(
            package='humanoid_system',
            executable='perception_node',
            name='perception_node',
            output='screen'
        )
    ])
```

## Step 6: Testing the Complete System

### 6.1 Start the Simulation

```bash
# Start Gazebo simulation with humanoid model
ros2 launch gazebo_ros empty_world.launch.py
# Or load your specific humanoid world
```

### 6.2 Launch the System

```bash
# Source your workspace
cd ~/humanoid_ws
source install/setup.bash

# Launch the complete system
ros2 launch humanoid_system humanoid_system.launch.py
```

### 6.3 Test Voice Commands

Speak voice commands to your microphone. The system should:
1. Capture and transcribe your voice command
2. Generate an action plan using the LLM
3. Validate the plan for safety
4. Execute the plan in simulation

## Troubleshooting

### Common Issues:

1. **Microphone not detected**: Check audio device permissions and configuration
2. **LLM API errors**: Verify API keys and connectivity
3. **Navigation failures**: Check map and localization
4. **Safety validation blocks execution**: Review safety constraints

### Debugging Tips:

- Monitor ROS 2 topics: `ros2 topic list` and `ros2 topic echo`
- Check node status: `ros2 node list`
- View logs: `ros2 launch` with `--log-level debug`

## Summary

This build guide provides the complete implementation for the autonomous humanoid system. The system integrates voice command processing, LLM-driven planning, safety validation, navigation, and perception to create a fully autonomous humanoid robot capable of understanding and executing natural language commands in simulation.

## References

1. ROS 2 Documentation
2. Voice Processing in Robotics
3. LLM Integration Patterns
4. Safety in Autonomous Systems