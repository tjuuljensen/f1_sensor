# Codex implementation spec: Add `font_style` typography control to F1 Sensor live data cards

## Objective

Add a shared Lovelace card option named `font_style` to the F1 Sensor bundled live data cards.

The option must solve mobile readability issues caused by the wide/compressed Formula 1 display typography while keeping the current visual style as the default.

Do **not** add `f1_font`. Do **not** add arbitrary external font URL support.

## User-facing API

Add this shared option:

```yaml
font_style: brand
```

Supported values:

```yaml
font_style: brand
font_style: readable
font_style: system
```

Behavior:

| Value      | Purpose                              | Typography behavior                                                                                                             |
| ---------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `brand`    | Current look                         | Preserve the existing F1 Sensor appearance as closely as possible. Use Formula 1 display/wide fonts as the card currently does. |
| `readable` | Recommended mobile-friendly F1 style | Keep some F1 character, but stop using the wide/compressed font for dense text, tables, countdowns, and compact labels.         |
| `system`   | Maximum readability                  | Use the Home Assistant/system font throughout the card.                                                                         |

Default:

```yaml
font_style: brand
```

Invalid values must not break cards. Normalize invalid/missing values to `brand`.

Example usage:

```yaml
type: custom:f1-next-race-card
entity: sensor.f1_next_race
theme_mode: dark
font_style: readable
```

Another example:

```yaml
type: custom:f1-race-lap-card
entity: sensor.f1_race_lap_count
font_style: system
```

## Non-goals

Do not implement:

```yaml
f1_font: false
```

Do not implement:

```yaml
font_family: Some Font
```

Do not implement:

```yaml
font_url: https://example.com/font.woff2
```

Rationale: arbitrary fonts create support, privacy, caching, browser, and CORS problems. CSS variables are the correct advanced-user escape hatch.

---

# Implementation requirements

## 1. Add shared constants/helpers

Find the shared card utility area near existing common options such as `theme_mode` and editor helpers.

Add constants similar to:

```js
const DEFAULT_FONT_STYLE = "brand";

const FONT_STYLE_OPTIONS = [
  {
    value: "brand",
    label: "Brand",
    description: "Original F1 Sensor typography.",
  },
  {
    value: "readable",
    label: "Readable",
    description: "F1-inspired style with improved readability on compact/mobile layouts.",
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

Do not warn on every render. If adding a warning for invalid config, make it once-per-card-instance or skip warning entirely.

## 2. Apply a host attribute to every card instance

Each card should expose the normalized value on the custom element host:

```html
<f1-next-race-card data-font-style="readable">
```

Use this pattern inside each card class, or preferably through a shared helper installed on each card class:

```js
const applyFontStyleAttribute = (element, config) => {
  const fontStyle = normalizeFontStyle(config?.font_style);
  if (element.getAttribute("data-font-style") !== fontStyle) {
    element.setAttribute("data-font-style", fontStyle);
  }
};
```

Call it whenever `config` changes.

Preferred implementation points:

1. In `setConfig(config)`, after assigning `this.config`.
2. In `willUpdate(changedProps)` if the card already centralizes config-driven updates there.
3. In `connectedCallback()` as a safety fallback.

The goal is that this works immediately after dashboard load and after visual-editor changes.

Acceptance check in browser devtools:

```js
document.querySelector("f1-next-race-card")?.getAttribute("data-font-style")
```

Expected result:

```text
readable
```

for a card configured with `font_style: readable`.

## 3. Normalize config in `setConfig`

Where card classes currently do something like:

```js
this.config = { ...config };
```

change to:

```js
this.config = {
  ...config,
  font_style: normalizeFontStyle(config?.font_style),
};
applyFontStyleAttribute(this, this.config);
```

Do not mutate the incoming `config` object directly.

If there are many duplicated `setConfig` methods, add a shared helper:

```js
const normalizeSharedCardConfig = (config = {}) => ({
  ...config,
  font_style: normalizeFontStyle(config.font_style),
});
```

Then use:

```js
this.config = normalizeSharedCardConfig(config);
applyFontStyleAttribute(this, this.config);
```

## 4. Add shared typography CSS variables

Find the shared theme CSS block, likely near `F1_THEME_STYLES`.

Add a central typography variable block.

Use variable names like this:

```css
:host {
  --f1-font-system: var(--primary-font-family, Roboto, Arial, sans-serif);
  --f1-font-brand-display: "Formula1 Display", "Titillium Web", var(--f1-font-system);
  --f1-font-brand-wide: "Formula1 Wide", "Formula1 Display", "Titillium Web", var(--f1-font-system);

  --f1-card-body-font-family: var(--f1-font-brand-display);
  --f1-card-heading-font-family: var(--f1-font-brand-wide);
  --f1-card-display-font-family: var(--f1-font-brand-wide);
  --f1-card-table-font-family: var(--f1-font-brand-display);
  --f1-card-label-font-family: var(--f1-font-brand-display);

  --f1-card-body-letter-spacing: 0.02em;
  --f1-card-heading-letter-spacing: 0.08em;
  --f1-card-display-letter-spacing: 0.04em;
  --f1-card-label-letter-spacing: 0.12em;
  --f1-card-table-letter-spacing: 0.04em;
}

:host([data-font-style="readable"]) {
  --f1-card-body-font-family: var(--f1-font-system);
  --f1-card-heading-font-family: var(--f1-font-brand-display);
  --f1-card-display-font-family: var(--f1-font-brand-display);
  --f1-card-table-font-family: var(--f1-font-system);
  --f1-card-label-font-family: var(--f1-font-system);

  --f1-card-body-letter-spacing: normal;
  --f1-card-heading-letter-spacing: 0.03em;
  --f1-card-display-letter-spacing: 0.01em;
  --f1-card-label-letter-spacing: 0.04em;
  --f1-card-table-letter-spacing: normal;
}

:host([data-font-style="system"]) {
  --f1-card-body-font-family: var(--f1-font-system);
  --f1-card-heading-font-family: var(--f1-font-system);
  --f1-card-display-font-family: var(--f1-font-system);
  --f1-card-table-font-family: var(--f1-font-system);
  --f1-card-label-font-family: var(--f1-font-system);

  --f1-card-body-letter-spacing: normal;
  --f1-card-heading-letter-spacing: normal;
  --f1-card-display-letter-spacing: normal;
  --f1-card-label-letter-spacing: normal;
  --f1-card-table-letter-spacing: normal;
}
```

Important: in `readable`, do not use `Formula1 Wide` for countdowns, dense labels, table cells, driver names, TLAs, session names, or compact status text.

## 5. Replace hard-coded `font-family` declarations

Search for all hard-coded font families:

```text
Formula1 Display
Formula1 Wide
Titillium Web
font-family:
font:
```

Replace card CSS with semantic variables.

Use these mappings:

### General card containers

Before:

```css
font-family: "Formula1 Display", "Titillium Web", Arial, sans-serif;
```

After:

```css
font-family: var(--f1-card-body-font-family);
letter-spacing: var(--f1-card-body-letter-spacing);
```

### Headers and titles

Use:

```css
font-family: var(--f1-card-heading-font-family);
letter-spacing: var(--f1-card-heading-letter-spacing);
```

Targets include card titles, section titles, weekend schedule titles, race names, list titles, chart titles, and visual branded headings.

### Countdown / large display numbers

Use:

```css
font-family: var(--f1-card-display-font-family);
letter-spacing: var(--f1-card-display-letter-spacing);
```

Targets include countdown values, lap counters, session time display, large race/session hero text, and big timing values.

### Tables

For table wrappers and row cells:

```css
font-family: var(--f1-card-table-font-family);
letter-spacing: var(--f1-card-table-letter-spacing);
```

For column headers, small labels, pills, badges:

```css
font-family: var(--f1-card-label-font-family);
letter-spacing: var(--f1-card-label-letter-spacing);
```

### Buttons/selects

Buttons should inherit the appropriate family:

```css
font-family: inherit;
```

or:

```css
font-family: var(--f1-card-label-font-family);
```

depending on the surrounding style.

## 6. Keep `brand` visually backward-compatible

The `brand` mode is the migration guarantee. Existing dashboards with no `font_style` must look the same or nearly the same.

Do not change spacing, sizing, or layout just to make `brand` prettier.

The only acceptable visual differences in `brand` are small side effects from moving font declarations to variables.

## 7. Improve readable mode where the bug appears

The originally reported issue is most visible on `custom:f1-next-race-card`, especially the countdown and schedule on mobile.

Prioritize these targets:

```text
f1-next-race-card
f1-season-calendar-card
f1-live-session-card
f1-race-control-card
f1-qualifying-timing-card
f1-practice-timing-card
f1-race-lap-card
f1-starting-grid-card
f1-driver-lap-times-card
```

Readable mode acceptance criteria:

* No horizontal overflow at 320px width.
* Countdown remains legible.
* `T-2d 08h 01m`-style text does not appear stretched/widened.
* Dense table rows do not use the wide F1 font.
* Labels may retain a subtle F1 look, but not at the expense of readability.

## 8. Add visual editor support

Create a shared editor renderer near `renderThemeModeSelect`.

Example:

```js
const renderFontStyleSelect = (editor) => renderEditorSelect(
  editor,
  "font_style",
  "Font style",
  [
    { value: "brand", label: "Brand" },
    { value: "readable", label: "Readable" },
    { value: "system", label: "System" },
  ],
  "Controls card typography. Brand keeps the original look; Readable improves compact/mobile readability; System uses the Home Assistant font.",
);
```

Add it to each card editor’s Display tab near `theme_mode`.

Preferred ordering:

```text
Theme mode
Font style
Title
Layout/display toggles
Column toggles
```

The docs say each card already has a visual editor with data-source and display options, so this should be a first-class editor option, not YAML-only. ([Nicxe][1])

## 9. Update docs

Update `cards-overview` docs.

In **Shared Options**, add:

```text
font_style | brand, readable, system | Typography style. brand keeps the original F1 Sensor look, readable keeps an F1-inspired style without the wide font in dense/mobile text, and system uses the Home Assistant/system font.
```

Also add a short example under Adding Cards or Shared Options:

```yaml
type: custom:f1-next-race-card
entity: sensor.f1_next_race
theme_mode: auto
font_style: readable
```

Do not add `f1_font` to docs.

Do not document arbitrary font-family support.

## 10. Preserve resource/cache behavior

No changes should be needed to Lovelace resource registration. The card is already bundled by F1 Sensor and registered automatically. ([Nicxe][1])

For release notes, tell users:

```text
After updating, reload the browser or clear the Home Assistant frontend cache if the old card styling remains visible.
```

This matches the existing docs’ guidance for bundled card updates and stale resources. ([Nicxe][1])

---

# Testing plan

## Static checks

Run these searches after implementation:

```text
font_style
Formula1 Wide
Formula1 Display
font-family:
letter-spacing:
```

Expected:

* `font_style` appears in helpers, editors, docs, and examples.
* `Formula1 Wide` appears only in base font variables or clearly intentional brand-only code.
* Dense components should use CSS variables, not hard-coded `Formula1 Wide`.
* Most `letter-spacing` values should use typography variables where font-dependent.

## Manual card tests

Create a dashboard with:

```yaml
type: vertical-stack
cards:
  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: brand

  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: readable

  - type: custom:f1-next-race-card
    entity: sensor.f1_next_race
    theme_mode: dark
    font_style: system
```

Then compare at these viewport widths:

```text
320px
360px
390px
430px
768px
desktop
```

Acceptance:

* `brand` resembles current behavior.
* `readable` fixes the wide/compressed mobile readability problem.
* `system` looks like a normal Home Assistant card.
* No card throws console errors.
* Visual editor can set and save `font_style`.
* Switching values in the visual editor updates the card without needing a full HA restart.

## Devtools checks

Select text inside the card and run:

```js
getComputedStyle($0).fontFamily
```

Expected examples:

For `font_style: brand`, headings/display elements may show Formula 1 font families.

For `font_style: readable`, dense text/table/countdown should not use `Formula1 Wide`.

For `font_style: system`, text should resolve to the HA/system font stack.

Also check host attribute:

```js
document.querySelector("f1-next-race-card")?.getAttribute("data-font-style")
```

## Regression cards

Test at least these card types:

```text
custom:f1-next-race-card
custom:f1-season-calendar-card
custom:f1-live-session-card
custom:f1-race-control-card
custom:f1-fia-documents-card
custom:f1-qualifying-timing-card
custom:f1-practice-timing-card
custom:f1-race-lap-card
custom:f1-starting-grid-card
custom:f1-driver-lap-times-card
custom:f1-track-limits-card
custom:f1-investigations-card
custom:f1-sensor-live-data-card
custom:f1-pitstop-overview-card
custom:f1-track-map-card
```

The public docs list these bundled card types and their current purposes, so the option should behave consistently across the card family. ([Nicxe][1])

---

# Suggested implementation order

1. Add constants/helper functions.
2. Add `data-font-style` host attribute support.
3. Add typography CSS variables to the shared style block.
4. Convert one card first: `f1-next-race-card`.
5. Validate the mobile readability issue is fixed.
6. Convert remaining card CSS.
7. Add visual editor control.
8. Update docs.
9. Run manual dashboard tests.
10. Prepare release notes.

---

# Release note text

Use something like:

```text
Add shared `font_style` option for bundled F1 Sensor live data cards.

New values:
- `brand`: preserves the original F1 Sensor look.
- `readable`: keeps an F1-inspired style while improving compact/mobile readability.
- `system`: uses the Home Assistant/system font throughout the card.

Existing dashboards keep the original `wide` behavior by default.
```

This is a safe, low-migration change: no entity changes and no dashboard card type changes.
Keep it in Git as dashboard/frontend infrastructure so future card updates can be reviewed cleanly. 

[1]: https://nicxe.github.io/f1_sensor/cards/cards-overview "Live Data Cards | F1 Sensor"
[2]: https://raw.githubusercontent.com/tjuuljensen/f1_sensor/feature/font-style/custom_components/f1_sensor/www/f1-sensor-live-data-card/f1-sensor-live-data-card.js "${round.longLabel || round.label}"
