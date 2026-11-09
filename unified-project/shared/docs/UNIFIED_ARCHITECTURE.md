# Unified Application Architecture & Integration Roadmap

**Source:** `bdsaymon/Call-apps` — branch `arena/bff36304-call-apps`  
**Date:** 2026-11-08  
**Status:** Technical Audit — awaiting approval before any integration begins  
**Extracted artifacts preserved:** `chat-master.zip`, `tindroid-master.zip`, `webapp-master.zip` and their extracted folders remain untouched.

---

## 1. Executive Summary

Three separate Tinode project archives were extracted into the repository:

| Archive | Extracted folder | Files | Role |
|---------|-----------------|-------|------|
| `chat-master.zip` | `chat-master/` | 264 | Backend server + SDKs + tooling |
| `tindroid-master.zip` | `tindroid-master/` | 575 | Android native client (Java) |
| `webapp-master.zip` | `webapp-master/` | 439 | Web single-page client (React) |

**No source code was modified, deleted, or merged.** This document is the *technical audit* and *unified integration plan*. All original ZIPs and extracted folders are preserved. Integration will begin only after your approval.

---

## 2. Technical Audit

### 2.1 Project 1 — `chat-master` (Backend Server)

**Identity:** Tinode Instant Messaging Server — full-stack backend.

| Aspect | Detail |
|--------|--------|
| **Primary language** | Go (Go 1.26.0, `module github.com/tinode/chat`) |
| **License** | GPL 3.0 (server), Apache 2.0 (clients) |
| **Build system** | Go modules, `build-all.sh` for cross-platform binaries (mac/linux/windows, amd64/arm64), Docker |
| **Wire protocols** | JSON over WebSocket (primary), long-polling fallback, protobuf v3 via gRPC |
| **Database adapters** | MySQL/MariaDB, PostgreSQL, MongoDB, RethinkDB (deprecated) — pluggable via `server/db/adapter.go` |
| **Authentication schemes** | Anonymous, Basic, REST, Token, Code (email/SMS), Mock |
| **Push notification backends** | FCM, stdout (dev), TNPC |
| **Media storage** | Local filesystem, Amazon S3 (pluggable `media.Handler`) |
| **Clustering** | Sharded clustering with failover (`cluster.go`, `cluster_leader.go`, `ringhash.go`) |
| **Real-time calls** | WebRTC signaling (offer/answer/ICE, ring/accept/hang-up), ICE server config |
| **Feature set** | 1:1 & group chat, channels (read-only subscribers), presence, typing, read/delivery receipts, message edit/forward/reply, pinned messages, wallpapers, light/dark themes, chatbot forms (Drafty), access control (ACL), user discovery by email/tel, plugins |

**Key Go source files (80 `.go` files under `server/`):**
- `main.go` — server entry point
- `datamodel.go` — wire protocol message structures (C2S/S2C)
- `topic.go` — isolated communication channel (P2P / group / channel), call state (`videoCall`)
- `user.go` — account creation, credentials, tags, access control
- `session.go` — client session management (WebSocket / long-poll / proxy)
- `hub.go` — message broadcast hub
- `calls.go` — video call establishment & termination logic
- `push.go` — push notification receipt preparation
- `store/store.go` + `store/types/` — storage abstraction & data types
- `db/{mysql,postgres,mongodb,rethinkdb}/adapter.go` — DB adapters
- `auth/{anon,basic,code,rest,token,mock_auth}/` — auth handlers
- `push/{fcm,stdout,tnpg}/` — push providers
- `media/{fs,s3}/` — media handlers
- `drafty/drafty.go` — rich text (Drafty) formatter
- `plugins.go` — server plugin system

**Client SDKs bundled:** Python chatbot (`chatbot/python/chatbot.py`), C# chatbot, PBX (`pbx/` with protobuf), `tn-cli/` Python admin CLI, `py_grpc/` Python gRPC bindings.

**Config:** `server/tinode.conf`, `INSTALL.md`, `docker/README.md`, `docker-compose/` for multi-container deploy.

---

### 2.2 Project 2 — `tindroid-master` (Android Client)

**Identity:** Tinode Messaging Client for Android — native frontend (not standalone; requires a Tinode backend).

| Aspect | Detail |
|--------|--------|
| **Primary language** | Java (JDK 17), Android SDK |
| **Build system** | Gradle 9.6.1, Android Gradle Plugin 9.4.1 |
| **compileSdk / targetSdk** | 36 (Android 15+) |
| **minSdk** | 27 (Android 8.1 Oreo) |
| **App ID** | `co.tinode.tindroidx` |
| **Architecture** | Two Gradle modules: `:app` + `:tinodesdk` (client SDK, no Android SDK deps) |
| **Default hosts** | Debug: `sandbox.tinode.co`; Release: `api.tinode.co` |
| **Transport** | WebSocket (`java-websocket`), JSON; gRPC not used by client |
| **Push** | Firebase Cloud Messaging (FCM) + Crashlytics + Google ML Kit (QR scanning) |
| **Calling** | WebRTC (`com.dafruits:webrtc:123.0.0`) for audio/video 1:1 calls |
| **Media** | CameraX 1.6.2, Media3 ExoPlayer 1.11.1, Coil image loading |
| **Phone numbers** | libphonenumber 9.0.40 |
| **UI** | Material Components 1.14.0, Google Flexbox 3.0.0, AndroidX |
| **Local storage** | SQLite via `db/` package (AccountDb, MessageDb, TopicDb, UserDb, SubscriberDb) |
| **Rich text** | Drafty formatters/spans (`format/`) |

**Java source: 192 `.java` files**

**App layer (`co.tinode.tindroid`):** Activities/Fragments for login, signup, chats, messages, contacts, groups, calls, settings, wallpapers, QR scanner, attachments, credentials.

**SDK layer (`co.tinode.tinodesdk`):** `Tinode.java` (main client), `Connection.java` (WebSocket wrapper with reconnect backoff), `Topic.java`, `ComTopic.java`, `FndTopic.java`, `MeTopic.java`, `User.java`, `Storage.java`, `LocalData.java`, large-file helper, message model classes (`MsgClientHi`, `MsgClientLogin`, `MsgClientPub`, `MsgServerData`, `MsgServerPres`, `MsgServerInfo`, `Drafty`, `TheCard`, `Acs`, `Subscription`, etc.)

**Features (from README):** 1:1 & group chats, channels, 1:1 video/audio calls, attachments/inline media/voice messages, unread counters, push + presence, typing/read/delivery receipts, markdown-style formatting (spannable), edit/reply/forward, verified badges, chatbot form messages, mute/permissions, Android Contacts integration, SMS/email invite, TLS (wss), pin conversations & messages, offline mode.

**Not done (from README):** E2E encryption, video message recording, multi-backend switching, multi-account, mentions/hashtags, link/audio/doc previews.

---

### 2.3 Project 3 — `webapp-master` (Web Client)

**Identity:** Tinode Web — single-page web chat application (React), frontend only; requires Tinode backend.

| Aspect | Detail |
|--------|--------|
| **Primary language** | JavaScript / JSX (React 19.3.0, ReactDOM 19.3.0) |
| **Build system** | npm, Webpack 5, Babel, PostCSS, Jest |
| **Output** | UMD bundles (`umd/index.prod.js`, `umd/index.dev.js`, sourcemaps) |
| **Internationalization** | react-intl 12.1.3, 14 locales (ar, de, en, es, fr, id, it, ko, ro, ru, th, uk, vi, zh, zh-TW) |
| **Push** | Firebase 13.1.0 (FCM web messaging) |
| **Phone numbers** | libphonenumber-js 1.13.13 |
| **QR codes** | qrcodejs 1.0.0 |
| **Media helpers** | webm-duration-fix 1.0.4 |
| **SDK dependency** | `tinode-sdk` 0.25.4 (the Tinode JS SDK — no external dependencies) |
| **PWA** | `manifest.json`, `service-worker.js` |
| **Entry HTML** | `index.html`, `index-dev.html` |
| **Styling** | `css/base.css` + minified `css/base.min.css` (PostCSS) |

**Source: 128 files under `src/`**

**Views (`src/views/`):** `tinode-web.jsx` (root), `messages-view.jsx`, `contacts-view.jsx`, `login-view.jsx`, `create-account-view.jsx`, `account-settings-view.jsx`, `sidepanel-view.jsx`, `info-view.jsx`, `new-topic-view.jsx`, `password-reset-view.jsx`, `settings-view.jsx`, `acc-general-view.jsx`, `acc-notifications-view.jsx`, `acc-security-view.jsx`, `acc-support-view.jsx`, `validation-view.jsx`, `wallpapers.jsx`, `topic-common-view.jsx`, `logo-view.jsx`.

**Widgets (`src/widgets/`):** call-incoming, call-panel, call-status, call-message, chat-message, contact-list, contact, group-manager, group-subs, image-preview, video-preview, audio-player, audio-recorder, inline-video, attachment, doc-preview, forward-dialog, pinned-messages, search-contacts, send-message, phone-edit, phone-country-selector, show-qrcode, upload-image, lazy-image, letter-tile, avatar-crop, avatar-upload, contact-badges, badge-list, permissions-editor, topic-security, topic-desc-edit, the-card-preview, the-card-mini, meta-message, received-marker, unread-badge, load-spinner, error-panel, context-menu, side-navbar, menu-start, menu-contacts, menu-cancel, invitation, credential-edit, visible-password, file-progress, button-back, chip, chip-input, tag-manager, alert.

**Libs (`src/lib/`):** `config.js`, `constants.js`, `host-name.js`, `local-storage.js`, `navigation.js` (hash-based routing), `blob-helpers.js`, `formatters.js`, `strformat.js`, `utils.js`, `react-intl-compat.js`.

**Config (`config.js`):** `API_KEY`, `DEFAULT_HOST` (`web.tinode.co` / `localhost:6060`), `KNOWN_HOSTS`, call state constants, media limits (`MAX_INBAND_ATTACHMENT_SIZE`, `MAX_EXTERN_ATTACHMENT_SIZE`, `MAX_IMAGE_DIM`), access mode strings (`JRWPS`, `JRWPSAO`, `JR`), i18n settings, wallpaper defaults, Firebase links.

---

## 3. Cross-Project Analysis

### 3.1 Shared / Duplicate Features (Integration Candidates)

| Feature | chat-master | tindroid | webapp | Integration approach |
|---------|-------------|----------|--------|----------------------|
| 1:1 messaging | ✅ Topic P2P | ✅ ChatsActivity/MessageActivity | ✅ messages-view.jsx | Single backend serves both clients |
| Group chats | ✅ Topic GRP | ✅ CreateGroupFragment | ✅ new-topic-group.jsx | Same topic model |
| Read-only channels | ✅ Topic CHN | ✅ Channels | ✅ group-subs.jsx | Same topic model |
| Video/voice 1:1 calls | ✅ calls.go (WebRTC signaling) | ✅ CallActivity + WebRTC | ✅ call-panel.jsx / call-incoming.jsx | Unified signaling via backend |
| Push notifications | ✅ push.go (FCM) | ✅ FCM + Crashlytics | ✅ Firebase web messaging | Shared FCM project config |
| Presence/typing/read receipts | ✅ session.go + pres.go | ✅ SDK | ✅ React state + SDK | Unified via wire protocol |
| Message edit/reply/forward | ✅ datamodel.go | ✅ format/ (Drafty) | ✅ chat-message.jsx | Shared wire protocol |
| Drafty rich text | ✅ drafty.go | ✅ format/ (Java) | ✅ JS SDK | Unified wire format |
| Wallpapers / themes | ✅ server + docs | ✅ WallpaperFragment | ✅ wallpapers.jsx | Shared asset + config |
| Contacts integration | N/A (server) | ✅ Android Contacts | N/A | Android-only feature |
| Auth: email/tel/anon | ✅ auth handlers | ✅ Login/SignUp | ✅ login/create-account | Unified auth endpoints |
| i18n | ✅ docs/translations.md | ✅ 14 languages | ✅ react-intl (14 locales) | Shared translation keys |
| Offline mode / local DB | N/A | ✅ SQLite (db/) | ✅ localStorage + service worker | Client-specific, keep separate |

### 3.2 Conflicts / Gaps to Resolve

| Conflict / Gap | Detail | Resolution in unified design |
|----------------|--------|------------------------------|
| **Different SDK versions** | tindroid-sdk (Java) vs tinode-sdk (JS, v0.25.4) vs chat-master Go models | Use the Go server's wire protocol as the single source of truth; keep client SDKs thin adapters |
| **Default hosts differ** | Android: sandbox/api.tinode.co; Web: web.tinode.co/localhost:6060 | Unified `host-selector` widget + `config.js` with env-overridable defaults |
| **Push providers** | Android FCM + Crashlytics; Web Firebase messaging | Single Firebase project; Android `google-services.json` + web `firebase-init.js` |
| **Calling stack** | Android WebRTC (native) vs Web WebRTC (browser) | Backend signaling is identical; client platforms handle media separately |
| **Local storage** | Android SQLite vs Web localStorage | Keep client-local; no unification needed |
| **Multi-account** | Not supported on either client | Post-MVP roadmap item |
| **E2E encryption** | Not implemented anywhere | Planned feature, not in current scope |
| **File size limits** | Web `MAX_EXTERN_ATTACHMENT_SIZE` 8MB vs Android Media3 | Align via server `max_message_size` config |

### 3.3 Reusable Components (Integration Inventory)

| Component | Source | Reuse in unified app |
|-----------|--------|----------------------|
| Backend server | `chat-master/server/` | Core runtime — unchanged |
| DB adapters | `chat-master/server/db/` | Pluggable storage layer |
| Auth handlers | `chat-master/server/auth/` | Unified auth endpoints |
| Push infra | `chat-master/server/push/` | FCM for both Android & web |
| Call signaling | `chat-master/server/calls.go` | Unified WebRTC signaling |
| Android SDK | `tindroid-master/tinodesdk/` | Android client network layer |
| Android UI | `tindroid-master/app/src/main/java/...` | Android screens |
| Web UI | `webapp-master/src/views/`, `src/widgets/` | Web screens |
| Web UMD build | `webapp-master/umd/` | Production web bundle |
| i18n strings | All three | Consolidated translation catalog |
| Assets (audio/img) | `webapp-master/audio/`, `img/`, `tindroid-master/app/src/main/res/` | Shared brand assets |
| Docker | `chat-master/docker/` | Unified container deployment |
| `tn-cli` | `chat-master/tn-cli/` | Admin CLI for unified backend |

---

## 4. Unified Application Architecture

### 4.1 High-Level Architecture

```
┌─────────────────┐      WebSocket (JSON/gRPC)      ┌──────────────────┐
│  Android Client │◄────────────────────────────────►│  Tinode Server   │
│  (tindroid)     │                                  │  (chat-master)   │
│  Java + WebRTC  │◄────────────────────────────────►│  Go 1.26         │
└─────────────────┘      HTTPS/FCM push              │  DB: MySQL/PG    │
                                                    │  /Mongo/Redis    │
┌─────────────────┐                                  │  Media: FS/S3    │
│   Web Client    │◄────────────────────────────────►│  Push: FCM       │
│  (React webapp) │                                  └──────────────────┘
│  JavaScript     │
│  + WebRTC       │
└─────────────────┘
```

### 4.2 Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                      │
│  ┌──────────────────┐  ┌─────────────────────────────────┐  │
│  │  Android App     │  │  Web App (React SPA)            │  │
│  │  (Java, minSdk27)│  │  (React 19, react-intl, UMD)    │  │
│  └──────────────────┘  └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                  CLIENT SDK / TRANSPORT LAYER               │
│  ┌──────────────────┐  ┌─────────────────────────────────┐  │
│  │  tinodesdk (Java)│  │  tinode-sdk (JavaScript)        │  │
│  │  WebSocket +     │  │  WebSocket + JSON               │  │
│  │  reconnect/backoff│ │                                 │  │
│  └──────────────────┘  └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY LAYER                      │
│  WebSocket (JSON)  │  Long-polling  │  gRPC (protobuf)    │
│  ─────────────────►  Tinode Server (Go)                   │
└─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   BUSINESS LOGIC LAYER                      │
│  topic.go  user.go  session.go  hub.go  calls.go  push.go   │
│  (P2P / Group / Channel topics, call signaling, notifications)│
└─────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   DATA / INFRASTRUCTURE LAYER                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │  MySQL   │ │PostgreSQL│ │ MongoDB  │ │Media FS/S3   │   │
│  │  adapter │ │  adapter │ │ adapter  │ │  handler     │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
│  Auth: anon/basic/rest/token/code          Push: FCM/tnpg  │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 Unified Data Model

The Go server (`datamodel.go`, `store/types/`) defines the canonical wire protocol:

- **C2S messages:** `hi`, `login`, `sub`, `pub`, `set`, `get`, `del`, `note`, `acc`
- **S2C messages:** `ctrl`, `data`, `meta`, `info`, `pres`
- **Topics:** `me` (self), `fnd` (find/users), `sys` (system), `slf` (self metadata), `grp*` (group), `p2p` (peer-to-peer), `nch` (channel)
- **Access modes:** `J` (join), `R` (read), `W` (write), `P` (presence), `S` (subscribe), `A` (admin), `O` (owner)
- **Calls:** `webrtc` header with offer/answer/ice-candidate/ring/accept/hang-up events

Both Android and web clients already speak this protocol via their respective SDKs — no wire-protocol change is required for integration.

---

## 5. Integration Roadmap (Phased)

**Rule:** No source code is modified until you approve this document. Each phase below is a plan only.

### Phase 0 — Audit & Approval (current)
- [x] Extract all three ZIPs (preserved in `chat-master/`, `tindroid-master/`, `webapp-master/`)
- [x] Technical audit complete (this document)
- [ ] **Your approval** to proceed to Phase 1

### Phase 1 — Repository Consolidation (no code changes, structure only)
- [ ] Create unified repo layout:
  ```
  /backend        → chat-master/server/, chat-master/tinode-db/, chat-master/docker/
  /sdk/android    → tindroid-master/tinodesdk/
  /app/android    → tindroid-master/app/
  /app/web        → webapp-master/src/, webapp-master/umd/, webapp-master/package.json
  /shared         → shared assets, i18n, docs
  ```
- [ ] Add unified `docker-compose.yml` that starts backend + web static assets
- [ ] Add unified `README.md` with build instructions for all three parts
- [ ] Keep original ZIPs untouched as reference

### Phase 2 — Build & CI Unification
- [ ] Unified build script (`build-all.sh`) that builds:
  - Go backend binary (multi-arch)
  - Android APK (debug/release)
  - Web UMD bundle (npm run build)
- [ ] CI pipeline (GitHub Actions) to build all three on push
- [ ] Unified versioning (single version across backend + clients)

### Phase 3 — Configuration Unification
- [ ] Single `tinode.conf` with unified defaults for ICE servers, FCM, max attachment size
- [ ] Unified `config.js` (web) and `build.gradle` `default_host_name` (android) pointing to same backend URL
- [ ] Single Firebase project serving both Android (`google-services.json`) and Web (`firebase-init.js`)
- [ ] Consolidated i18n: merge translation keys from Android strings + web JSON into one catalog

### Phase 4 — Feature Unification (client parity)
- [ ] Align feature flags between Android and Web:
  - Calls: 1:1 video/audio (both already support)
  - Push: silent + visible notifications (both already support)
  - Drafty rich text (both already support)
  - Wallpapers/themes (both already support)
- [ ] Add missing parity items (from "Not done" lists):
  - E2E encryption (roadmap item, server-planned)
  - Mentions/hashtags
  - Link/document previews
  - Multi-account support (backend + both clients)

### Phase 5 — Testing & Release
- [ ] Unified test suite: Go unit tests + Android unit/instrumentation + Web Jest
- [ ] End-to-end test: Android ↔ Web chat through same backend instance
- [ ] Release: Docker image + APK + Web static bundle tagged with same version

---

## 6. Approval Gate

**No integration code will be written until you confirm.** The following will remain untouched until approval:

- `chat-master.zip` ✅ preserved
- `tindroid-master.zip` ✅ preserved
- `webapp-master.zip` ✅ preserved
- `chat-master/` extracted folder ✅ untouched
- `tindroid-master/` extracted folder ✅ untouched
- `webapp-master/` extracted folder ✅ untouched

**Please reply with:** `APPROVED` (to begin Phase 1) or list any changes you want to this architecture/roadmap first.