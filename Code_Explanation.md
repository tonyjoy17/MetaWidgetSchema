# Metawidget Schema Upload – Complete Code Explanation
*(Single, self-contained guide for developers with little prior experience)*

---

## 1) What this app does (plain English)

You open a simple web page (`index.html`). You upload a **JSON schema** (a JSON file that describes what fields you want). The page:

- **Generates a form automatically** for all non-array fields (e.g., `name`, `email`, `address.city`).  
- **Generates dynamic list sections** for each **root-level array** in the schema (e.g., `children[]`), where you can **Add** and **Remove** rows.  
- Keeps a single **data object (`model`)** updated live while you type.  
- Lets you **Save** to your browser’s `localStorage` and **Load** it later.  
- Lets you **Download** a cleaned JSON (it contains **only** fields defined by your schema).

The app uses **Metawidget** (a very small UI engine) to create inputs automatically from the schema. No frameworks needed (no React/Angular/Vue).

---

## 2) Why this is useful

- You can change which fields appear by changing the **schema**, not the code.  
- You can capture any number of items in list sections (like `children`), without writing extra HTML.  
- You get **safe export**: only the fields defined in the schema will appear in downloaded data (prevents accidental extra fields).

---

## 3) How to run it

1. Put **two files** in the same folder:  
   - `index.html` (your app)  
   - `metawidget-core.min.js` (the Metawidget runtime)
2. Open `index.html` in Chrome/Edge/Firefox.
3. Click **Load Schema** and choose a JSON file with a structure like this:

```json
{
  "properties": {
    "fullName": { "type": "string", "required": true },
    "email": { "type": "string" },
    "address": {
      "type": "object",
      "properties": {
        "city": { "type": "string" },
        "pincode": { "type": "string", "title": "PIN Code" }
      }
    },
    "children": {
      "type": "array",
      "items": {
        "properties": {
          "name": { "type": "string", "title": "Child Name", "required": true },
          "age": { "type": "number", "title": "Age" }
        }
      }
    }
  }
}
```

4. The page renders:  
   - **Form** (non-array fields)  
   - **Children** section with **Add** and **Remove** buttons  
5. Use **Save** (stores in localStorage), **Load** (restores), or **Download Data** (downloads cleaned JSON).

---

## 4) Mental model (big picture)

```
          [Your JSON Schema]
                    |
                    v
        [Inspector / Schema Reader]
                    |
      +-------------+-------------+
      |                           |
      v                           v
 [Root Form Builder]     [List Section Builder]
 (object/scalar fields)   (one section per array)
      |                           |
      +-------------+-------------+
                    |
                    v
               [ model ]
        (single JS object with
           your live data)
```

- **Inspector**: a function that tells Metawidget what fields exist at a given location in the schema.
- **Root Form**: non-array fields render here.
- **List Sections**: each root array gets a separate card with rows you can add/remove.
- **model**: all inputs write into this one object.

---

## 5) File anatomy (index.html)

Your `index.html` has three parts:

1) **HTML** – structure and buttons (Load, Reset, Save, Load, Download).  
2) **CSS** – light styling.  
3) **Script** – all the logic (helpers, rendering, persistence).

---

## 6) Key data structures in the script

- `schema`: the uploaded JSON schema (as a JS object).  
- `model`: the live data that mirrors the schema (strings, numbers, booleans, arrays, objects).  
- `listKeys`: array of root property names that are **arrays** with `items.properties`. (e.g., `['children']`)  
- `listItemPropsMap`: a lookup: `{ listKey -> items.properties }`, used to render each row in list sections.  
- `rootMW`: a Metawidget instance for the root form.  
- `listMWsMap`: maps `listKey` to an array of Metawidget instances (one per row).

---

## 7) Helper functions (what each one does)

### 7.1 `readJsonFile(inputEl)`
- Reads the selected file from the file input.  
- Parses it as JSON and returns the object.  
- If invalid, shows an alert and returns `null`.

### 7.2 `getSubschema(rootSchema, names)`
- Walks inside the schema following a path (like `['address', 'city']`).  
- If it finds an array with `items`, it dives into `items`.  
- Returns the subschema at that location or `undefined`.

> Used by Metawidget’s **inspector** to know what to render at each depth.

### 7.3 `deepInitFromSchema(node)`
- Creates a **blank** object that looks like the schema:
  - object  -> `{}` (recursively initialized)
  - array   -> `[]`
  - boolean -> `false`
  - other   -> `''` (empty string)
- This ensures inputs always bind to something defined (no undefined errors).

### 7.4 `detectListKeys(schemaObj)`
- Scans **root** properties.
- If a property is an `array` and has `items.properties` (fields per row), it’s a **list key**.  
- Returns something like `['children', 'skills']`.

### 7.5 `rootSchemaWithoutLists(schemaObj, keysToRemove)`
- Creates a shallow copy of the root schema **without** the list keys.  
- Reason: list keys render in their own sections; we don’t want duplicate labels in the root form.

> Alternative: instead of deleting properties, set `{ hidden: true }`—same visual effect, keeps schema intact.

### 7.6 `titleize(s)`
- Makes labels slightly nicer (capitalize first letter and add spaces for camelCase).

### 7.7 `projectToSchema(dataNode, schemaNode)`
- **Most important safeguard.**  
- Builds a new object by **whitelisting** fields declared in the schema:
  - Arrays: maps each element through `items` schema
  - Objects: includes only known `properties`
  - Missing scalars -> `''`, missing arrays -> `[]`
- Result: **only expected fields** get exported (**no extra/unknown fields**).

### 7.8 `cleanedModelForExport()`
- Convenience: `return projectToSchema(model, schema)`.

### 7.9 `refreshDebug()`
- Shows the current `model` in the **Raw Data** panel (pretty-printed JSON).

---

## 8) Building the UI (step by step)

### 8.1 Root Form: `buildRootForm()`

```js
rootMW = new metawidget.Metawidget(formHost, {
  inspector: (toInspect, type, names) => {
    const path = names || [];
    if (path.length === 0) {
      return rootSchemaWithoutLists(schema, listKeys);
    }
    const node = getSubschema(schema, path);
    return (node && node.properties) ? node : undefined;
  },
  layout: new metawidget.layout.TableLayout({ numberOfColumns: 2 })
});
rootMW.toInspect = model;
rootMW.buildWidgets();
```

- Creates a Metawidget instance on the `formHost` div.  
- **Inspector**:  
  - At **root**, returns the schema **without arrays** (so arrays are not rendered here).  
  - For nested paths, returns the appropriate subschema.  
- Uses a **2-column table layout** for labels and inputs.  
- Binds the live data object: `rootMW.toInspect = model`.

### 8.2 List Sections: `renderAllLists()`

This creates a section card for each `listKey`:

- Ensures `model[key]` is an array.  
- Builds a header with a **+ Add** button.  
- Renders rows with a **mini Metawidget per row**.  
- Adds a **Remove** button to delete rows.

Snippet: add-row and remove-row logic

```js
// Add-row
addBtn.onclick = () => {
  const blank = {};
  const defs = listItemPropsMap[key] || {};
  for (const k of Object.keys(defs)) {
    const t = defs[k]?.type;
    blank[k] = (t === 'boolean') ? false : '';
  }
  model[key].push(blank);
  renderThisList();
  refreshDebug();
};

// Remove-row
removeBtn.onclick = (e) => {
  e.preventDefault();
  model[key].splice(index, 1);
  renderThisList();
  refreshDebug();
};
```

- `renderThisList()` rebuilds just the rows for that list.  
- Each row has its own Metawidget bound to `model[key][index]`.

---

## 9) Persistence & export

### 9.1 `flushUI()`
- Calls `.save()` on widgets (Metawidget API) to push current input values into `model` before saving/downloading.

### 9.2 `saveModel()`
- Requires a loaded schema.  
- Calls `flushUI()` → gets `cleanedModelForExport()` → stringifies to JSON → saves to `localStorage` under a constant key.

### 9.3 `loadModel()`
- Requires a loaded schema.  
- Reads JSON from `localStorage`, parses it into `model`, then rebuilds the UI (`buildRootForm()` and `renderAllLists()`), and calls `refreshDebug()`.

> Tip: For extra safety, you can wrap parse in try/catch and normalize with `projectToSchema` after load.

### 9.4 `downloadJSON(name, obj)`
- Creates a Blob from JSON and triggers a browser download.

### 9.5 Wiring the buttons
- `Load Schema` → `loadSchemaFromUpload()` (reads file, sets `schema`, initializes `model`, detects list keys, builds UI).  
- `Reset` → re-initializes `model` to a blank shape and rebuilds UI.  
- `Save` → `saveModel()`  
- `Load` → `loadModel()`  
- `Download Data` → `flushUI()` + `cleanedModelForExport()` + `downloadJSON()`

---

## 10) Flow on first use (exact sequence)

1. User clicks **Load Schema** → selects a `.json`.  
2. Code parses JSON into `schema`.  
3. Code calls `deepInitFromSchema(schema)` to build a blank `model`.  
4. Code calls `detectListKeys(schema)` and fills `listItemPropsMap`.  
5. Code calls `buildRootForm()` → draws non-array fields.  
6. Code calls `renderAllLists()` → creates card(s) for each root array.  
7. **Raw Data** shows live `model`.  
8. User types values, adds/removes rows.  
9. User clicks **Save** or **Download Data** (data is first flushed and then projected to schema).

---

## 11) Design decisions (and alternatives)

- **Arrays as separate sections**: improves clarity; array fields don’t clutter the root form.  
  - *Alternative*: keep arrays visible in the root form and render rows inline. (More compact, but less clear.)

- **Projection on export** (`projectToSchema`): avoids over-posting and accidental data fields (a common security/data-integrity pattern).  
  - *Alternative*: export `model` as-is (simpler but risky).

- **Delete array fields from root schema copy** when rendering root: simple and effective.  
  - *Alternative*: keep them and add `{ hidden: true }` so the schema remains intact for downstream consumers.

- **One file, no build step**: easy to share and run.  
  - *Alternative*: split into modules and add tests/CI for production use.

---

## 12) Customization points you might need

- **Ignore a specific array**: Add a custom flag (e.g., `"x-hide-list": true`) and change `detectListKeys()`:

```js
if (def?.type === 'array' && def.items?.properties && !def['x-hide-list']) keys.push(k);
```

- **Field order**: Add and read `x-order` in your inspector and sort `Object.keys(schema.properties)`.

- **Validation**: Add AJV, run `ajv.validate(schema, model)` before Save/Download; display errors next to fields.

- **Titles/Help**: Use `title` and `description` from the schema to enrich labels and tooltips.

---

## 13) Accessibility (A11y) notes

- Buttons are labeled clearly (“Load Schema”, “Reset”, “Save”, “Load”, “Download Data”).  
- Consider adding `aria-live="polite"` to the Raw Data panel so screen readers announce updates.  
- Ensure “Remove” buttons have an accessible name that includes row index if needed.

---

## 14) Performance tips (if schema grows)

- Memoize `getSubschema()` results (same path requested often during builds).  
- Rebuild only the changed list (already done).  
- For very large arrays (1000+ rows), add **virtualization** (render only visible rows).  
- Consider a Proxy to update the debug panel automatically (optional nicety).

---

## 15) Troubleshooting & common pitfalls

- **Nothing renders after choosing a file**  
  - The JSON is invalid or doesn’t have a top-level `properties` object.  
  - The file input is empty (choose a file).

- **“metawidget-core.min.js not found” error**  
  - Ensure the file sits **next to** `index.html`. The `<script src="./metawidget-core.min.js">` path is relative.

- **Array renders but row fields are empty**  
  - Check the schema has `items.properties` (each row needs its own fields).

- **Download includes unexpected fields**  
  - Confirm you’re using `projectToSchema` output, not the raw `model`.

- **Saved data won’t load**  
  - Clear `localStorage` or wrap `JSON.parse` in try/catch; optionally normalize with `projectToSchema` on load.

---

## 16) Frequently Asked Questions (FAQ)

**Q. Why does the `children` array show up as its own section?**  
Because the app treats every root-level `array` with `items.properties` as a repeatable list UI. It’s by design so the user can add/remove rows.

**Q. Can I hide a specific array section?**  
Yes—add a custom flag like `"x-hide-list": true` to that array in the schema and update `detectListKeys()` to skip it.

**Q. How do I enforce required fields or number ranges?**  
Add standard JSON Schema keywords (e.g., `required`, `minimum`, `format`) and wire a validator (AJV) on Save/Download.

**Q. Can this run offline?**  
Yes. Everything is client-side. No server needed.

**Q. Can I integrate this with an API?**  
Yes—after `projectToSchema(model, schema)`, send the JSON to your API using `fetch`.

---

## 17) Glossary (quick terms)

- **JSON Schema (lightweight here)**: We’re using the minimal idea: `properties`, `type`, and for arrays: `items.properties`.  
- **Metawidget**: A small library that builds UI widgets from metadata (our schema).  
- **Inspector**: A function Metawidget calls to get the schema at the current path.  
- **Model**: The single JS object that stores all current form values.  
- **Projection**: Creating a new object that contains only fields allowed by the schema.

---

## 18) Key code snippets (refer back quickly)

**Root inspector** (hide arrays from main form):  
```js
inspector: (toInspect, type, names) => {
  const path = names || [];
  if (path.length === 0) {
    return rootSchemaWithoutLists(schema, listKeys);
  }
  const node = getSubschema(schema, path);
  return (node && node.properties) ? node : undefined;
}
```

**Add row**:  
```js
const blank = {};
const defs = listItemPropsMap[key] || {};
for (const k of Object.keys(defs)) {
  const t = defs[k]?.type;
  blank[k] = (t === 'boolean') ? false : '';
}
model[key].push(blank);
renderThisList();
refreshDebug();
```

**Remove row**:  
```js
model[key].splice(index, 1);
renderThisList();
refreshDebug();
```

**Safe export**:  
```js
const clean = projectToSchema(model, schema);
downloadJSON('data-export.json', clean);
```

---

## 19) Summary (what to remember)

- Upload schema → app builds UI automatically.  
- Arrays become their **own sections** with add/remove.  
- Everything binds to `model`.  
- Export is **schema-projected** (only allowed fields).  
- Save/Load use `localStorage`.  
- Easy to extend with validation, ordering, flags like `x-hide-list`.

That’s it—you can now confidently explain and modify this app, even with limited prior experience.
