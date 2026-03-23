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
| **app_config** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/app_config/FeatureServer/0` | Key-value config store for shared dropdown lists (partner_depts, itd_teams) |
| **time_entries** | `https://services3.arcgis.com/9coHY2fvuFjG9HQX/ArcGIS/rest/services/time_series/FeatureServer/0` | Employee time tracking entries (start/stop timer records per task per day) |

### Key Fields by Service
- **team_members:** `name`, `role`, `team`, `skill`, `proj_pct`, `schedule_type`, `rdo_day`, `lunch_minutes`, `wk1_mon_start`–`wk1_fri_end`, `wk2_mon_start`–`wk2_fri_end`, `time_tracking` (legacy fields `week1_hours`, `week2_hours`, `wk1_mon`–`wk2_fri` still on service but computed client-side from start/end/lunch)
- **absences:** `name`, `week_date` (Date Only), `absence_hours`
- **allocations:** `name`, `project`, `week_date` (Date Only), `fraction`, `hours`, `project_status`, `project_type`, `analytics_id`
- **time_entries:** `name`, `task_idx`, `project_id`, `start_time` (Date and Time), `end_time` (Date and Time, nullable = timer running), `hours` (auto-calculated), `work_date` (Date Only), `notes`

### Access Control
- **Team Leads Group ID:** `2bd32af20dd745d6bdf3807446761973` (same group controls Idea promotion and Settings tab access)
- Settings tab and Idea promotion are restricted to members of this group

### ArcGIS Online Setup Requirements
- **Enable editing** on: `projects`, `tasks`, `team_members`, `absences`, `allocations`, `time_entries` (aka `time_series`)
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
proj_cap = (scheduled_hours - absence_hours) × 0.75 × proj_pct

Where scheduled_hours is computed from daily start/end times and lunch:
  daily_hours = (end_time - start_time) - lunch_minutes / 60
  week_hours = sum of daily_hours for Mon-Fri

Week A uses wk1_*_start/end fields, Week B uses wk2_*_start/end fields.
Pay period reference: 2025-12-28 = Week A start (Sunday). Alternates every 7 days.
Week A = week1_hours, Week B = week2_hours.
```
- **scheduled_hours** = computed from TimeOnly start/end fields + lunch_minutes (not stored)
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
| 0.8.3.0000 | Fix wizard link placement: embed inside category field for clear visual association |
| 0.8.3.0001 | Redesign wizard UX: link inline with CATEGORY label; clicking transforms the field into the wizard, then restores when done |
| 0.8.3.0002 | Searchable category dropdown: on focus, show all options and select text so user can easily change selection |
| 0.8.3.0003 | Convert Partner Department and ITD Team to select dropdowns with mergeEnums (18 departments, 4 ITD teams) |
| 0.8.3.0004 | Add Settings tab dropdown list editor for Partner Departments and ITD Teams; update guide with documentation |
| 0.8.4.0000 | Task status filters always visible; add Partner Department and ITD Team sidebar filters built from project data; update guide |
| 0.8.4.0001 | Fix project navigation bug: switch all UI navigation from p.id to p.objectId to prevent wrong project opening on click |
| 0.8.4.0002 | Persist Partner Department and ITD Team list edits to localStorage so changes survive page reloads |
| 0.8.4.0003 | Switch dropdown list persistence from localStorage to ArcGIS Online app_config service for cross-user shared storage |
| 0.9.0.0000 | Add My Work tab: personalized dashboard with KPIs, action items, projects, tasks, and weekly allocation breakdown; visible to logged-in users |
| 0.9.0.0001 | Fix login bug: displayName used before declaration caused fetchAgolUserInfo to fail silently; user name, My Work tab, and Settings tab never appeared |
| 0.9.0.0002 | My Work tab: move My Week section to top (below KPIs); two-column layout for remaining sections (left: Attention + Tasks, right: Projects) |
| 0.9.0.0003 | My Work tab: Attention Needed full-width below My Week; two columns now Projects (left) / Tasks (right); tasks default to In Progress + Waiting for Response with expandable toggle |
| 0.9.0.0004 | My Work tab: Attention Needed items in compact 3-column grid (responsive to 2-col and 1-col); smaller card styling |
| 0.9.0.0005 | My Work tab: My Week and Attention Needed match full grid width; Projects default to Active/Scheduled/Idea with expandable toggle for rest |
| 0.9.0.0006 | My Work tab: KPI cards moved inside the grid to match exact width of My Week, Attention, Projects, and Tasks sections |
| 0.9.0.0007 | Back button remembers origin tab: navigating from My Work returns to My Work; dynamic labels (Back to My Work / Back to Projects / Back to Project) |
| 0.9.0.0008 | My Work tab: Open Tasks KPI now counts only In Progress, On Hold, and Waiting for Response |
| 0.9.0.0009 | My Work Attention Needed: overdue/due from open tasks only; incomplete data from all tasks and projects (except Complete/Canceled); removed stale tasks alert |
| 0.9.1.0000 | Add employee work schedule support (5/8, 4/10, 9/80); schedule-aware capacity formula using pay period Week A/B; schedule fields in member form and Settings table; My Week shows schedule/RDO info |
| 0.9.1.0001 | Fix member save: detect schedule field names from service (case-insensitive); only send schedule fields if service supports them |
| 0.9.1.0002 | Fix member save: remove duplicate ObjectId/OBJECTID in update payload; add debug logging |
| 0.9.1.0003 | Add daily work hours (wk1_mon–wk2_fri) to member form with auto-fill from schedule type; week totals auto-calculated; My Week shows daily hours bar |
| 0.9.2.0000 | Add time tracking system: per-employee opt-in via time_tracking field; start/stop timer with multiple concurrent timers; auto-calculated hours; editable time entries; today's log with daily/weekly summaries; Settings toggle per employee |
| 0.9.2.0001 | Fix time tracking toggle: separate save from time entry loading to prevent cascade failure; better error message when time_tracking field is missing |
| 0.9.2.0002 | Fix DateOnly field handling: work_date sent as "YYYY-MM-DD" string (not epoch ms); date comparison uses string matching for today/week filters |
| 0.9.2.0003 | Add Task button on project detail page: opens new task form with project pre-selected |
| 0.9.2.0004 | My Work tab: add "Project:" prefix to My Week allocation rows; add "Task:"/"Project:" prefix to Attention Needed alert titles |
| 0.9.2.0005 | My Work My Week: allocation project rows are clickable, navigate to project detail page with back button returning to My Work |
| 0.9.2.0006 | Auto-derive daily hours from schedule_type and rdo_day when daily fields are all zero (handles existing records that haven't been edited) |
| 0.9.3.0000 | Replace stored daily hours with start/end time model: 20 TimeOnly fields (wk1/wk2 × mon-fri × start/end) + lunch_minutes; hours computed client-side; member form uses time pickers; My Week shows work window times |
| 0.9.3.0001 | Widen member form modal from 440px to 720px to fit daily schedule time picker grid |
| 0.9.3.0002 | Fix TimeOnly save: send "HH:MM:SS" strings instead of milliseconds for ArcGIS esriFieldTypeTimeOnly fields |
| 0.9.3.0003 | Business rule: project names must be unique (case-insensitive); enforced on both project create/edit and idea submission |
| 0.9.3.0004 | Resource allocation editor: project names are clickable links to project detail page; closes editor and navigates to project |
| 0.9.3.0005 | Fix absence save: remove duplicate ObjectId/OBJECTID in update payload; also fix app_config save to use OBJECTID |
| 0.9.3.0006 | Allocation editor: filter by current project status (not stale allocation status); hide Canceled and Complete projects |
| 0.9.3.0007 | Fix back button labels for all origin tabs (Back to Resources, Back to Tasks, etc.); sort allocation editor projects by status: Active → On Hold → Scheduled → Future → Idea |
| 0.9.3.0008 | Time tracking: add Pause button (yellow, stops timer for resuming); Resume button (green ▶) on today's log entries; one Resume per task, hidden if timer already active |
| 0.9.3.0009 | Fix task-project join: use project title as primary match (unique, reliable) with project_id as fallback; fixes tasks from wrong project appearing on detail page |
| 0.9.3.0010 | Show task hours (team total + personal) in 5 locations: task detail, project detail with total row, My Work tasks, time tracking panel cumulative, tasks tab list/grid |
| 0.9.3.0011 | Schedule-aware timer: stopTimer and saveTimeEntryEdit now calculate hours based on employee's daily work window, capping overnight/weekend timers to actual work hours |
| 0.9.3.0012 | Time tracking panel: add "Recent Entries" section showing past 7 days with date separators, editable entries, and expand/collapse toggle |
| 0.9.3.0013 | Task hero: solid amber background (no gradient); Complete button (✓) in 3 locations: task detail hero, project detail task rows, My Work task rows; markTaskComplete sets status + actual_end date |
| 0.9.3.0014 | Prevent browser back button from navigating to OAuth login pages after authentication; history guard using pushState/popstate |
| 0.9.3.0015 | Fix browser back button guard: push 5 buffer history entries + popstate listener to prevent reaching cross-origin ArcGIS login page |
| 0.9.3.0016 | Redesign back button guard: force page reload after OAuth token extraction so synchronous guard at script top runs first; prevents app from partially initializing before guard is active |
| 0.9.3.0017 | Fix back button guard: detect OAuth return via hash fragment (access_token in URL) in addition to sessionStorage; consolidate to single guard; remove duplicate popstate listener |
| 0.9.3.0018 | Fix login loop: extract and store OAuth token in the synchronous guard BEFORE pushState wipes the URL hash; clean URL with replaceState after token is safely in sessionStorage |
| 0.9.3.0019 | Allocation editor: only show Active projects (exclude On Hold, Scheduled, Future, Idea, Canceled, Complete) |
| 0.9.3.0020 | Attention Needed: add stale project detection — Active projects with no edits in 14+ days show with ⏳ icon and days since last update; requires editor tracking enabled on projects service |
| 0.9.3.0021 | Attention Needed: expand stale check to Active, On Hold, Waiting for Response; add 🚀 alert for Future/Scheduled projects past their start date |
| 0.9.3.0022 | Working due date + actual end date: auto-set working_due=due on create; auto-set actual_end=today on Complete; overdue/alerts use working_due; edit forms include working_due field; detail pages show original/working/actual dates; all list/grid/card views show working_due |
| 0.9.3.0023 | Fix admin user selector on My Work tab: await fetchAgolUserInfo before first render so _isTeamLead is set in time |
| 0.9.3.0024 | My Work: remove Complete and Canceled from expand sections; projects expand shows On Hold and Future only; tasks expand shows Pending and On Hold only |
| 0.9.3.0025 | Attention Needed: add overdue project alerts (Active/On Hold/Waiting for Response past working_due/end); add project due-this-week alerts; overdue KPI includes projects |
| 0.9.3.0026 | Lock original due date fields (end/due) on edit forms — disabled after creation to preserve baseline; tooltip directs users to Working Due Date |
| 0.9.3.0027 | My Work: add Gantt chart timeline showing active/on-hold/scheduled projects with nested task bars, today marker, month headers, overdue highlighting, clickable labels |
| 0.9.3.0028 | My Work: make Gantt chart collapsible and collapsed by default; click header to expand/collapse with ▶/▼ arrow indicator |
| 0.9.3.0029 | Gantt chart: add project filter with checkboxes, All/None buttons; filter persists while toggling; resets on user switch; chart body rebuilds without full re-render |
| 0.9.3.0030 | Gantt chart: collapsible projects (▶/▼ arrow hides/shows tasks); dropdown checklist replaces inline checkboxes; color legend for status bars and overdue indicator |
| 0.9.3.0031 | Fix lunch deduction: only subtract lunch when timer spans across the lunch hour (before noon AND after 1pm); afternoon-only and morning-only sessions no longer penalized |
| 0.9.3.0032 | Attention Needed: replace static "And xx more items" message with expandable button showing remaining alert cards |
| 0.9.3.0033 | Fix Attention Needed overflow: all cards in single grid so expanded items follow same 3-column layout and spacing as initial items |
| 0.9.3.0034 | My Work: add jump link nav bar for quick scrolling to Time, Week, Attention, Timeline, Projects, Tasks sections; smooth scroll with section IDs |
| 0.9.3.0035 | Sticky tab bar and jump links: remove content-area overflow so page body scrolls; jump nav sticks below tab bar; hide view toggle on Forecast tab; scroll-margin-top for section anchors |
| 0.9.3.0036 | My Work: unified sticky header containing greeting, date, employee picker, and jump links; sticks below tab bar as a single block while scrolling |
| 0.9.3.0037 | Opaque My Work sticky header (var(--surface)); move sidebar filters to horizontal bar below tab bar with pill-style buttons; sidebar sticky below tab bar; hidden on My Work/Settings tabs |
| 0.9.3.0038 | Revert sidebar to left column; move tab bar above sidebar+main wrapper so it spans full page width; sidebar stays as vertical column below it |
| 0.9.3.0039 | Move toolbar (result count, weekly review, sort, view toggle) to full width below tab bar; eliminates gap between sidebar and content |
| 0.9.3.0040 | My Work: remove gap above sticky header (negative top margin eats content-area padding); remove max-width cap so header matches section widths |
| 0.9.3.0041 | My Work: add ↑ Top button to jump links (right-aligned); restore max-width:1100px to constrain page width on wide screens |
| 0.9.3.0042 | My Work jump links: move ↑ Top to left of Time; combine Projects & Tasks into single button; sticky header explicit width:100% to match sections |
| 0.9.3.0043 | Fix My Work sticky header width: move header outside mywork-page; use negative margins to span full content-area; inner container constrained to match section max-width |
| 0.9.3.0044 | Fix jump link scroll offset: increase to 210px so section headers aren't hidden behind sticky elements |
| 0.9.3.0045 | Jump links: dynamic scroll offset measuring actual sticky header height |
| 0.9.3.0046 | Task editor: separate task category list (22 categories with descriptions); task-specific category wizard (2-step guided picker); project form keeps its own categories/wizard |
| 0.9.3.0047 | Add "Other" category to both project and task lists with description; add "None of these fit" fallback option to every step of both category wizards |
| 0.9.3.0048 | Task tool field: replace plain dropdown with searchable select (37 tools with descriptions); add tool wizard (2-3 step guided picker); "Not sure? Help me choose" link; Other fallback at every step |
| 0.9.3.0049 | Settings tab: add editors for Project Categories, Task Categories, and Task Tools with name+description; inline edit/remove/add; auto-save to ArcGIS Online; saveConfigKey creates new records when no OID exists |
| 0.9.3.0050 | Fix crash: description maps (CATEGORY_DESCRIPTIONS etc.) were calling buildDescMap before it was defined; initialize as empty objects, populated by refreshEnums on first data load |
| 0.9.3.0051 | Add Task Category and Task Tool sidebar filters; shown in Task Filters group; top 10 by count; cleared on tab switch; filterTasks uses taskCategory/taskTool instead of project category |
| 0.9.3.0052 | Fix crash after task edit: activeFilters reset was missing taskCategory and taskTool, causing filterTasks to throw on undefined.length |
| 0.9.3.0053 | Task filters: show all categories and tools instead of top 10; sort alphabetically |
| 0.9.3.0054 | Remove top-N limits from all sidebar filters (project category, team member, dept, team); sort all alphabetically |
| 0.9.3.0055 | Task tools: add active/retired flag; retired tools hidden from new task dropdown but shown on existing tasks; Settings editor shows active toggle with retired styling; new tools default to active |
| 0.9.3.0056 | Fix: always show active/retired toggle for task tools regardless of stored data |
| 0.9.3.0057 | Settings: replace dot toggle with Active/Retired word labels in green/red |
| 0.9.3.0058 | Add error handling to saveConfigKey with per-record failure detection and logging |
| 0.9.3.0059 | Compress config JSON for ArcGIS storage: use short keys (n/d/a) to fit within 4000-char field limit; expand on load |
| 0.9.3.0060 | Project detail: remove original end date from meta grid; add actual end date badge to hero panel |
| 0.9.3.0061 | Project detail: add Project Timeline section showing tasks as phase bars with dates, status colors, today marker, overdue outlines, complete hatching, and legend |
| 0.9.3.0062 | Project detail timeline: move above problem statement; redesign as single bar with colored segments per task (status = color); dynamic legend showing only used statuses; clickable segments |
| 0.9.3.0063 | Project detail timeline: redesign as project-level bar (not task-level); single bar colored by project status; markers for Start, Due, Done, Today; overdue striping when past due |
| 0.9.3.0064 | Project timeline: full-width bar; remove month header |
| 0.9.3.0065 | Project timeline: use status_history service for colored phase segments; fallback to single bar if no history |
| 0.9.3.0066 | Project timeline: add Edit History button (team leads only); inline editor to add/remove status history records with date picker; re-renders timeline bar after changes |
| 0.9.3.0067 | Project timeline: show completion date right-aligned below bar for completed projects; remove old start/completed row |
| 0.9.3.0068 | Add Developer Mode toggle in Settings (admin only); controls visibility of Edit History and future advanced tools; persists in sessionStorage; clears on logout and for non-admins |
| 0.9.3.0069 | Remap all status colors to City of Tucson brand palette: Saguaro Green (Active), Sky Blue (Complete), Sunset Orange (Future/Pending), Sun Yellow (On Hold), Cactus Fruit (Scheduled), Sonoran Sand (Idea), Tucson Blue (Waiting), Monsoon Gray (Canceled) |
| 0.9.3.0070 | Project timeline: filter bar to only show Active, On Hold, Waiting for Response phases; add start date on first segment; use project start date as range start; completion date uses Sky Blue brand color |
| 0.9.3.0071 | Project timeline: move completion date into labels row at right edge of bar |
| 0.9.3.0072 | (reverted in 0073) Status history fallback for completion date |
| 0.9.3.0073 | Revert 0072; completion date only from project actual_end field |
| 0.9.3.0074 | Guide: open via window.open() so window.close() works from back button; preserves tracker OAuth session |
| 0.9.3.0075 | Add Complete button to project detail page hero actions; markProjectComplete function |
| 0.9.3.0076 | Active/inactive team members: active flag on team_members; inactive members hidden from form dropdowns, sidebar filters, admin view-as selector; Settings shows Active/Inactive toggle per member; "Show former members" checkbox in sidebar filter |
| 0.9.3.0077 | Enable team member name editing with cascading rename across all services (projects, tasks, time entries, allocations, absences, status history); confirmation dialog; loading overlay during rename; local arrays updated immediately |
| 0.9.3.0078 | Settings: add Show former toggle to hide inactive members from team table |
| 0.9.3.0079 | Idea form: change department to dropdown using Partner Departments list; use active members for Submitted By |
| 0.9.3.0080 | Idea form: remove Description/Proposed Solution field |
| 0.9.3.0081 | Idea form: add Priority dropdown and Urgency/Timeline Notes field |
| 0.9.3.0082 | Idea review: priority badge, urgency section, submission date, sort by priority, reviewer notes |
| 0.9.3.0083 | Idea review cards: remove description/urgency/notes section |
| 0.9.3.0084 | Fix: remove stale isUrgency reference that broke idea review rendering |
| 0.9.3.0085 | Use urgency_notes and reviewer_notes fields; keep description clean for project descriptions |
| 0.9.3.0086 | Remove 60KB stale form HTML blob baked into source file |
| 0.9.3.0087 | Phase 1 code quality: debounce search (200ms); convert all var to const/let (0 var remaining; 1234 const, 134 let) |
| 0.9.3.0088 | Phase 2: extract sub-functions from 5 monster functions; 8 new helper functions; eliminated duplicate code in time entry rows and task rows |
| 0.9.3.0089 | Phase 3: move 63 inline styles to 28 CSS classes; clean 18KB stale allocation editor blob; file size 781KB→701KB |
| 0.9.3.0090 | Phase 4: targeted re-renders with _dataDirty flag; skip buildSidebarFilters on 16 of 35 render() calls; extract updateHeaderStats and updateTabCounts |
| 0.9.3.0091 | Phase 5: consolidate 25 globals into 3 namespace objects (Auth, Editor, Internal); 77→55 total globals; 165 reference replacements |
| 0.9.3.0092 | Phase 6: replace all 46 blocking alert() calls with non-blocking showToast(); add error/warn/info toast variants with color-coded display and scaled duration |
| 0.9.3.0093 | Fix: switch all task navigation/CRUD from idx to objectId; fixes tasks with null idx always opening wrong project; DataStore getTask/updateTask/deleteTask now find by objectId |
| 0.9.3.0094 | Fix: restore Editor.draft initialization to {} (was null from Phase 5); restore missing aeProjectDates declaration (lost in Phase 5 consolidation); fixes Edit Allocations button |
| 0.9.3.0095 | Attention Needed: restrict alerts to Active/In Progress/Scheduled statuses only; Pending/On Hold/Waiting for Response/Future/Idea items no longer flagged |
| 0.9.3.0096 | Attention Needed: add On Hold and Waiting for Response back to missing-data checks (projects and tasks) |
| 0.9.3.0097 | Phase A: unify task status 'In Progress' → 'Active' across all code (15 logic edits, 2 label fixes); STATUS_COLOR_MAP fallback kept until data migration |
| 0.9.3.0098 | Phase C: remove 'In Progress' fallback from STATUS_COLOR_MAP; migration complete |

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
- **Version format:** `MAJOR.MINOR.PATCH.BUILD` (e.g., `0.8.3.0000`). BUILD resets to `0000` whenever MAJOR, MINOR, or PATCH changes; increments within the same version prefix.
- **Git commit message:** Include with every new version
- **Guide check:** After every app update, check if `guide.html` needs updating
- **This document:** Update after every app change to keep in sync
- **Persistent data:** When requests involve persistent data, discuss storing in ArcGIS Online as new or existing REST service before implementing
