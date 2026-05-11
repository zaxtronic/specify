# T-800 Vision Lite

Aplicació híbrida amb `Ionic + Vue + Capacitor` que fa detecció d’objectes en temps real amb `TensorFlow.js` i `coco-ssd`.

## Funcionalitats
- Detecció en mode càmera, imatge pujada i vídeo pujat.
- Overlay amb bounding boxes, etiqueta i confiança.
- HUD amb FPS, telemetria i estat del model.
- Configuració de rendiment (interval, threshold, input size, mode fast/high).

## Execució local
```bash
npm install
npm run dev
```

## Build web
```bash
npm run build
```

## Build Android (Capacitor)
```bash
npx cap sync android
npx cap open android
```

## Documentació entrega AEA2
- `AEA2_ENTREGA.md`
- `SPEC.md`
- `PLAN.md`
- `TASKS.md`
- `PROMPTS.md`
