# Pluggable messenger providers (T7)

Goal: Telegram, Email and WhatsApp are separate modules behind one contract.
You can add a provider, or switch one off, without touching the others.

## What meta-sovereign does (inspiration, JS, read-only study)

Repo: link-foundation/meta-sovereign (`js/src/sources/`). Ideas worth taking:

- **A registry keyed by name.** `sourceRegistry = { email, telegram, whatsapp, vk, ... }`,
  with `getSource(name)` that fails with the list of known names. Adding a network
  means adding one entry.
- **One normalized message shape, namespaced by source.** `msg:<source>:<externalId>`,
  plus `sender:<source>:...` and `chat:<source>:...`. The shape never changes per
  network; optional fields (edited, deleted, attachments, reactions, delivery) are
  stored only when the network reports them.
- **Two halves per adapter.** `parseArchive()` (import an export file) and optional
  `live*` methods (real API). A network works at "import only" level first.
- **Honest per-network paths.** WhatsApp: chat-export import plus the official Cloud
  API (webhook-driven, only approved templates outside the 24 h window). Email: IMAP,
  POP3, SMTP, and HTTP APIs (JMAP, Gmail, Microsoft Graph). Browser-capable APIs run
  in the SPA, raw protocols need the local server.
- **A shared pull runner.** `ready()` (connected?), `run()`, `intervalMs`; manual fetch
  has a minimum gap, background runs back off exponentially after failures, cursors
  are saved only after the store write succeeds. We should copy this behaviour for
  polling providers (email, WhatsApp export folders).
- **Their own Rust server is the weak part** (in-memory only, no adapters), which
  tells us the Rust side must own the provider layer from day one.

Check their LICENSE before copying anything; take the design, not the code.

## Proposed design for vasya

### Crates

```
vasya-channel            traits + normalized types (no grammers, no network deps)
vasya-channel-telegram   today's vasya-core telegram/ code, implements the trait
vasya-channel-email      IMAP + SMTP (async-imap, lettre, mail-parser)
vasya-channel-whatsapp   Cloud API client + export importer (+ optional unofficial)
vasya-core               host: ProviderRegistry, shared services (STT, translation, media)
vasya-server             routes call the registry, never a concrete provider
```

Each provider crate is an optional Cargo feature of `vasya-server` and the apps
(`--no-default-features --features telegram,email`). Runtime on/off lives in config
(`providers.<id>.enabled`), so a build can contain WhatsApp while a user keeps it off.

### Trait sketch

```rust
#[async_trait]
pub trait ChannelProvider: Send + Sync {
    fn id(&self) -> &'static str;                 // "telegram" | "email" | "whatsapp"
    fn capabilities(&self) -> Capabilities;       // bitflags, see below
    async fn begin_auth(&self, req: AuthStart) -> Result<AuthStep>;   // code/2FA, password, OAuth, QR
    async fn submit_auth(&self, step: AuthInput) -> Result<AuthStep>;
    async fn list_chats(&self, acct: &AccountId, page: Page) -> Result<Vec<Chat>>;
    async fn history(&self, acct: &AccountId, chat: &ChatId, page: Page) -> Result<Vec<Message>>;
    async fn send(&self, acct: &AccountId, chat: &ChatId, msg: Outgoing) -> Result<Message>;
    fn events(&self) -> BoxStream<'static, ChannelEvent>;   // live updates, same bus as SSE
    async fn shutdown(&self);
}
```

`Capabilities` (bitflags) drives the UI and API instead of `if provider == ...`:
`HISTORY, SEND, EDIT, DELETE, REACTIONS, VOICE, TYPING, READ_RECEIPTS, FOLDERS,
CALLS, LIVE_UPDATES, THREADS`. Email has THREADS and no TYPING; Telegram has
nearly all; WhatsApp Cloud API has SEND and LIVE_UPDATES (webhook) only.

### Identity and compatibility

- IDs become `provider:account:chat` strings. The DTOs gain a `provider` field that
  defaults to `"telegram"`, so existing clients and the Tauri IPC keep working.
- Existing Telegram routes stay as aliases. New generic routes:
  `GET /api/v1/providers` (id, enabled, capabilities, accounts),
  `POST /api/v1/providers/{id}/enable|disable`, and the same chat/message routes
  under `/api/v1/{provider}/...`. A disabled provider returns `404 provider_disabled`.
- SSE/GraphQL events carry `provider`; the frontend filters by enabled providers.

### Provider notes

| Provider | Path | Risk |
| --- | --- | --- |
| Telegram | existing grammers code, moved behind the trait | none new |
| Email | IMAP (IDLE for live, poll fallback) + SMTP; later Gmail/Graph OAuth, JMAP | OAuth app review for Gmail; app passwords for the rest |
| WhatsApp, tier 1 | chat export import (offline, safe) | none |
| WhatsApp, tier 2 | official Cloud API: needs a Business number, public webhook endpoint, templates after 24 h | server must be reachable from Meta; fits web-first (T5) |
| WhatsApp, tier 3 | personal account via an unofficial multi-device library | violates WhatsApp ToS, account bans possible: opt-in feature flag with a warning, never default |

**Decision (2026-11-08): personal account (tier 3) is wanted too.** Build order stays
tier 1, tier 2, then tier 3 behind a Cargo feature `whatsapp-personal`, off by default,
with an in-app warning about ToS and ban risk before the user links an account.

### Order of work

1. Reconcile the two `vasya-core` copies (T5), otherwise the trait is built twice.
2. Create `vasya-channel`; wrap the current Telegram code behind it with no behaviour
   change; add `provider` to DTOs and the providers routes. Ship with only Telegram.
   **Done in vasya-engine v0.2.0.** Trait methods take a `Caller` (the server checks
   account ownership); enable/disable is admin-only and saved in `providers.json`;
   Telegram account and login routes answer `404 provider_disabled` when it is off.
   Not yet: generic `/api/v1/{provider}/...` chat routes, `provider` in SSE events.
3. ~~Email provider (IMAP/SMTP)~~ done in vasya-engine v0.3.0: TLS-only IMAP/SMTP, accounts at `/api/v1/email/accounts` (human session only, encrypted per user), generic `/api/v1/providers/{id}/accounts/{acc}/chats...` routes. Not yet: `provider` in SSE events, settings UI.
4. WhatsApp import, then Cloud API.
5. Settings UI: provider list with enable/disable and capability-driven views.

### Open questions for you

- ~~Personal WhatsApp account?~~ Yes, answered: opt-in tier 3.
- ~~Email scope?~~ Answered: IMAP/SMTP only at first (Gmail OAuth later, if ever).
- ~~Disabled provider and stored sessions?~~ Answered: keep them (encrypted). Removing data is a separate UI action.
