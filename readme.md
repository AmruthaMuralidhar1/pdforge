# PDForge

A fast, simple API to generate PDFs. Send your data and an HTML template — get a production-ready PDF back in milliseconds.

No sign-up friction. No dependencies to install. No headless browsers to manage. Just one HTTP call.

---

## Why PDForge?

Most PDF generation involves wrangling tools that are painful to install, slow to run, and fragile in production. PDForge takes all of that off your plate.

You send us an HTML/CSS template and your data as JSON. We render it through a real Chromium engine and return a pixel-perfect PDF. The whole round-trip takes about 200ms.

**Your data is never stored.** Every request is rendered in an isolated browser context and the result is streamed back to you immediately. Nothing hits disk. Nothing is logged. Once the response is sent, your data is gone.

---

## Quick Start

Generate a PDF in one call:

```bash
curl -X POST https://api.pdforge.dev/render \
  -H "Content-Type: application/json" \
  -d '{
    "html": "<h1>Hello {{ name }}</h1><p>Welcome aboard.</p>",
    "css": "body { font-family: sans-serif; padding: 40px; } h1 { color: #6366f1; }",
    "data": {"name": "Alice"}
  }' -o welcome.pdf
```

That's it. `welcome.pdf` is ready.

---

## Endpoints

### Generate a Single PDF

**`POST /render`** — Send an inline template + data, get a PDF.

```json
{
  "html": "<h1>Invoice for {{ client }}</h1>",
  "css": "body { font-family: sans-serif; }",
  "data": {"client": "Jane Smith"},
  "format": "A4",
  "landscape": false
}
```

**`POST /render/{template_name}`** — Use a saved template with your data.

```json
{
  "data": {"client": "Jane Smith", "amount": 4500},
  "format": "Letter"
}
```

### Generate Multiple PDFs (Batch)

Send an array of data objects. PDForge generates one PDF per object and returns them all in a single ZIP file.

**`POST /render/batch`** — Inline template + array of data.

```json
{
  "html": "<h1>{{ name }}</h1><p>{{ email }}</p>",
  "data": [
    {"name": "Alice", "email": "alice@acme.com"},
    {"name": "Bob", "email": "bob@acme.com"},
    {"name": "Carol", "email": "carol@acme.com"}
  ],
  "range": "1-2",
  "filename_key": "name"
}
```

This generates `Alice.pdf` and `Bob.pdf` (range `1-2` selects the first two), bundled in a ZIP.

**`POST /render/{template_name}/batch`** — Same thing, but using a saved template.

### Upload Files Instead of JSON

If you prefer to work with files, you can upload your `.html`, `.css`, and `.json` directly.

**`POST /upload/render`** — Upload files, get a single PDF.

```bash
curl -X POST https://api.pdforge.dev/upload/render \
  -F "html=@template.html" \
  -F "css=@style.css" \
  -F "data=@user.json;type=application/json" \
  -o output.pdf
```

**`POST /upload/batch`** — Upload files, get a ZIP of PDFs.

```bash
curl -X POST https://api.pdforge.dev/upload/batch \
  -F "html=@template.html" \
  -F "css=@style.css" \
  -F "data=@users.json;type=application/json" \
  -F "range=1-5" \
  -F "filename_key=name" \
  -o batch.zip
```

The JSON file must contain an array of objects for batch endpoints.

### Manage Templates

Save templates once, use them forever. No need to send the HTML/CSS on every request.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/templates/{name}` | Save a new template |
| `GET` | `/templates` | List all saved templates |

---

## Options

### Range Selection

Select which items to render in a batch. Supports any combination:

| Range | Result |
|-------|--------|
| `"1-5"` | First 5 items |
| `"3"` | Item 3 only |
| `"1,4,7"` | Items 1, 4, and 7 |
| `"1-3,8-10"` | Items 1 through 3, and 8 through 10 |

Ranges are 1-indexed. If omitted, all items are rendered.

### Filename Key

By default, batch PDFs are named `document_1.pdf`, `document_2.pdf`, etc. Set `filename_key` to a field in your data to name them automatically:

```json
"filename_key": "name"
```

If the data contains `{"name": "Alice Johnson"}`, the PDF is named `Alice_Johnson.pdf`.

### Page Format

| Format | Size |
|--------|------|
| `A3` | 297 x 420 mm |
| `A4` | 210 x 297 mm (default) |
| `A5` | 148 x 210 mm |
| `Letter` | 8.5 x 11 in |
| `Legal` | 8.5 x 14 in |
| `Tabloid` | 11 x 17 in |

Set `"landscape": true` for landscape orientation. Custom margins are supported via the `margin` field:

```json
"margin": {"top": "10mm", "bottom": "10mm", "left": "20mm", "right": "20mm"}
```

---

## Templates

Templates use [Jinja2](https://jinja.palletsprojects.com/) syntax inside standard HTML. Any valid HTML and CSS works — if it looks right in Chrome, it looks right in the PDF.

```html
<h1>Invoice #{{ invoice_number }}</h1>
<p>Client: {{ client_name }}</p>

<table>
  {% for item in items %}
  <tr>
    <td>{{ item.name }}</td>
    <td>${{ "%.2f"|format(item.price) }}</td>
  </tr>
  {% endfor %}
</table>

{% if notes %}
  <p>{{ notes }}</p>
{% endif %}
```

Full CSS3 is supported: Flexbox, Grid, custom fonts via Google Fonts, media queries, and print stylesheets.

---

## Privacy

PDForge does not store your data. Every request is processed in an isolated rendering context. Your HTML, CSS, JSON, and the generated PDF exist only in memory for the duration of the request. Once the response is sent, everything is discarded. No logs, no copies, no persistence.
