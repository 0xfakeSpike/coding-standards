# CSS

This document covers styling approach, Tailwind, design tokens, the cascade, layout, animation, and dark mode. For breakpoints and mobile, see [responsive-and-assets.md](responsive-and-assets.md). For contrast and focus styles, see [accessibility.md](accessibility.md). For logical properties used for RTL, see [i18n.md](i18n.md).

## Choosing an approach

### Choose one main approach per project [Recommended]

Choose one of Tailwind, CSS Modules, or Vue scoped as the main approach.

**Why:** When approaches are mixed, it is hard to tell what overrides what, and the same visual gets multiple implementations.

**Exceptions:** Overriding third-party component styles.

### Scope styles by default [Recommended]

Global styles hold only reset, design tokens, and base typography. Scope all other styles to the component. When a global class name is truly needed, use BEM, such as `.card__title--active`.

## Tailwind rules

The `@theme`, `@reference`, `@custom-variant`, and `bg-(--x)` syntax in this section requires Tailwind CSS 4 or later.

### Write complete class names, do not build them dynamically [Required]

Tailwind scans source files as plain text. Map conditional branches and props to complete class name strings.

**Why:** Concatenated class names do not appear in the source, the matching CSS is not generated, and styles silently go missing at runtime.

**Example:**

Avoid:

```tsx
<button className={`bg-${color}-600`}>Save</button>
```

Prefer:

```tsx
const colorVariants = {
  primary: 'bg-blue-600 hover:bg-blue-500',
  danger: 'bg-red-600 hover:bg-red-500',
};

<button className={colorVariants[color]}>Save</button>;
```

**Source:** Tailwind CSS Docs

### Put design system values in `@theme` [Recommended]

Write reusable colors, spacing, fonts, and breakpoints in `@theme`. Use arbitrary values only for one-off pixel adjustments, such as `top-[117px]`. Put plain variables that do not need generated utility classes in `:root`.

**Why:** `@theme` variables generate both utility classes and CSS variables. Writing `bg-[#2563eb]` everywhere bypasses design tokens.

**Source:** Tailwind CSS Docs

### Do not write conflicting utility classes [Recommended]

Keep one utility class per property, and select with a condition, such as `isGrid ? 'grid' : 'flex'`. Component libraries expose props rather than letting callers append conflicting class names.

**Why:** The class that wins is the one later in the generated stylesheet, not the one later in the `class` attribute.

**Source:** Tailwind CSS Docs

### Reuse through components, not custom class names [Recommended]

Extract styles reused across files into components or template fragments. Small single-element styles may go in `@layer components`, where utility classes can still override them. `@apply` is mainly for overriding third-party styles. When using `@apply` in component style blocks or CSS Modules, write `@reference` first. Pass runtime values into CSS variables through `style`, then reference them from utility classes, such as `bg-(--brand-color)`. Leave class sorting to `prettier-plugin-tailwindcss`.

**Source:** Tailwind CSS Docs

## Design tokens

### Use design tokens for visual values [Recommended]

Use design tokens for colors, font sizes, spacing, border radius, and shadows. Do not write literal values everywhere, except for one-off adjustments. Name variables by meaning: use `--color-danger`, not `--color-red`. `var()` works only in property values, not in selectors or media query conditions.

**Why:** Themes and redesigns can then be changed in one place, and dark mode only replaces variable values.

**Example:**

```css
:root {
  --color-text: #111827;
  --space-4: 1rem;
}

.card {
  padding: var(--space-4);
  color: var(--color-text);
}
```

**Source:** MDN

### Manage `z-index` with layered variables [Recommended]

Use layered variables such as `--z-dropdown`, `--z-modal`, and `--z-toast`. Do not write one-upmanship values such as `9999`. When a layer does not take effect, check the stacking context first. `opacity`, `transform`, `filter`, and `will-change` all create stacking contexts, and children cannot escape the parent's context.

**Source:** web.dev Learn CSS

## Selectors and the cascade

### Do not use `!important` [Recommended]

This includes Tailwind's `!` suffix.

**Why:** `!important` does not raise specificity. It moves the declaration into a higher cascade origin, after which only another `!important` can counter it.

**Exceptions:** When overriding third-party styles and no other way truly works, add a one-line comment nearby explaining why.

**Source:** web.dev Learn CSS, Tailwind CSS Docs

### Do not use ID selectors for styling [Recommended]

Style selectors use class names. Use IDs only for anchors and `label` association.

**Why:** One ID outweighs any number of class names in specificity, which makes it hard to override later.

**Source:** web.dev Learn CSS

### Keep specificity as low as works [Recommended]

Use a single class name where possible, and do not nest more than 3 levels. Do not raise specificity with `.btn.btn`, and do not write selectors that depend on DOM structure, such as `.a > div > span`. For base styles that must be easy to override, use `:where()`, which has zero specificity.

**Source:** web.dev Learn CSS

### Manage cascade order with `@layer` [Recommended]

When there are multiple style sources, declare layer order in one statement at the entry point, such as `@layer reset, vendor, components, utilities;`. Layers declared later take precedence.

**Why:** Layer order comes before selector specificity. Unlayered styles beat all layers, so third-party styles must go into a layer. For `!important` inside layers, the order is reversed.

**Source:** MDN

## Layout

### Keep margins off components [Recommended]

Use `gap` or single-direction `margin` for spacing between components, and let the parent control arrangement.

### Use `rem` for font sizes [Recommended]

`rem` is based on the root element and follows the user's browser font size setting. Details such as borders may use `px`. For long text, limit line width with units such as `max-width: 60ch`.

**Source:** web.dev Learn CSS

## Animation

### Prefer animating `transform` and `opacity` [Recommended]

**Why:** These two properties run only in the compositing stage. Animating properties such as `width` and `top` triggers layout and paint, which easily causes jank.

**Source:** web.dev: Animation Guide

### Respect `prefers-reduced-motion` [Recommended]

**Applies to:** production service

Provide reduced versions of large animations such as movement, scaling, and parallax. When keeping necessary feedback, use fades instead of motion.

**Why:** Large motion can make users with vestibular disorders unwell. This maps to WCAG 2.2 2.3.3 (level AAA).

**Example:**

```css
.panel {
  transition: transform 300ms ease;
}

@media (prefers-reduced-motion: reduce) {
  .panel {
    transition: opacity 150ms linear;
  }
}
```

**Source:** MDN, WCAG 2.2

### Use `will-change` only for confirmed performance problems [Recommended]

Do not put it in global styles or on large sets of elements. When needed, set it with script before the animation and reset it afterward.

**Why:** Overuse increases memory and slows the page, and it also creates stacking contexts early.

**Source:** MDN

## Dark mode

### Switch themes through semantic variables [Recommended]

Components reference only semantic variables, and theme switching changes only variable values, for example by rewriting `:root` variables under `prefers-color-scheme: dark`. Do not write paired light and dark literal colors in components. Both themes must meet the contrast requirements in [accessibility.md](accessibility.md).

### Declare `color-scheme` [Recommended]

When supporting dark mode, write `color-scheme: light dark` on `:root`, and write `<meta name="color-scheme">` in `<head>` before the CSS. When offering a manual toggle, change `color-scheme` to the single matching value once a theme is chosen.

**Why:** It determines the default colors of form controls and scrollbars. Without it, controls do not match the page theme, and the page may also flash on load.

**Source:** MDN

### Avoid first-paint flash with a manual toggle [Recommended]

Server-render the user's choice onto `<html>`, or write it from an inline script in `<head>` before paint. The inline script must meet the CSP requirements in [security.md](security.md). In Tailwind projects, use `@custom-variant` to make `dark:` match a class or attribute instead.

**Source:** Tailwind CSS Docs
