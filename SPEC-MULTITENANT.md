# MOLAB Screens — Multi-tenant SaaS Specification

## 1. Overview

Transform MOLAB Screens from a single-tenant demo into a multi-tenant SaaS platform where:
- **Admin Molab** manages all clients, creates accounts, can impersonate any client
- **Clients** log in with email+password, manage their own brands, screens, media, playlists

## 2. Data Model

### 2.1 Database: SQLite (better-sqlite3)

```sql
-- Clients (organizations/companies)
CREATE TABLE clients (
  id TEXT PRIMARY KEY,                    -- slug, e.g. "alexis"
  name TEXT NOT NULL,                     -- "Alexis Beauty"
  plan TEXT NOT NULL DEFAULT 'standard',  -- samodzielne|standard|pro|premium
  logo_url TEXT,                          -- white-label logo
  settings TEXT DEFAULT '{}',             -- JSON: custom config
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Users (login accounts)
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'client',     -- admin|client
  client_id TEXT REFERENCES clients(id),   -- NULL for admin
  name TEXT,
  must_change_password INTEGER DEFAULT 1,  -- force change on first login
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Screens (display endpoints)
CREATE TABLE screens (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  client_id TEXT NOT NULL REFERENCES clients(id),
  name TEXT NOT NULL,                       -- "Ekran recepcja"
  display_token TEXT UNIQUE NOT NULL,       -- 10-char alphanumeric, used in URL
  orientation TEXT NOT NULL DEFAULT 'landscape', -- landscape|portrait
  config TEXT DEFAULT '{}',                 -- JSON: per-screen settings
  is_active INTEGER DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Screen heartbeats (online/offline tracking)
CREATE TABLE screen_heartbeats (
  screen_id INTEGER PRIMARY KEY REFERENCES screens(id),
  last_seen TEXT NOT NULL,
  ip TEXT,
  user_agent TEXT
);
```

### 2.2 File Storage (per-client isolation)

```
data/
├── clients/
│   ├── alexis/
│   │   ├── brands/
│   │   │   └── alexis-beauty/
│   │   │       ├── brand.json
│   │   │       ├── references/
│   │   │       ├── logos/
│   │   │       ├── outputs/
│   │   │       └── templates/
│   │   ├── media/              ← all client media (images, videos)
│   │   └── screens/
│   │       ├── <screen_id>/
│   │       │   ├── playlist.json
│   │       │   └── current.json
│   └── other-client/
│       └── ...
├── db.sqlite                   ← main database
└── tmp/                        ← shared temp dir for uploads
```

## 3. Authentication & Authorization

### 3.1 Auth Flow
- **Login:** POST `/api/auth/login` → email + password → JWT token (24h expiry)
- **JWT payload:** `{ userId, email, role, clientId }`
- **Password hashing:** bcrypt (12 rounds)
- **First login:** `must_change_password` flag → force password change

### 3.2 Middleware Stack
```
authMiddleware       → verify JWT, attach user to req
clientIsolation      → if role=client, scope all queries to user.clientId
adminOnly            → reject if role != admin
planGate(feature)    → check client plan allows feature
```

### 3.3 Admin Impersonation
- Admin clicks "Enter as client" → GET `/api/admin/impersonate/:clientId`
- Returns new JWT with `{ ...adminUser, impersonating: clientId }`
- UI switches to client view with yellow banner "Viewing as: Alexis Beauty [Exit]"
- All data queries scoped to impersonated clientId
- Admin can exit back to admin view at any time

## 4. Plans & Feature Gates

| Feature | Samodzielne | Standard | Pro | Premium |
|---------|------------|----------|-----|---------|
| Manual upload | ✅ | ✅ | ✅ | ✅ |
| Brands | ✅ | ✅ | ✅ | ✅ |
| Playlists + scheduling | ✅ | ✅ | ✅ | ✅ |
| Multiple screens | ✅ | ✅ | ✅ | ✅ |
| AI image generation | ❌ | ✅ | ✅ | ✅ |
| Telegram bot | ❌ | ✅ | ✅ | ✅ |
| AI video/animation | ❌ | ❌ | ✅ | ✅ |
| Content calendar (Molab team) | ❌ | ❌ | ✅ | ✅ |
| Dedicated account manager | ❌ | ❌ | ❌ | ✅ |

Feature gates are **server-side enforced** (API returns 403) + **client-side hidden** (UI doesn't show tabs).

## 5. Display (Player) Endpoint

### URL: `/display/:token`

- Token: 10-character alphanumeric (a-z, A-Z, 0-9), generated with crypto.randomBytes
  - Example: `k7Mx9pRw2Q`
  - Readable enough to type, ~60 bits of entropy (impossible to guess)
- No auth required (public URL for TV/display)
- Player loads playlist for that screen, respects:
  - Orientation (landscape/portrait)
  - Schedule (date range, time range, days of week)
  - Transitions between items
- **Heartbeat:** Player POSTs `/api/display/:token/heartbeat` every 30s
  - Server records last_seen, IP, user_agent
  - Panel shows: 🟢 online (seen <2min), 🟡 stale (2-10min), 🔴 offline (>10min)

### Security
- Token not guessable (cryptographic random)
- No sensitive data exposed (only media URLs for that screen)
- Rate-limited heartbeat endpoint
- No cross-client data leakage possible (player only sees its own playlist)

## 6. API Routes

### Auth
```
POST   /api/auth/login              → { email, password } → { token, user }
POST   /api/auth/change-password    → { oldPassword, newPassword }
GET    /api/auth/me                 → current user info
```

### Admin (role=admin only)
```
GET    /api/admin/clients           → list all clients
POST   /api/admin/clients           → create client
PATCH  /api/admin/clients/:id       → update client (name, plan, etc.)
DELETE /api/admin/clients/:id       → delete client + all data

GET    /api/admin/clients/:id/users → list users for client
POST   /api/admin/users             → create user (assign to client)
DELETE /api/admin/users/:id         → delete user

GET    /api/admin/impersonate/:clientId → get impersonation token
GET    /api/admin/stats             → dashboard stats (total clients, screens, media usage)
```

### Client-scoped (auto-filtered by clientId)
```
-- Brands (same as current, but scoped to client)
GET    /api/brands
POST   /api/brands
PATCH  /api/brands/:id
DELETE /api/brands/:id
...etc (references, logos, templates — same endpoints)

-- Screens
GET    /api/screens                 → list my screens
POST   /api/screens                 → create screen (generates display_token)
PATCH  /api/screens/:id             → update screen (name, orientation)
DELETE /api/screens/:id             → delete screen

-- Screen playlist
GET    /api/screens/:id/playlist
POST   /api/screens/:id/playlist    → save playlist (with schedule per item)
POST   /api/screens/:id/playlist/clear

-- Screen current
GET    /api/screens/:id/current
POST   /api/screens/:id/current     → set immediate display

-- Media (client-isolated)
GET    /api/media                   → list my media
POST   /api/upload                  → upload to my media folder
DELETE /api/media/:filename         → delete my media

-- AI Generation (plan-gated)
POST   /api/generate                → generate AI image (standard+)
POST   /api/generate-variants       → generate 4 variants (standard+)
POST   /api/animate                 → animate image to video (pro+)

-- Config
GET    /api/config                  → get my settings
POST   /api/config                  → update settings
```

### Display (public, no auth)
```
GET    /display/:token              → player HTML
GET    /api/display/:token/playlist → playlist for this screen
GET    /api/display/:token/config   → screen config (orientation etc.)
POST   /api/display/:token/heartbeat → { } → screen reports alive
```

## 7. UI Changes

### 7.1 Login Page
- Clean login form: email + password
- "Molab Screens" branding
- Error messages for wrong credentials
- First-login: force password change screen

### 7.2 Admin Panel (role=admin)
- **Dashboard:** total clients, screens, online screens, media usage
- **Clients list:** table with name, plan, screens count, status
  - Actions: Edit, Impersonate ("Wejdź jako klient"), Delete
- **Create client:** form (name, plan, initial user email+password)
- **Client detail:** edit plan, see screens, see users
- When impersonating: yellow top banner "Przeglądasz jako: [Client Name] — [Wyjdź]"

### 7.3 Client Panel (role=client or impersonating)
- Same tabs as now: AI Generator, Upload, Media, Playlista, Brandy, Ustawienia, Podgląd
- **NEW tab: Ekrany** — manage screens (add, remove, see online/offline status, copy display URL)
- Tabs hidden based on plan (e.g., no "AI Generator" for Samodzielne)
- White-label: client logo in header (if set), otherwise Molab logo
- Screen selector: dropdown to switch between screens (playlista/podgląd are per-screen)

## 8. Migration Plan

### Existing data → Client "Alexis"
1. Create client `alexis` with plan `standard`
2. Create user `mariusz.grela@gmail.com` assigned to `alexis`
3. Move `data/brands/alexis/` → `data/clients/alexis/brands/alexis/`  
   (keep other brands if they exist as separate clients or archive)
4. Move `data/media/` → `data/clients/alexis/media/`
5. Create default screen for Alexis with existing playlist
6. Generate display token for that screen

### Admin account
- Create admin user (e.g. `admin@molab.pl` or `mariusz.grela@molab.pl`)

## 9. Telegram Bot (Standard+ plans)

### Current architecture (molab-screens-bot)
- Single bot, multi-tenant via SQLite
- Already maps chat_id → brand_id

### Target architecture
- Keep single-bot as default (Molab-managed bot)
- **Optional:** Client enters their own BOT_TOKEN in settings
  - System spawns separate bot instance for that token
  - Bot process management: PM2 or simple child_process pool
  - Fall back to shared Molab bot if client bot is not configured

### Phase 2 (not now)
- Will integrate after core multi-tenant is stable

## 10. Tech Stack

- **Backend:** Node.js + Express (existing)
- **Database:** better-sqlite3 (upgrade from flat JSON files)
- **Auth:** bcrypt + jsonwebtoken (JWT)
- **Frontend:** Vanilla JS (existing) — refactor admin.html/admin.js
- **Deploy:** Coolify (existing infrastructure)
- **Development:** Claude Code + Opus (frontend-design skill for UI)

## 11. Implementation Phases

### Phase 1: Foundation (DB + Auth + Isolation) 
- [ ] Add better-sqlite3, bcrypt, jsonwebtoken
- [ ] Create DB schema + migration script
- [ ] Auth system (login, JWT middleware, password change)
- [ ] Client isolation middleware
- [ ] Admin CRUD (clients, users)
- [ ] Admin impersonation
- [ ] Migrate file storage to per-client structure
- [ ] Migration script for Alexis data

### Phase 2: Multi-screen + Display
- [ ] Screen CRUD (create, edit, delete)
- [ ] Display token generation
- [ ] Per-screen playlist + current
- [ ] Player endpoint `/display/:token`
- [ ] Heartbeat system (online/offline)
- [ ] UI: screen management tab, screen selector

### Phase 3: Plans + Feature Gates
- [ ] Plan-based feature gating (server + client)
- [ ] UI tab visibility per plan
- [ ] Admin: plan management per client
- [ ] Usage tracking (generations count per client)

### Phase 4: Bot Integration (later)
- [ ] Per-client Telegram bot config
- [ ] Bot instance management
- [ ] Connect bot to client's screens

## 12. Security Considerations

- **Passwords:** bcrypt, min 8 chars
- **JWT:** 24h expiry, httpOnly cookie or Authorization header
- **Display tokens:** crypto.randomBytes(8).toString('base62') — ~48 bits entropy
- **File access:** all media served through auth-checked API (except display endpoint)
- **Rate limiting:** already in place, extend to auth endpoints
- **CORS:** restrict to same origin
- **Admin impersonation:** logged in audit trail
