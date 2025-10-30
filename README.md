# Camera Journal

A Vue 3 + TypeScript progressive camera journal that works on the web and can be packaged as a native application with Capacitor. Capture photos, add a short description, and keep your memories synced across devices.

## Getting started

```bash
npm install
npm run dev
```

The development server runs on [http://localhost:5173](http://localhost:5173).

### Building for production

```bash
npm run build
```

The production-ready assets are output to the `dist/` directory.

## Native builds with Capacitor

1. Initialize the native platforms you want to target:

   ```bash
   npx cap add android
   npx cap add ios
   ```

2. Each time you build the web app, copy the files into the native project:

   ```bash
   npm run build
   npx cap copy
   ```

3. Open the native IDE to run the application on a device or emulator:

   ```bash
   npx cap open android
   # or
   npx cap open ios
   ```

The app automatically uses the Capacitor Camera plugin when running on a native platform and falls back to the browser file picker when running on the web.

## Project structure

- `src/App.vue` – orchestrates capturing photos, collecting descriptions, and persisting data in `localStorage`.
- `src/components/CameraCapture.vue` – handles photo capture via the Capacitor Camera plugin or a browser fallback.
- `src/components/PhotoGallery.vue` – renders the responsive gallery of saved memories.
- `capacitor.config.ts` – Capacitor project configuration shared by all native targets.

## Useful scripts

- `npm run dev` – start the Vite development server.
- `npm run build` – build the web application for production.
- `npm run preview` – locally preview the production build.
