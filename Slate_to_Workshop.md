# Foundry Workshop in 2 Hours
### For a Python developer migrating a Slate + Phonograph dashboard

---

## Read this before minute 0

Two facts that shape your whole project:

1. **Phonograph is not a thing you migrate *to*. It is a thing you migrate *off*.** Phonograph is Object Storage v1 (OSv1), the Ontology's legacy backing store. Palantir's docs list it as *planned deprecation, unavailable after June 30, 2026* — a date that has already passed. Depending on your stack's version, your object types may already be forced onto Object Storage v2 (OSv2), or you may be on borrowed time.
   **Day-one task:** open **Upgrade Assistant** in your Foundry instance and look for the intervention named *"Migrate object types and many-to-many link types from Object Storage v1 to v2."* Whatever it says is the real deadline for your project, not this document.

2. **Your project is probably two migrations, not one.**
   - Migration A: Slate UI → Workshop UI
   - Migration B: Phonograph-backed data/writeback → OSv2-backed Ontology with Actions

   People scope only A, discover B in week three, and blow the estimate. Scope both in week one.

---

## Part 0 — The mental model (10 min)

Say this out loud until it feels true:

> **Slate is a canvas where you write code against data. Workshop is a configuration surface over a typed object model.**

Concretely:

| | Slate | Workshop |
|---|---|---|
| How you get data | You write a query (SQL, Phonograph, API) | You *declare an object set* |
| How you transform | You write JavaScript | You configure, or write a Function |
| How you render | HTML/CSS + widgets | Fixed widget library |
| How you write back | You call something | You invoke an **Action Type** |
| Where the logic lives | In the app | In the **Ontology** (shared, org-wide) |

**The single most important consequence:** Workshop has no query language. You cannot write `SELECT`. Every single thing a Workshop widget displays is an *object set* derived from another object set by filtering, traversing links, or aggregating.

So the migration question for every widget in your Slate app is:

> *Can this be expressed as a filter / traversal / aggregation over object sets?*
> - **Yes** → straightforward.
> - **No, but it's a computation** → write a Function.
> - **No, the data isn't shaped right** → change the pipeline or the ontology. This is the expensive answer.

Categorising your widgets into these three buckets *is* your migration estimate.

---

## Part 1 — The Ontology (25 min) — the part you said you never understood

### 1.1 The problem the Ontology solves

A Foundry **dataset** is basically a parquet file. It has columns and rows. It has no identity, no meaning, no relationships, no permissions at the row level, and no behaviour. Two teams looking at `/data/flights_clean` have to agree by convention on what a row means.

The **Ontology** is a typed, permissioned, shared, *organisation-wide* object layer sitting on top of those datasets. It turns "rows in a table" into "Flights that belong to an Aircraft and can be Cancelled."

It is not a database schema you own privately. It is a shared model. This matters for governance, and it's why you don't just go create object types unilaterally.

### 1.2 The mapping to Python

This table is the whole thing. Learn it and the confusion goes away.

| Foundry concept | Closest Python analogue | Notes |
|---|---|---|
| **Dataset** | a parquet file / DataFrame | Raw storage. No identity, no semantics. |
| **Object Type** | `class Flight:` — the class definition | The *type*. Lives in Ontology Manager. |
| **Object** | one instance of `Flight` | One row, but with identity and permissions. |
| **Property** | an attribute (`flight.status`) | Typed: string, double, timestamp, geo, array, struct, media, attachment. |
| **Primary key** | `__hash__` / a mandatory unique id | **Required.** Non-negotiable. See gotchas. |
| **Title property** | `__repr__` | What the object displays as everywhere in the platform. |
| **Link Type** | `flight.aircraft` and `aircraft.flights` | A first-class, bidirectional, traversable relationship. Replaces JOINs. |
| **Object Set** | a lazily-evaluated Django QuerySet | Composable, server-side, not materialised until rendered. **This is the currency of Workshop.** |
| **Action Type** | a method that mutates state, with validation + audit log | The only sanctioned way a user changes data. |
| **Function** | a module-level function or `@property` over objects | Written in TypeScript v2 or **Python**. |
| **Interface** | an ABC / `Protocol` | "Anything with a `status` and an `assignee`." Lets one widget work across object types. |
| **Derived property** | a computed `@property` | Configured, not coded, for simple cases. |

### 1.3 Object Set — internalise this one

An **object set** is *a definition, not data*. It is lazy. It is composed like this:

```
start with:      all Flight objects
filter:          status == "DELAYED"
filter:          departure_date within last 7 days
search around:   Flight → Aircraft          (traverse the link)
search around:   Aircraft → MaintenanceRecord
```

That's a five-step object set. In SQL it would be a three-table join with two predicates. In Workshop it is a variable you build in a dropdown-driven popover, and it is **indexed and fast**, because the Ontology maintains the indexes.

Set operations exist too: union, intersect, subtract two object sets.

Mentally: `ObjectSet` ≈ `QuerySet`. You chain, you don't execute. A widget executes it.

### 1.4 Action Types — the writeback story

In Slate, writeback is "run this thing that mutates data." In Foundry, writeback is an **Action Type**, defined in Ontology Manager, and it has real structure:

- **Parameters** — typed inputs (a string, a date, an object reference, a list of objects)
- **Rules** — what actually changes: create object / modify object / delete object / create link
- **Validation & submission criteria** — server-side conditions that must hold before submit
- **Permissions** — who may run it (separate from who may see the data)
- **Side effects** — notifications, webhooks, trigger a schedule build
- **Action log** — every invocation is audited and attributable

Actions can also be **function-backed** when the logic is too complex for rules — you write the edit logic in TypeScript or Python and the Action calls it.

If your Slate app writes back at all, **the Action Types are the hard part of your migration**, not the UI.

### 1.5 Functions — where your Python goes

You know Python well, so: Foundry Functions can be written in Python (as well as TypeScript v2). A Function can read object sets, compute, and return a value or return **Ontology edits**. Workshop can call them for:

- function-backed variables (a computed number, string, struct, or object set)
- custom aggregations that the built-in aggregation options can't express
- function-backed Actions
- function-backed widgets

**Constraint to respect:** Functions are for *interactive-latency* computation over modest data. They are not a place to reimplement a Spark job. Heavy aggregation belongs in the pipeline, materialised into a dataset, and surfaced as an object type.

---

## Part 2 — Workshop anatomy (20 min)

### 2.1 The three layers

```
LAYOUT      sections, tabs, containers, collapsible panels
   ↑
WIDGETS     read from and write to variables
   ↑
VARIABLES   the entire state model of your app
```

That's it. There is no fourth layer. If you understand variables, you understand Workshop.

### 2.2 Variables — the state model

Variable types you'll actually use:

- **Object set** — a set of objects (your bread and butter)
- **Object set filter** — a bundle of property/value filter pairs, produced by filter widgets
- **String / Numeric / Boolean / Date / Timestamp**
- **Array** — of any of the above
- **Struct** — a map of field → value, usually produced by a Function

And the ways a variable gets its value:

- **Static** — you type it
- **Object set definition** — object type + filters + link traversals
- **Object set aggregation** — count / sum / avg / min / max over an object set
- **Object property** — a property of one selected object
- **Function** — computed by a Function
- **Variable transformation** — chained operations referencing other variables (string concat, arithmetic, conditionals)

Variables form a **reactive dependency graph**, like a spreadsheet. Change an upstream variable and everything downstream recomputes. This *is* Workshop's programming model. There is no imperative script.

### 2.3 Widgets you need for a dashboard migration

| Widget | Use for |
|---|---|
| **Object table** | Your main grid. Selection produces a variable. |
| **Filter list** | Faceted filtering. Produces an object set filter variable. |
| **Metric card** | KPI numbers, backed by an object set aggregation |
| **Charts** | Bar / line / pie / scatter, aggregated server-side |
| **Object list / cards** | Prettier alternative to a table |
| **Button group** | Triggers Actions, navigation, and events |
| **Action form** | A full form for an Action Type |
| **Markdown / text** | Static and variable-interpolated copy |
| **Map** | Embed a Map template |
| **Time series** | Sensor / temporal data |
| **Embed Foundry app / iframe** | Escape hatch. Use sparingly. |
| **Nested module** | Compose modules; good for reuse |

### 2.4 Events

Widgets fire events — on load, on click, on selection change, on variable change — and events do things: set a variable, run an Action, run a Function, navigate to another page or module, open a dialog.

This is your replacement for Slate's event-driven JS. It is far more constrained. Accept that early.

### 2.5 The one gotcha that trips up every beginner

**Give your filter widget its own object set variable — do not point it at the same variable your table uses.**

If the Filter List and the Object Table share one object set variable, the filter narrows its own source and the UI behaves bizarrely (options disappear as you filter). The correct wiring is:

```
osv_all_flights        ← Flight (unfiltered)   → backs the Filter List
filter_flights         ← object set filter      ← produced by Filter List
osv_filtered_flights   ← osv_all_flights, filtered by filter_flights → backs the Table, Metric, Chart
```

---

## Part 3 — Hands-on lab (25 min)

Do not skip this. Twenty-five minutes of clicking beats two hours of reading.

Follow Palantir's official Workshop *Getting started* tutorial (it uses the `[Example Data] Flight Alert` object type, which most stacks have). If your stack lacks example data, substitute any object type you can already see in **Object Explorer**.

**Build this, in this order:**

1. Create a new Workshop module.
2. Add an **Object table**, back it with a new object set variable on some object type. → *You now understand object set variables.*
3. Add a **Filter list** on the left, with its **own** object set variable, filtering on 2–3 properties. → *You now understand the filter gotcha.*
4. Point the table's object set at the filtered variable. Filter something. Watch the table react. → *You now understand the reactive graph.*
5. Add a **Metric card** backed by an object set aggregation (count) of the same filtered set. Filter again. Watch the number move. → *You now understand aggregation variables.*
6. Add a **Chart** grouped by one property. → *You now understand server-side aggregation.*
7. Add a **Button group** with a navigation event. → *You now understand events.*
8. If you have permission: define a trivial **Action Type** in Ontology Manager (modify one property) and wire it to a button. → *You now understand writeback.*

If you complete step 8, you have touched every concept your migration needs.

---

## Part 4 — Slate → Workshop translation table (20 min)

You said you don't want to learn Slate. You don't need to *build* in it — but you do need to *read* the existing app well enough to inventory it. Here is the minimum mapping.

| What you'll find in the Slate app | Where it goes in Workshop |
|---|---|
| A **query** (SQL / Phonograph search / Phonograph aggregation / API) | Object set variable (filters + search arounds), or an object set aggregation variable, or a Function |
| A query that joins several datasets | **Link types** in the ontology, traversed via search around. If the links don't exist, you must create them. |
| A query doing heavy aggregation or windowing | Precompute in the pipeline → materialise as a dataset → expose as an object type. Do **not** try to do this in the app. |
| A **Slate variable** referenced via templating | A Workshop variable. But note: no arbitrary expression templating — use *variable transformations*. |
| A **JavaScript transform / function** | A **Function** (Python or TypeScript v2), or a *variable transformation* for simple arithmetic/string/conditional work |
| **HTML / CSS** widget, custom styling | Fixed widget library + Markdown widget. **Custom DOM does not survive the migration.** |
| **Writeback** (Phonograph writeback dataset, or an API call) | An **Action Type**, with parameters, rules, validation, and permissions |
| Conditional show/hide of widgets | Widget visibility bound to a boolean variable |
| **URL parameters / deep links** | Workshop URL parameters bound to variables |
| A "Refresh" button | Mostly obsolete — object sets read the live index. Freshness is a *pipeline/sync* concern now, not an app concern. |
| An embedded iframe | *Embed Foundry app* widget |
| Per-user or per-role visibility logic in JS | Ontology object security + Workshop module permissions. **Move it out of the app.** |

---

## Part 5 — Cautions (this is the section to reread)

**1. Confirm your Phonograph/OSv2 position before anything else.**
If the Slate app queries Phonograph or writes to a Phonograph writeback dataset, and those object types are still on OSv1, you have a mandatory data-layer migration underneath your UI migration. Check Upgrade Assistant.

**2. Migrating OSv1 → OSv2 can drop user edit history.**
Palantir's migration flow asks you to acknowledge that edit history (Actions) is not preserved unless you explicitly enable preservation, and that once done it can't be recovered from OSv1. If your dashboard's value depends on "who changed what, when," raise this *before* anyone clicks migrate. Decimal properties may also need explicit precision/scale set.

**3. There is no arbitrary JavaScript. None.**
This is the #1 estimate-killer. Inventory every line of custom JS in the Slate app early, and classify each: (a) reproducible by configuration, (b) needs a Function, (c) needs a pipeline change, (d) gets dropped. Get stakeholder sign-off on the (d) list up front.

**4. The ontology may not exist yet.**
If the Slate app reads datasets via SQL, there may be no object types at all. Building the ontology — object types, properties, links, primary keys — is frequently *the majority of the work*, and it's shared-governance work, not solo work. Check Object Explorer and Ontology Manager on day one and find out.

**5. Primary keys will hurt.**
Slate SQL doesn't care about uniqueness. Object types require a genuine unique primary key. Duplicate keys in your source data will surface as indexing failures or silently-collapsed rows. Run a `count(*)` vs `count(distinct pk)` check on every candidate dataset before you promise a timeline.

**6. Permissions are enforced differently, and users may see less.**
Ontology object security is real and granular (restricted views, multi-datasource object types, property-level permissions). A Slate app that ran broadly may have masked this. **Test with an actual end-user account, not yours.** "It works for me" is not a test result.

**7. Numbers will disagree, and you must reconcile them.**
Run Slate and Workshop side by side and reconcile every KPI. Mismatches almost always come from: null handling, timezone (`date` vs `timestamp` properties), deduplication, filter semantics (inclusive vs exclusive bounds), or the ontology sync lagging the dataset. Budget real time for this. It is the step that makes or breaks user trust.

**8. Don't promise pixel parity.**
Workshop is a design system with a fixed widget library. If the Slate app has bespoke layout, the Workshop version will look different. Set that expectation with stakeholders in the kickoff, not at UAT.

**9. Freshness semantics change.**
Slate queries could hit data directly. Workshop reads the Ontology index, which is populated by a sync from the backing dataset. There is latency between "pipeline built" and "visible in app." Understand and communicate that number.

**10. Watch aggregation and scale limits.**
High-cardinality group-bys, very large object sets, and expensive properties have limits. "Complex non-performant object properties" are explicitly not supported as Workshop variables. If you hit a wall, the answer is almost always *precompute upstream*, not *try harder in the app*.

**11. The iframe escape hatch is a trap.**
You *can* embed things. Every time you do, you give up Workshop's variable wiring, permissions integration, and maintainability. Treat it as a last resort you have to justify.

**12. Ontology changes are shared changes.**
Adding an object type is not a local decision. Search for an existing one first; duplicating `Customer` because you didn't look is a real and common failure. Use ontology branches/proposals if your org has them enabled.

---

## Part 6 — Your migration playbook, step by step

**Step 1 — Inventory.** Build a spreadsheet with one row per Slate widget/query/writeback/JS function. Columns: what it does, what data it touches, complexity, target Workshop equivalent, blocker.

**Step 2 — Classify.** Tag every row: `config-only` / `needs-function` / `needs-ontology-change` / `needs-pipeline-change` / `drop`. Your estimate is now defensible.

**Step 3 — Trace the data.** For every query, find the underlying dataset. Then check: does an object type already exist over it? Is it on OSv1 or OSv2? Who owns it?

**Step 4 — Design the ontology.** Object types, properties, links, primary keys. Review with whoever owns the ontology. Do not skip this review.

**Step 5 — Model writeback as Actions.** For every mutation the Slate app performs, define an Action Type: parameters, rules, validation, permissions, side effects.

**Step 6 — Build a vertical slice.** One page: one filter, one table, one metric, one action. End to end. Deploy it. Test performance and permissions with a real user. **This slice will reveal 80% of your unknowns for 10% of the effort.** Do it before committing to a full timeline.

**Step 7 — Build out.** Page by page. Keep the inventory spreadsheet updated as your source of truth.

**Step 8 — Reconcile.** Side-by-side numbers, every KPI, documented.

**Step 9 — UAT under real permissions.** With real users, on their accounts.

**Step 10 — Parallel run, then cut over.** Both apps live for an agreed period, then deprecate Slate. Don't delete it the same day.

---

## What to do in the week after these two hours

1. Open **Object Explorer** and browse your org's existing object types. Find the ones relevant to your dashboard. This is the fastest way to make the Ontology stop being abstract.
2. Open **Ontology Manager** on one object type and read every tab: Datasources, Properties, Links, Actions, Permissions.
3. Check **Upgrade Assistant** for the OSv1→OSv2 intervention.
4. Write one Python Function that reads an object set and returns a number. Wire it to a Workshop metric card. Now you've closed the loop between what you already know and what's new.
5. Read Palantir's *Ontology design: Best practices* and *Anti-patterns* pages before you propose any new object type.

---

## Glossary

- **AIP** — Foundry's AI layer; AIP Logic is a no-code way to build LLM-backed functions
- **Action Type** — a defined, validated, permissioned data mutation
- **Function** — server-side code (Python / TypeScript v2) callable from Workshop, Actions, and pipelines
- **Interface** — a structural contract across object types (like a `Protocol`)
- **Object Explorer** — ad-hoc UI for searching and exploring objects; great for learning
- **Object Set** — a lazy, composable, server-side set of objects
- **Ontology Manager** — where object types, link types, and action types are defined
- **OSv1 / Phonograph** — legacy Ontology backing store, being retired
- **OSv2** — current Ontology backing store
- **Search Around** — traversing a link type from one object set to another
- **Upgrade Assistant** — Foundry's tool listing migrations your org must complete
- **Workshop module** — one Workshop application

## Key documentation

- Workshop Getting started — `palantir.com/docs/foundry/workshop/getting-started`
- Workshop core concepts: Variables — `palantir.com/docs/foundry/workshop/concepts-variables`
- Ontology core concepts — `palantir.com/docs/foundry/ontology/core-concepts`
- Ontology design best practices / anti-patterns — under `palantir.com/docs/foundry/ontology/`
- Action types getting started — `palantir.com/docs/foundry/action-types/getting-started`
- Python Functions in Workshop — `palantir.com/docs/foundry/functions/python-functions-workshop`
- OSv1 → OSv2 migration — `palantir.com/docs/foundry/object-backend/osv1-osv2-migration`
- Breaking changes OSv1 → OSv2 — `palantir.com/docs/foundry/object-backend/object-storage-v2-breaking-changes`
