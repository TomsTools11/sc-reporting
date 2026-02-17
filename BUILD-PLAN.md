# Supreme Choice Report Dashboard — Unified Build Plan

## Context

The site currently serves a single static HTML report (`public/index.html`) via Vercel. We're transforming it into a multi-report dashboard where:
- Reports persist in Vercel Blob storage
- A sidebar lets users pick which report to view
- New HTML reports can be uploaded directly from the site
- No authentication required

A design system has been generated in `.ui-design/` with CSS custom property tokens for colors, typography, spacing, shadows, and radii. This plan unifies that design system with the dashboard implementation.

## Design System Integration

The `.ui-design/tokens/tokens.css` provides generic design tokens. The existing reports use **GOAL brand colors** which take priority. The dashboard will:

1. **Import** `.ui-design/tokens/tokens.css` for typography, spacing, shadows, and radii
2. **Override/extend** with GOAL brand colors as semantic dashboard variables:

```css
/* GOAL Brand — mapped onto design tokens */
--dashboard-bg: #00172D;           /* sidebar/topbar dark background */
--dashboard-brand: #077BE5;        /* primary action color */
--dashboard-accent: #3AEECA;       /* active state highlights */
--dashboard-accent-soft: #97C2E8;  /* secondary text/icons */
--dashboard-surface: #F8F9FC;      /* content area background */
--dashboard-text: #001020;         /* primary text */
--dashboard-border: var(--color-neutral-200);  /* reuse token */
```

3. **Reuse from design tokens directly:** `--font-family-sans`, `--font-size-*`, `--font-weight-*`, `--spacing-*`, `--radius-*`, `--shadow-*`

---

## File Changes

| File | Action | Purpose |
|------|--------|---------|
| `package.json` | CREATE | `@vercel/blob` dependency |
| `vercel.json` | MODIFY | Add API route rewrites before catch-all |
| `.gitignore` | MODIFY | Add `.env` |
| `api/reports.js` | CREATE | `GET /api/reports` — list reports from blob |
| `api/upload.js` | CREATE | `POST /api/upload?filename=x.html` — upload HTML to blob |
| `api/serve.js` | CREATE | `GET /api/serve?filename=x.html` — proxy report from blob |
| `public/index.html` | REPLACE | Dashboard shell (sidebar + iframe + upload modal) |
| `scripts/seed.js` | CREATE | One-time upload of existing reports to blob |

---

## Implementation Steps

### Step 1: Project scaffolding

**`package.json`** — create with `@vercel/blob` dependency

**`vercel.json`** — update with API rewrites (must come before catch-all):
```json
{
  "buildCommand": null,
  "outputDirectory": "public",
  "framework": null,
  "rewrites": [
    { "source": "/api/reports", "destination": "/api/reports.js" },
    { "source": "/api/upload", "destination": "/api/upload.js" },
    { "source": "/api/serve", "destination": "/api/serve.js" },
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

**`.gitignore`** — add `.env`

### Step 2: API routes

#### `api/reports.js`
- `list({ prefix: 'reports/' })` from `@vercel/blob`
- Returns JSON array: `{ filename, url, size, uploadedAt }`
- Sorted newest-first

#### `api/upload.js`
- `POST` with raw HTML body, filename via `?filename=my-report.html`
- Sanitize filename (alphanumeric, hyphens, underscores, dots)
- Validate `.html` extension
- `put('reports/<filename>', body, { access: 'public', contentType: 'text/html' })`
- Disable body parsing: `module.exports.config = { api: { bodyParser: false } }`

#### `api/serve.js`
- Lookup blob by `?filename=` query param
- Fetch content from blob's public URL
- Return as `text/html` with 5-min cache header

### Step 3: Dashboard shell (`public/index.html`)

Replace the current single-report HTML with a dashboard that imports and uses design tokens.

**Structure:**
```
+--------------------------------------------------+
|  GOAL   Report Dashboard             [Upload]    |  <- Top bar (#00172D bg)
+----------+---------------------------------------+
|          |                                       |
| Report 1 |                                       |
| Report 2 |        <iframe>                       |
| Report 3 |        (selected report)              |
|          |                                       |
+----------+---------------------------------------+
   Sidebar              Content (#F8F9FC bg)
  (#00172D)
```

**Design token usage in the dashboard CSS:**
- **Top bar:** `background: #00172D`, text in `#3AEECA`, height uses `--spacing-16`
- **Sidebar:** `background: #00172D`, width `260px`, text `--color-neutral-200`
- **Active item:** `border-left: 3px solid #3AEECA`, text `white`
- **Sidebar items:** `padding: var(--spacing-3) var(--spacing-4)`, `font-size: var(--font-size-sm)`
- **Upload button:** `background: #077BE5`, `border-radius: var(--radius-lg)`, `font-weight: var(--font-weight-semibold)`
- **Upload modal:** `border-radius: var(--radius-xl)`, `box-shadow: var(--shadow-xl)`, backdrop overlay
- **Drag-drop zone:** dashed border `var(--color-neutral-300)`, `border-radius: var(--radius-lg)`
- **Content area:** `background: var(--color-neutral-50)`
- **iframe:** full-size, no border, `border-radius: var(--radius-md)`

**Behavior:**
- On load: fetch `/api/reports`, render sidebar, auto-select newest
- Click sidebar item: load report in iframe via `/api/serve?filename=...`
- Upload modal: drag-and-drop zone + file picker, validates `.html`, POSTs to `/api/upload`
- Success feedback: close modal, refresh sidebar, select uploaded report

**Token import:** The dashboard HTML will include the design system tokens inline (copied from `.ui-design/tokens/tokens.css`) in a `<style>` block, then layer GOAL brand overrides on top. This avoids an extra network request since it's a single-page vanilla app.

### Step 4: Seed script (`scripts/seed.js`)
- Reads existing HTML files (`Supreme-Choice-Recommendations-v2.html`, `Supreme-Choice-Home-CA-Client-Report.html`)
- Uploads each to Vercel Blob at `reports/<filename>`
- Run: `BLOB_READ_WRITE_TOKEN=<token> node scripts/seed.js`

---

## Post-deploy Setup

1. Add Vercel Blob store in Vercel dashboard (Storage > Create > Blob)
2. Set `BLOB_READ_WRITE_TOKEN` env var in Vercel project settings
3. Deploy
4. Run seed script or upload existing reports via the UI

---

## Verification

- [ ] Dashboard loads with GOAL-branded top bar and dark sidebar
- [ ] Design tokens applied (correct fonts, spacing, shadows, radii throughout)
- [ ] Seed/upload existing reports — sidebar populates, sorted newest-first
- [ ] Click report in sidebar — iframe renders it with correct styling
- [ ] Upload modal opens with drag-and-drop zone
- [ ] Upload an HTML file — appears in sidebar and is viewable
- [ ] Refresh page — reports persist (Vercel Blob)
- [ ] Non-HTML files rejected with error message

---

## Notes

- **No auth:** Anyone with the link can upload. Add shared secret later if needed.
- **Filename collisions:** Re-uploading same filename overwrites (intentional for updates).
- **File size limit:** Vercel serverless has 4.5MB response limit. If exceeded, switch `api/serve.js` to redirect to blob CDN URL.
- **No build step:** Vercel handles `api/` as serverless functions and `public/` as static assets natively.
- **Design tokens source of truth:** `.ui-design/tokens/tokens.css` — but GOAL brand colors override the generic primary/secondary palette for this dashboard.
