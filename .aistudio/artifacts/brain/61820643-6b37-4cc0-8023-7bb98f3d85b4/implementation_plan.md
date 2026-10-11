# Ranked Anime List Application

A mobile-first, high-performance client-side web application for creating, managing, ranking, and exporting personal anime rankings powered by AniList GraphQL search and local browser persistence.

## User Review & Critical Decisions

> [!IMPORTANT]
> The clarifying questions were dismissed, so standard recommended patterns will be adopted:
> - **Reordering**: Dedicated Up (▲) and Down (▼) touch controls with disabled states at boundary ranks (#1 cannot move up; bottom cannot move down), plus instant state persistence.
> - **Card Presentation**: Streamlined dark-mode item cards featuring cover thumbnail with fallback placeholder, rank label, title with truncation, average community score, and immediate action buttons.
> - **Scoring**: Displayed cleanly as numerical average percentage score (`84%`) or `N/A` for manual entries, adhering to zero-pill typography standards.
> - **Architecture**: Delivered as an interactive, fully compliant client-side application in `index.html` backed by Tailwind CSS dark theme (`zinc-950` canvas, `zinc-900` cards, `zinc-100` typography).

---

## 1. Overview & Core Concept

- **What It Does**: Allows anime enthusiasts to curate a personal ranked anime top-list directly on their mobile device or desktop. Users can search anime in real-time via the AniList GraphQL API or manually add niche titles, reorder items up/down, delete entries, and persist their collection in `localStorage` with JSON export and import capabilities.
- **Target Audience**: Anime viewers, seasonal watchers, and tier-list creators wanting a distraction-free, zero-latency ranking tool that works offline once loaded.
- **Key Value**: Immediate client-side responsiveness, zero account registration needed, resilient offline state via `localStorage`, and easy migration through file import/export.

---

## 2. User Experience & Visual Design

### Key User Flows
1. **Initial Zero-State**: On first load with empty storage, the app automatically pre-populates with two sample anime ("The Eminence in Shadow" and "Bleach") featuring authentic covers and scores.
2. **Search & Select**:
   - User types in the search bar.
   - After a 400ms debounce, a GraphQL query fetches matching anime from `https://graphql.anilist.co`.
   - Results appear in an absolute floating dropdown under the search input with thumbnail, romaji/english title, and score.
   - Tapping an item adds it to the bottom of the ranked list, clears the search input, and closes the dropdown.
3. **Manual Addition**:
   - For unlisted or custom titles, the user types in the "Manual Add" field and taps "Add".
   - The item is inserted at the bottom with a clean placeholder cover and `N/A` score.
4. **Reordering & Deletion**:
   - Tapping ▲ shifts the item up one rank.
   - Tapping ▼ shifts the item down one rank.
   - Tapping ✕ prompts/removes the item with instantaneous index recalculation (#1, #2, etc.).
5. **Data Backup**:
   - Tapping "Export" generates and triggers a direct file download of `my-anime-ranking.json`.
   - Tapping "Import" opens a file dialog accepting `.json`, parses the contents, validates schema, updates state, and re-renders the list.

### Visual Identity & Theme
- **Aesthetic Direction**: Tactical minimalist dark aesthetic inspired by modern media catalogs. High contrast, clean hairline dividers, zero visual clutter.
- **Color Palette & Tokens**:
  - Canvas / Background: `zinc-950` (`#09090b`)
  - Elevated Cards / Surfaces: `zinc-900` (`#18181b`)
  - Subtle Borders & Hairlines: `zinc-800` (`#27272a`)
  - Primary Text: `zinc-100` (`#f4f4f5`)
  - Secondary Text / Muted: `zinc-400` (`#a1a1aa`)
  - Accent / Focus: `sky-400` / `indigo-400` for active indicators and ratings
  - Danger / Delete: `rose-400` on hover/active
- **Typography**:
  - Sans display & body font: Modern system sans stack (`system-ui, -apple-system, sans-serif`).
  - Numerals: Tabular monospace figures (`font-mono tabular-nums`) for rank badges (`#01`, `#02`) and percentage scores.
- **Zero-Pill Discipline**:
  - Rank and scores rendered as unboxed typographic markers with subtle tonal separators rather than bubble chips.
  - Buttons styled with tactile geometric borders and soft hover highlights.
- **Touch Ergonomics**:
  - Touch targets for ▲, ▼, and ✕ sized to $\ge 44\text{px}$ touch hitboxes.
  - Sticky header capped under 15% of the viewport height.

---

## 3. Key Product Decisions & Trade-Offs

- **AniList GraphQL API via Direct Client Fetch**:
  - *Decision*: Query `https://graphql.anilist.co` directly via standard `fetch(..., { method: 'POST' })` with `Content-Type: application/json`.
  - *Why*: AniList allows CORS requests directly from web browsers without requiring an API key for public queries.
  - *Alternative*: Server proxy was considered, but direct client requests satisfy the zero-backend constraint perfectly.
- **Search Debounce (400ms) with AbortController**:
  - *Decision*: Debounce user input with 400ms delay and cancel pending in-flight requests using `AbortController`.
  - *Why*: Prevents race conditions where an older query returns after a newer one and protects AniList rate limits.
- **Client-Side Storage**:
  - *Decision*: Store data under a designated key `my_anime_ranking_v1` in `window.localStorage`.
  - *Why*: Instant synchronous load and save, zero latency, persistence across page reloads.
- **Resilient Image Fallback**:
  - *Decision*: SVG data-URI / styled container placeholder for manual items or broken image URLs (`referrerPolicy="no-referrer"`).
  - *Why*: Guarantees zero broken image frames even if network or hotlink restrictions occur.

---

## 4. Technical Architecture & Data Strategy

### System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      Sticky Header Bar                       │
│  [ Title: "My Ranking" ]            [ Export ]  [ Import ]   │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────┴───────────────────────────────┐
│                    Search & Add Section                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ [🔍 Search AniList...] (400ms Debounce + Cancel)       │  │
│  │  └─► Floating Autocomplete Dropdown (Results list)     │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────┐ ┌──────────────────┐  │
│  │ [ Manual title input...         ] │ │ [ + Add Manual ] │  │
│  └───────────────────────────────────┘ └──────────────────┘  │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────┴───────────────────────────────┐
│                  Ranked Scrollable List                      │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ #1 │ [Thumb] │ Title (Truncated) │ 88% │ [▲] [▼] [✕]   │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │ #2 │ [Thumb] │ Title (Truncated) │ 82% │ [▲] [▼] [✕]   │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬───────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
       [ localStorage ]             [ File System (.json) ]
   Key: my_anime_ranking_v1        Export / Import Handler
```

### Data Model

```typescript
interface AnimeItem {
  id: string; // Unique UUID or timestamp-based ID
  title: string; // Display title (English or Romaji)
  coverImage: string; // URL or placeholder SVG data URI
  averageScore: number | null; // e.g., 85, or null for manual additions
  anilistId?: number; // Optional AniList ID if from API
}
```

### AniList GraphQL Query

```graphql
query ($search: String) {
  Page(page: 1, perPage: 8) {
    media(search: $search, type: ANIME, sort: SEARCH_MATCH) {
      id
      title {
        romaji
        english
      }
      coverImage {
        large
        medium
      }
      averageScore
    }
  }
}
```

### Verification & Testing Plan
- Verify search debounce and AniList response rendering.
- Verify manual entry with placeholder graphics.
- Verify ranking adjustment (▲ / ▼) with boundaries disabled appropriately.
- Verify deletion and index re-numbering.
- Verify localStorage persistence across reloads.
- Verify JSON export generation and JSON import parsing with schema validation.
- Run `compile_applet` to ensure seamless web compilation.
