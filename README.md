# 🌐 SpexWorx

> **A web IDE and runtime where Documents, Data, and Code live together.**

SpexWorx is a small, self-contained platform for building web applications.

Its unusual idea is simple:

**Documents, Data, and Code are all treated as things an application can store, display, connect, and run.**

---

## What does that mean?

Most software keeps these things in different worlds:

* 📄 **Documents** — pages, text, images, media
* 📊 **Data** — tables, records, relationships
* ⚙️ **Code** — programs and rules

SpexWorx puts them together.

For example, a database cell can contain a JavaScript program. That program can be connected to an application event and executed by the runtime.

A complete multiplayer [pickleball game](docs/assets/pickle_01.js) can therefore exist as **a piece of data in a table** — and still be a running program.

That is the central idea behind SpexWorx.

---

## Why?

The Web made documents easy to **publish and link**.

Databases made data easy to **store and organize**.

No-code platforms made application construction easier by putting these capabilities behind visual tools.

SpexWorx asks a slightly different question:

> **What if Code were treated as another kind of application data?**

In SpexWorx, code can have an address, live in a table, move with an application, and be attached directly to the application's events.

---

## A Little History

The architecture behind SpexWorx began roughly **three decades ago**, originally as an attempt to make database application development less dependent on tedious SQL programming.

The current implementation is a modern reimplementation using **Java and JavaScript**.

The system evolved through a sequence of fairly practical ideas:

```text
Make database rows active
        ↓
Rows begin fetching their own children
        ↓
Parent → child UI stacks appear
        ↓
The server mirrors users' UI stacks
        ↓
Complex database editing becomes visual
        ↓
Applications become compact XML descriptions
        ↓
Table cells can contain HTML, media and scripts
        ↓
Scripts become connected to application events
        ↓
The IDE can generate a new application from itself
```

None of this was originally designed as one grand architecture.

**One thing simply led to another.**

---

## The Three Citizens

### 📄 Documents

HTML pages, help text and media can live inside application tables.

### 📊 Data

Relational tables provide ordinary application data, including CRUD operations, relationships and cascading navigation.

### ⚙️ Code

JavaScript programs can be stored as data.

The [Hook Switchboard](resources/user/js/Runtime/Hooks/HookSwitchboard.js) connects those programs to application events such as inserting, updating or deleting a row.

So instead of:

```text
database → application code → user interface
```

SpexWorx can treat the pieces more like:

```text
        ┌───────────────┐
        │  DOCUMENTS    │
        ├───────────────┤
        │     DATA      │
        ├───────────────┤
        │     CODE      │
        └───────────────┘
              ↕
          Runtime
```

They are different kinds of content, but they live in the same application world.

---

## Self-Replicating Applications

SpexWorx can generate a new application from the running IDE.

Click **Generate App** and the system:

1. takes the running engine,
2. adds the application's schema,
3. creates a new standalone JAR,
4. and produces an application that can run by itself.

The generated application contains two servers:

* **User Port** — normally `8081`, serving the application
* **Dev Port** — normally `8082`, providing the development IDE

The interesting part is that the generated application still contains the machinery needed to **generate another application**.

In other words:

> **The tool that builds the application is itself part of the application.**

---

## Quick Start

### Option A — Run the JAR

You need **Java 17+**.

Download `SpexWorx.jar` from [Releases](../../releases), then run:

```text
java -jar SpexWorx.jar
```

On first startup, SpexWorx creates a startup script for your operating system.

Use that script for subsequent launches.

### Option B — Build It Yourself

Install a **Java JDK 17+**, then run:

```text
compile_for_GitHub.bat
```

The resulting JAR will be:

```text
build/SpexWorx.jar
```

---

## No Framework Stack

SpexWorx deliberately keeps the runtime small.

* Java backend
* JavaScript frontend
* Embedded H2 database
* One JAR

No Spring.
No Node.js.
No external database server.
No Docker required.

**Download → run → build an application.**

---

## See It

Documentation is still being assembled.

For now, here is a short video demonstration:

https://youtu.be/68sF-4wijSs

---

## Open Source

SpexWorx is released under the **[MIT License](LICENSE)**.

It also uses:

* [Tabulator](https://tabulator.info/) — MIT
* [CodeMirror 6](https://codemirror.net/) — MIT
* [H2 Database Engine](https://h2database.com/) — MPL 2.0 / EPL 1.0
