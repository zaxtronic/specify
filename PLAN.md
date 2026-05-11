# T-800 Vision Lite - PLAN

## Architecture
- Ionic + Vue UI
- Capacitor for native bridge
- TFJS + Coco‑SSD in WebView
- Overlay canvas on top of `<video>`

## Key Libraries
- `@tensorflow/tfjs`
- `@tensorflow-models/coco-ssd`
- Ionic Vue UI components

## Implementation Steps
1. **Scaffold Ionic/Vue app**
   - `ionic start t800-vision tabs --type=vue` (or blank)
   - Add Capacitor: `npx cap init`
2. **Camera Access**
   - Use `navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } })`
   - Bind stream to `<video>`
3. **TFJS Setup**
   - Load backend: `tf.setBackend('webgl')` then `tf.ready()`
   - Load model: `cocoSsd.load()`
4. **Inference Loop**
   - Use `requestAnimationFrame`
   - Throttle inference to ~10–20 FPS if needed
5. **Overlay**
   - Canvas on top of video
   - Draw boxes + labels + confidence
6. **HUD Panel**
   - Show target object, confidence, status, FPS
7. **Performance**
   - Skip frames if `model.detect` is still running
   - Resize video feed to lower resolution if needed
8. **Capacitor Android Build**
   - `npx cap add android`
   - `npm run build`
   - `npx cap sync android`
   - Open Android Studio and build APK

## Files To Create
- `SPEC.md` (done)
- `PLAN.md` (this file)
- `TASKS.md`
- `PROMPTS.md`

## Risks / Mitigations
- **Slow FPS**: Lower input resolution, reduce detection frequency.
- **Camera permission**: Use proper permissions in `AndroidManifest.xml`.
- **Model load delay**: Show loading indicator and cache model.

