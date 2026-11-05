# Responsive design and static assets

This document covers breakpoints, mobile adaptation, safe areas, images, fonts, icons, and touch. See [html.md](html.md) for how to write `alt`, and [performance.md](performance.md) for the loading priority of above-the-fold images.

## Breakpoints

### Keep one copy of breakpoint constants [Required]

Define breakpoints in one place in the project. CSS and scripts both read from it. Do not write numbers like `768px` in various places. Tailwind projects define them as `--breakpoint-*` in `@theme`, all in the same unit. Other setups generate CSS at build time from one set of constants, because `var()` cannot be used in media query conditions. Sass variables, `@custom-media`, or another approach the project already has all count as one copy. In scripts, use `matchMedia`. Do not read `window.innerWidth` and compare it yourself.

**Why:** Multiple copies of breakpoints eventually diverge, and at some widths the page matches two layouts at once. When Tailwind breakpoints use mixed units, the generated styles are sorted incorrectly.

**Example:**

```css
@theme {
  --breakpoint-*: initial;
  --breakpoint-tablet: 48rem;
  --breakpoint-desktop: 80rem;
}
```

**Source:** Team, Tailwind CSS Docs, MDN

### Go mobile-first and let content set breakpoints [Recommended]

Write base styles for narrow screens, then layer upward with `min-width`. Tailwind's `sm:` means "from this breakpoint up", not "on small screens". Put narrow-screen styles on unprefixed classes. Set breakpoints at the widths where content starts to get hard to read. Do not chase popular device sizes.

**Source:** Tailwind CSS Docs, web.dev Learn Design

### Make components respond to container size [Recommended]

Use container queries for reusable components and media queries for page-level layout. A container's `container-type` is usually `inline-size`.

**Why:** When the same component goes in a sidebar or main column, viewport width does not show its available space. A container cannot rely on its children for its size, or it collapses.

**Source:** web.dev Learn CSS, Tailwind CSS Docs

## Mobile adaptation

### Set viewport and do not disable zoom [Required]

**Applies to:** production service

Use `<meta name="viewport" content="width=device-width, initial-scale=1">`. Do not write `user-scalable=no`, and do not limit zoom with `maximum-scale`.

**Why:** Low-vision users rely on zoom to read. Text must be resizable to 200% (WCAG 2.2 1.4.4).

**Source:** MDN, WCAG 2.2

### Avoid horizontal scrolling on narrow screens [Recommended]

**Applies to:** production service

Content stays usable at 320 CSS pixels wide, without scrolling in two directions at once. Content such as maps, data tables, and video may be excepted.

**Why:** This is WCAG 2.2 1.4.10. Desktop users zoomed to 400% also land at this width.

**Source:** WCAG 2.2

### Use the new viewport units for full-screen height [Recommended]

Use `svh` when content must not be cut off. Use `dvh` when it must fit the visible area, and accept size changes during scrolling. Do not use `100vh` directly: on phones it equals `lvh`, and the bottom is covered when the address bar is expanded.

**Why:** `dvh` changes with the browser toolbar and can cause relayout during scrolling, so prefer the stable `svh`.

**Source:** MDN

### Detect input capability, not screen width [Recommended]

Use the `pointer` and `hover` media features to detect the input method. Content that appears only on hover must also support `:focus` and provide a touch entry point.

**Why:** Laptops may have touchscreens, and tablets may have mice.

**Source:** web.dev Learn Design

## Safe areas

### Stay clear of safe areas when filling the screen [Recommended]

When viewport sets `viewport-fit=cover`, add padding with `env(safe-area-inset-*)` to edge-hugging navigation bars, bottom buttons, and overlays. Use `calc()` to add it to existing spacing, and write a default value.

**Why:** Otherwise content is covered by the notch, rounded corners, or the system gesture bar.

**Example:**

```css
.bottom-bar {
  position: sticky;
  bottom: 0;
  padding-block-end: calc(1rem + env(safe-area-inset-bottom, 0px));
}
```

**Source:** MDN

## Images

### Declare width and height or `aspect-ratio` [Recommended]

Write `width` and `height` on `<img>` and `<video>`, using the intrinsic pixel size. Let CSS do the scaling: `max-inline-size: 100%; block-size: auto`.

**Why:** Otherwise the page jumps after loading, which hurts CLS. Lazy-loaded images without dimensions may all load up front.

**Source:** MDN, web.dev Learn Design, web.dev: Lazy Loading Images

### Serve images by display size [Recommended]

For the same image at several sizes, use `srcset` `w` descriptors with `sizes`. For images with a fixed display size, use `x` descriptors. Use `media` on `<picture>` when the composition must change on narrow screens. `sizes` must match the actual layout.

**Why:** The browser picks an image before parsing CSS, so only the markup can declare this. Swapping URLs with scripts causes an extra download.

**Example:**

```html
<img
  src="hero-800.jpg"
  srcset="hero-480.jpg 480w, hero-800.jpg 800w, hero-1600.jpg 1600w"
  sizes="(width <= 600px) 100vw, 800px"
  width="800"
  height="450"
  alt="Warehouse interior"
/>
```

**Source:** MDN, web.dev Learn Design

### Use AVIF or WebP [Recommended]

Provide formats with `<source type="image/avif">` in `<picture>`, highest compression first. Put a universal format on `<img>` and write `alt`. Add `loading="lazy"` to images outside the initial viewport. Do not lazy-load above-the-fold images. See [performance.md](performance.md) for details.

**Source:** MDN, web.dev Learn Design, web.dev: Lazy Loading Images

## Fonts

### Set an explicit font loading strategy [Recommended]

Write `font-display` in `@font-face`. Body text generally uses `swap`: text is visible immediately but may shift when the font swaps. Use `optional` when a brand font is not essential: there is no shift, but first-time visitors may not see the web font.

**Source:** MDN, web.dev: Font Optimization

### Control font file size [Recommended]

Load only the weights you use, in WOFF2 format. Subset CJK fonts such as Chinese on demand, because a full font is usually several MB. Use `<link rel="preload" as="font">` only for fonts actually used above the fold, with `crossorigin`, including for self-hosted fonts.

**Why:** Too many preloads take bandwidth from critical resources.

**Source:** web.dev: Font Optimization

## Icons

### Use SVG for icons [Recommended]

Use inline SVG or `<use>` when icons must follow the text color, and set the fill to `currentColor`. Compress SVGs exported from design tools first.

**Source:** web.dev Learn Design

### Distinguish decorative and standalone icons [Recommended]

**Applies to:** production service

An icon next to text is decorative: add `aria-hidden="true"` to inline SVG and write `alt=""` on images. An icon that carries meaning on its own needs an accessible name that describes the meaning, not the appearance. See [accessibility.md](accessibility.md) for how. Do not use CSS background images for standalone icons, because background images cannot provide a name.

**Source:** web.dev Learn Design

## Touch

### Make tap targets large enough [Recommended]

**Applies to:** production service

All pointer targets are at least 24x24 CSS pixels. This is WCAG 2.2 2.5.8 (Level AA). When a target is smaller, a 24-pixel circle centered on it must not intersect other targets. Inline links and similar cases are excepted. Make primary actions on mobile 44 to 48 pixels, with about 8 pixels between adjacent targets. For small icons, expand the tap area with padding.

**Source:** WCAG 2.2, Understanding WCAG 2.2, MDN, web.dev: Touch Targets

### Provide a single-pointer alternative to dragging [Recommended]

**Applies to:** production service

Drag features such as sorting, kanban boards, and sliders must also offer a click method, such as up and down buttons or a move-to-column menu.

**Why:** This is WCAG 2.2 2.5.7 (Level AA). Some users cannot press and move a pointer.

**Source:** WCAG 2.2, Understanding WCAG 2.2

### Keep gestures local [Recommended]

Set `touch-action` only on the element with the custom gesture. Use `manipulation` when no custom gesture is needed, because it keeps panning and pinch zoom. Use `none` only for elements such as maps and canvases.

**Why:** `none` can prevent users from zooming.

**Source:** MDN
