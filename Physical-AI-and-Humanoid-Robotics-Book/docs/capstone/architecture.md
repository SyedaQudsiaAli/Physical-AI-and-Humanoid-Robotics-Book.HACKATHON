```mermaid
graph TB
    subgraph "User Interface"
        A[Voice Command]
    end

    subgraph "VLA Cognition Layer"
        B[Whisper - Voice to Text]
        C[LLM - Task Decomposition]
        D[Safety Validator]
    end

    subgraph "Planning Layer"
        E[Action Planner]
        F[Path Planner - Nav2]
    end

    subgraph "Control Layer"
        G[ROS 2 Nodes]
        H[Action Servers]
        I[Service Servers]
    end

    subgraph "Perception Layer"
        J[VSLAM - Isaac ROS]
        K[Object Detection - Isaac ROS]
        L[Sensor Fusion]
    end

    subgraph "Simulation Layer"
        M[Gazebo Physics]
        N[Humanoid Model]
        O[Sensor Simulation]
    end

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    E --> G
    F --> G
    G --> H
    G --> I
    G --> J
    G --> K
    J --> L
    K --> L
    H --> M
    I --> M
    J --> M
    K --> M
    L --> M
    M --> N
    M --> O
    N --> G
    O --> J
    O --> K
    O --> L

    style A fill:#e1f5fe
    style N fill:#f3e5f5
    style M fill:#e8f5e8
    style J fill:#fff3e0
    style C fill:#fce4ec
    style D fill:#ffebee
```

## Autonomous Humanoid System Architecture

This diagram illustrates the complete architecture of the autonomous humanoid system, showing the integration of all four layers of humanoid intelligence:

1. **VLA Cognition Layer**: Processes voice commands using Whisper and LLMs
2. **Planning Layer**: Decomposes tasks and plans navigation
3. **Control Layer**: ROS 2 middleware for robot control
4. **Perception Layer**: Isaac ROS for sensing and understanding
5. **Simulation Layer**: Gazebo for physics simulation

The system demonstrates the complete workflow from voice command to robotic action execution in simulation.