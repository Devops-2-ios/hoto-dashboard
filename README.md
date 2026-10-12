# Defect Tracker

A web dashboard for the joint STEL / LTA defect register. Upload the Excel file, and every defect is extracted, tagged and kept intact: all 20 columns, plus the before / after photos attached to the correct row. Every upload is saved, so you can see how each defect moved over time.

* **No build step, no server code.** Plain HTML, CSS and JavaScript. Open `index.html` through any web server.
* **Works on a closed network.** Excel and zip libraries are bundled in `vendor/`; there are no CDN calls.
* **Light, Apple-style interface** with animated charts, a sliding drawer, board view and a photo lightbox.

> **This build is a front-end release.** Sign-in is a *demo* and history lives in the visitor's own browser. See "Going multi-user" for the production path.

---

## 1. Run it

**Locally (quickest check)**

```bash
cd defect-tracker
python3 -m http.server 8080     # or: npx serve .
# open http://localhost:8080
```

**GitHub Pages**

1. Create a repository and upload the contents of this folder (keep the folder structure; `.nojekyll` must stay).
2. *Settings → Pages → Deploy from a branch → `main` / root.*
3. Open `https://<user>.github.io/<repo>/`.

> ⚠️ GitHub Pages sites are **public** unless your organisation has GitHub Enterprise with private Pages. The sample data in this build is fictional, which is why it is safe to show management there. **Never commit a real defect register or photos to the repository.** Real files are uploaded in the browser and are never part of the code.

**Your own server (recommended for real use):** copy the folder to any static host (IIS, nginx, Apache, S3 + CloudFront, SharePoint site assets, an internal web server). Serve over HTTPS.

---

## 2. How an upload works

1. Drop the `.xlsx` (or `.csv`) on the Upload sheet. The file is read **in the browser**.
2. **Header row and columns are auto-detected** by their headings, so column order can change. Anything unmatched can be mapped by hand (and remembered).
3. **One spreadsheet row = one defect record.** Fields are bound together; sorting, filtering and searching always move the whole record, so details never cross over between defects.
4. **Photos are attached by position.** Each picture belongs to the row it sits on (using the anchor in the file, with crop and rotation applied). Pictures in or after *Defect Rectification (by LTA)* become **After**, in or after *Sign-Off* become **Sign-off**, the rest are **Before**. A picture overlapping two rows is given to the row holding most of it and flagged for a quick check.
5. **Review before saving:** row-count check (Excel rows vs defects read), column mapping, data-quality list, and a comparison with the previous upload (new / updated / closed / missing, with field-by-field old → new values). Doubtful matches are put to the user to confirm.
6. **Confirm** and the dashboard updates. Nothing is overwritten; each upload is a snapshot.

### Data rules built in

| Situation | What happens |
|---|---|
| S/N used more than once | Renumbered `56.1`, `56.2` … and reported |
| Blank S/N | Kept, labelled `R<row>`, reported |
| `Safety` in Classification | Standardised to *Safety Related* |
| Free text in Classification | Shown as *Unclassified*, original text kept and flagged |
| Dates | Day / month / year. Several dates in one cell (`3/7/2025 & 4/7/2025`) become a list; first and latest are used |
| Status wording | `Opened`, `open` etc. all count as *Open* (editable in Settings) |
| Merged cells | Only the top-left value is read; nothing is copied into other rows; reported |

### Workflow stage (derived)

*Closed* (Status = Closed, Date Closed filled, or an accepted Sign-Off) → *Rework required* (Sign-Off contains words such as "not accepted") → *Rectified, awaiting sign-off* (LTA rectification text, rectification date or after photo) → *Pending LTA action*. Rules are editable in **Settings**.

---

## 3. Privacy and storage (this build)

* The uploaded **Excel file is not stored.** Its defects, photos (resized to 1600 px) and notes are saved in the browser's IndexedDB on that device.
* Nothing is sent to any server. Clearing browser data, or *Settings → Delete all data*, removes everything.
* History is therefore **per browser, per device**. Colleagues do not see each other's uploads. That is the main reason for the multi-user step below.
* Demo accounts (`admin@demo.local`, `stel@demo.local`, `contractor@demo.local`, `viewer@demo.local`, any password) are **not secure**.

---

## 4. Going multi-user (production)

The code is arranged so the two browser-only pieces can be swapped without touching the screens.

| Replace | File | Contract |
|---|---|---|
| Storage | `js/store.js` | Same async methods: `init, getKV/setKV, getBatches, getRecords, getRecordHistory, saveBatch(batch, records, mediaMap), deleteBatch, photoURL(hash, 'thumb'\|'full'), getNotes/addNote/deleteNote, wipe, usage` |
| Sign-in | `js/auth.js` | `current()`, `signIn()`, `signOut()`, `can(action)`, `users`, `ROLES` |

Suggested stack: **Supabase** (Postgres + Auth + Storage) or your organisation's own API with SSO (Azure AD / Okta).

* `batches` → table; `records` → table keyed by `(batch_id, key)` where `key` is the stable defect identity; `photos` → object storage keyed by `hash`; `notes` → table.
* Enforce roles **on the server** (row-level security), not only in the UI: Admin, STEL, Contractor (upload / notes), Viewer (read only).
* Add audit fields (who uploaded, when) — already captured in `batch.uploadedBy`.
* If the shared store is large, move parsing server-side only if needed; client-side parsing is fast for hundreds of rows with photos.

---

## 5. Files

```
index.html          entry page
css/styles.css      design system, components, animation, print
vendor/             SheetJS (xlsx) and JSZip, bundled for offline use
js/util.js          helpers (dates, hashing, formatting)
js/domain.js        fields, stages, classification, derived values
js/parser.js        Excel reader: columns, rows, dates, photo extraction
js/diff.js          matches defects between uploads, field-level changes
js/store.js         storage adapter (IndexedDB)
js/auth.js          sign-in adapter (demo)
js/filters.js       filter model
js/charts.js        SVG / CSS charts
js/exporter.js      CSV, Excel, PDF report
js/demo.js          fictional sample data (3 uploads with photos)
js/ui.js            modal, drawer, lightbox, menu, toast
js/app.js           routing, state, saving an upload
js/view-*.js        Overview, Defects, Upload flow, Uploads, Settings
```

## 6. Exports

* **Excel** — all 20 register columns in their original order plus stage, age and overdue; a Summary sheet; an Upload-history sheet. Respects the current filters and sort.
* **CSV** — same columns, UTF-8.
* **PDF report** — choose *Save as PDF* in the print dialog. Summary page, defect list, and optionally one page per defect with its photos.

## 7. Browser support

Current Chrome, Edge, Safari and Firefox. Photos are read from `.xlsx` files only (not `.xls` or `.csv`). Keep the full folder together; do not open `index.html` with `file://` in browsers that block IndexedDB there — use a web server.

## 8. Third-party

SheetJS Community Edition (Apache-2.0, see `vendor/LICENSE-sheetjs.txt`) and JSZip (MIT / GPLv3 dual).
