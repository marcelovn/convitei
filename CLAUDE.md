# CLAUDE.md — Convitei (Nogue Convites)

This file provides AI assistants with essential context about this codebase.

---

## Project Overview

**Convitei** is an interactive digital invitation platform with a gamified RSVP system. Users create personalized invitations with animated themes, fun "No button" mechanics, and optional mini-game challenges that guests must complete before confirming attendance.

- **Framework:** Angular 21 with Signals + `ChangeDetectionStrategy.OnPush`
- **Backend:** Supabase (PostgreSQL + Auth + Storage)
- **Language:** TypeScript 5.9 (strict mode)
- **Styles:** SCSS
- **Testing:** Vitest 4 + jsdom
- **Deployment:** Vercel — https://convitei.vercel.app
- **Locale:** Portuguese (pt-BR)

---

## Development Commands

```bash
npm install          # Install dependencies
npm start            # Dev server at http://localhost:4200
npm run build        # Production build → dist/
npm test             # Run tests (Vitest)
npm run watch        # Build with file watcher
```

> Angular CLI is also available directly: `ng serve`, `ng build`, `ng test`

---

## Directory Structure

```
src/
├── app/
│   ├── components/             # Feature components (one folder per feature)
│   │   ├── auth/               # Login + Register pages
│   │   ├── card-editor/        # Invite creation wizard
│   │   ├── card-preview/       # Public invite display (guest-facing)
│   │   ├── color-scheme/       # Color scheme picker
│   │   ├── confirm-dialog/     # Reusable confirmation dialog
│   │   ├── event-detail/       # Event detail view + notes management
│   │   ├── event-finance/      # Budget and expense tracking
│   │   ├── event-form/         # Event creation/edit form
│   │   ├── game-challenge/     # Mini-games: snake + space-shooter
│   │   ├── guests-manager/     # Guest list + WhatsApp sending
│   │   ├── invite-manager/     # Edit existing invite + live preview
│   │   ├── no-button-mechanics/# Configure the "No button" behavior
│   │   ├── rsvp-dashboard/     # Main authenticated dashboard
│   │   ├── theme-selector/     # Visual theme picker
│   │   └── welcome/            # Landing/welcome page
│   ├── guards/
│   │   └── auth.guard.ts       # Protects authenticated routes; awaits authReady
│   ├── models/
│   │   ├── card.model.ts       # Card, RSVPEntry, RSVPStats, InviteToken interfaces
│   │   ├── event.model.ts      # AppEvent, EventExpense, EventCategory, EventType
│   │   ├── guest.model.ts      # Guest, GuestStats interfaces
│   │   └── constants.ts        # Static data: themes, color schemes, mechanics, games
│   ├── services/
│   │   ├── auth.ts             # Authentication (login, register, logout, session)
│   │   ├── card.ts             # Card CRUD, file uploads, share links
│   │   ├── event.service.ts    # Event CRUD
│   │   ├── event-category.service.ts  # Expense categories
│   │   ├── event-finance.service.ts   # Budget tracking with computed signals
│   │   ├── guest.service.ts    # Guest CRUD, token generation, WhatsApp links
│   │   ├── invite-token.ts     # 32-char token lifecycle (create, validate, mark used)
│   │   ├── rsvp.ts             # RSVP entries and statistics
│   │   ├── supabase.ts         # Supabase client initialization and auth wrapper
│   │   └── theme.ts            # UI theme/color selection state
│   ├── app.ts                  # Root component
│   ├── app.config.ts           # Angular app config (providers, locale)
│   ├── app.routes.ts           # Route definitions
│   ├── app.scss                # Global component styles
│   └── app.spec.ts             # Root-level tests
├── environments/
│   └── environment.ts          # Supabase URL + anon key (safe to commit — public key)
├── main.ts                     # Bootstrap entry point
├── styles.scss                 # Global SCSS
└── index.html                  # HTML shell
```

---

## Routing

| Route | Auth Required | Component | Purpose |
|-------|:---:|-----------|---------|
| `/` | No | Redirects to `/login` | — |
| `/login` | No | LoginComponent | Email/password sign in |
| `/register` | No | RegisterComponent | Account creation |
| `/editor` | Yes | CardEditor | Create a new invitation |
| `/preview` | No | CardPreview | Development preview |
| `/manage/:id` | Yes | InviteManager | Edit existing invitation |
| `/invite/:id` | No | CardPreview | Public invite link |
| `/invite/:id/:token` | No | CardPreview | Guest-specific tracked link |
| `/events/new` | Yes | EventFormComponent | Create event |
| `/events/:eventId/edit` | Yes | EventFormComponent | Edit event |
| `/events/:eventId` | Yes | EventDetailComponent | Event detail + notes |
| `/events/:eventId/editor` | Yes | CardEditor | Create invite for event |
| `/dashboard` | Yes | RsvpDashboard | Main user dashboard |

The `authGuard` awaits `authService.authReady` (a Promise) before activating protected routes, ensuring session hydration completes before redirecting unauthenticated users.

---

## Architecture Patterns

### State Management
- **Angular Signals** are the primary reactive primitive. Services expose `signal()`, `computed()`, and `effect()` for state management.
- **RxJS BehaviorSubject** is used for stream-based data sharing (e.g., `cards$`, `currentCard$` in `CardService`).
- Prefer signals for new state; use BehaviorSubject only where observable streams are needed by consumers.

### Services
- All services use `providedIn: 'root'` for singleton injection and tree-shaking.
- Services own their Supabase queries directly — there is no abstracted repository layer.
- File naming: `auth.ts`, `card.ts` (no `-service` suffix for auth/card/rsvp/theme); `guest.service.ts`, `event.service.ts`, `event-finance.service.ts`, `event-category.service.ts` (with `-service` suffix).

### Components
- Use `ChangeDetectionStrategy.OnPush` throughout.
- Each component has its own folder containing `.ts`, `.html`, and `.scss` files.
- Components consume signals from services via injection — no input/output chaining for service state.

### Models
- TypeScript interfaces live in `src/app/models/`.
- Static configuration data (theme list, color schemes, game IDs, mechanics) lives in `constants.ts`.
- Database field names use `snake_case`; TypeScript properties use `camelCase`.

---

## Database Schema (Supabase)

### `cards`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK → auth.users | |
| event_id | uuid FK → events | nullable |
| sender_name | text | |
| title | text | |
| message | text | |
| theme | text | matches `CardTheme.id` from constants |
| color_scheme | text | matches `ColorScheme.id` from constants |
| no_button_mechanic | text | one of 4 mechanic IDs |
| challenge_mode_enabled | boolean | |
| challenge_games | text[] | array of `ChallengeGameId` values |
| floating_emoji | text | |
| photo_url | text | nullable, path in `card_media` bucket |
| music_url | text | nullable, path in `card_media` bucket |
| share_link | text | |
| yes_count | int | denormalized counter |
| no_count | int | denormalized counter |
| created_at | timestamptz | |

### `guests`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| card_id | uuid FK → cards | |
| user_id | uuid FK → auth.users | |
| name | text | |
| phone | text | |
| token | text unique | 24-char random, used in invite URL |
| status | text | `pending` \| `sent` \| `viewed` \| `confirmed` \| `declined` |
| response | text | `yes` \| `no`, nullable |
| adults | int | |
| children | int | |
| notes | text | nullable |
| confirmed_at / viewed_at / sent_at | timestamptz | nullable |
| created_at | timestamptz | |

### `rsvp_entries`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| card_id | uuid FK → cards | |
| response | text | `yes` \| `no` |
| guest_name | text | |
| guest_email | text | nullable |
| timestamp | timestamptz | |

### `invite_tokens`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| card_id | uuid FK → cards | |
| token | text unique | 32-char random |
| expires_at | timestamptz | 7 days from creation |
| used_at | timestamptz | nullable — null means unused |
| used_by | text | email or `'anonymous'` |
| created_at | timestamptz | |

### `events`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid FK → auth.users | |
| name | text | |
| event_type | text | `EventType` enum values |
| event_date | date | `YYYY-MM-DD` |
| event_time | text | `HH:mm` |
| event_location | text | |
| additional_notes | text | nullable |
| budget_total | numeric | |
| created_at | timestamptz | |

### `event_expenses`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| event_id | uuid FK → events | |
| description | text | |
| amount | numeric | |
| expense_type | text | `expense` \| `receipt` \| `supplier` |
| paid | boolean | |
| supplier_name | text | nullable |
| supplier_contact | text | nullable |
| created_at | timestamptz | |

### `event_categories`
| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| event_id | uuid FK → events | |
| name | text | |
| notes | text | nullable |
| display_order | int | |
| created_at | timestamptz | |

### Storage
Bucket: `card_media`
- Photos: `/photos/{user_id}/{timestamp}_{filename}`
- Music: `/music/{user_id}/{timestamp}_{filename}`

---

## Key Domain Concepts

### Invitation Themes (5)
Defined in `constants.ts` as `CARD_THEMES`:
- `elegant-minimal` — Poppins font
- `romantic-glow` — Lora font
- `vibrant-party` — Fredoka One font
- `luxury-gold` — Playfair Display font
- `ocean-serene` — Quicksand font

### Color Schemes (5)
Defined in `constants.ts` as `COLOR_SCHEMES`: classic-blue, purple-love, coral-crush, emerald, true-red.

### No-Button Mechanics (4)
Defined in `constants.ts` as `NO_BUTTON_MECHANICS`:
- `teleporting` — "No" button vanishes on hover
- `growing-yes` — "Yes" grows larger on each click
- `multiplying-yes` — Extra "Yes" buttons multiply
- `shrinking-no` — "No" button shrinks progressively

### Challenge Games (2)
Defined in `constants.ts` as `CHALLENGE_GAMES` and typed as `ChallengeGameId`:
- `snake` — Classic snake; user's upload photo is the character
- `space-shooter` — Shoot enemies with photo-themed ship

### Guest Token Flow
1. Guest added → unique 24-char token stored in `guests.token`
2. Host shares `/invite/:cardId/:token` via WhatsApp
3. App calls `GuestService.markAsViewed()` when link is opened
4. Guest confirms → `GuestService.recordResponse()` updates status atomically
5. Prevents double-confirmation via SQL condition in update query

### RSVP Statistics
Stats are aggregated from **both** `rsvp_entries` table and `guests` table (confirmed/declined guests). `RsvpService.getStats()` merges both sources.

---

## Code Conventions

### TypeScript
- Strict mode is enabled — no implicit `any`, explicit return types, no fallthrough switch cases.
- Models use PascalCase interfaces; database fields are mapped from `snake_case`.
- Service methods return `Promise<T>` for async Supabase calls.
- Use `signal<T>()` for mutable state in services; expose as readonly where possible.
- `computed()` for derived values (e.g., `totalPaid`, `totalPending` in `EventFinanceService`).
- `effect()` for side effects that react to signal changes (e.g., reload cards when auth state changes).

### SCSS
- Each component has its own `.scss` file scoped to that component.
- Global styles in `src/styles.scss`.
- Theme and color variables applied dynamically via Angular bindings, not CSS custom properties.

### Naming
- Component files: `component-name.ts` / `.html` / `.scss` (no `.component.` infix — Angular 21 style)
- Service files: mixed — some omit `-service` suffix (`auth.ts`, `card.ts`), some include it (`guest.service.ts`)
- Follow existing naming patterns in the directory when adding new services or components

### Error Handling
- Auth errors are translated to user-friendly Portuguese messages in `AuthService`.
- Supabase errors are checked with `if (error)` and thrown/logged; no global error boundary.
- Use `console.error()` for unexpected errors (no dedicated logging service).

---

## Testing

- **Framework:** Vitest 4 + jsdom 27
- **Config:** `tsconfig.spec.json`
- **Run:** `npm test`
- Test files use `.spec.ts` suffix alongside their source files.
- Current coverage is minimal — `app.spec.ts` exists as a base. When adding features, add matching `.spec.ts` files.

---

## Environment Configuration

`src/environments/environment.ts` contains the Supabase connection details:

```typescript
export const environment = {
  production: false,
  appUrl: 'https://convitei.vercel.app',
  supabase: {
    url: 'https://dkkximuqnzmsibnnkioe.supabase.co',
    key: '<anon-key>'   // Safe to commit — this is the public anon key
  }
};
```

The anon key has RLS (Row Level Security) enforced at the Supabase level — it is intentionally public.

---

## Build & Deployment

- **Build:** `ng build` — outputs to `dist/`
- **Deployment:** Vercel auto-deploys from the main branch
- **Build budgets:** 700 kB initial warning / 1 MB error; 9 kB per-component style warning
- **Output hashing:** enabled in production for cache busting
- **Angular builder:** `@angular/build:application` (esbuild-based, fast)

---

## Important Notes for AI Assistants

1. **Language:** All UI text, comments, and user-facing strings are in **Portuguese (pt-BR)**. Keep this consistent when adding new UI.

2. **Signals over RxJS:** Prefer Angular Signals for new state. Use `signal()`, `computed()`, and `effect()` — not `Subject` or `BehaviorSubject` — unless integrating with existing observable-based code.

3. **Supabase queries:** Always check for `error` after Supabase calls. Pattern:
   ```typescript
   const { data, error } = await this.supabase.client.from('table').select();
   if (error) throw error;
   ```

4. **Guest token uniqueness:** Guest tokens must be unique per card. The 24-char random token is generated client-side; ensure uniqueness with a retry loop or rely on DB unique constraint.

5. **Invite token vs guest token:** These are two separate systems:
   - `invite_tokens` — 32-char, 7-day expiry, one-time use, for general sharing
   - `guests.token` — 24-char, permanent, per-guest, for tracked individual links

6. **No-button mechanics are visual only** — the actual RSVP response is always recorded the same way regardless of which mechanic is active.

7. **Route params:** Card ID from `/invite/:id` is a Supabase UUID. Guest token from `/invite/:id/:token` is the 24-char string in `guests.token`.

8. **Change detection:** All components use `OnPush`. Signals trigger change detection automatically; if using imperative DOM updates, call `ChangeDetectorRef.markForCheck()`.

9. **File uploads:** Use `CardService.uploadFile()` — it handles Supabase Storage path conventions and returns the public URL.

10. **Event types:** `EventType` is a string enum in `event.model.ts`: `birthday`, `baby-shower`, `revelation`, `christening`, `graduation`, `wedding`, `other`.
