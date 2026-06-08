# CMS Platform

A full-stack, multi-tenant content management system built as a suite of microservices with a React Native mobile client. Authors can create and publish structured content entries, upload media, organise their workspace with categories and tags, and expose published content to any frontend through a public API.

The project is split into five repositories, each included here as a Git submodule.

---

## Repositories

| Submodule | Language / Framework | Role |
|---|---|---|
| [`auth`](./auth) | C# / ASP.NET Core 8 | User accounts, login, JWT, refresh tokens, 2FA, notifications |
| [`organisations`](./organisations) | C# / ASP.NET Core 8 | Tenants, membership, role enforcement, public API key lifecycle |
| [`content`](./content) | C# / ASP.NET Core 8 | Entry authoring, Elasticsearch search, Redis cache, public read API |
| [`assets`](./assets) | TypeScript / NestJS 11 | File uploads to MinIO (S3), asset metadata, Kafka event publishing |
| [`generic-mobile`](./generic-mobile) | TypeScript / Expo (React Native) | Mobile client for iOS and Android |

Each submodule has its own in-depth README covering its database schema, API reference, configuration, and local setup.

---

## Cloning

```bash
git clone --recurse-submodules https://github.com/genc-v/generic-cms-project
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

To pull the latest commits across all submodules:

```bash
git submodule update --remote --merge
```

---

## Architecture overview

```
┌─────────────────────────────────────────────────┐
│                 Mobile App (Expo)                │
│  Auth screens · Org dashboard · Entry editor    │
│  Media library · Settings · Notifications       │
└────┬──────────┬──────────┬──────────┬───────────┘
     │ JWT      │ JWT      │ JWT      │ API Key
     ▼          ▼          ▼          ▼
┌─────────┐ ┌───────┐ ┌────────┐ ┌──────────────┐
│  Auth   │ │  Org  │ │ Assets │ │   Content    │
│ Service │ │Service│ │Service │ │   Service    │
└────┬────┘ └───┬───┘ └───┬────┘ └──────┬───────┘
     │ issues   │ role    │ Kafka        │ public
     │ JWT      │ checks  │ files.upload │ read API
     └──────────┘  ↑──────┘─────────────┘
                   │ cmsOrg is called by Assets
                   │ and Content on every request
```

**How the services relate:**

- **Auth** is the only service that creates users and issues JWTs. Every other service trusts the same JWT secret and validates tokens locally — no service calls Auth to verify a token.
- **Organisations** is the central authority for tenant membership and API key validity. Both Content and Assets call it inline during request processing to check a user's role or validate a public key.
- **Assets** stores uploaded files in MinIO and publishes a `files.uploaded` Kafka event after every upload. It calls Organisations to enforce upload permissions.
- **Content** consumes the Kafka events from Assets and writes the file URL back to the entry automatically. It also calls Organisations to enforce role-based access on every private route and to validate public API keys.
- **Mobile** is the client that ties all four services together. It talks to each service directly over HTTP using the user's JWT.

---

## Technology stack

| Concern | Technology |
|---|---|
| Mobile client | Expo 56 (React Native 0.85), Expo Router, TypeScript |
| Auth backend | ASP.NET Core 8, EF Core, MySQL, SignalR |
| Org backend | ASP.NET Core 8, EF Core, MySQL, MongoDB (audit logs), Redis |
| Content backend | ASP.NET Core 8, EF Core, MySQL, Elasticsearch, Redis, Kafka |
| Assets backend | NestJS 11, MongoDB (Mongoose), MinIO (S3), Redis, Kafka |
| Message broker | Apache Kafka (Confluent) + Zookeeper |
| Object storage | MinIO (S3-compatible) |
| Primary databases | MySQL 8 (Auth, Org, Content) |
| Document store | MongoDB 7 (Org audit logs, Assets metadata) |
| Cache | Redis 7 (Org, Content, Assets) |
| Search | Elasticsearch 8 (Content, optional) |
| Containerisation | Docker + Docker Compose |

---

## End-to-end user journey

Here is what happens when a new user goes from zero to a published content entry with an uploaded image:

**1. Register and log in**
The user registers through the mobile app (`Auth` service). On login, Auth returns a JWT and a refresh token. The mobile app stores both securely and attaches the JWT to every subsequent request.

**2. Create an organisation**
The user creates an organisation (`Organisations` service). They are automatically assigned the Admin role for it. Other users can be invited and assigned Viewer, Editor, or Admin roles.

**3. Create an entry**
The mobile app calls `GET /{orgId}/entry/new-id` on the `Content` service to reserve a content ID. A stub entry with status `New` is created in the database. The Content service middleware verifies the JWT and checks with Organisations that the user has at least Viewer role in that org.

**4. Upload a file**
The mobile app uploads a file to the `Assets` service (`POST /organisations/:orgId/assets`), passing the `entryId` from step 3. Assets checks with Organisations that the user has Editor or Admin role, stores the file in MinIO under the key `orgs/{orgId}/{entryId}/{timestamp}-{filename}`, saves the metadata to MongoDB, and publishes a `files.uploaded` Kafka event.

**5. Entry auto-updates**
The `Content` service consumes the Kafka event and writes the file URL to the entry's `AssetUrl` field. It then re-evaluates whether the entry is now complete (title + rich content + category + asset URL all present). If it is, the status advances to `Published` automatically.

**6. Fill in the entry**
The editor types a title and rich content in the mobile app. The app saves via `PUT /{orgId}/entry/{entryId}` on the Content service. The Content service auto-generates a URL slug from the title (e.g. `/my-article`) on first save, assigns the category and tags, and updates the status.

**7. Public delivery**
Any external frontend — a website, a landing page, a static-site generator — can fetch published entries via the public API (`GET /api/public/content`) using an API key issued from the Organisations service. The API key is validated against Organisations, results are scoped to that org, and responses are cached in Redis.

---

## Service summaries

### Auth (`../auth`)

Handles the full user identity lifecycle. Provides registration, login, logout, JWT + refresh token issuance, 2FA via Google Authenticator, user profile management, in-app notifications over SignalR, and an admin interface for user and role management. Uses MySQL as its primary store and broadcasts real-time events to connected clients via a SignalR hub.

See [`auth/README.md`](./auth/README.md) for full documentation.

---

### Organisations (`../organisations`)

The central authority for the multi-tenant model. Owns organisation creation, membership (one role per user per org), the role hierarchy (Admin > Editor > Viewer), and the public API key lifecycle. Every other service that needs to answer "can this user do this in this org?" calls `GET /organisations/{orgId}/role` on this service. Public API key validation (`GET /api-keys/validate`) is also handled here. Writes an audit log to MongoDB for every action (fire-and-forget, never blocks responses). Caches org lookups in Redis.

See [`organisations/README.md`](./organisations/README.md) for full documentation.

---

### Content (`../content`)

Manages the lifecycle of content entries from first draft through publication. Every entry has a title, a URL slug (auto-generated from the title), rich HTML content, an optional category, tags, and an asset URL. Status is computed automatically — an entry becomes `Published` when all required fields are complete, and returns to `Draft` if any are cleared. Supports full-text search via Elasticsearch with a MySQL fallback. Caches public reads in Redis. Consumes the `files.uploaded` Kafka topic from Assets to attach file URLs to entries. Exposes a separate public API (API key authenticated) for external consumer frontends.

See [`content/README.md`](./content/README.md) for full documentation.

---

### Assets (`../assets`)

Handles binary file storage. Accepts multipart file uploads from the mobile client, stores files in MinIO with org-and-entry-scoped S3 keys, persists metadata as embedded documents in MongoDB, and publishes a `files.uploaded` Kafka event so the Content service can close the loop. Access to upload is gated behind an org role check against Organisations (Editor or Admin). Asset lists and info responses are cached in Redis using a generation-based invalidation strategy. Also exposes an anonymous `GET /uploads/*` streaming endpoint so uploaded files can be used directly in `<img>` and `<video>` tags.

See [`assets/README.md`](./assets/README.md) for full documentation.

---

### Mobile client (`../generic-mobile`)

A React Native app built with Expo and Expo Router. Targets iOS and Android. Implements the full CMS workflow:

- **Auth screens** — login, register, 2FA setup and verification
- **Organisation list** — create and switch between organisations
- **Org dashboard** — tabbed view of entries (with status filters and full-text search), categories, tags, and media library
- **Entry editor** — title, rich text body with formatting toolbar, category and tag pickers, asset attachment from the media library, live status indicator, markdown preview, save / unpublish / delete
- **Media library** — paginated grid of uploaded assets per org, asset detail sheet with metadata and URL copy, delete
- **Org settings** — rename org, manage members and their roles, manage public API keys (create / toggle / delete), export and import entries/categories/tags in JSON, CSV, or Excel
- **Profile** — account details, password change, 2FA management, notification history
- **Admin panel** — manage all platform users and their system roles (Admin only)

The app follows an MVVM pattern: each screen delegates all state and API logic to a dedicated viewmodel hook, keeping the UI layer thin. Auth state (JWT + refresh token) is persisted in `expo-secure-store`.

---

## Running the full stack locally

Each service has its own `docker compose` or `docker-compose.yml`. The recommended startup order is:

```bash
# 1. Start Organisations (needed by Content and Assets for role checks)
cd organisations && docker compose up -d

# 2. Start Auth
cd ../auth && docker compose up -d

# 3. Start Assets (includes Kafka — Content depends on the same Kafka network)
cd ../assets && docker compose up -d

# 4. Start Content (joins the Kafka network created by Assets)
cd ../content && docker compose up -d

# 5. Run the mobile app
cd ../generic-mobile
npm install
npx expo start
```

Point the mobile app's API base URLs at the services running on your machine. Each service's README has its port mappings and configuration reference.

| Service | Default port |
|---|---|
| Auth API | `5050` |
| Organisations API | `5059` |
| Content API | `5054` |
| Assets API | `3000` |
| MinIO API | `9000` |
| MinIO Console | `9001` |
| Kafka | `29092` (external) |
| Elasticsearch | `9200` (optional) |
