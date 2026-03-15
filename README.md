# PDForge

A fast, simple API to generate PDFs. Send your data and an HTML template — get a production-ready PDF back in milliseconds.

No sign-up friction. No dependencies to install. No headless browsers to manage. Just one HTTP call.

---

## Why PDForge?

Most PDF generation involves wrangling Puppeteer, wkhtmltopdf, or WeasyPrint — tools that are painful to install, slow to run, and fragile in production. PDForge takes all of that off your plate.

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

## All Endpoints

| Method | Endpoint | Input | Output | Description |
|--------|----------|-------|--------|-------------|
| `POST` | `/render` | JSON body | PDF | Inline template + data → single PDF |
| `POST` | `/render/batch` | JSON body | ZIP | Inline template + array of data → ZIP of PDFs |
| `POST` | `/render/{name}` | JSON body | PDF | Saved template + data → single PDF |
| `POST` | `/render/{name}/batch` | JSON body | ZIP | Saved template + array of data → ZIP of PDFs |
| `POST` | `/upload/render` | File upload | PDF | Upload .html, .css, .json → single PDF |
| `POST` | `/upload/batch` | File upload | ZIP | Upload .html, .css, .json (array) → ZIP of PDFs |
| `POST` | `/templates/{name}` | JSON body | JSON | Save a reusable template |
| `GET` | `/templates` | — | JSON | List all saved templates |
| `GET` | `/health` | — | JSON | Health check (returns status, version, engine) |

---

## Endpoints in Detail

### Single PDF — JSON Body

**`POST /render`** — Send an inline template + data, get a PDF back.

```json
{
  "html": "<h1>Invoice for {{ client }}</h1>",
  "css": "body { font-family: sans-serif; }",
  "data": {"client": "Jane Smith"},
  "format": "A4",
  "landscape": false
}
```

**`POST /render/{template_name}`** — Use a saved template instead. Just send the data.

```json
{
  "data": {"client": "Jane Smith", "amount": 4500},
  "format": "Letter"
}
```

### Batch PDFs — JSON Body

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

**`POST /render/{template_name}/batch`** — Same thing, using a saved template.

```json
{
  "data": [
    {"client_name": "Alice", "items": [{"name": "Service", "qty": 1, "price": 500}]},
    {"client_name": "Bob", "items": [{"name": "Support", "qty": 5, "price": 100}]}
  ],
  "filename_key": "client_name"
}
```

### Single PDF — File Upload

Upload `.html`, `.css`, and `.json` files directly instead of embedding them in a JSON body.

**`POST /upload/render`**

```bash
curl -X POST https://api.pdforge.dev/upload/render \
  -F "html=@template.html" \
  -F "css=@style.css" \
  -F "data=@user.json;type=application/json" \
  -F "format=A4" \
  -o output.pdf
```

The `.css` and `.json` files are optional. If you skip the data file, the template renders as-is.

### Batch PDFs — File Upload

**`POST /upload/batch`** — The JSON file must contain an array of objects.

```bash
curl -X POST https://api.pdforge.dev/upload/batch \
  -F "html=@template.html" \
  -F "css=@style.css" \
  -F "data=@users.json;type=application/json" \
  -F "range=1-5" \
  -F "filename_key=name" \
  -F "format=Letter" \
  -F "landscape=true" \
  -o batch.zip
```

### Template Management

Save templates once, reuse them on every request without resending the HTML/CSS.

**`POST /templates/{name}`** — Save a template.

```json
{
  "html": "<h1>Receipt</h1><p>Amount: ${{ amount }}</p>",
  "css": "body { font-family: sans-serif; }",
  "sample_data": {"amount": "99.00"}
}
```

**`GET /templates`** — List all saved templates. Returns name and sample data for each.

### Health Check

**`GET /health`** — Returns the API status.

```json
{"status": "ok", "version": "0.2.0", "engine": "chromium"}
```

Use this to verify the API is running. Render uses it for automatic health monitoring and restarts.

---

## Options

These options are available on all render endpoints (JSON body and file upload).

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

### Landscape & Margins

Set `"landscape": true` for landscape orientation.

Custom margins are supported via the `margin` field:

```json
"margin": {"top": "10mm", "bottom": "10mm", "left": "20mm", "right": "20mm"}
```

Default margins are 20mm top/bottom, 15mm left/right.

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
