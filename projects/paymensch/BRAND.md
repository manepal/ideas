# Paymensch — Brand Guide

Design system for all Paymensch properties: marketing site, merchant dashboard, developer docs, super admin, API responses, and emails. Tokens-first approach — every color, font, and spacing value references a semantic CSS custom property. Changing the brand identity means editing one file, not 500 components.

---

## Design Tokens

### Color Palette

Dark-mode-first. The primary experience (dashboard, docs) defaults to dark. Marketing site defaults to light. Both modes use the same token names.

#### Brand Colors

| Token | Light | Dark | Role |
|-------|-------|------|------|
| `--color-brand` | `#4F46E5` | `#818CF8` | Primary brand. Buttons, links, active states, logo. |
| `--color-brand-hover` | `#4338CA` | `#A5B4FC` | Button/link hover. Always 1 step darker (light) or lighter (dark). |
| `--color-brand-subtle` | `#EEF2FF` | `#1E1B4B` | Brand tint backgrounds. Selected rows, info callouts. |
| `--color-accent` | `#F59E0B` | `#FBBF24` | CTAs, highlights, new/beta badges, unread indicators. |
| `--color-accent-subtle` | `#FEF3C7` | `#422006` | Accent tint backgrounds. Empty states, onboarding, success summaries. |

**Why indigo + amber:** Indigo is blue-trusted but warm-shifted to distance from cold corporate banking. Amber is the "mensch" — human, warm, unexpected in payments. This combination was validated by ui-ux-pro-max fintech color search: "Gold trust + purple tech" is a recognized fintech palette.

**Anti-pattern avoidance:** The indigo is intentionally blue-shifted (`#4F46E5`, hue ~243°) rather than purple-shifted (`#7C3AED`, hue ~270°) to avoid the "AI purple/pink gradient" anti-pattern flagged by ui-ux-pro-max. When in doubt, lean bluer.

#### Neutrals

| Token | Light | Dark | Role |
|-------|-------|------|------|
| `--color-surface` | `#FAFAF9` | `#0C0A09` | Page background. Warm white/black, never pure #FFF or #000. |
| `--color-surface-raised` | `#FFFFFF` | `#1C1917` | Cards, modals, dropdowns. Sits above surface. |
| `--color-border` | `#E7E5E4` | `#292524` | Subtle borders, dividers, input outlines. |
| `--color-text-primary` | `#1C1917` | `#FAFAF9` | Headings, body text. 4.5:1 minimum on surface. |
| `--color-text-secondary` | `#78716C` | `#A8A29E` | Labels, captions, placeholder text. 3:1 minimum. |
| `--color-text-tertiary` | `#A8A29E` | `#78716C` | Disabled text, legal footnotes. |

#### Semantic Colors

| Token | Light | Dark | Role |
|-------|-------|------|------|
| `--color-success` | `#059669` | `#34D399` | Completed payments, success toasts. Teal-shifted green — feels earned, not robotic. |
| `--color-error` | `#DC2626` | `#F87171` | Failed payments, destructive actions. Muted red, not screaming. |
| `--color-warning` | `#D97706` | `#FBBF24` | Pending states, sandbox mode indicator. |

#### Chart Palette (Grafana + Dashboard)

Accessible, WCAG-compatible, colorblind-safe. Never red/green only.

```
#4F46E5  Indigo (primary series)
#F59E0B  Amber (secondary series)
#06B6D4  Cyan (tertiary)
#8B5CF6  Violet (quaternary)
#EC4899  Pink (fifth — rarely used, only when needed)
#78716C  Warm gray (gridlines, reference lines)
```

### Typography

**Primary: Inter** — all UI text across every site. Single typeface strategy with weight + tracking variations. Validated by ui-ux-pro-max as the recommended system for "developer tools, fintech/trading, dashboards."

**Monospace: JetBrains Mono** — code, API keys, transaction IDs, amounts in tables, webhook payloads, docs code blocks. Validated by ui-ux-pro-max as the recommended pair for "developer tools, documentation, code editors."

#### Scale with Letter-Spacing

| Token | Size / Line | Weight | Tracking | Use |
|-------|------------|--------|----------|-----|
| `text-display` | 48px / 1.2 | 700 | `-0.025em` | Landing page hero only. Not used in dashboard or docs. |
| `text-h1` | 32px / 1.3 | 600 | `-0.0125em` | Page titles. |
| `text-h2` | 24px / 1.4 | 600 | `-0.0125em` | Section headers, modal titles. |
| `text-h3` | 18px / 1.5 | 600 | `0` | Card titles, emphasized body. |
| `text-body` | 16px / 1.5 | 400 | `0` | Body text, form inputs, descriptions. |
| `text-sm` | 14px / 1.25 | 400 | `0` | Labels, captions, table cells. |
| `text-xs` | 12px / 1 | 500 | `+0.025em` | Legal, footnotes, status badges (uppercase). |
| `text-mono` | 14px / 1.5 | 400 | `0` | Code, API keys, IDs, amounts. |

#### Font Loading

```css
/* Critical path — subset Inter for Latin */
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap');
```

`font-display: swap` on all. Reserve space to prevent CLS. Preload Inter 400+600 weights only.

### Spacing

8px grid system. Every margin, padding, and gap is a multiple of 8.

| Token | Value | Use |
|-------|-------|-----|
| `space-xs` | 4px | Icon-to-text gap, badge internal padding |
| `space-sm` | 8px | Inline gaps between chips/tags, compact card padding |
| `space-md` | 16px | Standard card padding, form field spacing, list item gaps |
| `space-lg` | 24px | Page section spacing, modal padding, form group gaps |
| `space-xl` | 32px | Hero spacing, major content breaks between sections |
| `space-2xl` | 48px | Landing page section separation |

Border radius: `4px` for inputs and buttons (precise), `8px` for cards and modals (friendly), `12px` for marketing hero cards.

### Icons

**Lucide** — all icons across every site. 1.5px stroke, 24px default size with `--color-text-secondary`. Filled variants for active/selected states only. No mix of outline and filled at the same hierarchy level.

Icon sizes as design tokens: `icon-sm` (16px), `icon-md` (24px), `icon-lg` (32px). Touch target expansion via `hitSlop` when icon is smaller than 24px — minimum interactive area 44×44px everywhere.

### Motion

| Token | Value | Use | Reduced Motion |
|-------|-------|-----|----------------|
| `duration-instant` | 100ms | Hover states, focus rings | 0ms |
| `duration-fast` | 150ms | Micro-interactions, toasts, tooltips | 0ms |
| `duration-normal` | 250ms | Transitions, expand/collapse, page enter | 0ms |
| `duration-slow` | 350ms | Modal enter/exit, drawer slide | 0ms |
| `easing-enter` | `cubic-bezier(0, 0, 0.2, 1)` | Elements appearing | `linear` |
| `easing-exit` | `cubic-bezier(0.4, 0, 1, 1)` | Elements disappearing | `linear` |

**Rules:** Never animate width/height/top/left — transform + opacity only. Exit animations ~60% of enter duration. Spring physics preferred for drag/flick interactions. Respect `prefers-reduced-motion` — all durations become 0ms.

---

## Logo

**Wordmark only.** "paymensch" set in Inter 600 with `-0.0125em` tracking. All lowercase. The "m" in "mensch" has a subtle flowing curve in the final stroke — a small distinctive detail, not a logo mark. No icon. No shield. No rupee symbol.

**Variants:**
- Full color: `--color-brand` indigo
- Reversed: white on `--color-brand` background
- Monochrome: `--color-text-primary` for embedded contexts

**Favicon:** "pm" monogram in a rounded square (8px radius at 48px canvas). Indigo fill, white letters. Designed to read clearly at 16×16px, 32×32px, and 180×180px (Apple Touch Icon).

**Clear space:** Minimum padding equal to the height of the "p" ascender on all four sides.

**Sizes:** SVG source. PNG exports at 1x, 2x, 3x. Favicon as .ico + .svg + apple-touch-icon.png.

---

## Site-Specific Palettes

### Marketing (paymensch.io) — Light Mode
Uses light tokens exclusively. Hero background: `--color-brand-subtle` (soft indigo tint). No dark mode toggle — the dashboard handles that.

### Dashboard (app.paymensch.io) — Dark-First
Default theme: dark. Toggle available in settings. Long-session product — dark mode reduces eye strain. Charts default to dark gridlines.

### Docs (docs.paymensch.io) — Dark-First
Matches dashboard. Code blocks use a warm dark background (`#1C1917`) with JetBrains Mono in `--color-text-primary`. No theme toggle — follows system preference.

### Admin (admin.paymensch.io) — Dark-First
Matches dashboard. Sandbox/live mode indicator in `--color-accent` (sandbox) and `--color-success` (live).

### Emails — Light
Transactional emails use light tokens for maximum email client compatibility. Logo: reversed white on indigo header.

---

## Illustration Style

Line art with a single accent color. Thin (1.5px), precise strokes matching Lucide's visual weight. Characters (when used) are minimal — faceless, gesture-focused. Accent fills via `--color-accent-subtle` backgrounds.

**When to illustrate:** Empty states, onboarding welcome, error pages (500 not 404), sandbox setup guide.

**When NOT to illustrate:** Transaction rows, gateway config forms, API key cards, settings pages. These are data-dense and dead-serious — decoration erodes trust.

**Generation:** SVG by hand or AI-assisted with post-processing for stroke consistency. No stock illustration libraries. No photos of people. No handshake images.

---

## Tone of Voice

### Product (Dashboard UI, Error Messages, Emails)
Precise. Calm. Direct. Never quirky. Never apologetic. The user is an adult running a business. A failed payment says: "Payment declined by eSewa. Try another gateway or contact eSewa support." Not "Oops! Something went wrong."

### Docs (API Reference, Guides, Quickstarts)
Clean. Technical. Respectful of the reader's intelligence. No hype words ("seamless," "frictionless," "revolutionary," "cutting-edge"). Code before explanation. "Here's a curl command that makes a payment." Not "Our platform enables synergistic payment orchestration."

### Marketing (Landing, Pricing, Blog)
Confident. Warm. Slightly opinionated. The voice of a senior developer explaining why they built this to another developer. "One API. Every Nepali payment gateway. That's it." Short sentences. No jargon cascade.

### Word Bank

| Use | Avoid |
|-----|-------|
| Merchant | User, customer, client |
| Payment / Transaction | Purchase, order (that's the merchant's domain) |
| Gateway | Processor, provider |
| Sandbox / Live | Test mode / Production (sandbox = developer term) |
| Initiate / Capture / Refund | Start / Take / Return (be precise) |
| API key | Token, password |

---

## Component Guidelines

### Buttons

Three variants across all sites: primary (indigo fill), secondary (indigo outline), ghost (transparent, for icon-only and toolbar actions).

Four sizes: `sm` (32px height), `md` (40px), `lg` (48px, CTAs only), `icon` (40×40px square).

Loading state: spinner replaces text, button width locks to prevent layout shift. Disabled: opacity 0.4, `cursor: not-allowed`, `aria-disabled`.

### Forms

Labels above inputs (not placeholder-only). Error below the field. Helper text below that. Required fields marked with `*` in `--color-text-secondary`.

Validation on blur, not keystroke. Focus ring: `2px solid var(--color-brand)` with `2px` offset.

Input height minimum 44px (touch target). Placeholder text in `--color-text-tertiary`.

### Tables

Background: `--color-surface-raised` (card appearance). Header row: `--color-brand-subtle` background, `text-xs` uppercase with `+0.025em` tracking.

Rows: alternating `--color-surface` / `--color-surface-raised` for readability. Row hover: `--color-brand-subtle` tint. Selected row: `--color-brand-subtle` with a 2px left border in `--color-brand`.

Amount columns: JetBrains Mono, right-aligned, tabular figures.

### Cards

Background: `--color-surface-raised`. Border: `1px solid var(--color-border)`. Radius: `8px`. Padding: `space-lg` (24px).

Hover: subtle shadow lift (`box-shadow` transition 250ms) for clickable cards. Non-clickable cards don't hover.

### Status Badges

Rounded pill. `text-xs` uppercase. Background: semantic color at 10% opacity + matching text color.

| Status | Color |
|--------|-------|
| Completed / Active / Live | `--color-success` |
| Failed / Suspended / Error | `--color-error` |
| Pending / Processing / Sandbox | `--color-warning` |
| Draft / Inactive / Disabled | `--color-text-tertiary` |

---

## Tailwind Config Strategy

Tailwind `theme.extend` reads from CSS custom properties, not hardcoded values:

```js
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        brand: 'var(--color-brand)',
        'brand-hover': 'var(--color-brand-hover)',
        'brand-subtle': 'var(--color-brand-subtle)',
        accent: 'var(--color-accent)',
        'accent-subtle': 'var(--color-accent-subtle)',
        surface: 'var(--color-surface)',
        'surface-raised': 'var(--color-surface-raised)',
        border: 'var(--color-border)',
        'text-primary': 'var(--color-text-primary)',
        'text-secondary': 'var(--color-text-secondary)',
        'text-tertiary': 'var(--color-text-tertiary)',
        success: 'var(--color-success)',
        error: 'var(--color-error)',
        warning: 'var(--color-warning)',
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
        mono: ['JetBrains Mono', 'monospace'],
      },
      borderRadius: {
        input: '4px',
        card: '8px',
        hero: '12px',
      },
      spacing: {
        xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px', '2xl': '48px',
      },
    },
  },
};
```

---

## Switching the Brand

To change the entire brand identity, edit exactly **8 CSS variables** in `--color-brand*` and `--color-accent*` groups. Everything else is semantic and stays.

```css
/* Current: Indigo + Amber */
--color-brand: #4F46E5;
--color-accent: #F59E0B;

/* Rebrand: Terracotta + Warm Sand */
--color-brand: #C46A4A;
--color-accent: #E8D5B5;

/* Rebrand: Ochre + Deep Charcoal */
--color-brand: #D4A030;
--color-accent: #2D2824;
```

All components, charts, badges, and illustrations follow the tokens. The rebrand is a CSS-only change.

---

## Pre-Delivery Checklist (Per UI/UX Pro Max)

Before any UI code ships, verify:

### Visual Quality
- [ ] No emojis as icons — Lucide SVG only
- [ ] Consistent icon family, stroke width, and corner radius
- [ ] Semantic tokens used — no raw hex colors in components
- [ ] Pressed states don't shift layout (transform/opacity only)

### Interaction
- [ ] Touch targets ≥44×44px (hitSlop for small icons)
- [ ] Micro-interactions 150–300ms with native easing
- [ ] Disabled states low-opacity + `cursor: not-allowed` + `aria-disabled`
- [ ] Focus order matches visual order

### Light/Dark Mode
- [ ] Primary text contrast ≥4.5:1 in BOTH modes
- [ ] Secondary text contrast ≥3:1 in BOTH modes
- [ ] Dividers/borders visible in both modes
- [ ] Both themes tested before delivery

### Layout
- [ ] Safe areas respected (headers, tab bars, CTA bars)
- [ ] Content not hidden behind fixed/sticky bars
- [ ] Verified at 375px small phone + 768px tablet + 1024px desktop
- [ ] 4/8px spacing rhythm consistent

### Accessibility
- [ ] All images/icons have labels
- [ ] Forms: visible labels, errors near fields, helper text
- [ ] Color never the only differentiator (status = badge shape + color + text)
- [ ] `prefers-reduced-motion` respected — all durations become 0ms
- [ ] Dynamic type scaling supported (text doesn't truncate or overflow)
