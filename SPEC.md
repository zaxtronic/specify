# T-800 Vision Lite - SPEC

## Summary
Mobile hybrid app (Ionic + Vue + Capacitor) that runs **on-device object detection** in real time and displays a sci‑fi HUD overlay. No internet is required for inference.

## Target Users
Students and everyday users who want a quick “scanner” to detect common objects with a Terminator‑style overlay.

## Core Use Cases
1. Open camera and see live detections with bounding boxes.
2. Show top detected object and confidence in a HUD panel.
3. Work fully offline after the model is loaded.

## Model Choice
- **Model**: Coco‑SSD (TensorFlow.js)
- **Reason**: Fast, pre‑trained, good for real‑time detection on mobile.
- **Model Size**: ~16–20 MB total assets (varies by TFJS version and backend).
- **Classes**: Standard COCO object categories.

## Camera + Inference Flow
1. App requests camera permission via Capacitor.
2. Live video is captured with `getUserMedia` in the WebView.
3. A `requestAnimationFrame` loop:
   - Grabs the current video frame.
   - Runs `model.detect(video)`.
   - Draws bounding boxes on a canvas overlay.
4. The highest confidence detection is shown as “Target”.

## UI Feedback
- **HUD Overlay**: Bounding boxes + label + confidence.
- **Side Panel**:
  - Target object name
  - Confidence %
  - FPS estimate
- **Status**: “Model loading… / Ready / No objects found”

## Performance Requirements
- Target ≥ 15 FPS on mid‑range Android.
- Inference loop must not block UI.
- Use `tf.setBackend('webgl')` when available.

## Offline Requirement
All inference runs locally in the WebView. Network is not required once the model is cached.

## Out of Scope
- Cloud inference
- User authentication
- Uploading images

## Acceptance Criteria
1. App opens camera and shows live preview.
2. Coco‑SSD loads locally and detects at least 5 common objects.
3. Overlay displays bounding boxes and labels.
4. Works without internet during inference.
5. APK builds and installs successfully.

