# AppML Syntax Reference

**WishTool / SpexWorx — Companion to the Agents Guide**

Status: living document — updated as wish→app conversion testing surfaces new cases.

This document specifies the concrete AppML XML syntax: elements, attributes, and confirmed runtime behavior. It assumes the architectural reasoning process (ownership vs. reference, pattern selection, navigation design) covered in the Agents Guide. Use that guide to decide *what* to build; use this document for *how to write it*.

Items not yet confirmed are marked **TBD**. Do not guess at TBD items when generating AppML — omit the feature or ask.

---

## 1. Root Structure

```xml
<app-schema>
  <application user-port="8081" dev-port="8082" name="AppName" />
  <theme name="default" />
  <landing component="c_1" name="Main Menu" />
  <!-- components -->
</app-schema>
```

- `<application>` — identity and runtime ports. `user-port` serves the running app; `dev-port` serves the Developer IDE.
  - `app-resources-path` — **deprecated. Do not emit.** Hydration content (media, seed data) is embedded directly in the generated application jar.
- `<theme name="...">` — sets the application-wide default theme. See §13.
- `<landing component="#id" name="...">` — defines the entry component shown on launch.
  - Whether the maintainer port (`:8082`) can have its own separate landing, distinct from the user port's — **TBD**.
- XML prolog (`<?xml version="1.0" encoding="UTF-8"?>`) — required or optional — **TBD**.

---

## 2. Component Types

| Element | Purpose |
|---|---|
| `<menu>` | Navigation entry point / hub. No persistent data. |
| `<table-1fk>` | A table capable of at most one owning parent. Used for both owned tables and standalone lookup tables (no parent). |
| `<table-multi-fk>` | A table that cross-references multiple independent components via `<fk-keys>`, optionally alongside ownership. |
| `<panel-dropzone>` | A drag/paste target that triggers a script. Carries no logic itself. |
| `<date-picker>` | A standalone input widget. Used to feed `<fk-keys>` as a filter source. |

There is no bare `<table>` element and no `<foreign-keys>`/`<fk>` wrapper — both appeared in one early draft and are invalid. See §15.

Every component carries:
```xml
id="..." name="..." theme="..." grid="x,y"
<bounds grid-rect="" form-rect="" />
```
`grid` and `<bounds>` are presentation only — leave `<bounds>` empty; the runtime computes it. Negative grid coordinates are used by convention to keep maintainer-only system tables visually off the main canvas — this is not a functional rule, just a convention observed in every sample.

---

## 3. Menu

```xml
<menu id="c_1" name="Main Menu" theme="default" grid="0,0">
  <bounds grid-rect="" form-rect="" />
  <action-links>
    <a href="#c_2">Artists</a>
    <a href="#c_4">Keywords</a>
  </action-links>
</menu>
```

- `<action-links>` — a flat list of plain navigation links, each `href` pointing at a component id (hash-prefixed).
- A menu can itself be the target of an `<action-menu>` (see §9) — this is how a table exposes a fan-out of unrelated destinations (e.g. a dropzone plus a reference table) without those destinations being owned children.
- Bare `<a>text</a>` entries with no `href` were observed once in a scratch test file — status **TBD**, likely inert placeholder junk rather than a real feature. Do not emit these.

---

## 4. table-1fk

```xml
<table-1fk id="c_2" name="Artists" theme="classic" grid="1,0">
  <bounds grid-rect="" form-rect="" />
  <columns>
    <col name="Artist" type="string" visible="true" is-key="true" table-tab="" form-tab="" />
    <col name="Biography" type="text box" visible="true" is-key="false" table-tab="" form-tab="" />
  </columns>
  <children>
    <child href="#c_5" />
  </children>
</table-1fk>
```

- `<children>` is optional. When present it declares exactly **one direct child** — a genuinely owned table, reached by pressing Enter (or opening/double-clicking a row) on a selected record. The XML schema does not forbid multiple `<child>` elements, but only the first is honored by the runtime.
- A `table-1fk` with no `<children>` and not referenced as anyone's child is a standalone/lookup table (e.g. Genres, Studios, Years) — this is the correct way to model independent lookup tables. There is no separate element for this case.
- A `table-1fk` can also carry a single `<action-menu>` (see §9).

---

## 5. table-multi-fk

```xml
<table-multi-fk id="c_5" name="Papers" theme="classic" grid="4,0">
  <bounds grid-rect="" form-rect="" />
  <action-menu href="#c_8" />
  <columns>
    <col name="Title" type="string" visible="true" is-key="false" table-tab="" form-tab="" />
  </columns>
  <children>
    <child href="#c_6" />
  </children>
  <fk-keys>
    <key source="c_2" display-col="AuthorName" label="First Author" order="0" />
    <key source="c_4" display-col="Keyword" label="Concept 1" order="2" group="Concepts" />
  </fk-keys>
</table-multi-fk>
```

A `table-multi-fk` can carry, simultaneously and independently:
- its own native `<columns>`,
- one owned `<children>` direct child,
- one `<action-menu>`,
- and an `<fk-keys>` block of cross-references.

None of these four are exclusive of the others.

### fk-keys

`<fk-keys>` is the correct, current syntax for cross-references. Each `<key>`:

| Attribute | Meaning |
|---|---|
| `source` | The bare id (no `#`) of the linked component. May be a table (`table-1fk`/`table-multi-fk`) **or** a standalone input widget such as `date-picker`. |
| `display-col` | The column (or the widget's own value) pulled from `source` and shown as this key's value. |
| `label` | Display text override. Falls back to `display-col` when blank. |
| `order` | Display sequence. |
| `group` | Optional. See below. |

**AND / OR grouping:** Keys are AND-combined by default — every declared key narrows the result further. Keys sharing the same `group` value instead OR together: any combination of them may match, and any of them may be left blank. This is how a single table (e.g. Keywords) can populate several parallel slots — Concept 1 / Concept 2 / Concept 3 — where a record matching *any* of the supplied concepts qualifies, rather than requiring all three.

Query operators (`=`, and presumably range/contains variants) are uniform across all key types regardless of the source column's type — the runtime does not currently validate operator use against column type.

A hard technical ceiling on the number of `<key>` elements per table — as opposed to a design-time heuristic (keep it browsable) — is **TBD**. One production sample carries 11 without issue.

---

## 6. panel-dropzone

```xml
<panel-dropzone id="c_9" name="Dropzone Panel" grid="4,4">
  <instructions>Drag%20or%20Paste%20here%20a%20link</instructions>
  <options status-bar="false" progress-bar="true" stop-button="false" />
</panel-dropzone>
```

- `<instructions>` text is URL-encoded.
- The dropzone itself performs no logic — it exists purely to trigger a script on drop/paste (see §11, `sys_scripts`). All actual behavior (parsing, fetching, creating rows) is implemented in the bound script.

---

## 7. date-picker

```xml
<date-picker id="c_10" name="Date Picker" grid="3,0" />
```

A standalone input widget, no `<columns>` of its own. Its sole confirmed purpose is to act as a filter `source` inside another component's `<fk-keys>`. Other standalone input types likely exist on the same principle (any single-value input widget can act as a filter source) — the full list beyond `date-picker` is **TBD**.

---

## 8. Columns & Types

```xml
<col name="ColumnName" type="string" visible="true" is-key="false" table-tab="" form-tab="" />
```

Valid `type` values (exact XML strings — note the space, not underscore, in multi-word types):

`string`, `number`, `boolean`, `date`, `time`, `media box`, `text box`, `coop box`, `html box`, `js box`

- `visible` and `is-key` should always be declared explicitly — there is no reliance on defaults.
- `is-key="true"` marks a table's primary identity/display column (one per table, in every observed sample — e.g. "Artist", "Genre", "Person"). This is the column shown when the table is referenced as an fk-key `display-col` elsewhere. The finer distinction between key columns, invisible system-managed columns, and invisible user-defined columns exists at the runtime level but the compiler does not need to model it explicitly.
- `table-tab` / `form-tab` — assign a field to a named tab in the grid view / record-edit form respectively, for tables with many fields. Leave as empty string when unused.

### Column type behaviors
- **media box** — universal rich-media container: images, PDF, audio, video. Supports drag-and-drop, in-place voice recording (Record button), and optional auto-transcription via a "Transcribe to Text Box" toggle. Transcription requires a sibling `text box` column on the same table; the runtime resolves the destination automatically — no explicit wiring in the XML.
- **coop box** — collaborative/shared editing.
- **text box** — long-form notes/documentation.
- **html box** — rendered HTML content (used by `sys_help`).
- **js box** — script source (used by `sys_scripts`).

---

## 9. Ownership vs. Lateral Navigation

Two distinct, non-overlapping mechanisms:

| | `<children><child href="#x"/></children>` | `<action-menu href="#x"/>` |
|---|---|---|
| Cardinality | One direct child per component | One action-menu per component, fanning out to unlimited step-children via its target `<menu>` |
| Semantics | Ownership | Pure navigation, no ownership implied |
| Activation | Enter (open/select row) | F2 |
| Target | Any component | Conventionally a `<menu>` component (canonical UI/UX) |

Both can be attached to the same component at once. A table can own exactly one direct child and additionally expose any number of secondary destinations (reference tables, dropzones, utility panels) through a single action-menu.

Keyboard/mouse/voice bindings live in a runtime commands table; voice triggers exist only as an unimplemented proof of concept. Enter and F2 are the confirmed defaults — changing them is possible in source but not expected.

---

## 10. System Tables

All system tables are `table-1fk` elements with `id` values prefixed `sys_`. Confirmed set:

### sys_scripts — "Scripted Components"
```xml
<col name="Component Name" type="string" is-key="true" />
<col name="Action Line" type="string" />
<col name="Event" type="string" />
<col name="Script" type="js box" />
```
Binds a script to a component and a trigger event. Confirmed event value: `F5`. Full event vocabulary and the purpose of `Action Line` — **TBD**. Whether `Component Name` must match a component's `id` or its `name` attribute — **TBD**.

Scripts cannot query external/cross-origin servers directly from client-side JS; that has been done via backend Java delegation in at least one case (sourcing author metadata from an external API), but the exact mechanism is **TBD** — flagged for follow-up.

### sys_themes — "Custom Themes"
```xml
<col name="Theme Name" type="string" is-key="true" />
<col name="Theme CSS" type="text box" />
```
See §13.

### sys_help — "Help Pages"
```xml
<col name="ID" type="string" is-key="true" />
<col name="Page Name" type="string" />
<col name="Content" type="html box" />
```
Owns `sys_help_media` as its one direct child. Internal help links use bare page ids (`href="getting_started"`), a scheme deliberately separate from component navigation's hash-prefixed ids.

### sys_help_media — "Help Media"
```xml
<col name="Media ID" type="string" is-key="true" />
<col name="File" type="media box" />
```

Whether `sys_` is a runtime-recognized prefix or pure naming convention, and the full canonical list of system tables beyond these four (auth, permissions, audit log) — **TBD**.

System tables may be given fixed explicit `bounds` values (e.g. `grid-rect="10,10,800,400"`) for simplicity; this is not load-bearing.

---

## 11. Seed Data

```xml
<seed-data>
  <row>
    <f_0>value for column 1</f_0>
    <f_1><![CDATA[ multi-line or rich content ]]></f_1>
  </row>
</seed-data>
```

- Fields are addressed positionally (`f_0`, `f_1`, ...) matching declared column order.
- CDATA wraps multi-line content (CSS, JS, HTML).
- Media can be embedded directly as base64 rather than a file path.
- In at least one child-table example, seed rows carried one field beyond the declared columns, apparently encoding the parent row's id for an owned relationship — whether this is a general rule for all owned seed-data or specific to that example is **TBD**.

---

## 12. Themes

- App-wide default: `<theme name="..."/>` at the root.
- Per-component override: `theme="..."` attribute on any component.
- Confirmed usable built-in ids: `default`, `classic`.
- The runtime theme picker also lists four additional hardcoded themes (labeled "Default Dark", "Glassy Blue (Datalator)", "Classic Light (Windows/Java)", "Solarized Dark"). These are legacy debugging artifacts, cannot currently be removed, and are **not intended for use** — do not reference them by name when generating AppML.
- Additional custom themes are defined via `sys_themes` seed rows. Each row's `Theme CSS` is a CDATA block scoped as:
  ```css
  body {
      --theme-base-bg: ...;
      --theme-comp-bg: ...;
      --theme-border: ...;
      --theme-title-bg: ...;
      --theme-title-color: ...;
      --theme-item-bg: ...;
      --theme-item-color: ...;
      --theme-item-active-bg: ...;
      --theme-item-active-color: ...;
  }
  ```
  Custom themes can be added at runtime on a live app. Whether they can subsequently be removed is **TBD** (untested).
- A component's `theme` attribute pointing at another component's `id` rather than an actual theme name was observed once (`sys_help_media` using `theme="sys_help"`) — meaning **TBD**.

---

## 13. Deprecated / Invalid Syntax — do not generate

| Syntax | Status |
|---|---|
| `<foreign-keys><fk label="..." source="#..."/></foreign-keys>` | Invalid. Use `<fk-keys><key .../></fk-keys>` (§5). |
| Bare `<table>` element | Invalid. Use `<table-1fk>` for all tables, including standalone lookup tables. |
| `app-resources-path` attribute | Deprecated. Omit entirely. |
| The four hardcoded debug themes (§12) | Not intended for use. |

---

## 14. Open Questions (TBD)

1. External/cross-server query mechanism used from scripts (backend Java delegation) — exact implementation forgotten, pending follow-up.
2. Full list of valid `sys_scripts` `Event` values beyond `F5`.
3. Purpose and format of `Action Line` in `sys_scripts`.
4. Whether `Component Name` in `sys_scripts` matches a component's `id` or `name`.
5. Whether the XML prolog is required.
6. Full canonical list of `sys_` tables beyond scripts/themes/help/help-media.
7. Whether `sys_` is a functionally special prefix or pure convention.
8. Whether owned-table seed-data rows always carry an implicit trailing parent-id field.
9. Whether there's a hard ceiling on `<fk-keys>` entries per table, or only a design heuristic.
10. Meaning of a `theme` attribute pointing at another component's id.
11. Whether custom themes can be removed at runtime once added.
12. Whether href-less `<a>` entries in `<action-links>` do anything.
13. Full list of standalone input component types (beyond `date-picker`) usable as fk-key sources.
14. Whether `<landing>` supports a separate value per port.
