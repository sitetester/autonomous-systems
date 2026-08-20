# Autonomous Drone Delivery System - E2E Architecture (Self Study)

At initial stage, the entire system is validated in Gazebo - a physics-based robotics simulator. A virtual drone model with simulated sensors (GPS, IMU, camera, barometer) operates in a 3D environment. ROS2 nodes connect to the simulated drone exactly as they would to real hardware, enabling full integration testing of perception, planning, and control logic without physical flight risk. Scenarios like GPS dropout, wind gusts, dynamic obstacles, and sensor failures are scripted and tested repeatedly. Once validated in simulation, the same ROS2 stack deploys to real hardware with minimal changes.

An operator plans a delivery mission in QGroundControl and transmits it to the drone (Telemetry Radio encodes and transmits MAVLink packets as radio signals). Mission waypoints include: takeoff location, navigation route (altitude gain, intermediate waypoints), delivery zone (GPS coordinates: lat, lon, altitude), and return home. The drone's onboard Telemetry Radio receives and decodes the signals. The FCU (running ArduPilot firmware) reads the waypoints, arms the motors and begins the flight. GNC continuously operates - Guidance computes the trajectory, Navigation estimates position via sensor fusion, and Control adjusts motors to follow the path.

While flying, the drone constantly estimates its own position by combining data from multiple sensors: GPS provides location, IMU tracks movement and orientation, Barometer measures altitude (by comparing current air pressure against launch time ground pressure). Since each sensor has noise and drift, a Kalman Filter fuses them into a single reliable state estimate (mathematically reducing noise from individual sensors). This process is updated ~100 times per second.

At the same time, the camera feeds live video into an object detection pipeline. OpenCV captures and preprocesses each frame (resize, normalize, color convert), then YOLO processes it to identify obstacles - outputting what was detected, where it is, and how confident the model is (class labels + bounding boxes + confidence scores). NMS then removes duplicate overlapping detections, leaving one clean detection per object. If an obstacle is detected, the flight path is adjusted automatically by GNC.

ROS2 acts as the nervous system - all components (sensors, detectors, planners) communicate by publishing and subscribing to data topics via DDS (Data Distribution Service), which handles discovery, serialization and message delivery, e.g.:

- Kalman Filter node subscribes to /gps/fix, /imu/data, /baro/data and publishes to /state_estimate
- YOLO detector node subscribes to /camera/image_raw and publishes to /object_detections
- Path planner node subscribes to /state_estimate, /object_detections and publishes to /desired_trajectory
- GNC/controller node subscribes to /desired_trajectory, /state_estimate and publishes to /motor_commands
Each component is independent and can be swapped or tested in isolation.

At the hardware level, the MCU initializes peripherals at boot (SPI / I2C buses, timers, GPIO), reads sensor data continuously via interrupt-driven or DMA-based acquisition, converts raw ADC values to physical units via calibration coefficients, and shares processed data with other MCUs over CAN bus - all running with deterministic timing and no OS overhead.

When the drone reaches the delivery zone, CV algorithms analyze the landing area in real time - identifying a safe, flat spot clear of obstacles and people. The drone descends precisely using visual servoing and state estimation, releases the package, and returns to the home waypoint.

The system continuously monitors system health (GPS loss, low battery, sensor failures, radio link dropout) and activates pre-programmed behavior (loiter, return-to-home, or land). Manual RC override remains available as final fallback.
