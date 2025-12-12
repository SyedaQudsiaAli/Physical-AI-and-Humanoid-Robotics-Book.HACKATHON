---
sidebar_position: 1
---

# Lesson 3.1: Synthetic Data Generation for Training with Isaac Sim

This lesson covers the comprehensive use of Isaac Sim for generating synthetic data to train perception systems for humanoid robots, providing detailed insights into photorealistic simulation and domain randomization techniques.

## Introduction

Isaac Sim is NVIDIA's advanced robotics simulator that enables the generation of high-quality synthetic data for training AI models. This lesson explores how Isaac Sim can accelerate the development of perception systems for humanoid robots through photorealistic simulation, domain randomization, and automated data collection pipelines. Isaac Sim combines the power of NVIDIA Omniverse with robotics-specific capabilities, enabling the creation of diverse, realistic training datasets that can bridge the sim-to-real gap.

## Key Concepts

- Isaac Sim architecture and photorealistic rendering capabilities
- Synthetic sensor data generation for LiDAR, cameras, and IMU
- Domain randomization techniques for robust perception training
- Automated data collection and annotation pipelines
- PhysX physics simulation for realistic interactions
- USD (Universal Scene Description) for scene composition

## Isaac Sim Architecture and Setup

Isaac Sim provides a comprehensive simulation environment built on NVIDIA Omniverse:

```python
# Isaac Sim setup and basic configuration
import omni
import carb
from omni.isaac.core import World
from omni.isaac.core.utils.stage import add_reference_to_stage
from omni.isaac.core.utils.nucleus import get_assets_root_path
from omni.isaac.core.utils.prims import get_prim_at_path
from omni.isaac.synthetic_utils import SyntheticData
import numpy as np

class IsaacSimDataGenerator:
    """
    Class for generating synthetic data using Isaac Sim
    """
    def __init__(self):
        self.world = World(stage_units_in_meters=1.0)
        self.assets_root_path = get_assets_root_path()
        self.synthetic_data = SyntheticData()

        # Initialize camera and sensor configurations
        self.setup_sensors()

        # Initialize domain randomization parameters
        self.setup_domain_randomization()

    def setup_sensors(self):
        """
        Configure sensors for data collection
        """
        # RGB camera configuration
        self.rgb_camera = self.world.scene.add(
            prim_path="/World/rgb_camera",
            name="rgb_camera",
            translation=np.array([0.0, 0.0, 1.0]),
            orientation=np.array([0.0, 0.0, 0.0, 1.0])
        )

        # Depth camera configuration
        self.depth_camera = self.world.scene.add(
            prim_path="/World/depth_camera",
            name="depth_camera",
            translation=np.array([0.0, 0.0, 1.0]),
            orientation=np.array([0.0, 0.0, 0.0, 1.0])
        )

        # LiDAR sensor configuration
        self.lidar = self.world.scene.add(
            prim_path="/World/lidar",
            name="lidar",
            translation=np.array([0.0, 0.0, 1.5]),
            orientation=np.array([0.0, 0.0, 0.0, 1.0])
        )

    def setup_domain_randomization(self):
        """
        Configure domain randomization parameters
        """
        self.domain_params = {
            'lighting': {
                'intensity_range': (0.5, 2.0),
                'color_temperature_range': (3000, 8000),
                'directional_light_range': (0.0, 360.0)
            },
            'materials': {
                'roughness_range': (0.0, 1.0),
                'metallic_range': (0.0, 1.0),
                'albedo_range': (0.0, 1.0)
            },
            'objects': {
                'position_jitter': 0.1,
                'rotation_jitter': 15.0,
                'scale_range': (0.8, 1.2)
            },
            'backgrounds': {
                'texture_variety': 20,
                'environment_types': ['indoor', 'outdoor', 'warehouse']
            }
        }

    def generate_scene(self, scene_type="random"):
        """
        Generate a randomized scene for data collection
        """
        # Reset the stage
        self.world.reset()

        # Randomize lighting
        self.randomize_lighting()

        # Randomize materials
        self.randomize_materials()

        # Place objects with randomization
        self.place_random_objects()

        # Add humanoid robot to the scene
        self.add_humanoid_robot()

        # Step the physics engine
        self.world.step(render=True)

    def randomize_lighting(self):
        """
        Apply lighting randomization
        """
        # Get all lights in the scene
        lights = self.world.scene.get_lights()

        for light in lights:
            # Randomize intensity
            intensity = np.random.uniform(
                self.domain_params['lighting']['intensity_range'][0],
                self.domain_params['lighting']['intensity_range'][1]
            )
            light.intensity = intensity

            # Randomize color temperature
            color_temp = np.random.uniform(
                self.domain_params['lighting']['color_temperature_range'][0],
                self.domain_params['lighting']['color_temperature_range'][1]
            )
            light.color = self.color_temperature_to_rgb(color_temp)

    def color_temperature_to_rgb(self, color_temp):
        """
        Convert color temperature to RGB values
        """
        temp = color_temp / 100
        if temp <= 66:
            red = 255
            green = temp
            green = 99.4708025861 * np.log(green) - 161.1195681661
        else:
            red = temp - 60
            red = 329.698727446 * (red ** -0.1332047592)
            green = temp - 60
            green = 288.1221695283 * (green ** -0.0755148492)

        blue = temp - 10
        blue = 138.5177312231 * np.log(blue) - 305.0447927307

        # Clamp values to [0, 255]
        red = np.clip(red, 0, 255) / 255.0
        green = np.clip(green, 0, 255) / 255.0
        blue = np.clip(blue, 0, 255) / 255.0

        return np.array([red, green, blue, 1.0])

    def randomize_materials(self):
        """
        Apply material randomization to objects
        """
        # Get all objects in the scene
        objects = self.world.scene.get_objects()

        for obj in objects:
            # Randomize material properties
            roughness = np.random.uniform(
                self.domain_params['materials']['roughness_range'][0],
                self.domain_params['materials']['roughness_range'][1]
            )
            metallic = np.random.uniform(
                self.domain_params['materials']['metallic_range'][0],
                self.domain_params['materials']['metallic_range'][1]
            )
            albedo = np.random.uniform(
                self.domain_params['materials']['albedo_range'][0],
                self.domain_params['materials']['albedo_range'][1],
                size=3
            )

            # Apply material properties (simplified)
            obj.roughness = roughness
            obj.metallic = metallic
            obj.albedo = albedo

    def place_random_objects(self):
        """
        Place objects with randomized positions, rotations, and scales
        """
        # Define object types and their placement ranges
        object_types = ['cube', 'sphere', 'cylinder', 'capsule']
        num_objects = np.random.randint(5, 15)  # Random number of objects

        for i in range(num_objects):
            obj_type = np.random.choice(object_types)

            # Random position
            pos_x = np.random.uniform(-5.0, 5.0)
            pos_y = np.random.uniform(-5.0, 5.0)
            pos_z = np.random.uniform(0.1, 2.0)  # Above ground

            # Random rotation
            rot_x = np.random.uniform(-15, 15)
            rot_y = np.random.uniform(0, 360)
            rot_z = np.random.uniform(-15, 15)

            # Random scale
            scale = np.random.uniform(
                self.domain_params['objects']['scale_range'][0],
                self.domain_params['objects']['scale_range'][1]
            )

            # Create object
            self.create_object(obj_type, [pos_x, pos_y, pos_z], [rot_x, rot_y, rot_z], scale)

    def create_object(self, obj_type, position, rotation, scale):
        """
        Create an object of specified type
        """
        # This is a simplified representation
        # In actual Isaac Sim, you would use specific API calls
        print(f"Creating {obj_type} at {position}, scale: {scale}")

    def add_humanoid_robot(self):
        """
        Add a humanoid robot to the scene
        """
        # Load humanoid robot model
        if self.assets_root_path:
            humanoid_asset_path = self.assets_root_path + "/Isaac/Robots/Humanoid/humanoid.usd"
            add_reference_to_stage(usd_path=humanoid_asset_path, prim_path="/World/Humanoid")

            # Set initial position and orientation
            humanoid = self.world.scene.get_object("Humanoid")
            if humanoid:
                humanoid.set_world_pose(position=np.array([0.0, 0.0, 0.5]))

    def capture_synthetic_data(self, output_dir="synthetic_data"):
        """
        Capture synthetic data from the current scene
        """
        import os
        import cv2
        from PIL import Image

        # Create output directory
        os.makedirs(output_dir, exist_ok=True)

        # Capture RGB image
        rgb_data = self.rgb_camera.get_rgb()
        rgb_image = Image.fromarray(rgb_data)
        rgb_image.save(f"{output_dir}/rgb_{carb.tokens.get_time()}.png")

        # Capture depth image
        depth_data = self.depth_camera.get_depth()
        depth_image = Image.fromarray((depth_data * 255).astype(np.uint8))
        depth_image.save(f"{output_dir}/depth_{carb.tokens.get_time()}.png")

        # Capture segmentation mask
        seg_data = self.rgb_camera.get_segmentation()
        seg_image = Image.fromarray((seg_data * 255).astype(np.uint8))
        seg_image.save(f"{output_dir}/segmentation_{carb.tokens.get_time()}.png")

        # Capture LiDAR data
        lidar_data = self.lidar.get_lidar()
        np.save(f"{output_dir}/lidar_{carb.tokens.get_time()}.npy", lidar_data)

        print(f"Synthetic data captured and saved to {output_dir}")

    def run_data_generation(self, num_scenes=1000):
        """
        Run the synthetic data generation process
        """
        for i in range(num_scenes):
            print(f"Generating scene {i+1}/{num_scenes}")

            # Generate a randomized scene
            self.generate_scene()

            # Capture synthetic data
            self.capture_synthetic_data(f"synthetic_data/scene_{i:04d}")

            # Step the world to ensure physics update
            self.world.step(render=True)

        print("Synthetic data generation completed!")

def main():
    """
    Main function to run synthetic data generation
    """
    # Initialize Isaac Sim
    sim_app = omni.kit.app.get_app_interface()

    # Create data generator
    generator = IsaacSimDataGenerator()

    # Run data generation
    generator.run_data_generation(num_scenes=100)  # For demo, use fewer scenes

    # Cleanup
    generator.world.clear()
    sim_app.close()

if __name__ == "__main__":
    main()
```

## Advanced Domain Randomization Techniques

Domain randomization is crucial for creating robust perception systems:

```python
import numpy as np
import random
from dataclasses import dataclass
from typing import Dict, List, Tuple

@dataclass
class DomainRandomizationConfig:
    """
    Configuration for domain randomization parameters
    """
    # Lighting parameters
    lighting_intensity_range: Tuple[float, float] = (0.3, 2.0)
    lighting_color_temperature_range: Tuple[float, float] = (3000, 8000)
    lighting_direction_range: Tuple[float, float] = (0, 360)

    # Material parameters
    material_roughness_range: Tuple[float, float] = (0.0, 1.0)
    material_metallic_range: Tuple[float, float] = (0.0, 1.0)
    material_albedo_range: Tuple[float, float] = (0.0, 1.0)

    # Object placement parameters
    object_position_jitter: float = 0.1
    object_rotation_jitter: float = 15.0
    object_scale_range: Tuple[float, float] = (0.8, 1.2)

    # Camera parameters
    camera_position_jitter: float = 0.05
    camera_rotation_jitter: float = 2.0

    # Environmental parameters
    background_texture_count: int = 20
    environment_types: List[str] = None

    def __post_init__(self):
        if self.environment_types is None:
            self.environment_types = ["indoor", "outdoor", "warehouse", "office", "kitchen"]

class AdvancedDomainRandomizer:
    """
    Advanced domain randomization for synthetic data generation
    """
    def __init__(self, config: DomainRandomizationConfig):
        self.config = config
        self.random_state = np.random.RandomState()

    def randomize_lighting(self, scene):
        """
        Apply advanced lighting randomization
        """
        # Randomize directional light
        intensity = self.random_state.uniform(
            self.config.lighting_intensity_range[0],
            self.config.lighting_intensity_range[1]
        )

        color_temp = self.random_state.uniform(
            self.config.lighting_color_temperature_range[0],
            self.config.lighting_color_temperature_range[1]
        )

        direction = self.random_state.uniform(
            self.config.lighting_direction_range[0],
            self.config.lighting_direction_range[1],
            size=3
        )
        direction = direction / np.linalg.norm(direction)  # Normalize

        # Apply to scene (simplified)
        print(f"Applied lighting: intensity={intensity:.2f}, color_temp={color_temp:.0f}, direction={direction}")

        # Add multiple light sources for complex lighting
        self.add_multiple_lights(scene)

    def add_multiple_lights(self, scene):
        """
        Add multiple light sources for realistic lighting
        """
        num_lights = self.random_state.randint(2, 5)

        for i in range(num_lights):
            light_type = random.choice(["directional", "point", "spot"])
            intensity = self.random_state.uniform(0.1, 1.0)

            # Random position for point/spot lights
            if light_type in ["point", "spot"]:
                position = self.random_state.uniform(-10, 10, size=3)
                position[2] = max(1.0, position[2])  # Ensure above ground

            print(f"Added {light_type} light with intensity {intensity:.2f}")

    def randomize_materials(self, scene_objects):
        """
        Apply advanced material randomization
        """
        for obj in scene_objects:
            # Randomize multiple material properties
            roughness = self.random_state.uniform(
                self.config.material_roughness_range[0],
                self.config.material_roughness_range[1]
            )

            metallic = self.random_state.uniform(
                self.config.material_metallic_range[0],
                self.config.material_metallic_range[1]
            )

            albedo = self.random_state.uniform(
                self.config.material_albedo_range[0],
                self.config.material_albedo_range[1],
                size=3
            )

            # Add texture variations
            texture_variation = self.random_state.uniform(0.0, 0.3)

            # Apply to object (simplified)
            print(f"Applied material properties to {obj}: "
                  f"roughness={roughness:.2f}, metallic={metallic:.2f}, "
                  f"albedo={albedo}, texture_var={texture_variation:.2f}")

    def randomize_object_placement(self, objects):
        """
        Apply advanced object placement randomization
        """
        for obj in objects:
            # Apply position jitter
            pos_jitter = self.random_state.uniform(
                -self.config.object_position_jitter,
                self.config.object_position_jitter,
                size=3
            )

            # Apply rotation jitter
            rot_jitter = self.random_state.uniform(
                -self.config.object_rotation_jitter,
                self.config.object_rotation_jitter,
                size=3
            )

            # Apply scale variation
            scale_factor = self.random_state.uniform(
                self.config.object_scale_range[0],
                self.config.object_scale_range[1]
            )

            # Apply transformations (simplified)
            print(f"Applied placement randomization to {obj}: "
                  f"pos_jitter={pos_jitter}, rot_jitter={rot_jitter}, scale={scale_factor:.2f}")

    def randomize_environment(self):
        """
        Randomize the environment
        """
        env_type = random.choice(self.config.environment_types)

        # Load random background texture
        bg_texture_idx = self.random_state.randint(0, self.config.background_texture_count)

        # Apply environmental effects
        fog_density = self.random_state.uniform(0.0, 0.1)
        atmospheric_haze = self.random_state.uniform(0.0, 0.05)

        print(f"Environment randomized: type={env_type}, "
              f"bg_texture={bg_texture_idx}, fog={fog_density:.3f}, "
              f"haze={atmospheric_haze:.3f}")

    def randomize_camera(self, camera):
        """
        Apply camera randomization
        """
        # Apply position jitter
        pos_jitter = self.random_state.uniform(
            -self.config.camera_position_jitter,
            self.config.camera_position_jitter,
            size=3
        )

        # Apply rotation jitter
        rot_jitter = self.random_state.uniform(
            -self.config.camera_rotation_jitter,
            self.config.camera_rotation_jitter,
            size=3
        )

        # Randomize camera parameters
        focal_length = self.random_state.uniform(18, 50)  # mm
        f_stop = self.random_state.uniform(1.4, 16.0)

        print(f"Camera randomized: pos_jitter={pos_jitter}, "
              f"rot_jitter={rot_jitter}, focal={focal_length:.1f}mm, f_stop={f_stop:.1f}")

# Example usage of advanced domain randomization
def example_domain_randomization():
    """
    Example of using advanced domain randomization
    """
    config = DomainRandomizationConfig()
    randomizer = AdvancedDomainRandomizer(config)

    # Simulate scene objects
    scene_objects = [f"object_{i}" for i in range(10)]

    # Apply randomizations
    randomizer.randomize_lighting("scene")
    randomizer.randomize_materials(scene_objects)
    randomizer.randomize_object_placement(scene_objects)
    randomizer.randomize_environment()
    randomizer.randomize_camera("rgb_camera")

    print("Advanced domain randomization applied successfully!")

if __name__ == "__main__":
    example_domain_randomization()
```

## Synthetic Data Annotation Pipeline

Creating properly annotated synthetic data is essential for training perception models:

```python
import json
import os
from typing import Dict, List, Any
from dataclasses import dataclass
import numpy as np

@dataclass
class BoundingBox:
    """
    Bounding box annotation for object detection
    """
    x_min: float
    y_min: float
    x_max: float
    y_max: float
    class_id: int
    class_name: str
    confidence: float = 1.0

@dataclass
class InstanceSegmentation:
    """
    Instance segmentation annotation
    """
    mask: np.ndarray  # Binary mask
    class_id: int
    class_name: str
    object_id: int

class SyntheticAnnotationPipeline:
    """
    Pipeline for generating annotations for synthetic data
    """
    def __init__(self, output_dir: str = "annotations"):
        self.output_dir = output_dir
        os.makedirs(output_dir, exist_ok=True)

        # Define class mapping for humanoid robotics
        self.class_mapping = {
            0: "humanoid_robot",
            1: "person",
            2: "chair",
            3: "table",
            4: "cabinet",
            5: "door",
            6: "window",
            7: "obstacle",
            8: "target_object",
            9: "floor",
            10: "wall"
        }

    def generate_bounding_boxes(self, scene_data: Dict[str, Any]) -> List[BoundingBox]:
        """
        Generate bounding box annotations from scene data
        """
        bounding_boxes = []

        # Extract object information from scene
        for obj_name, obj_info in scene_data.get("objects", {}).items():
            if "bbox_2d" in obj_info:
                bbox_2d = obj_info["bbox_2d"]

                # Map class name to ID
                class_name = obj_info.get("class_name", "unknown")
                class_id = self.get_class_id(class_name)

                bbox = BoundingBox(
                    x_min=bbox_2d["x_min"],
                    y_min=bbox_2d["y_min"],
                    x_max=bbox_2d["x_max"],
                    y_max=bbox_2d["y_max"],
                    class_id=class_id,
                    class_name=class_name
                )

                bounding_boxes.append(bbox)

        return bounding_boxes

    def generate_segmentation_masks(self, scene_data: Dict[str, Any]) -> List[InstanceSegmentation]:
        """
        Generate instance segmentation masks from scene data
        """
        segmentation_masks = []

        # Extract segmentation data from scene
        for obj_id, seg_info in scene_data.get("segmentation", {}).items():
            if "mask" in seg_info:
                mask = np.array(seg_info["mask"])

                # Map class name to ID
                class_name = seg_info.get("class_name", "unknown")
                class_id = self.get_class_id(class_name)

                instance_seg = InstanceSegmentation(
                    mask=mask,
                    class_id=class_id,
                    class_name=class_name,
                    object_id=obj_id
                )

                segmentation_masks.append(instance_seg)

        return segmentation_masks

    def generate_keypoints(self, humanoid_data: Dict[str, Any]) -> Dict[str, List[float]]:
        """
        Generate keypoint annotations for humanoid robot
        """
        keypoints = {}

        # Define humanoid joint keypoints
        joint_names = [
            "head", "neck", "left_shoulder", "right_shoulder",
            "left_elbow", "right_elbow", "left_wrist", "right_wrist",
            "left_hip", "right_hip", "left_knee", "right_knee",
            "left_ankle", "right_ankle"
        ]

        for joint_name in joint_names:
            if joint_name in humanoid_data.get("joints", {}):
                joint_pos = humanoid_data["joints"][joint_name]
                # Convert 3D position to 2D image coordinates if needed
                keypoints[joint_name] = [joint_pos[0], joint_pos[1], 2.0]  # x, y, confidence

        return keypoints

    def save_annotations(self, scene_id: str, annotations: Dict[str, Any]):
        """
        Save annotations to JSON file
        """
        annotation_file = os.path.join(self.output_dir, f"{scene_id}_annotations.json")

        # Convert dataclass objects to dictionaries
        if "bounding_boxes" in annotations:
            bbox_list = []
            for bbox in annotations["bounding_boxes"]:
                bbox_dict = {
                    "x_min": bbox.x_min,
                    "y_min": bbox.y_min,
                    "x_max": bbox.x_max,
                    "y_max": bbox.y_max,
                    "class_id": bbox.class_id,
                    "class_name": bbox.class_name,
                    "confidence": bbox.confidence
                }
                bbox_list.append(bbox_dict)
            annotations["bounding_boxes"] = bbox_list

        # Save to file
        with open(annotation_file, 'w') as f:
            json.dump(annotations, f, indent=2)

        print(f"Annotations saved to {annotation_file}")

    def get_class_id(self, class_name: str) -> int:
        """
        Get class ID from class name
        """
        for class_id, name in self.class_mapping.items():
            if name.lower() == class_name.lower():
                return class_id
        return -1  # Unknown class

    def process_scene(self, scene_data: Dict[str, Any], scene_id: str):
        """
        Process a single scene and generate all annotations
        """
        annotations = {}

        # Generate bounding boxes
        bboxes = self.generate_bounding_boxes(scene_data)
        annotations["bounding_boxes"] = bboxes

        # Generate segmentation masks
        seg_masks = self.generate_segmentation_masks(scene_data)
        annotations["segmentation_masks"] = seg_masks

        # Generate keypoints if humanoid is present
        if "humanoid" in scene_data:
            keypoints = self.generate_keypoints(scene_data["humanoid"])
            annotations["keypoints"] = keypoints

        # Add metadata
        annotations["scene_id"] = scene_id
        annotations["timestamp"] = scene_data.get("timestamp", 0)
        annotations["domain_randomization_params"] = scene_data.get("dr_params", {})

        # Save annotations
        self.save_annotations(scene_id, annotations)

        return annotations

# Example usage of annotation pipeline
def example_annotation_pipeline():
    """
    Example of using the synthetic annotation pipeline
    """
    pipeline = SyntheticAnnotationPipeline()

    # Simulated scene data
    scene_data = {
        "objects": {
            "humanoid_0": {
                "bbox_2d": {"x_min": 100, "y_min": 150, "x_max": 200, "y_max": 300},
                "class_name": "humanoid_robot"
            },
            "chair_0": {
                "bbox_2d": {"x_min": 300, "y_min": 200, "x_max": 400, "y_max": 350},
                "class_name": "chair"
            }
        },
        "segmentation": {
            0: {
                "mask": [[1, 1, 0], [1, 1, 0], [0, 0, 0]],  # Simplified mask
                "class_name": "humanoid_robot"
            }
        },
        "humanoid": {
            "joints": {
                "head": [150, 160, 1.0],
                "neck": [150, 180, 1.0],
                "left_shoulder": [130, 200, 1.0]
            }
        },
        "timestamp": 1234567890,
        "dr_params": {
            "lighting": "random",
            "materials": "random",
            "objects": "random"
        }
    }

    annotations = pipeline.process_scene(scene_data, "scene_0001")
    print("Annotation pipeline completed successfully!")
    print(f"Generated {len(annotations.get('bounding_boxes', []))} bounding boxes")

if __name__ == "__main__":
    example_annotation_pipeline()
```

## Benefits and Applications of Synthetic Data

Synthetic data generation with Isaac Sim offers numerous benefits for humanoid robotics:

### Advantages:
1. **Controlled Environment Conditions**: Perfect control over lighting, weather, and scene composition
2. **Ground Truth Annotations**: Automatic generation of pixel-perfect labels for training
3. **Scalable Data Collection**: Generate thousands of diverse scenarios efficiently
4. **Safety During Development**: No risk to physical robots or humans during data collection
5. **Cost-Effective**: Significantly cheaper than real-world data collection
6. **Reproducible Results**: Exact same conditions can be recreated for testing

### Applications:
- Training perception models (object detection, segmentation, pose estimation)
- Sim-to-real transfer learning
- Reinforcement learning environment training
- Testing edge cases and rare scenarios
- Sensor fusion algorithm development

## Integration with Training Pipelines

Connecting Isaac Sim synthetic data generation to actual training pipelines:

```python
import torch
import torchvision.transforms as transforms
from torch.utils.data import Dataset, DataLoader
import cv2
import numpy as np
import json
import os
from PIL import Image

class SyntheticDataset(Dataset):
    """
    Dataset class for loading synthetic data from Isaac Sim
    """
    def __init__(self, data_dir, transform=None, task="classification"):
        self.data_dir = data_dir
        self.transform = transform
        self.task = task

        # Load annotation files
        self.annotation_files = []
        for file in os.listdir(data_dir):
            if file.endswith("_annotations.json"):
                self.annotation_files.append(os.path.join(data_dir, file))

        # Create image file mapping
        self.image_mapping = {}
        for ann_file in self.annotation_files:
            scene_id = os.path.basename(ann_file).replace("_annotations.json", "")
            rgb_file = os.path.join(data_dir, f"rgb_{scene_id}.png")
            if os.path.exists(rgb_file):
                self.image_mapping[ann_file] = rgb_file

    def __len__(self):
        return len(self.annotation_files)

    def __getitem__(self, idx):
        ann_file = self.annotation_files[idx]
        img_file = self.image_mapping[ann_file]

        # Load image
        image = Image.open(img_file).convert('RGB')

        # Load annotations
        with open(ann_file, 'r') as f:
            annotations = json.load(f)

        # Apply transforms
        if self.transform:
            image = self.transform(image)

        # Prepare targets based on task
        if self.task == "detection":
            targets = self.prepare_detection_targets(annotations)
        elif self.task == "segmentation":
            targets = self.prepare_segmentation_targets(annotations)
        else:  # classification
            targets = self.prepare_classification_targets(annotations)

        return image, targets

    def prepare_detection_targets(self, annotations):
        """
        Prepare targets for object detection
        """
        boxes = []
        labels = []

        for bbox in annotations.get("bounding_boxes", []):
            boxes.append([bbox["x_min"], bbox["y_min"], bbox["x_max"], bbox["y_max"]])
            labels.append(bbox["class_id"])

        targets = {
            "boxes": torch.tensor(boxes, dtype=torch.float32),
            "labels": torch.tensor(labels, dtype=torch.int64)
        }

        return targets

    def prepare_segmentation_targets(self, annotations):
        """
        Prepare targets for segmentation
        """
        # This would load and process segmentation masks
        # For now, returning a placeholder
        return torch.zeros((1, 224, 224))  # Placeholder

    def prepare_classification_targets(self, annotations):
        """
        Prepare targets for classification
        """
        # Return class labels
        return torch.zeros(1)  # Placeholder

def create_training_dataloader(data_dir, batch_size=8, task="detection"):
    """
    Create a dataloader for synthetic training data
    """
    # Define transforms
    transform = transforms.Compose([
        transforms.Resize((224, 224)),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406],
                           std=[0.229, 0.224, 0.225])
    ])

    # Create dataset
    dataset = SyntheticDataset(data_dir, transform=transform, task=task)

    # Create dataloader
    dataloader = DataLoader(
        dataset,
        batch_size=batch_size,
        shuffle=True,
        num_workers=4,
        pin_memory=True
    )

    return dataloader

def train_with_synthetic_data():
    """
    Example training function using synthetic data
    """
    # Create dataloader
    dataloader = create_training_dataloader("synthetic_data", batch_size=8)

    # Initialize model (example with torchvision detection model)
    from torchvision.models.detection import fasterrcnn_resnet50_fpn

    model = fasterrcnn_resnet50_fpn(pretrained=False, num_classes=11)  # 10 classes + 1 background
    model.train()

    # Define optimizer
    optimizer = torch.optim.SGD(model.parameters(), lr=0.001, momentum=0.9)

    # Training loop
    for epoch in range(10):  # 10 epochs
        total_loss = 0
        for batch_idx, (images, targets) in enumerate(dataloader):
            optimizer.zero_grad()

            # Forward pass
            loss_dict = model(images, targets)
            losses = sum(loss for loss in loss_dict.values())

            # Backward pass
            losses.backward()
            optimizer.step()

            total_loss += losses.item()

            if batch_idx % 10 == 0:
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {losses.item():.4f}")

        print(f"Epoch {epoch} completed, Average Loss: {total_loss / len(dataloader):.4f}")

# Example usage
def main():
    print("Isaac Sim synthetic data generation setup complete!")
    print("Benefits of synthetic data for humanoid robotics:")
    print("- Photorealistic rendering with PhysX physics")
    print("- Domain randomization for robust perception")
    print("- Automatic annotation generation")
    print("- Scalable data collection pipeline")
    print("- Sim-to-real transfer capabilities")

if __name__ == "__main__":
    main()
```

## Summary

This lesson provided a comprehensive overview of synthetic data generation using Isaac Sim for humanoid robotics applications. We explored the architecture and setup of Isaac Sim, advanced domain randomization techniques, synthetic data annotation pipelines, and integration with training workflows. Isaac Sim's capabilities for photorealistic simulation, physics accuracy, and automated data generation make it an invaluable tool for developing robust perception systems for humanoid robots.

## References

1. NVIDIA. (2022). Isaac Sim: NVIDIA's next-generation robotics simulation application. *NVIDIA Developer*. https://developer.nvidia.com/isaac-sim
2. Sadeghi, F., & Levine, S. (2017). CAD2RL: Real single-image flight without a single real image. *Proceedings of the 1st Annual Conference on Robot Learning*, 209-219. https://proceedings.mlr.press/v78/sadeghi17a.html
3. James, S., Davison, A. J., & Johns, E. (2019). Translating images into maps. *Proceedings of the IEEE/CVF International Conference on Computer Vision*, 4880-4889. https://doi.org/10.1109/ICCV.2019.00498