# Pomodoro Timer

Minimal SvelteKit pomodoro timer with work/break presets, persistent local settings, and a high-contrast timer display.

Live page: `https://zerviate.github.io/pomodoro-timer/`

## Use

1. Open the live page or run it locally.
2. Pick `25+5`, `50+10`, or enter custom work and break minutes.
3. Press `Start`, `Stop`, or `Reset`.

Settings are stored in browser `localStorage` on the current device.

## Local Setup

```bash
npm install
npm run dev
```

## Build

```bash
npm run build
npm run preview
```

GitHub Pages is deployed by `.github/workflows/deploy-pages.yml` with `BASE_PATH=/pomodoro-timer`.
