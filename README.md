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
   npm init @capacitor/app@latest
   npm i @capacitor/core
   npm i -D @capacitor/cli

   ```
   
2. Initialize Capacitor.config:

   ```bash
   npx cap init
   ```
   
3. Install Android/IOS Plugins:

   ```bash
   npm i @capacitor/android
   #or
   npm i @capacitor/ios
   ```
   
4. Generate native projects:

   ```bash
   npx cap add android
   #or
   npx cap add ios
   ```
5. Sync projects:

   ```bash
   npx cap sync
   ```
6. Open the native IDE to run the application on a device or emulator:

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

## Useful scripts

- `npm run dev` – start the Vite development server.
- `npm run build` – build the web application for production.
- `npm run preview` – locally preview the production build.
