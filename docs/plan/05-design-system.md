# 05 — Design System

The UI is the user's emotional impression of the product. The bar is **Linear, Raycast, Arc, Vercel dashboards** — apps that feel calm, fast, and considered. Polished but not decorative.

This doc captures the non-negotiables. Tokens and components live in `apps/desktop/src/lib/theme/` and `apps/desktop/src/lib/components/ui/`. The visual contract is the prototype at [`../prototypes/ui-end-state.html`](../prototypes/ui-end-state.html); every principle below is demonstrated in the renderings under [`../prototypes/screens/`](../prototypes/screens/).

![Live Run — dense, calm, tabular, single primary action](../prototypes/screens/04-live-run.png)

---

## 1. Voice

| Quality | Meaning |
|---|---|
| **Calm** | The UI does not yell. No bright reds for warnings, no animated spinners for routine state. |
| **Precise** | Numbers, labels, IDs are exact. Truncated values get tooltips with full content. |
| **Honest** | If a thing is loading, say "loading." If it failed, say what failed and why. No "Oops!" |
| **Confident** | One primary action per view. The product knows what you probably want to do next. |
| **Reserved** | We don't celebrate routine success with confetti. Big completions get a quiet animation, nothing more. |

The product is for engineers. Treat the user like a peer, not a tourist.

---

## 2. Color

**Dark-first. Light mode is a port, not a co-equal.**

Base scale: `zinc` (cool neutral) by default; `stone` (warm neutral) as an opt-in user preference.

Accent: single user-configurable warm hue (default: `amber-500` for accent surfaces, `amber-400` for interactive). No secondary brand color. No gradients.

Semantic palette:

| Token | Light | Dark |
|---|---|---|
| `bg.canvas` | `zinc-50` | `zinc-950` |
| `bg.surface` | `white` | `zinc-900` |
| `bg.elevated` | `zinc-100` | `zinc-800` |
| `border.subtle` | `zinc-200` | `zinc-800` |
| `border.default` | `zinc-300` | `zinc-700` |
| `text.primary` | `zinc-900` | `zinc-100` |
| `text.secondary` | `zinc-600` | `zinc-400` |
| `text.tertiary` | `zinc-500` | `zinc-500` |
| `accent.bg` | `amber-500` | `amber-500` |
| `accent.fg` | `zinc-950` | `zinc-950` |
| `success` | `emerald-600` | `emerald-500` |
| `warn` | `amber-600` | `amber-500` |
| `danger` | `rose-600` | `rose-500` |

**Rules:**
- No raw color names in component code — only tokens.
- No more than two semantic colors in a single view above the fold.
- Status colors are used for **state badges only**, never for layout chrome.

---

## 3. Type

- **Sans:** Inter Variable. Used everywhere except code.
- **Mono:** JetBrains Mono. Used for: code, paths, IDs, tokens, hashes, command output, log lines.

Scale (in `rem`, fluid-clamped):

| Token | Size | Use |
|---|---|---|
| `text-2xs` | 0.6875 | Badges, table small caps |
| `text-xs` | 0.75 | Metadata, breadcrumbs |
| `text-sm` | 0.875 | Body, controls (**default**) |
| `text-base` | 1.0 | Reading content |
| `text-lg` | 1.125 | Section headers |
| `text-xl` | 1.375 | Page titles |
| `text-2xl` | 1.75 | Empty-state heroes |

Weights: 400 body, 500 emphasized, 600 headings. No 700+.

Line height: 1.5 for body, 1.2 for headings, 1.6 for long-form reading.

---

## 4. Spacing & rhythm

Base unit: `4px`. Tailwind scale untouched. Component padding follows a strict ladder: 8, 12, 16, 24, 32. Never 6, 14, 18.

Cards use **inner padding 16** (compact lists) or **24** (primary surfaces).

Vertical rhythm between sections: **40px** at base, **24px** in dense panes.

---

## 5. Motion

Library: **Motion** (formerly Framer Motion, since renamed `motion`) for cross-cutting; native Svelte `transition:` for one-off node enters/exits.

Defaults:

| Use case | Spec |
|---|---|
| Page transition | `opacity` 150ms ease-out |
| Card hover lift | `y: -2px`, `shadow: md → lg`, spring `{ stiffness: 300, damping: 20 }` |
| Drawer open | spring `{ stiffness: 260, damping: 26 }` from right, 320ms |
| Modal | scale `0.96 → 1` + opacity, 180ms ease-out |
| Toast | slide-up + fade, 220ms |
| List item insert | `height: auto` morph + fade, 200ms |
| Number tick (KPI) | `motion`'s `useTransform` with `damping: 40`, no bounce |

**Discipline rules:**
- No animation > 300ms unless it's a hero transition the user explicitly triggered.
- Respect `prefers-reduced-motion` — falls back to instant.
- No spinners for actions expected to complete in <200ms.
- For longer ops, use a **progress bar with a real estimate**, not a spinner.

---

## 6. Components — house style

Built on **shadcn-svelte** primitives. Patches:

- **Buttons.** Three variants: `primary` (accent), `secondary` (subtle bg, default), `ghost` (no bg). Icon-only buttons require a tooltip.
- **Inputs.** No floating labels. Label above, helper text below (`text-2xs text-secondary`), error text replaces helper.
- **Tables.** Zebra stripes off. Hovered row uses `bg.elevated`. Row hover-actions appear on the right edge with a subtle slide-in.
- **Cards.** `bg.surface`, `border.subtle`, radius `8px`, `shadow-sm` on hover only.
- **Modals.** Dismiss via Esc, backdrop click, or close button. Always focus-trap.
- **Drawers.** Right-side default; `60vw` max width; sticky header + footer.
- **Tabs.** Underline style, never pill style. Active tab uses accent underline + body color.
- **Toasts.** Top-right stack. Auto-dismiss in 5s; persistent for errors. Max 3 visible.
- **Empty states.** Always include: a one-line headline, a one-line subline, and one CTA.

---

## 7. Iconography

**Lucide** exclusively. 16px in dense UI, 20px in primary surfaces, 24px in hero areas. Never stroke-width inconsistencies in the same view. No emoji.

---

## 7.1 In situ — design discipline applied

The two screens below illustrate the principles in this doc: dark-first neutrals, single amber accent restricted to status and primary actions, monospace only for code/IDs/numbers, tabular numerics throughout, generous card padding with tight inner rhythm, and one primary CTA per view.

![Dashboard — calm density, single accent for the "New session" CTA](../prototypes/screens/01-dashboard.png)

![Plan Review — the dotted-grid Codemap canvas and a single primary "Approve & execute"](../prototypes/screens/02-plan-review.png)

---

## 8. Borrowed UX moves (with attribution)

| Move | Borrowed from | Where we use it |
|---|---|---|
| Plan / Act mode toggle | Cline | Live Run header pill — "Currently in Plan mode" / "Currently executing" |
| Rewindable cumulative diff | Plandex | Review Verdict screen — scrub through slice boundaries |
| Visual codemap of the plan DAG | Windsurf | Plan Review canvas |
| Skill marketplace one-click GitHub install | Claude Code | Skills screen |
| ⌘K command palette | Linear / Raycast | Global (every screen) |
| Row-edge hover actions | Linear | Tables and lists |
| Side-by-side diff | Roo Code | Review Verdict |
| KPI tiles with sparkline | Vercel dashboard | Dashboard screen |
| Status pill with subtle pulse for in-progress | Stripe dashboard | Slice list, agent list |

---

## 9. Accessibility

- **WCAG AA contrast minimum** for all text. Verified by axe in CI.
- All interactive elements reachable by keyboard. Visible focus rings (`ring-2 ring-amber-400 ring-offset-2 ring-offset-bg-canvas`).
- Screen-reader-friendly labels on icon buttons.
- Respect `prefers-reduced-motion`.
- Targets at least 32×32 px.

---

## 10. What we deliberately don't do

- ❌ Gradients as decoration.
- ❌ Glassmorphism.
- ❌ Drop shadows as layout dividers.
- ❌ Skeleton screens longer than 500ms — if the data takes longer, show real progress.
- ❌ "AI" badges or emoji.
- ❌ Marketing-style hero animations inside the product.
