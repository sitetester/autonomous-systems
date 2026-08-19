# Publishing to the System

Once NMS has collapsed the raw detections down to one clean bounding box per obstacle, the final step is getting that data out of the detection node and feeding it into the rest of the drone's software stack.

- **The Format:** The node packages the surviving detections into a `vision_msgs/msg/Detection2DArray` — the standard ROS 2 message type for a list of 2D bounding-box detections. Each entry in the array holds the object's class label, confidence score, and bounding box (center X/Y, width, height), converted back from YOLO's normalized `[0.0–1.0]` output into real pixel coordinates.

**Example entry:**

```yaml
class_id: "0"        # e.g., tree
score: 0.91
bbox:
  center.x: 192
  center.y: 320
  size_x: 200
  size_y: 400
```

**Detection2DArray structure:**

```
Detection2DArray
├── header (timestamp + frame_id)
└── detections[]  (one per surviving post-NMS obstacle)
    ├── results[].hypothesis.class_id  (e.g. "0" — tree)
    ├── results[].hypothesis.score     (confidence, e.g. 0.91)
    └── bbox: { center.x, center.y, size_x, size_y }  (pixel coordinates)
```

**Worked example — the single-tree detection from earlier:**

```
detections[0]:
results[0].hypothesis: { class_id: "0", score: 0.91 }
bbox: { center: {x: 192, y: 320}, size_x: 200, size_y: 300 }
```

- **The Publish:** Every time the node finishes processing a camera frame, it sends the resulting array to a topic (e.g. `/detections`) at the same rate it processes incoming camera frames — so consumers always have a fresh, current list of obstacles rather than a stale one.
- **The Consumers:** Both the path planner & the EKF SF ROS 2 nodes subscribe to `/detections`. The planner acts on it immediately for obstacle avoidance — "there's an obstacle at this position in the frame" combining this info along with "how far away" it is (by using some distance sensor), adjust the trajectory to avoid it. The EKF can use each detection as a measurement update to refine the drone's estimate of nearby obstacle positions alongside its other sensor inputs.
- **The Result:** A raw camera frame has now turned into a structured, low-latency stream of "what, where, how confident" — a lightweight message, a fraction of the size of a raw frame/image, and standardized enough that any other node (planner, fusion, logging) can consume it without needing to know anything about YOLO or NMS internals.