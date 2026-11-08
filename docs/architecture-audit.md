# Architecture audit: shared core and web-first (T5)

Date: 2026-11-08. Read-only audit of the workspace submodules.

## What exists

| Layer | Where | Role |
| --- | --- | --- |
| Telegram engine | `vasyaapp/src-tauri/vasya-core` (Tauri-free Rust, grammers) | MTProto, sessions, media, STT, translation |
| API server | `vasyaapp/src-tauri/vasya-server` | REST `/api/v1`, GraphQL, SSE `/events`, agent keys, audit log, MCP (`vasya-mcp`) |
| Tauri app | `vasyaapp` (React 19 + Rust shell) | Desktop UI, calls the engine via IPC |
| Web UI | same React code, `src/transport/http.ts` | Talks to a remote `vasya-server` over REST/SSE |
| Native apps | `vasyaapp-native` (GPUI, Iced) | Rust UIs on a copy of the engine and server |
| Backend | `vasya-backend` (axum + Postgres) | Accounts, JWT, metadata sync |

The React app already has a `TelegramTransport` interface with two
implementations (`tauri.ts`, `http.ts`). So Tauri and web share one frontend and
one command contract. That part of "shared logic" is real.

## The problem: the engine is copied, not shared

`vasyaapp-native/crates/vasya-core` and `crates/vasya-server` are copies of the
Tauri repo's crates, taken at upstream commit `d8e1670` (see `UPSTREAM.md`), then
ported by hand for later fixes. `diff -rq` shows ~10 differing files in
`vasya-core` (client_manager, updates, encrypted_session, master_key, ...) and
differing files in `vasya-server`. Bug fixes (such as the channel-id panic fixed in
this session) must now be made twice, and the copies will keep drifting.

Also: `vasya-backend` exists twice (standalone repo and `vasyaapp-native/crates`).

## Proposal: one core, three shells, web first

1. Extract `vasya-core` + `vasya-server` into one repo (or a Cargo workspace
   published as git dependencies). Tauri and native depend on it by tag. Delete
   the copies. First step: reconcile the diffs (merge the native fixes back).
2. Make `vasya-server` the product boundary. The server API (REST + SSE, already
   byte-compatible with the Tauri IPC results) is the only contract. Web,
   Tauri (embedded or remote), native and agents all speak it.
3. Web is the primary client. It needs a deployable server (the Dockerfile and
   `deploy/` exist) plus cookie + CSRF auth (server foundation is in; frontend half
   marked M-4). Desktop apps become thin shells: embedded server in-process,
   plus OS integration (notifications, keychain, tray).
4. Add the channel abstraction in the core before WhatsApp/email (T4): a
   `Channel` trait (list chats, history, send, updates stream, auth flow) with
   Telegram as the first implementation. Server DTOs gain a `channel` field.

## Risks

- Web-first with Telegram MTProto means the server holds user sessions: needs
  encryption at rest (today: ChaCha20-Poly1305 with OS keychain key; a server needs
  a KMS or env-held key), per-user isolation, and a clear self-host story.
- Local STT (Whisper) and VoIP sidecars are desktop-only; web uses cloud STT or
  server-side sidecars.
- Merging the two engine copies is the real work; estimate it before promising dates.

## Next

- Do the diff reconcile (small, high value): one session, no new features.
- T4 research builds on step 4.
- T6 landing diagram can already show: core -> server API -> web / Tauri / native /
  agents.

## Merge progress (2026-11-08)

- `vasya-core`: after `rustfmt` on both sides only two files really differed. The one
  real difference (logout-safe session removal, `disable_persistence`) is now in the
  Tauri repo (vasyaapp `ba9ba59`). Behaviorally the two core copies are now equal.
  The native copy lacks the vendored `grammers-session` channel-id fix, so it still
  has the `unreachable!()` panic on some channel ids.
- `vasya-server`: native has real extra hardening/features (about 270 changed lines in
  8 files): media header validation (`x-topic-id`, `x-voice`), `native:resync` SSE
  event after a lagged consumer, no token printed to stdout, translation scope.
  Needs a careful merge, not a copy.
- Still open: where the single source lives. Native depending on a path inside the
  vasyaapp repo is fragile; a separate repo (or workspace) is cleaner.
