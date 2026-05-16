# Accessibility (a11y) checklist

This checklist defines what the a11y-reviewer subagent looks for. It targets WCAG 2.1 AA compliance violations and accessibility regressions in UI changes (JSX, HTML, Vue, Svelte templates).

The bar for flagging is high. Each finding must identify a **specific WCAG criterion violation** or a clear usability barrier for assistive technology users. Vague "could be more accessible" suggestions are out of scope.

## What to Flag

### Semantic HTML

- `<div>` or `<span>` used with `onClick` (or framework equivalent) where a `<button>` or `<a>` would be correct — these are not keyboard-focusable or screen-reader-announced as actionable.
- Headings that skip levels (e.g., `<h1>` followed directly by `<h3>` with no `<h2>`).
- Multiple `<h1>` elements on a page without sectioning landmarks.
- `<table>` used for layout instead of CSS grid/flex.
- Form inputs without associated `<label>` (or `aria-label`/`aria-labelledby`).
- Icon-only buttons without accessible names (no `aria-label`, no visually-hidden text).
- Lists of items not wrapped in `<ul>`/`<ol>`/`<li>`.

### ARIA usage

- `aria-label` applied to non-interactive elements (purely decorative).
- `role="button"` on an actual `<button>` (redundant or worse, removes native semantics).
- Custom `role` values not in WAI-ARIA spec.
- `aria-hidden="true"` on an interactive element (removes from screen reader but still focusable — broken).
- `tabindex="-1"` on an interactive element that should be reachable by keyboard.
- `tabindex` values greater than `0` (anti-pattern, disrupts focus order).
- Missing `aria-current` on active navigation items.
- Missing `aria-expanded` / `aria-controls` on disclosure widgets.
- Missing `aria-live` regions for dynamically updating content (e.g., toast notifications, search-as-you-type results).

### Keyboard accessibility

- Click handlers on `<div>`/`<span>` without corresponding `onKeyDown` handlers for `Enter`/`Space`.
- Modal dialogs that don't trap focus inside them.
- Modal dialogs that don't return focus to the trigger element on close.
- Custom dropdowns/comboboxes without arrow key navigation.
- Skip-to-content link missing on pages with significant top navigation.
- `outline: none` or `outline: 0` on focusable elements without a replacement focus indicator.
- Newly added interactive elements that aren't keyboard-focusable.

### Color and contrast

- Inline `color` and `background-color` combinations in the diff that fail WCAG AA contrast (4.5:1 for normal text, 3:1 for large text). Flag only when both values are visible in the diff.
- Information conveyed only by color (e.g., red text for errors with no icon or text label).
- Focus indicators with insufficient contrast against background.

### Images and media

- `<img>` without `alt` attribute (must be present; can be empty `alt=""` for decorative).
- `<img>` with `alt` text that duplicates surrounding text or says "image of".
- Background images conveying meaningful content without an accessible alternative.
- Video without captions or transcript reference.
- Auto-playing audio or video without user control.

### Forms

- Required fields not indicated to screen readers (visual asterisk only).
- Error messages not associated with their fields via `aria-describedby`.
- Inline validation errors that fire on every keystroke without a debounce or accessible announcement.
- Submit buttons that change to a loading state without `aria-busy` or text update.

### Reading order and layout

- CSS `order` property that changes visual order without matching DOM order (screen readers follow DOM).
- `display: none` toggled to show content that should announce — needs `aria-live` or focus management.
- Sticky elements that obscure content keyboard users may scroll to.

## What NOT to Flag

- Visual design choices (typography, spacing, animations) unless they have specific a11y impact.
- Performance suggestions unrelated to a11y.
- "Add ARIA to everything" — only flag missing ARIA where semantics genuinely require it.
- Suggesting adding `alt` text on `<img>` that already has one (even if the text could be better).
- Adding `tabindex="0"` to `<button>` or `<a>` (redundant — they're already focusable).
- Strict color contrast checks when colors come from theme variables that aren't visible in the diff.
- Asking for `role="main"` on `<main>` (redundant).

## Common false positives to actively avoid

- Empty `alt=""` on decorative images — this is correct, not a missing alt.
- `aria-hidden="true"` on visual decoration (correct usage).
- `<button type="button">` for buttons that don't submit forms — correct, prevents accidental submission.
- Skip-link patterns that look unfamiliar — many implementations differ but all are valid.
- React fragments (`<>`) without ARIA roles — fragments are transparent to the DOM.

## Severity calibration examples

- **critical**: form submit button with no accessible name; modal trap focus broken so keyboard users get stuck; click handler on `<div>` with no keyboard handler in a critical user flow.
- **warning**: missing `aria-label` on icon-only delete button; heading levels skipped on a content page; inline contrast 3.5:1 on body text.
- **suggestion**: add `aria-current="page"` to active nav item; consider `aria-live="polite"` on search results; add a visually-hidden label for screen readers.
