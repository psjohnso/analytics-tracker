# City of Tucson Analytics Project Tracker — Reconstruction Blueprint

> **Purpose:** This is a living document containing everything needed for a new Claude instance to reconstruct the Analytics Project Tracker application from scratch. It includes architecture, data models, code patterns, implementation details, edge cases, and branding specifications.
>
> **Update Rule:** This document MUST be updated after every application change. When working on this application, always check if this document needs updating.
>
> **Last Updated:** March 2026
> **Current Version:** 0.10.0.0005 (see Version History at bottom)

---

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [ArcGIS Online Configuration](#2-arcgis-online-configuration)
3. [Feature Services — Complete Data Model](#3-feature-services--complete-data-model)
4. [HTML Structure](#4-html-structure)
5. [CSS Architecture](#5-css-architecture)
6. [JavaScript Architecture](#6-javascript-architecture)
7. [OAuth 2.0 Implementation](#7-oauth-20-implementation)
8. [ArcGIS REST API Layer](#8-arcgis-rest-api-layer)
9. [Tab Implementations](#9-tab-implementations)
10. [Form Modals (Project & Task CRUD)](#10-form-modals-project--task-crud)
11. [Resources & Allocation System](#11-resources--allocation-system)
12. [Capacity & Utilization Math](#12-capacity--utilization-math)
13. [Time Tracking System](#13-time-tracking-system)
14. [Settings Tab (Admin)](#14-settings-tab-admin)
15. [Edge Cases & Bug Fixes](#15-edge-cases--bug-fixes)
16. [Branding & Design Specifications](#16-branding--design-specifications)
17. [Deployment](#17-deployment)
18. [Development Conventions](#18-development-conventions)
19. [Reconstruction Checklist](#19-reconstruction-checklist)
20. [Default Configuration Values](#20-default-configuration-values)
21. [Version History](#21-version-history)

---

## 1. Application Overview

Single-page HTML application (~11,000+ lines) for managing analytics projects, tasks, team resources, and capacity planning for the City of Tucson ITD GIS/Data Analytics team.

**Files:**
- `index.html` — The entire application (single file: HTML + CSS + JavaScript)
- `guide.html` — Standalone user guide with matching branding
- `PROJECT_REFERENCE.md` — This document (also pushed to the repo)

**Hosted on GitHub Pages:** [psjohnso/analytics-tracker](https://github.com/psjohnso/analytics-tracker)

### Key Design Decisions

- **Single HTML file** — no build process, no npm, no bundler, no server-side code
- **All data in ArcGIS Online** feature services with full CRUD via REST API
- **Configuration stored in app_config** service (key-value pairs of JSON arrays), editable from Settings tab
- **Weeks generated dynamically** via `generateWeeks(2026)` — 52 Sunday-start dates (displayed as Monday in UI)
- **Capacity computed client-side** from schedule, absences, and proj_pct — never stored
- **Access control** via ArcGIS Online group membership (no application-level user management)
- **Allocation hours recomputed from fractions on load** — stored hours may be stale, fractions are source of truth

---

## 2. ArcGIS Online Configuration

### OAuth 2.0

| Setting | Value |
|---------|-------|
| Portal URL | `https://cotgis.maps.arcgis.com` |
| Client ID | `H8cR2cAUoy0fVrJF` |
| Flow | OAuth 2.0 Implicit Grant |
| Token Lifetime | 120 minutes |
| Token Error Codes | 498/499 trigger automatic re-authentication |
| Redirect URI | GitHub Pages URL (must be registered in ArcGIS Online) |

**Setup:** Content → My Content → New Item → Application → Register Application. Add the GitHub Pages URL as a redirect URI.

### Access Control

| Setting | Value |
|---------|-------|
| Team Leads Group ID | `2bd32af20dd745d6bdf3807446761973` |
| Group Check | On login, query `/community/groups/{id}/users` for membership |
| Controls | Settings tab visibility, Idea promotion to Active project |

### Permission Matrix

| Action | Standard User | Admin (Team Lead) |
|--------|:---:|:---:|
| View all tabs (except Settings) | ✅ | ✅ |
| View Settings tab | ❌ | ✅ |
| Create / edit projects and tasks | ✅ | ✅ |
| Submit project ideas | ✅ | ✅ |
| Promote ideas to active projects | ❌ | ✅ |
| Edit allocations | ✅ | ✅ |
| Manage team members | ❌ | ✅ |
| Edit absences / time off | ❌ | ✅ |
| Enable/disable time tracking | ❌ | ✅ |
| Log time entries (if enabled) | ✅ | ✅ |
| Edit dropdown lists | ❌ | ✅ |

### Pay Period Configuration

| Setting | Value |
|---------|-------|
| Pay Period Start Day | Sunday |
| Week A Reference Date | December 28, 2025 |
| Week A Hours (9/80) | 44 |
| Week B Hours (9/80) | 36 (includes RDO) |
| Week type detection | `(daysDiff / 7) % 2 === 0` → Week A |

---

## 3. Feature Services — Complete Data Model

All services are **Tables (no geometry)** with **editing enabled** (Add, Update, Delete).

**Base URL pattern:**
```
https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/{name}/FeatureServer/0
```

### 3.1 projects

| Field | Type | Length | Null | Description |
|-------|------|--------|------|-------------|
| `id` | Integer | — | No | Unique analytics project ID |
| `pid` | String | 100 | Yes | Generated identifier (title + id) |
| `title` | String | 255 | No | Project name |
| `status` | String | 50 | No | Active, Completed, On Hold, Cancelled, Idea |
| `priority` | String | 20 | Yes | High, Medium, Low |
| `contact` | String | 100 | Yes | Primary contact / project lead |
| `other_members` | String | 500 | Yes | Comma-separated other team members |
| `partner_dept` | String | 100 | Yes | Partner department (from app_config) |
| `category` | String | 100 | Yes | Project category (from app_config) |
| `start` | Date Only | — | Yes | Planned start date |
| `end_` | Date Only | — | Yes | Planned end date (**note underscore — "end" is reserved**) |
| `actual_end` | Date Only | — | Yes | Actual completion date |
| `description` | String | 4000 | Yes | Project description |
| `problem_statement` | String | 4000 | Yes | Problem statement |
| `itd_team` | String | 50 | Yes | ITD team (from app_config) |
| `project_size` | String | 5 | Yes | Project size: S, M, L, XL — drives allocation defaults |

**Critical:** The `end_` field mapping is the ONLY field name difference between ArcGIS and the app. Handled by:
```javascript
const PROJECT_FIELD_MAP = { end: 'end_' };
const PROJECT_AGOL_TO_LOCAL = { 'end_': 'end' };
```

### 3.2 tasks

| Field | Type | Length | Null | Description |
|-------|------|--------|------|-------------|
| `idx` | Integer | — | No | Unique task ID (auto-incremented by app) |
| `id` | Integer | — | Yes | Legacy analytics ID |
| `title` | String | 255 | No | Task name |
| `status` | String | 50 | No | Not Started, In Progress, Completed, On Hold, Waiting for Response, Cancelled |
| `priority` | String | 20 | Yes | High, Medium, Low |
| `assignee` | String | 100 | Yes | Assigned person |
| `project_id` | Integer | — | Yes | Links to project `id` field |
| `project` | String | 255 | Yes | Parent project title (denormalized) |
| `category` | String | 100 | Yes | Task category (from app_config) |
| `start` | Date Only | — | Yes | Start date |
| `due` | Date Only | — | Yes | Due date |
| `actual_end` | Date Only | — | Yes | Actual completion date |
| `tool` | String | 100 | Yes | Primary tool (from app_config) |
| `hours` | String | 20 | Yes | Estimated hours (**stored as string, not number**) |
| `hours_worked` | Double | — | Yes | Actual hours worked |
| `description` | String | 4000 | Yes | Task description |

**Task field names are identical in ArcGIS and the app — no mapping needed.**

**Pagination:** Tasks can exceed 1,000 records (ArcGIS query limit). The app paginates using `resultOffset` until all records are loaded.

### 3.3 team_members

**Core fields:**

| Field | Type | Length | Null | Description |
|-------|------|--------|------|-------------|
| `name` | String | 100 | No | Full name (**join key** to absences, allocations, time_entries) |
| `role` | String | 100 | Yes | Job title |
| `team` | String | 50 | Yes | GIS, Analytics, AI, Server, Management |
| `skill` | String | 100 | Yes | Primary skill area |
| `proj_pct` | Double | — | Yes | Fraction of productive time for projects (e.g., 0.63) |
| `schedule_type` | String | 10 | Yes | 5/8, 4/10, 9/80 |
| `rdo_day` | String | 10 | Yes | Regular day off (e.g., "Friday"). Null for 5/8 |
| `lunch_minutes` | Integer | — | Yes | Daily lunch duration in minutes |
| `time_tracking` | String | 10 | Yes | "true" or "false" — per-employee opt-in |

**Daily schedule fields (20 total):**
Pattern: `wk{1|2}_{mon|tue|wed|thu|fri}_{start|end}` — String(5) — Values like `"08:00"`, `"17:00"`. Null/empty = day off.

```
wk1_mon_start, wk1_mon_end, wk1_tue_start, wk1_tue_end, wk1_wed_start, wk1_wed_end,
wk1_thu_start, wk1_thu_end, wk1_fri_start, wk1_fri_end,
wk2_mon_start, wk2_mon_end, wk2_tue_start, wk2_tue_end, wk2_wed_start, wk2_wed_end,
wk2_thu_start, wk2_thu_end, wk2_fri_start, wk2_fri_end
```

**Computing daily hours from schedule:**
```javascript
daily_hours = (parseTime(end) - parseTime(start)) - lunch_minutes / 60
// parseTime("17:00") = 17.0, parseTime("08:00") = 8.0
// If start or end is null/empty → day off (0 hours)
weekly_hours = sum of daily_hours for Mon-Fri of current week type (A or B)
```

### 3.4 absences

| Field | Type | Null | Description |
|-------|------|------|-------------|
| `name` | String(100) | No | Employee name (joins to team_members) |
| `week_date` | Date Only | No | Sunday start of week (YYYY-MM-DD) |
| `absence_hours` | Double | No | Hours off that week (0-40) |

**Only weeks with non-zero absences have records.** Missing week = 0 hours off.

### 3.5 allocations

| Field | Type | Null | Description |
|-------|------|------|-------------|
| `name` | String(100) | No | Employee name |
| `project` | String(255) | No | Project title |
| `week_date` | Date Only | No | Sunday start of week (YYYY-MM-DD) |
| `fraction` | Double | Yes | Allocation 0.0–1.0 of project capacity |
| `hours` | Double | Yes | Planned hours (fraction × proj_cap) |
| `project_status` | String(50) | Yes | Denormalized project status |
| `project_type` | String(100) | Yes | Denormalized project category |
| `analytics_id` | Integer | Yes | Links to project `id` field |
| `project_role` | String(20) | Yes | Person's role on this project: Lead, Contributor, Reviewer |

**IMPORTANT:** `fraction` is the source of truth. `hours` is recomputed on load from `fraction × local_proj_cap` to prevent stale data bugs (the "133% utilization bug").

### 3.6 app_config

| Field | Type | Null | Description |
|-------|------|------|-------------|
| `config_key` | String(100) | No | Unique key (e.g., "partner_depts") |
| `config_value` | String(5000) | No | JSON-encoded array of values |

**Current keys:** partner_depts, itd_teams, task_categories, task_tools, task_statuses, project_statuses, project_categories, allocation_defaults

**Note:** `allocation_defaults` stores a JSON **object** (not an array): `{"S":{"Lead":15,"Contributor":10,"Reviewer":5},"M":{...},...}`. The `applyAppConfig` function handles this specially — it checks for objects before the array check.

### 3.7 time_entries

| Field | Type | Null | Description |
|-------|------|------|-------------|
| `name` | String(100) | No | Employee name |
| `task_idx` | Integer | No | Reference to task `idx` field |
| `project_id` | Integer | Yes | Reference to parent project `id` |
| `start_time` | Date and Time | No | When work began (epoch ms) |
| `end_time` | Date and Time | Yes | When work ended (null = timer running) |
| `hours` | Double | Yes | Auto-calculated: (end - start) in hours |
| `work_date` | Date Only | No | Calendar date for daily grouping |
| `notes` | String(500) | Yes | Work description |

### 3.8 weekly_capacity (LEGACY — NO LONGER QUERIED)

This service exists but is **not used by the application**. All capacity is computed client-side.

---

## 4. HTML Structure

The entire application is one HTML file with this structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Analytics Project Tracker — City of Tucson</title>
  <link href="https://fonts.googleapis.com/css2?family=Lato:wght@300;400;700;900&family=Cardo:ital,wght@0,400;0,700;1,400&display=swap" rel="stylesheet">
  <style>
    /* ~1,800 lines of CSS */
  </style>
</head>
<body>

  <!-- HEADER (fixed, navy background) -->
  <header id="app-header">
    <!-- App title, live stats, user name, Refresh/Help buttons -->
  </header>

  <!-- MAIN LAYOUT -->
  <div id="app-layout">
    <!-- LEFT SIDEBAR (collapsible, 280px) -->
    <aside id="sidebar">
      <!-- Tab navigation buttons -->
      <!-- Filter panels (status, priority, assignee, etc.) -->
    </aside>

    <!-- CONTENT AREA -->
    <main id="content-area">
      <!-- Dynamically rendered by render() based on currentTab -->
    </main>
  </div>

  <!-- MODALS (hidden by default) -->
  <div id="form-modal-backdrop"><!-- Project/Task create/edit form --></div>
  <div id="member-form-backdrop"><!-- Team member create/edit form --></div>

  <script>
    /* ~5,000+ lines of JavaScript */
  </script>
</body>
</html>
```

### Header Structure

```html
<header id="app-header" style="position:fixed; top:0; left:0; right:0; height:56px;
  background:var(--navy); display:flex; align-items:center; padding:0 20px; z-index:1000;">

  <!-- Left: App title -->
  <div style="font-weight:900; font-size:18px; color:#fff; letter-spacing:0.02em;">
    Analytics Project Tracker
  </div>

  <!-- Center: Live stats badges -->
  <div id="header-stats" style="display:flex; gap:12px; margin-left:auto;">
    <!-- Dynamically rendered: Active Projects (N), Tasks (N), Team Utilization (N%) -->
  </div>

  <!-- Right: User info + buttons -->
  <div style="display:flex; align-items:center; gap:10px; margin-left:20px;">
    <span id="user-display"><!-- User full name --></span>
    <button id="btn-refresh" onclick="refreshData()">↻ Refresh</button>
    <button onclick="window.open('guide.html','_blank')">About / Help</button>
    <button id="btn-save-file" onclick="saveAllData()">💾</button>
  </div>
</header>
```

### Sidebar Structure

The sidebar contains:
1. **Tab navigation buttons** — one per tab, highlighted when active
2. **Filter sections** (only shown on Projects/Tasks tabs) — expandable/collapsible
3. **Filter counts** — badge showing how many items match each filter value

```javascript
// Tab buttons generated dynamically
var tabs = ['overview', 'mywork', 'projects', 'tasks', 'resources', 'forecast'];
if (_isTeamLead) tabs.push('settings');
```

The sidebar is hidden (display:none) on Overview, My Work, Resources, Forecast, and Settings tabs.

---

## 5. CSS Architecture

### CSS Variables (defined on :root)

```css
:root {
  --navy: #002669;        /* Primary — header, buttons, text */
  --orange: #C24200;      /* Accent — subheadings, alerts */
  --yellow: #FFDB22;      /* Highlights, warnings */
  --green: #83AC16;       /* Success, completed */
  --sky-blue: #0088FF;    /* Links, active states */
  --night-sky: #140233;   /* Dark alternative */
  --cactus-fruit: #9E0059;/* Department accent */
  --monsoon-gray: #E1E2DD;/* Borders */
  --text: #1F2937;        /* Body text */
  --text-muted: #6B7280;  /* Secondary text */
  --card-bg: #FFFFFF;     /* Card backgrounds */
  --bg: #F9FAFB;          /* Page background */
  --border: #E5E7EB;      /* Default border */
}
```

### Key Layout Patterns

**App layout (header + sidebar + content):**
```css
body { margin:0; padding-top:56px; background:var(--bg); font-family:'Lato',sans-serif; }
#app-layout { display:flex; min-height:calc(100vh - 56px); }
#sidebar { width:280px; flex-shrink:0; background:#fff; border-right:1px solid var(--border);
  overflow-y:auto; position:sticky; top:56px; height:calc(100vh - 56px); }
#content-area { flex:1; padding:24px 32px; min-width:0; overflow-x:hidden; }
```

**Cards (grid view):**
```css
.card-grid { display:grid; grid-template-columns:repeat(auto-fill, minmax(340px, 1fr)); gap:16px; }
.project-card, .task-card {
  background:#fff; border:1px solid #E5E7EB; border-radius:12px;
  padding:20px; cursor:pointer; transition:all 0.15s;
  border-left:4px solid var(--navy); /* Color varies by status */
}
.project-card:hover { border-color:var(--navy); box-shadow:0 4px 12px rgba(0,0,0,0.08); }
```

**Status badge colors:**
```css
/* Applied via inline styles or CSS classes */
Active:    background:#DBEAFE; color:#1E40AF;
Completed: background:#D1FAE5; color:#065F46;
On Hold:   background:#FEF3C7; color:#92400E;
Cancelled: background:#FEE2E2; color:#991B1B;
Idea:      background:#F3E8FF; color:#6B21A8;
In Progress: background:#DBEAFE; color:#1E40AF;
Not Started: background:#F3F4F6; color:#374151;
Waiting:   background:#FEF3C7; color:#92400E;
```

**Form modal:**
```css
#form-modal-backdrop {
  display:none; position:fixed; inset:0; z-index:9999;
  background:rgba(0,0,0,0.4); align-items:flex-start; justify-content:center;
  padding:40px 20px; overflow-y:auto;
}
/* When shown: display:flex */
.form-modal {
  background:#fff; border-radius:16px; width:100%; max-width:720px;
  box-shadow:0 20px 60px rgba(0,0,0,0.2); overflow:hidden;
}
.fm-grid { display:grid; grid-template-columns:1fr 1fr; gap:12px 16px; }
.fm-field.span2 { grid-column:span 2; }
```

---

## 6. JavaScript Architecture

### Global State Variables

```javascript
// ── Data ──
let PROJECTS = [];              // Array of project objects
let TASKS = [];                 // Array of task objects
let RESOURCES_DATA = {          // Reconstructed on load
  weeks: [],                    // 52 week objects [{start, end, label}, ...]
  people: {},                   // { "Name": {role, team, skill, proj_pct, ...} }
  absences: {},                 // { "Name": { weekIndex: hours, ... } }
  allocations: {},              // { "Name": { "ProjectTitle": { weekIdx: {frac, hrs, oid}, ... } } }
  proj_cap: {},                 // { "Name": [52 floats] } — computed
  weekly_allocated: {},         // { "Name": [52 floats] } — computed
  utilization: {},              // { "Name": [52 floats] } — computed
};
let TIME_ENTRIES = [];          // Current user's time entries
let CONFIG_DATA = {};           // { partner_depts:[], itd_teams:[], ... }

// ── UI State ──
let currentTab = 'overview';
let currentView = 'grid';       // 'grid' or 'list'
let currentDetail = null;       // { type:'project'|'task', id:N, _fromProject:N, _returnTab:str }
let _sidebarCollapsed = false;

// ── Auth State ──
let _agolToken = null;
let _agolTokenExpiry = 0;
let _userFullName = '';
let _isTeamLead = false;

// ── Resources State ──
let currentWeekIdx = 0;         // Index into RESOURCES_DATA.weeks for current calendar week
let _selectedPerson = null;     // Currently selected person on Resources tab
let _allocEditorData = null;    // Working copy during allocation editing

// ── Time Tracking State ──
let _timeEntryTimerInterval = null;  // setInterval ID for updating elapsed time display

// ── Settings State ──
let _settingsSelectedMember = null;
let _absEditorWindowStart = 0;
const ABS_COLS = 8;             // Number of weeks visible in absence editor

// ── Allocation Defaults ──
let _allocationDefaults = {     // Loaded from app_config, editable in Settings
  S: {Lead:15, Contributor:10, Reviewer:5},
  M: {Lead:25, Contributor:15, Reviewer:5},
  L: {Lead:40, Contributor:20, Reviewer:10},
  XL:{Lead:50, Contributor:30, Reviewer:10}
};

// ── Size Wizard State ──
let _sizeWizardAnswers = [-1,-1,-1,-1];
let _sizeWizardStep = 0;
```

### Application Configuration Object

```javascript
const ARCGIS_CONFIG = {
  portalUrl: 'https://cotgis.maps.arcgis.com',
  clientId: 'H8cR2cAUoy0fVrJF',
  projectsUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/projects/FeatureServer/0',
  tasksUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/tasks/FeatureServer/0',
  teamMembersUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/team_members/FeatureServer/0',
  absencesUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/absences/FeatureServer/0',
  allocationsUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/allocations/FeatureServer/0',
  appConfigUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/app_config/FeatureServer/0',
  timeEntriesUrl: 'https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/time_entries/FeatureServer/0',
  teamLeadsGroupId: '2bd32af20dd745d6bdf3807446761973',
};
```

### Week Generation

```javascript
function generateWeeks(year) {
  // Find first Sunday of the year (or last Sunday of prior year)
  var weeks = [];
  var d = new Date(year, 0, 1);
  while (d.getDay() !== 0) d.setDate(d.getDate() - 1); // back up to Sunday
  for (var i = 0; i < 52; i++) {
    var start = new Date(d);
    var end = new Date(d);
    end.setDate(end.getDate() + 6);
    weeks.push({
      start: start,
      end: end,
      label: formatMonLabel(start), // "Jan 6" (shows Monday, not Sunday)
      dateStr: toDateStr(start),    // "2026-01-04" (Sunday for data matching)
    });
    d.setDate(d.getDate() + 7);
  }
  return weeks;
}
```

**Critical:** Data stores Sunday dates. UI displays Monday dates (+1 day). The `formatMonLabel()` function adds 1 day to the Sunday start when rendering labels.

### Main Render Pipeline

```javascript
function render() {
  updateHeaderStats();  // Recalculate KPI badges in header
  
  // Show/hide sidebar based on tab
  var showSidebar = (currentTab === 'projects' || currentTab === 'tasks');
  document.getElementById('sidebar').style.display = showSidebar ? 'block' : 'none';
  
  if (showSidebar) updateSidebarFilters();
  
  var content = document.getElementById('content-area');
  
  // If showing a detail view
  if (currentDetail) {
    if (currentDetail.type === 'project') { renderProjectDetail(content); return; }
    if (currentDetail.type === 'task') { renderTaskDetail(content); return; }
  }
  
  switch (currentTab) {
    case 'overview':  renderOverview(content); break;
    case 'mywork':    renderMyWork(content); break;
    case 'projects':  renderProjects(content); break;
    case 'tasks':     renderTasks(content); break;
    case 'resources': renderResources(content); break;
    case 'forecast':  renderForecast(content); break;
    case 'settings':  renderSettings(content); break;
  }
}
```

---

## 7. OAuth 2.0 Implementation

### Flow

```javascript
// On page load:
function initAuth() {
  // 1. Check URL hash for token (redirect back from ArcGIS)
  var hash = window.location.hash;
  if (hash && hash.includes('access_token')) {
    _agolToken = extractParam(hash, 'access_token');
    _agolTokenExpiry = Date.now() + (parseInt(extractParam(hash, 'expires_in')) * 1000);
    window.location.hash = ''; // Clean URL
    onAuthSuccess();
    return;
  }
  
  // 2. No token — redirect to ArcGIS login
  redirectToLogin();
}

function redirectToLogin() {
  var redirectUri = encodeURIComponent(window.location.origin + window.location.pathname);
  var url = ARCGIS_CONFIG.portalUrl + '/sharing/rest/oauth2/authorize'
    + '?client_id=' + ARCGIS_CONFIG.clientId
    + '&response_type=token'
    + '&redirect_uri=' + redirectUri
    + '&expiration=120';
  window.location.href = url;
}

function onAuthSuccess() {
  // 1. Get user identity
  // GET {portalUrl}/sharing/rest/community/self?f=json&token={token}
  // Extract: fullName, username, email
  
  // 2. Check Team Leads group membership
  // GET {portalUrl}/sharing/rest/community/groups/{groupId}/users?f=json&token={token}
  // Check if username is in the users array → _isTeamLead = true/false
  
  // 3. Load all data
  loadAllData();
}
```

### Token Management

```javascript
function getToken() {
  if (!_agolToken || Date.now() >= _agolTokenExpiry) {
    redirectToLogin(); // Token expired — re-auth
    throw new Error('Token expired');
  }
  return _agolToken;
}

// In REST API calls, check for error codes 498/499 (expired/invalid token)
// and automatically re-authenticate
```

---

## 8. ArcGIS REST API Layer

### Query Function (with pagination)

```javascript
async function agolQuery(url, where, outFields) {
  where = where || '1=1';
  outFields = outFields || '*';
  var allFeatures = [];
  var offset = 0;
  var batchSize = 1000; // ArcGIS Online limit
  
  while (true) {
    var queryUrl = url + '/query?where=' + encodeURIComponent(where)
      + '&outFields=' + outFields
      + '&f=json'
      + '&resultOffset=' + offset
      + '&resultRecordCount=' + batchSize
      + '&token=' + getToken();
    
    var response = await fetch(queryUrl);
    var data = await response.json();
    
    if (data.error) {
      if (data.error.code === 498 || data.error.code === 499) {
        redirectToLogin();
        return [];
      }
      throw new Error('ArcGIS query error: ' + data.error.message);
    }
    
    var features = data.features || [];
    allFeatures = allFeatures.concat(features);
    
    if (!data.exceededTransferLimit && features.length < batchSize) break;
    offset += features.length;
  }
  
  return allFeatures;
}
```

### Apply Edits Function

```javascript
async function agolApplyEdits(url, edits) {
  var body = new URLSearchParams();
  body.append('f', 'json');
  body.append('token', getToken());
  
  if (edits.adds && edits.adds.length > 0) {
    body.append('adds', JSON.stringify(edits.adds.map(function(a) {
      return { attributes: a.attributes || a };
    })));
  }
  if (edits.updates && edits.updates.length > 0) {
    body.append('updates', JSON.stringify(edits.updates.map(function(u) {
      return { attributes: u.attributes || u };
    })));
  }
  if (edits.deletes && edits.deletes.length > 0) {
    body.append('deletes', JSON.stringify(edits.deletes));
  }
  
  var response = await fetch(url + '/applyEdits', {
    method: 'POST',
    body: body,
  });
  var result = await response.json();
  
  // Check for errors in results
  if (result.error) throw new Error('ArcGIS applyEdits error: ' + result.error.message);
  
  return result;
}
```

### Date Handling

```javascript
// ArcGIS Date Only fields may return epoch ms or "YYYY-MM-DD" strings
// The app auto-detects and handles both:
function epochToDateStr(val) {
  if (!val) return null;
  if (typeof val === 'string' && val.match(/^\d{4}-\d{2}-\d{2}/)) return val.slice(0, 10);
  if (typeof val === 'number') {
    var d = new Date(val);
    return d.getFullYear() + '-' + pad(d.getMonth()+1) + '-' + pad(d.getDate());
  }
  return null;
}

// When WRITING Date Only fields, always send "YYYY-MM-DD" strings
// When WRITING Date and Time fields, send epoch milliseconds
```

---

## 9. Tab Implementations

### Overview Tab
- KPI summary cards in a responsive grid
- Project counts by status (Active, Completed, On Hold, Idea, Cancelled)
- Active task count, overdue task count
- Team utilization percentage (current week average)
- No sidebar, no detail views

### My Work Tab
- Personal view filtered to `_userFullName`
- **My Week section:** Shows schedule type, pay period week (A/B), hours, RDO indicator
- **KPI cards:** My active tasks, overdue tasks, weekly hours (if time tracking enabled)
- **Needs Attention:** Tasks that are overdue or stalled
- **My Tasks:** Compact list of assigned tasks grouped by status
- **My Projects:** Projects where user is contact or other_member
- **Time Tracking panel:** (only if enabled) — see Section 13

### Projects Tab
- Grid or list view toggle
- Sort by: title, status, priority, start date, end date
- Search box filters by title
- Sidebar filters: status, priority, contact, partner_dept, category, itd_team
- Click card → project detail view
- "＋ New Project" and "💡 Submit Idea" buttons
- **Project Detail:** Full metadata, problem statement, linked tasks with "＋ Add Task" button, back navigation

### Tasks Tab
- Same grid/list/sort/search/filter pattern as Projects
- Sidebar filters: status, priority, assignee, project, category, tool
- Click card → task detail view
- "＋ New Task" button
- **Task Detail:** Full metadata, parent project link, back navigation

### Resources Tab — see Section 11

### Forecast Tab
- **Capacity Planner** (top section) — "When can each person take on a new project?"
  - Dropdowns for project size (S/M/L/XL) and role (Lead/Contributor/Reviewer)
  - Runs `findEarliestStart()` for each team member
  - Shows timeline bars (existing util + proposed new project overlay)
  - Shows blocking projects when start is delayed
  - `SIZE_DURATIONS = { S: 2, M: 6, L: 13, XL: 26 }` (weeks)
  - State: `_cpSize`, `_cpRole`; functions: `cpRenderPlanner()`, `buildCapacityPlannerSection()`
- **Summary Cards** — per-person capacity cards sorted most-available first
- Forward-looking capacity using real records first, snapshot fallback for gaps
- Configurable window (default 13 weeks from current week)
- Project end dates respected — allocation freed when project ends
- Team-level utilization stacked area chart
- Per-person utilization heatmap

### Idea Review — Team Availability
- Each idea card has a collapsible "Team availability" section
- Shows when each team member can start as Lead or Contributor
- Size dropdown per idea (defaults to project's size or M)
- Uses same `findEarliestStart()` engine as Capacity Planner
- Functions: `toggleIdeaAvail()`, `renderIdeaAvail()`

### `findEarliestStart(avData, personName, projectSize, role)`
Core engine shared by Capacity Planner and Idea Review:
```javascript
// For each candidate start week W (from current week forward):
//   For each week W to W + SIZE_DURATIONS[size]:
//     needed = (allocationDefaults[size][role] / 100) × proj_cap[W]
//     if (current_alloc[W] + needed > proj_cap[W]) → fail, try next W
//   If ALL weeks pass → return { startWeek: W }
// If no slot found → return { startWeek: -1, blockers: [project names consuming capacity] }
```

### Settings Tab — see Section 14

---

## 10. Form Modals (Project & Task CRUD)

### Form Helper Functions

```javascript
// Reusable form field generators
function fmField(label, inputHtml, required, span2) {
  return '<div class="fm-field' + (span2 ? ' span2' : '') + '">' +
    '<label class="fm-label">' + label + (required ? ' <span class="fm-req">*</span>' : '') + '</label>' +
    inputHtml + '</div>';
}

function fmInput(id, value, placeholder) {
  return '<input class="fm-input" id="' + id + '" value="' + esc(value || '') + '" placeholder="' + placeholder + '">';
}

function fmSelect(id, options, selected, placeholder) {
  var html = '<select class="fm-input" id="' + id + '"><option value="">' + placeholder + '</option>';
  options.forEach(function(opt) {
    html += '<option value="' + esc(opt) + '"' + (opt === selected ? ' selected' : '') + '>' + esc(opt) + '</option>';
  });
  return html + '</select>';
}

function fmSearchableSelect(id, options, selected, placeholder, descriptions) {
  // Custom searchable dropdown with type-ahead filtering
  // Returns HTML for input + dropdown panel
}

function fmTextarea(id, value, placeholder, rows) {
  return '<textarea class="fm-input" id="' + id + '" rows="' + rows + '" placeholder="' + placeholder + '">' + esc(value || '') + '</textarea>';
}
```

### Project Form Sections (in order)
1. **Basic Info:** Title*, Status, Priority, Project Size (S/M/L/XL with "🧭 Not sure?" wizard), Primary Contact, Team Members
2. **Classification:** Category (searchable select with "🧭 Help me choose" wizard), Partner Department, ITD Team  
3. **Timeline:** Start Date, Original End Date, Working Due Date
4. **Details:** Problem Statement (textarea, span2), Description (textarea, span2)

### Task Form Sections (in order)
1. **Basic Info:** Title*, Status, Priority, Assignee
2. **Details:** Description (textarea, span2)
3. **Classification:** Project (dropdown), Tool/Technology, Category (searchable with wizard)
4. **Timeline:** Start Date, Due Date, Actual End Date
5. **Effort:** Estimated Hours, Hours Worked

### Category Wizard
- Inline 2-question decision tree below the category input
- Step 1: 5 broad options (e.g., "Building or fixing something", "Working with data")
- Step 2: Follow-up based on Step 1 answer (2-5 specific options)
- Result: Shows recommended category with "Use This Category" button
- User can go back, start over, or cancel — wizard is fully optional

### Save Logic (Project)
```javascript
function saveProject(mode) { // mode = 'new-project' or 'edit-project'
  var fields = {
    title: getValue('fm-title-val'),
    status: getValue('fm-status'),
    // ... all fields
  };
  
  if (mode === 'new-project') {
    fields.id = Math.max(...PROJECTS.map(p => p.id), 0) + 1;
    await agolApplyEdits(ARCGIS_CONFIG.projectsUrl, { adds: [{ attributes: localToAgolProject(fields) }] });
    PROJECTS.push(fields);
  } else {
    var existing = PROJECTS.find(p => p.objectId === editObjectId);
    await agolApplyEdits(ARCGIS_CONFIG.projectsUrl, {
      updates: [{ attributes: { OBJECTID: editObjectId, ...localToAgolProject(fields) } }]
    });
    Object.assign(existing, fields);
  }
  
  closeFormModal();
  render();
}
```

---

## 11. Resources & Allocation System

### Data Loading (parallel queries)

```javascript
async function loadResourcesData() {
  var [teamFeatures, absFeatures, allocFeatures] = await Promise.all([
    agolQuery(ARCGIS_CONFIG.teamMembersUrl),
    agolQuery(ARCGIS_CONFIG.absencesUrl),
    agolQuery(ARCGIS_CONFIG.allocationsUrl),
  ]);
  
  // Reconstruct RESOURCES_DATA from raw features
  RESOURCES_DATA.weeks = generateWeeks(2026);
  RESOURCES_DATA.people = {}; // Index by name
  RESOURCES_DATA.absences = {}; // { name: { weekIdx: hours } }
  RESOURCES_DATA.allocations = {}; // { name: { project: { weekIdx: {frac, hrs, oid} } } }
  
  // Then compute derived arrays:
  // proj_cap, weekly_allocated, utilization — see Section 12
}
```

### Allocation Editor

The allocation editor is a modal grid: **projects (rows) × weeks (columns)**.

**Key behaviors:**
- Enter percentages (0-100%) per cell
- Tab key moves **down the column** (column-major tabindex) — efficient for filling one week at a time
- Total row shows combined allocation — **over 100% highlighted red**
- "✓ Apply Changes" triggers save with change detection
- **Role dropdown** per project row (Lead / Contributor / Reviewer) — auto-inferred from project contact
- **⚡ Auto-fill** per project — fills empty weeks within date range using allocation defaults (project size × role)

**Role inference:**
- If person is project's `contact` → Lead
- If person is in `other_members` → Contributor
- Stored `project_role` from previous saves takes precedence
- Role can be changed manually via dropdown

**Auto-fill logic** (`aeAutofillProject`):
1. Check role is selected (prompt if not)
2. Look up project's `project_size` (prompt if not set)
3. Get default % from `_allocationDefaults[size][role]`
4. Fill only weeks where: fraction is currently 0 AND week is within project start/end date range
5. Never overwrites existing values
6. Re-renders grid after filling

**Save pipeline:**
1. Editor captures original allocations on open
2. On Apply, compare current values to original
3. Changed cells generate adds/updates/deletes:
   - New (was 0, now >0) → `adds` with {name, project, week_date, fraction, hours, project_role}
   - Changed → `updates` (delete + add pattern)
   - Removed (was >0, now 0) → `deletes` with objectId array
4. Single `agolApplyEdits()` call with batched changes
5. Resources data reloaded, UI re-rendered

### Allocation Defaults

Stored in `app_config` as key `allocation_defaults` (JSON object, not array):

```javascript
let _allocationDefaults = {
  S:  { Lead: 15, Contributor: 10, Reviewer: 5 },
  M:  { Lead: 25, Contributor: 15, Reviewer: 5 },
  L:  { Lead: 40, Contributor: 20, Reviewer: 10 },
  XL: { Lead: 50, Contributor: 30, Reviewer: 10 }
};
```

Editable in Settings tab → "Allocation Defaults by Project Size" section.

---

## 12. Capacity & Utilization Math

### Core Formula

```
weekly_hours = sum of daily_hours for each work day
  where daily_hours = (end_time - start_time) - lunch_minutes/60
  where null/empty start or end = day off (0 hours)

proj_cap = (weekly_hours - absence_hours) × 0.75 × proj_pct
  where 0.75 = productivity factor (75%)
  where proj_pct = fraction from team_members (e.g., 0.63)

allocation_hours = fraction × proj_cap
  where fraction = 0.0 to 1.0 from allocations service

utilization = sum(allocation_hours for all projects) / proj_cap
  displayed as percentage
```

### Week Type Detection (for 9/80)

```javascript
const PAY_PERIOD_REF = new Date(2025, 11, 28); // Dec 28, 2025 = Week A (Sunday)

function getPayPeriodWeek(date) {
  var daysDiff = Math.round((date - PAY_PERIOD_REF) / (1000 * 60 * 60 * 24));
  var weeksDiff = Math.floor(daysDiff / 7);
  return (weeksDiff % 2 === 0) ? 'A' : 'B';
}
// Week A: use wk1_* schedule fields, week1_hours
// Week B: use wk2_* schedule fields, week2_hours
```

### Example Calculation

```
Employee: proj_pct = 0.63, schedule = 9/80
Week B: 36 scheduled hours, 8 hours PTO

proj_cap = (36 - 8) × 0.75 × 0.63 = 13.23 hours available

If allocated 50% to Project A: hours = 0.50 × 13.23 = 6.615
If allocated 30% to Project B: hours = 0.30 × 13.23 = 3.969
Total allocated: 10.584 hours
Utilization: 10.584 / 13.23 = 80%
```

---

## 13. Time Tracking System

### Per-Employee Opt-In

- `time_tracking` field on `team_members` (String: "true"/"false")
- Toggled in Settings tab per employee
- My Work tab shows time panel only if enabled
- **Case-insensitive field detection** — ArcGIS may return field as `time_tracking`, `Time_Tracking`, etc.

### Timer Flow

```
1. User selects task from dropdown (shows In Progress / Waiting / On Hold tasks)
2. Clicks "▶ Start" → creates time_entries record:
   { name, task_idx, project_id, start_time: Date.now(), end_time: null, work_date: today }
3. Active timer shows: green pulsing dot, task name, project, elapsed time (updates every 10s)
4. Multiple concurrent timers allowed
5. Clicks "⏹ Stop" → updates record:
   { end_time: Date.now(), hours: (end - start) / 3600000 }
6. Entry appears in Today's Log with start/end times, hours, notes
```

### Entry Management

- **Edit:** Inline form with datetime-local pickers for start/end and text input for notes
- **Delete:** Remove with confirmation
- **Resume:** Green ▶ button on completed entries starts new timer on same task
- **Daily/weekly totals** displayed above the log

---

## 14. Settings Tab (Admin)

### Team Members Section

- Table: Name, Role, Team, Schedule, RDO, Proj %, Time Tracking toggle, Actions
- **Add Member:** Modal form with all fields including daily start/end time grid
- **Edit Member:** Same form, pre-populated
- **Delete Member:** Confirmation dialog (allocation and absence history preserved)
- **Absences:** Opens inline 8-week grid centered on current week

### Absence Editor

```
- 8 weeks visible at a time, scrollable left/right
- Enter hours off (0-40) per week per person
- Auto-saves each cell change (no Save button)
- Read-only "Proj Capacity" row shows computed impact
- Current week column highlighted
- "Today" button jumps to current week
```

### Dropdown List Editor

- Grid of cards, one per configurable list
- Each card shows items with remove button and add input
- Changes saved to `app_config` service immediately
- Lists: partner_depts, itd_teams, task_categories, task_tools, task_statuses, project_statuses, project_categories

### Allocation Defaults Editor

- Table with rows for each size (S, M, L, XL) and columns for each role (Lead, Contributor, Reviewer)
- Number inputs for each cell (0-100%, step 5)
- "Save Defaults" button writes to `app_config` as key `allocation_defaults`
- Rendered by `renderAllocDefaultsEditor()`, saved by `saveAllocDefaults()`

### Size Wizard

Scoring-based 4-question wizard for project sizing. Used in project form via "🧭 Not sure? Help me choose a size" link.

**Questions and scoring (1-4 points each):**
1. Duration: <2wk(1), 2-8wk(2), 2-4mo(3), 4+mo(4)
2. Team size: Solo(1), 2-3(2), 4-6(3), 7+(4)
3. Deliverables: 1(1), 2-3(2), 4-6(3), 7+(4)
4. Stakeholder complexity: Internal(1), 1 dept(2), Multi-dept(3), External(4)

**Score → Size mapping:**
- 4-6 → S (Small)
- 7-9 → M (Medium)
- 10-12 → L (Large)
- 13-16 → XL (Extra large)

**Implementation:** Data in `SIZE_WIZARD_QUESTIONS` and `SIZE_WIZARD_MAP` arrays. State in `_sizeWizardAnswers` and `_sizeWizardStep`. Functions: `sizeWizardOpen()`, `sizeWizardPick()`, `sizeWizardBack()`, `sizeWizardReset()`, `sizeWizardRender()`, `sizeWizardApply()`.

---

## 15. Edge Cases & Bug Fixes

### Critical Issues Resolved

1. **133% Utilization Bug:** Stored allocation hours became stale when capacity changed (absences added after allocations saved). Fix: Always recompute `hours = fraction × local_proj_cap` on load — never trust stored hours.

2. **Date Only field format:** ArcGIS returns Date Only as either epoch ms or "YYYY-MM-DD" string depending on service configuration. Fix: `epochToDateStr()` auto-detects and handles both. Always WRITE as "YYYY-MM-DD" strings.

3. **Task pagination:** Tasks exceed 1,000 record ArcGIS limit. Fix: Paginate with `resultOffset` until `exceededTransferLimit` is false.

4. **"end" reserved word:** ArcGIS doesn't allow "end" as field name. Fix: Store as "end_" with bidirectional field mapping.

5. **Case-insensitive field detection for time_tracking:** ArcGIS may return fields with different casing. Fix: Check all casing variants when reading the field.

6. **Week date display mismatch:** Data stores Sunday-start dates, UI shows Monday. Fix: All display functions add 1 day to the data date.

7. **Allocation editor column-major tab order:** Default browser tabindex goes left-to-right (row-major), but users fill week-by-week (column-major). Fix: Explicit tabindex assignment in column order.

8. **Stale allocation saves:** Without change detection, saving could re-write unchanged allocations or miss deletions. Fix: Capture original state on editor open, diff against current state, generate minimal adds/updates/deletes.

---

## 16. Branding & Design Specifications

### City of Tucson Brand

Follows the City of Tucson Brand Guide (02.15.22). See `COT_Brand_Guide_1.pdf` in project files.

### Colors
| Name | Hex | CSS Variable | Usage |
|------|-----|-------------|-------|
| Innovation Blue | #002669 | --navy | Header, buttons, primary text |
| Sunset Orange | #C24200 | --orange | Accents, subheadings |
| Sun Yellow | #FFDB22 | --yellow | Highlights, warnings |
| Saguaro Green | #83AC16 | --green | Success, completed |
| Sky Blue | #0088FF | --sky-blue | Links, active |
| Cactus Fruit | #9E0059 | --cactus-fruit | Department accent |

### Typography
| Typeface | Source | Usage |
|----------|--------|-------|
| Lato (300, 400, 700, 900) | Google Fonts | Headings, UI, navigation |
| Cardo (400, 700, italic) | Google Fonts | Body copy, descriptions |
| Arial | System | Email/fallback |

### Google Fonts Import
```html
<link href="https://fonts.googleapis.com/css2?family=Lato:wght@300;400;700;900&family=Cardo:ital,wght@0,400;0,700;1,400&display=swap" rel="stylesheet">
```

---

## 17. Deployment

### GitHub Pages

1. Repository: `psjohnso/analytics-tracker` (public, free plan)
2. Enable Pages: Settings → Pages → Deploy from Branch → main → / (root)
3. Files: `index.html`, `guide.html`, `PROJECT_REFERENCE.md`
4. Register GitHub Pages URL as OAuth redirect URI in ArcGIS Online
5. Changes go live in ~2 minutes after push

### Deployment Workflow

```bash
git add index.html guide.html PROJECT_REFERENCE.md
git commit -m "v{X.Y.Z.NNNN} — Description"
git push origin main
```

---

## 18. Development Conventions

- **Version format:** `MAJOR.MINOR.PATCH.BUILD` — BUILD resets to 0000 when any preceding number changes
- **Git commit:** Include version number and description in every commit
- **After every change:**
  1. Check if `guide.html` needs updating
  2. Update this document (`PROJECT_REFERENCE.md`)
  3. Copy both to outputs
- **Persistent data:** When features involve persistent data, discuss storing in ArcGIS Online before implementing
- **Code style:** Use `var` (not `let/const` in loops), `function` declarations, explicit `forEach` over arrow functions — keeps compatibility simple
- **HTML generation:** All UI is built via string concatenation in JavaScript (`html += '<div>...'`), not template literals or JSX

---

## 19. Reconstruction Checklist

### Phase 1: ArcGIS Online Infrastructure
- [ ] Create 7 feature services (tables, no geometry) per Section 3
- [ ] Enable editing (Add, Update, Delete) on all services
- [ ] Populate app_config with dropdown list values (Section 20)
- [ ] Populate team_members with team roster
- [ ] Register OAuth application → note Client ID
- [ ] Create Team Leads group → note Group ID
- [ ] Share services with organization or group

### Phase 2: Application Shell
- [ ] Create index.html with `<style>`, `<body>`, `<script>` structure
- [ ] CSS: variables, layout (header + sidebar + content), cards, modals, badges
- [ ] Load Google Fonts (Lato + Cardo)
- [ ] Implement OAuth 2.0 (redirect, token extraction, user identity, group check)
- [ ] Build agolQuery() with pagination and agolApplyEdits()
- [ ] Implement field mapping (end_ ↔ end)
- [ ] Build data loading pipeline (parallel queries, RESOURCES_DATA reconstruction)
- [ ] Implement render() with tab switching and detail views

### Phase 3: Core Features
- [ ] Overview tab (KPI cards)
- [ ] Projects tab (grid/list, sort, search, filters, detail view, CRUD modal)
- [ ] Tasks tab (same pattern as Projects)
- [ ] My Work tab (personal view, schedule display)
- [ ] Form modals with configurable dropdowns from app_config
- [ ] Category wizard (2-question decision tree)

### Phase 4: Resources & Capacity
- [ ] Resources tab (person selector, capacity chart, allocation table)
- [ ] Allocation editor modal with change detection, batch saves, role dropdowns, and auto-fill
- [ ] Project size field with scoring wizard (4-question, score→S/M/L/XL)
- [ ] Allocation defaults system (app_config, Settings editor, auto-fill integration)
- [ ] Capacity formula: (weekly_hours - absences) × 0.75 × proj_pct
- [ ] Week type detection for 9/80 schedules
- [ ] Forecast tab (snapshot projection, team heatmap, capacity planner with findEarliestStart)
- [ ] Idea Review team availability section (collapsible, per-idea size selector)

### Phase 5: Admin & Advanced
- [ ] Settings tab (Team Leads only)
- [ ] Team member CRUD with schedule configuration
- [ ] Absence editor (8-week grid, auto-save)
- [ ] Dropdown list editor (app_config CRUD)
- [ ] Time tracking system (per-employee opt-in, start/stop timers)

### Phase 6: Documentation & Deploy
- [ ] Create guide.html with matching branding
- [ ] Set up GitHub repo, enable Pages
- [ ] Register redirect URI
- [ ] Test all features end-to-end
- [ ] Verify capacity math

---

## 20. Default Configuration Values

### Task Categories
AI & Machine Learning, Application Development, Application Maintenance, Automation & Scripting, Code Review & QA, Data Discovery & Profiling, Data Engineering, Data Maintenance, Data Visualization & Reporting, Deployment & Release, Documentation, Geospatial Analysis, Infrastructure & Server Management, Meeting & Collaboration, Professional Development, Project Planning & Scoping, Research & Prototyping, Security & Compliance, Statistical Analysis, Stakeholder Support, Troubleshooting, User Training & Enablement, Other

### Task Tools
AI Assistant, Alation, ArcGIS Dashboard, ArcGIS Enterprise, ArcGIS Experience Builder, ArcGIS Field Maps, ArcGIS Hub Site, ArcGIS Instant Apps, ArcGIS Online, ArcGIS Pro, ArcGIS Python API, ArcGIS Server, ArcGIS StoryMap, ArcGIS Survey123, ArcMap, Azure/Cloud, Docker, Excel, FME, Geocortex/VertiGIS, Git/GitHub, Jupyter/Notebooks, Microsoft Forms, Microsoft Teams, Power Automate, Power BI, PowerPoint, PowerShell, Python, Qlik, SharePoint, Smartsheet, Snowflake, SQL, TimeXtender, Word, Other

### Project Statuses
Active, Completed, On Hold, Cancelled, Idea

### Task Statuses
Not Started, In Progress, Completed, On Hold, Waiting for Response, Cancelled

### Priority Levels
High, Medium, Low

### Category Wizard Decision Tree
```
Step 1: "What best describes the primary nature of this project/task?"
  → "Building or fixing something" → App Dev, App Maintenance, Automation & Scripting
  → "Working with data" → Data Engineering, Data Maintenance, Data Discovery
  → "Creating visuals or reports" → Data Viz, Geospatial Analysis, Statistical Analysis
  → "Supporting people or processes" → Stakeholder Support, User Training, Meeting & Collaboration
  → "Infrastructure or compliance" → Infrastructure, Security & Compliance, Deployment
```

---

## 21. Version History

| Version | Description |
|---------|-------------|
| 0.0–0.5.1 | Initial ArcGIS migration, OAuth, UI, sidebar, access control |
| 0.5.2–0.5.3 | Fix sidebar filter counts, Resources allocation table filter |
| 0.6.0 | Resources data migrated from embedded JSON to ArcGIS REST |
| 0.6.1–0.6.10 | Bug fixes: view toggle, utilization, week highlighting, allocation saves, Date Only |
| 0.7.0 | Settings tab: team member CRUD, absence editor (Team Leads only) |
| 0.7.1–0.7.2 | Compute capacity from formula, apply 75% productivity factor |
| 0.7.3–0.7.6 | User guide (Word + HTML), Help button |
| 0.7.7–0.7.8 | Forecast week labels, utilization recompute bug fix |
| 0.8.0+ | Configurable dropdowns via app_config, Settings list editor |
| 0.8.1+ | My Work tab with personal view |
| 0.8.2+ | Category wizard (2-question decision tree) |
| 0.8.3.0000 | Fix wizard link placement inside category field |
| 0.9.0+ | Work schedule system (5/8, 4/10, 9/80), daily start/end times |
| 0.9.1.0000 | Schedule-aware capacity formula, My Week display |
| 0.9.2+ | Time tracking: per-employee opt-in, start/stop timers, auto-calculated hours |
| 0.10.0.0000 | Project sizing (S/M/L/XL) with 4-question scoring wizard, allocation defaults by size x role, role dropdown and auto-fill in allocation editor, allocation defaults editor in Settings |
| 0.10.0.0001 | Lock Original End Date on Active projects; auto-copy Working Due Date; auto-fill allocations on new Active project creation; require size for Active |
| 0.10.0.0002 | Include Scheduled status in auto-fill allocation trigger and size requirement |
| 0.10.0.0003 | Add hover tooltips to Project Review status buttons |
| 0.10.0.0004 | Forecast uses real allocation records for all weeks, snapshot only as fallback for gaps |
| 0.10.0.0005 | Capacity Planner on Forecast tab (earliest start finder); Team Availability on Idea Review cards |

---

*End of Reconstruction Blueprint — City of Tucson Analytics Project Tracker*
