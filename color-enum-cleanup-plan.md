# Cleanup: use the `Color` enum instead of raw-`String` colors

## Context

Today UI colors are represented as raw `String` at the Java data-layer boundary. The
element/model layers are *already* typed with the semantic `Color` enum
(`cyai.uiai.data.Color` — 10 values, `IDisplayableEnum`, wire strings
`primary`/`success`/`neutral`/`textPrimary`…), but `AbstractSSElement` flattens the enum to a
bare string via `.getValue()` before it ever reaches the data class. The data classes
(`AbstractElementData.color`, `CalloutData.accentColor`, two cell renderers) and their builders
still expose `String`, and ~50 test call sites pass string literals — several of them the
invalid placeholder `"color"`. This is error-prone and untyped.

**Goal:** make the Java data-layer API `Color`-typed and eliminate raw-string color usage, **while
keeping the JSON wire a bare string** so the React frontend is completely untouched. This is a
purely server-side type-safety cleanup.

**Why bare-string wire (not the enum object form):** React `AbstractElement.buildCommonProperties()`
(`ui-react/src/elements/AbstractElement.jsx:74`) reads `data.color` and passes it verbatim to MUI's
`color` prop; native-MUI-color sinks (`DialogElement:249`, `TabElement:107`, `ChipContent:134`,
`StatusBadgeCell:20`, `TimelineDot`) would break on the `{jsontype,value,…}` object form. The
tool is the **string-bridge idiom** already used by `ThemePreference.mode` and `ChartsAxis.scaleType`.

**Not in scope (documented, left as-is):**
- Fields carrying arbitrary hex/CSS/vars that the 10-value enum *cannot* represent — must stay
  `String`: `StatTileModel.color` (`var(--obs-color-*)`), `ChartVisualMap.colors` (hex ramp),
  `ChartsSeries.color` (hex), `GaugeProgress.color` (hex), `ChipData.bgColor` (hex).
  `SankeyFlavor`/`ChordFlavor.edgeColor` use a different enum (`RelationalEdgeColor`).
- The two fields already `Color`-typed but using the **object form** (`VisualMapPiece.color`,
  Timeline `dotColor`/`dotVariant`/`dot`): they are *not* raw-string offenders, and re-aligning them
  would require React changes. Left unchanged; the inconsistency is documented as intentional.
- React-side follow-ups (see "Optional follow-ups").

## Decisions taken (defaults, no user response)
- **Wire = bare string** via string-bridge (React untouched).
- **`withColor(String)` is fully replaced** by `withColor(Color)` — not kept/deprecated. Reasons:
  the placeholder `"color"` test sites prove raw strings are error-prone, and two `withColor`
  overloads would collide in `AbstractBuilder.withData`'s reflection map (keyed by lowercased
  `"color"`), a latent bug.
- **Object-form fields deferred** (VisualMapPiece / Timeline unchanged).
- **Java-only PR** — React follow-ups deferred.

---

## Phase 0 — `Color.fromValue` (blocking prerequisite)

`Color` has no reverse lookup; the global `IDisplayableEnum` deserializer round-trips by `name` and
won't apply to a String-typed bridge field, so the bridge setter needs a by-**wire-value** lookup.

File: `ui-core/ui-model/src/main/java/cyai/uiai/data/Color.java`
- Add `private static final Map<String, Color> BY_VALUE` (static init over `values()` keyed by
  `getValue()`) and `public static Color fromValue(String value)` returning
  `value == null ? null : BY_VALUE.get(value)`. Mirror `ThemeMode.fromValue`
  (`data/preferences/ThemeMode.java:23-38`) — return `null` (do not throw) for null/unknown, matching
  the existing bridges' null guards and `NON_NULL` wire omission.

## Phase 1 — `AbstractElementData.color` bridge (central, serial unit)

Highest blast radius; every element data class extends this. Edit + fix the one producer atomically
(compilation is broken between the retype and the producer fix).

Files:
- `ui-core/ui-model/src/main/java/cyai/uiai/data/AbstractElementData.java`
  - Field (`:35`): `String color` → `Color color`. **Drop `final` on this field only** so the
    Jackson string-bridge setter can write it (exactly as `ThemePreference.mode` is non-final);
    document why in a field comment.
  - Replace `getColor()` (`:98`) with the 3-method bridge:
    - `@JsonIgnore public Color getColor()` — typed getter (used by `equals()` + producers)
    - `@JsonProperty("color") public String getColorValue()` → `color != null ? color.getValue() : null`
    - `@JsonProperty("color") public void setColorValue(String v)` → `this.color = Color.fromValue(v)`
  - `equals()` (`:141`) keeps `Objects.equals(getColor(), data.getColor())` — now compares `Color`.
  - Builder field (`:162`) → `Color`; `withColor(String)` (`:225`) → **`withColor(Color)`** (replace,
    no String overload).
- `ui-core/ui-model/src/main/java/cyai/uiai/elements/impl/AbstractSSElement.java` (`:304-305`)
  - `builder.withColor(getColor().get() != null ? getColor().get().getValue() : null)` →
    `builder.withColor(getColor().get())` (element already holds `BiProperty<IObjectProperty<Color>,Color>`).

## Phase 2 — `CalloutData.accentColor` bridge (independent, parallelizable with Phase 3)

Files:
- `ui-core/ui-model/src/main/java/cyai/uiai/data/layouts/CalloutData.java` — same pattern as Phase 1
  on `accentColor` (`:19`, `withAccentColor(String)` `:77` → `withAccentColor(Color)`, bridge on wire
  key `"accentColor"`, drop `final`).
- Producer `ui-core/ui-model/src/main/java/cyai/uiai/elements/impl/layouts/CalloutElement.java` (`:49`)
  — drop `.getValue()`, pass the `Color`.

## Phase 3 — Cell renderers (two independent files, parallelizable)

These extend `AbstractCellRenderer` (not `AbstractElementData`); same bridge idiom, wire key `"color"`.
Files:
- `ui-core/ui-model/src/main/java/cyai/uiai/data/cells/renderers/StatusBadgeCellRenderer.java`
  (`String color` `:16`, `withColor(String)` `:80`)
- `ui-core/ui-model/src/main/java/cyai/uiai/data/cells/renderers/InlineChartCellRenderer.java`
  (`String color`, `withColor(String)` `:114`) — leave `flavor`/`showArea` etc. untouched.
- Producer `DemoStickyTableModel` (in `ui-demo-app`, `:188-199`) — feeds a `Color` via `.getValue()`;
  drop `.getValue()`.

## Phase 4 — Test + demo call-site sweep (parallelizable per file after Phases 1–3)

Every `.withColor("…")` / `.withAccentColor("…")` on a converted builder becomes a `Color` constant
(`Utils.assertJson` round-trips exercise the bridge). Add `import cyai.uiai.data.Color;` per file.
- **Valid wire strings → direct swap** (`"primary"`→`Color.PRIMARY`, etc.): `TabDataTest`,
  `LinearProgressDataTest`, `CircularProgressDataTest`, `ToggleButtonGroupDataTest`, `DialogDataTest`,
  `StatusBadgeCellRendererTest`, `InlineChartCellRendererTest`.
- **Placeholder `"color"` → real enum** (never valid): `ButtonDataTest`, `IconButtonDataTest`,
  `FloatingActionButtonDataTest`, `TextFieldDataTest`, `LoginDataTest`, the picker data tests,
  `TreeDataTest`, `DataTableDataTest`, `FormControlDataTest`, `DynamicDivDataTest`.
- **Do NOT touch** (hex / different class): `GaugeChartDataTest`, `GaugeChartElementTest`
  (`GaugeProgress.withColor("#…")`), demo `KpiLineTrendModel`/`KpiHorizontalBarModel`
  (`ChartsSeries.withColor("#…")`).
- Final guard: repo-wide grep for `withColor(` / `withAccentColor(` — the compiler flags any remaining
  String call sites regardless.

## Phase 5 — Documentation

Update `ui-core/ui-model/CLAUDE.md`:
- New "Semantic Color (string-bridge)" note near the "Icon Enum Pattern": `AbstractElementData.color`,
  `CalloutData.accentColor`, and the two cell renderers are `Color`-typed, serialized as a bare
  `.getValue()` string via the Jackson bridge (same idiom as `ThemePreference.mode`), because React
  reads `data.color` verbatim into MUI's `color` prop; `Color.fromValue` backs deserialization.
- List the fields that intentionally **stay `String`** (with reasons) and note the deferred
  object-form fields (`VisualMapPiece`, Timeline) as an intentional inconsistency.

---

## Verification

1. `mvn test -pl ui-core/ui-model` — full suite; the `Utils.assertJson` round-trips are the primary
   gate (exercise the bridge getter/setter for every converted class).
2. `mvn -pl ui-core/ui-demo-app -am compile` — ensures the A5 producer + demo call sites compile
   (demo app can fail even when ui-model passes).
3. `ui-core/ui-react` build — should be a no-op (wire unchanged); confirms nothing references a
   changed shape.
4. Playwright `charts` + `timeline` suites — confirm the bare-string `color` still lands in the MUI
   sinks; no baseline regeneration expected.
5. Spot-check one serialized element JSON (e.g. `ButtonDataTest`) shows `"color":"primary"` (bare
   string), not an object.

## Risks
- **`final` removal on `color`/`accentColor`** — mutable field on an otherwise-immutable DTO; only the
  Jackson setter writes it; mirrors `ThemePreference.mode`. Documented.
- **`fromValue` null semantics** — unknown wire string → `null` (not exception); acceptable given
  NON_NULL omission and enum-only producers; matches existing bridges.
- **Missed producers** — only `AbstractSSElement`, `CalloutElement`, `DemoStickyTableModel` produce
  these; compiler flags the rest.

## Optional follow-ups (separate PRs, not this effort)
- Re-align `VisualMapPiece.color` (and optionally Timeline) to the bare-string bridge for one
  consistent wire convention; drop the React `.value` reads.
- Consolidate the 3 duplicate React resolvers (`IconElement.resolveSxColor`,
  `CalloutElement.resolveAccent`, `SpinnerElement.paletteColor`) into `resolveInlineChartColor`.
- Route Cartesian + CalendarHeatmap visualMap piece colors through `resolveInlineChartColor` so
  semantic colors (`neutral`/`success`) actually render (currently only `HeatmapChartElement` does).