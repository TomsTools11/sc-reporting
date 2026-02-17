# Report Management Dashboard for Supreme Choice

## Context

The site currently serves a single static HTML report (`public/index.html`) via Vercel. The goal is to transform it into a multi-report dashboard where reports persist in Vercel Blob storage, a sidebar lets the client pick which report to view, and new HTML reports can be uploaded directly from the site. No authentication — anyone with the link can upload.

## Architecture

```
+--------------------------------------------------+
|  GOAL   Report Dashboard             [Upload]    |  <- Top bar
+----------+---------------------------------------+
|          |                                       |
| Report 1 |                                       |
| Report 2 |        <iframe>                       |
| Report 3 |        (selected report)              |
|          |                                       |
+----------+---------------------------------------+
   Sidebar                Content
```

- **Storage:** Vercel Blob (`@vercel/blob`) — reports stored at `reports/<filename>.html`
- **API:** 3 Vercel serverless functions in `api/`
- **Frontend:** Vanilla HTML/CSS/JS dashboard shell with iframe for report isolation
- **iframe rationale:** Reports are full HTML documents with their own `<html>`/`<head>`/`<body>` and CSS variables — iframe prevents style collisions

## File Changes

| File | Action | Purpose |
|------|--------|---------|
| `package.json` | CREATE | `@vercel/blob` dependency |
| `vercel.json` | MODIFY | Add API route rewrites before the catch-all |
| `.gitignore` | MODIFY | Add `.env` |
| `api/reports.js` | CREATE | `GET /api/reports` — list all reports from blob |
| `api/upload.js` | CREATE | `POST /api/upload?filename=x.html` — upload HTML to blob |
| `api/serve.js` | CREATE | `GET /api/serve?filename=x.html` — proxy report HTML from blob |
| `public/index.html` | REPLACE | Dashboard shell (sidebar + iframe + upload modal) |
| `scripts/seed.js` | CREATE | One-time script to upload existing reports to blob |

## Implementation Steps

### 1. Project scaffolding
- Create `package.json` with `@vercel/blob` dependency
- Update `.gitignore` to add `.env`
- Update `vercel.json` with API rewrites:
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

### 2. API routes

#### `api/reports.js` — List all reports
- Calls `list({ prefix: 'reports/' })` from `@vercel/blob`
- Returns JSON array with `filename`, `url`, `size`, `uploadedAt`
- Sorted newest-first

#### `api/upload.js` — Upload HTML report
- Accepts `POST` with raw HTML body
- Filename passed via query param: `?filename=my-report.html`
- Sanitizes filename (alphanumeric, hyphens, underscores, dots only)
- Validates `.html` extension
- Calls `put('reports/<filename>', body, { access: 'public', contentType: 'text/html' })`
- Disable body parsing to stream raw content: `module.exports.config = { api: { bodyParser: false } }`

#### `api/serve.js` — Proxy report content
- Looks up blob by `?filename=` query param
- Fetches content from blob's public URL
- Returns as `text/html` with 5-min cache header
- This keeps report URLs on our domain (important for iframe)

### 3. Dashboard shell (`public/index.html`)

Replace the current report HTML with a dashboard:

- **Top bar:** GOAL branding + "Upload Report" button
- **Sidebar** (dark, `--goal-dark-blue: #00172D`): Lists reports with name + upload date. Active item highlighted with `--goal-accent-1: #3AEECA` left border
- **Main content:** Full-size iframe loading `/api/serve?filename=...`
- **Upload modal:** Drag-and-drop zone + file picker, validates `.html` extension, POSTs raw file content to `/api/upload`
- **On load:** Fetches `/api/reports`, renders sidebar, auto-selects newest report

Key CSS variables to reuse from existing design system:
```css
--goal-brand: #077BE5;
--goal-dark-blue: #00172D;
--goal-accent-1: #3AEECA;
--goal-accent-2: #97C2E8;
--goal-light-grey: #F8F9FC;
--goal-text: #001020;
--border-gray: #E5E7EB;
```

### 4. Seed existing reports
- Create `scripts/seed.js` that reads the 2 existing HTML files and uploads them to blob
- Run once: `BLOB_READ_WRITE_TOKEN=<token> node scripts/seed.js`
- Alternatively, just upload them through the dashboard UI after deploying

## Post-deploy setup

1. Add Vercel Blob store in Vercel dashboard (Storage > Create > Blob)
2. Set `BLOB_READ_WRITE_TOKEN` environment variable in Vercel project settings
3. Deploy
4. Upload existing reports via seed script or the upload UI

## Verification

- [ ] Visit deployed site — dashboard loads with sidebar
- [ ] Run seed script or upload existing reports — sidebar populates
- [ ] Click a report in sidebar — iframe renders it fully with correct styling
- [ ] Click "Upload Report" — modal appears with drag-and-drop zone
- [ ] Upload an HTML file — it appears in sidebar and is viewable
- [ ] Upload a second file — both listed and selectable
- [ ] Refresh page — reports persist (stored in Vercel Blob)

## Notes

- **No auth:** Anyone with the link can upload. Add a shared secret later if needed.
- **Filename collisions:** Re-uploading the same filename overwrites the previous version (intentional — allows updates).
- **File size:** Vercel serverless functions have a 4.5MB response limit. If reports exceed this, switch `api/serve.js` to redirect to the blob CDN URL instead of proxying.
- **No build step needed:** Vercel handles `api/` as serverless functions and `public/` as static assets natively.
