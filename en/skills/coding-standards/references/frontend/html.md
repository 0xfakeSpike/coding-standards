# HTML

This document covers HTML semantics, document structure, metadata, form elements, media, and links. For ARIA and keyboard behavior, see [accessibility.md](accessibility.md). For image dimensions and formats, see [responsive-and-assets.md](responsive-and-assets.md). For form validation, see [forms.md](forms.md).

## Semantics

### Choose elements by meaning [Recommended]

Choose elements by the meaning of the content, not by appearance. `<div>` and `<span>` have no semantics, so use them only when no suitable element exists. Leave appearance to CSS.

**Why:** Screen readers and search engines rely on element semantics. `role` only adds semantics and brings no focus or keyboard behavior.

**Example:**

Avoid:

```html
<div class="btn" onclick="save()">Save</div>
```

Prefer:

```html
<button type="button" id="save">Save</button>
```

**Tooling:** `jsx-a11y/no-static-element-interactions`, `jsx-a11y/click-events-have-key-events`.

**Source:** web.dev Learn HTML, web.dev Learn Accessibility

### Divide pages with landmark elements [Recommended]

Divide pages with `<header>`, `<nav>`, `<main>`, `<aside>`, and `<footer>`. Each page has only one `<main>`. `<section>` must have a heading, and `<article>` is for self-contained content. Distinguish multiple `<nav>` elements with `aria-label`.

**Why:** Landmarks are jump points for screen reader users. Only top-level `<header>` and `<footer>` count as landmarks. Those nested in `<article>` or `<section>` do not.

**Source:** web.dev Learn HTML

### Keep heading levels continuous [Recommended]

Use `<h1>` for the page's main heading and go down level by level without skipping. Control visual size with CSS, not heading level.

**Why:** Browsers do not compute an outline from nesting, so only the author can get levels right.

**Source:** web.dev Learn HTML

### Write full headers for data tables [Recommended]

Data tables state their purpose with `<caption>`, write `scope="col"` or `scope="row"` on `<th>`, and divide sections with `<thead>` and `<tbody>`.

**Why:** Some assistive technologies cannot work out on their own which cells a header applies to.

**Source:** MDN

## Document structure

### Complete the page skeleton [Recommended]

Write the doctype on the first line. Write `lang` on `<html>`. `<meta charset>` is the first element in `<head>`, followed by `<title>` and viewport. Foreign-language fragments in the body carry their own `lang` on their element. For `lang` values, see [i18n.md](i18n.md).

**Why:** A missing doctype triggers quirks mode. A missing `lang` makes screen readers mispronounce text, and it also violates WCAG 2.2 3.1.1 and 3.1.2.

**Example:**

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>Order details - Example Store</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body></body>
</html>
```

**Source:** web.dev Learn HTML, MDN, WCAG 2.2

## SEO metadata

### Give every page a unique title [Recommended]

`<title>` states what the page is about. Do not use vague words such as "Home", and do not stuff keywords. For pages that should be indexed, summarize the page in one or two sentences with `<meta name="description">`. Client-rendered pages also update these on route changes. Do not write `keywords`, because Google Search does not use it. For pages meant to be shared, write `og:title`, `og:description`, and `og:image`, with an absolute URL for the image.

**Why:** With identical titles, users cannot tell pages apart in search results and tabs. This is also WCAG 2.2 2.4.2.

**Source:** Google Search Central, web.dev Learn HTML, WCAG 2.2

### Declare the canonical URL [Recommended]

For pages with duplicate content, point `<link rel="canonical">` at the preferred URL, and have the canonical page point to itself. Write a complete absolute URL, and keep only one per page. Prefer outputting it in HTML rather than injecting it by script. Do not use `robots.txt` for canonicalization.

**Source:** Google Search Central

### Use `noindex` for pages that should not be indexed [Recommended]

Use `<meta name="robots" content="noindex">` or the `X-Robots-Tag` response header. These pages must not also be blocked by `robots.txt`. For pages that should be indexed, do not write `noindex` in the initial HTML and then remove it by script.

**Why:** Crawlers cannot read `noindex` on blocked pages, so the page may still appear in results. After seeing `noindex`, Google may also skip script execution.

**Source:** Google Search Central

### Use real paths for indexable pages [Recommended]

Client routes that should be indexed use the History API, not `#/path`. Internal links must be `<a>` with `href`. Add `noindex` to client-rendered error pages, and do not redirect error pages. See [routing.md](routing.md). For route implementation, see [routing.md](routing.md).

**Why:** Elements with only `onclick` are not reliable crawl entry points, and fragment routes cannot be parsed reliably.

**Source:** Google Search Central

### Declare `hreflang` reciprocally on multilingual pages [Recommended]

Each language version lists all versions, including itself, and the set stays consistent across versions. Write URLs with the full protocol and domain. Use `x-default` for the language selection page.

**Why:** When two pages do not point to each other, the annotations are ignored.

**Source:** Google Search Central

## Form elements

### Associate a `<label>` with every control [Required]

**Applies to:** production service

Use `for` matching the control's `id`, or wrap the control in `<label>`. `placeholder` and `title` cannot replace labels. Do not put other interactive elements inside `<label>`. Put related radio buttons and checkboxes in a `<fieldset>`, with `<legend>` as the group title.

**Why:** Labels provide the accessible name and enlarge the click area. Placeholders disappear after input. This is WCAG 2.2 3.3.2.

**Tooling:** `jsx-a11y/label-has-associated-control`.

**Source:** web.dev Learn HTML, WAI-ARIA APG, WCAG 2.2

### Write input types and purposes accurately [Recommended]

Match data with `type` and `inputmode`, so phones show the matching keyboard. Fields that collect user information write `autocomplete`, such as `email`, `tel`, `current-password`, `new-password`, and `one-time-code`. Do not disable autofill for these fields with `autocomplete="off"`.

**Why:** This reduces input errors and is also WCAG 2.2 1.3.5. `type="tel"` does not validate format, so add `pattern` when needed.

**Source:** web.dev Learn HTML, web.dev Learn Design, MDN, WCAG 2.2

## Media

### Write alternative text for images [Required]

**Applies to:** production service

Meaningful images use `alt` to describe their content or purpose. Do not write "image" or the file name. Images that trigger an action describe the action, such as "Next page". Decorative images write `alt=""`, and the attribute must not be omitted. `title` cannot replace `alt`.

**Why:** This is WCAG 2.2 1.1.1. When `alt` is omitted, screen readers may read out the file name.

**Tooling:** `jsx-a11y/alt-text`.

**Source:** MDN, Google Search Central, WCAG 2.2

### Provide controls and captions for audio and video [Recommended]

**Applies to:** production service

`<video>` and `<audio>` write `controls`, or provide complete custom controls. Videos with speech provide WebVTT captions with `<track>`. Use autoplay only for muted decorative background videos. Moving content that starts automatically and lasts more than 5 seconds must be pausable, stoppable, or hideable.

**Why:** Some browsers ignore `autoplay` when `muted` is not set. Content that cannot be paused violates WCAG 2.2 2.2.2.

**Source:** web.dev Learn HTML, WCAG 2.2

## Links

### Use `<a>` for navigation and `<button>` for actions [Required]

Use `<a>` with `href` when there is a target URL. Use `<button>` for all other actions. Do not write `href="#"` or `href="javascript:void(0)"`. Do not nest buttons in links or links in buttons.

**Why:** Fake links cannot be copied, opened in a new tab, or bookmarked, and their semantics are wrong.

**Tooling:** `jsx-a11y/anchor-is-valid`.

**Source:** MDN, web.dev Learn HTML

### Make link text state the destination [Recommended]

The destination must be clear out of context. Write "View product documentation", not "Click here" or "More". Image links use `alt` as the name.

**Why:** Screen reader users often browse by link list, and search engines use anchor text to understand the target page. This is WCAG 2.2 2.4.4.

**Source:** MDN, Google Search Central, WCAG 2.2

### Flag new windows and downloads, and mark outbound links by purpose [Recommended]

When a link opens a new tab or downloads a file, say so in the link text, such as "(opens in a new tab)" or "(PDF)". For `opener` security requirements, see [security.md](security.md). Use `rel="sponsored"` for paid links, `rel="ugc"` for user-submitted links, and `rel="nofollow"` for untrusted sources. Do not add `nofollow` to all outbound links.

**Source:** MDN, web.dev Learn HTML, Google Search Central
