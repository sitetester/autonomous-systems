# Non-Maximum Suppression (NMS)

Because the YOLO model separates the camera frame into a grid, multiple overlapping squares will often detect the exact same obstacle, e.g.,

- Neighboring cells within the **SAME grid scale** can both claim a large object whose edges spill across cell boundaries
- Different grid scales (8×8 / 16×16 / 32×32) can each independently detect the **SAME object** at their own resolution

This creates a cluster of duplicate bounding boxes around a single object. Non-Maximum Suppression (NMS) is the cleanup step that eliminates these duplicates, ensuring only one perfect box remains per obstacle.

- **The Problem:** If a drone approaches a single tree, the AI engine might draw five different bounding boxes around it at slightly different sizes. Without filtering, the path planner would treat this as five separate obstacles grouped together, wasting computational power.
- **How it Works:** The local pipeline reviews all overlapping bounding boxes and uses a multi-step sorting process:
  - **Find the Leader:** The system selects the box with the absolute highest confidence score and marks it as the master box.
  - **Calculate the Overlap (IoU):** The system measures the intersection-over-union area — meaning how much the other boxes overlap with the master box.
  - **Suppress the Rest:** Any neighbouring box that overlaps the master box beyond a specific cutoff limit (e.g., a 45% overlap threshold) is immediately deleted.
- **The Result:** The cluster of chaotic, repeating boxes is compressed down to a single clean outline, passing a precise location down to the trajectory calculations.

## Example

Suppose three overlapping boxes are detected around the same tree:

| Box | Confidence | IoU with Box A |
|-----|------------|----------------|
| A   | 91%        | — (reference)  |
| B   | 78%        | 0.62           |
| C   | 65%        | 0.20           |

1. **Find the Leader:** Box A (91%) has the highest confidence — it becomes the master box.
2. **Calculate Overlap:** Box B overlaps Box A by 62% (IoU = 0.62); Box C overlaps by only 20% (IoU = 0.20).
3. **Suppress the Rest:** With a 45% IoU threshold, Box B (0.62 > 0.45) gets suppressed — it's clearly the same object as Box A. Box C (0.20 < 0.45) is kept — its overlap is too small to be confidently called "the same object," so NMS treats it as a possibly separate, real obstacle nearby.

**Result:** Only Box A and Box C remain — one confirmed detection per distinct object, no duplicates.