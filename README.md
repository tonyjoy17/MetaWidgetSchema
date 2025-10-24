# MetaWidgetSchema
Zero‑code, schema‑driven form renderer. Upload a JSON schema and the UI is generated at runtime: a root form for scalar/object fields and separate dynamic list sections for every root‑level array (add/remove rows). All data stays client‑side, can be saved to localStorage, and exported as a schema‑projected JSON (no extra fields).
# Metawidget – Dynamic Schema Upload Form

Zero‑code, schema‑driven form renderer built using **Vanilla JavaScript** and **Metawidget Core**.

Upload a JSON schema, and the UI is automatically generated at runtime — including forms for object fields and dynamic sections for root-level arrays (add/remove rows).

---

## 🚀 Quick Start

1. Clone or download this repository.
2. Place `metawidget-core.min.js` next to `index.html`.
3. Open `index.html` in a browser.
4. Click **Load Schema** → choose a JSON schema file.
5. Edit the form, save to `localStorage`, or download the generated JSON data.

---

## ✨ Features

- Dynamic form rendering from uploaded JSON schema
- Automatic handling of object and array (list) fields
- Add/remove support for array items
- Save and load from `localStorage`
- Safe export (schema-projected JSON only)
- Lightweight, no frameworks required

---

## 🧠 How It Works

- **Inspector Logic**: Custom inspector hides array fields from the root form and renders them in their own sections.
- **Dynamic Lists**: Each root-level array becomes its own section with “Add” and “Remove” buttons.
- **Two-Way Binding**: Metawidget synchronizes the data model automatically.
- **Safe Export**: `projectToSchema()` ensures only valid schema-defined fields are exported.

---

## 📂 Project Structure

```
/
├─ index.html                 # All-in-one app (HTML, CSS, JS)
└─ metawidget-core.min.js     # Required Metawidget runtime
```

---

## 🛠 Example Schema

```json
{
  "properties": {
    "name": { "type": "string" },
    "email": { "type": "string" },
    "children": {
      "type": "array",
      "items": {
        "properties": {
          "name": { "type": "string" },
          "age": { "type": "number" }
        }
      }
    }
  }
}
```

---

## 🧩 Key Files and Functions

- `deepInitFromSchema()` – creates a blank model matching the schema
- `buildRootForm()` – generates the main form
- `renderAllLists()` – renders all dynamic list sections
- `projectToSchema()` – cleans and exports only valid fields
- `saveModel()` / `loadModel()` – handle persistence in localStorage

---

## 🧾 License

This project is for educational and demonstration purposes only.
