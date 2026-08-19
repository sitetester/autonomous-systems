# Autonomous Systems — Robotics Portfolio

Self-study documentation for a **ROS 2–based autonomous drone delivery system**, covering perception, sensor fusion, planning, and control — from simulation (Gazebo) to real hardware deployment.

---

## 📄 Documents

| Document | Description |
|----------|-------------|
| **[End-to-End System Architecture](e2e-flow.md)** | Full-stack drone delivery flow: mission planning (QGroundControl/MAVLink) → GNC loop (Guidance, Navigation, Control) → EKF sensor fusion (GPS + IMU + Barometer @ 100 Hz) → vision-based obstacle detection → ROS 2 topic graph (DDS) → MCU firmware (SPI/I2C/CAN, DMA, no OS) → precision landing & failsafes. |
| **[Object Detection Pipeline](object-detection-pipeline.md)** | 2D vision pipeline: frame acquisition → preprocessing → YOLO inference → confidence thresholding → NMS → ROS 2 `Detection2DArray` publishing. Includes cloud-to-edge model deployment (TensorRT/ONNX), multi-scale grid architecture, and timing/latency considerations for real-time flight. |
---