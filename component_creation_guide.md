# SpexWorx Component Implementation Guide

This guide outlines the end-to-end process for implementing a new component type within the SpexWorx architecture, covering both the Developer IDE (`/dev`) and the Runtime Application (`/user`), as well as the Java backend.

## 1. Developer UI Registration (`/dev` frontend)

To make the component available in the developer workspace:

### a) Register the Component Type
Update `resources/dev/js/components.js` to define your new component (assigning it a type string like `my-new-comp`).

### b) Add to the Palette
Update `resources/dev/js/palette.js` to add an icon/button to the East Palette so the developer can drag or click to place it on the grid.

### c) Wire the Interaction (Grids)
Update `resources/dev/js/Grids/grid_2d.js` (and `grid_25d.js` if applicable). In the `contextmenu` (right-click) event listener for the grid items, add a branch for your new component type to launch its specific Editor:
```javascript
else if (comp.type === 'my-new-comp') {
    if (window.MyNewCompEditor) window.MyNewCompEditor.open(comp);
}
```

## 2. Component Editor (`/dev` frontend)

The Editor is responsible for configuring the component's properties and persisting them to the database during development.

### a) Create the Editor Class
Create a new file under `resources/dev/js/Editors/` (e.g., `MyNewCompEditor.js`). 
- It must implement an `open(component)` static method to launch a lightbox/modal.
- It should fetch existing properties via `GET /dev/components/mycomp?id=...`.
- It must provide UI for configuring properties (Name, instructions, specific options).
- It must persist changes via `POST /dev/components/mycomp`.
- It must handle deletion via `DELETE /dev/components/mycomp`.

### b) Include the Editor Script
Make sure to add the `<script src="/js/Editors/MyNewCompEditor.js"></script>` tag in `resources/dev_index.html`.

## 3. Backend Developer API (Java)

You need to provide the REST endpoints for the Editor to save and load properties.

### a) Create the Handler
Create a new Java class implementing `HttpHandler` (e.g., `src/root/server/dev/components/MyCompHandler.java`).
- Implement `handleGet` to fetch properties from the DB.
- Implement `handlePost` to insert/update properties (and potentially update the name in `workspace_components`).
- Implement `handleDelete` to remove the component and cascade deletions to related properties/links.

### b) Database Initialization
Inside your Handler, include a static `initDB(Connection conn)` method to create your specific properties table (e.g., `mycomp_properties`).

### c) Register the Handler
In `src/root/server/ServerMain.java`:
- Call your `initDB` method during startup.
- Register the HTTP context: `server.createContext("/dev/components/mycomp", new MyCompHandler(Database::getConnection));`

## 4. AppML Schema Integration (Definitely required)

If the component needs to be persisted into the `app_schema.xml` generation, update `resources/dev/js/Editors/AppML/AppMLEditor.js`. Ensure that the XML generator recognizes your component type and correctly serializes its properties from the DB into XML nodes/attributes.

## 5. Runtime Operator (`/user` frontend)

This is the class that actually renders the component for the end-user.

### a) Create the Operator Class
Create a new file under `resources/user/js/Operators/` (e.g., `MyCompOperator.js`).
- It must expose an `async render(node, container, parentContext = null)` method.
- **Data Hydration**: Fetch properties from the DB via `/api/component/properties`. If the backend DB doesn't exist or isn't hydrated, read fallback properties directly from the embedded AppML XML `node` (seeds).
- **Layout Math**: Implement initial layout logic. Use **Breadcrumb Flow** (relative to `parentContext`'s active row) if launched from a menu/table, or **Centering** if it's a standalone modal/dropzone.
- **DOM Construction**: Build the HTML, apply the correct `theme` classes, and append it to the `container`.
- **State Management**: Listen to `user-state-changed` events to update its visual stack depth (`zIndex`, `boxShadow`, blur) and apply the `inactive` class when it is pushed back in the stack.

### b) Wire the Operator
Update `resources/user/js/app.js`:
- Import your new Operator.
- Inside the main component routing `switch` statement (which reads the component type), instantiate your Operator and call `.render(...)`.

## 6. Runtime Backend API (Java)

If your component requires fetching dynamic data at runtime (beyond just its static configuration properties), you will need to add new endpoints in the `src/root/server/api/` directory (e.g., `TableDataHandler.java`) and register them in `ServerMain.java`. 

*(Note: The standard `/api/component/properties` endpoint used by panels already exists and serves properties out of the DB, so you may not need a new endpoint for basic configuration fetching).*
