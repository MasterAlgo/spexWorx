# WishTool Project Memo

## The Big Idea & Purpose
SpecWorx is a self-replicating, lightweight, dependency-free Java/JS web IDE and application server. Its primary goal is to provide a unified Developer/Admin environment that can visually architect an application and then instantly spin it out as a standalone executable. 

The core feature is a collaborative, real-time 2D/2.5D visual workspace where developers drag, drop, lock, and configure UI components. These components are compiled into a declarative XML-like schema (`AppML`). When the developer clicks "Generate App", the backend clones its own executing Fat JAR, injects the `AppML` schema into it, and produces a ready-to-deploy executable. 

Because the generated app contains the exact same engine, it boots up with a **Dual-Server Architecture**:
1. **User Application Server (e.g., port 8081)**: Interprets the `AppML` schema to dynamically render the live User UI.
2. **Dev/Admin Server (e.g., port 8083)**: Exposes the full original Dev IDE, allowing the generated app to be further customized or even used to generate yet another app (a true fractal application ecosystem). 

Ultimately, this fractal architecture means the app's structural lightboxes (Menus -> Submenus) effectively act as a self-hosted directory tree. Just like a file system, these hierarchical UI menus will route down into specific data containers capable of holding raw text, HTML, Java, or JS payloads—turning the app into its own infinitely expandable operating system/IDE.

## Core Architectural Principles
1. **Zero External Dependencies (Backend)**: Uses raw Java `HttpServer` and standard libraries. Avoids heavy external parsers, relying on lightweight custom text formats.
2. **Relational Database**: Uses an embedded H2 database (`wishtool.mv.db`). The generated app spins up a fresh, blank-slate database upon its first boot.
3. **Dual Endpoints**: Logical split into Developer endpoints (`/dev`) and User endpoints (`/api`), with robust token-based authentication.
4. **Real-time Collaboration**: The dev grid leverages a polling/heartbeat mechanic (`WORKSPACE_UPDATED` broadcasts) with component locks (`locked_by`) to prevent collisions.
5. **Global Stylings**: Avoid placing theme/color controls inside individual component editors. Keep them centralized to ensure styles are aligned across the project.
6. **Clever UX over Clunky UI**: Solve complex UI problems cleanly (e.g., using the drag handle as a double-click Hashtag Tip selector).
7. **Isolated DB Instantiation (Frontend-Driven Hydration)**: The Dev IDE never populates the User App's database. The User App spins up with a blank database. To hydrate it, the architecture relies entirely on the User UI acting as the orchestrator:
   - *Step 1*: The User UI fetches the generated `app_schema.xml` on boot.
   - *Step 2*: When a component is activated (e.g. Table), its JS Operator fires a `GET` request to the backend to check for existing state.
   - *Step 3*: If the backend DB is empty, the JS Operator falls back to parsing its configuration nodes directly from the XML schema.
   - *Step 4*: The Operator packages the schema intent into a massive `POST` payload.
   - *Step 5*: The backend Handler intercepts this `POST` and dynamically executes `CREATE TABLE IF NOT EXISTS` to instantiate the necessary tables, then hydrates them with the payload.
   *(See `resources/help/maintainer/user_db_instantiation.html` for a complete developer breakdown of this mechanic).*

## Access Control Architecture: 2 Executables, 3 Domains, 6 Roles
The platform utilizes a self-bootstrapping hierarchy of permissions spanning the application's lifecycle:

**Executable 1: The Dev Environment (IDE)**
- **Domain 1 (Dev):** 
  - **Admin-Dev:** The first person to log in; approves/denies Devs.
  - **Dev:** Regular developers who cooperatively develop and generate the end-user app (JAR).

**Executable 2: The Generated App (Standalone JAR)**
The generated JAR opens two designated ports for serving different domains (e.g., 8081 for Users, 8082 for Maintainers).
- **Domain 2 (Users):**
  - **Admin-User:** The first user to log into the User port; approves/denies Users.
  - **User:** Standard end-users.
- **Domain 3 (Maintainers):**
  - **Admin-Maintainer:** The first person to log into the Maintainer port; approves/denies Maintainers.
  - **Maintainer:** Approved maintainers who manage the live app.

## Current State of Development (Dev IDE Phase - COMPLETE)
- **Database & Auth**: Full authentication system, admin roles, and session persistence are functional. 
- **Grids & Rendering**: 2D and isometric 2.5D grids are fully rendered and synced. Connections are highlighted dynamically.
- **Component Editors**: `MenuEditor`, `MenuLinkEditor`, `TabPane`, `TableEditor`, `TableLinkEditor`, `DatePicker`, `Dropzone`, and `CoopBox` are fully operational.
- **System Components**: `sys_scripts`, `sys_themes`, `sys_help`, and `sys_help_media` integration is complete, allowing automatic hydration of system resources like help documents on first boot.
- **AppML Schema Engine**: The IDE's AppML Tab provides full bidirectional Pull/Deploy capabilities, completely abstracting away SQL into a formal, declarative, interaction-first visual schema (`app_schema.xml`).
  - *Recent Fixes*: Hardened XML tag generation and robust port injection mechanics to perfectly sync `Admin` tab parameters (like `dev-port` and `user-port`) dynamically before `Generate App`.
- **App Deployment ("Generate App")**: Fully implemented the standalone JAR generation via `GenerateAppHandler`. 
- **Boot Sequence & Persistence**: Implemented a "Factory Reset" guard in `ServerMain.java` that intelligently detects if the Dev DB (`table_properties`) is already populated, skipping eager XML parsing to ensure Maintainer UI layouts persist seamlessly across server reboots.

## Current State of Development (User App Phase - IN PROGRESS)
- **Dual-Server Setup & User Auth**: Fully operational. The generated app reads `app_schema.xml` to dynamically assign its `user-port` and `dev-port`.
- **Interpreter Core**: `app.js` and `SchemaHandler` parse the frontend XML and route to the landing component.
- **Interactive UI Mechanics**:
  - **Draggable & Resizable Layouts**: Components can be freely moved and resized on the User UI. 
  - **Admin Geometry Saving**: F9 interception elegantly saves live `left,top,width,height` geometries to the database.
  - **Theme Engine**: Pure CSS variable framework (`themes.css`) with global persistence.
  - **User Heartbeat**: Real-time server connection polling with a matching "pulsing" green/red dot UI.
- **Runtime Commander & State Broker**: 
  - **State-Driven Multi-Tier Cascade**: Nested submenus now spawn exactly beneath their parent's active "breadcrumb" row. Geometry is calculated using bulletproof `getBoundingClientRect` DOM boundaries to ensure pixel-perfect, non-overlapping layouts regardless of font or scaling.
  - **Dynamic Breadcrumbs**: When a child menu spawns, the parent menu goes `.inactive`, instantly fading its title and collapsing all inactive rows. Crucially, the active breadcrumb row grays out (Classic Windows style) so focus remains entirely on the active child window.
  - **Theme Focus States**: Expanded the CSS engine to support `--theme-item-inactive-active-bg` and `--theme-title-inactive-color`, providing rich visual distinction between active and inactive cascade windows across all standard themes (Glassy Blue, Default Dark, Classic Light, Solarized Dark).
- **Table Mechanics**:
  - **Dynamic Schema Bootstrapping**: To maintain the "Isolated DB" principle, tables in the user app parse their schema directly from the embedded `app_schema.xml` on first load. The JS `TableOperator` extracts the columns and performs a heavy `POST` to `UserTablePropertiesHandler.java`, which dynamically executes `CREATE TABLE IF NOT EXISTS` for `table_properties` and `table_columns` to hydrate the user database.
  - **Auto-Fitting Geometry**: The Tabulator grids dynamically calculate row heights based on CSS themes (`--theme-font-size`) and aggressively snap the component window height during resize. This guarantees pixel-perfect integer heights, preventing empty dead space or ugly scrollbars.
  - **Multi-FK Tables**: Fully implemented support for tables with multiple foreign keys pointing to the exact same source component (e.g. `Author` -> `First Author`, `Last Author`), resolving previous database `UPSERT` squashing bugs via precision `INSERT` and `order_idx`-based unlinking.
- **Hook Switchboard & Custom Scripting**: Fully implemented, providing a comprehensive list of hooks to user scripts (`resources/user/js/Runtime/Hooks/HookSwitchboard.js`).

## Next Steps
*(To be determined in the current session)*

---

### 🛑 CRITICAL DIRECTIVE: THE "NO CODING" RULE
*IMPORTANT: Never implement until ordered. Provide advice, analysis, and architecture design when asked, but DO NOT modify code, create implementation plans, or run scripts without explicit instructions to proceed. Wait for the green light on every single step.*
