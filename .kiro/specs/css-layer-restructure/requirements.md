# Requirements Document

## Introduction

This feature restructures `static/dashboard.css` by organising all existing
rules into named CSS `@layer` blocks. The change is a pure refactor: no
selector, property value, specificity outcome, or visual result changes. The
existing `tests/dashboard_asset_tests.rs` suite must continue to pass without
modification, and the stylesheet must render identically in current Chrome,
Firefox, and Safari at both desktop and mobile widths.

No API key or credential may ever appear in a URL, a log line, or a console
output — a constraint already enforced by the rest of the codebase (issue #685)
and reaffirmed here because the stylesheet is served over HTTP and must not
inadvertently surface sensitive data in any server log or DevTools network
panel.

## Glossary

- **Stylesheet**: `static/dashboard.css`, the single self-contained CSS file
  served as `/dashboard/app.css`.
- **Layer**: A named cascade layer declared with the CSS `@layer` at-rule.
- **Layer order declaration**: The leading `@layer name1, name2, …;` statement
  that fixes cascade priority before any rules are written.
- **Token block**: One of the four `:root` / `html[data-theme]` rule-sets that
  define CSS custom properties for light and dark palettes.
- **Asset test suite**: `tests/dashboard_asset_tests.rs`, the Rust integration
  tests that inspect the compiled-in CSS text and serve it through the real HTTP
  router.
- **Browser matrix**: Current stable releases of Chrome, Firefox, and Safari,
  tested at a desktop viewport (≥ 1024 px wide) and a mobile viewport
  (≤ 720 px wide).

---

## Requirements

### Requirement 1: Layer order declaration

**User Story:** As a browser, I need to know the intended cascade priority of
every layer before any rules are parsed, so that a rule written later in the
file does not silently outrank an earlier one simply because layers were
encountered in a different order.

#### Acceptance Criteria

1. THE Stylesheet SHALL begin with a single `@layer` statement that names all
   layers in priority order before any rule-bearing `@layer` block appears.
2. WHEN the Stylesheet is parsed, THE Browser SHALL resolve all inter-layer
   specificity conflicts using the order established by that leading declaration,
   regardless of the physical order the blocks appear in the file.
3. THE Stylesheet SHALL NOT declare any layer name inside a rule-bearing
   `@layer` block that was not listed in the leading declaration.

---

### Requirement 2: Token layer

**User Story:** As a developer, I want all CSS custom-property (token)
definitions to live in one named layer, so that theme tokens are easy to locate
and cannot accidentally be overridden by component rules.

#### Acceptance Criteria

1. THE Stylesheet SHALL place every `:root`, `:root:not([data-theme])`,
   `html[data-theme="light"]`, and `html[data-theme="dark"]` rule-set inside a
   single layer named `tokens`.
2. THE Stylesheet SHALL preserve the exact custom-property names, values, and
   `color-scheme` declarations from the original token blocks inside the
   `tokens` layer.
3. WHEN the `tokens` layer is parsed, THE Browser SHALL apply the four theme
   selectors with the same computed specificity relationships as in the
   pre-refactor stylesheet.

---

### Requirement 3: Reset layer

**User Story:** As a developer, I want global reset rules (`*`, `[hidden]`,
`body`, heading elements) isolated in one layer so their low-specificity
defaults cannot compete with component rules.

#### Acceptance Criteria

1. THE Stylesheet SHALL place the `* { box-sizing }`, `[hidden]`, `.visually-hidden`,
   `body`, `h1`/`h2`/`h3`, `.muted`, `.small`, `.mono`, and `.card` rules
   inside a single layer named `base`.
2. THE Stylesheet SHALL preserve every property value in those rules unchanged
   inside the `base` layer.

---

### Requirement 4: Focus layer

**User Story:** As a developer, I want all focus-ring and keyboard-navigation
rules in one auditable layer, so the WCAG 2.4.7 compliance rules cannot be
accidentally overridden by a later component rule.

#### Acceptance Criteria

1. THE Stylesheet SHALL place the `:focus-visible`, `.payments tbody
   tr:focus-visible`, and `.search input:focus-visible` rules — as well as
   `.row-active` — inside a single layer named `focus`.
2. THE `focus` layer SHALL preserve the `outline-offset: 2px` property so the
   contrast-audit reasoning in `tests/dashboard_asset_tests.rs` continues to
   hold.
3. THE `focus` layer SHALL preserve the `.payments tbody tr:focus-visible`
   `box-shadow` override that replaces `outline` for table rows.

---

### Requirement 5: Components layer

**User Story:** As a developer, I want all component and layout rules in one
layer so they can be read and edited together without affecting global tokens or
reset defaults.

#### Acceptance Criteria

1. THE Stylesheet SHALL place all remaining rules — form controls, buttons,
   top bar, toolbar, table, detail panel, skeleton, help overlay, pills, chips,
   empty state, error state, delivery list — inside a single layer named
   `components`.
2. THE Stylesheet SHALL preserve every selector, property name, property value,
   and media-query condition from the original component rules inside the
   `components` layer.
3. THE `components` layer SHALL contain the `.detail::backdrop` rule so the
   test assertion `DASHBOARD_CSS.contains(".detail::backdrop")` continues to
   pass.

---

### Requirement 6: No visual regression

**User Story:** As an operator, I want the refactored stylesheet to render every
page state identically to the original, so that no user ever sees a layout or
colour change from this work.

#### Acceptance Criteria

1. THE Stylesheet SHALL produce the same computed styles for every selector
   present in the original file when loaded in the Browser matrix.
2. WHEN the Stylesheet is loaded in a viewport ≤ 720 px wide, THE Browser SHALL
   apply the mobile overrides (stacked table, skeleton stack) identically to the
   original.
3. THE Stylesheet SHALL preserve the `.detail::backdrop`, `html[data-theme="light"]`,
   `html[data-theme="dark"]`, `:root:not([data-theme])`, `:focus-visible`, and
   `color-scheme` declarations that `tests/dashboard_asset_tests.rs` asserts on
   verbatim.

---

### Requirement 7: Asset-test compliance

**User Story:** As a CI pipeline, I need `tests/dashboard_asset_tests.rs` to
pass without modification after the refactor, so that no existing contract
(content-type, CSP, security headers, substring assertions) is broken.

#### Acceptance Criteria

1. WHEN the test suite compiles `DASHBOARD_CSS` via `include_str!`, THE
   Stylesheet SHALL contain every literal string asserted by
   `tests/dashboard_asset_tests.rs`, including `.detail::backdrop`,
   `:root:not([data-theme])`, `html[data-theme="light"]`,
   `html[data-theme="dark"]`, `color-scheme: light dark;`,
   `color-scheme: light;`, `color-scheme: dark;`, `:focus-visible {`,
   `outline-offset: 2px;`, `border: 1px solid var(--control-border);`,
   `input[type="date"]`, `select`, `--control-border:` (× 4), and
   `--focus:` (× 4).
2. THE Stylesheet SHALL be served byte-for-byte by the HTTP router at
   `/dashboard/app.css` with `Content-Type: text/css; charset=utf-8`, the full
   `Content-Security-Policy`, `X-Content-Type-Options: nosniff`,
   `Referrer-Policy: no-referrer`, and `Cache-Control: no-store`.

---

### Requirement 8: No credential exposure

**User Story:** As a security-conscious operator (issue #685), I want to be
certain that restructuring the stylesheet introduces no path by which an API
key could appear in a URL, log line, or console output.

#### Acceptance Criteria

1. THE Stylesheet SHALL NOT reference any URL that includes query parameters or
   fragment identifiers.
2. THE Stylesheet SHALL NOT contain any string that matches the pattern of a
   Stellar API key or bearer token.
3. WHEN the HTTP server logs a request for `/dashboard/app.css`, THE Server
   SHALL log only the request path and response status — no credential value
   SHALL appear in the log entry.
