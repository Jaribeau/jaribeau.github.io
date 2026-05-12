# FitNotes Web Viewer — Technical Design Doc

## Overview

A static single-page web app for browsing personal workout history exported
from the FitNotes Android app. Optimized for desktop. No backend, no auth, no
user accounts.

- **v1 entrypoint:** drag-and-drop CSV upload.
- **v2 (future):** load CSV(s) from a public Google Drive folder via Drive API key.

## Goals

- Desktop-optimized read-only viewer for FitNotes workout data.
- Single-binary static site, deployable to any static host (GitHub Pages,
  Netlify, Cloudflare Pages, S3 static, etc).
- Drag-and-drop a CSV export and immediately see useful views.
- Persist the last-loaded dataset locally so the user doesn't re-upload every
  visit.
- Fast enough to handle several years of workout data (~5k–50k rows) without
  UI lag.

## Non-goals

- Editing or syncing data back to FitNotes.
- Mobile-first UI. (Responsive enough not to break, but desktop is primary.)
- User accounts or any server-side persistence.
- Real-time sync. The user re-uploads / reloads from Drive when they want
  fresh data.

## Data source: FitNotes CSV format

FitNotes Android exports a CSV via **Settings → Export → Export Workouts to
CSV**. Expected column set (verify against the user's actual export — header
strings drift slightly across versions):

| Column          | Type                | Notes                                          |
|-----------------|---------------------|------------------------------------------------|
| Date            | `YYYY-MM-DD`        | e.g. `2024-10-15`                              |
| Exercise        | string              | e.g. `Barbell Bench Press`                     |
| Category        | string              | e.g. `Chest`                                   |
| Weight (kgs)    | float \| empty      | Either kgs **or** lbs is populated, not both   |
| Weight (lbs)    | float \| empty      |                                                |
| Reps            | int \| empty        |                                                |
| Distance        | float \| empty      | Cardio entries                                 |
| Distance Unit   | string \| empty     | e.g. `Kilometers`, `Miles`                     |
| Time            | `HH:MM:SS` \| empty | Cardio / timed entries                         |
| Comment         | string \| empty     |                                                |

Notes:
- Each row = one set (strength) or one cardio entry.
- iOS version adds a `Kind` column and uses slightly different header strings
  (`Weight (kg)` vs `Weight (kgs)`, `Notes` vs `Comment`). The schema parser
  should be lenient: case-insensitive header match, tolerate trailing-`s`
  variations, and skip unknown columns.
- Empty string and absent field should both normalize to `null`.

## Tech stack

| Layer              | Choice                              | Reason                                                    |
|--------------------|-------------------------------------|-----------------------------------------------------------|
| Build / dev        | Vite                                | Fastest static SPA toolchain                              |
| Framework          | React 18 + TypeScript               | Component model fits; TS pays off for data-heavy app      |
| Styling            | Tailwind CSS                        | Fast iteration, no design-system overhead                 |
| CSV parsing        | Papa Parse                          | Handles quoted commas, streaming for big files            |
| Charts             | Recharts                            | Clean React API, sufficient for line / bar / heatmap      |
| Dates              | date-fns                            | Tree-shakeable, no moment baggage                         |
| Routing            | React Router                        | Per-exercise / per-workout deep-linkable URLs             |
| Local persistence  | idb-keyval (IndexedDB)              | Datasets may exceed localStorage's 5 MB limit             |
| State              | React Context + useReducer          | Reach for Zustand only if it gets messy                   |

No backend. No service worker in v1.

## Architecture

```
┌────────────────────────────────────────────┐
│ Browser (static SPA)                       │
│                                            │
│  ┌────────┐    ┌──────────┐    ┌────────┐  │
│  │ Drop   │ -> │ Parse +  │ -> │ Store  │  │
│  │ zone   │    │ normalize│    │(memory │  │
│  │        │    │          │    │  + IDB)│  │
│  └────────┘    └──────────┘    └────────┘  │
│                                    │       │
│                                    v       │
│                          ┌──────────────┐  │
│                          │ Pages / views│  │
│                          │ (Dashboard,  │  │
│                          │  Exercises…) │  │
│                          └──────────────┘  │
└────────────────────────────────────────────┘
```

Flow:

1. User drops a CSV (or selects via file picker).
2. Papa Parse reads it; rows → `WorkoutEntry[]`.
3. Normalized data goes into the in-memory store and is persisted to
   IndexedDB under a single key.
4. On page load, the app hydrates from IndexedDB if present; otherwise it
   shows the drop zone full-screen.
5. All views derive from the same in-memory dataset.

## Data model

```ts
// Raw row after CSV parse, before normalization
type RawRow = {
  Date: string;
  Exercise: string;
  Category: string;
  'Weight (kgs)'?: string;
  'Weight (lbs)'?: string;
  Reps?: string;
  Distance?: string;
  'Distance Unit'?: string;
  Time?: string;
  Comment?: string;
};

// Normalized in-app record. Internally everything is kg / km / seconds.
type WorkoutEntry = {
  id: string;              // hash(date + exercise + row-index)
  date: string;            // ISO YYYY-MM-DD, kept as string for grouping
  exercise: string;
  category: string;
  weightKg: number | null;
  originalUnit: 'kg' | 'lbs' | null;
  reps: number | null;
  distanceKm: number | null;
  timeSeconds: number | null;
  comment: string | null;
};

type Dataset = {
  entries: WorkoutEntry[];
  loadedAt: string;        // ISO timestamp
  sourceFileName: string;
};
```

Display unit (kg vs lbs) is a separate user preference, persisted in
localStorage. Storage is always kg.

## Derived data

Computed lazily and memoized via `useMemo` keyed on dataset identity. Pure
functions in `lib/analytics.ts`, no React:

- **Exercise index**: `Map<exerciseName, { category, firstSeen, lastSeen, totalSets }>`
- **Workouts**: `Map<date, WorkoutEntry[]>`
- **Personal records per exercise**:
  - Heaviest weight (any reps).
  - Best e1RM (Epley: `weight * (1 + reps/30)`).
  - Highest single-set volume (`weight * reps`).
  - Highest single-workout volume for that exercise.
- **Volume over time** (per exercise and overall).
- **Training frequency**: workouts per week / month.
- **Category breakdown**: sets per category over a time window.

## Features (v1)

### Landing / drop zone

- Full-screen drop target when no dataset is loaded.
- Also accepts file-picker click.
- Shows parse progress for large files.
- Surfaces parse errors clearly (row number, column, raw value).
- "Load sample data" button for first-time visitors.

### Dashboard (`/`)

- Headline stats: total workouts, total sets, date range, most-trained
  exercise, current week vs prior week volume.
- Training-frequency heatmap (GitHub-style calendar grid).
- Recent workouts (last 10) with quick-link to detail.

### Workouts list (`/workouts`)

- Reverse-chronological, paginated or virtualized.
- Filter by date range, category, exercise.
- Each row expands to show all sets of that workout.

### Workout detail (`/workouts/:date`)

- All exercises performed that day.
- Sets table per exercise.
- Total volume, duration if available.
- Prev / next navigation between workouts.

### Exercises catalog (`/exercises`)

- All exercises grouped by category.
- Sortable by frequency, last performed, total volume.
- Click → exercise detail.

### Exercise detail (`/exercises/:name`)

- Top-line PRs (heaviest weight, best e1RM, highest-volume set).
- Weight-progression chart (top set per workout over time).
- Volume-per-workout chart.
- Full set history table.

### Global controls

- Unit toggle (kg / lbs) in header — persists in localStorage.
- "Reload data" button (clears IndexedDB, returns to drop zone).
- Dataset metadata (filename, loaded date) in footer.

## File structure

```
fitnotes-viewer/
├── public/
│   └── sample.csv              # small synthetic dataset for the demo button
├── src/
│   ├── main.tsx
│   ├── App.tsx                 # router + layout
│   ├── types.ts
│   ├── lib/
│   │   ├── csv.ts              # Papa Parse wrapper
│   │   ├── normalize.ts        # RawRow -> WorkoutEntry
│   │   ├── analytics.ts        # all derived calculations
│   │   ├── storage.ts          # idb-keyval wrapper
│   │   ├── units.ts            # kg/lbs conversion + format
│   │   └── dates.ts            # date helpers
│   ├── store/
│   │   └── DatasetContext.tsx  # provider + hook
│   ├── components/
│   │   ├── DropZone.tsx
│   │   ├── Layout.tsx
│   │   ├── Sidebar.tsx
│   │   ├── UnitToggle.tsx
│   │   ├── Heatmap.tsx
│   │   ├── StatCard.tsx
│   │   └── charts/
│   │       ├── ProgressionChart.tsx
│   │       └── VolumeChart.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Workouts.tsx
│   │   ├── WorkoutDetail.tsx
│   │   ├── Exercises.tsx
│   │   └── ExerciseDetail.tsx
│   └── styles/
│       └── globals.css
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
├── package.json
└── README.md
```

## Future enhancements (out of scope for v1)

### Google Drive folder loading

- Settings panel: Drive API key + folder URL/ID.
- Button-triggered refresh: list CSVs in folder, pick newest.
- API key stored in localStorage, restricted in Google Cloud Console to
  (a) Drive API only, (b) the deployed site's HTTP referrer.
- Endpoint: `GET https://www.googleapis.com/drive/v3/files?q='{folderId}'+in+parents&key={API_KEY}`
- Download via `files/{id}?alt=media&key={API_KEY}`.

### Other

- Body-weight tracking (FitNotes has a separate body tracker export).
- Period-over-period comparisons (this month vs last).
- Routine / split detection from workout patterns.
- Cardio analytics (pace, distance trends).
- Export charts as images.
- PWA + offline support (manifest + service worker).

## Open questions / to verify

1. **Confirm exact header strings** on the user's current FitNotes Android
   export. The normalizer should fall back gracefully if e.g. `Weight (kgs)`
   is actually `Weight (kg)`.
2. **Date format**: confirm always `YYYY-MM-DD` for the user's locale. If
   not, add locale-aware parsing.
3. **Top-line PR definition**: heaviest weight, best e1RM, or both side by
   side? Default: show both.
4. **Large-file behavior**: at what row count do we need streamed parsing /
   virtualized lists? Likely not needed under ~50k rows.
5. **Heatmap encoding**: workout-yes/no, set count, or volume? Default:
   workout-yes/no for simplicity.

## Deployment

`npm run build` produces a `dist/` folder. Drop it on any static host. No
environment variables required for v1.
