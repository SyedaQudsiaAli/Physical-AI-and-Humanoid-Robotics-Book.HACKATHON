---
sidebar_position: 3
---

# Lesson 2.3: Integrating and Comparing Simulated Sensor Data (LiDAR, Depth, IMU)

This lesson covers integrating and comparing simulated sensor data from LiDAR, Depth, and IMU sensors in humanoid robotics, providing a comprehensive understanding of sensor fusion and data validation techniques.

## Introduction

Simulated sensors provide the perception capabilities for humanoid robots in simulation environments. This lesson explores how to integrate and compare data from different sensor types, with a focus on LiDAR, depth cameras, and IMU sensors. We'll examine sensor fusion techniques, data validation methods, and the unique challenges of working with simulated sensor data in humanoid robotics applications.

## Key Concepts

- Simulated LiDAR sensors and their characteristics
- Depth camera simulation and point cloud generation
- IMU sensor simulation with realistic noise models
- Sensor data fusion and validation techniques
- Comparison methodologies for different sensor modalities
- Noise modeling and uncertainty quantification

## Advanced Sensor Data Integration

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import LaserScan, Image, Imu, PointCloud2, PointField
from geometry_msgs.msg import PointStamped, TransformStamped
from nav_msgs.msg import OccupancyGrid
from std_msgs.msg import Header
import numpy as np
import cv2
from cv_bridge import CvBridge
import sensor_msgs.point_cloud2 as pc2
from tf2_ros import TransformListener, Buffer
import threading
from collections import deque
import time

class AdvancedSensorFusionNode(Node):
    """
    Advanced sensor fusion node for humanoid robot with multiple sensor types
    """
    def __init__(self):
        super().__init__('advanced_sensor_fusion_node')

        # Initialize data storage with time-stamped buffers
        self.lidar_buffer = deque(maxlen=10)
        self.depth_buffer = deque(maxlen=10)
        self.imu_buffer = deque(maxlen=100)  # Higher frequency IMU data

        # ROS 2 publishers
        self.fused_pointcloud_pub = self.create_publisher(PointCloud2, 'fused_pointcloud', 10)
        self.occupancy_grid_pub = self.create_publisher(OccupancyGrid, 'local_occupancy_grid', 10)
        self.fused_imu_pub = self.create_publisher(Imu, 'fused_imu', 10)

        # ROS 2 subscribers with QoS for sensor data
        from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
        sensor_qos = QoSProfile(
            depth=5,
            reliability=ReliabilityPolicy.BEST_EFFORT,
            history=HistoryPolicy.KEEP_LAST
        )

        self.lidar_sub = self.create_subscription(LaserScan, '/humanoid/lidar_scan',
                                                 self.lidar_callback, sensor_qos)
        self.depth_sub = self.create_subscription(Image, '/humanoid/depth_camera/image_raw',
                                                 self.depth_callback, sensor_qos)
        self.imu_sub = self.create_subscription(Imu, '/humanoid/imu/data',
                                               self.imu_callback, sensor_qos)

        # TF2 for coordinate transformations
        self.tf_buffer = Buffer()
        self.tf_listener = TransformListener(self.tf_buffer, self)

        # CV Bridge for image processing
        self.cv_bridge = CvBridge()

        # Sensor fusion parameters
        self.grid_resolution = 0.1  # 10cm resolution
        self.grid_size = 10.0       # 10m x 10m grid
        self.origin_x = 0.0
        self.origin_y = 0.0

        # Timer for fusion processing
        self.fusion_timer = self.create_timer(0.1, self.sensor_fusion_callback)

        self.get_logger().info('Advanced sensor fusion node initialized')

    def lidar_callback(self, msg):
        """Process LiDAR data with timestamp"""
        self.lidar_buffer.append({
            'timestamp': msg.header.stamp,
            'data': msg,
            'frame_id': msg.header.frame_id
        })

    def depth_callback(self, msg):
        """Process depth camera data with timestamp"""
        self.depth_buffer.append({
            'timestamp': msg.header.stamp,
            'data': msg,
            'frame_id': msg.header.frame_id
        })

    def imu_callback(self, msg):
        """Process IMU data with timestamp"""
        self.imu_buffer.append({
            'timestamp': msg.header.stamp,
            'data': msg,
            'frame_id': msg.header.frame_id
        })

    def sensor_fusion_callback(self):
        """Main sensor fusion processing"""
        # Get latest synchronized sensor data
        lidar_data = self.get_latest_lidar()
        depth_data = self.get_latest_depth()
        imu_data = self.get_latest_imu()

        if lidar_data and depth_data:
            # Fuse LiDAR and depth data
            fused_pointcloud = self.fuse_lidar_depth(lidar_data['data'], depth_data['data'])
            if fused_pointcloud:
                self.fused_pointcloud_pub.publish(fused_pointcloud)

        if lidar_data:
            # Generate occupancy grid from LiDAR
            occupancy_grid = self.generate_occupancy_grid(lidar_data['data'])
            if occupancy_grid:
                self.occupancy_grid_pub.publish(occupancy_grid)

        if imu_data:
            # Process IMU data (could include sensor fusion with other sources)
            processed_imu = self.process_imu_data(imu_data['data'])
            self.fused_imu_pub.publish(processed_imu)

    def get_latest_lidar(self):
        """Get the most recent LiDAR data"""
        if self.lidar_buffer:
            return self.lidar_buffer[-1]
        return None

    def get_latest_depth(self):
        """Get the most recent depth data"""
        if self.depth_buffer:
            return self.depth_buffer[-1]
        return None

    def get_latest_imu(self):
        """Get the most recent IMU data"""
        if self.imu_buffer:
            return self.imu_buffer[-1]
        return None

    def fuse_lidar_depth(self, lidar_msg, depth_msg):
        """Fuse LiDAR and depth camera data"""
        try:
            # Convert depth image to OpenCV
            depth_cv = self.cv_bridge.imgmsg_to_cv2(depth_msg, desired_encoding='passthrough')

            # Convert LiDAR ranges to Cartesian points
            lidar_points = []
            angle_increment = lidar_msg.angle_increment
            angle_min = lidar_msg.angle_min

            for i, range_val in enumerate(lidar_msg.ranges):
                if not (float('inf') > range_val > lidar_msg.range_min):
                    continue  # Skip invalid ranges

                angle = angle_min + i * angle_increment
                x = range_val * np.cos(angle)
                y = range_val * np.sin(angle)
                lidar_points.append([x, y, 0.0])  # LiDAR is typically 2D

            # Combine LiDAR and depth points (simplified fusion)
            all_points = np.array(lidar_points)

            # Create PointCloud2 message
            header = Header()
            header.stamp = self.get_clock().now().to_msg()
            header.frame_id = 'base_link'

            # Create point cloud with x, y, z fields
            fields = [
                PointField(name='x', offset=0, datatype=PointField.FLOAT32, count=1),
                PointField(name='y', offset=4, datatype=PointField.FLOAT32, count=1),
                PointField(name='z', offset=8, datatype=PointField.FLOAT32, count=1)
            ]

            # Create the point cloud
            points_list = []
            for point in all_points:
                points_list.append([point[0], point[1], point[2]])

            pointcloud_msg = pc2.create_cloud(header, fields, points_list)
            return pointcloud_msg

        except Exception as e:
            self.get_logger().error(f'Error in LiDAR-depth fusion: {e}')
            return None

    def generate_occupancy_grid(self, lidar_msg):
        """Generate occupancy grid from LiDAR data"""
        try:
            # Create occupancy grid
            grid_width = int(self.grid_size / self.grid_resolution)
            grid_height = int(self.grid_size / self.grid_resolution)
            occupancy_grid = OccupancyGrid()

            occupancy_grid.header.stamp = self.get_clock().now().to_msg()
            occupancy_grid.header.frame_id = 'map'
            occupancy_grid.info.resolution = self.grid_resolution
            occupancy_grid.info.width = grid_width
            occupancy_grid.info.height = grid_height
            occupancy_grid.info.origin.position.x = self.origin_x - self.grid_size/2
            occupancy_grid.info.origin.position.y = self.origin_y - self.grid_size/2

            # Initialize grid with unknown (-1)
            occupancy_grid.data = [-1] * (grid_width * grid_height)

            # Process LiDAR ranges to update occupancy grid
            angle_increment = lidar_msg.angle_increment
            angle_min = lidar_msg.angle_min

            for i, range_val in enumerate(lidar_msg.ranges):
                if not (float('inf') > range_val > lidar_msg.range_min):
                    continue

                angle = angle_min + i * angle_increment
                x = range_val * np.cos(angle)
                y = range_val * np.sin(angle)

                # Convert to grid coordinates
                grid_x = int((x - occupancy_grid.info.origin.position.x) / self.grid_resolution)
                grid_y = int((y - occupancy_grid.info.origin.position.y) / self.grid_resolution)

                # Check bounds
                if 0 <= grid_x < grid_width and 0 <= grid_y < grid_height:
                    # Set cell as occupied (100 = occupied, 0 = free, -1 = unknown)
                    occupancy_grid.data[grid_y * grid_width + grid_x] = 100

            return occupancy_grid

        except Exception as e:
            self.get_logger().error(f'Error generating occupancy grid: {e}')
            return None

    def process_imu_data(self, imu_msg):
        """Process and potentially fuse IMU data"""
        # For now, just return the original IMU data
        # In a real implementation, this might fuse with other sensors
        processed_imu = Imu()
        processed_imu.header = imu_msg.header
        processed_imu.orientation = imu_msg.orientation
        processed_imu.angular_velocity = imu_msg.angular_velocity
        processed_imu.linear_acceleration = imu_msg.linear_acceleration

        # Add any processing or filtering here
        return processed_imu

def main(args=None):
    rclpy.init(args=args)
    sensor_fusion_node = AdvancedSensorFusionNode()

    try:
        rclpy.spin(sensor_fusion_node)
    except KeyboardInterrupt:
        sensor_fusion_node.get_logger().info('Sensor fusion node shutting down...')
    finally:
        sensor_fusion_node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Sensor Noise Modeling and Validation

Realistic noise modeling is crucial for humanoid robotics applications:

```python
import numpy as np
from scipy.spatial.transform import Rotation as R

class SensorNoiseModel:
    """
    Class for modeling realistic sensor noise for humanoid robotics
    """
    def __init__(self):
        # LiDAR noise parameters
        self.lidar_range_noise_std = 0.01  # 1cm standard deviation
        self.lidar_angular_noise_std = 0.001  # 0.057 degrees standard deviation

        # Depth camera noise parameters
        self.depth_noise_bias = 0.005  # 5mm bias
        self.depth_noise_std_factor = 0.001  # 0.1% of depth

        # IMU noise parameters
        self.imu_gyro_noise_density = 1.6e-4  # rad/s/sqrt(Hz)
        self.imu_gyro_random_walk = 1.6e-5  # rad/s^2/sqrt(Hz)
        self.imu_accel_noise_density = 2.0e-3  # m/s^2/sqrt(Hz)
        self.imu_accel_random_walk = 3.0e-3  # m/s^3/sqrt(Hz)

        # Initialize random walk states
        self.gyro_bias = np.zeros(3)
        self.accel_bias = np.zeros(3)

    def add_lidar_noise(self, ranges, angles):
        """
        Add realistic noise to LiDAR measurements
        """
        noisy_ranges = []
        for i, (range_val, angle) in enumerate(zip(ranges, angles)):
            if range_val < 0:  # Invalid range
                noisy_ranges.append(range_val)
                continue

            # Range-dependent noise
            range_noise_std = self.lidar_range_noise_std * (1 + range_val * 0.01)
            noisy_range = range_val + np.random.normal(0, range_noise_std)

            # Angular noise (small effect on range)
            angular_noise = np.random.normal(0, self.lidar_angular_noise_std)
            noisy_angle = angle + angular_noise

            # Apply angular noise effect on range
            corrected_range = noisy_range * np.cos(angular_noise)
            noisy_ranges.append(max(0.0, corrected_range))  # Ensure positive range

        return np.array(noisy_ranges)

    def add_depth_noise(self, depth_image):
        """
        Add realistic noise to depth image
        """
        # Convert to float for processing
        depth_float = depth_image.astype(np.float32)

        # Generate noise based on depth value
        noise_std = self.depth_noise_bias + depth_float * self.depth_noise_std_factor
        noise = np.random.normal(0, noise_std)

        # Add noise
        noisy_depth = depth_float + noise

        # Ensure valid depth values
        noisy_depth = np.maximum(0.1, noisy_depth)  # Minimum 10cm

        return noisy_depth.astype(depth_image.dtype)

    def add_imu_noise(self, gyro_measurement, accel_measurement, dt):
        """
        Add realistic noise to IMU measurements including bias drift
        """
        # Update bias using random walk model
        self.gyro_bias += np.random.normal(0, self.imu_gyro_random_walk * np.sqrt(dt), 3)
        self.accel_bias += np.random.normal(0, self.imu_accel_random_walk * np.sqrt(dt), 3)

        # Add noise to gyro measurement
        gyro_noise_std = self.imu_gyro_noise_density / np.sqrt(dt)
        noisy_gyro = gyro_measurement + self.gyro_bias + \
                     np.random.normal(0, gyro_noise_std, 3)

        # Add noise to accel measurement
        accel_noise_std = self.imu_accel_noise_density / np.sqrt(dt)
        noisy_accel = accel_measurement + self.accel_bias + \
                      np.random.normal(0, accel_noise_std, 3)

        return noisy_gyro, noisy_accel

# Example usage of noise modeling
def simulate_noisy_sensors():
    """
    Example of simulating noisy sensors for humanoid robot
    """
    noise_model = SensorNoiseModel()

    # Simulate LiDAR data
    angles = np.linspace(-np.pi, np.pi, 360)
    true_ranges = 2.0 + 0.5 * np.sin(angles * 5)  # Simulated environment
    noisy_ranges = noise_model.add_lidar_noise(true_ranges, angles)

    # Simulate IMU data
    dt = 0.01  # 100Hz
    true_gyro = np.array([0.01, 0.02, 0.005])  # Small angular velocities
    true_accel = np.array([0.1, 0.2, 9.8])     # Gravity + small acceleration
    noisy_gyro, noisy_accel = noise_model.add_imu_noise(true_gyro, true_accel, dt)

    print(f"LiDAR range error (mean): {np.mean(noisy_ranges - true_ranges):.4f}m")
    print(f"LiDAR range std: {np.std(noisy_ranges - true_ranges):.4f}m")
    print(f"Gyro error (mean): {np.mean(noisy_gyro - true_gyro):.6f} rad/s")
    print(f"Accel error (mean): {np.mean(noisy_accel - true_accel):.6f} m/s²")

if __name__ == "__main__":
    simulate_noisy_sensors()
```

## Sensor Comparison and Validation

Effective comparison of different sensor modalities requires understanding their characteristics:

| Sensor Type | Data Rate | Accuracy | Range | Field of View | Key Limitations |
|-------------|-----------|----------|-------|---------------|-----------------|
| LiDAR | 5-20 Hz | ±1-3 cm | 0.1-100m | 360° H, ±15° V | Reflective surfaces, weather |
| Depth Camera | 15-30 Hz | ±1-5 mm (close), ±5-10 cm (far) | 0.3-5m | 60-90° H, 45-70° V | Lighting, reflective surfaces |
| IMU | 100-1000 Hz | Drift over time | N/A | N/A | Integration errors, bias drift |

### Sensor Validation Techniques

```python
import matplotlib.pyplot as plt
from scipy.spatial.distance import cdist
import statistics

class SensorValidator:
    """
    Class for validating and comparing sensor data
    """
    def __init__(self):
        self.validation_results = {}

    def validate_lidar_data(self, lidar_msg):
        """
        Validate LiDAR data quality
        """
        # Check for invalid ranges
        valid_ranges = [r for r in lidar_msg.ranges if lidar_msg.range_min <= r <= lidar_msg.range_max]
        invalid_count = len(lidar_msg.ranges) - len(valid_ranges)

        # Check for sudden jumps (indicating sensor errors)
        range_diffs = np.diff(valid_ranges)
        jump_count = np.sum(np.abs(range_diffs) > 0.5)  # More than 50cm jump

        # Calculate statistics
        if len(valid_ranges) > 0:
            mean_range = np.mean(valid_ranges)
            std_range = np.std(valid_ranges)
        else:
            mean_range = std_range = 0

        return {
            'valid_percentage': len(valid_ranges) / len(lidar_msg.ranges) * 100,
            'invalid_count': invalid_count,
            'jump_count': jump_count,
            'mean_range': mean_range,
            'std_range': std_range,
            'data_rate': 1.0 / lidar_msg.time_increment if lidar_msg.time_increment > 0 else 0
        }

    def validate_depth_data(self, depth_msg):
        """
        Validate depth image quality
        """
        try:
            depth_cv = CvBridge().imgmsg_to_cv2(depth_msg, desired_encoding='passthrough')

            # Check for invalid depth values
            valid_pixels = np.isfinite(depth_cv) & (depth_cv > 0)
            valid_percentage = np.sum(valid_pixels) / depth_cv.size * 100

            # Check for depth discontinuities (edges)
            grad_x = np.gradient(depth_cv, axis=1)
            grad_y = np.gradient(depth_cv, axis=0)
            grad_magnitude = np.sqrt(grad_x**2 + grad_y**2)
            edge_pixels = grad_magnitude > 0.1  # 10cm difference
            edge_percentage = np.sum(edge_pixels) / grad_magnitude.size * 100

            return {
                'valid_percentage': valid_percentage,
                'edge_percentage': edge_percentage,
                'mean_depth': np.mean(depth_cv[valid_pixels]) if np.any(valid_pixels) else 0,
                'std_depth': np.std(depth_cv[valid_pixels]) if np.any(valid_pixels) else 0,
                'resolution': (depth_msg.width, depth_msg.height)
            }
        except Exception as e:
            return {'error': str(e)}

    def validate_imu_data(self, imu_msg):
        """
        Validate IMU data quality
        """
        # Check orientation quaternion normalization
        quat_norm = np.sqrt(imu_msg.orientation.x**2 + imu_msg.orientation.y**2 +
                           imu_msg.orientation.z**2 + imu_msg.orientation.w**2)
        quat_normalized = abs(quat_norm - 1.0) < 0.01

        # Check for extreme values
        gyro_magnitude = np.sqrt(imu_msg.angular_velocity.x**2 +
                                imu_msg.angular_velocity.y**2 +
                                imu_msg.angular_velocity.z**2)
        accel_magnitude = np.sqrt(imu_msg.linear_acceleration.x**2 +
                                 imu_msg.linear_acceleration.y**2 +
                                 imu_msg.linear_acceleration.z**2)

        extreme_gyro = gyro_magnitude > 10.0  # 10 rad/s
        extreme_accel = accel_magnitude > 20.0  # 20 m/s²

        return {
            'quaternion_normalized': quat_normalized,
            'extreme_gyro': extreme_gyro,
            'extreme_accel': extreme_accel,
            'gyro_magnitude': gyro_magnitude,
            'accel_magnitude': accel_magnitude
        }

# Example validation
def run_sensor_validation_example():
    """
    Example of running sensor validation
    """
    validator = SensorValidator()

    # This would typically be called with real sensor messages
    # For example:
    # lidar_validation = validator.validate_lidar_data(lidar_msg)
    # depth_validation = validator.validate_depth_data(depth_msg)
    # imu_validation = validator.validate_imu_data(imu_msg)

    print("Sensor validation methods available:")
    print("- validate_lidar_data: Check LiDAR data quality")
    print("- validate_depth_data: Check depth image quality")
    print("- validate_imu_data: Check IMU data quality")

if __name__ == "__main__":
    run_sensor_validation_example()
```

## Sensor Fusion Algorithms for Humanoid Robotics

For humanoid robots, sensor fusion is particularly important for balance and navigation:

```python
class HumanoidSensorFusion:
    """
    Specialized sensor fusion for humanoid robot balance and navigation
    """
    def __init__(self):
        # Initialize Extended Kalman Filter for state estimation
        self.state = np.zeros(12)  # [pos, vel, orientation, angular_vel]
        self.covariance = np.eye(12) * 0.1

        # Process noise
        self.process_noise = np.eye(12) * 0.01

        # Measurement noise for different sensors
        self.lidar_noise = np.eye(2) * 0.05  # x, y position from LiDAR
        self.imu_noise = np.eye(6) * 0.01    # orientation, angular vel from IMU
        self.camera_noise = np.eye(3) * 0.1  # position from camera

    def predict(self, dt):
        """
        Prediction step of the filter
        """
        # Simplified state transition (constant velocity model)
        F = np.eye(12)
        F[0:3, 3:6] = dt * np.eye(3)  # position from velocity
        F[6:9, 9:12] = dt * np.eye(3)  # orientation from angular velocity

        # Predict state and covariance
        self.state = F @ self.state
        self.covariance = F @ self.covariance @ F.T + self.process_noise

    def update_lidar(self, lidar_position):
        """
        Update with LiDAR position measurement
        """
        # Measurement matrix for position (first 2 elements)
        H = np.zeros((2, 12))
        H[0, 0] = 1  # x position
        H[1, 1] = 1  # y position

        # Measurement
        z = np.array([lidar_position[0], lidar_position[1]])

        # Innovation
        y = z - H @ self.state
        S = H @ self.covariance @ H.T + self.lidar_noise

        # Kalman gain
        K = self.covariance @ H.T @ np.linalg.inv(S)

        # Update state and covariance
        self.state = self.state + K @ y
        self.covariance = (np.eye(12) - K @ H) @ self.covariance

    def update_imu(self, orientation, angular_velocity):
        """
        Update with IMU measurement
        """
        # Measurement matrix for orientation and angular velocity
        H = np.zeros((6, 12))
        H[0:3, 6:9] = np.eye(3)   # orientation
        H[3:6, 9:12] = np.eye(3)  # angular velocity

        # Measurement
        z = np.concatenate([orientation, angular_velocity])

        # Innovation
        y = z - H @ self.state
        S = H @ self.covariance @ H.T + self.imu_noise

        # Kalman gain
        K = self.covariance @ H.T @ np.linalg.inv(S)

        # Update state and covariance
        self.state = self.state + K @ y
        self.covariance = (np.eye(12) - K @ H) @ self.covariance

    def get_pose_estimate(self):
        """
        Get current pose estimate
        """
        return {
            'position': self.state[0:3],
            'velocity': self.state[3:6],
            'orientation': self.state[6:9],
            'angular_velocity': self.state[9:12],
            'position_uncertainty': np.sqrt(np.diag(self.covariance[0:3, 0:3])),
            'orientation_uncertainty': np.sqrt(np.diag(self.covariance[6:9, 6:9]))
        }

def main_sensor_fusion_example():
    """
    Example of humanoid sensor fusion
    """
    fusion = HumanoidSensorFusion()

    # Simulate sensor readings over time
    dt = 0.01  # 100Hz
    for t in range(1000):  # 10 seconds of simulation
        # Simulate time step
        fusion.predict(dt)

        # Simulate sensor measurements (with some noise)
        if t % 10 == 0:  # LiDAR at 10Hz
            lidar_pos = np.array([t*dt, np.sin(t*dt), 0.0]) + np.random.normal(0, 0.05, 3)
            fusion.update_lidar(lidar_pos)

        if t % 1 == 0:  # IMU at 100Hz
            orientation = np.array([0.0, 0.0, t*dt*0.1])  # Slow rotation
            angular_vel = np.array([0.0, 0.0, 0.1]) + np.random.normal(0, 0.01, 3)
            fusion.update_imu(orientation, angular_vel)

        if t % 5 == 0:  # Print every 50ms
            pose = fusion.get_pose_estimate()
            print(f"Time: {t*dt:.2f}s, Position: [{pose['position'][0]:.3f}, {pose['position'][1]:.3f}], "
                  f"Uncertainty: [{pose['position_uncertainty'][0]:.3f}, {pose['position_uncertainty'][1]:.3f}]")

if __name__ == "__main__":
    main_sensor_fusion_example()
```

## Summary

This lesson provided a comprehensive overview of integrating and comparing simulated sensor data for humanoid robotics applications. We explored advanced sensor fusion techniques, realistic noise modeling, validation methodologies, and specialized algorithms for humanoid balance and navigation. The integration of LiDAR, depth cameras, and IMU sensors enables humanoid robots to perceive and navigate their environment effectively in simulation and real-world applications.

## References

1. Thrun, S., Burgard, W., & Fox, D. (2005). *Probabilistic robotics*. MIT Press. https://mitpress.mit.edu/books/probabilistic-robotics
2. Lobo, J., & Dias, J. (2007). Relative pose calibration between visual and inertial sensors. *The International Journal of Robotics Research*, 26(6), 561-575. https://doi.org/10.1177/0278364907079276
3. Scaramuzza, D., & Fraundorfer, F. (2011). Visual odometry: Part I: The first 30 years and fundamentals. *IEEE Robotics & Automation Magazine*, 18(4), 80-92. https://doi.org/10.1109/MRA.2011.2164865