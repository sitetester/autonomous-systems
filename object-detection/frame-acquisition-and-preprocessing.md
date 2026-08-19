
## Frame Acquisition & Preprocessing (OpenCV)
resize -> colour convert -> normalize

### Preprocessing (resize -> colour convert -> normalize)

**Resize:** Changes the high resolution image size (e.g., 1920 x 1080 pixels) to match what the YOLO model was trained to read (e.g., 640x640 pixels).

**Color Convert:** Switches the image colors from standard video format (BGR) to the layout the YOLO model expects (RGB).

**Normalize:** Changes pixel brightness values from [0–255] to simple decimal numbers between [0.0–1.0] for faster processing. Normalization mathematically scales every single pixel array value by dividing it by 255.0. Every single pixel in the image is a 3-value array representing the intensity of three colors: Red, Green, and Blue, e.g., [0, 127, 255] (Blue is low, Green is medium, Red is maximum).

## Inference (YOLO - You Only Look Once)

Once the image is preprocessed, it enters the inference phase where the system scans for obstacles.

### How the YOLO Model Works

**Single Pass:** It runs the preprocessed 640x640 frame through an under-the-hood inference engine like TensorRT via a GPU once, analyzing all areas simultaneously. Internally, the model imposes a virtual grid over the frame at three scales at once — 8x8, 16x16, and 32x32 — with each individual cell assigned a job: the specific cell containing the center of an obstacle is responsible for detecting that entire object.

- 8x8 — for large, close obstacles
- 16x16 — for medium obstacles
- 32x32 — for small, distant obstacles

All three scales run simultaneously, not sequentially — each scanning the same frame at a different resolution, so nothing gets missed regardless of object size.

**Instant Output:** It instantly predicts bounding boxes and confidence scores for any obstacles it spots. Each of the three detection heads outputs its own predictions (boxes + confidence) at its own scale. Afterward, all three sets merge into one combined list, and NMS removes duplicate/overlapping boxes — including cases where two different scales flagged the same object. This "single look" approach makes it incredibly fast, allowing the drone to detect obstacles and update its flight path in real time while flying.

## Model Deployment: Cloud vs. Local Drone

### Training -> Optimization -> Deployment

The model is created and trained in the cloud / backend by ML engineers, but it runs locally on the drone's companion computer.

#### Step 1: Prepared in the Cloud / Backend

The Machine Learning (ML) engineers use powerful cloud servers or desktop workstations equipped with massive enterprise GPUs (like NVIDIA A100s).

- What they do: They feed millions of labeled obstacle images into the YOLO framework to teach it what a tree, building, or power line looks like.
- The output: Once training is complete, they export a single compiled file called the weight file (e.g., yolov8n.pt or an optimized model.engine file). This file contains all the mathematical parameters the model learned.

#### Step 2: Model Optimisation - Cloud to Edge (drone) Conversion

Before the compiled weight file is moved to the drone, it goes through an optimisation step. Cloud models are designed for massive servers, so they must be streamlined to run on a small companion computer.

- What it does: Optimisation software (like NVIDIA TensorRT or ONNX Runtime) rewrites the model's math. It fuses redundant layers together and compresses the file size without losing accuracy.
- Why it matters: This conversion process unlocks the full power of the onboard GPU. It slashes inference times, turning a heavy AI model into a lightweight, high-speed engine tailored perfectly for local drone hardware.

#### Step 3: Where it Runs on the Local Drone

You copy that compiled weight file directly onto the drone's physical hardware. It does not look at the cloud while flying because internet connections are too slow and unreliable for real-time collision avoidance.

It runs locally on an onboard Companion Computer.

- **The Hardware:** This is a small, lightweight circuit board mounted directly onto the drone frame, completely separate from the basic flight controller. Common examples include an NVIDIA Jetson (which contains an embedded local tiny lightweight, low-power GPU) or a Raspberry Pi (using a CPU or a small AI accelerator stick).
- **The Process:** The ROS 2 detection node loads the weight file into this companion computer's memory at boot. When the camera feeds a frame to the node, the onboard GPU runs the mathematical calculations locally on the drone to detect the obstacles instantly.

## What the Neural Network Does Here

- **Pattern Recognition:** It analyses pixel patterns to find distinct shapes, edges, and colour contrasts. They don't know what a tree or a building is yet. They only look for lines and changes in brightness to separate foreground shapes from the background sky.
- **Feature Extraction:** It recognises that a specific cluster of pixels matches the structural "features" of an obstacle.
- **Prediction Generation:** It outputs the exact location boundaries (bounding boxes) and how sure it is about what it sees (confidence scores).

**THE FINAL DATA REPORT:**

- Where is it? [e.g., X coordinate = 0.3 and Y coordinate = 0.5 — 30% of the way across the frame from the left (left-of-center), and exactly halfway down vertically]
- How big? [200 pixels wide]
- How sure are you? [95% Sure]