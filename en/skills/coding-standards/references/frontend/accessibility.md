# Accessibility

This document sets requirements for keyboard, focus, ARIA, contrast, and common components. For element semantics, `label`, and `alt`, see [html.md](html.md). For click targets, see [responsive-and-assets.md](responsive-and-assets.md). For form errors, see [forms.md](forms.md). For how to write tests, see [testing.md](testing.md).

**Applies to:** production service (all rules in this document)

## WCAG 2.2 key points

### Use WCAG 2.2 level AA as the baseline [Recommended]

Level AA requires meeting all level A and level AA success criteria. Beyond the rules in this document, focus on the criteria added in 2.2.

| Number | Name | Level | Practice |
|---|---|---|---|
| 2.4.11 | Focus Not Obscured (Minimum) | AA | Fixed layers do not fully cover the focused element |
| 2.5.7 | Dragging Movements | AA | Dragging has a single-pointer alternative |
| 2.5.8 | Target Size (Minimum) | AA | Targets are at least 24x24 CSS pixels |
| 3.2.6 | Consistent Help | A | Help entry points stay in the same position across pages |
| 3.3.7 | Redundant Entry | A | Content already entered in the flow is filled in automatically or selectable, except in cases such as confirming a new password |
| 3.3.8 | Accessible Authentication (Minimum) | AA | Login does not depend on memory or transcription |

**Source:** WCAG 2.2, Understanding WCAG 2.2

### Do not block paste or autofill in login flows [Required]

Allow paste in password and verification code fields, and do not block password managers. Verification codes split across multiple boxes must accept the whole code in one paste. Transcription codes and arithmetic challenges need an alternative that does not rely on a cognitive test.

**Why:** Blocking paste forces users to transcribe, which fails 3.3.8.

**Source:** Understanding WCAG 2.2

## Keyboard

### Make all functionality keyboard operable [Required]

Anything the mouse can do, the keyboard can do, and focus never gets trapped inside a component (2.1.1, 2.1.2). Controls built with a custom `role` must implement keyboard behavior themselves.

**Tooling:** `jsx-a11y/click-events-have-key-events`.

**Source:** WCAG 2.2, WAI-ARIA APG

### Make focus order follow the DOM [Recommended]

Use only `0` and `-1` for `tabindex`, never positive values. To change the order, adjust the DOM. Do not let visual order drift from DOM order (2.4.3). The first focusable element inside `<body>` is a link that skips to `<main>`. It may stay hidden until it receives focus (2.4.1).

**Tooling:** `jsx-a11y/tabindex-no-positive`.

**Source:** WAI-ARIA APG, web.dev Learn Accessibility, MDN, WCAG 2.2

## Focus

### Keep focus styles visible [Required]

Do not remove focus styles with `outline: none` without providing a replacement. For custom styles, use `:focus-visible` with at least 3:1 contrast against adjacent colors, and keep them distinct from the selected state.

**Why:** Removing the focus indicator violates 2.4.7 (level AA), and keyboard users cannot tell where they are.

**Example:**

```css
button:focus-visible {
  outline: 2px solid var(--color-focus);
  outline-offset: 2px;
}
```

**Source:** Understanding WCAG 2.2, WAI-ARIA APG, MDN

### Do not let fixed layers cover focus [Recommended]

With a sticky header, bottom bar, or cookie banner, reserve space with `scroll-padding-block`.

**Source:** Understanding WCAG 2.2

### Do not lose focus [Recommended]

Before an element is removed or hidden, move focus to a sensible nearby location. Do not let it fall back to `<body>`. After a route change in a single-page app, move focus to the new page's main heading or `<main>`, and set `tabindex="-1"` on the target element.

**Why:** After focus is lost, keyboard and screen reader users have to start again from the top of the page.

**Source:** WAI-ARIA APG

## ARIA

### Prefer native elements [Recommended]

Do not use ARIA when a native element does the job. Do not add redundant roles such as `<ul role="list">`, and do not change native semantics. `role` provides no keyboard behavior or styling. The author must implement those.

**Why:** Wrong ARIA is worse than no ARIA.

**Source:** WAI-ARIA APG, web.dev Learn Accessibility

### Do not hide focusable elements [Required]

Do not use `aria-hidden="true"` or `role="presentation"` on focusable elements or their ancestors.

**Why:** Users land on a "blank" that screen readers cannot read.

**Source:** web.dev Learn Accessibility

### Give every control an accurate name [Required]

Prefer visible text. Use `aria-labelledby` when there is visible text to reference. Use `aria-label` only when neither applies, for example the icon-only button `<button aria-label="Close">`. The name must include the text of the visible label (2.5.3). One to three words is enough. Do not include role words such as "button". For supplementary descriptions, use `aria-describedby`, not `title`.

**Why:** A control's name, role, and value must be programmatically readable (4.1.2). `aria-label` on a button overrides the text inside the button.

**Source:** WAI-ARIA APG, WCAG 2.2

### Announce dynamic state with live regions [Recommended]

For messages that do not move focus, such as save success, search result counts, and loading, use `role="status"` and write `aria-live="polite"`. Use `role="alert"` only for important, urgent errors. Put the container in the initial HTML and write content into it later. For partially updated text such as counters, add `aria-atomic="true"`.

**Why:** Otherwise screen reader users do not receive the message, which violates 4.1.3 (level AA). Dynamically inserted containers may not be announced, except `role="alert"`.

**Source:** Understanding WCAG 2.2, MDN

### Use `aria-disabled` for disabled items that must stay discoverable [Recommended]

When controls such as toolbar buttons, menu items, and tabs are disabled, keep them focusable, write `aria-disabled="true"`, and block the action.

**Why:** Native `disabled` removes the control from the Tab order, so screen reader users cannot discover it.

**Source:** WAI-ARIA APG

## Contrast

### Meet AA contrast [Required]

Body text and text in images need at least 4.5:1. Large text (at least 18pt, or 14pt bold) needs at least 3:1 (1.4.3). Required graphics such as icons, input borders, and state indicators need at least 3:1 against adjacent colors (1.4.11). Disabled controls, purely decorative text, and logos are exempt. Check both dark and light themes.

**Source:** WCAG 2.2, Understanding WCAG 2.2

### Do not rely on color alone [Required]

Errors, required fields, and states must also have text or an icon. Links in body text must have an underline in addition to color, or 3:1 contrast with the surrounding text (1.4.1). Charts distinguish series with patterns or labels.

**Source:** Understanding WCAG 2.2, web.dev Learn Accessibility

### Keep content usable after zoom and spacing changes [Recommended]

No content is lost when text is enlarged to 200% (1.4.4). No content is clipped when users set line height to 1.5 times the font size, paragraph spacing to 2 times, letter spacing to 0.12 times, and word spacing to 0.16 times (1.4.12). Do not hard-code the height of containers that hold text.

**Source:** WCAG 2.2

### Account for forced colors mode [Recommended]

`box-shadow` has no effect under `forced-colors: active`, so boundaries expressed with shadows need a border. Make local fixes only. Do not build a separate design.

**Source:** MDN

## Common component patterns

### Prefer native `<dialog>` or a mature accessible component library for dialogs [Recommended]

Open it with `showModal()`. The browser isolates the background, constrains focus, and responds to `Esc`. Provide a visible close button, and point `aria-labelledby` at the title. Do not write `tabindex` on `<dialog>`. On open, put focus on a suitable element inside: at the start when the content is long, and on the least dangerous option for irreversible actions. On close, return focus to the triggering element, or move it to a sensible location if that element no longer exists. When writing everything by hand, add `inert` to the background and set `role="dialog"` and `aria-modal="true"`.

**Why:** Writing these behaviors by hand is error-prone.

**Source:** MDN, WAI-ARIA APG

### Implement custom components by APG patterns [Recommended]

When native elements are not enough, APG is the authority on roles, states, and keys. Multiple controls in a group take only one Tab stop, and arrow keys move within the group, implemented by moving `tabindex="0"` or with `aria-activedescendant`.

| Component | Roles and states | Keyboard |
|---|---|---|
| Disclosure | `aria-expanded` on the button | `Enter`, `Space` toggle |
| Tabs | `tablist`, `tab`, `tabpanel`, `aria-selected`, `aria-controls` | Left and right arrow keys switch, `Tab` enters the panel |
| Menu button | `aria-haspopup`, `aria-expanded`, container `menu` | `Enter`, `Space`, Down arrow open and focus the first item |
| Combobox | `combobox`, `aria-expanded`, `aria-controls`, `aria-activedescendant` | Down arrow enters the list, `Esc` closes, `Enter` selects |

Activate tabs automatically on focus only when panel content has no noticeable delay. For site navigation, prefer `<nav>`, links, and the disclosure pattern. `menu` is mainly for command sets similar to operating system menus, where `Tab` does not move between items.

**Source:** WAI-ARIA APG

### Make content shown on hover and focus controllable [Recommended]

`Esc` dismisses the tooltip content, the pointer can move onto it, and it stays visible until the user leaves (1.4.13). Use `role="tooltip"` on the tooltip, associate it with `aria-describedby` on the trigger, and keep focus on the trigger. When the content contains focusable elements, use a non-modal dialog instead.

**Source:** Understanding WCAG 2.2, WAI-ARIA APG

## Automated checks

### Run lint in CI [Recommended]

React projects enable the `recommended` config of `eslint-plugin-jsx-a11y`. For how to write axe tests, see [testing.md](testing.md).

**Source:** eslint-plugin-jsx-a11y Docs

### Check key flows by hand [Recommended]

A perfect automated score does not mean the page is usable. Complete the main flow with the keyboard, verify names and states with a screen reader, zoom the page to 200%, and check on narrow screens.

**Source:** web.dev Learn Accessibility, axe-core Docs
