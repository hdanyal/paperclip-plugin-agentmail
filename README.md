# @hdanyal-ts/paperclip-plugin-agentmail

**Turn AgentMail into Paperclip work.**  
This plugin bridges [AgentMail](https://agentmail.to) and a [Paperclip](https://github.com/paperclipai/paperclip) host: inbound mail becomes **issues**, files become **documents**, and your agents get **REST tools** to read threads and reply—without leaving the control plane.

It is an **independent third-party** plugin (not an official `paperclipai` product). Install from **your** git repo or a **local path**; publishing to npm is optional. For third-party install patterns, see [THIRD_PARTY_PLUGINS.md](https://github.com/paperclipai/paperclip/blob/master/doc/plugins/THIRD_PARTY_PLUGINS.md) in the upstream Paperclip repo (or the same path in your monorepo fork).

---

### At a glance

| | |
|---:|---|
| **Source / issues** | [github.com/hdanyal-ts/paperclip-plugin-agentmail](https://github.com/hdanyal-ts/paperclip-plugin-agentmail) |
| **Plugin id** (webhooks, tools) | `hdanyal-ts.paperclip-plugin-agentmail` |
| **Requires** | Paperclip-compatible host, **Node ≥ 20**, `@paperclipai/plugin-sdk` (from npm when installed) |

### How mail becomes work

```
AgentMail (inbox)  --webhook and/or websocket-->  plugin worker
                                                      |
                                                      v
                                              Paperclip issues
                                              + documents + comments
                                                      |
                                                      v
                                              Agent REST tools (reply, thread, etc.)
```

Live events drive the pipeline; if Paperclip was **down**, **unread catch-up** (see below) replays missed mail through the **same** ingest path—so you are not blind after a restart.

---

## Install from git (recommended)

`POST /api/plugins/install` accepts any **`npm` install spec** the server can resolve, including a **tagged** git URL:

```bash
curl -X POST "https://<your-board-host>/api/plugins/install" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <board-or-operator-token>" \
  -d '{"packageName":"git+https://github.com/hdanyal-ts/paperclip-plugin-agentmail.git#v0.3.0"}'
```

Private repos need the Paperclip **server** to reach `git` (SSH, credential helper, or HTTPS token). Do not commit tokens into shared config.

### Optional: npm registry

If you publish `@hdanyal-ts/paperclip-plugin-agentmail` to a registry, use e.g. `"packageName":"@hdanyal-ts/paperclip-plugin-agentmail","version":"0.3.0"`. This repo defaults to **`"private": true`** (no public npm by default).

### Install from a local path (dev)

Build first (`npm run build` → `dist/`).

```bash
curl -X POST http://127.0.0.1:3100/api/plugins/install \
  -H "Content-Type: application/json" \
  -d '{"packageName":"/absolute/path/to/paperclip-plugin-agentmail","isLocalPath":true}'
```

From a **Paperclip monorepo** (this package lives at `packages/plugins/plugin-agentmail-paperclip`):

```bash
pnpm --filter @hdanyal-ts/paperclip-plugin-agentmail build
```

Developer notes: [CONTRIBUTING.md](CONTRIBUTING.md), standalone export: [STANDALONE.md](STANDALONE.md).

---

## Upgrading from older plugin ids

If you used **`paperclipai.agentmail-paperclip`** or another old manifest id: **remove** the old instance in Plugin Manager, **reinstall**, re-apply settings, and point AgentMail at the current webhook:

`https://<your-board-host>/api/plugins/hdanyal-ts.paperclip-plugin-agentmail/webhooks/agentmail-inbound`

---

## What you get (v1)

- **Delivery you can mix:** **`message.received`** over **HTTPS webhooks** (Svix-signed) and/or an **outbound WebSocket** to AgentMail (`wss://…/v0` + per-inbox `am_…` key). Same `issues` + `issue.documents` pipeline; **`message_id`** dedup if you run **both** during a migration.
- **One issue per thread:** `agentmail.thread` maps `inbox_id:thread_id` to a single issue. Replies **comment**, bump the issue to **`todo`**, and re-run attachment ingest (so a late PDF still lands on the right card). Odd `thread_id` on a reply is merged using **`in_reply_to` / `references`**, or **`GET /messages/{id}`** when the webhook is thin. Stale thread rows fall back sensibly before opening a duplicate issue.
- **Attachments that actually show up:** Payloads are **merged** with `GET /messages/{id}`; ids accept `attachment_id`, `id`, or `attachmentId`. If both sides are empty briefly, the worker **retries** once after a short delay.
- **Dedup before noise:** `agentmail.message` tracks AgentMail `message_id` so replays do not duplicate work.
- **Routing:** `mailboxes[]` / `assignments[]`, plus optional **`recipientMatch`** when humans share one inbox.
- **Operator context:** `handlingInstructions` fold into the issue description.
- **Agent tools (REST):** `agentmail_get_handling_context`, `agentmail_list_messages`, `agentmail_get_message`, **`agentmail_get_thread`**, `agentmail_send_message`, `agentmail_reply_to_message`. Tool names are **`hdanyal-ts.paperclip-plugin-agentmail:<bare_name>`**. Inbound issues carry an **`### AgentMail`** block. For prompt examples, see `skills/paperclip/SKILL.md` in a full Paperclip checkout.
- **Spam guard:** optional **skip issue creation** when labels match `attachmentPolicy.skipIfLabels` (default includes spam-like labels).
- **Unread catch-up** and **blog dedupe** (optional)—the next sections go deep.

---

## Unread catch-up and wakeup sync

If Paperclip or the worker **is not running**, webhooks and WS frames do not run—but **mail still arrives** in AgentMail, often as **`unread`**. This plugin **catches up** by listing unread mail per mailbox and feeding it through the **same ingest code** as live traffic (internal source `catchup`, ids like `catchup:{message_id}`). Existing **`message_id`** dedup keeps reruns safe.

**When it runs**

| Trigger | What happens |
|--------|----------------|
| **Worker startup** | **Sync unread on startup** (default **on**) kicks a pass after the worker is up. |
| **Config saved** | With startup sync on, saving settings triggers a pass (useful after mailbox edits). |
| **Scheduled job** | Job key **`unread_sync`** (“Unread mail reconciliation”) runs on a **once-per-minute** host schedule. **`unreadSyncIntervalSeconds`** (default **300**) throttles how often a **scheduled** run does real work after the last one. Set to **0** to **disable that throttle** (job can work every tick, still capped per mailbox and by the lock)—**not** a no-op. |
| **Manual** | In **Plugin Manager → this plugin → Settings**, **Sync unread now** runs immediately and returns JSON; the UI can show the last **sync summary**. |

**Settings** (also in the **Catch-up** card in the UI; full schema: `instanceConfigSchema` in `src/manifest.ts`):

- **`unreadSyncOnStartup`** — Run on worker start and after config change when `true` (default `true`).
- **`unreadSyncIntervalSeconds`** — Minimum gap between **scheduled** runs that process work (`0`–`86400`, default `300`). Does not slow **startup** or **manual** sync.
- **`unreadSyncMaxPerRun`** — Max messages **per mailbox** per run (`1`–`500`, default `50`) so one huge backlog cannot wedged the worker.

**Safety + visibility**

- A **distributed lock** (instance state) prevents overlapping catch-up; if the lock is held, another run may log and exit early.
- Per-run **stats** feed the “last unread sync summary” for the UI.

**Marking read:** after a successful catch-up ingest, the worker can strip AgentMail’s **`unread`** label, same rules as live mail. If **blog dedupe** is on, **`blogDedupe.markReadOnDuplicate`** decides whether duplicates should still clear `unread` (default: clear).

---

## Blog dedupe (optional)

Optional **fingerprinting** for blog-style mail: title+body (or PDF/DOCX text), optional **sitemap** URL checks, optional small **visibility issues** when duplicates are detected. Same pipeline as webhooks—**including catch-up**. Fields: **`blogDedupe.enabled`**, **`sitemapUrls`**, **`markReadOnDuplicate`**, **`createIssueForDuplicate`**, plus extraction limits—see the settings form and `src/manifest.ts`.

---

## Security

- Treat **`am_…`** inbox keys and **`whsec_…`** webhook secrets as **secrets**; use Paperclip **secret references** for webhook material, not plaintext in repos.
- Prefer **HTTPS** for the board host; lock down who may call **`/api/plugins/install`**.

---

## Event delivery: webhook vs WebSocket

| | Webhook (default) | WebSocket |
|---|-------------------|-----------|
| **Inbound URL** | Public `POST` on your Paperclip host | None — worker dials **out** to AgentMail |
| **Trust** | Svix signature (`whsec_…`) | TLS + inbox key in the WS URL (no Svix on frames) |
| **When to use** | Common in production | Dev, tight egress, or “no public URL” setups — [AgentMail WebSockets](https://docs.agentmail.to/websockets/quickstart.md) |

In settings, **Event delivery** is `webhook`, `websocket`, or `both` (duplicates dedupe on `message_id`).

---

## Configure (checklist)

1. For **webhook** or **both**: store AgentMail’s **`whsec_…`** as a Paperclip **secret**; put its **reference** in **Webhook secret ref**. Put each inbox **`am_…`** key in the mailbox row (plugin config).
2. **Plugin Manager → AgentMail → Settings → Core:** **Company ID**, **Event delivery**, webhook ref, and optional **WebSocket URL**, **default project**, **API base**, **title prefix**.
3. **Mailboxes:** inboxes, assignments, optional **Only when addressed to** for shared inboxes.
4. **Save configuration.**
5. In **AgentMail**, set the **Inbound URL** from the plugin’s Webhook card:  
   `https://<your-board-host>/api/plugins/hdanyal-ts.paperclip-plugin-agentmail/webhooks/agentmail-inbound`  
   (If your host uses an install-specific path, use the URL the UI shows.)

---

## HTTP semantics (Paperclip host)

Happy path (worker finishes without throwing) → **200**. Verification failures **throw** → host may return **502** (provider may retry).

---

## References

- [Paperclip plugin spec](https://github.com/paperclipai/paperclip/blob/master/doc/plugins/PLUGIN_SPEC.md)
- [AgentMail webhook setup](https://docs.agentmail.to/webhook-setup.mdx)
- [`message.received`](https://docs.agentmail.to/api-reference/webhooks/events/message-received.mdx)

---

## Tests

```bash
npm test
```

Paperclip monorepo:

```bash
pnpm --filter @hdanyal-ts/paperclip-plugin-agentmail test
```

## Changelog

See [CHANGELOG.md](CHANGELOG.md).
