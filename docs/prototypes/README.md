# UI Prototypes

Static HTML prototypes that show the polished end-state of the Pi-Archon desktop UI. These are **not** the real implementation — they exist to lock in the visual language, density, and interaction patterns described in [`../plan/05-design-system.md`](../plan/05-design-system.md) before any Tauri/SvelteKit code is written.

## Two variants

- **`ui-end-state.standalone.html`** — fully offline. Tailwind CSS, Lucide icons, and JS all inlined. ~510 KB. Use this if your network blocks `cdn.tailwindcss.com`, `unpkg.com`, or `fonts.googleapis.com`. Web fonts fall back to the system stack.
- **`ui-end-state.html`** — CDN-based, smaller source (~100 KB), needs an internet connection to render. Use this if you want to fiddle with classes during dev.

Either file: open directly in any modern browser — no build step.

## `screens/` — pre-rendered

PNG screenshots of every screen at 2× retina, rendered with headless Chromium. Use these for slide decks, PR descriptions, or just to skim without opening a browser.

**What's in it:**

- Sidebar shell with workspace switcher, primary nav, and user footer.
- **Dashboard** — KPI tiles, recent-sessions table, factory capacity, audit feed, review-score distribution.
- **Plan Review** — slice DAG canvas (dotted-grid Codemap), side panel with selected slice, agents, attached skills, approve/regenerate actions.
- **Live Run (showpiece)** — three-pane layout (slices · agent stream · artifacts), Plan/Act toggle, live-token caret, queue gauge, agent panel, steering composer, diff preview.
- **Skills** — marketplace grid with scope/source filters, install card.
- **Models & Providers** — provider cards with reachability, model rows, default-per-agent picker.
- **Sessions** list and **People & roles** admin table.
- **Command palette (⌘K)** overlay.

**Keyboard shortcuts in the prototype:**

| Key | Action |
|---|---|
| ⌘/Ctrl + K | Open command palette |
| 1 | Dashboard |
| 2 | Sessions |
| 3 | Plan Review |
| 4 | Live Run |
| 5 | Skills |
| 6 | Models |
| 7 | People & roles |
| Esc | Close palette |

**Stack used (CDN-only, prototype-grade):**

- Tailwind via CDN (the real app uses Tailwind v4 with the SvelteKit Vite plugin).
- Inter and JetBrains Mono via Google Fonts.
- Lucide icons.
- Vanilla JS for nav + palette.

**What this prototype deliberately does:**

- Dark-first neutral palette (zinc base, amber accent, no rainbow gradients).
- Linear / Raycast spacing and weight discipline.
- Subtle animations only — `fade-in`, `caret blink`, `pulse-dot` for live state, `shimmer` on the in-progress slice bar, dashed-edge animation on the active DAG path.
- Tabular numerics everywhere they matter (counts, scores, costs).
- Row-hover actions slide in from the right edge.
- One primary action per view, accent-only.

**What it deliberately doesn't do:**

- No glassmorphism, no drop-shadow dividers, no skeleton loaders, no AI/sparkle decoration.
- No marketing-style hero animations.
- Live data is mocked — the real app drives every panel from gRPC server streams.

## How to view

```sh
# from repo root
open docs/prototypes/ui-end-state.html
# or just double-click the file
```

## Next iteration

When the real app is built, this prototype becomes the visual regression target. Anything that ships should look at least this considered.
