# Requirements Document

## Introduction

The dashboard currently builds payment-row and webhook-delivery DOM nodes entirely by hand: `appendRow` (via `rowFor`) assembles `<tr>` / `<td>` elements with `document.createElement` and the `el()` helper, while `deliveryItem` does the same for `<li>` structures. Both functions are correct but verbose — every structural change requires touching several imperative lines in JavaScript, and the visual shape of the output is hard to read from the code alone.

This feature replaces that hand-built DOM with HTML `<template>` elements declared in `dashboard.html`. JavaScript clones a template's content, selects named slots by `data-*` attribute, and fills them with `textContent` only (never `innerHTML`). The API contract, the security model, the test suite, and the browser-compatibility floor are all unchanged; only the mechanism for constructing those two sets of nodes changes.

The work depends on issue #676 (which introduced the current `el()` helper pattern).

## Glossary

- **Template_Element**: An HTML `<template>` element. Its content is an inert `DocumentFragment` that the browser parses but does not render until cloned via `content.cloneNode(true)`.
- **Row_Template**: The `<template id="tmpl-payment-row">` element in `dashboard.html` that represents one `<tr>` in the payments table.
- **Delivery_Template**: The `<template id="tmpl-delivery-item">` element in `dashboard.html` that represents one `<li>` in the webhook-deliveries list.
- **Slot**: A descendant element inside a template identified by a `data-slot` attribute (e.g., `data-slot="status"`). JavaScript fills it by setting `textContent` on the cloned slot node.
- **Pill_Slot**: A Slot whose class is also set by JavaScript (e.g., `data-slot="status-pill"`) to apply the correct status colour.
- **Dashboard_JS**: `static/dashboard.js` — the main dashboard controller.
- **Dashboard_HTML**: `static/dashboard.html` — the dashboard markup, where Template_Elements are declared.
- **rowFor**: The JavaScript function in Dashboard_JS that currently builds a payment `<tr>` node imperatively. After this feature it clones Row_Template instead.
- **deliveryItem**: The JavaScript function in Dashboard_JS that currently builds a delivery `<li>` node imperatively. After this feature it clones Delivery_Template instead.

---

## Requirements

### Requirement 1: Declare a Row_Template in Dashboard_HTML

**User Story:** As a developer maintaining the dashboard, I want the structure of a payment-table row expressed as HTML markup in the template, so that I can read and change the row layout without tracing imperative `createElement` calls.

#### Acceptance Criteria

1. THE Dashboard_HTML SHALL contain a `<template id="tmpl-payment-row">` element whose content is one `<tr>` with five `<td>` children, matching the column order Status → Amount → Memo → Created → Payment ID.
2. WHEN the Row_Template is rendered in a browser, THE Row_Template SHALL declare `data-label` attributes on each `<td>` matching the values used by the CSS card layout (`"Status"`, `"Amount"`, `"Memo"`, `"Created"`, `"Payment ID"`).
3. THE Row_Template SHALL mark each fillable node with a `data-slot` attribute whose value identifies the field (e.g., `data-slot="status-pill"`, `data-slot="amount"`, `data-slot="memo"`, `data-slot="created"`, `data-slot="id"`).
4. THE Row_Template SHALL place the status pill span inside the Status `<td>` so that the cloned span can have its `className` and `textContent` set without creating a new element.
5. IF `document.getElementById("tmpl-payment-row")` returns `null` at runtime, THEN THE Dashboard_JS SHALL throw a descriptive error during initialisation rather than failing silently when the first row is appended.

---

### Requirement 2: Declare a Delivery_Template in Dashboard_HTML

**User Story:** As a developer maintaining the dashboard, I want the structure of a webhook-delivery list item expressed as HTML markup in the template, so that I can see and adjust the delivery card layout in one place.

#### Acceptance Criteria

1. THE Dashboard_HTML SHALL contain a `<template id="tmpl-delivery-item">` element whose content is one `<li class="delivery">` containing all sub-elements the current `deliveryItem` function produces: a `delivery-head` div (event name + status pill), a URL div, two meta divs (attempts, last-attempt, created), a conditional error div, a conditional pending div, and a Redeliver button.
2. THE Delivery_Template SHALL mark each fillable text node's container with a `data-slot` attribute (e.g., `data-slot="event"`, `data-slot="status-pill"`, `data-slot="url"`, `data-slot="attempts"`, `data-slot="last-attempt"`, `data-slot="created-at"`, `data-slot="redeliver-btn"`).
3. WHEN the Delivery_Template is rendered in a browser, THE Delivery_Template SHALL include a `data-slot="error-msg"` node and a `data-slot="retry-state"` node that are hidden by default (via the `hidden` attribute) so JavaScript can reveal them conditionally without creating new elements.
4. IF `document.getElementById("tmpl-delivery-item")` returns `null` at runtime, THEN THE Dashboard_JS SHALL throw a descriptive error during initialisation rather than failing silently when the first delivery is appended.

---

### Requirement 3: Clone Row_Template in rowFor

**User Story:** As a developer, I want `rowFor` to clone and fill the Row_Template rather than building nodes imperatively, so that the JavaScript function is shorter and the markup shape is visible in the HTML file.

#### Acceptance Criteria

1. WHEN `rowFor` is called with a payment object, THE Dashboard_JS SHALL clone the Row_Template via `document.getElementById("tmpl-payment-row").content.cloneNode(true)` and return the resulting `<tr>`.
2. THE Dashboard_JS SHALL fill every Slot in the cloned row by setting `textContent` on the selected node — never by setting `innerHTML`.
3. THE Dashboard_JS SHALL set the status-pill slot's `className` to the value returned by `pillClass(p.status)` and its `textContent` to `p.status`.
4. THE Dashboard_JS SHALL set `tr.tabIndex = 0`, `tr.dataset.paymentId`, `tr.className` (for the active state), and `tr.setAttribute("aria-current", "true")` on the cloned `<tr>` exactly as `rowFor` does today.
5. THE Dashboard_JS SHALL attach the `click` and `keydown` event listeners to the cloned `<tr>` exactly as `rowFor` does today, so that clicking or pressing Enter on a row still opens the detail drawer.
6. WHEN `rowFor` fills the Payment ID slot, THE Dashboard_JS SHALL pass the value through `shortId()` exactly as today.
7. WHEN `rowFor` fills the Created slot, THE Dashboard_JS SHALL pass the value through `fmtTime()` exactly as today.

---

### Requirement 4: Clone Delivery_Template in deliveryItem

**User Story:** As a developer, I want `deliveryItem` to clone and fill the Delivery_Template rather than building nodes imperatively, so that the delivery card structure is readable in HTML.

#### Acceptance Criteria

1. WHEN `deliveryItem` is called with a payment ID and delivery object, THE Dashboard_JS SHALL clone the Delivery_Template via `document.getElementById("tmpl-delivery-item").content.cloneNode(true)` and return the resulting `<li>`.
2. THE Dashboard_JS SHALL fill every Slot by setting `textContent` on the selected slot node — never by setting `innerHTML`.
3. THE Dashboard_JS SHALL set the status-pill slot's `className` to `pillClass(d.status)` and its `textContent` to `d.status`.
4. WHEN `d.status === "failed"`, THE Dashboard_JS SHALL reveal the `data-slot="error-msg"` node (remove `hidden`) and set its `textContent` to the failure message.
5. WHEN `d.status !== "delivered"`, THE Dashboard_JS SHALL reveal the `data-slot="retry-state"` node (remove `hidden`) and set its `textContent` to the retry-state message.
6. THE Dashboard_JS SHALL set the `title` attribute of the last-attempt and created-at slots to the full ISO timestamp via `fmtTime()`, exactly as `deliveryItem` does today.
7. THE Dashboard_JS SHALL attach the Redeliver button's `click` listener to the node identified by `data-slot="redeliver-btn"`, preserving the existing confirm-dialog → disable → POST → reload flow with no behavioural change.

---

### Requirement 5: No API key in URLs, logs, or console output

**User Story:** As a security-conscious operator, I want to be certain that the template-cloning code path never leaks the API key into a URL, a log line, or the browser console, so that the existing security properties are preserved end-to-end.

#### Acceptance Criteria

1. THE Dashboard_JS SHALL attach the API key exclusively as an `Authorization: Bearer` header and SHALL NOT include it in any `data-*` attribute, template slot value, `textContent` assignment, or URL parameter.
2. WHEN a template slot is filled, THE Dashboard_JS SHALL only call `textContent` or `className` or `setAttribute` with values derived from payment or delivery data fields — never from `store.get().key` or any session-storage value.
3. THE Dashboard_HTML template content SHALL NOT contain any placeholder text or `data-*` value that names or references the API key.

---

### Requirement 6: tests/dashboard_asset_tests.rs passes without modification

**User Story:** As a CI maintainer, I want the existing Rust asset test suite to continue passing after the template work lands, so that existing correctness guarantees are not regressed.

#### Acceptance Criteria

1. WHEN the Rust test suite is run after this feature is merged, THE Dashboard_JS SHALL still contain `var API_BASE = "/v1";` at exactly one location, so `dashboard_api_requests_use_canonical_v1_base` passes.
2. THE Dashboard_JS SHALL still define all helper functions enumerated in `every_dashboard_helper_the_script_calls_is_defined`: `formatAmount`, `countdown`, `relativeTime`, `explorerTx`, `readHashState`, `writeHashState`, `syncFilterUi`, `openModal`, `closeModal`, `dismissOnBackdrop`, `onDetailClosed`, `initTheme`, `applyTheme`, `effectiveTheme`, `syncThemeUi`, `focusDetail`, `trapDetailFocus`, `keepFocusInDetail`.
3. THE Dashboard_HTML SHALL retain its `<dialog id="detail" … aria-modal="true" aria-labelledby="detail-title">` structure so `detail_drawer_is_a_modal_dialog` passes.
4. THE Dashboard_HTML SHALL not introduce an `id="scrim"` element (removed in #715), so `detail_drawer_is_a_modal_dialog` continues to assert its absence.
5. WHEN `<template>` elements are added to Dashboard_HTML, THE Dashboard_HTML SHALL remain a valid HTML5 document that the `include_str!` macro can compile and the test server can serve with content type `text/html; charset=utf-8`.
