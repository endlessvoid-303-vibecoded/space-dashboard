# 🌍 Space Dashboard — For Nerds

A reactive 3D globe visualising global launch sites, each marked by the operating
company/agency logo (SpaceX, NASA, Blue Origin, ESA, …), with live upcoming-launch
data, countdown timers, webcast embeds, and raw-data inspection.

**Data source:** [The Space Devs — Launch Library 2 API](https://ll.thespacedevs.com/2.3.0/) (v2.3.0, keyless)
**Globe engine:** [Globe.gl](https://github.com/vastur-io/globe.gl) (Three.js)
**Framework:** [SvelteKit](https://kit.svelte.dev/)
**Styling:** [Tailwind CSS](https://tailwindcss.com/)

See [PLAN.md](./PLAN.md) for the full technical specification.

## Features

- 🗺️ **3D rotating globe** with launch pad markers at exact lat/lon coordinates
- 🏢 **Company logos** rendered as sprite billboards on the globe surface
- ⏱️ **Live countdown timer** (client-side, updates every second)
- 🎥 **Webcast embeds** — click a launch to watch the live stream
- 📊 **Multiple views** — globe, sortable table, and timeline
- 🔭 **Nerd mode** — raw JSON inspection, rocket specs, mission details
- 💾 **Rate-limit smart** — server-side caching proxy respects LL2's 15 req/hour budget
- 📱 **Responsive** — mobile-friendly side panel overlays
- 🌙 **Dark theme** by default

## Development

```bash
# Install dependencies
npm install

# Start the dev server
npm run dev

# Run tests
npm test

# Run linter
npm run lint

# Build for production
npm run build

# Preview production build
npm run preview
```

## Deployment

Deploys automatically to Vercel on merge to `main` via GitHub Actions.

## License

MIT
