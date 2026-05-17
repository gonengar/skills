# Design States — Suggestions

Inference rules used by [`DESIGN-STATES.md`](./DESIGN-STATES.md). Read the component's React file and apply these rules to decide which states to declare.

**Not authoritative for shipped behavior** — what matters at runtime is whatever ends up in `<ComponentName>.extension.ts`. These rules only guide which states fit a given component.

---

## Public native pseudo-class values

The public schema supports exactly four `pseudoClass` values, mapped to standard CSS pseudo-classes:

| `pseudoClass` | CSS pseudo-class | MDN reference |
|---|---|---|
| `'hover'` | [`:hover`](https://developer.mozilla.org/en-US/docs/Web/CSS/:hover) | Pointer is over the element |
| `'focus'` | [`:focus`](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus) | Element has keyboard focus |
| `'disabled'` | [`:disabled`](https://developer.mozilla.org/en-US/docs/Web/CSS/:disabled) | Form element is disabled |
| `'invalid'` | [`:invalid`](https://developer.mozilla.org/en-US/docs/Web/CSS/:invalid) | Form element fails validation |

Anything outside this set is a **custom state**: declare a `className` and omit `pseudoClass`. The Editor applies the class when the state is selected; activation on the live site is up to your component's CSS / class toggling.

---

## HTML element / ARIA role → native states that fire

Before suggesting native states, look at the React element the component actually renders. A state only fires on the live site if the rendered DOM supports it.

| Rendered element | `:hover` | `:focus` | `:disabled` | `:invalid` |
|---|:---:|:---:|:---:|:---:|
| `<button>` | ✓ | ✓ | ✓ | — |
| `<a href="…">` | ✓ | ✓ | — | — |
| `<a>` (no `href`) | ✓ | — | — | — |
| `<input type="text" \| "email" \| "url" \| "tel" \| "number" \| "password" \| "search">` | ✓ | ✓ | ✓ | ✓ (with `required` / `pattern` / `min` / `max` / `type=email\|url`) |
| `<input type="checkbox" \| "radio">` | ✓ | ✓ | ✓ | ✓ (with `required`) |
| `<input type="submit" \| "button" \| "reset">` | ✓ | ✓ | ✓ | — |
| `<textarea>` | ✓ | ✓ | ✓ | ✓ (with `required` / `minlength` / `maxlength`) |
| `<select>` | ✓ | ✓ | ✓ | ✓ (with `required`) |
| `<fieldset>` | ✓ | — | ✓ | — |
| `<summary>` | ✓ | ✓ | — | — |
| `<details>` | ✓ | — | — | — |
| `<dialog>` | ✓ | — | — | — |
| `<label>` | ✓ | — | — | — |
| `<div role="button" tabIndex={0}>` | ✓ | ✓ | — (no `:disabled`; use a custom `disabled` state with `aria-disabled`) | — |
| `<div role="link" tabIndex={0}>` | ✓ | ✓ | — | — |
| `<div>` / `<span>` (no role, no `tabIndex`) | ✓ (rarely meaningful) | — | — | — |

Sources of truth:
- [`:disabled`](https://developer.mozilla.org/en-US/docs/Web/CSS/:disabled) — only matches `button`, `input`, `select`, `textarea`, `optgroup`, `option`, `fieldset`, and form-associated custom elements.
- [`:invalid`](https://developer.mozilla.org/en-US/docs/Web/CSS/:invalid) — matches form elements (and the form itself) when their content fails the constraints declared in HTML attributes.
- [`:focus`](https://developer.mozilla.org/en-US/docs/Web/CSS/:focus) — matches anything that is focusable (natively focusable or has `tabindex`).
- [`:hover`](https://developer.mozilla.org/en-US/docs/Web/CSS/:hover) — matches any element under the pointer; only *visually meaningful* when the element is interactive.

**Implications for suggestions:**

- Don't suggest `disabled` for a `<div role="button">` — `:disabled` won't match. Either (a) refactor to a real `<button>`, or (b) declare a custom `disabled` state (no `pseudoClass`, just a `className`) and have the component toggle that class based on its `isDisabled` prop.
- Don't suggest `focus` for an element that isn't focusable. Check for `tabIndex={0}` (or a positive `tabindex`) on non-native-focusable elements.
- Don't suggest `invalid` unless the React file actually emits validation attributes (`required`, `pattern`, `min`, `max`, `minLength`, `maxLength`, `type="email"`, `type="url"`) or sets validity programmatically.

---

## Role → suggested states

| Role | Suggested states |
|---|---|
| Form control with `disabled` semantics (`<button>`, `<input type="submit">`, etc.) | `hover`, `focus`, `disabled` |
| Form input with validation (`<input type="email">`, `<input required>`, `<textarea required>`, etc.) | `hover`, `focus`, `disabled`, `invalid` |
| Clickable interactive control with no disabled affordance (`<a href>`, `<summary>`, `<div role="button">`) | `hover`, `focus` |
| Self-contained interactive surface (handles its own UX inside; e.g. an embedded player) | no states on root |
| Decorative / pure-display (`<img>` block, `<figure>`, hero panel without interaction) | no states |
| Per-child-link wrapper (interactive children inside; root is just a layout shell) | no states on root |

If the component has a discriminating prop (`variant`, `featured`, `selected`, `active`, `expanded`) that controls a meaningful visual variant, add a matching **custom** state (no `pseudoClass`).

---

## Per-state relevance

Cross-reference with the element table above before suggesting any native state.

| State | Suggest when… | Don't suggest when… |
|---|---|---|
| `hover` | the component reacts visually to mouse-over (button feedback, hoverable card, menu item highlight). | the component is decorative, a host surface, or interactivity is delegated to children. |
| `focus` | the component is keyboard-targetable: rendered as a natively-focusable element (`<button>`, `<a href>`, `<input>`, `<select>`, `<textarea>`, `<summary>`) or has `tabIndex={0}`/positive. | the rendered element isn't focusable — `:focus` will never match on the live site. |
| `disabled` | the rendered element is form-associated (`<button>`, `<input>`, `<select>`, `<textarea>`, `<fieldset>`) **and** the component has an `isDisabled` / `disabled` prop. | the rendered element isn't form-associated (no `:disabled` match) — use a **custom** `disabled` state instead. |
| `invalid` | the rendered element is a form input/select/textarea with HTML validation attributes (`required`, `pattern`, `min`, `max`, `minlength`, `maxlength`, or a constrained `type`). | no validation attributes are present and the component doesn't set custom validity via JS. |

Event-handler props (`onMouseEnter`, `onFocus`, `onClick`, etc.) are **not** state signals — they expose event hooks to the site owner. Don't suggest a state just because such a handler exists; the question is whether the component has a meaningful *visual* variant for that state.

---

## Inner-element heuristics

Most components carry per-child states more than root states. When walking `editorElement.elements`, run the same checks (HTML element / role → states) against the inner element's rendered DOM, then layer the role suggestion:

- If the inner element renders a list of interactive sub-items (links, cards, list items) — typically `<a>` / `<button>` / `<li>` with an interactive child — suggest **`hover`, `selected`** on the item (`selected` is a custom state; declare it with a `className`).
- If the inner element is an action button (close, navigation, expand) rendered as `<button>` — suggest **`hover`** (and `disabled` if the component has a corresponding prop **and** the rendered element supports `:disabled`).
- If the inner element is a `refElement` — **skip silently**. `refElement` is not yet supported by the public schema.
- If the inner element is structural (a container that only positions children) — suggest **no states**.