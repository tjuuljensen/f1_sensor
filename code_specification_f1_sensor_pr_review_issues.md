# Task: Fix PR review issues for `font_style` in F1 Sensor bundled cards

## Goal

Clean up the `font_style` implementation so the PR satisfies the maintainer review:

1. The branch must contain only the font-style feature changes.
2. The default/missing `font_style` behavior must preserve the existing card appearance exactly.
3. `font_style: wide` must also preserve the existing card appearance exactly.
4. Only `font_style: balanced` and `font_style: system` may intentionally change typography.

Do not implement `f1_font`.
Do not add arbitrary external font support.
Do not make unrelated package/dependency changes.

---

## Current branch context

The feature branch has already been rebased onto the upstream `dev` branch.

Before editing, inspect the branch state. Do not run destructive Git commands unless explicitly asked.

Run:

```bash
git status
git log --oneline --decorate --max-count=20
git diff --name-only upstream/dev..HEAD
git diff upstream/dev..HEAD -- package.json package-lock.json
````

Expected:

* The branch should only contain the font-style/card/docs changes.
* `package.json` and `package-lock.json` should not appear in the diff.
* If package files appear, do not edit them manually. Report that the branch still contains unrelated changes.

---

## Main review issue to fix

The maintainer said:

> Ensure that the default wide option preserves the existing appearance exactly. Some letter-spacing values currently change even when users have not configured font_style, which would alter existing dashboards.

This means:

```yaml
# no font_style configured
type: custom:f1-next-race-card
```

and:

```yaml
type: custom:f1-next-race-card
font_style: wide
```

must render exactly like upstream `dev` rendered before this feature.

The current implementation must be changed if it defines default semantic typography variables in `:host` that alter existing `font-family` or `letter-spacing`.

---

## Golden rule

`wide` must be a no-op.

The feature may add:

```html
data-font-style="wide"
```

to the card host, but there must be no CSS selector or default CSS variable that changes the visual style for `wide`.

Only these selectors may intentionally override typography:

```css
:host([data-font-style="balanced"]) { ... }
:host([data-font-style="system"]) { ... }
```

Avoid this pattern for semantic typography variables:

```css
:host {
  --f1-card-heading-letter-spacing: 0.08em;
  --f1-card-table-letter-spacing: 0.04em;
}
```

because it can change the existing default rendering.

Use this pattern instead:

```css
.some-existing-selector {
  font-family: var(
    --f1-card-heading-font-family,
    "THE EXACT ORIGINAL FONT FAMILY FROM UPSTREAM DEV"
  );
  letter-spacing: var(
    --f1-card-heading-letter-spacing,
    THE_EXACT_ORIGINAL_LETTER_SPACING_FROM_UPSTREAM_DEV
  );
}

:host([data-font-style="balanced"]) {
  --f1-card-heading-font-family: "Formula1 Display", "Titillium Web", var(--primary-font-family, Roboto, Arial, sans-serif);
  --f1-card-heading-letter-spacing: 0.03em;
}

:host([data-font-style="system"]) {
  --f1-card-heading-font-family: var(--primary-font-family, Roboto, Arial, sans-serif);
  --f1-card-heading-letter-spacing: normal;
}
```

If an element did not previously have `letter-spacing`, do not add a default `letter-spacing` declaration to it. Only add letter-spacing changes inside explicit `balanced` or `system` overrides.

---

## Required implementation approach

### 1. Keep `font_style` config behavior

Keep the shared option:

```yaml
font_style: wide
font_style: balanced
font_style: system
```

Default:

```js
const DEFAULT_FONT_STYLE = "wide";
```

Invalid or missing values must normalize to `"wide"`.

Keep or add a helper like:

```js
const DEFAULT_FONT_STYLE = "wide";

const FONT_STYLE_OPTIONS = [
  {
    value: "wide",
    label: "Wide",
    description: "Original F1 Sensor typography with prominent Formula 1 wide/display styling.",
  },
  {
    value: "balanced",
    label: "Balanced",
    description: "Formula 1-inspired typography with reduced wide-font usage in dense areas.",
  },
  {
    value: "system",
    label: "System",
    description: "Use the Home Assistant/system font.",
  },
];

const VALID_FONT_STYLES = new Set(FONT_STYLE_OPTIONS.map((option) => option.value));

const normalizeFontStyle = (value) => {
  const normalized = String(value || "").trim().toLowerCase();
  return VALID_FONT_STYLES.has(normalized) ? normalized : DEFAULT_FONT_STYLE;
};
```

Do not support `f1_font`.

### 2. Keep host attribute support

Cards should apply:

```html
data-font-style="wide"
data-font-style="balanced"
data-font-style="system"
```

The host attribute is fine. It must not change styling by itself.

Keep a helper like:

```js
const applyFontStyleAttribute = (element, config) => {
  const fontStyle = normalizeFontStyle(config?.font_style);
  if (element.getAttribute("data-font-style") !== fontStyle) {
    element.setAttribute("data-font-style", fontStyle);
  }
};
```

Call it after config normalization.

### 3. Remove default typography semantic variables that affect `wide`

Search for these or similar variables:

```css
--f1-card-body-font-family
--f1-card-heading-font-family
--f1-card-display-font-family
--f1-card-table-font-family
--f1-card-label-font-family
--f1-card-body-letter-spacing
--f1-card-heading-letter-spacing
--f1-card-display-letter-spacing
--f1-card-table-letter-spacing
--f1-card-label-letter-spacing
```

If they are defined in plain `:host`, remove those default definitions unless they are harmless constants that are not applied directly.

Allowed in `:host`:

```css
:host {
  --f1-font-system: var(--primary-font-family, Roboto, Arial, sans-serif);
  --f1-font-brand-display: "Formula1 Display", "Titillium Web", var(--f1-font-system);
  --f1-font-brand-wide: "Formula1 Wide", "Formula1 Display", "Titillium Web", var(--f1-font-system);
}
```

Not allowed in plain `:host`:

```css
:host {
  --f1-card-heading-letter-spacing: 0.08em;
  --f1-card-table-letter-spacing: 0.04em;
}
```

because those may affect default rendering.

### 4. Preserve original CSS as fallbacks

For every modified `font-family` or `letter-spacing` declaration, compare against upstream `dev`.

Use commands like:

```bash
git show upstream/dev:custom_components/f1_sensor/www/f1-sensor-live-data-card/f1-sensor-live-data-card.js > /tmp/f1-card-upstream.js
```

Then compare the original CSS values.

If upstream had:

```css
.some-selector {
  font-family: "Formula1 Wide", "Formula1 Display", "Titillium Web", sans-serif;
  letter-spacing: 0.08em;
}
```

the new default must be:

```css
.some-selector {
  font-family: var(--f1-card-heading-font-family, "Formula1 Wide", "Formula1 Display", "Titillium Web", sans-serif);
  letter-spacing: var(--f1-card-heading-letter-spacing, 0.08em);
}
```

If upstream had:

```css
.some-selector {
  font-family: "Formula1 Display", "Titillium Web", sans-serif;
}
```

and no letter-spacing, the new default must be:

```css
.some-selector {
  font-family: var(--f1-card-body-font-family, "Formula1 Display", "Titillium Web", sans-serif);
}
```

Do not add:

```css
letter-spacing: var(--f1-card-body-letter-spacing, normal);
```

unless upstream already had an explicit `letter-spacing` on that selector.

### 5. Explicitly override only balanced/system

Add balanced/system overrides through host selectors.

Example:

```css
:host([data-font-style="balanced"]) {
  --f1-card-body-font-family: var(--f1-font-system);
  --f1-card-heading-font-family: var(--f1-font-brand-display);
  --f1-card-display-font-family: var(--f1-font-brand-display);
  --f1-card-table-font-family: var(--f1-font-system);
  --f1-card-label-font-family: var(--f1-font-system);

  --f1-card-heading-letter-spacing: 0.03em;
  --f1-card-display-letter-spacing: 0.01em;
  --f1-card-label-letter-spacing: 0.04em;
}

:host([data-font-style="system"]) {
  --f1-card-body-font-family: var(--f1-font-system);
  --f1-card-heading-font-family: var(--f1-font-system);
  --f1-card-display-font-family: var(--f1-font-system);
  --f1-card-table-font-family: var(--f1-font-system);
  --f1-card-label-font-family: var(--f1-font-system);

  --f1-card-heading-letter-spacing: normal;
  --f1-card-display-letter-spacing: normal;
  --f1-card-label-letter-spacing: normal;
}
```

Only define a variable if it is actually used.

If a shared semantic variable causes too many default-risk changes, prefer targeted selectors:

```css
:host([data-font-style="balanced"]) .timing-table,
:host([data-font-style="balanced"]) .timing-table * {
  font-family: var(--f1-font-system);
  letter-spacing: normal;
}
```

This is acceptable because it only affects explicit non-default modes.

### 6. Make the smallest safe change

Prefer minimal changes that satisfy the feature.

Do not refactor unrelated layout, spacing, colors, sizing, or card behavior.

Do not reformat the entire bundled card file.

Do not change generated package lock files.

Do not change Python integration code unless the existing feature already requires it.

---

## Documentation requirements

Keep docs for:

```yaml
font_style: wide
font_style: balanced
font_style: system
```

Update wording if needed:

```text
font_style controls card typography.

wide keeps the original F1 Sensor wide/display typography and is the default.
balanced keeps an F1-inspired look while reducing use of wide display fonts in dense or compact card areas.
system uses the Home Assistant/system font throughout the card.
```

Do not document `f1_font`.

Do not document arbitrary custom font families or font URLs.

---

## Visual editor requirements

If the card visual editor was updated, keep the `font_style` control.

Labels:

```text
Wide
Balanced
System
```

Default should be `wide`.

The visual editor must not write unrelated config changes.

---

## Verification commands

After making code changes, run:

```bash
git status
git diff --name-only upstream/dev..HEAD
git diff upstream/dev..HEAD -- package.json package-lock.json
```

Expected:

* No package file diff.
* No unrelated dependency changes.
* Only card/docs/translation files related to this feature.

Then inspect typography diffs:

```bash
git diff upstream/dev..HEAD -- custom_components/f1_sensor/www/f1-sensor-live-data-card/f1-sensor-live-data-card.js | grep -nE "font-family|letter-spacing|data-font-style|font_style|--f1-card|--f1-font" || true
```

Confirm:

* No default semantic typography variables in plain `:host` that affect `wide`.
* No `:host([data-font-style="wide"])` visual overrides.
* Existing font-family and letter-spacing values remain as per-selector fallbacks.
* `balanced` and `system` are the only modes that override typography.

---

## Build/test

Run the project’s normal card build commands based on the repository scripts.

First inspect available scripts:

```bash
cat package.json
```

Then run the appropriate existing build/check command, for example one of:

```bash
npm install
npm run build
npm run lint
npm test
```

Do not modify `package.json` or `package-lock.json` as part of this task.

If dependency installation changes `package-lock.json`, revert that file unless the project explicitly requires the lockfile update for this feature.

---

## Manual visual test checklist

Create three cards side by side or stacked:

```yaml
type: vertical-stack
cards:
  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark

  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: wide

  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: balanced

  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: system
```

Acceptance:

* First card and `font_style: wide` card look identical.
* Both match the old upstream/default appearance.
* `balanced` improves compact typography without removing all F1 character.
* `system` uses the Home Assistant/system font throughout.
* No horizontal overflow on mobile widths caused by the typography change.
* No console errors.

Test widths:

```text
320px
360px
390px
430px
desktop
```

---

## Final response required from Codex

When done, summarize:

1. What files changed.
2. Whether `package.json` and `package-lock.json` are clean.
3. How default `wide` behavior is preserved.
4. What build/check commands were run.
5. Any manual visual checks still required.







After Codex edits it, I would also run this yourself before pushing:

```bash
git diff --name-only upstream/dev..HEAD
git diff upstream/dev..HEAD -- package.json package-lock.json
git log --oneline upstream/dev..HEAD
````

The second command should output nothing. The log should show only your font-style commit or your font-style commit plus a small review-fix commit. Treat this as a small, reviewable infrastructure change in Git, not a broad cleanup pass. 
