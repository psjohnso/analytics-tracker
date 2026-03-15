# City of Tucson Analytics Project Tracker — Project Reference

## Overview
Single-page HTML application (~6600+ lines) for managing analytics projects, tasks, team resources, and capacity planning for the City of Tucson ITD GIS/Data Analytics team. Originally migrated from SharePoint/MSAL to ArcGIS Online REST API with OAuth 2.0.

**Hosted on GitHub:** [psjohnso/analytics-tracker](https://github.com/psjohnso/analytics-tracker)

**Files:**
- `index.html` — The main application (single-file HTML with embedded CSS and JavaScript)
- `guide.html` — Standalone user guide styled to match the app

---

## ArcGIS Online Configuration

### OAuth 2.0
- **Portal URL:** `https://cotgis.maps.arcgis.com`
- **Client ID:** `H8cR2cAUoy0fVrJF`
- **Flow:** OAuth 2.0 Implicit Grant, 120-minute token lifetime
- **Token errors:** Codes 498/499 trigger automatic re-authentication

### Feature Services (REST Endpoints)
| Service | URL | Purpose |
|---------|-----|---------|
| **projects** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/projects/FeatureServer/0` | Analytics project records |
| **tasks** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/tasks/FeatureServer/0` | Individual work items linked to projects |
| **team_members** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/team_members/FeatureServer/0` | Team roster with roles and capacity settings |
| **absences** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/absences/FeatureServer/0` | Per-person, per-week time off records |
| **allocations** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/allocations/FeatureServer/0` | Per-person, per-project, per-week allocation records |
| **weekly_capacity** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/weekly_capacity/FeatureServer/0` | Legacy — no longer queried by the app |

### Key Fields by Service
- **team_members:** `name`, `role`, `team`, `skill`, `proj_pct`
- **absences:** `name`, `week_date` (Date Only), `absence_hours`
- **allocations:** `name`, `project`, `week_date` (Date Only), `fraction`, `hours`, `project_status`, `project_type`, `analytics_id`

### Access Control
- **Team Leads Group ID:** `2bd32af20dd745d6bdf3807446761973` (same group controls Idea promotion and Settings tab access)
- Settings tab and Idea promotion are restricted to members of this group

### ArcGIS Online Setup Requirements
- **Enable editing** on: `projects`, `tasks`, `team_members`, `absences`, `allocations`
- **Date Only fields:** `week_date` in `absences` and `allocations` — always send `"YYYY-MM-DD"` strings
- Register hosting URL as OAuth Redirect URI
- Change sharing from public to Organization or specific Group (pending)

---

## Architecture Decisions

### Data Loading
- On startup, query 3 services in parallel: `team_members`, `absences`, `allocations`
- `weekly_capacity` is **no longer queried** — all capacity is computed from formula
- Weeks generated dynamically via `generateWeeks(2026)` — 52 Sunday-start dates
- Dates from ArcGIS come as epoch ms or `"YYYY-MM-DD"` strings, handled by `epochToDateStr()`

### Capacity Formula
```
proj_cap = (40 - absence_hours) × 0.75 × proj_pct
```
- **40** = base work week hours (hardcoded, all employees)
- **0.75** = productivity factor (75% productive time, based on workplace research)
- **proj_pct** = per-employee percentage of time dedicated to project work
- **absence_hours** = hours off for that week (from absences service)

### Allocation Hours
```
allocated_hours = allocation_fraction × proj_cap
```
- `weekly_allocated` and `utilization` are **always computed client-side** from fractions and proj_cap
- On load, stored `hours` from REST are **ignored** — hours are recomputed from `fractions × proj_cap`
- This ensures consistency even if the capacity formula changes

### Utilization
```
utilization = total_allocated_hours / proj_cap
```
- Stored as a ratio (0.0–1.0+), displayed as percentage
- Color coding: green (<70%), amber (70–90%), red (>90%)

### Current Week Handling
- `currentWeekIdx` = actual calendar week (for visual highlights)
- `dataWeekIdx` = last week with entered data (for KPI numbers)

### Allocation Save Pipeline
- **Strategy:** Delete-then-add (delete existing records for person, then add all current allocations)
- `week_date` is Date Only field — always send `"YYYY-MM-DD"` strings
- Canonicalize project names for matching
- Comprehensive console logging with `[Allocations]` prefix

### Forecast Tab — Snapshot-Based Future Availability
1. **Past/present weeks:** Uses actual entered allocation hours
2. **Snapshot week:** Most recent week where person had allocations entered (total fracs > 0)
3. **Future projection:** Each project's snapshot fraction is carried forward into future weeks
4. **Project end dates:** Allocation drops to zero after project ends
5. **Availability:** `proj_cap - allocated_hours` per week
6. **Limitations:** Doesn't account for future absences not yet entered; no allocation data = fully available

### Settings Tab
- Visible only to Team Leads group members
- Sidebar hidden on Settings tab
- Team member CRUD: add, edit, remove members via `team_members` REST service
- Absence editor: auto-saves each cell change to `absences` REST service (no explicit save button)

### Week Dates Display
- All week dates are Sunday-start dates in the data
- For display, dates are converted to Monday (`Sun→Mon`, +1 day) across all tabs for consistency

---

## Version History

| Version | Git Message |
|---------|-------------|
| 0.0.0.0001–0.5.1.0016 | Initial ArcGIS migration, OAuth, UI cleanup, sidebar, user display, access control, login/logout, header stats |
| 0.5.2.0017 | Fixed sidebar filter counts showing zeros |
| 0.5.3.0018 | Fixed Resources tab allocation table filter |
| 0.6.0.0019 | **Resources data migrated from embedded JSON to ArcGIS Online REST services** |
| 0.6.1.0020 | Fixed view toggle: Grid button now has `active` class on load |
| 0.6.2.0021 | Fixed utilization: `weekly_allocated` and `utilization` computed client-side |
| 0.6.3.0022 | Fixed current week highlighting across allocation editor and Resources chart |
| 0.6.4.0023 | Fixed utilization stored as ratio not percentage; added `saveAllocationsToRest()`; Refresh reloads resources |
| 0.6.5.0024 | Rewrote allocation save pipeline with proper change detection, canonicalization, logging |
| 0.6.6.0025 | Auto-detect week_date field type (epoch vs string) for allocation saves |
| 0.6.7.0026 | Fixed Date Only field format — always send "YYYY-MM-DD" strings |
| 0.6.8.0027 | Allocation editor: projects sorted alphabetically; Tab moves down columns (column-major tabindex) |
| 0.6.9.0028 | Added "Current Week Project Allocations" header above team member cards |
| 0.6.10.0029 | Enlarged section header, reduced top margin, added divider between team cards and detail view |
| 0.7.0.0030 | **Settings tab** with team member CRUD and per-person absence editor, restricted to Team Leads group |
| 0.7.1.0031 | Compute proj_cap from absences formula instead of weekly_capacity table; removed unused weekly_capacity query |
| 0.7.2.0032 | Apply 75% productivity factor to capacity formula |
| 0.7.3.0033 | Renamed "Approver" badge to "Admin"; created comprehensive user guide (Word doc + standalone HTML) |
| 0.7.4.0034 | Added Help button linking to standalone `guide.html` |
| 0.7.5.0035 | Restyled Help button with solid background |
| 0.7.6.0036 | Restyled Help as solid blue button labeled "About / Help" |
| 0.7.7.0037 | Fix Forecast tab week labels to show Monday dates (consistent with Resources tab); expand guide with detailed Forecast calculation docs |
| 0.7.8.0038 | Fix 133% utilization bug: recompute allocation hours from fractions and local proj_cap instead of using stale stored hours |
| 0.7.9.0039 | Task form: move Details section above Classification for better UX flow |
| 0.8.0.0040 | Task form: replace category dropdown with searchable type-to-filter dropdown for faster selection |
| 0.8.1.0041 | Add 2 new project categories (Interdepartmental Consulting & Support, Public Engagement & Open Data); add descriptions to all 15 categories shown in searchable dropdown; upgrade project form to use searchable category dropdown too |
| 0.8.2.0042 | Add guided 2-question category wizard ("Help me choose") to both project and task forms; update guide with wizard documentation |

---

## Application Tabs

### Overview
- Dashboard with KPI summary cards (project counts by status, active tasks, team utilization)
- No sidebar or view toggle

### Projects
- Grid/list views of all analytics projects
- Filtering by status, priority, category, team member
- Detail page with all fields, associated tasks, timeline
- Project lifecycle: Idea → Active → On Hold / Complete / Canceled

### Tasks
- Individual work items linked to projects
- Status set: Not Started, In Progress, Complete, Deferred, Canceled
- "＋ New Task" button; "Weekly Review" filter

### Resources
- "Current Week Project Allocations" — team member cards with utilization badges
- 20-week stacked bar chart with capacity line
- KPI row: utilization %, project capacity hours, allocated hours, active projects, YTD hours
- Project allocation table per selected person
- "Edit Allocations" popup: projects sorted alphabetically, column-major tab order, delete-then-add save strategy

### Forecast
- Summary cards sorted by most available
- Team load area chart (combined allocated vs capacity)
- Utilization heatmap (per-person, per-week)
- Snapshot-based future projection (see Architecture Decisions above)

### Settings (Admin Only)
- Team Members table with Edit / Absences / Delete actions
- Add Member form
- Absence editor: 8-week scrollable grid, auto-save on each change
- Restricted to Team Leads group

---

## Deliverable Files

| File | Purpose |
|------|---------|
| `index.html` | Main application — single-file HTML with all CSS and JS embedded |
| `guide.html` | Standalone user guide — matches app styling, has scroll-spy sidebar TOC |

Both files must be in the same directory (guide.html is linked from the "About / Help" button in the tracker header).

---

## Pending Items
- Enable editing on ALL feature services (Settings → Feature Layer → Enable editing) — team_members and absences especially
- Change sharing from public to Organization or specific Group
- Register hosting URL as OAuth Redirect URI
- Verify actual field names returned by resource services (token required)
- Test allocation save end-to-end (check browser console for `[Allocations]` log messages)
- Update guide documentation when Forecast tab snapshot-based calculation changes
- Ensure week starting dates remain consistent across all tabs

---

## Development Conventions
- **Version format:** `MAJOR.MINOR.PATCH.BUILD` (e.g., `0.7.8.0038`)
- **Git commit message:** Include with every new version
- **Guide check:** After every app update, check if `guide.html` needs updating
- **This document:** Update after every app change to keep in sync
- **Persistent data:** When requests involve persistent data, discuss storing in ArcGIS Online as new or existing REST service before implementing
