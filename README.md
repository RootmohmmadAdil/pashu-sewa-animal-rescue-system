# PashuSewa — Animal Rescue Reporting System

[![Status](https://img.shields.io/badge/status-active-brightgreen.svg)]()
[![Pages](https://img.shields.io/badge/frontend-Cloudflare%20Pages-blue.svg)]()
[![Workers](https://img.shields.io/badge/backend-Cloudflare%20Workers-blueviolet.svg)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)]()

A modern, mobile-first web application that empowers communities to report injured animals and enables NGOs and rescue teams to triage and respond efficiently. Built for reliability at the edge using Cloudflare Pages (frontend) and Cloudflare Workers + D1 (backend).

Table of contents
- [Project overview](#project-overview)
- [Key features](#key-features)
- [Architecture & tech stack](#architecture--tech-stack)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [API reference](#api-reference)
- [Database schema](#database-schema)
- [AI detection & image handling](#ai-detection--image-handling)
- [Security & privacy](#security--privacy)
- [Testing & development](#testing--development)
- [Contributing](#contributing)
- [Roadmap](#roadmap)
- [Acknowledgments & contact](#acknowledgments--contact)
- [License](#license)

---

## Project overview
PashuSewa simplifies the process of reporting injured animals and streamlines rescue workflows for organizations. Reporters can capture and submit photos with GPS coordinates while NGOs receive a central dashboard to monitor, filter, and manage incidents by status and proximity.

Goals
- Reduce time-to-rescue through precise location and image-based triage.
- Provide offline-resilient reporting with client-side fallbacks.
- Offer an extensible platform for NGOs to integrate their workflows.

---

## Key features

User (Reporter)
- Mobile-first reporting UI with direct camera capture
- Automatic GPS location capture with manual update
- Smart image upload with client-side validation and compression
- Auto-populated notes via AI-based species detection
- Real-time submission confirmation and status updates

NGO / Admin
- Management dashboard: view, filter, and sort reports
- Geo-filtering: find reports within a configurable radius
- Status lifecycle: Pending → In Progress → Resolved
- Distance calculation from NGO location
- Quick map links (Google Maps) to report locations
- Responsive and accessible admin UI

AI & image features
- Multi-tier detection (Google Vision / Clarifai ready)
- Client-side fallback analysis (color, shape, pattern heuristics)
- Weighted probability model for robust suggestions
- Always-provide fallback when external APIs are unavailable

---

## Architecture & tech stack

Frontend (Cloudflare Pages)
- Static HTML/CSS/JS (mobile-first)
- Files: frontend/index.html, frontend/admin.html, frontend/css/style.css, frontend/js/*.js

Backend (Cloudflare Workers, D1)
- Worker: backend/worker.js — HTTP API endpoints
- Database schema: backend/schema.sql (D1)
- Deployment config: backend/wrangler.toml

Other
- Optional: Google Vision API / Clarifai for improved detection
- CI/CD: Cloudflare Pages + Wrangler deploys

Architecture overview
- Browser → Cloudflare Pages (static assets)
- Pages → Cloudflare Worker API (POST/GET)
- Worker → D1 (SQLite-compatible) for persistence
- External AI services (optional) called from Worker / client

---

## Quick start

Prerequisites
- Cloudflare account with Pages & Workers access
- Node.js (LTS)
- Wrangler CLI (for Worker deployment)

Clone repository
```bash
git clone https://github.com/RootmohmmadAdil/pashu-sewa-animal-rescue-system.git
cd pashu-sewa-animal-rescue-system
```

Backend (Cloudflare Workers + D1)
1. Install Wrangler:
   npm install -g wrangler
2. Login to Cloudflare:
   wrangler login
3. Create a D1 database:
   wrangler d1 create pashusewa
4. Update backend/wrangler.toml with your database_id and other settings.
5. Run migrations:
   wrangler d1 execute pashusewa --file=backend/schema.sql
6. Deploy the worker:
   cd backend
   wrangler deploy

Frontend (Cloudflare Pages)
1. Update frontend/js/config.js with your Worker URL (API endpoint).
2. Deploy to Cloudflare Pages via the Cloudflare dashboard or GitHub integration.

Local development
- Serve the frontend statically (any local server).
- Use Wrangler dev for local Worker testing:
  wrangler dev backend/worker.js --local

---

## Configuration

frontend/js/config.js
```js
// Replace with your deployed Worker URL
const API_URL = "https://your-worker-name.your-subdomain.workers.dev";
```

backend/wrangler.toml (example)
```toml
name = "pashusewa"
main = "worker.js"
compatibility_date = "2026-01-01"

[[d1_databases]]
binding = "DB"
database_name = "pashusewa"
database_id = "your-database-id"
```

Environment variables & secrets
- Store API keys (Google Vision, Clarifai) in Worker secrets:
  wrangler secret put VISION_API_KEY

---

## API reference

All responses are JSON unless specified.

GET /api/reports
- Description: Get all reports (newest first)
- Response:
```json
[
  {
    "id": 1,
    "image": "data:image/jpeg;base64,...",
    "latitude": 28.6139,
    "longitude": 77.2090,
    "note": "Injured dog",
    "status": "Pending",
    "created_at": "2025-01-15T10:30:00.000Z"
  }
]
```

POST /api/report
- Description: Create a new animal report
- Request body:
```json
{
  "image": "data:image/jpeg;base64,...",
  "latitude": 28.6139,
  "longitude": 77.2090,
  "note": "Injured dog"
}
```
- Response: 201 Created with created report object

POST /api/update-status
- Description: Update report status (NGO/Admin only - protected)
- Request body:
```json
{
  "id": 1,
  "status": "In Progress"
}
```
- Response: 200 OK with updated report object

Authentication & Authorization
- Admin endpoints should be protected (API key, JWT, or Cloudflare Access). Implement authorization in backend/worker.js before updating status or accessing admin-only data.

Example curl
```bash
curl -X POST "$API_URL/api/report" \
  -H "Content-Type: application/json" \
  -d '{"image":"data:image/jpeg;base64,...","latitude":28.6139,"longitude":77.2090,"note":"Injured dog"}'
```

---

## Database schema

backend/schema.sql
```sql
CREATE TABLE IF NOT EXISTS reports (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  image TEXT NOT NULL,               -- Base64 encoded image
  latitude REAL NOT NULL,
  longitude REAL NOT NULL,
  note TEXT,
  status TEXT NOT NULL DEFAULT 'Pending',
  created_at TEXT NOT NULL           -- ISO 8601 timestamp
);
CREATE INDEX IF NOT EXISTS idx_reports_created_at ON reports(created_at DESC);
```

Notes
- Images are stored as Base64 strings for simplicity. For production, consider object storage (R2) with URLs in the DB.

---

## AI detection & image handling

Detection strategy (tiered)
1. Client-side heuristics: color histograms, shape/pattern checks (always available offline).
2. Optional calls to external APIs (Google Vision, Clarifai) for higher accuracy.
3. Weighted probability engine to combine results and produce a friendly note (e.g., "Injured dog").

Recommendations
- Use external APIs only server-side (Worker) to protect API keys.
- Keep client-side detection lightweight for performance and privacy.

Image handling
- Validate formats (JPEG/PNG/WebP) and limit size (recommended < 2 MB).
- Compress images client-side before upload to balance quality and bandwidth.

---

## Security & privacy
- Require explicit user consent before capturing GPS coordinates.
- Restrict CORS to production domains.
- Sanitize all inputs and avoid exposing internal error traces.
- Do not log raw image data in plain logs. If necessary, redact or hash.
- Store API keys securely using Wrangler secrets or Cloudflare KV/Secrets.

---

## Testing & development
- Test across multiple devices and browsers (mobile-first).
- Use Wrangler dev for Worker iteration and a static server for the frontend.
- Add unit tests for detection heuristics and API validation logic.
- Use Lighthouse and WebPageTest to validate performance and accessibility.

---

## Contributing
We welcome contributions!
1. Fork the repository
2. Create a feature branch: git checkout -b feature/your-feature
3. Add tests and documentation
4. Commit and push: git commit -m "Add feature" && git push origin feature/your-feature
5. Open a pull request describing your changes

Please follow the project's code style and include meaningful commit messages.

---

## Roadmap
Planned improvements
- R2-backed image storage (instead of Base64 in DB)
- Authentication for NGO/admin (OAuth or SSO)
- Real-time notifications (WebSockets / Push)
- Multi-language support and localized detection models
- Integration with local rescue databases and hotlines

---

## Acknowledgments & contact
Built with ❤️ for animal welfare.

- Cloudflare — Pages, Workers, D1
- Google Vision, Clarifai — optional integrations
- Open-source community and animal rescue organizations for guidance

Repository: https://github.com/RootmohmmadAdil/pashu-sewa-animal-rescue-system

For issues and support: open a GitHub Issue or Discussion in the repository.

---

## License
This project is open source. Recommended: MIT License — see LICENSE file.
