# SpexWorx Agents Guide 3.0
**For coding agents (Gemini, GPT, Claude, …) with no prior project knowledge**

---

## Purpose of This Guide

This document is SpexWorx's unified generative specification. Its purpose is to enable any LLM to convert a free-form natural-language "wish" or "intent" — typically a few paragraphs describing what kind of application a user wants — into a complete, strongly formalized set of output artifacts:

1. **`app_schema.xml`** — The full AppML schema defining all components, hierarchies, FK relationships, and system tables.
2. **Theme CSS** — Custom theme definitions (to be inserted into `sys_themes`).
3. **Hook Scripts** — Custom JavaScript scripts and macros (to be inserted into `sys_scripts`).
4. **Help Pages** — HTML help files and their companion media folder structure (to be placed in `resources/help/user/`).

3.0 merges the original *Comprehensive Agent Development Guide* (technical spec) with the *2.0 Philosophy* addendum (reasoning layer) into one three-part document:

- **Part I — Reasoning Layer** (Chapters 0–3): how to think about a wish before writing any XML. Read this in full before touching a wish.
- **Part II — Platform Reference** (Chapters 4–11): the complete AppML technical specification — schema syntax, scripting, theming, help system, system tables, deletion safety, and a full worked example. Consult this while generating.
- **Part III — Generation Workflow** (Chapters 12–16): a single merged step-by-step process, output manifest, ID rules, validation checklist, and pitfalls list. Follow this as the literal execution path, and check off Chapter 15 before returning any artifact.

---

# Part I — Reasoning Layer
*Do this before generating anything.*

## Chapter 0: Philosophy

SpexWorx is not a CRUD framework. It is a **navigation-oriented application architecture**.

The generator's primary task is to discover the user's information space and map it into navigable persistent structures.

Core concepts:

- Tables = persistent objects
- Hierarchies = ownership / containment
- Multi-FK = relationships across independent hierarchies
- Menus = entry points
- Scripts = automation
- Themes = presentation
- Help = documentation

Generate XML only after understanding the domain — Part II's syntax is only useful once this reasoning is done.

## Chapter 1: Mental Model

For every request, classify objects into:

1. Actors
2. Persistent entities
3. Owned entities
4. Reference entities
5. Workflows
6. Automation opportunities

Containment example:

Customer → Orders → Items

Reference example:

Order → Product → Employee → Warehouse

## Chapter 2: Domain Discovery

Before generating anything, answer:

- Who uses the application?
- What objects persist?
- Which objects own others?
- Which objects only reference others?
- Which rich media are needed?
- Which repetitive actions deserve scripts?

## Chapter 3: Architectural Heuristics

Rules:

- Exclusive ownership ⇒ `table-1fk` hierarchy.
- Independent lookup ⇒ standalone table.
- Multiple independent references ⇒ `table-multi-fk`.
- Menus point to hierarchy roots.
- Navigation should mirror how users think.

Architectural smells:

- Menu links directly to leaf table.
- Hierarchy used where references are appropriate.
- Multi-FK used when ownership exists.

---

# Part II — Platform Reference
*The technical spec — consult while generating.*

## Chapter 4: Platform Overview

### 4.1 What is SpexWorx?
SpexWorx is a self-replicating, zero-dependency Java/JS web IDE and application server. Its purpose is to let developers visually architect an application on a 2D grid canvas, then compile it into a standalone executable FAT JAR. The visual layout is translated into a declarative XML schema called **AppML** (`app_schema.xml`).

### 4.2 Dual-Server Architecture
When a generated app boots, a single JAR binds to **two ports** simultaneously:
- **User Port** (e.g., `8081`): Interprets the AppML schema to dynamically render the live User UI.
- **Maintainer Port** (e.g., `8082`): Exposes the full IDE, allowing the generated app to be further customized or even used to generate yet another app (fractal architecture).

Port configuration is dynamically mapped in the `<application>` tag:
```xml
<application user-port="8081" dev-port="8082" name="My App" />
```

### 4.3 Isolated Database Hydration
The Dev IDE **never** populates the User App's database. The User App spins up with a blank H2 database. On first boot:
1. The User UI fetches the embedded `app_schema.xml`.
2. When a component is activated, its JS Operator parses the schema and fires a heavy `POST` payload to the backend.
3. The backend executes `CREATE TABLE IF NOT EXISTS` to instantiate tables and hydrate them.

### 4.4 Access Control: 2 Executables, 3 Domains, 6 Roles

| Executable | Domain / Port | Admin Role (First Login) | Regular Role (Requires Approval) |
|---|---|---|---|
| Dev IDE | Dev (e.g., 8083) | Admin-Dev | Dev |
| Generated JAR | User (e.g., 8081) | Admin-User | User |
| Generated JAR | Maintainer (e.g., 8082) | Admin-Maintainer | Maintainer |

The first person to log into any domain is automatically promoted to the Admin role for that domain. All subsequent accounts enter a `PENDING` state until approved by the Admin.

---

## Chapter 5: AppML Schema Development

### 5.1 Schema Skeleton
An `app_schema.xml` document must contain a root `<app-schema>` tag with global metadata and component nodes:

```xml
<app-schema>
  <!-- Global Metadata -->
  <application user-port="8081" dev-port="8082" name="My App" />
  <theme name="default" />
  <landing component="c_1" name="Main Menu" />

  <!-- Components go here -->
</app-schema>
```

**Key elements:**
- `<application>`: Defines ports and app name.
- `<theme>`: Sets the global theme applied to all components unless overridden per-component.
- `<landing>`: Declares the **landing component** — the first component rendered on boot. This can be a Menu, a Table, or a Tab Pane. It does NOT have to be a Menu.

### 5.2 Grid Layout
The Dev Canvas uses a **2D coordinate system** (`grid="row,col"`). Coordinates are zero-based. Assign unique `grid` values to prevent overlap:
```
grid="0,0"  grid="0,1"  grid="0,2"
grid="1,0"  grid="1,1"  grid="1,2"
grid="2,0"  grid="2,1"  grid="2,2"
```
System components use negative coordinates (e.g., `grid="-1,-1"`) to keep them off the visible canvas.

### 5.3 Component Types

#### 5.3.1 Menu (`<menu>`)
A structural lightbox that organizes navigation via `<action-links>`:
```xml
<menu id="c_1" name="Main Menu" theme="default" grid="0,0">
  <bounds grid-rect="" form-rect="" />
  <action-links>
     <a href="#c_2">Authors Directory</a>
     <a href="#c_5">Papers Database</a>
  </action-links>
</menu>
```
**Rules:**
- Menu links reference component IDs via `href="#c_N"`.
- Menus should link to **hierarchy roots** and FK tables. Never link a menu directly to a child (leaf) table — that would bypass parent-row scoping and break child insert logic.
- A menu can also serve as an **Action Menu** for a table (accessed via F2 key), providing contextual operations like linking to step-child tables or launching tools like Dropzones.

#### 5.3.2 Tab Pane (`<tab-pane-top>` / `<tab-pane-side>`)
Navigation containers with tabbed layout. Uses `<tabs>` instead of `<action-links>`:
```xml
<tab-pane-top id="c_1" name="My Tabs" theme="default" grid="0,0">
  <tabs>
    <tab href="#c_2">Tab One</tab>
    <tab href="#c_5">Tab Two</tab>
  </tabs>
</tab-pane-top>
```
- `<tab-pane-top>`: Horizontal tabs.
- `<tab-pane-side>`: Vertical tabs.

#### 5.3.3 Simple Table (`<table-1fk>`)
The core data table supporting natural cascading hierarchies of **unlimited depth**:
```xml
<table-1fk id="c_2" name="Countries" theme="classic" grid="1,0">
  <bounds grid-rect="" form-rect="" />
  <columns>
    <col name="Name" type="string" visible="true" is-key="true" table-tab="" form-tab="" />
    <col name="Population" type="number" visible="true" is-key="false" table-tab="" form-tab="" />
  </columns>
  <children>
     <child href="#c_3" />
  </children>
</table-1fk>
```
**Hierarchy:** Parent-to-child links are declared via `<children><child href="#c_N" /></children>`. Each table can have at most one direct child. When the user presses **Enter** on a row, the app drills down into the child table, scoped to that parent row via `_parent_row_id`.

Natural hierarchies can be as deep as needed:
```
Country → State → City → Hotel → Room
   c_2  →  c_3  → c_4  → c_5  → c_6
```

**Action Menu:** A table can optionally reference a special-purpose Menu for contextual actions (accessed via the F2 key):
```xml
<action-menu href="#c_8" />
```

#### 5.3.4 Multi-FK Table (`<table-multi-fk>`)
A filtered table keyed by multiple foreign-key columns from other tables. Before viewing data, the user selects filter values from the source components:

```xml
<table-multi-fk id="c_5" name="Papers" theme="classic" grid="4,0">
  <bounds grid-rect="" form-rect="" />
  <columns>
    <col name="Title" type="string" visible="true" is-key="false" table-tab="" form-tab="" />
    <col name="Abstract" type="text box" visible="true" is-key="false" table-tab="" form-tab="" />
    <col name="PDF" type="media box" visible="true" is-key="false" table-tab="" form-tab="" />
  </columns>
  <fk-keys>
    <key source="c_2" display-col="AuthorName" label="First Author" order="0" />
    <key source="c_2" display-col="AuthorName" label="Last Author" order="1" />
    <key source="c_4" display-col="Keyword" label="Concept 1" order="2" group="Concepts" />
    <key source="c_4" display-col="Keyword" label="Concept 2" order="3" group="Concepts" />
    <key source="c_10" display-col="Date" label="" order="5" />
  </fk-keys>
</table-multi-fk>
```

**FK Key attributes:**
| Attribute | Description |
|---|---|
| `source` | Component ID of the source table (e.g., `c_2`). |
| `display-col` | Column from the source table to show in the filter picker. |
| `label` | Human-readable label for the filter field. |
| `order` | Visual order of the filter field (0-based). |
| `group` | **OR-group logic:** Assigning the same group string (e.g., "Concepts") to multiple keys logically binds them into a single "OR" query on the backend. Visually, they are grouped together in the filter UI. |

**Critical:** The same source table CAN appear multiple times with different labels (e.g., `c_2` as "First Author" AND "Last Author"). This is the Multi-FK mechanism for tables needing multiple references to the same source.

#### 5.3.5 Date Picker (`<date-picker>`)
A standalone date input component usable as an FK source:
```xml
<date-picker id="c_10" name="Date Picker" grid="3,0" />
```

#### 5.3.6 Dropzone Panel (`<panel-dropzone>`)
A file ingestion terminal for drag-and-drop or paste operations:
```xml
<panel-dropzone id="c_9" name="Dropzone Panel" grid="4,4">
  <instructions>Drag%20or%20Paste%20here</instructions>
  <options status-bar="false" progress-bar="true" stop-button="false" />
</panel-dropzone>
```

### 5.4 Column Types Reference

#### Primitive Data Types
| Type | SQL Mapping | Description |
|---|---|---|
| `string` | `VARCHAR(1000)` | General text. |
| `number` | `DOUBLE` | Numeric values. |
| `boolean` | `BOOLEAN` | True/false flag. |
| `date` | `DATE` | ISO date string. |
| `time` | `TIME` | Time value. |

#### Rich East Panel Box Types
These column types render as interactive panels on the right side of the table (the "East Panel"). They are stored as BLOBs or text payloads, not inline cell data.

| Type | Description |
|---|---|
| `text box` | A click-to-edit text area with a full-screen modal editor. Supports plain text, code, and markdown. |
| `media box` | A smart attachment container. See Section 5.5 for full details. |
| `coop box` | A real-time collaborative multi-user text editor. |
| `js box` | An executable JavaScript payload container. Used by `sys_scripts`. |
| `html box` | An HTML content container. Used by `sys_help`. |

#### Column Attributes
```xml
<col name="Title" type="string" visible="true" is-key="false" table-tab="" form-tab="" />
```
| Attribute | Description |
|---|---|
| `name` | Display header label. |
| `type` | One of the types above. |
| `visible` | Whether the column appears in the grid view. Rich box types are typically `visible="true"` but render in the East Panel, not inline. |
| `is-key` | The "master" / "hashtag tip" column — the primary display field. |
| `table-tab` | Tab assignment for grid view (when using tabbed column layouts). |
| `form-tab` | Tab assignment for F5 form view. |

### 5.5 The Smart Media Box

The `media box` column type is an intelligent attachment container. It natively accepts any BLOB (images, PDFs, documents) via drag-and-drop, but it possesses specialized behaviors based on the content type:

**Image files** (`image/*`): Renders an inline `<img>` preview with a toolbar containing a **Full Screen (⛶)** button and a **Delete (✕)** button.

**Audio files** (`audio/*`, including recorded `audio/webm`): Transforms its toolbar to provide native audio controls:
- **Play (▶)**: Starts playback via a hidden HTML5 `<audio>` element.
- **Pause (❚❚)**: Pauses playback.
- **Stop (⏹)**: Stops and rewinds to the beginning.
- A large musical note icon (♫) is displayed in the box body.
- The Full Screen button is hidden for audio content.

**Other files** (PDFs, video, etc.): Renders an `<embed>` tag with Full Screen and Delete buttons.

**Microphone Recording:** When the media box is empty, it displays a **Record (🎤)** button alongside the standard "drop media here" prompt. Clicking Record:
1. Requests microphone access via `navigator.mediaDevices.getUserMedia()`.
2. Captures audio using the `MediaRecorder` API.
3. The UI transforms into a pulsing red "Stop Recording..." state.
4. On stop, the recorded blob is automatically uploaded as the row's media payload.

**Note:** Microphone recording requires `localhost` or an HTTPS connection. For production deployment, use a reverse proxy (Nginx, Caddy) for SSL termination.

**Auto-Transcription:** A **"☑ Transcribe to Text Box"** checkbox appears next to the Record button (checked by default). When enabled:
1. Recording simultaneously activates the browser's native `SpeechRecognition` API.
2. As the user speaks, the transcript is built in real-time.
3. On stop, the final transcript is injected into the row's sibling `text box` column (if one exists) and saved to the database.

This feature is only available during **live microphone recording**. Drag-and-dropped audio files (e.g., MP3s) are not transcribed — the `SpeechRecognition` API only works with live microphone input.

> **Design imperative:** Choose `media box` whenever users may plausibly attach rich content — images, PDFs, audio, or voice notes. Design applications to exploit these capabilities (drag/drop, playback, live recording, auto-transcription into a sibling text box) whenever appropriate, not just when explicitly requested.

### 5.6 Hierarchy & Linkage Rules

1. **Natural Depth:** Cascading hierarchies (`table-1fk` → child → grandchild → …) can be as deep as needed. Example: `Country → State → City → Hotel → Room`.
2. **Menu/Tab Links:** Menus and Tabs should link to **hierarchy roots** — never directly to a leaf table.
3. **One Direct Child:** Each `table-1fk` can declare at most one direct child via `<children>`.
4. **Action Menu (Step-Children):** For tables that need access to secondary related tables (not direct children), use an Action Menu via `<action-menu href="#c_N" />`. The user presses **F2** to access it.
5. **FK Key Reuse:** A single source table can appear multiple times in `<fk-keys>` with different labels. This enables patterns like "First Author" and "Last Author" both pointing to the same Authors table.
6. **FK Key OR-Grouping:** Use the `group` attribute to functionally unite multiple foreign keys into a single logical "OR" group (e.g., `group="Concepts"` for Concept 1 and Concept 2). Instead of filtering by "Concept 1 AND Concept 2", the frontend packages these as a pipe-delimited payload (e.g., `f_2|f_3=val`), and the backend executes an "OR" SQL query (`WHERE col1 = X OR col2 = X`). This is essential for binding different key fields to the same table (e.g., a master Keywords table).

### 5.7 Keyboard Navigation Reference

| Key | Context | Action |
|---|---|---|
| **Enter** | Table row | Drill down into the child table scoped to that row. |
| **ESC** | Child table | Return to the parent level. |
| **Arrow Keys** | Any table/menu | Navigate the active cursor. |
| **PageUp/PageDown** | Any table | Scroll through pages of data. |
| **F2** | Table | Open the Action Menu (if one is linked). |
| **F5** | Table | Open the Form View modal for Create/Read/Update/Search. |
| **F8** | Table row | Delete the selected record (with cascading safety workflow). |
| **F9** | Any component | Save current window geometry (position, size) to the database. |

### 5.8 Database Seeding (`<seed-data>`)

To initialize tables with predefined rows on first boot (e.g., system configuration, default dropdown values, help documents), include a `<seed-data>` block inside any table definition.

```xml
<table-1fk id="c_100" name="StatusCodes" theme="classic" grid="5,5">
  <columns>
    <col name="Code" type="string" visible="true" is-key="true" />
    <col name="Description" type="string" visible="true" is-key="false" />
  </columns>
  <seed-data>
    <row>
      <f_0>OPEN</f_0>
      <f_1>Task is open</f_1>
    </row>
    <row>
      <f_0>CLOSED</f_0>
      <f_1>Task is closed</f_1>
    </row>
  </seed-data>
  <!-- if it has a child table -->
  <children><child href="#c_101" /></children>
</table-1fk>
```

**Field Mapping Rules:**
- The `<f_N>` tags directly correspond to the 0-indexed order of columns defined in the `<columns>` block.
  - `<f_0>` maps to the 1st column.
  - `<f_1>` maps to the 2nd column, and so on.
- **Child Tables:** If the table is a child of another table, the final `<f_N>` tag (where N = number of columns) represents the `_parent_row_id`. For example, if a child table has 2 columns (indices 0 and 1), `<f_2>` will store the foreign key linking it to the parent row.

**Rich Content & Media Rules:**
- **Code/HTML:** If seeding an `html box`, `js box`, or `text box` that contains raw HTML, scripts, or special characters, wrap the content in a `<![CDATA[ ... ]]>` block.
- **Binary Media:** To seed a `media box` (like images or PDFs), encode the raw file into a **Base64 string** and place it inside the tag. The backend will decode the Base64 payload upon database hydration.

```xml
<!-- Example of Base64 image payload -->
<col name="File" type="media box" visible="true" is-key="false" />
<seed-data>
  <row>
    <f_0>iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=</f_0>
  </row>
</seed-data>
```

---

## Chapter 6: Custom Scripting & Macro System

### 6.1 Architecture Overview

SpexWorx provides a JavaScript scripting engine via the **Hook Switchboard** (`HookSwitchboard.js`). Scripts are stored in the `sys_scripts` system table and executed in a sandboxed async context. The macro standard library (`WishMacros.js`) is automatically injected into every script, providing a compact DSL for UI automation and data manipulation.

### 6.2 The `sys_scripts` System Table

Scripts are stored as rows in `sys_scripts` with these columns:

| Column | Description |
|---|---|
| `Component Name` | The exact name of the target component (e.g., "Papers"). |
| `Action Line` | Optional: targets a specific action link label. Leave blank for event-level hooks. |
| `Event` | The event type to bind to (e.g., `Enter`, `ESC`, `F5`, `On Drop`, `On Filter`, `On Start`, `On Logoff`). |
| `Script` | The JavaScript code (stored in a `js box` column). |

### 6.3 Hook Types

**Event Hooks:** Fire when a specific event occurs on a component.
- Example: When the user presses Enter on a row in "Authors", execute a script.
- Defined by: `Component Name = "Authors"`, `Event = "Enter"`, `Action Line = ""`.

**Action Hooks:** Fire when a specific action link is clicked within an Action Menu.
- Example: When the user clicks "Run Analysis" in the Action Menu attached to the "Reports" table.
- Defined by: `Component Name = "Reports"` (MUST be the parent table's name, not the action menu's name), `Action Line = "Run Analysis"`, `Event = ""`.
- **CRITICAL DIRECTIVE:** The `<a>` tag in the Action Menu MUST omit the `href` attribute. If an `href` is present, the engine treats it as navigation and skips the script hook.

**Complete Action Hook Example in XML:**
```xml
  <!-- 1. The Action Menu attached to the parent table -->
  <menu id="c_5" name="Action Menu" theme="default" grid="1,2">
    <action-links>
       <!-- Omit href to trigger a Script Hook -->
       <a>Run Analysis</a>
    </action-links>
  </menu>

  <!-- 2. The sys_scripts definition targeting the parent table -->
  <table-1fk id="sys_scripts" name="Scripted Components" grid="-1,-1">
    <!-- ... columns ... -->
    <seed-data>
      <row>
        <f_0>Reports</f_0> <!-- Matches the parent table name -->
        <f_1>Run Analysis</f_1>
        <f_2></f_2>
        <f_3><![CDATA[console.log("Analysis Started!");]]></f_3>
      </row>
    </seed-data>
  </table-1fk>
```

### 6.4 Sandbox Context Variables

Every script automatically receives these variables:

| Variable | Type | Description |
|---|---|---|
| `delay` | `int` | Default delay (ms) used by UI macros. Mutable — set `delay = 100;` to speed things up. |
| `columns` | `Array<{name, type, value}>` | The active table's column schema. Modify `value` to stage data for off-screen macros. |
| `rowsNumber` | `int` | Total rows in the active table. |
| `activeRow` | `int` | The integer index of the active cursor position. |
| `activeRowId` | `string` | The database row ID of the currently selected row. |
| `tableId` | `string` | The active table's component ID. |

### 6.5 The WishMacros Standard Library

All macros are `async` and automatically respect the `delay` variable unless overridden.

#### Common
| Macro | Description |
|---|---|
| `await sleep(ms)` | Pauses execution for the specified milliseconds. |

#### On-Screen (UI Emulation)
These macros simulate user keyboard interaction. They visibly affect the UI and wait for `delay` before executing.

| Macro | Description |
|---|---|
| `await ArrowDown(ms?)` | Moves the cursor down. Optional custom delay. |
| `await ArrowUp(ms?)` | Moves the cursor up. |
| `await ArrowLeft(ms?)` | Navigates left. |
| `await ArrowRight(ms?)` | Navigates right. |
| `await PageDown(ms?)` | Scrolls down one page. |
| `await PageUp(ms?)` | Scrolls up one page. |
| `await Tab(ms?)` | Emulates Tab key. |
| `await UIEnter(ms?)` | Emulates Enter (e.g., save and close a modal). |
| `await Escape(ms?)` | Emulates Escape (e.g., close modal, go back). |
| `await F5(ms?)` | Opens the Form View / Edit modal. |
| `FillFormInputs(values)` | Fills open form inputs with an array of values. |

#### Off-Screen (Background Data)
These macros manipulate data directly via the backend API without any visible UI prompts. Use `RefreshGrid()` after background changes to update the visible table.

| Macro | Description |
|---|---|
| `await InsertRow()` | Inserts a new row using staged `columns[].value` data. Automatically resets values after insert. |
| `await UpdateActiveRow()` | Updates the active row using staged `columns[].value` data. |
| `await DeleteActiveRow()` | Silently deletes the active row. |
| `RefreshGrid()` | Commands the grid to pull fresh data from the server. |

### 6.6 Script Examples

**Example 1: On-Screen F5 Edit Flow**
```javascript
// Open edit modal, fill fields, and save
await F5();
FillFormInputs(["John Doe", "john.doe@example.com", "555-0192"]);
await UIEnter();
```

**Example 2: Off-Screen Bulk Data Injection**
```javascript
for (let i = 1; i <= 5; i++) {
    columns.forEach(col => {
        if (col.type === 'string') col.value = "Bulk Data " + i;
    });
    await InsertRow();
}
RefreshGrid();
```

**Example 3: Grid Traversal**
```javascript
const steps = Math.max(0, (rowsNumber - 1) - activeRow);
for (let i = 0; i < steps; i++) {
    await ArrowDown();
}
```

**Example 4: Speedy Form Entry**
```javascript
delay = 100; // Speed up all macros
await F5();
FillFormInputs(["Product A", "SKU-9921", "14.99"]);
await Tab();
await UIEnter();
delay = 1000; // Reset
```

**Example 5: The Wiper (Delete Last N Rows)**
```javascript
const rowsToDelete = Math.min(3, rowsNumber);
for (let i = 0; i < rowsToDelete; i++) {
    await ArrowDown(10);
    await DeleteActiveRow();
}
RefreshGrid();
```

**Example 6: Fast "DDoS-Level" Macro (Maximum Throughput)**
Use this approach when you need to process massive amounts of data as quickly as possible (e.g., bulk initialization or aggressive stress testing). 
*Warning:* Running massive loops without delay will trigger cascading `table-data-updated` events to all connected clients. If running at scale, this can overwhelm browser connection pools and cause `Failed to fetch` errors unless event propagation is temporarily muted.
```javascript
// Temporarily mute live UI updates to prevent browser memory/connection exhaustion
window.suppressTableUpdates = true;
const blockUpdate = (e) => { if (window.suppressTableUpdates) e.stopImmediatePropagation(); };
document.addEventListener('table-data-updated', blockUpdate, true);

for (let i = 1; i <= 10000; i++) {
    columns.forEach(col => { if (col.type === 'string') col.value = "Fast Data " + i; });
    // Insert without any additional delay
    await InsertRow(); 
}

// Unmute and trigger one final refresh
window.suppressTableUpdates = false;
RefreshGrid();
document.removeEventListener('table-data-updated', blockUpdate, true);
```

**Example 7: Slow "Humanoid" Macro (Realistic Load Testing)**
Use this approach to simulate a realistic human typing and clicking pace (e.g., a "monkey banging on a keyboard" at 2 to 5 actions per second). This allows the browser to naturally digest live UI updates and garbage collect memory without crashing.
```javascript
delay = 500; // Humanoid pace: 500ms (2 actions per second)

for (let i = 1; i <= 2000; i++) {
    columns.forEach(col => { if (col.type === 'string') col.value = "Humanoid Data " + i; });
    
    // The insert triggers a backend update
    await InsertRow(); 
    
    // CRITICAL: Await the delay to yield the event loop and allow the browser 
    // to process the incoming 'table-data-updated' websocket/polling events
    await sleep(delay); 
}
// Refresh grid is optional here since the live UI updates were not muted
RefreshGrid();
```

### 6.7 Extending the Macro Registry
The `WishMacros` object is attached to `window.WishMacros`. Because it is a simple registry object, you can dynamically add or override macros at runtime from any script:
```javascript
window.WishMacros['MyCustomMacro'] = (ctx) => async () => {
    // Your custom logic here
    await ctx.sleep(500);
    console.log("Custom macro executed!");
};
```

---

## Chapter 7: Custom Themes Development

### 7.1 Theme Architecture
The application's visual identity is controlled by a centralized CSS variable engine defined in `themes.css`. Themes override these variables to change colors, borders, shadows, and typography globally across all components.

### 7.2 System Themes (Hardcoded)
System themes are defined in `resources/user/css/themes.css`. Available system themes include:
- `default` — Dark theme
- `theme-glassy-blue` — Datalator 3.1 style glassmorphism
- `theme-classic-light` — Windows/Java System style
- `theme-solarized-dark` — Solarized color palette

### 7.3 Custom Themes (Database-Stored)
Custom themes are stored in the `sys_themes` system table. Each row has:
- `Theme Name`: The CSS class name (e.g., `theme-my-custom`).
- `Theme CSS`: The full CSS override (stored in a `text box` column).

Custom themes are loaded on app boot and injected as `<style>` tags. They are immediately available in the theme dropdown.

### 7.4 Applying Themes
- **Globally:** Set in the `<theme>` tag of the schema: `<theme name="theme-my-custom" />`.
- **Per-Component:** Override via the `theme` attribute: `<menu id="c_1" theme="theme-glassy-blue" ...>`.

### 7.5 CSS Variable Reference

```css
body.theme-my-custom {
    /* Global App Background */
    --bg-main: #0b1320;
    --text-main: #f1f5f9;

    /* Component Structure */
    --theme-base-bg: #0b1320;
    --theme-comp-bg: #111e30;
    --theme-border: 1px solid rgba(16, 185, 129, 0.4);
    --theme-border-radius: 12px;
    --theme-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.5);

    /* Title Bar (Active) */
    --theme-title-bg: linear-gradient(135deg, #065f46, #047857, #065f46);
    --theme-title-color: #ecfdf5;
    --theme-title-border: 1px solid rgba(16, 185, 129, 0.4);
    --theme-title-padding: 6px 12px;

    /* Title Bar (Inactive — when a child menu overlays the parent) */
    --theme-title-inactive-bg: linear-gradient(135deg, #1e293b, #334155);
    --theme-title-inactive-color: #94a3b8;

    /* List/Grid Items */
    --theme-item-bg: #16263c;
    --theme-item-color: #f8fafc;
    --theme-item-hover-bg: #1b384a;
    --theme-item-hover-color: #6ee7b7;
    --theme-item-selected-bg: #1d4b5a;
    --theme-item-active-bg: #059669;
    --theme-item-active-color: #ffffff;
    --theme-item-border: none;

    /* Inactive Window Active Breadcrumb (Classic Windows-style gray) */
    --theme-item-inactive-active-bg: #475569;
    --theme-item-inactive-active-color: #f1f5f9;

    /* Font Sizing */
    --theme-font-size: 13px;
    --theme-title-font-size: 14px;
}

/* Glassmorphism Effect */
body.theme-my-custom .wish-component {
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
}
```

### 7.6 Complete Theme Example: Crisp Light Greenish Glassy

```css
body {
    --theme-base-bg: #f0fdf4;
    --theme-comp-bg: rgba(16, 185, 129, 0.15);
    --theme-border: 1px solid rgba(16, 185, 129, 0.4);
    --theme-border-radius: 12px;
    --theme-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.25);

    --theme-title-bg: linear-gradient(135deg, #065f46, #047857, #065f46);
    --theme-title-color: #ecfdf5;

    --theme-title-inactive-bg: linear-gradient(135deg, #475569, #64748b);
    --theme-title-inactive-color: #f1f5f9;

    --theme-item-bg: rgba(255, 255, 255, 0.6);
    --theme-item-color: #064e3b;
    --theme-item-hover-bg: rgba(16, 185, 129, 0.25);
    --theme-item-active-bg: #10b981;
    --theme-item-active-color: #ffffff;

    --theme-item-inactive-active-bg: #64748b;
    --theme-item-inactive-active-color: #ffffff;
}

body .wish-component {
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
}
```

---

## Chapter 8: Help System Development

### 8.1 Architecture Overview
The help system is powered by two system tables:
- **`sys_help`**: Stores HTML content pages. Columns: `ID` (page slug), `Page Name`, `Content` (`html box`).
- **`sys_help_media`**: Child table of `sys_help`. Stores image/media files. Columns: `Media ID` (filename), `File` (`media box`).

On first boot, the app hydrates these tables from the `<seed-data>` inline XML payloads embedded in the schema. After hydration, the pages are served entirely from the database.

### 8.2 File Structure Convention
Help pages must follow a strict naming convention in the `resources/help/user/` directory:

```
resources/help/user/
├── index.html              ← The help directory landing page
├── index/                  ← Media folder for the index page
│   └── welcome.png
├── getting_started.html    ← A help topic page
├── getting_started/        ← Companion media folder (MUST share exact same name)
│   └── main_menu.png
├── custom_themes.html
├── custom_themes/
│   ├── themes_editor.png
│   └── custom_theme_example.png
```

**Rules:**
1. Every help topic consists of an **HTML file** and a **companion folder** that share the exact same base name.
2. The companion folder contains all images/media referenced by that page.
3. The HTML file should contain **raw HTML snippets only** — no `<html>`, `<head>`, or `<body>` wrappers. The app engine wraps content automatically.

### 8.3 Writing Help Content

```html
<h1>Getting Started & Overview</h1>
<p>Welcome to the application!</p>

<!-- Embed media from the companion folder by filename only -->
<img src="main_menu.png" style="max-width: 500px; border-radius: 8px;">

<h2>Where to Go From Here</h2>
<ul>
    <li><strong>Authors:</strong> Manage authors and papers.</li>
    <li><strong>Papers Database:</strong> The core tracker.</li>
</ul>

<br><br>
<a href="navigation_and_hotkeys">Learn Navigation Hotkeys</a> | <a href="index">Back to Directory</a>
```

### 8.4 Linking Rules

**Page-to-page links:** Reference the page slug (filename without `.html`):
```html
<a href="getting_started">Getting Started</a>
<a href="index">Back to Directory</a>
```

**Component-context links:** Add `data-context` to make a link that also highlights a specific app component:
```html
<a href="papers_database" data-context="c_5">Papers Database Help</a>
```
When clicked, this navigates to the help page AND focuses the app on component `c_5`.

**Media references:** Use just the filename. The engine automatically routes to the `sys_help_media` database:
```html
<img src="screenshot.png" style="max-width: 400px; border-radius: 8px;">
```

### 8.5 The Help Directory (Index Page)
The `index.html` is the entry point. It should list all topics with links:
```html
<h1>Help Directory</h1>
<p>Click on any topic below to learn more.</p>

<img src="pre_start_config.png" style="max-width: 400px; border-radius: 8px;">

<hr>
<h2>Core Topics</h2>
<ul>
  <li><a href="getting_started" data-context="c_1">Getting Started</a></li>
  <li><a href="navigation_and_hotkeys">Navigation & Hotkeys</a></li>
</ul>

<h2>Maintainer Features</h2>
<ul>
  <li><a href="custom_scripts">How to Create Custom Scripts</a></li>
  <li><a href="custom_themes">How to Create Custom Themes</a></li>
  <li><a href="custom_help">How to Create Custom Help Files</a></li>
</ul>
```

### 8.6 Updating Help Post-Deployment
After the app has booted and hydrated, help content lives in the database. To update:
1. Navigate to `sys_help` via the Admin-User interface.
2. Edit the `Content` (html box) column for any page.
3. To add/replace media, navigate into `sys_help_media` (the child table) and drag-drop images into the media box.

---

## Chapter 9: System Components Reference

System components are special tables that manage the app's internal configuration. They are declared in the schema with IDs prefixed by `sys_` and placed at negative grid coordinates:

```xml
<!-- Custom Scripts -->
<table-1fk id="sys_scripts" name="Scripted Components" theme="classic" grid="-1,-1">
  <columns>
    <col name="Component Name" type="string" visible="true" is-key="true" />
    <col name="Action Line" type="string" visible="true" is-key="false" />
    <col name="Event" type="string" visible="true" is-key="false" />
    <col name="Script" type="js box" visible="true" is-key="false" />
  </columns>
  <seed-data>
    <row>
      <f_0>Authors</f_0>
      <f_1></f_1>
      <f_2>Enter</f_2>
      <f_3>console.log("Hello");</f_3>
    </row>
  </seed-data>
</table-1fk>

<!-- Custom Themes -->
<table-1fk id="sys_themes" name="Custom Themes" theme="default" grid="-1,-2">
  <columns>
    <col name="Theme Name" type="string" visible="true" is-key="true" />
    <col name="Theme CSS" type="text box" visible="true" is-key="false" />
  </columns>
  <seed-data>
    <row>
      <f_0>theme-custom-1</f_0>
      <f_1>body { --theme-base-bg: #000; }</f_1>
    </row>
  </seed-data>
</table-1fk>

<!-- Help Pages -->
<table-1fk id="sys_help" name="Help Pages" theme="default" grid="-1,-3">
  <columns>
    <col name="ID" type="string" visible="true" is-key="true" />
    <col name="Page Name" type="string" visible="true" is-key="false" />
    <col name="Content" type="html box" visible="true" is-key="false" />
  </columns>
  <seed-data>
    <row>
      <f_0>getting_started</f_0>
      <f_1><![CDATA[<h1>Welcome</h1><img src="logo.png">]]></f_1>
    </row>
  </seed-data>
  <children>
     <child href="#sys_help_media" />
  </children>
</table-1fk>

<!-- Help Media (child of sys_help) -->
<table-1fk id="sys_help_media" name="Help Media" theme="sys_help" grid="-1,-4">
  <columns>
    <col name="Media ID" type="string" visible="true" is-key="true" />
    <col name="File" type="media box" visible="true" is-key="false" />
  </columns>
  <seed-data>
    <row>
      <f_0>logo.png</f_0>
      <f_1>iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=</f_1>
      <f_2>1</f_2> <!-- parent row ID references the exact row inserted into sys_help -->
    </row>
  </seed-data>
</table-1fk>
```

**Important:** System components are automatically hydrated on first boot **only if they are included in the schema**. If you omit them, the generated app boots cleanly without those features.

---

## Chapter 10: Data Deletion & Cascading Safety

Deleting records in SpexWorx triggers a multi-phase safety protocol:

1. **Leaf Records:** Simple confirmation dialog. If another user has a lock on the record, the delete is rejected.
2. **Cascading Deletes (Parent Records):**
   - **Step 1 — "DO NOT ENTER" Broadcast:** The system locks the entire cascade chain and broadcasts a lockdown to all connected users.
   - **Step 2 — Danger Zone Scan:** The server inspects all active sessions for users currently inside the affected tables.
   - **Step 3 — Impact Dialog:** Shows a side-by-side report: (Left) how many records will be deleted per table, (Right) any active users in the zone.
   - **Step 4 — Approval:** If the zone is empty, the Proceed button is enabled. If users are present, deletion is blocked.
   - **Step 5 — Release:** Cancelling releases all locks and broadcasts a release message.

---

## Chapter 11: Complete Worked Example (arXiv Papers Tracker)

This real-world schema demonstrates: deep cascading hierarchies, Multi-FK tables with repeated source references and grouping, action menus, dropzone panels, date pickers, and all four system component types.

```xml
<app-schema>
  <application user-port="8081" dev-port="8082" name="arXivePapers"
               app-resources-path="C:\path\to\sys_tables" />
  <theme name="default" />
  <landing component="c_1" name="Main Menu" />

  <!-- Main Menu (Landing) -->
  <menu id="c_1" name="Main Menu" theme="default" grid="0,0">
    <bounds grid-rect="" form-rect="" />
    <action-links>
       <a href="#c_2">Authors Directory</a>
       <a href="#c_4">Keywords</a>
       <a href="#c_5">Papers Database</a>
    </action-links>
  </menu>

  <!-- Authors → Co-Authors (depth=2 hierarchy) -->
  <table-1fk id="c_2" name="Authors" theme="classic" grid="1,0">
    <bounds grid-rect="" form-rect="" />
    <columns>
      <col name="AuthorName" type="string" visible="true" is-key="true" />
    </columns>
    <children><child href="#c_3" /></children>
  </table-1fk>

  <table-1fk id="c_3" name="Co-Authors" theme="classic" grid="1,1">
    <bounds grid-rect="" form-rect="" />
    <columns>
      <col name="CoAuthorName" type="string" visible="true" is-key="false" />
      <col name="PaperTitle" type="string" visible="true" is-key="false" />
      <col name="Date" type="date" visible="true" is-key="false" />
    </columns>
  </table-1fk>

  <!-- Keywords (standalone, used as FK source) -->
  <table-1fk id="c_4" name="Keywords" theme="classic" grid="2,0">
    <columns>
      <col name="Keyword" type="string" visible="true" is-key="true" />
    </columns>
  </table-1fk>

  <!-- Date Picker (FK source for publication date) -->
  <date-picker id="c_10" name="Date Picker" grid="3,0" />

  <!-- Papers (Multi-FK — the beast) -->
  <table-multi-fk id="c_5" name="Papers" theme="classic" grid="4,0">
    <bounds grid-rect="" form-rect="" />
    <action-menu href="#c_8" />
    <columns>
      <col name="Title" type="string" visible="true" is-key="false" />
      <col name="PublicationDate" type="date" visible="true" is-key="false" />
      <col name="Abstract" type="text box" visible="true" is-key="false" />
      <col name="PDF" type="media box" visible="true" is-key="false" />
    </columns>
    <children><child href="#c_6" /></children>
    <fk-keys>
      <key source="c_2" display-col="AuthorName" label="First Author" order="0" />
      <key source="c_2" display-col="AuthorName" label="Last Author" order="1" />
      <key source="c_4" display-col="Keyword" label="Concept 1" order="2" group="Concepts" />
      <key source="c_4" display-col="Keyword" label="Concept 2" order="3" group="Concepts" />
      <key source="c_4" display-col="Keyword" label="Concept 3" order="4" group="Concepts" />
      <key source="c_10" display-col="Date" label="" order="5" />
    </fk-keys>
  </table-multi-fk>

  <!-- Notes (child of Papers) -->
  <table-1fk id="c_6" name="Notes" theme="classic" grid="5,0">
    <columns>
      <col name="NoteTitle" type="string" visible="true" is-key="false" />
      <col name="Content" type="coop box" visible="true" is-key="false" />
      <col name="Attachment" type="media box" visible="true" is-key="false" />
    </columns>
  </table-1fk>

  <!-- References (step-child of Papers via Action Menu) -->
  <table-1fk id="c_7" name="References" theme="classic" grid="5,2">
    <columns>
      <col name="Reference" type="string" visible="true" is-key="true" />
    </columns>
  </table-1fk>

  <!-- Action Menu for Papers (accessed via F2) -->
  <menu id="c_8" name="Action Menu" theme="default" grid="4,2">
    <action-links>
       <a href="#c_9">Import from arXiv</a>
       <a href="#c_7">Paper References</a>
    </action-links>
  </menu>

  <!-- Dropzone Panel (import tool) -->
  <panel-dropzone id="c_9" name="Dropzone Panel" grid="4,4">
    <instructions>Drag%20or%20Paste%20an%20arXiv%20link</instructions>
    <options status-bar="false" progress-bar="true" stop-button="false" />
  </panel-dropzone>

  <!-- System Components (auto-hydrated on first boot) -->
  <table-1fk id="sys_scripts" name="Scripted Components" grid="-1,-1">
    <columns>
      <col name="Component Name" type="string" visible="true" is-key="true" />
      <col name="Action Line" type="string" visible="true" is-key="false" />
      <col name="Event" type="string" visible="true" is-key="false" />
      <col name="Script" type="js box" visible="true" is-key="false" />
    </columns>
  </table-1fk>

  <table-1fk id="sys_themes" name="Custom Themes" grid="-1,-2">
    <columns>
      <col name="Theme Name" type="string" visible="true" is-key="true" />
      <col name="Theme CSS" type="text box" visible="true" is-key="false" />
    </columns>
  </table-1fk>

  <table-1fk id="sys_help" name="Help Pages" grid="-1,-3">
    <columns>
      <col name="ID" type="string" visible="true" is-key="true" />
      <col name="Page Name" type="string" visible="true" is-key="false" />
      <col name="Content" type="html box" visible="true" is-key="false" />
    </columns>
    <children><child href="#sys_help_media" /></children>
  </table-1fk>

  <table-1fk id="sys_help_media" name="Help Media" grid="-1,-4">
    <columns>
      <col name="Media ID" type="string" visible="true" is-key="true" />
      <col name="File" type="media box" visible="true" is-key="false" />
    </columns>
  </table-1fk>
</app-schema>
```

---

# Part III — Generation Workflow
*Reasoning, executed.*

## Chapter 12: Step-by-Step Process

When you receive a user's free-form wish, follow this workflow. It merges 2.0's conceptual pipeline (Wish → Domain understanding → Entity extraction → Ownership analysis → Reference analysis → Navigation design → Schema generation → Themes → Scripts → Help → Validation) with 1.0's executable steps.

**Step 0 — Domain Understanding:** Before extracting a single entity, answer Chapter 2's Domain Discovery questions: who uses the app, what persists, what owns what, what merely references, which rich media are needed, which repetitive actions deserve scripts.

**Step 1 — Entity Extraction:** Read the wish and classify every object per Chapter 1's Mental Model (Actors, Persistent entities, Owned entities, Reference entities, Workflows, Automation opportunities). Look for:
- Nouns → Tables (e.g., "customers", "orders", "products").
- Adjectives/descriptors → Columns (e.g., "customer name", "order date", "price").
- Hierarchical phrases ("each customer has orders", "each order has items") → Parent-child cascades.
- Cross-referencing phrases ("orders reference a customer AND a product") → Multi-FK tables.
- Rich content needs ("notes", "attachments", "descriptions") → East Panel box types (`text box`, `media box`, `coop box`).

**Step 2 — Ownership Analysis (Hierarchy Design):** Apply Chapter 3's heuristics — exclusive ownership ⇒ `table-1fk` hierarchy. Organize owned entities into cascading chains (parent → child → grandchild …) for natural containment relationships. Watch for the smell: hierarchy used where references are appropriate.

**Step 3 — Reference Analysis (Multi-FK Design):** Multiple independent references ⇒ `table-multi-fk`. Identify standalone root tables for independent lookup/reference data (e.g., categories, tags, statuses), and Multi-FK tables for entities that aggregate data across multiple independent axes. Watch for the smell: Multi-FK used when ownership actually exists.

**Step 4 — Navigation Design:** Decide the landing component and navigation structure:
- A `<menu>` for simple apps with a few top-level sections.
- A `<tab-pane-top>` or `<tab-pane-side>` for apps with parallel equal-weight sections.
- Action Menus (`<action-menu>`) for contextual operations on specific tables.
- Menus point to hierarchy roots — never to leaf tables (see Chapter 3's architectural smells).

**Step 5 — ID Assignment:** Assign component IDs systematically per Chapter 14.

**Step 6 — Grid Layout:** Assign `grid="row,col"` coordinates:
- Landing component at `grid="0,0"`.
- Hierarchy roots in column 0, their children in columns 1, 2, 3, etc.
- Unrelated hierarchies on separate rows.
- System components at negative coordinates: `grid="-1,-1"`, `grid="-1,-2"`, etc.

**Step 7 — Schema Generation:** Produce the complete `app_schema.xml` following Chapter 5, applying the media-box design imperative from Section 5.5 wherever rich content was flagged in Step 1.

**Step 8 — Theme Generation:** If the user mentions any aesthetic preferences ("dark mode", "modern", "corporate blue"), produce a custom theme CSS following Chapter 7. If no preference is stated, use `<theme name="default" />`.

**Step 9 — Script Generation:** If the user describes any automation, data import, or custom behavior, produce hook scripts following Chapter 6.

**Step 10 — Help Generation:** Produce a help page set following Chapter 8:
- `index.html` — Directory listing all topics with links.
- One `topic.html` + companion `topic/` folder per major feature.
- Use `data-context` attributes to link help pages to specific components.

**Step 11 — Validation:** Run every artifact through Chapter 15's unified checklist before returning anything. The objective is not merely valid XML but an application that is intuitive to navigate.

## Chapter 13: Output Manifest

The agent must produce these files:

| File | Location | Description |
|---|---|---|
| `app_schema.xml` | Root output | The complete AppML schema. |
| Theme CSS snippets | Inline in docs or as separate `.css` files | To be pasted into `sys_themes` via the Themes Editor. |
| Script JS snippets | Inline in docs or as separate `.js` files | To be entered into `sys_scripts` via the Script Editor. |
| `index.html` | `resources/help/user/` | Help directory page. |
| `<topic>.html` | `resources/help/user/` | One per help topic. |
| `<topic>/` | `resources/help/user/` | Companion media folders (can be empty initially). |

## Chapter 14: ID Assignment Rules

| Category | ID Pattern | Example |
|---|---|---|
| User components | `c_N` (sequential from 1) | `c_1`, `c_2`, `c_3` |
| System scripts table | `sys_scripts` (fixed) | Always `sys_scripts` |
| System themes table | `sys_themes` (fixed) | Always `sys_themes` |
| System help table | `sys_help` (fixed) | Always `sys_help` |
| System help media | `sys_help_media` (fixed) | Always `sys_help_media` |

The landing component is typically `c_1`.

## Chapter 15: Unified Validation Checklist

Before delivering output, the agent must verify both the **architecture** (is this the right design?) and the **structure** (is the XML correct?).

**Tier 1 — Conceptual:**
- [ ] Navigation is natural — it mirrors how users think, not how tables happen to be stored.
- [ ] Every hierarchy has a single, clear owner.
- [ ] Every Multi-FK represents genuinely independent references, not disguised ownership.
- [ ] Menu/Tab links point to hierarchy roots or standalone tables — never to leaf (child) tables.

**Tier 2 — Structural:**
- [ ] Root `<app-schema>` tag wraps everything.
- [ ] `<application>` tag has `user-port` (e.g., 8081), `dev-port` (e.g., 8082 for maintainer), and `name` attributes.
- [ ] `<landing>` tag references a valid component ID using the `component` and `name` attributes (e.g., `<landing component="c_1" name="Main Menu" />`).
- [ ] Every component has a unique `id` and a unique `grid="row,col"` — no collisions anywhere in the schema.
- [ ] All `href` references (`#c_N`) resolve to components that exist in the schema.
- [ ] Each `<table-1fk>` has at most one `<child>` declaration.
- [ ] `<fk-keys>` source IDs point to valid `<table-1fk>` or `<date-picker>` components.
- [ ] `<fk-keys>` `display-col` values match actual column names in the source table.
- [ ] `<fk-keys>` `order` values are sequential and unique within the FK table.
- [ ] All column `type` values are valid: `string`, `number`, `boolean`, `date`, `time`, `text box`, `media box`, `coop box`, `js box`, `html box`.
- [ ] System components (if included) use the fixed IDs and negative grid coordinates.
- [ ] `sys_help` has a `<child>` pointing to `sys_help_media`.
- [ ] Help HTML files are raw snippets (no `<html>`, `<head>`, `<body>` wrappers).
- [ ] Help links use page slugs without `.html` extension.
- [ ] Help media references use filename only (no paths).
- [ ] The XML is internally consistent end to end.

## Chapter 16: Common Pitfalls

| Mistake | Effect | Fix |
|---|---|---|
| Menu links to a leaf table | Child insert ignores parent-row scope | Remove leaf from menu; reach it via Enter on parent row |
| Single-integer grid (`grid="0"`) | Invalid layout | Always use `grid="row,col"` |
| `href` typo (`#c_99` when `c_99` doesn't exist) | Component not found on boot | Cross-check all href references |
| FK `display-col` doesn't match source column name | Filter field shows blank values | Match `display-col` exactly to a `<col name="...">` in the source |
| Missing `<bounds>` tag on a table | No error, but geometry persistence won't work | Always include `<bounds grid-rect="" form-rect="" />` |
| Wrapping help HTML in `<html><body>` tags | Double-wrapped rendering | Use raw snippets only |
| Help media `src` includes a path | Image not found | Use filename only: `src="screenshot.png"` |
| Action Hook `Component Name` is "Action Menu" | Script does not execute (No custom script defined for action X on Y) | Set `Component Name` to the name of the **parent table**, not "Action Menu". |
| Action link for script has `href` attribute | Triggers navigation instead of the script | Omit the `href` attribute (e.g., `<a>Run Script</a>`). |
| Placing `app_schema.xml` in `/resources` directory manually | Dev environment mistakenly boots in user/deployment mode | **NEVER** place `app_schema.xml` in `/resources`. The IDE manages deployment. |

---

*3.0 merges the original Comprehensive Agent Development Guide (technical spec) with the 2.0 Philosophy addendum (reasoning layer) into a single reference. Assembled for SpexWorx — July 2026.*