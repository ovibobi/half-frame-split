# Half-Frame Film Splitter

A web application for splitting scanned half-frame film images into individual frames. Perfect for digitizing half-frame film photography where two vertical frames are captured on a standard 35mm horizontal frame.

## Features

- **Automatic Image Splitting**: Automatically detects image orientation and splits horizontally or vertically
- **Batch Processing**: Import and process multiple images at once with progress tracking
- **Image Rotation**: Rotate individual halves with directional controls (top, right, bottom, left)
- **Fullscreen Preview**: Click any half to view in fullscreen with zoom and pan controls
- **Selective Download**: Choose which frames to download with checkboxes
- **Batch Download**: Download all selected frames with progress tracking
- **Responsive Design**: Works on desktop and mobile devices
- **Adjustable Grid**: Zoom slider to adjust thumbnail size

## How It Works

1. **Add Images**: Upload one or more scanned film images
2. **Auto-Split**: Images are automatically split into left/right or top/bottom halves
3. **Rotate**: Use directional buttons to orient each frame correctly
4. **Select & Download**: Check the frames you want and download them all at once

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Run Unit Tests with [Vitest](https://vitest.dev/)

```sh
npm run test:unit
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```

## Technology Stack

- **Vue 3**: Progressive JavaScript framework
- **TypeScript**: Type-safe development
- **Vite**: Fast build tool and dev server
- **Vitest**: Unit testing framework

## License

MIT
