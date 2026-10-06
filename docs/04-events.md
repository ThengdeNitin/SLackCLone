# Event catalogue

Every topic, every event, who produces it and who consumes it. This is the contract between
services — changing something here breaks a service that is not in the same pull request, so it
is the one document worth keeping exactly current.

## Topics

| Topic | Produced by | Consumed by | Why it is its own topic |
|---|---|---|---|
| `message.created` | message-service | workspace-service, realtime-service, search-service, notification-service | By far the highest volume. Separate so a consumer that only cares about new messages does not deserialise and discard every edit and deletion in the product |
| `message.updated` | message-service | search-service | Only search cares. Realtime pushes edits over the socket directly from the create path |
| `message.deleted` | message-service | search-service | A deleted message that stays searchable is a genuine data-leak path |
| `channel.events` | workspace-service | notification-service | Workspace and channel lifecycle share one topic: a handful of events per day per workspace, and every consumer wants most of them. The split above is justified by volume; applying it here would be ceremony |
| `user.events` | auth-service | workspace-service, notification-service | User lifecycle. This is how the other services find out an account was deactivated, given there is no foreign key to lean on |
| `notification.dispatch` | workspace-service | notification-service | Things that exist only to become a notification, so notification-service does not have to subscribe to a domain topic to find one event in it |

All topics: 3 partitions, replication factor 1 locally (3 in production).

## Headers, on every event

| Header | Purpose |
|---|---|
| `X-Event-Id` | The deduplication key. Stable across redeliveries — **not** a Kafka offset, because a rebalance can change the partition assignment and the same event can be redelivered at a different offset |
| `X-Event-Type` | Lets a consumer filter without deserialising the body |
| `X-Trace-Id` | Carries the trace across the broker. Without it, async work appears as unattributable fragments and "why did this take eight seconds to show up in search?" is unanswerable |
| `X-Occurred-At` | When the producer staged it, which is not when the consumer sees it |

The event type is in the header **and** in the payload, deliberately. The header lets a consumer
filter cheaply; the body field means the event is self-describing if it is ever dumped to a file
or replayed by hand.

## The message key is the aggregate id

Always. Kafka only guarantees order *within a partition*, and the partition is chosen by hashing
the key — so keying on the aggregate id means every event about one channel lands on the same
partition and arrives in the order it happened.

Without a key, `channel.renamed` can be processed before `channel.created`, and the consumer has
to cope with an event about a channel it has never heard of.

**What this does not give you** is ordering *between* aggregates. A workspace and a channel
inside it have different ids, so they are on independent partitions and the broker guarantees
nothing about their relative order. A consumer that only works in one order is wrong, not merely
fragile — notification-service had exactly that bug (see
[02 — Low-level design](02-low-level-design.md)).

---

## Event payloads

### `message.created`

```json
{
  "eventType": "message.created",
  "messageId": "...", "channelId": "...", "workspaceId": "...", "userId": "...",
  "text": "ship it",
  "mentions": ["user-id", "..."],
  "mentionsEveryone": false,
  "threadRootId": null,
  "memberIds": ["...", "..."],
  "createdAt": "2026-10-01T07:00:00Z"
}
```

Four things in the payload are decisions rather than data:

**`mentions` is the parsed result, not the raw text.** Extracted once at write time, so every
consumer that notifies or counts uses the same list. Two parsers for one rule is how a mention
comes to highlight in the UI without notifying anyone.

**`mentionsEveryone` is separate** from the mention list, so `@channel` does not require a
consumer to expand it against a membership table it may not have.

**`memberIds` is a snapshot** of who was in the channel when the message was sent. It rides along
so the notification consumer does not call workspace-service for something the producer already
had in hand. It being a snapshot is correct: someone who joins a second later was not in the
channel when it was sent and should not be notified about it.

**`text` is included**, which is the one payload field worth questioning. It means message
content crosses the broker. The alternative — consumers fetching the message by id — adds a
synchronous call from four consumers to message-service on every message, which is strictly
worse for both availability and load. The topic is internal with short retention, and in
production is encrypted at rest.

Consumed by:

| Service | What it does |
|---|---|
| workspace-service | Advances `lastMessageAt`, increments unread and mention counters. **The only consumer that needs a dedupe ledger**, because an increment cannot be made repeatable |
| realtime-service | Pushes to connected clients. Per-instance consumer group |
| search-service | Indexes the document, keyed by message id so a redelivery is a no-op |
| notification-service | Decides whether to email each mentioned user |

### `message.updated`

Same shape plus `previousText`. Consumed only by search-service, which re-indexes the full
document — making it idempotent, since re-indexing the same document to the same id changes
nothing.

### `message.deleted`

```json
{
  "eventType": "message.deleted",
  "messageId": "...", "channelId": "...", "workspaceId": "...", "userId": "...",
  "deletedBy": "...",
  "createdAt": "..."
}
```

Carries **no content**, because the content is what is being removed. search-service tombstones
the document — `isDeleted: true`, empty text — with `doc_as_upsert`, so a delete arriving before
the create it refers to is harmless.

### `channel.events`

| Event type | Payload | Notes |
|---|---|---|
| `workspace.created` | `workspaceId`, `slug`, `name`, `createdBy`, `defaultChannelId` | Every workspace is born with a `#general` |
| `workspace.updated` | `workspaceId`, `name` | A rename |
| `channel.created` | `channelId`, `workspaceId`, `name`, `type`, `createdBy`, `memberIds` | |
| `channel.updated` | `channelId`, `workspaceId`, `name`, `topic`, `purpose` | |
| `channel.archived` | `channelId`, `workspaceId`, `archivedBy` | No name: an archive event has no reason to restate one, which is why the consumer's upsert guards against overwriting a known name with an empty string |
| `channel.member_added` | `channelId`, `userId`, `addedBy` | |
| `channel.member_removed` | `channelId`, `userId`, `removedBy` | |
| `workspace.member_joined` | `workspaceId`, `userId`, `role` | |
| `workspace.member_role_changed` | `workspaceId`, `userId`, `oldRole`, `newRole` | |

### `user.events`

| Event type | Payload | Notes |
|---|---|---|
| `user.registered` | `userId`, `email`, `username`, `displayName`, `timezone` | Carries the email, because notification-service has no user table and needs an address to send to |
| `user.profile_updated` | `userId`, `displayName`, `timezone` | Deliberately **no email**. An event should carry what its consumers use, not everything the aggregate holds — and this one has no consumer that needs an address. It is also why the consumer's upsert uses `COALESCE`: a blanket overwrite would null the stored address and silently destroy the ability to email that person |
| `user.deactivated` | `userId`, `deactivatedAt` | **This event is the cascade.** There is no `ON DELETE` across schemas, so nothing else removes a deleted account from its workspaces and channels |

Consumed by:

| Service | What it does |
|---|---|
| workspace-service | Removes the account from every workspace and channel. Needs no dedupe ledger: "mark these memberships inactive" is repeatable by construction |
| notification-service | Maintains the contacts read model; clears the address on deactivation, because leaving a working address downstream means "delete my account" did not delete the thing most people mean by it |

### `notification.dispatch`

| Event type | Payload | Notes |
|---|---|---|
| `workspace.invited` | `invitationId`, `workspaceId`, `workspaceName`, `workspaceSlug`, `email`, `role`, `invitedBy`, `token`, `expiresAt` | The **plaintext token** crosses the broker, which is a real consideration rather than an oversight: notification-service needs it to build the link, and the alternative is the notifier calling back for a credential. The topic is internal, short-retention, encrypted at rest in production. It is also why the token is absent from the API response and from every log line |

---

## Delivery semantics, precisely

**At-least-once. Not exactly-once.** The outbox poller can die after Kafka acknowledges but
before the row is marked published, so the event is sent again. There is no broker setting, no
transaction mode and no library that makes this exactly-once end to end — at-least-once plus
idempotent consumers is the real answer.

Which puts the burden on consumers, and the right order to reach for solutions is:

1. **Make the handler naturally repeatable.** "Set this timestamp if newer", "index this
   document by id", "mark this membership inactive". No extra state to store, index or clean up.
   Most handlers in this system are here.
2. **Record the event id.** Only for operations that genuinely cannot be repeated: incrementing
   a counter, sending an email. Two handlers in the system need this.

Reaching for (2) first is a common instinct and it is the expensive answer — a dedupe table is a
table to purge, an index to maintain, and a transaction boundary to get right.

## Producer settings worth knowing

Shared by both producers via `OutboxProducerProperties`, because almost every value was arrived
at by something breaking:

| Setting | Value | Why |
|---|---|---|
| `acks` | `all` | The default (`1`) acknowledges as soon as the leader has written, so a leader crash before replication loses a message the producer was told had succeeded |
| `enable.idempotence` | `true` | A retry after a network timeout can otherwise write the message twice. Note this is per-producer-session dedupe, not end-to-end exactly-once |
| `compression.type` | `gzip` | **Not** snappy or zstd. Both ship as JNI wrappers around a glibc-linked native library, and the runtime image is Alpine (musl). The failure is at first *use*, not at startup: the service comes up healthy and then every publish fails, so the outbox quietly fills while the broker is fine |
| `max.block.ms` | `5000` | How long `send()` blocks when the broker is unreachable. The 60s default would hold the poller's transaction open for a minute per event during an outage |
| `delivery.timeout.ms` | `60000` | Must be `>= linger.ms + request.timeout.ms`. Violate it and the producer throws `ConfigException` when it is *constructed* — so every publish fails with a message about timeout arithmetic while the broker is perfectly healthy. This cost real debugging time |

## Consumer group strategy

The single most important table in this document, because getting it backwards produces bugs
that do not reproduce on one replica.

| Consumer | Group id | Strategy | Why |
|---|---|---|---|
| workspace-service | `workspace-service.channel-activity` | **Shared** | Builds durable state. Each event must be applied once in total |
| search-service | `search-service.indexer` | **Shared** | Same. A per-instance group would index every message once per replica |
| notification-service | `notification-service.dispatcher` | **Shared** | Sends email — a side effect outside itself. A per-instance group would send one email per container |
| realtime-service | `realtime-service.{instanceId}` | **Per-instance** | The fan-out mechanism. A client's socket lives on exactly one pod, so every instance needs every message. With a shared group a message reaches one instance and the clients on the others never see it |

And `auto.offset.reset`:

| Consumer | Value | Why |
|---|---|---|
| search-service | `earliest` | A new deployment backfills the index from whatever Kafka retains, rather than starting empty and appearing to have lost every old message |
| workspace-service | `earliest` | Same reasoning for counters |
| realtime-service | `latest` | A realtime push about a message from an hour ago is noise |
| notification-service | `latest` | The opposite of search, and the difference is the point: replaying history here means emailing everyone about every mention of the last fortnight on first boot |

## Adding an event

1. Add the type constant to `KafkaTopics.Types` in common-lib, and the topic if it is new.
2. If the topic is new, add it to the `kafka-init` job in `docker-compose.yml` — auto-creation is
   disabled, deliberately, so a typo in a topic name is an error rather than a new topic nobody
   consumes.
3. Produce it through `OutboxService.record(...)` **inside** the transaction that makes the
   change. It throws if there is no active transaction, because a committed event describing a
   change that rolled back is the exact bug the outbox exists to prevent.
4. Document it here, including who consumes it.
5. Decide explicitly whether the consumer is naturally idempotent. If it is not, say so in the
   handler and add the ledger — do not leave it as something a reader has to work out.
