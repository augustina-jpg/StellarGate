# Design Document: template-dom-helpers

## Overview

The dashboard currently constructs payment table rows (`rowFor`) and webhook delivery list items (`deliveryItem`) entirely through imperative `document.createElement` calls. While correct, this approach buries the visual structure inside JavaScript, making layout changes harder to follow and review.

This feature replaces both construction functions with an HTML `<template>`-based approach. Two `<template>` elements are declared in `dashboard.html`; JavaScript clones their content and fills named slots by setting `textContent` (never `innerHTML`). The external API contract, security model, browser-compatibility floor, and the complete Rust test suite remain unchanged.

The change is a pure refactor: identical rendered output, smaller JS, layout visible in HTML.

---

## Architecture

The design follows the native [HTML `<template>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template) pattern, available in all evergreen browsers and IE11+ (via polyfill, though the project already requires modern browsers).

```
dashboard.html                          dashboard.js
─────────────────────────────────       ──────────────────────────────────────
<template id="tmpl-payment-row">        function rowFor(p, isActive):
  <tr>                                    tmpl = getElementById("tmpl-payment-row")
    <td data-slot="status-pill">          frag = tmpl.content.cloneNode(true)
    <td data-slot="amount">               tr   = frag.querySelector("tr")
    <td data-slot="memo">                 // fill slots via textContent
    <td data-slot="created">              // set tr attributes & listeners
    <td data-slot="id">                   return tr
  </tr>
</template>

<template id="tmpl-delivery-item">      function deliveryItem(paymentId, d):
  <li class="delivery">                   tmpl = getElementById("tmpl-delivery-item")
    <div data-slot="event">               frag = tmpl.content.cloneNode(true)
    <span data-slot="status-pill">        li   = frag.querySelector("li")
    <div data-slot="url">                 // fill slots via textContent
    ...                                   // reveal conditional nodes
    <div data-slot="error-msg" hidden>    // attach redeliver listener
    <div data-slot="retry-state" hidden>  return li
    <button data-slot="redeliver-btn">
  </li>
</template>
```

The `el()` helper and `labelledCell()` helper are retained for use elsewhere in the file (skeleton rows, summary cards, detail panel fields). Only the two target functions change.

### Initialization guard

Both templates are looked up once during initialization, immediately after the DOM is ready. If either `getElementById` call returns `null`, the script throws a descriptive error rather than deferring the failure to the first row render. A missing template is a deployment error (the wrong `dashboard.html` was served alongside `dashboard.js`), so fast failure is the correct behavior.

```
var rowTmpl      = getElementById("tmpl-payment-row")   || throwMissing("tmpl-payment-row");
var deliveryTmpl = getElementById("tmpl-delivery-item") || throwMissing("tmpl-delivery-item");
```

---

## Components and Interfaces

### Template elements (dashboard.html)

#### `<template id="tmpl-payment-row">`

Structure mirrors the current imperative output exactly:

```html
<template id="tmpl-payment-row">
  <tr>
    <td data-label="Status">
      <span data-slot="status-pill"></span>
    </td>
    <td data-label="Amount"   data-slot="amount"></td>
    <td data-label="Memo"     data-slot="memo"   class="mono"></td>
    <td data-label="Created"  data-slot="created"></td>
    <td data-label="Payment ID" data-slot="id"   class="mono"></td>
  </tr>
</template>
```

Key design decisions:
- `data-label` values match the exact strings the CSS card layout uses (`"Status"`, `"Amount"`, `"Memo"`, `"Created"`, `"Payment ID"`).
- The status `<span>` lives inside its `<td>` in the template so JavaScript sets `className` and `textContent` on the pre-existing node without `createElement`.
- `class="mono"` is baked into the template for Memo and Payment ID, matching the current `labelledCell("Memo", "mono", …)` call.

#### `<template id="tmpl-delivery-item">`

Structure mirrors the current `deliveryItem` output:

```html
<template id="tmpl-delivery-item">
  <li class="delivery">
    <div class="delivery-head">
      <strong data-slot="event"></strong>
      <span   data-slot="status-pill"></span>
    </div>
    <div class="mono"          data-slot="url"></div>
    <div class="delivery-meta" data-slot="attempts"></div>
    <div class="delivery-meta" data-slot="last-attempt"></div>
    <div class="delivery-meta" data-slot="created-at"></div>
    <div class="error"         data-slot="error-msg"   hidden></div>
    <div class="delivery-meta" data-slot="retry-state" hidden></div>
    <button class="ghost"      data-slot="redeliver-btn" type="button">Redeliver</button>
  </li>
</template>
```

Key design decisions:
- `hidden` is set on `error-msg` and `retry-state` at declaration time; JavaScript removes the attribute conditionally.
- The Redeliver button text (`"Redeliver"`) is baked in; JavaScript only replaces it mid-flight (`"Sending…"`) and restores it on failure, exactly as today.
- `data-slot` values are flat strings rather than hierarchical paths; the `querySelector` call at fill time uses `[data-slot="…"]` attribute selectors.

### Helper: `fillSlots(root, slots)`

A small private helper takes a cloned fragment root and a plain object mapping slot names to values:

```javascript
function fillSlots(root, slots) {
  Object.keys(slots).forEach(function(name) {
    var node = root.querySelector('[data-slot="' + name + '"]');
    if (node) node.textContent = String(slots[name] === null || slots[name] === undefined ? "" : slots[name]);
  });
}
```

This is intentionally minimal. `className` assignments (for pills) and `hidden` toggling are done explicitly in the calling function because they are not text fills.

### `rowFor(p, isActive)` — revised

```
1. Clone Row_Template content (cloneNode(true))
2. Grab the <tr> from the fragment
3. Set tr.tabIndex, tr.dataset.paymentId, tr.className, tr.setAttribute("aria-current")
4. Set status-pill: className = pillClass(p.status), textContent = p.status
5. fillSlots(tr, { amount, memo, created, id }) using fmtTime / shortId
6. Attach click and keydown listeners (identical to today)
7. Return tr
```

No `document.createElement` calls inside `rowFor` after the change.

### `deliveryItem(paymentId, d)` — revised

```
1. Clone Delivery_Template content (cloneNode(true))
2. Grab the <li> from the fragment
3. Set status-pill: className = pillClass(d.status), textContent = d.status
4. fillSlots(li, { event, url, attempts, last-attempt, created-at })
5. Set title on last-attempt and created-at slots via fmtTime (setAttribute)
6. Conditionally show error-msg (if status === "failed")
7. Conditionally show retry-state (if status !== "delivered")
8. Wire Redeliver button click (identical flow to today)
9. Return li
```

No `el()` calls inside `deliveryItem` after the change.

---

## Data Models

No new data models are introduced. The feature operates on the existing API response shapes:

### Payment object (relevant fields)
```typescript
{
  id:         string,   // UUID
  status:     string,   // "pending" | "completed" | "expired" | "underpaid"
  amount:     string,   // decimal string e.g. "10.00"
  asset:      string,   // e.g. "USDC"
  memo:       string,   // merchant-controlled
  created_at: string,   // ISO 8601
}
```

### Delivery object (relevant fields)
```typescript
{
  id:              string,
  event:           string,   // merchant-controlled webhook event name
  status:          string,   // "delivered" | "failed" | "pending"
  url:             string,   // merchant-controlled webhook endpoint
  attempts:        number,
  manual_attempts: number,
  last_attempt:    string,   // ISO 8601
  created_at:      string,   // ISO 8601
}
```

Both `memo` and `event` are merchant-controlled text and are therefore the primary XSS-concern fields. Filling them via `textContent` is the existing invariant; this feature preserves it.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Template slots never interpret values as HTML

*For any* payment or delivery object whose text fields (memo, event name, URL, status, etc.) contain HTML special characters (`<`, `>`, `&`, `"`, `'`) or a complete script tag, the textContent of every filled slot in the rendered row or delivery item SHALL equal the literal input string, with no HTML parsing or execution occurring.

**Validates: Requirements 3.2, 4.2**

---

### Property 2: Status pill reflects pillClass for any status value

*For any* payment or delivery object with any status string, the element with `data-slot="status-pill"` in the rendered row or delivery item SHALL have `className === pillClass(status)` and `textContent === status`.

**Validates: Requirements 3.3, 4.3**

---

### Property 3: Slot formatters are consistently applied

*For any* payment object, the `data-slot="id"` cell textContent SHALL equal `shortId(p.id)`, the `data-slot="created"` cell textContent SHALL equal `fmtTime(p.created_at)`, and for any delivery object the `title` attribute of `data-slot="last-attempt"` SHALL equal `fmtTime(d.last_attempt)` and the `title` attribute of `data-slot="created-at"` SHALL equal `fmtTime(d.created_at)`.

**Validates: Requirements 3.6, 3.7, 4.6**

---

### Property 4: Retry-state node visibility matches delivery status

*For any* delivery object, the `data-slot="retry-state"` node SHALL be visible (not hidden) if and only if `d.status !== "delivered"`.

**Validates: Requirements 4.5**

---

### Property 5: API key never appears in rendered DOM nodes

*For any* payment or delivery rendered when a known API key string is held in the session store, scanning every text node and attribute value in the rendered row or delivery item DOM SHALL yield zero occurrences of that key string.

**Validates: Requirements 5.1, 5.2**

---

## Error Handling

### Missing template elements

If either template element is absent from the DOM at initialization time (e.g., stale HTML cached while new JS is deployed), the script throws immediately:

```javascript
function throwMissing(id) {
  throw new Error(
    "StellarGate: required template element #" + id + " not found. " +
    "Ensure dashboard.html is up to date."
  );
}
```

This surfaces a clear, actionable error in the browser console rather than a cryptic `TypeError: Cannot read properties of null` when the first row is rendered.

### Null / undefined slot values

`fillSlots` coerces `null` and `undefined` to an empty string before assigning to `textContent`, matching the existing behavior of `el()` which guards `if (text !== undefined && text !== null)`. No slot will display the literal string `"null"`.

### Conditional nodes

`error-msg` and `retry-state` remain hidden (via the `hidden` HTML attribute baked into the template) unless the delivery status explicitly calls for them. There is no "accidental reveal" path.

---

## Testing Strategy

### Existing Rust tests (no changes required)

`tests/dashboard_asset_tests.rs` includes all static asset checks. After this feature:
- `dashboard_api_requests_use_canonical_v1_base` — still passes; `var API_BASE = "/v1";` is untouched.
- `every_dashboard_helper_the_script_calls_is_defined` — still passes; no helpers are removed.
- `detail_drawer_is_a_modal_dialog` — still passes; the `<dialog>` structure is untouched.
- `dashboard_assets_keep_content_type_and_csp` — still passes; no new asset routes are introduced.

New Rust tests to add to `dashboard_asset_tests.rs`:
- `payment_row_template_exists_with_correct_slots` — asserts `tmpl-payment-row` is present in `DASHBOARD_HTML`, has five `<td>` children (after stripping comments), and each has the expected `data-label` and `data-slot` values.
- `delivery_item_template_exists_with_correct_slots` — asserts `tmpl-delivery-item` is present, has `<li class="delivery">` as its root, and all required `data-slot` values are present, including `error-msg` and `retry-state` with the `hidden` attribute.

### JavaScript unit tests

The existing JS test suite lives under `static/tests/` and runs with `node --test`. Because `rowFor` and `deliveryItem` are closures inside an IIFE in `dashboard.js`, they are not directly importable. The property-based tests below target the pure helper `fillSlots` and the template-slot logic by extracting them into a testable form (or by testing via JSDOM in the existing test runner).

**Property-based testing library**: [`fast-check`](https://fast-check.dev/) — already the standard for JS property testing, zero-dependency in test code.

**Minimum iterations**: 100 per property test.

#### Unit tests (example-based)

- `rowFor returns a <tr> for a minimal payment` — smoke test that the function returns the right element type.
- `deliveryItem returns a <li> for a minimal delivery` — smoke test.
- `rowFor with active=true sets row-active class and aria-current` — specific example.
- `deliveryItem with status="failed" reveals error-msg and hides retry-state is false` — specific example for 4.4.
- `deliveryItem with status="delivered" keeps retry-state hidden` — specific example for 4.5 boundary.
- Missing template throws a descriptive error — example for requirements 1.5, 2.4.

#### Property-based tests

**Test tag format:** `Feature: template-dom-helpers, Property {N}: {short title}`

**Property 1 — Template slots never interpret values as HTML**
```javascript
// Feature: template-dom-helpers, Property 1: textContent never interprets HTML
fc.assert(fc.property(
  fc.record({
    id: fc.uuid(),
    status: fc.constantFrom("pending","completed","expired","underpaid"),
    amount: fc.string(), asset: fc.string(),
    memo: fc.string(),    // may contain < > & etc.
    created_at: fc.string(),
  }),
  fc.boolean(),
  function(payment, isActive) {
    var tr = rowFor(payment, isActive);
    // No slot's textContent should have been parsed as markup
    var memoCell = tr.querySelector('[data-slot="memo"]');
    return memoCell.textContent === (payment.memo || "");
  }
), { numRuns: 100 });
```
The same shape is applied to `deliveryItem` with delivery fields.

**Property 2 — Status pill reflects pillClass for any status**
```javascript
// Feature: template-dom-helpers, Property 2: pill className matches pillClass
fc.assert(fc.property(
  fc.record({ ...paymentArb, status: fc.string() }),
  function(payment) {
    var tr = rowFor(payment, false);
    var pill = tr.querySelector('[data-slot="status-pill"]');
    return pill.className === pillClass(payment.status)
        && pill.textContent === payment.status;
  }
), { numRuns: 100 });
```

**Property 3 — Slot formatters consistently applied**
```javascript
// Feature: template-dom-helpers, Property 3: formatters applied to slots
fc.assert(fc.property(paymentArb, function(payment) {
  var tr = rowFor(payment, false);
  var idSlot      = tr.querySelector('[data-slot="id"]');
  var createdSlot = tr.querySelector('[data-slot="created"]');
  return idSlot.textContent      === shortId(payment.id)
      && createdSlot.textContent === fmtTime(payment.created_at);
}), { numRuns: 100 });
```

**Property 4 — Retry-state visibility matches delivery status**
```javascript
// Feature: template-dom-helpers, Property 4: retry-state visible iff not delivered
fc.assert(fc.property(deliveryArb, function(delivery) {
  var li = deliveryItem("any-payment-id", delivery);
  var retryNode = li.querySelector('[data-slot="retry-state"]');
  var expectVisible = delivery.status !== "delivered";
  return retryNode.hidden === !expectVisible;
}), { numRuns: 100 });
```

**Property 5 — API key never appears in rendered DOM**
```javascript
// Feature: template-dom-helpers, Property 5: API key never leaks into DOM
fc.assert(fc.property(fc.string({ minLength: 8 }), paymentArb, function(apiKey, payment) {
  // Inject a known key into the store mock
  mockStore.key = apiKey;
  var tr = rowFor(payment, false);
  // Walk every text node and attribute in the rendered row
  return !domContains(tr, apiKey);
}), { numRuns: 100 });
```

### Integration / E2E

The existing Playwright tests in `e2e/tests/dashboard.spec.js` exercise the full payment table render path. After this feature lands, those tests verify end-to-end that:
- Rows appear with correct text content.
- Clicking a row opens the detail drawer.
- Webhook deliveries appear in the panel.

No new Playwright tests are required; the existing suite provides regression coverage.
