# 🌐 SpexWorx

> **A self-replicating web IDE and runtime where Documents, Data, and Code live together as equal citizens.**

---

## What Is This?

Every computer program in the world is built from three ingredients:

1. **📄 Documents** — things you read (text, web pages, photos, videos).
2. **📊 Data** — things you organize into tables (names, prices, inventories).
3. **⚙️ Code** — the rules that make things *happen* ("when this button is clicked, do something").

Most software keeps these three separated. You write your code in one tool, manage your data in another, and view your documents somewhere else entirely.

**SpexWorx puts all three in the same place.** A piece of JavaScript code sits right inside a database cell—alongside names, images, and HTML pages—and it plugs directly into the system's event life cycle. You can literally store a [playable pickleball game](docs/assets/pickle_01.js) as a text field in a table row, and the system knows how to run it.

---

## A Bit of History

This architecture was first implemented roughly **three decades ago** to solve a practical headache: tedious SQL database programming. The idea was reimplemented in its current form using modern Java and JavaScript.

Along the way, platforms like **Notion**, **Airtable**, and **Bubble.io** appeared. They are excellent tools, and in the first approximation, they all live in the same "no-code / low-code" space. Bubble in particular comes close in ambition—the visual UI builder approach is different, but the spirit is similar.

Where SpexWorx diverges is in how it treats **code**:

| | Traditional No-Code Platforms | SpexWorx |
| :--- | :--- | :--- |
| **Documents** | ✅ Managed (pages, rich text, media) | ✅ Managed |
| **Data** | ✅ Managed (tables, relations, queries) | ✅ Managed |
| **Code** | ⚠️ Bolted on (webhooks, proprietary workflows, hidden server functions) | ✅ **First-class citizen** — stored in data cells, connected to lifecycle hooks |

The Web (1990s) automated the linking and hosting of **documents**. Modern no-code platforms automated the management of **data** (and sometimes documents). SpexWorx adds the third piece — it gives **code** the same addressable, navigable treatment. Scripts attach to specific events, data rows, or document lifecycles through a [Hook Switchboard](resources/user/js/Runtime/Hooks/HookSwitchboard.js), completing the set.

---

## How It Was Built (One Thing Led to Another)

```
  Making database rows "active" (self-fetching)
       ↓
  Cascading parent→child UI stacks appeared naturally
       ↓
  The server started mirroring every user's UI stack (real-time sync)
       ↓
  Complex multi-table editing got a visual "key appliance" instead of custom SQL
       ↓
  Applications became compact XML descriptions (AppML) instead of programs
       ↓
  Table cells expanded to hold text, HTML, media — and executable scripts
       ↓
  A Hook Switchboard connected scripts to insert/update/delete lifecycle events
       ↓
  The IDE learned to clone its own JAR + inject a new schema = self-replication
```

---

## Quick Start

### Option A: Run the Pre-Built JAR (Easiest)
1. Make sure you have **Java 17+** installed.
2. Download `SpexWorx.jar` from the [Releases](../../releases) page (or from the `build/` folder).
3. Double-click the JAR or run:
   ```
   java -jar SpexWorx.jar
   ```
4. On first launch, SpexWorx opens a setup window and **generates its own startup script** for your OS (`.bat` for Windows, `.sh` for macOS/Linux). Use that script for subsequent launches.

### Option B: Build from Source
1. Make sure you have a **Java JDK 17+** installed (it will be auto-detected).
2. Run:
   ```
   compile_for_GitHub.bat
   ```
3. The output JAR will be at `build/SpexWorx.jar`.

---

## Key Concepts

### The Three Citizens
- **Documents**: HTML pages, help files, and media stored in system tables (`sys_help`, `sys_help_media`).
- **Data**: Relational tables with full CRUD, multi-foreign-key support, and cascading navigation.
- **Code**: JavaScript scripts stored as table cell content. See [`docs/assets/pickle_01.js`](docs/assets/pickle_01.js) — a complete multiplayer game engine living inside a data row.

### The Hook Switchboard
Scripts don't float in isolation. The [HookSwitchboard](resources/user/js/Runtime/Hooks/HookSwitchboard.js) connects them to real system events — row inserts, updates, deletes, component activations. When something happens, the switchboard pulls the script from the database and runs it with full context.

### Self-Replicating Architecture
Click **"Generate App"** in the Dev IDE and the engine clones its own running Java process, injects the new AppML schema, and produces a standalone executable JAR. That new JAR boots up with two servers:
- **User Port** (default 8081): The live end-user application.
- **Dev Port** (default 8082): A full copy of the visual IDE — allowing further customization or generating *yet another* app.

### Zero External Dependencies
The backend is raw Java with an embedded H2 database. No Spring, no Node, no Docker required. One JAR file, one `java -jar` command.

---

## Documentation

Coming soon
https://youtu.be/68sF-4wijSs
---

## License

SpexWorx is released under the **[MIT License](LICENSE)**.

### Open Source Dependencies
- **[Tabulator](https://tabulator.info/)** (MIT) — Client-side data grids.
- **[CodeMirror 6](https://codemirror.net/)** (MIT) — In-browser code editor.
- **[H2 Database Engine](https://h2database.com/)** (MPL 2.0 / EPL 1.0) — Embedded Java SQL database.
