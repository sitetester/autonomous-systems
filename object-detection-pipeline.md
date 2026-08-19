# Overview

This pipeline outputs 2D image-space detections only — no distance/depth information. Real-world obstacle range must come from a separate sensor (or estimation method) and gets combined with these detections downstream, most likely in the sensor fusion layer.

While the drone flies its mission, a forward-facing camera streams live video into a detection pipeline running independently of the sensor fusion loop (see [sensor-fusion-ekf.md](sensor-fusion-ekf.md)). Its job is to turn raw camera frames into a clean list of obstacles, tracking "what it is, where it is in the camera frame, and how confident the system is" (class label, bounding box, confidence score) fast enough for the path planner to react. Note that "where it is" means a location in the 2D image, not a real-world position; the box simply marks which rectangle of pixels the obstacle occupies in the frame. The pipeline operates as an independent ROS 2 node, subscribing to `/camera/image_raw` and publishing detections as `vision_msgs/msg/Detection2DArray` — the standard ROS 2 message type for 2D bounding-box detections.

## Core Pipeline Stages

| Stage                                    | Description | Details |
|------------------------------------------|-------------|---------|
| 1. **Frame Acquisition & Preprocessing** | OpenCV resize → color convert → normalize | [Details](object-detection/frame-acquisition-and-preprocessing.md) |
| 2. **Confidence Thresholding**           | Filter low-certainty detections | [Details](object-detection/confidence_thresholding.md) |
| 3. **Non-Maximum Suppression (NMS)**     | Remove duplicate overlapping boxes | [Details](object-detection/nms.md) |
| 4. **Publishing to the System**          | ROS 2 `Detection2DArray` output | [Details](object-detection/publishing.md) |
| 5. **Timing & Rate Considerations**      | Latency budgets, hardware constraints | [Details](object-detection/timing.md) |

## Model Deployment

- [Cloud Training → Edge Optimization → Local Deployment](object-detection/frame-acquisition-and-preprocessing.md#model-deployment-cloud-vs-local-drone)

## Neural Network Internals

- [Pattern Recognition, Feature Extraction, Prediction Generation](object-detection/frame-acquisition-and-preprocessing.md#what-the-neural-network-does-here)