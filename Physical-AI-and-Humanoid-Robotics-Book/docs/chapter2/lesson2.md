---
sidebar_position: 2
---

# Lesson 2.2: High-Fidelity Rendering and Human-Robot Interaction (HRI) in Unity

This lesson covers high-fidelity rendering and Human-Robot Interaction (HRI) in Unity for humanoid robotics applications, providing a comprehensive understanding of Unity's capabilities for creating realistic simulation environments and effective HRI interfaces.

## Introduction

Unity provides high-fidelity rendering capabilities that are essential for creating realistic simulation environments and enabling effective Human-Robot Interaction (HRI) for humanoid robots. With its advanced graphics pipeline, physics engine, and asset ecosystem, Unity offers unique advantages for humanoid robotics simulation, particularly for high-fidelity rendering, Human-Robot Interaction (HRI), and photorealistic sensor simulation. This lesson explores Unity's integration with ROS 2 through the Unity Robotics Hub and Unity ML-Agents, focusing on creating immersive environments for humanoid robot development.

## Key Concepts

- Unity for high-fidelity rendering and photorealistic simulation
- Human-Robot Interaction (HRI) principles and interface design
- Physics simulation in Unity with realistic materials and lighting
- Integration with ROS 2 through Unity Robotics Hub and ROS#
- Sensor simulation including cameras, LiDAR, and IMU in Unity
- Photorealistic environment creation for training perception systems

## Unity Robotics Hub Integration

The Unity Robotics Hub provides essential tools for connecting Unity with ROS 2:

```csharp
// Example Unity script for humanoid control using ROS# integration
using UnityEngine;
using ROS2;
using System.Collections;
using System.Collections.Generic;
using JointStateMsg = sensor_msgs.JointState;
using JointTrajectoryMsg = trajectory_msgs.JointTrajectory;

public class UnityHumanoidController : MonoBehaviour
{
    [Header("ROS Connection")]
    public string rosMasterURL = "http://localhost:11311";
    public string robotNamespace = "/humanoid";

    [Header("Joint Configuration")]
    public List<Transform> jointTransforms = new List<Transform>();
    public List<string> jointNames = new List<string>();

    private ROS2UnityComponent ros2Unity;
    private ROS2Node ros2Node;
    private Publisher<JointStateMsg> jointStatePublisher;
    private Subscriber<JointStateMsg> jointCommandSubscriber;

    private JointStateMsg jointStateMsg;
    private JointStateMsg jointCommandMsg;

    void Start()
    {
        // Initialize ROS 2 connection
        ros2Unity = GetComponent<ROS2UnityComponent>();
        ros2Unity.ROS2Settings = new ROS2Settings
        {
            rosMasterURL = rosMasterURL,
            logLevel = LogLevel.Information
        };
        ros2Unity.Initialize();

        // Create ROS 2 node
        ros2Node = ros2Unity.CreateNode("unity_humanoid_controller");

        // Initialize publishers and subscribers
        InitializeROSInterfaces();

        // Initialize joint state message
        jointStateMsg = new JointStateMsg();
        jointStateMsg.name = jointNames.ToArray();
        jointStateMsg.position = new double[jointNames.Count];
        jointStateMsg.velocity = new double[jointNames.Count];
        jointStateMsg.effort = new double[jointNames.Count];

        // Start publishing joint states
        StartCoroutine(PublishJointStates());
    }

    private void InitializeROSInterfaces()
    {
        // Publisher for joint states
        jointStatePublisher = ros2Node.CreatePublisher<JointStateMsg>($"{robotNamespace}/joint_states");

        // Subscriber for joint commands
        jointCommandSubscriber = ros2Node.CreateSubscriber<JointStateMsg>($"{robotNamespace}/joint_commands",
            JointCommandCallback);
    }

    void JointCommandCallback(JointStateMsg msg)
    {
        // Process incoming joint commands
        for (int i = 0; i < msg.name.Count; i++)
        {
            string jointName = msg.name[i];
            int jointIndex = jointNames.IndexOf(jointName);

            if (jointIndex >= 0 && jointIndex < jointTransforms.Count)
            {
                // Update joint position based on command
                if (i < msg.position.Count)
                {
                    // Apply joint position with constraints
                    ApplyJointPosition(jointIndex, (float)msg.position[i]);
                }
            }
        }
    }

    void ApplyJointPosition(int jointIndex, float position)
    {
        // Apply position to the joint transform
        // This is a simplified implementation - real implementation would consider joint limits
        Transform joint = jointTransforms[jointIndex];
        joint.localEulerAngles = new Vector3(0, 0, position * Mathf.Rad2Deg);
    }

    IEnumerator PublishJointStates()
    {
        while (ros2Unity.Ok())
        {
            // Update joint state message with current positions
            for (int i = 0; i < jointTransforms.Count; i++)
            {
                jointStateMsg.position[i] = jointTransforms[i].localEulerAngles.z * Mathf.Deg2Rad;
            }

            // Set timestamp
            jointStateMsg.header.stamp = new TimeStamp(ros2Unity.GetROSTime());
            jointStateMsg.header.frame_id = "base_link";

            // Publish joint states
            jointStatePublisher.Publish(jointStateMsg);

            yield return new WaitForSeconds(0.01f); // 100Hz update rate
        }
    }

    void Update()
    {
        // Handle real-time updates if needed
    }

    void OnDestroy()
    {
        if (ros2Unity != null)
        {
            ros2Unity.Shutdown();
        }
    }
}
```

## High-Fidelity Rendering Pipeline

Unity's rendering pipeline enables photorealistic simulation crucial for humanoid robotics:

```csharp
// Unity script for high-fidelity rendering configuration
using UnityEngine;
using UnityEngine.Rendering;

public class HighFidelityRenderer : MonoBehaviour
{
    [Header("Rendering Quality Settings")]
    public bool useHDR = true;
    public bool useMSAA = true;
    public bool usePostProcessing = true;
    public bool useRealisticLighting = true;

    [Header("Material Properties")]
    public PhysicMaterial robotMaterial;
    public PhysicMaterial floorMaterial;

    [Header("Lighting Configuration")]
    public Light mainLight;
    public float ambientIntensity = 0.2f;
    public Color ambientColor = Color.gray;

    void Start()
    {
        ConfigureRenderingPipeline();
        ConfigureMaterials();
        ConfigureLighting();
    }

    void ConfigureRenderingPipeline()
    {
        // Enable HDR rendering
        if (useHDR)
        {
            GetComponent<Camera>().allowHDR = true;
        }

        // Configure MSAA
        if (useMSAA)
        {
            GetComponent<Camera>().allowMSAA = true;
        }

        // Configure post-processing if available
        if (usePostProcessing)
        {
            // Enable bloom, chromatic aberration, etc. for realistic effects
            ConfigurePostProcessing();
        }
    }

    void ConfigurePostProcessing()
    {
        // This would configure Unity's Post-Processing Stack
        // for realistic camera effects like bloom, chromatic aberration, etc.
        Debug.Log("Post-processing effects configured for photorealistic rendering");
    }

    void ConfigureMaterials()
    {
        // Configure materials for realistic physics and appearance
        if (robotMaterial != null)
        {
            robotMaterial.staticFriction = 0.8f;
            robotMaterial.dynamicFriction = 0.6f;
            robotMaterial.bounciness = 0.1f;
        }

        if (floorMaterial != null)
        {
            floorMaterial.staticFriction = 0.9f;
            floorMaterial.dynamicFriction = 0.8f;
            floorMaterial.bounciness = 0.0f;
        }
    }

    void ConfigureLighting()
    {
        // Configure realistic lighting
        RenderSettings.ambientIntensity = ambientIntensity;
        RenderSettings.ambientLight = ambientColor;

        if (mainLight != null)
        {
            mainLight.shadows = LightShadows.Soft;
            mainLight.shadowStrength = 0.8f;
            mainLight.range = 20f;
        }
    }
}
```

## Human-Robot Interaction (HRI) Interface Design

Creating effective HRI interfaces in Unity requires careful consideration of user experience and interaction patterns:

```csharp
// Unity script for HRI interface
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using TMPro;

public class HumanoidHRIInterface : MonoBehaviour
{
    [Header("HRI Interface Elements")]
    public GameObject hriCanvas;
    public Button speakButton;
    public Button gestureButton;
    public Button emergencyStopButton;
    public Slider speedSlider;
    public TextMeshProUGUI statusText;
    public TextMeshProUGUI commandText;

    [Header("Voice Command Simulation")]
    public string[] voiceCommands = {
        "Walk forward",
        "Turn left",
        "Turn right",
        "Stop",
        "Pick up object",
        "Follow me"
    };

    void Start()
    {
        InitializeHRIInterface();
    }

    void InitializeHRIInterface()
    {
        // Setup button event listeners
        if (speakButton != null)
        {
            speakButton.onClick.AddListener(OnSpeakButtonClicked);
        }

        if (gestureButton != null)
        {
            gestureButton.onClick.AddListener(OnGestureButtonClicked);
        }

        if (emergencyStopButton != null)
        {
            emergencyStopButton.onClick.AddListener(OnEmergencyStopClicked);
        }

        if (speedSlider != null)
        {
            speedSlider.onValueChanged.AddListener(OnSpeedChanged);
        }
    }

    void OnSpeakButtonClicked()
    {
        // Simulate voice command recognition
        string randomCommand = voiceCommands[Random.Range(0, voiceCommands.Length)];
        ProcessVoiceCommand(randomCommand);
    }

    void OnGestureButtonClicked()
    {
        // Process gesture-based commands
        Debug.Log("Gesture command received");
        ProcessGestureCommand();
    }

    void OnEmergencyStopClicked()
    {
        // Send emergency stop command to humanoid
        SendEmergencyStop();
        statusText.text = "EMERGENCY STOP ACTIVATED";
        statusText.color = Color.red;
    }

    void OnSpeedChanged(float value)
    {
        // Adjust humanoid speed based on slider
        Debug.Log($"Speed adjusted to: {value}");
    }

    void ProcessVoiceCommand(string command)
    {
        commandText.text = $"Command: {command}";
        statusText.text = "Processing voice command...";

        // In a real implementation, this would send the command to ROS
        SendCommandToHumanoid(command);
    }

    void ProcessGestureCommand()
    {
        // Process gesture recognition
        statusText.text = "Processing gesture command...";
    }

    void SendCommandToHumanoid(string command)
    {
        // This would interface with ROS to send commands
        Debug.Log($"Sending command to humanoid: {command}");
        statusText.text = $"Command sent: {command}";
    }

    void SendEmergencyStop()
    {
        // Send emergency stop to humanoid through ROS
        Debug.Log("Emergency stop command sent to humanoid");
    }
}
```

## Sensor Simulation in Unity

Unity enables realistic sensor simulation for humanoid robots:

```csharp
// Unity script for sensor simulation
using UnityEngine;
using System.Collections;
using ROS2;
using sensor_msgs;

public class UnitySensorSimulator : MonoBehaviour
{
    [Header("Sensor Configuration")]
    public Camera rgbCamera;
    public Camera depthCamera;
    public Transform lidarOrigin;
    public int lidarPoints = 1080;
    public float lidarRange = 10.0f;
    public float lidarFov = 360.0f;

    private ROS2Node ros2Node;
    private Publisher<Image> rgbPublisher;
    private Publisher<Image> depthPublisher;
    private Publisher<LaserScan> lidarPublisher;

    private Image rgbImageMsg;
    private Image depthImageMsg;
    private LaserScan lidarScanMsg;

    void Start()
    {
        // Initialize ROS interfaces
        InitializeSensors();

        // Start sensor simulation
        StartCoroutine(SimulateSensors());
    }

    void InitializeSensors()
    {
        ros2Node = GetComponent<ROS2UnityComponent>().CreateNode("unity_sensor_simulator");

        // Initialize publishers
        rgbPublisher = ros2Node.CreatePublisher<Image>("/humanoid/rgb_camera/image_raw");
        depthPublisher = ros2Node.CreatePublisher<Image>("/humanoid/depth_camera/image_raw");
        lidarPublisher = ros2Node.CreatePublisher<LaserScan>("/humanoid/lidar_scan");

        // Initialize messages
        rgbImageMsg = new Image();
        depthImageMsg = new Image();
        lidarScanMsg = new LaserScan();
    }

    IEnumerator SimulateSensors()
    {
        while (true)
        {
            // Simulate RGB camera
            SimulateRGBCamera();

            // Simulate depth camera
            SimulateDepthCamera();

            // Simulate LiDAR
            SimulateLiDAR();

            yield return new WaitForSeconds(0.1f); // 10Hz sensor update
        }
    }

    void SimulateRGBCamera()
    {
        // Capture RGB image and publish
        if (rgbCamera != null)
        {
            // In a real implementation, capture and encode the RGB image
            // This is a simplified placeholder
            Debug.Log("RGB camera simulation");
        }
    }

    void SimulateDepthCamera()
    {
        // Capture depth image and publish
        if (depthCamera != null)
        {
            // In a real implementation, capture and encode the depth image
            Debug.Log("Depth camera simulation");
        }
    }

    void SimulateLiDAR()
    {
        // Simulate LiDAR scan
        if (lidarOrigin != null)
        {
            lidarScanMsg.ranges = new float[lidarPoints];
            lidarScanMsg.intensities = new float[lidarPoints];

            for (int i = 0; i < lidarPoints; i++)
            {
                float angle = (i * lidarFov / lidarPoints) * Mathf.Deg2Rad;

                // Perform raycast to simulate LiDAR measurement
                Vector3 direction = new Vector3(Mathf.Cos(angle), 0, Mathf.Sin(angle));
                RaycastHit hit;

                if (Physics.Raycast(lidarOrigin.position, direction, out hit, lidarRange))
                {
                    lidarScanMsg.ranges[i] = hit.distance;
                    lidarScanMsg.intensities[i] = 1.0f; // Reflectance
                }
                else
                {
                    lidarScanMsg.ranges[i] = lidarRange;
                    lidarScanMsg.intensities[i] = 0.0f;
                }
            }

            // Set scan parameters
            lidarScanMsg.angle_min = -lidarFov * Mathf.Deg2Rad / 2;
            lidarScanMsg.angle_max = lidarFov * Mathf.Deg2Rad / 2;
            lidarScanMsg.angle_increment = (lidarFov * Mathf.Deg2Rad) / lidarPoints;
            lidarScanMsg.time_increment = 0;
            lidarScanMsg.scan_time = 0.1f; // 10Hz
            lidarScanMsg.range_min = 0.1f;
            lidarScanMsg.range_max = lidarRange;

            // Set timestamp
            lidarScanMsg.header.stamp = new TimeStamp(Time.time);
            lidarScanMsg.header.frame_id = "lidar_link";

            // Publish LiDAR scan
            lidarPublisher.Publish(lidarScanMsg);
        }
    }
}
```

## Unity ML-Agents for Humanoid Training

Unity ML-Agents enables reinforcement learning training for humanoid robots:

```csharp
// Unity ML-Agents script for humanoid training
using Unity.MLAgents;
using Unity.MLAgents.Sensors;
using Unity.MLAgents.Actuators;
using UnityEngine;

public class HumanoidAgent : Agent
{
    [Header("Humanoid Configuration")]
    public Transform target;
    public float moveSpeed = 3.0f;
    public float rotationSpeed = 100.0f;

    [Header("Reward Settings")]
    public float reachTargetReward = 5.0f;
    public float stepReward = -0.01f;
    public float fallPenalty = -1.0f;

    private Rigidbody rb;
    private Vector3 initialPosition;

    public override void Initialize()
    {
        rb = GetComponent<Rigidbody>();
        initialPosition = transform.position;
    }

    public override void OnEpisodeBegin()
    {
        // Reset humanoid position
        transform.position = initialPosition;
        rb.velocity = Vector3.zero;
        rb.angularVelocity = Vector3.zero;
    }

    public override void CollectObservations(VectorSensor sensor)
    {
        // Add observations for the ML agent
        sensor.AddObservation(transform.position);
        sensor.AddObservation(target.position);
        sensor.AddObservation(rb.velocity);
        sensor.AddObservation(transform.rotation);
    }

    public override void OnActionReceived(ActionBuffers actions)
    {
        // Process actions from the ML agent
        float moveX = actions.ContinuousActions[0];
        float moveZ = actions.ContinuousActions[1];
        float rotate = actions.ContinuousActions[2];

        // Apply movement
        Vector3 moveDirection = new Vector3(moveX, 0, moveZ).normalized;
        rb.AddForce(moveDirection * moveSpeed, ForceMode.VelocityChange);

        // Apply rotation
        transform.Rotate(Vector3.up, rotate * rotationSpeed * Time.deltaTime);

        // Calculate reward
        float distanceToTarget = Vector3.Distance(transform.position, target.position);
        SetReward(stepReward - distanceToTarget * 0.01f);

        // Check if target reached
        if (distanceToTarget < 2.0f)
        {
            SetReward(GetReward() + reachTargetReward);
            EndEpisode();
        }

        // Check if humanoid fell
        if (transform.position.y < initialPosition.y - 1.0f)
        {
            SetReward(GetReward() + fallPenalty);
            EndEpisode();
        }
    }

    public override void Heuristic(in ActionBuffers actionsOut)
    {
        // For manual control during testing
        var continuousActionsOut = actionsOut.ContinuousActions;
        continuousActionsOut[0] = Input.GetAxis("Horizontal");
        continuousActionsOut[1] = Input.GetAxis("Vertical");
        continuousActionsOut[2] = Input.GetAxis("Rotate");
    }
}
```

## HRI Considerations for Humanoid Robots

Effective Human-Robot Interaction in Unity simulations should consider several important factors:

### Visual Feedback Mechanisms
- Status indicators for robot state (idle, moving, processing command)
- Visual cues for robot attention (eye contact simulation)
- Feedback for successful command execution
- Warning indicators for errors or safety concerns

### Intuitive Interaction Interfaces
- Natural gesture recognition
- Voice command simulation
- Visual programming interfaces
- Context-aware command interpretation

### Realistic Rendering for Perception
- Photorealistic textures and materials
- Accurate lighting simulation
- Realistic shadow casting
- Proper camera parameters matching real sensors

## Integration with Perception Systems

Unity enables photorealistic sensor data generation for training perception systems:

```csharp
// Script for generating synthetic training data
using UnityEngine;
using System.Collections;
using System.IO;

public class SyntheticDataGenerator : MonoBehaviour
{
    [Header("Data Generation Settings")]
    public Camera rgbCamera;
    public Camera depthCamera;
    public int numScenes = 1000;
    public string outputDirectory = "SyntheticData";

    [Header("Environment Variations")]
    public Light[] lights;
    public Material[] materials;
    public GameObject[] objects;

    void Start()
    {
        StartCoroutine(GenerateSyntheticData());
    }

    IEnumerator GenerateSyntheticData()
    {
        for (int i = 0; i < numScenes; i++)
        {
            // Randomize environment
            RandomizeEnvironment();

            // Capture RGB and depth images
            CaptureRGBImage(i);
            CaptureDepthImage(i);

            // Wait for next frame
            yield return new WaitForEndOfFrame();

            // Wait a bit between captures
            yield return new WaitForSeconds(0.1f);
        }
    }

    void RandomizeEnvironment()
    {
        // Randomize lighting conditions
        foreach (Light light in lights)
        {
            light.intensity = Random.Range(0.5f, 1.5f);
            light.color = Random.ColorHSV(0.0f, 1.0f, 0.8f, 1.0f, 0.8f, 1.0f);
        }

        // Randomize object positions
        foreach (GameObject obj in objects)
        {
            obj.transform.position = new Vector3(
                Random.Range(-5f, 5f),
                Random.Range(0f, 2f),
                Random.Range(-5f, 5f)
            );
        }
    }

    void CaptureRGBImage(int index)
    {
        // Capture and save RGB image
        RenderTexture currentRT = RenderTexture.active;
        RenderTexture.active = rgbCamera.targetTexture;
        rgbCamera.Render();

        Texture2D image = new Texture2D(rgbCamera.targetTexture.width, rgbCamera.targetTexture.height);
        image.ReadPixels(new Rect(0, 0, rgbCamera.targetTexture.width, rgbCamera.targetTexture.height), 0, 0);
        image.Apply();

        byte[] bytes = image.EncodeToPNG();
        File.WriteAllBytes(Path.Combine(outputDirectory, $"rgb_{index:0000}.png"), bytes);

        RenderTexture.active = currentRT;
        DestroyImmediate(image);
    }

    void CaptureDepthImage(int index)
    {
        // Capture and save depth image
        RenderTexture currentRT = RenderTexture.active;
        RenderTexture.active = depthCamera.targetTexture;
        depthCamera.Render();

        Texture2D image = new Texture2D(depthCamera.targetTexture.width, depthCamera.targetTexture.height);
        image.ReadPixels(new Rect(0, 0, depthCamera.targetTexture.width, depthCamera.targetTexture.height), 0, 0);
        image.Apply();

        byte[] bytes = image.EncodeToPNG();
        File.WriteAllBytes(Path.Combine(outputDirectory, $"depth_{index:0000}.png"), bytes);

        RenderTexture.active = currentRT;
        DestroyImmediate(image);
    }
}
```

## Summary

This lesson provided a comprehensive overview of high-fidelity rendering and Human-Robot Interaction (HRI) in Unity for humanoid robotics applications. We explored Unity Robotics Hub integration, high-fidelity rendering pipelines, HRI interface design, sensor simulation, and synthetic data generation for training perception systems. The next lesson will build on these concepts to explore simulated sensor data integration and comparison techniques.

## References

1. Unity Technologies. (2021). Unity Robotics Hub: Connecting Unity and ROS. *Unity Technologies*. https://github.com/Unity-Technologies/Unity-Robotics-Hub
2. Juliani, A., Berges, V. P., Teng, E., Cohen, A., Harper, J., Elion, C., Goy, C., Gao, Y., Henry, H., Mattar, M., & Lange, D. (2020). Unity: A general platform for intelligent agents. *arXiv preprint arXiv:1809.02600*. https://arxiv.org/abs/1809.02600
3. Hwangbo, J., Lee, J., Dosovitskiy, A., Mordatch, I., & Koltun, V. (2019). Learning agile and dynamic motor skills for legged robots. *Science Robotics*, 4(26), eaau5872. https://doi.org/10.1126/scirobotics.aau5872