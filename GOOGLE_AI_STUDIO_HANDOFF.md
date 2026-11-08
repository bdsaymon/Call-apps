# Google AI Studio Migration & Development Handoff Package

**Repository:** `bdsaymon/Call-apps`  
**Branch:** `arena/bff36304-call-apps`  
**Date:** 2026-11-08  
**Target Platform:** Google AI Studio (development, integration, improvement)  
**Goal:** ONE unified Android + Web application using the three existing Tinode projects

---

## ⚠️ CONSTRAINTS (READ FIRST)

1. **Do not modify existing source code** in `chat-master/`, `tindroid-master/`, `webapp-master/`.
2. **Do not delete any files.**
3. **Do not merge projects** yet.
4. **Preserve all three ZIP files:** `chat-master.zip`, `tindroid-master.zip`, `webapp-master.zip`.
5. **Do not expose secrets or private credentials.** Use placeholders in config templates.
6. All original ZIPs and extracted folders must remain untouched until explicit approval for integration.

---

# 1. Complete Project Architecture

## 1.1 High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION LAYER                              │
│                                                                         │
│  ┌─────────────────────────┐                      ┌────────────────┐   │
│  │   ANDROID CLIENT        │                      │   WEB CLIENT   │   │
│  │   (tindroid-master)     │                      │  (webapp-master)│   │
│  │   Language: Java 17     │                      │  Language: JSX │   │
│  │   Framework: Android SDK│                      │  Framework: React│   │
│  │   minSdk: 27 / target: 36│                     │  Build: Webpack 5│   │
│  │   Transport: WebSocket  │                      │  Transport: WS │   │
│  │   Push: Firebase FCM    │                      │  Push: Firebase  │   │
│  │   Calls: Native WebRTC  │                      │  Calls: Browser WS│  │
│  └───────────┬─────────────┘                      └───────┬─────────┘   │
│              │                                            │              │
│              │  WebSocket JSON (protocol v0.25)           │              │
│              │  Endpoint: wss://host/v0/channels          │              │
│              └───────────────────────┬────────────────────┘              │
│                                      ▼                                   │
┌─────────────────────────────────────────────────────────────────────────┐
│                         API / TRANSPORT LAYER                           │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │              TINODE GO SERVER (chat-master)                     │   │
│  │  Language: Go 1.26.0  |  Version: 0.25  |  Module: chat        │   │
│  │                                                                   │   │
│  │  Core: main.go, topic.go, user.go, session.go, hub.go           │   │
│  │  Calls: calls.go (WebRTC signaling)                             │   │
│  │  Push: push.go + push/fcm/ (Firebase Cloud Messaging)           │   │
│  │  Auth: auth/{anon,basic,code,token,rest}/                       │   │
│  │  DB: db/{mysql,postgres,mongodb,rethinkdb}/adapter.go           │   │
│  │  Media: media/{fs,s3}/ (file upload/serve)                      │   │
│  │  Drafty: drafty/ (rich text formatting)                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                      │                                  │
┌──────────────────────────────────────┼──────────────────────────────────┐
│                    DATA / INFRASTRUCTURE LAYER                          │
│                                                                         │
│  ┌─────────────┐  ┌────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │   MySQL     │  │ PostgreSQL │  │   MongoDB   │  │   FS / S3    │   │
│  │   adapter   │  │   adapter  │  │   adapter   │  │ media handler│   │
│  └─────────────┘  └────────────┘  └─────────────┘  └──────────────┘   │
│  Auth: anon/basic/code/token/rest          Push: FCM / stdout / tnpg   │
│  Clustering: sharded + failover (ringhash)                              │
└─────────────────────────────────────────────────────────────────────────┘
```

## 1.2 Project Roles

| Project | Folder | Role | Language | Build | Files |
|---------|--------|------|----------|-------|-------|
| Backend | `chat-master/` | Tinode IM Server (Go) | Go 1.26.0 | Go modules / Docker | 264 |
| Android | `tindroid-master/` | Native Android client | Java 17 | Gradle 9.6.1 | 575 |
| Web | `webapp-master/` | React SPA web client | JavaScript/JSX | npm + Webpack 5 | 439 |

## 1.3 Unified Target Architecture

The end state is a single deployment where:
- One Go backend serves both Android and Web clients simultaneously
- Both clients connect to the same WebSocket endpoint (`/v0/channels`)
- Push notifications use one Firebase project for both platforms
- WebRTC call signaling is handled by the backend (media is client-to-client)
- Database is the system of record; client-side storage (SQLite/localStorage) is cache only

---

# 2. Full Directory Structure

## 2.1 chat-master/ (Backend — 264 files)

```
chat-master/
├── .github/ISSUE_TEMPLATE/
├── build-all.sh              # Cross-platform release builder (mac/linux/windows)
├── build-py-grpc.sh
├── chatbot/                  # Python & C# chatbot examples
│   ├── csharp/
│   └── python/               # chatbot.py, setup.py, quotes.txt
├── docker/
│   ├── chatbot/Dockerfile
│   ├── docker-compose/       # Multi-container deployment
│   ├── exporter/
│   └── tinode/
├── docs/                     # API.md, INSTALL.md, translations, screenshots
├── keygen/                   # API key generation
├── loadtest/                 # Load testing (Scala/Erlang)
├── monitoring/exporter/      # Prometheus exporter
├── pbx/                      # Protobuf model (model.proto, model.pb.go)
├── py_grpc/                  # Python gRPC bindings
├── rest-auth/                # REST auth example
├── server/                   # CORE BACKEND (80 .go files)
│   ├── auth/{anon,basic,code,rest,token,mock_auth}/
│   ├── concurrency/
│   ├── db/{common,mysql,postgres,mongodb,rethinkdb}/
│   ├── drafty/
│   ├── logs/
│   ├── media/{fs,s3}/
│   ├── push/{common,fcm,stdout,tnpg}/
│   ├── ringhash/
│   ├── store/{mock_store,types}/
│   ├── templ/
│   ├── validate/{email,tel}/
│   ├── api_key.go
│   ├── calls.go
│   ├── cluster.go / cluster_leader.go
│   ├── datamodel.go          # Wire protocol message structures
│   ├── hdl_files.go / hdl_grpc.go / hdl_longpoll.go / hdl_websock.go
│   ├── http.go / http_pprof.go
│   ├── hub.go
│   ├── init_topic.go
│   ├── main.go               # Entry point, config, route registration
│   ├── pbconverter.go
│   ├── plugins.go
│   ├── pres.go
│   ├── session.go / sessionstore.go
│   ├── stats.go
│   ├── topic.go / topic_proxy.go
│   ├── user.go
│   ├── utils.go
│   └── validate/validator.go
├── tinode-db/                # DB seed data
└── tn-cli/                   # Python admin CLI
```

## 2.2 tindroid-master/ (Android — 575 files)

```
tindroid-master/
├── .github/ISSUE_TEMPLATE/
├── build.gradle              # Root Gradle config (AGP 9.4.1, Firebase plugins)
├── settings.gradle           # Includes :app, :tinodesdk
├── gradle/wrapper/           # Gradle 9.6.1 wrapper
├── contacts.vcf              # Test contacts
├── app/                      # Android application module
│   ├── build.gradle          # App config (minSdk 27, targetSdk 36)
│   ├── proguard-rules.pro
│   └── src/
│       ├── androidTest/java/co/tinode/tindroid/
│       ├── debug/AndroidManifest.xml
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── assets/dcodes.json
│       │   ├── java/co/tinode/tindroid/     # ~100 Java files
│       │   │   ├── Activities: LoginActivity, MessageActivity, ChatsActivity,
│       │   │   │   CallActivity, InitRouterActivity, StartChatActivity, SendToActivity
│       │   │   ├── Fragments: LoginFragment, SignUpFragment, MessagesFragment,
│       │   │   │   ChatsFragment, ContactsAdapter, CreateGroupFragment, CallFragment,
│       │   │   │   IncomingCallFragment, TopicInfoFragment, WallpaperFragment, etc.
│       │   │   ├── Services: FBaseMessagingService, TinodeAccountService,
│       │   │   │   ContactsSyncAdapter, CallConnectionService
│       │   │   ├── DB: AccountDb, MessageDb, TopicDb, UserDb, SubscriberDb, SqlStore
│       │   │   ├── Format: Drafty formatters/spans
│       │   │   ├── Media: VxCard, Wallpapers, MediaControl, AudioControl
│       │   │   ├── Call: CallManager, CallConnection, HangUpBroadcastReceiver
│       │   │   └── Utils: Const, Cache, BrandingConfig, UiUtils, UtilsString, etc.
│       │   └── res/          # Layouts, drawables, values, icons
│       └── test/java/co/tinode/tindroid/
└── tinodesdk/                # Tinode Android SDK (no Android SDK deps)
    ├── src/main/java/co/tinode/tinodesdk/
    │   ├── Tinode.java           # Main client class (PROTOVERSION="0", VERSION="0.25")
    │   ├── Connection.java       # WebSocket wrapper (java-websocket Draft_6455)
    │   ├── Topic.java, ComTopic.java, FndTopic.java, MeTopic.java
    │   ├── User.java, Storage.java, LocalData.java
    │   ├── LargeFileHelper.java, ExpBackoff.java, PromisedReply.java
    │   └── model/                # Protocol message models
    │       ├── MsgClientHi, MsgClientLogin, MsgClientPub, MsgClientSub
    │       ├── MsgClientSet, MsgClientDel, MsgClientAcc, MsgClientNote
    │       ├── MsgServerData, MsgServerPres, MsgServerInfo, MsgServerCtrl
    │       ├── Drafty, Acs, Subscription, TheCard, Credential, AuthScheme
    │       └── ... (40+ model classes)
    └── src/test/java/co/tinode/tinodesdk/
```

## 2.3 webapp-master/ (Web — 439 files)

```
webapp-master/
├── .babelrc
├── .gitignore
├── .well-known/assetlinks.json
├── audio/                    # call-end.m4a, call-in.m4a, call-out.m4a, dialing.m4a, msg.m4a
├── css/
│   ├── base.css              # Main stylesheet
│   └── base.min.css          # Minified
├── firebase-init.js          # Firebase config (PLACEHOLDER keys)
├── img/                      # badge96.png, bkg/ backgrounds
├── index-dev.html            # Dev entry point
├── index.html                # Production entry (loads umd bundles)
├── manifest.json             # PWA manifest
├── package.json              # npm config (React 19.3, tinode-sdk 0.25.4, Firebase 13.1)
├── package-lock.json
├── postcss.config.js
├── push.md                   # Push notification setup guide
├── scripts/                  # i18n, version generation
├── service-worker.js         # PWA service worker
├── umd/                      # Built bundles (index.prod.js, index.dev.js, tinode.prod.js, etc.)
├── version.js
├── web-app.svg, web-desktop.jpg, web-mob-*.png
├── webpack.config.js         # Webpack 5 config (UMD output, CopyPlugin for tinode-sdk)
└── src/                      # React source (128 files)
    ├── config.js             # API_KEY, DEFAULT_HOST, limits, access modes
    ├── constants.js          # Call states, call head constants
    ├── dcodes.json
    ├── index.js              # React entry, IntlProvider, language detection
    ├── version.js
    ├── i18n/                 # 14 locale JSON files (ar, de, en, es, fr, id, it, ko, ro, ru, th, uk, vi, zh, zh-TW)
    ├── i18n.min/             # Minified i18n
    ├── lib/                  # Utilities
    │   ├── blob-helpers.js
    │   ├── formatters.js
    │   ├── host-name.js
    │   ├── local-storage.js
    │   ├── navigation.js     # Hash-based routing
    │   ├── react-intl-compat.js
    │   ├── strformat.js
    │   └── utils.js
    ├── views/                # React views (~25 files)
    │   ├── tinode-web.jsx    # Root component (call state, Firebase init)
    │   ├── messages-view.jsx
    │   ├── contacts-view.jsx
    │   ├── login-view.jsx
    │   ├── create-account-view.jsx
    │   ├── account-settings-view.jsx
    │   ├── sidepanel-view.jsx
    │   ├── info-view.jsx
    │   ├── new-topic-view.jsx
    │   ├── settings-view.jsx
    │   └── ... (acc-*, password-reset, validation, wallpapers, logo, topic-common)
    └── widgets/              # React widgets (~60 files)
        ├── call-incoming.jsx, call-panel.jsx, call-status.jsx, call-message.jsx
        ├── chat-message.jsx
        ├── contact-list.jsx, contact.jsx, contact-badges.jsx
        ├── group-manager.jsx, group-subs.jsx
        ├── image-preview.jsx, video-preview.jsx, inline-video.jsx
        ├── audio-player.jsx, audio-recorder.jsx
        ├── attachment.jsx, doc-preview.jsx, forward-dialog.jsx
        ├── send-message.jsx, pinned-messages.jsx, search-contacts.jsx
        ├── phone-edit.jsx, phone-country-selector.jsx
        ├── show-qrcode.jsx, upload-image.jsx, lazy-image.jsx
        ├── avatar-crop.jsx, avatar-upload.jsx, letter-tile.jsx
        ├── topic-security.jsx, topic-desc-edit.jsx, permissions-editor.jsx
        ├── the-card-preview.jsx, the-card-mini.jsx, badge-list.jsx
        ├── meta-message.jsx, received-marker.jsx, unread-badge.jsx
        ├── load-spinner.jsx, error-panel.jsx, context-menu.jsx
        ├── side-navbar.jsx, menu-start.jsx, menu-contacts.jsx, menu-cancel.jsx
        ├── invitation.jsx, credential-edit.jsx, visible-password.jsx
        ├── file-progress.jsx, button-back.jsx, chip.jsx, chip-input.jsx
        ├── tag-manager.jsx, alert.jsx, checkbox.jsx, cropper.jsx
        └── ... (host-selector.jsx, in-place-edit.jsx, new-topic-by-id.jsx, etc.)
```

---

# 3. Important Source Files and Responsibilities

## 3.1 Backend — `chat-master/server/`

| File | Responsibility |
|------|---------------|
| `main.go` | Server entry point, config loading, route registration (`/v0/channels` WebSocket), DB adapter registration, auth handler registration, push provider registration, media handler registration |
| `datamodel.go` | Wire protocol message structures (C2S: hi/login/sub/pub/set/get/del/note/acc; S2C: ctrl/data/meta/info/pres) |
| `topic.go` | Topic object (P2P/group/channel), message broadcast, call state (`videoCall`), call event handling |
| `user.go` | Account creation, authentication, credentials, tags, access control |
| `session.go` | Client session management (WebSocket, long-poll, proxy) |
| `hub.go` | Message routing hub between topics and sessions |
| `calls.go` | WebRTC call signaling: invite, accept, offer/answer, ICE candidate, hang-up, call establishment timeout |
| `push.go` | Push notification receipt preparation (FCM payload construction) |
| `store/store.go` | Storage abstraction interface |
| `store/types/types.go` | Data types: User, Topic, Subscription, AccessMode, TopicCat, Uid |
| `db/adapter.go` | Pluggable database adapter interface |
| `db/mysql/adapter.go` | MySQL/MariaDB adapter |
| `db/postgres/adapter.go` | PostgreSQL adapter |
| `db/mongodb/adapter.go` | MongoDB adapter |
| `db/rethinkdb/adapter.go` | RethinkDB adapter (deprecated) |
| `auth/auth.go` | Auth handler base |
| `auth/anon/auth_anon.go` | Anonymous authentication |
| `auth/basic/auth_basic.go` | Basic (username:password) auth |
| `auth/code/auth_code.go` | Code-based auth (email/SMS verification) |
| `auth/token/auth_token.go` | Token auth |
| `auth/rest/auth_rest.go` | REST auth |
| `push/push.go` | Push provider interface |
| `push/fcm/push_fcm.go` | Firebase Cloud Messaging push provider |
| `push/stdout/push_stdout.go` | Stdout push (dev) |
| `push/tnpg/push_tnpg.go` | Tinode Push Gateway |
| `media/media.go` | Media handler interface (file upload/serve) |
| `media/fs/filesys.go` | Local filesystem media handler |
| `media/s3/s3.go` | Amazon S3 media handler |
| `drafty/drafty.go` | Drafty rich text formatter |
| `plugins.go` | Server plugin system |

## 3.2 Android SDK — `tindroid-master/tinodesdk/`

| File | Responsibility |
|------|---------------|
| `Tinode.java` | Main client class. Connection management, login, topic operations, message publishing. PROTOVERSION="0", VERSION="0.25" |
| `Connection.java` | WebSocket wrapper using `java-websocket` Draft_6455 with PerMessageDeflate. Reconnect with exponential backoff |
| `Topic.java` | Base topic class |
| `ComTopic.java` | Common topic (group/P2P) with subscription management |
| `FndTopic.java` | Find/users topic |
| `MeTopic.java` | Self topic |
| `User.java` | User object |
| `Storage.java` | Storage interface |
| `LocalData.java` | Local data persistence |
| `LargeFileHelper.java` | Out-of-band file transfer helper |
| `ExpBackoff.java` | Exponential backoff for reconnection |
| `PromisedReply.java` | Promise-based async request/response |
| `RFC3339Format.java` | Timestamp formatting |
| `model/MsgClientHi.java` | Client handshake {hi} |
| `model/MsgClientLogin.java` | Client login {login} |
| `model/MsgClientPub.java` | Client publish {pub} |
| `model/MsgClientSub.java` | Client subscribe {sub} |
| `model/MsgClientSet.java` | Client set {set} |
| `model/MsgClientDel.java` | Client delete {del} |
| `model/MsgClientAcc.java` | Client account {acc} |
| `model/MsgClientNote.java` | Client note {note} |
| `model/MsgServerData.java` | Server data {data} |
| `model/MsgServerPres.java` | Server presence {pres} |
| `model/MsgServerInfo.java` | Server info {info} |
| `model/MsgServerCtrl.java` | Server control {ctrl} |
| `model/MsgServerMeta.java` | Server meta {meta} |
| `model/Drafty.java` | Drafty rich text |
| `model/Acs.java` | Access control |
| `model/Subscription.java` | Subscription object |
| `model/TheCard.java` | User profile card |
| `model/Credential.java` | Credential object |
| `model/AuthScheme.java` | Authentication scheme |

## 3.3 Android App — `tindroid-master/app/src/main/java/co/tinode/tindroid/`

| File | Responsibility |
|------|---------------|
| `TindroidApp.java` | Application class, global state |
| `Cache.java` | Singleton cache for Tinode instance, call state, selected topic |
| `Const.java` | Constants (intent actions, notification channels) |
| `LoginActivity.java` / `LoginFragment.java` | Login screen |
| `SignUpFragment.java` | Registration screen |
| `ChatsActivity.java` / `ChatsFragment.java` | Chat list |
| `MessageActivity.java` / `MessagesFragment.java` | Message conversation |
| `ContactsAdapter.java` | Contacts list adapter |
| `CreateGroupFragment.java` | Create group chat |
| `CallActivity.java` | Call screen (incoming/active) |
| `CallFragment.java` | Call UI fragment |
| `IncomingCallFragment.java` | Incoming call UI |
| `CallManager.java` | Call state management, TelecomManager integration |
| `CallConnection.java` / `CallConnectionService.java` | Android Telecom API integration |
| `HangUpBroadcastReceiver.java` | Call hangup broadcast |
| `FBaseMessagingService.java` | Firebase FCM message handling (push notifications, call notifications) |
| `TinodeAccountService.java` | Android account authenticator |
| `ContactsSyncAdapter.java` / `ContactsSyncService.java` | Android Contacts sync |
| `AccountDb.java`, `MessageDb.java`, `TopicDb.java`, `UserDb.java`, `SubscriberDb.java` | SQLite local database |
| `SqlStore.java` | SQLite store implementation |
| `StoredAccount.java`, `StoredMessage.java`, `StoredTopic.java`, `StoredUser.java`, `StoredSubscription.java` | Stored model objects |
| `format/AbstractDraftyFormatter.java` | Drafty text formatting for Android |
| `format/FullFormatter.java`, `CopyFormatter.java`, `QuoteFormatter.java`, etc. | Various text formatters |
| `media/VxCard.java` | vCard for user profile |
| `media/Wallpapers.java` | Wallpaper management |
| `services/CallConnectionService.java` | Call connection service |
| `widgets/QRCodeScanner.java` | QR code scanning |
| `widgets/AttachmentPickerDialog.java` | Attachment picker |

## 3.4 Web — `webapp-master/src/`

| File | Responsibility |
|------|---------------|
| `index.js` | React entry point, IntlProvider, language detection from URL/hash/browser, renders `TinodeWeb` |
| `tinode-web.jsx` | Root React component. Manages connection state, call state (incoming/outgoing/in-progress), Firebase init, topic navigation |
| `config.js` | API_KEY, APP_NAME, DEFAULT_HOST, KNOWN_HOSTS, call config, media limits (MAX_INBAND_ATTACHMENT_SIZE=8MB, MAX_IMAGE_DIM=1024), access modes (JRWPS, JRWPSAO, JR), i18n settings, wallpaper defaults |
| `constants.js` | Call state constants (CALL_STATE_NONE, OUTGOING_INITATED, INCOMING_RECEIVED, IN_PROGRESS), call head constants |
| `lib/config.js` | (same as config.js) |
| `lib/host-name.js` | Server address detection (localhost vs remote, secure vs insecure) |
| `lib/local-storage.js` | Local storage wrapper |
| `lib/navigation.js` | Hash-based URL routing |
| `lib/blob-helpers.js` | Blob/base64 image helpers |
| `lib/formatters.js` | Message formatters |
| `lib/strformat.js` | String formatting utilities |
| `lib/utils.js` | General utilities (wallpaper, favicon) |
| `lib/react-intl-compat.js` | React-intl compatibility shim |
| `views/messages-view.jsx` | Message list view |
| `views/contacts-view.jsx` | Contacts view |
| `views/login-view.jsx` | Login view |
| `views/create-account-view.jsx` | Account creation view |
| `views/account-settings-view.jsx` | Account settings |
| `views/sidepanel-view.jsx` | Side panel |
| `views/info-view.jsx` | Topic info view |
| `views/new-topic-view.jsx` | New topic creation |
| `views/settings-view.jsx` | App settings |
| `views/acc-*.jsx` | Account sub-views (general, notifications, security, support) |
| `views/password-reset-view.jsx` | Password reset |
| `views/validation-view.jsx` | Credential validation |
| `views/wallpapers.jsx` | Wallpaper selection |
| `widgets/call-incoming.jsx` | Incoming call UI (accept/reject, ring sound) |
| `widgets/call-panel.jsx` | Active call UI (local/remote video, controls, WebRTC) |
| `widgets/call-status.jsx` | Call status indicator |
| `widgets/call-message.jsx` | Call event message in chat |
| `widgets/chat-message.jsx` | Individual chat message |
| `widgets/contact-list.jsx` | Contact list |
| `widgets/group-manager.jsx` | Group management |
| `widgets/group-subs.jsx` | Group subscribers |
| `widgets/image-preview.jsx` | Image preview |
| `widgets/video-preview.jsx` | Video preview |
| `widgets/inline-video.jsx` | Inline video playback |
| `widgets/audio-player.jsx` / `audio-recorder.jsx` | Audio playback/recording |
| `widgets/attachment.jsx` | File attachment |
| `widgets/doc-preview.jsx` | Document preview |
| `widgets/forward-dialog.jsx` | Message forwarding |
| `widgets/pinned-messages.jsx` | Pinned messages |
| `widgets/search-contacts.jsx` | Contact search |
| `widgets/send-message.jsx` | Message input/send |
| `widgets/phone-edit.jsx` / `phone-country-selector.jsx` | Phone number input |
| `widgets/show-qrcode.jsx` | QR code display |
| `widgets/upload-image.jsx` | Image upload |
| `widgets/lazy-image.jsx` | Lazy image loading |
| `widgets/avatar-crop.jsx` / `avatar-upload.jsx` | Avatar handling |
| `widgets/letter-tile.jsx` | Avatar letter tile |
| `widgets/topic-security.jsx` / `topic-desc-edit.jsx` | Topic security/description |
| `widgets/permissions-editor.jsx` | Permission editing |
| `widgets/the-card-preview.jsx` / `the-card-mini.jsx` | Profile card |
| `widgets/badge-list.jsx` / `contact-badges.jsx` | Badges (verified, staff) |
| `widgets/meta-message.jsx` | System messages |
| `widgets/received-marker.jsx` / `unread-badge.jsx` | Read/unread indicators |
| `widgets/load-spinner.jsx` / `error-panel.jsx` | Loading/error UI |
| `widgets/context-menu.jsx` | Context menus |
| `widgets/side-navbar.jsx` | Navigation bar |
| `widgets/menu-start.jsx` / `menu-contacts.jsx` / `menu-cancel.jsx` | Menu items |
| `widgets/invitation.jsx` | Invite contacts |
| `widgets/credential-edit.jsx` | Credential editing |
| `widgets/visible-password.jsx` | Password toggle |
| `widgets/file-progress.jsx` | Upload progress |
| `widgets/button-back.jsx` | Back button |
| `widgets/chip.jsx` / `chip-input.jsx` | Chip UI |
| `widgets/tag-manager.jsx` | Tag management |
| `widgets/alert.jsx` | Alert dialog |
| `widgets/checkbox.jsx` | Checkbox |
| `widgets/cropper.jsx` | Image cropper |
| `widgets/host-selector.jsx` | Server host selector |
| `widgets/in-place-edit.jsx` | Inline editing |
| `widgets/new-topic-by-id.jsx` | New topic by ID |
| `i18n/` | 14 language translation files |
| `i18n.min/` | Minified translations |

---

# 4. Backend API and WebSocket Documentation

## 4.1 Protocol Version

| Component | Version |
|-----------|---------|
| Backend (Go) | `currentVersion = "0.25"`, `minSupportedVersion = "0.20"` |
| Android SDK | `VERSION = "0.25"`, `PROTOVERSION = "0"` |
| Web JS SDK | `tinode-sdk ^0.25.4` |

## 4.2 WebSocket Endpoint

```
URL:  ws://host/v0/channels   (or wss://host/v0/channels with TLS)
Path: config.ApiPath + "v0/channels"   (ApiPath defaults to "/")
```

- **Android:** `Connection.java` connects to `ws://host/v0/channels`
- **Web:** JavaScript `tinode-sdk` connects to same endpoint
- **Backend:** `main.go:799` registers `mux.HandleFunc(config.ApiPath+"v0/channels", serveWebSocket)`
- **Long-poll fallback:** `config.ApiPath + "v0/channels/lp"`
- **File upload:** `config.ApiPath + "v0/file/u/"`
- **File serve:** `config.ApiPath + "v0/file/s/"`

## 4.3 Wire Protocol Messages

### Client → Server (C2S)

| Message | Purpose | Key Fields |
|---------|---------|------------|
| `{hi}` | Handshake | id, ver, ua, dev, lang, bkg |
| `{login}` | Authenticate | scheme, secret, cred[] |
| `{sub}` | Subscribe to topic | topic, mode (want/given) |
| `{pub}` | Publish message | topic, content, head, mime |
| `{set}` | Set metadata/desc/sub | desc, sub, tags, cred |
| `{get}` | Get data/meta/desc | what (desc/sub/data/del), opts |
| `{del}` | Delete messages | ranges, since, before |
| `{note}` | Send note (call events, typing) | what, payload |
| `{acc}` | Account operations | login, secret, state, tags |

### Server → Client (S2C)

| Message | Purpose | Key Fields |
|---------|---------|------------|
| `{ctrl}` | Control/error | code, text, params, ref |
| `{data}` | Message data | topic, from, seq, timestamp, content, head |
| `{meta}` | Metadata update | topic, desc, sub, tags |
| `{info}` | Info notification | what, event, seqId, payload |
| `{pres}` | Presence update | topic, from, online, away |

## 4.4 Access Modes

| Mode | Meaning |
|------|---------|
| `J` | Join |
| `R` | Read |
| `W` | Write |
| `P` | Presence |
| `S` | Subscribe |
| `A` | Admin |
| `O` | Owner |

| Combination | Use |
|-------------|-----|
| `JRWPS` | Default P2P access (from Android/Web config) |
| `JRWPSAO` | New group topic access |
| `JR` | Channel (read-only) access |
| `N` | No access |

## 4.5 Topic Types

| Type | Prefix | Description |
|------|--------|-------------|
| `me` | `me` | Self topic |
| `fnd` | `fnd` | Find/users discovery |
| `sys` | `sys` | System topic |
| `slf` | `slf` | Self metadata |
| `grp` | `grpXXX` | Group topic |
| `p2p` | (user ID) | Peer-to-peer topic |
| `nch` | `nch` | Channel (read-only subscribers) |

## 4.6 Call Signaling (WebRTC)

| Event | Direction | Purpose |
|-------|-----------|---------|
| `started` | Client → Server | Call invitation |
| `ringing` | Server → Client | Callee notified |
| `accept` | Client → Server | Call accepted |
| `offer` | Client → Client (via server) | WebRTC SDP offer |
| `answer` | Client → Client (via server) | WebRTC SDP answer |
| `ice-candidate` | Client → Client (via server) | ICE candidate exchange |
| `hang-up` | Server → Client | Call terminated |
| `missed` / `declined` / `finished` / `disconnected` | Server → Client | Call end states |

**Backend:** `calls.go` handles all signaling. Media is client-to-client WebRTC; backend never sees media data.

---

# 5. Android SDK Integration Documentation

## 5.1 SDK Location

`tindroid-master/tinodesdk/` — Java library with **no Android SDK dependencies**. Can be used in plain Java projects.

## 5.2 Integration Steps

```java
// 1. Create Tinode instance
Tinode tinode = new Tinode(context, appName, apiKey, listener);

// 2. Connect to server
URI serverURI = new URI("wss://your-server.com/v0/channels");
tinode.connect(serverURI, false);

// 3. Login
tinode.login("basic", "username:password");

// 4. Get topics
Topic[] topics = tinode.getStore().topicGetAll(tinode);

// 5. Subscribe to a topic
Topic topic = tinode.getTopic("grp123");
topic.sub(Tinode.ACCESS_MODE_READ | Tinode.ACCESS_MODE_WRITE, ...);

// 6. Publish a message
topic.publish(content, mimeType, head);

// 7. Handle incoming messages
topic.setOnDataListener((topic, msg) -> { ... });
```

## 5.3 Connection Details

- **WebSocket library:** `org.java_websocket` Draft_6455 with PerMessageDeflate extension
- **Reconnect:** Exponential backoff (`ExpBackoff.java`)
- **Authentication:** API key sent as `X-Tinode-APIKey` header
- **JSON serialization:** Jackson (`com.fasterxml.jackson`)
- **Message models:** Jackson-annotated POJOs in `model/` package

## 5.4 App-Level Integration

- `app/build.gradle` includes `implementation project(path: ':tinodesdk')`
- `tinodesdk` is a separate Gradle module
- FCM push: `FBaseMessagingService.java` receives push, calls `tinode.oobNotification()`
- Call notifications: FCM data payload with `webrtc` field triggers `handleCallNotification()`
- Local storage: SQLite via `db/` package (`AccountDb`, `MessageDb`, `TopicDb`, etc.)

---

# 6. React Web Integration Documentation

## 6.1 SDK Location

Web app uses `tinode-sdk` npm package (v0.25.4) — the JavaScript Tinode SDK. Not bundled in repo; installed via `npm install`.

## 6.2 Integration Steps

```jsx
// 1. Import Tinode
import { Drafty, Tinode } from 'tinode-sdk';

// 2. Setup connection
const tinode = TinodeWeb.tnSetup(serverAddress, isSecure, transport, ...);

// 3. Login
tinode.login(cred);

// 4. Subscribe to topics
const topic = tinode.getTopic(topicName);
topic.sub(accessMode);

// 5. Render messages
topic.onData = (topic, msg) => { setMessages(...); };

// 6. Publish
topic.publish(content, 'text/plain', null);
```

## 6.3 Webpack Configuration

`webpack.config.js`:
- Entry: `src/index.js`
- Output: `umd/[name].{mode}.js` (UMD format)
- `tinode-sdk` is externalized as `var tinode` (loaded from UMD bundle via CopyPlugin)
- React and react-intl are externalized (loaded from esm.sh CDN in `index.html`)
- CopyPlugin copies `node_modules/tinode-sdk/umd/tinode.{mode}.js` to output

## 6.4 Internationalization

- **Library:** `react-intl` v12.1.3
- **Locales:** 14 languages (ar, de, en, es, fr, id, it, ko, ro, ru, th, uk, vi, zh, zh-TW)
- **Files:** `src/i18n/*.json` (source) + `src/i18n.min/*.json` (minified)
- **Extraction:** `formatjs extract` CLI
- **Language detection:** URL hash `#?hl=XX` → browser language → fallback `en`
- **RTL support:** `ar`, `fa`, `he`, `ur`

## 6.5 Push Notifications (Web)

- **Library:** Firebase v13.1.0 (`firebase/messaging`)
- **Config:** `firebase-init.js` (API key, projectId `tinode-1000`, VAPID key)
- **Service Worker:** `service-worker.js` handles background push
- **Token:** `firebaseGetToken()` → sent to backend via SDK

## 6.6 WebRTC Calls (Web)

- **API:** Browser native `RTCPeerConnection`
- **UI:** `call-incoming.jsx` (incoming), `call-panel.jsx` (active call)
- **Sounds:** `audio/call-in.m4a`, `audio/call-out.m4a`, `audio/call-end.m4a`, `audio/dialing.m4a`
- **Media constraints:** `audio: true, video: !callAudioOnly`
- **ICE candidates:** Exchanged via backend signaling (`webrtc` header on `{note}` messages)

---

# 7. Database Schema and Configuration

## 7.1 Supported Databases

| Adapter | Package | Status |
|---------|---------|--------|
| MySQL / MariaDB | `server/db/mysql/adapter.go` | Production-ready |
| PostgreSQL | `server/db/postgres/adapter.go` | Production-ready |
| MongoDB | `server/db/mongodb/adapter.go` | Production-ready |
| RethinkDB | `server/db/rethinkdb/adapter.go` | Deprecated (dropped 2027) |

## 7.2 Database Selection

Configured in `tinode.conf` under `db` section. Adapter is registered via import side-effect in `main.go`:
```go
_ "github.com/tinode/chat/server/db/mysql"
_ "github.com/tinode/chat/server/db/postgres"
```

## 7.3 Core Data Types (`server/store/types/`)

| Type | Fields |
|------|--------|
| `User` | ID, state, public, trusted, private, credentials, tags, defacs, aux |
| `Topic` | name, category (P2P/GRP/CHN/SLF/FND/SYS), owner, access modes, tags, subscribers, sessions |
| `Subscription` | user, topic, mode (want/given), deleted, lastSeq, unread count |
| `AccessMode` | Bitmask of J/R/W/P/S/A/O |
| `TopicCat` | P2P, GRP, CHN, SLF, FND, SYS |
| `Uid` | User ID type |

## 7.4 Configuration (`tinode.conf`)

Location: `chat-master/server/tinode.conf` (template exists)

Key sections:
```json
{
  "db": { "adapter": "mysql", "datasource": "user:pass@tcp(host:3306)/dbname" },
  "push": [
    { "name": "fcm", "service_account": {...} },
    { "name": "stdout" }
  ],
  "media": { "handler": "fs", "path": "/var/tinode/media" },
  "calls": { "enabled": true, "ice_servers": [...], "call_establishment_timeout": 30 },
  "auth": { "schemes": ["anon", "basic", "code", "token"] },
  "api_path": "/",
  "listen": ":6060",
  "max_message_size": 524288,
  "ice_servers": [{ "urls": ["stun:stun.l.google.com:19302"] }]
}
```

## 7.5 Client-Side Storage

| Client | Storage | Purpose |
|--------|---------|---------|
| Android | SQLite (`db/` package) | Local cache of accounts, messages, topics, users |
| Web | `localStorage` + service worker | Local cache, PWA offline support |

**Note:** Client-side storage is cache only. Backend DB is system of record.

---

# 8. Authentication and Firebase Setup Instructions

## 8.1 Authentication Schemes (Backend)

| Scheme | Package | Use Case |
|--------|---------|----------|
| Anonymous | `auth/anon/` | Public read-only access |
| Basic | `auth/basic/` | Username:password login |
| Code | `auth/code/` | Email/SMS verification code |
| Token | `auth/token/` | Bearer token auth |
| REST | `auth/rest/` | External REST auth server |
| Mock | `auth/mock_auth/` | Testing |

## 8.2 Credential Validators

| Validator | Package | Purpose |
|-----------|---------|---------|
| Email | `validate/email/` | Email format validation |
| Tel | `validate/tel/` | Phone number validation (Twilio) |

## 8.3 Firebase Cloud Messaging (Push)

### For Android:
1. Create Firebase project at https://console.firebase.google.com/
2. Add Android app with package name `co.tinode.tindroidx`
3. Download `google-services.json`
4. Place at `tindroid-master/app/google-services.json` (gitignored)
5. Copy FCM server key to `tinode.conf` push.fcm section

### For Web:
1. Add Web app to same Firebase project
2. Get config object (apiKey, projectId, messagingSenderId, appId, messagingVapidKey)
3. Update `webapp-master/firebase-init.js` with real values

### For Backend:
1. Generate Firebase Admin SDK service account JSON
2. Configure in `tinode.conf` push.fcm.service_account

**CRITICAL:** All three (Android, Web, Backend) must use the **same Firebase project** for unified push notifications.

### Current Placeholder Values (DO NOT USE IN PRODUCTION):
- Web `firebase-init.js`: `projectId: "tinode-1000"` (demo keys)
- Backend `tinode.conf`: `client_email: "firebase-adminsdk-abc123@your-project-id.iam.gserviceaccount.com"`

---

# 9. Voice/Video Calling Architecture

## 9.1 Call Flow

```
Caller                            Backend                            Callee
  │                                  │                                  │
  │  {pub head=[webrtc: started]}    │                                  │
  │  ───────────────────────────────►│                                  │
  │                                  │  {info what=call event=ringing} │
  │                                  │  ───────────────────────────────►│
  │                                  │                                  │
  │◄─────────────────────────────────│  {info what=call event=accept}  │
  │  {note what=call event=offer}    │                                  │
  │  ───────────────────────────────►│                                  │
  │                                  │  {note what=call event=answer}  │
  │  ◄───────────────────────────────│                                  │
  │  WebRTC media (direct P2P) ◄────►│                                  │
  │                                  │                                  │
  │  {note what=call event=hang-up}  │                                  │
  │  ───────────────────────────────►│                                  │
  │                                  │  {info what=call event=hang-up} │
  │                                  │  ───────────────────────────────►│
```

## 9.2 Backend Call Handling (`calls.go`)

- `videoCall` struct: tracks parties, sequence, content, accepted time
- `handleCallInvite()`: initiates call, starts establishment timer
- `handleCallEvent()`: handles ringing/accept/offer/answer/ice-candidate/hang-up
- `maybeEndCallInProgress()`: finalizes call (finished/missed/declined/disconnected)
- `terminateCallInProgress()`: server-initiated termination
- Config: `callConfig{Enabled, CallEstablishmentTimeout, ICEServers[], ICEServersFile}`

## 9.3 Android Call Implementation

- **Activity:** `CallActivity.java` (incoming/active call UI)
- **Fragment:** `CallFragment.java`, `IncomingCallFragment.java`
- **Manager:** `CallManager.java` (TelecomManager integration, PhoneAccount)
- **Connection:** `CallConnection.java`, `CallConnectionService.java` (Android Telecom API)
- **WebRTC:** `com.dafruits:webrtc:123.0.0`
- **Camera:** CameraX 1.6.2
- **Audio:** Media3 ExoPlayer 1.11.1
- **FCM:** Call notifications via `FBaseMessagingService.handleCallNotification()`

## 9.4 Web Call Implementation

- **Incoming:** `call-incoming.jsx` (accept/reject, ring sound `audio/call-in.m4a`)
- **Active:** `call-panel.jsx` (local/remote video, mute, hang-up, `RTCPeerConnection`)
- **Sounds:** `audio/call-out.m4a`, `audio/call-end.m4a`, `audio/dialing.m4a`
- **WebRTC:** Browser native `RTCPeerConnection`, `getUserMedia`
- **ICE:** Candidates exchanged via backend signaling

## 9.5 ICE Server Configuration

Configured in `tinode.conf`:
```json
"calls": {
  "enabled": true,
  "call_establishment_timeout": 30,
  "ice_servers": [
    { "urls": ["stun:stun.l.google.com:19302"] },
    { "urls": ["turn:your-turn-server:3478"], "username": "...", "credential": "..." }
  ]
}
```

---

# 10. Build Instructions for Android APK and Web

## 10.1 Prerequisites

| Tool | Android | Web | Backend |
|------|---------|-----|---------|
| JDK 17 | Required | N/A | N/A |
| Android SDK Platform 36 | Required | N/A | N/A |
| Go 1.26.0+ | N/A | N/A | Required |
| Node.js LTS | N/A | Required | N/A |
| npm | N/A | Required | N/A |
| Gradle 9.6.1 | Wrapper included | N/A | N/A |
| Docker | Optional | Optional | Optional |

## 10.2 Android APK Build

```bash
cd tindroid-master

# 1. Create keystore.properties (REQUIRED for release, NOT committed)
cat > keystore.properties << EOF
storeFile=/path/to/release.keystore
storePassword=your-store-password
keyAlias=your-key-alias
keyPassword=your-key-password
EOF

# 2. Place google-services.json (REQUIRED for FCM, gitignored)
cp /path/to/google-services.json app/

# 3. Debug build
./gradlew assembleDebug
# Output: app/build/outputs/apk/debug/app-debug.apk

# 4. Release build
./gradlew assembleRelease
# Output: app/build/outputs/apk/release/app-release.apk

# 5. Install on device
./gradlew installDebug

# 6. Run tests
./gradlew test
./gradlew connectedDebugAndroidTest
```

**Note:** If building from extracted archive (not git clone), replace `gitVersionCode()`/`gitVersionName()` in `build.gradle` with static values.

## 10.3 Web Build

```bash
cd webapp-master

# 1. Install dependencies
npm install

# 2. Build (production + development)
npm run build
# Runs: vers → build:css → extract → build:i18n → build:prod → build:dev
# Output: umd/index.prod.js, umd/index.dev.js, css/base.min.css, etc.

# 3. Serve static files
# Deploy umd/, css/, img/, audio/, index.html, manifest.json,
# service-worker.js, firebase-init.js, version.js to web server
```

**npm scripts:**
| Script | Purpose |
|--------|---------|
| `npm run build` | Full production + dev build |
| `npm run build:prod` | Webpack production mode |
| `npm run build:dev` | Webpack development mode |
| `npm run build:css` | PostCSS minification |
| `npm run build:i18n` | Translation compilation |
| `npm run clean` | Remove built assets |
| `npm test` | Jest unit tests |

## 10.4 Backend Build

```bash
cd chat-master

# 1. Install Go 1.26.0+
# 2. Download dependencies
go mod download

# 3. Build
go build -o tinode ./server

# 4. Configure tinode.conf
cp server/tinode.conf server/tinode.conf.local
# Edit: db, push, media, calls, auth sections

# 5. Run
./tinode

# 6. Docker (alternative)
cd docker
docker-compose up -d
```

## 10.5 Unified Deployment Topology

```
┌─────────────┐     WSS/WSS      ┌──────────────────┐
│  Android    │◄─────────────────│   Load Balancer  │
│   APK       │   (same host)    │   (nginx/TLS)    │
└─────────────┘                  └────────┬─────────┘
                                          │
┌─────────────┐     HTTPS/WSS      ┌──────▼────────┐
│    Web      │──────────────────►│  Tinode Go      │
│   (PWA)     │   (static files)  │  Server 0.25    │
└─────────────┘                   │  + MySQL/PG     │
                                  │  + FCM Push     │
                                  │  + Media FS/S3  │
                                  └─────────────────┘
```

---

# 11. All Missing Dependencies and Credentials

## 11.1 Missing Files (Must Be Created Externally)

| File | Path | Purpose | Contains Secrets? |
|------|------|---------|-------------------|
| `google-services.json` | `tindroid-master/app/google-services.json` | Android FCM config | YES |
| `keystore.properties` | `tindroid-master/keystore.properties` | Android release signing | YES |
| `tinode.conf` | `chat-master/server/tinode.conf` | Backend runtime config | YES (DB passwords, FCM keys) |
| `firebase-init.js` (real keys) | `webapp-master/firebase-init.js` | Web Firebase config | YES |

## 11.2 Missing Toolchains

| Tool | Version | Install From |
|------|---------|--------------|
| Go | 1.26.0+ (per go.mod) | https://go.dev/dl/ |
| Java (JDK) | 17 | https://adoptium.net/ |
| Android SDK | Platform 36 | Android Studio SDK Manager |
| Gradle | 9.6.1 | Wrapper included (`gradlew`) |
| Node.js | LTS | https://nodejs.org/ |
| npm | 10+ | Bundled with Node |

## 11.3 Firebase Project Requirements

One Firebase project serving all three platforms:

| Platform | Config Needed |
|----------|---------------|
| Android | `google-services.json` (package: `co.tinode.tindroidx`) |
| Web | `firebase-init.js` (apiKey, projectId, messagingSenderId, appId, messagingVapidKey) |
| Backend | Firebase Admin SDK service account JSON |

**Do not use placeholder/demo keys in production.**

## 11.4 External npm Dependencies (Web)

Resolved via `npm install` (639 packages). Key ones:
- `react` ^19.3.0, `react-dom` ^19.3.0
- `react-intl` ^12.1.3
- `tinode-sdk` ^0.25.4
- `firebase` ^13.1.0
- `libphonenumber-js` ^1.13.13
- `qrcodejs` ^1.0.0
- `webm-duration-fix` ^1.0.4
- `webpack` ^5.110.3, `babel-loader`, `postcss`, `jest`

## 11.5 Android Dependencies (from app/build.gradle)

Key implementation dependencies (41 total):
- AndroidX: activity, appcompat, cardview, constraintlayout, coordinatorlayout, core-splashscreen, emoji2, exifinterface, fragment, lifecycle, preference, recyclerview, vectordrawable, viewpager2, work-runtime
- CameraX 1.6.2, Media3 1.11.1
- Firebase: core, crashlytics, messaging
- Google ML Kit barcode-scanning 17.3.0
- libphonenumber 9.0.40
- WebRTC com.dafruits:webrtc 123.0.0
- ICU4J 78.3, Coil 2.7.0, QR code 1.8.0
- Material Components 1.14.0, Flexbox 3.0.0

---

# 12. Integration Risks and Compatibility Issues

## 12.1 Confirmed Compatible (No Issues)

| Aspect | Status | Evidence |
|--------|--------|----------|
| Protocol version | ✅ Aligned | All at v0.25 |
| WebSocket endpoint | ✅ Match | Both clients use `/v0/channels` |
| Message models | ✅ Identical | Same field names on both clients |
| Access modes | ✅ Identical | JRWPS, JRWPSAO, JR in both |
| Topic types | ✅ Identical | me/fnd/sys/slf/grp/p2p/nch |
| Drafty rich text | ✅ Identical | Same format across all three |
| Call signaling | ✅ Compatible | Backend is media-agnostic |
| Push notifications | ✅ Compatible | FCM for both platforms |

## 12.2 Risks and Mitigations

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| 1 | **Go toolchain missing** | High | Install Go 1.26+ before backend build |
| 2 | **JDK 17 + Android SDK missing** | High | Install Java 17 and Android SDK 36 |
| 3 | **Missing `google-services.json`** | High | Generate from Firebase console (same project as Web) |
| 4 | **Missing `keystore.properties`** | Medium | Create for release builds; debug works without |
| 5 | **Missing backend `tinode.conf`** | High | Create with real DB, FCM, ICE, media config |
| 6 | **Firebase project mismatch** | High | Use one Firebase project for Android + Web + Backend |
| 7 | **Go 1.26.0 version** | Low | Verify `go 1.26.0` in go.mod is valid; may need downgrade |
| 8 | **Web `tinode-sdk` external** | Medium | `npm install` resolves; vendor copy as backup |
| 9 | **CORS/origin** | Medium | Configure CORS in backend for web origin |
| 10 | **Android minSdk 27** | Low | Drops Android < 8.1; document |
| 11 | **Web browser support** | Low | Tested on Chrome/Firefox; add Safari/Edge to CI |
| 12 | **E2E encryption missing** | Medium | Not in any codebase; plan as post-MVP feature |
| 13 | **Multi-account missing** | Medium | Not implemented; plan as post-MVP feature |

## 12.3 Known Gaps (Not Implemented in Any Codebase)

| Feature | Status | Plan |
|---------|--------|------|
| End-to-end encryption | ❌ Missing | Post-MVP roadmap |
| Multi-account | ❌ Missing | Post-MVP roadmap |
| Multi-backend switching | ❌ Missing | Post-MVP roadmap |
| Mentions/hashtags | ❌ Missing | Post-MVP roadmap |
| Link/document previews | ❌ Missing | Post-MVP roadmap |
| Video message recording | ❌ Missing | Post-MVP roadmap |
| Federation | ❌ Missing | Server-planned, not implemented |

---

# 13. Step-by-Step Development Roadmap for Google AI Studio

## Phase 0: Environment Setup (Week 1)

**Google AI Studio tasks:**
1. Upload this repository to Google AI Studio workspace
2. Install Go toolchain (1.26+), JDK 17, Android SDK 36, Node.js LTS
3. Create Firebase project (one project for all platforms)
4. Generate `google-services.json`, `firebase-init.js` (real keys), `keystore.properties`
5. Create `tinode.conf` with production values (DB, FCM, ICE, media)
6. Verify all three ZIPs and extracted folders are preserved

**Deliverable:** Working build environment with all credentials configured.

---

## Phase 1: Individual Build Verification (Week 1-2)

**Google AI Studio tasks:**
1. **Backend:** `cd chat-master && go mod download && go build -o tinode ./server`
2. **Android:** `cd tindroid-master && ./gradlew assembleDebug`
3. **Web:** `cd webapp-master && npm install && npm run build`
4. Fix any build errors without modifying source logic (only build config if needed)
5. Document all build outputs and artifacts

**Deliverable:** All three projects build independently.

---

## Phase 2: Configuration Unification (Week 2-3)

**Google AI Studio tasks:**
1. Create unified `tinode.conf` serving both Android and Web
2. Ensure Firebase project is same for Android `google-services.json`, Web `firebase-init.js`, and backend FCM
3. Align default hosts: Android `default_host_name` and Web `DEFAULT_HOST` point to same backend
4. Configure CORS for web origin on backend
5. Create unified environment variables template (`.env.example`)

**Deliverable:** Single configuration set for all three platforms.

---

## Phase 3: Repository Consolidation (Week 3-4)

**Google AI Studio tasks (structure only — NO source code changes):**
1. Create unified directory layout:
   ```
   /backend        → chat-master/server/, chat-master/tinode-db/, chat-master/docker/
   /sdk/android    → tindroid-master/tinodesdk/
   /app/android    → tindroid-master/app/
   /app/web        → webapp-master/src/, webapp-master/umd/, webapp-master/package.json
   /shared         → shared assets, i18n, docs
   ```
2. Add unified `docker-compose.yml`
3. Add unified `README.md` with build instructions
4. Add CI/CD pipeline (GitHub Actions) for all three builds
5. Keep original ZIPs untouched as reference

**Deliverable:** Unified repository structure, all builds passing in CI.

---

## Phase 4: Feature Parity & Integration Testing (Week 4-6)

**Google AI Studio tasks:**
1. Verify Android ↔ Web chat through same backend instance
2. Test push notifications on both platforms
3. Test WebRTC 1:1 calls between Android and Web
4. Test file uploads (image, video, document)
5. Test presence, typing, read receipts across platforms
6. Test authentication (anon, basic, code, token)
7. Identify and document any parity gaps

**Deliverable:** Fully working unified Android + Web application with shared backend.

---

## Phase 5: Polish & Release (Week 6-8)

**Google AI Studio tasks:**
1. Android release APK (signing with `keystore.properties`)
2. Web production build deployed to hosting
3. Backend Docker image published
4. E2E test suite
5. Documentation and runbooks
6. Release tagging (single version across all three)

**Deliverable:** Production-ready unified application.

---

## Approval Checklist (Before Starting Phase 1)

- [ ] Go toolchain installed
- [ ] JDK 17 + Android SDK 36 installed
- [ ] Firebase project created (one for all platforms)
- [ ] `google-services.json` generated for Android
- [ ] `firebase-init.js` updated with real keys for Web
- [ ] `keystore.properties` created for Android release
- [ ] `tinode.conf` created with production values
- [ ] All three ZIPs preserved: `chat-master.zip`, `tindroid-master.zip`, `webapp-master.zip`
- [ ] No source code modified in extracted folders

---

## Preservation Status (Current)

| Artifact | Location | Status |
|----------|----------|--------|
| `chat-master.zip` | `/home/user/Call-apps/` | ✅ Preserved |
| `tindroid-master.zip` | `/home/user/Call-apps/` | ✅ Preserved |
| `webapp-master.zip` | `/home/user/Call-apps/` | ✅ Preserved |
| `chat-master/` | `/home/user/Call-apps/` | ✅ Untouched (264 files) |
| `tindroid-master/` | `/home/user/Call-apps/` | ✅ Untouched (575 files) |
| `webapp-master/` | `/home/user/Call-apps/` | ✅ Untouched (439 files) |
| `GOOGLE_AI_STUDIO_HANDOFF.md` | `/home/user/Call-apps/` | ✅ New document |

---

**This document is ready for Google AI Studio development. Wait for explicit approval before beginning Phase 1 integration work.**
