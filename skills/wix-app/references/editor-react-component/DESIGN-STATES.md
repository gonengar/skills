# Design States

Step 6 of [`EDITOR_REACT_COMPONENT.md`](../EDITOR_REACT_COMPONENT.md) — add design states to an Editor React Component so Wix users can style each state (`hover`, `focus`, `disabled`, `invalid`, plus custom keys) independently in the design panel.

States are declared on `editorElement.states` (root) or per-element `inlineElement.states` (inner elements) via a manifest override in `<ComponentName>.extension.ts`. The Editor applies the configured `className` to the matched element when the state is active; native pseudo-class states also fire at runtime via the matching `pseudoClass`. The platform owns the visual rules — **do not author state CSS** (`:hover`, `:focus`, `:disabled`, `[data-state]`) in component CSS, per [`CSS-GUIDELINES.md`](./CSS-GUIDELINES.md).

**Authoritative reference:** [States — Wix Build Apps Docs](https://dev.wix.com/docs/build-apps/develop-your-app/extensions/site-extensions/editor-react-components/manifest-reference/editor-element/states).

**Related local docs:**
- Inference rules → [`DESIGN-STATES-SUGGESTIONS.md`](./DESIGN-STATES-SUGGESTIONS.md)
- Why state CSS is forbidden in component files → [`CSS-GUIDELINES.md`](./CSS-GUIDELINES.md) ("Don't author state styles" section)
- General override mechanics → [`COMPONENT-CONFIGURATION.md`](./COMPONENT-CONFIGURATION.md)

---

## Convention

Every state entry follows this canonical shape:

```ts
states: {
  hover: {
    displayName: 'Hover',                    // capitalized key with spaces on word boundaries
    className: 'elemHover',                  // 'elem<PascalKey>' — `hover` → `elemHover`, `in-progress` → `elemInProgress`
    pseudoClass: 'hover',                    // only for native keys; omitted for custom
    props: { isDisabled: true },             // only for `disabled`, only if an inverted prop is detected
  },
}
```

| Field | Type | Notes |
|---|---|---|
| `displayName` | `string` (≤100, translatable) | Label in the state selector. Capitalized key, spaces on word boundaries (`in-progress` → `In Progress`). |
| `className` | `string` (≤100) | **Standardize on `elem<PascalKey>`** (`hover` → `elemHover`, `selected` → `elemSelected`, `in-progress` → `elemInProgress`). Required for both native and custom states. |
| `pseudoClass` | `'hover' \| 'focus' \| 'disabled' \| 'invalid'` | **Native keys only.** Omit for custom states. String literal — no enum import. |
| `props` | `Record<string, Value>` | Editor-stage preview props (not on live site). Used here only for `disabled` with an inverted-prop detected on the component. |
| `displayFilters` | `DisplayFilters` | Out of scope for this step; see the [States docs](https://dev.wix.com/docs/build-apps/develop-your-app/extensions/site-extensions/editor-react-components/manifest-reference/editor-element/states). |

Native keys (`hover` / `focus` / `disabled` / `invalid`) get a `pseudoClass`. Custom keys (`selected`, `featured`, `loading`, anything else) omit `pseudoClass`. Both still need a `className` of the standard `elem<PascalKey>` form.

**Convention-drift detection** — when an existing state entry uses an alternate shape (e.g. `className: 'hover'` without the `elem` prefix), surface it under "Convention drift" in the final summary. The collision rule still applies (this step never modifies existing entries) — normalization is the author's call.

TypeScript types for the manifest come from `@wix/react-component-schema`, already installed by the Editor React Component boilerplate (see the dependency-check script in [`EDITOR_REACT_COMPONENT.md`](../EDITOR_REACT_COMPONENT.md)).

---

## Where to declare

Override `editorElement.states` inside `<ComponentName>.extension.ts`, following the same pattern as every other extension override (see [`COMPONENT-CONFIGURATION.md`](./COMPONENT-CONFIGURATION.md)):

```ts
import { extensions } from '@wix/astro/builders';
import { manifest } from './ComponentName.generated';

const componentExtension = extensions.editorReactComponent({
  // …other fields…
  editorElement: {
    ...manifest.editorElement,
    states: {
      hover: {
        displayName: 'Hover',
        className: 'elemHover',
        pseudoClass: 'hover',
      },
      disabled: {
        displayName: 'Disabled',
        className: 'elemDisabled',
        pseudoClass: 'disabled',
        props: { isDisabled: true },
      },
    },
  },
});
```

For inner elements, place a `states` block inside each `inlineElement` entry under `editorElement.elements` that needs one. `refElement` is **not yet supported** by the public schema — skip inner-element state work for any `refElement` entries.

---

## Insertion order

Inside the `editorElement` (or `inlineElement`) override, place `states` after `cssCustomProperties` / `cssProperties` / `selector`. Existing fields keep their relative order.

---

## Key-collision rule

If a state key already exists in the target `states` block (root or the inner element being patched), **skip that key silently** and surface it under "Skipped (collision)" in the summary. Existing wins — no merge, no overwrite, no inline replacement. If the existing entry uses a non-canonical shape, also surface it under "Convention drift".

---

## Steps

### Preconditions

| Check | Action on failure |
|---|---|
| Component exists at `src/site/components/<ComponentName>/` | Abort with the expected path. |
| `<ComponentName>.extension.ts` exists | Abort — scaffold the component first per [`EDITOR_REACT_COMPONENT.md`](../EDITOR_REACT_COMPONENT.md). |
| `editorElement` exists on the generated manifest (`<ComponentName>.generated.ts`) | Abort — this step only applies to components with an `editorElement` block. |

### Procedure

1. **Read the component.**
   - `<ComponentName>.tsx` (React)
   - `<ComponentName>.extension.ts` (overrides)
   - `<ComponentName>.generated.ts` (generated manifest, **read-only**)

2. **Decide the state list.**
   - **Root states:** match the React element against the "HTML element / ARIA role → native states" matrix and the "Role → suggested states" table in [`DESIGN-STATES-SUGGESTIONS.md`](./DESIGN-STATES-SUGGESTIONS.md). If the caller (typically the create-component flow, or the user) provided specific states, use those instead.
   - **Inner-element states:** walk `editorElement.elements` (recursively, depth ≤ 3). For each `inlineElement`, run the same checks against its rendered React + selector. `refElement` entries are skipped silently.

3. **Detect the current state.** Identify:
   - `editorElement.selector` (e.g. `.button`).
   - Existing `editorElement.states` override (if any) — flag convention drift if any entry uses a non-canonical shape.
   - For each inline-element target: its selector, existing `states` (if any), and convention drift.
   - For `disabled` root entries: any candidate inverted-prop in the React component (`isEnabled`, `isDisabled`, `disabled`, `enabled`) to use under `props`.

4. **Patch the root `editorElement.states` override.** Skip if the root state list is empty.
   - If no existing `editorElement.states` override: add it inside the `editorElement` override block.
   - If an existing override: merge new entries after the existing ones. Don't touch existing entries.
   - Collision check: for each key already present, skip and add to "Skipped (collision)".
   - For each native key, set `pseudoClass` to the matching string literal (`'hover'`, `'focus'`, `'disabled'`, `'invalid'`).
   - For each key, set `className` to the standard `elem<PascalKey>` form.
   - For `disabled` with a detected inverted-prop: add `props: { <prop>: <inverted value> }`.
   - Do not emit any comments.

5. **Patch inner elements.**
   - For each inline-element target with a non-empty suggested state list:
     - Insert/merge `states` the same way as root, inside its `inlineElement` block.
     - The `disabled` inverted-prop rule does **not** apply to inner elements (the prop lives on the component root). Do not emit comments.
   - For `refElement` targets: skip. Add to "Skipped (refElement)".

6. **Regenerate the manifest.**
   ```
   npx wix build && npx wix generate manifest
   ```
   This refreshes `<ComponentName>.generated.ts` (read-only). The extension overrides reapply on top at runtime.

7. **Validate.**
   ```
   npx tsc --noEmit
   npx wix build
   ```
   Both must exit 0.

8. **Summarize** as part of the create-component flow's overall completion report:
   ```
   Design states added: <ComponentName>

   Root states added:        <comma-separated keys, or "(none)">
   Inner-element states:
   - <path> (<displayName>): <comma-separated keys>
   - …                                                       (or "(none)")
   Skipped (collision):      <scope.key>, …                  (or "(none)")
   Skipped (refElement):     <innerName>, …                  (or "(none)")
   Convention drift:         <existing entries using legacy shape>  (or "(none)")

   File modified:
   - src/site/components/<ComponentName>/<ComponentName>.extension.ts
   ```

---

## Self-checks before reporting success

| Check | How |
|---|---|
| Every chosen key produced a state entry in the correct scope | Read `<ComponentName>.extension.ts` |
| Each native key has the matching `pseudoClass` string literal | Read the extension file |
| Each entry has a `className` in `elem<PascalKey>` form | Read the extension file |
| Each inner-element key resolved to an `inlineElement` (or was deliberately skipped) | Step 3 lookup |
| `npx tsc --noEmit` exits 0 | Step 7 |
| `npx wix build` exits 0 | Step 7 |

---

## Out of scope

- Authoring per-state CSS in component files — banned by [`CSS-GUIDELINES.md`](./CSS-GUIDELINES.md). The platform writes state CSS at runtime.
- Per-state CSS defaults (`statesDefaultValues` on `cssProperties` / `cssCustomProperties` items) — separate field; see the [States docs](https://dev.wix.com/docs/build-apps/develop-your-app/extensions/site-extensions/editor-react-components/manifest-reference/editor-element/states).
- `displayFilters` configuration — covered in the [States docs](https://dev.wix.com/docs/build-apps/develop-your-app/extensions/site-extensions/editor-react-components/manifest-reference/editor-element/states).
- States on `refElement`-backed inner elements — `refElement` is not yet supported by the public schema.
- Modifying existing state entries (no merge, no overwrite — collision rule).
- Removing states (this step only adds).
- Normalizing existing convention drift — surfaced in the summary; manual decision.
