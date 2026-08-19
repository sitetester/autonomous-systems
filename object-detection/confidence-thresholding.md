# Confidence Thresholding

When the YOLO model outputs its raw predictions during the Inference stage, it generates bounding boxes for everything it thinks might be an obstacle. This step filters those guesses to keep the system accurate:

- **The Setup (ML Backend):** During the cloud training phase, ML engineers train the YOLO model on labeled data & thus the model learns to assign a certainty percentage (0% to 100%) to every object it finds.
- **The Problem:** The model will often make low-certainty guesses on background noise, weather effects, changing shadows, or blurry artifacts (distorted, fuzzy, or messy parts of a video frame). If the drone reacts to these, it will constantly dodge "ghost" obstacles, causing unstable flight.
- **How it Works:** The local pipeline applies a strict cutoff percentage (e.g., a 50% confidence threshold). Any detected object with a certainty score below this limit is immediately wiped out and ignored.
- **The Result:** Only highly reliable, confirmed obstacle detections are allowed to pass through to the next stage of the pipeline.

## Background Noise

In video frames means any visual clutter or useless data that is not an actual obstacle. For the drone's camera, this includes things like:

- **The ground texture:** Rough asphalt, gravel, or waving grass.
- **Weather effects:** Raindrops on the lens, fog, or snow.
- **Camera imperfections:** Grainy video quality in low light or sun glare.

Confidence thresholding improves YOLO model accuracy by filtering out low-certainty detections—such as shadows or background noise—based on a set percentage, ensuring only reliable obstacle data passes to the flight system.

## Bounding Box

In the context of YOLO detection pipeline, a bounding box is a rectangular border that is drawn around a detected obstacle in a camera frame. It tells the drone's flight system exactly where an object is located in the 2D image space using four primary coordinates:

- **X and Y center:** This refers to the x, y coords of the CENTER of object (like the middle of a tree trunk) inside video frame, expressed as a decimal ratio (a number between 0.0 and 1.0) of the frame's total width and height.
- **Width and Height:** The pixel size of the box spanning the object.

**Example:** If the tree's center sits at 30% of the frame's width (measured from the left edge), its X coordinate becomes exactly 0.30, regardless of whether the frame is 640px or 1920px wide. Here is exactly how that looks across the horizontal scale:

### Horizontal X-Coordinate at 30% Width

```
                              HORIZONTAL WIDTH SCALE (0.0 to 1.0)
      0.0 (Far Left)           0.30 (30% Mark)                             1.0 (Far Right)
       ┌────────────────────────┬────────────────────────────────────────────────────────┐
       │                        │                                                        │
       │                        │                                                        │
       │                     ┌──┴──┐                                                     │
       │                     │     │                                                     │
       │                     │ [*] │                                                     │
       │                     │     │                                                     │
       │                     └──┬──┘                                                     │
       │                        │                                                        │
       │                        │                                                        │
       │                        │                                                        │
       └────────────────────────┴────────────────────────────────────────────────────────┘
       ◄───────────────────────►
          30% of Total Width
           Ratio X = 0.30
```

### What This Tells the Drone

1. **Position Shift:** The drone's system reads 0.30 and instantly calculates that the obstacle has shifted toward the left side of its flight path.
2. **Evasion Direction:** Since the tree is taking up the left-center space (0.30), the flight controller knows that turning right (toward 0.70 or 0.80) is the safer, clear path to execute an evasion maneuver.

## Example Output Format

When the YOLO model identifies an obstacle, it does not generate a visual image. Instead, it outputs raw text data consisting of plain numerical strings. The standard format uses decimals between 0.0 and 1.0 (normalized) instead of absolute pixel numbers, keeping the data independent of the camera's resolution:

```
Format: [Class_ID] [Center_X] [Center_Y] [Width] [Height]
```

For example, a single detected obstacle positioned left-of-center in the frame maps to this raw data string:

```
0 0.300000 0.500000 0.312500 0.625000
```

- `0` — Class_ID - The class list (which integer maps to which object category — "tree," "person," "power line," etc.) is defined during dataset labeling/training, on the cloud side by the ML team.
- `0.300000` — Center_X = 30% from the left (left-of-center)
- `0.500000` — Center_Y = dead center vertically
- `0.312500` — Width = the object's bounding box spans about 31.25% of the frame's total width. So for a 640px-wide frame, that's roughly 200 pixels wide (0.3125 × 640 ≈ 200) — this is the box's size, distinct from Center_X (which tells you where the box is positioned, not how big it is).
- `0.625000` — Height = about 62.5% of the frame's total height. For a 640px-tall frame, that's 0.625 × 640 = 400 pixels tall.

The ROS 2 detection node captures this text string and translates the normalized values back into real pixel boundaries—such as top-left (92, 120) and bottom-right (292, 520) —so the flight system can execute real-time evasion maneuvers.