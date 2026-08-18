# Recharge Widget — One-Time Purchase Link Concept

Extension of Recharge's turnkey Subscription Widget. It produces a **subscription-first** layout: selling plans render as stacked cards, and the one-time purchase option appears last as a quiet link that adds the item to the cart via AJAX.

Two files do all the work:

| File                                             | What it is                                  | Where it lives                                    |
| ------------------------------------------------ | ------------------------------------------- | ------------------------------------------------- |
| `recharge-custom-widget-styling.css`             | Layout + styling for the widget             | Merchant portal → Widget → Edit styles → Advanced |
| `recharge-widget-advanced-customizations.liquid` | Behavior: custom one-time add-to-cart logic | Theme snippet rendered on the product page        |

---

## Background: Recharge's Shadow DOM

Recharge renders its widget inside a **Shadow DOM**. This means:

- Normal CSS selectors (`.class`, `#id`) **cannot reach** the widget internals.
- Styling only works through `::part()` selectors using Recharge's documented part values (e.g. `::part(rc-plans-button)`).
- JavaScript must traverse the DOM via `event.composedPath()` to detect clicks on widget elements.

This is why `recharge-custom-widget-styling` is a long series of `::part()` rules and why `recharge-widget-advanced-customizations` inspects `part` attributes instead of classes.

---

## What `recharge-custom-widget-styling` does

All rules target `::part(rc-*)` components. High-level intent, section by section:

**1. Widget shell — remove the default boxes**

- `::part(rc-content-wrap)`: strips border, shadow, background, padding so the widget blends into the theme.
- `::part(rc-purchase-option)`, `::part(rc-purchase-option__subscription)`, `::part(rc-purchase-option__selected)`: the two purchase options render borderless/transparent — the plan cards carry all visual structure instead.

**2. Header row**

- `::part(rc-purchase-option__selector_subscription)`: makes "Subscribe & Save up to 20%" large and bold (24px).
- `::part(rc-purchase-option__badge)`: hides the floating savings badge — savings are communicated in the label text instead.
- `::part(rc-learn-more__trigger)`: restyles "Why Subscribe?" as a quiet underlined link.
- `::part(rc-benefits__list)`: repurposed as a single muted subtitle line ("Free shipping. Pause or cancel anytime."), bullet icon removed.
- `::part(rc-plans__label)`: hides the "Deliver every:" label.

**3. Plan rows → stacked cards**

- `::part(rc-plans-button-group)` / `::part(rc-plans-button-list)`: flex column layout with 12px gaps — plans stack vertically instead of side-by-side.
- `::part(rc-plans-button)`: each plan becomes a full-width card (grid: title left, discount right) with soft border, rounded corners, hover state.
- `::part(rc-plans-button__selected)`: selected card gets a white background + green border (the "Most Popular" treatment).
- `::part(rc-plans-button__interval)`: plan title styled bold (18px), left-aligned.
- `::part(rc-plans-button__discount)`: "save 10%" lighter and right-aligned.
- `::part(rc-plans-button__selected)::before`: (commented out) would render a "Most Popular" banner across the top of the selected card.
- `::part(rc-plans-button__selected)::after`: adds two checkmark benefit lines inside the selected card ("Save on every order" / "Free Shipping, Pause or Cancel Anytime").

**4. Prices (subscription header)**

- `::part(rc-purchase-option__prices)`: forces original + discounted price onto one row.
- `::part(rc-purchase-option__original-price)`: muted grey strikethrough (compare-at price).
- `::part(rc-purchase-option__discounted-price)`: large, bold, colored.

**5. One-time section ("Prefer just one?")**

- `::part(rc-purchase-option__onetime)`: centered flex column, spaced below the plans, no box.
- `::part(rc-purchase-option__price)`: muted inline price beside the link.
- `::part(rc-purchase-option__selector_onetime)`: styled as an underlined text link (bold, 14px) with a hover state that removes the underline.
- `::part(rc-purchase-option__label)`: shared with the subscription header — centers the one-time row, keeps label + price on one line (`flex-wrap: nowrap`).
- `::part(rc-purchase-option__checked-indicator)`, `::part(rc-purchase-option__input)`: hidden — selection is communicated by the green card border, not a radio/checkbox.

**Editor settings it depends on** (set in the widget editor): frequency display **List**, plan-name display, subscription/one-time labels, and strikethrough price enabled — the CSS matches that rendered structure.

---

## What `recharge-widget-advanced-customizations` does

The same custom-JS logic, packaged as a Liquid snippet so the variant ID can be injected from the product page.

```liquid
const VARIANT_ID = {{ product.selected_or_first_available_variant.id }};
```

**Behavior flow:**

1. Hooks into Recharge via the global `window.RechargeSubscriptionWidgetReady(api)` callback — runs once the widget API is ready.
2. Registers a **capture-phase** click listener on `document`. Capture (instead of bubble) is needed because the widget lives in a Shadow DOM; the listener must see the click before the widget's own handlers consume it.
3. On every click, walks `event.composedPath()` — the path crosses the shadow boundary — and checks whether any element's `part` attribute includes `rc-purchase-option__onetime`. That's how it detects "the user clicked the one-time option" despite being unable to query the shadow tree directly.
4. If it matches: `preventDefault()` + `stopPropagation()` to suppress Recharge's normal subscription add-to-cart.
5. Calls `addItemsToCart(VARIANT_ID)` → `POST /cart/add.js` with `{ items: [{ id: variantID, quantity: 1 }] }`. No `selling_plan` is sent, so Shopify treats it as a one-time purchase.
6. On success, opens the theme's cart drawer via `document.querySelector("cart-drawer")`.

**Config points / assumptions:**

- `product.selected_or_first_available_variant.id` — must be a **variant** ID (`/cart/add.js` rejects product IDs). Requires the snippet to render where `product` is available (product page/product template).
- Subscriptions are handled natively by Recharge and are left untouched. This only intercepts the one-time click event and submits the item to the cart instead.

---

## Install summary

1. Paste `recharge-custom-widget-styling` into: Merchant portal → Cross-Sell & Upsell → Subscription widget → Customize → Edit styles → Advanced CSS.
2. Add `recharge-widget-advanced-customizations` to the theme (`snippets/`) and render it on the product page (or wherever the widget appears and `product` is in scope).
3. Match the editor settings listed in the `recharge-custom-widget-styling` header comment so the styles align with the rendered structure.
