# Low-level design

What is inside each service, and the handful of mechanics that are easy to get wrong. Read
[01 — High-level design](01-high-level-design.md) first.

This document is organised around *the things that bit*. A tour of every class would restate the
code; what follows is the reasoning that is not recoverable from reading it.

---

## auth-service

### Refresh token rotation with reuse detection

The mechanic that makes a long-lived session safe.

Access tokens last 15 minutes and are stateless JWTs — fast to verify, impossible to revoke.
Refresh tokens last 7 days, are stored, and are **rotated on every use**: redeeming one
immediately invalidates it and issues a new one.

The important part is what happens when an already-redeemed token is presented again. That means
one of two things — a client raced itself, or somebody stole a token — and the two are
indistinguishable from the server. So the entire token chain is revoked, which logs the real
user out and the thief with them.

```
         issue T1
            │
      use T1 ──> issue T2, mark T1 used
            │
      use T1 again  ──>  401, and revoke the whole chain
```

Two implementation details that both caused bugs:

**The revocation must survive the throw.** The handler detects reuse, revokes the chain, then
throws a 401. Inside one transaction, the throw rolls back the revocation — so the attack is
detected, reported, and nothing is actually revoked. The fix is a separate bean with
`Propagation.REQUIRES_NEW`, which commits in its own transaction before the exception
propagates.

It has to be a *separate bean*: Spring's `@Transactional` is implemented with a proxy, so calling
`this.revoke()` inside the same class bypasses the annotation entirely and the propagation
setting does nothing. This is the single most common Spring transaction mistake, and it fails
silently.

The same pattern appears in the failed-login counter for exactly the same reason — a lockout
counter that is rolled back by the 401 it causes never locks anything out. Both are in
`LoginStateService` and `TokenRevocationService`.

**Refresh tokens are opaque and hashed, not JWTs.** 32 random bytes, stored as SHA-256. A JWT
refresh token cannot be revoked without a store, which defeats the point of having two token
types; and hashing means a database dump does not hand over live sessions.

### Login does not reveal whether an account exists

Every failure returns the same `INVALID_CREDENTIALS` — same status, same body, same code.
Distinguishing "no such user" from "wrong password" turns the login endpoint into an account
enumeration oracle: an attacker submits a list of addresses and learns which are registered,
which is exactly what they need before a credential-stuffing run.

The remaining leak is timing: a missing user returns in microseconds, a real user with a wrong
password costs a ~100ms BCrypt verification. Closed by verifying against a dummy hash when the
user does not exist, so both paths cost the same.

### The `inet` column

`RefreshToken.ipAddress` maps to Postgres' `inet` type. Hibernate 6 sends a Java `String` as
`varchar`, and Postgres refuses to compare `varchar` to `inet` — SQLState 42804, on insert, with
a message about operator types that does not mention the entity.

The fix is `@ColumnTransformer(write = "?::inet")`. Worth recording because the obvious
alternative, `@JdbcTypeCode(SqlTypes.OTHER)`, is wrong: Hibernate 6 maps `OTHER` to a binary
type and the error changes rather than going away.

---

## workspace-service

### The internal access endpoint

`GET /internal/v1/channels/{channelId}/access/{userId}` returns `{allowed, writable, type,
memberIds}`. message-service and search-service call it before every read or write.

Three decisions in it:

**It is under `/internal`, which the gateway does not route.** A browser cannot reach it even
though it shares a port with the public API. That is a perimeter control, not authorisation —
the JWT filter still runs on it.

**The caller forwards the end user's own token.** The tempting alternative is for
message-service to send a trusted `X-User-Id` header, which makes any process that can reach the
port able to impersonate any user. Fine until a misconfigured ingress, a debug port, or another
pod in the namespace makes that possible. Verifying a token is one HMAC and no network call.

**`memberIds` rides along with the answer** so the caller can fan an event out without a second
request — and is returned **empty** when access is denied, because the membership of a private
channel is itself information.

### Direct messages and the participants key

Two people clicking each other's name simultaneously must not produce two DM channels each
holding half the conversation.

The key is `SHA-256(sorted participant ids)` with a unique constraint. Sorted, so Alice→Bob and
Bob→Alice hash identically. Unique, because the lookup-then-create is two operations and both
requests can miss the lookup — only the constraint can arbitrate between two processes.

The creator uses `saveAndFlush` on the key row *before* writing memberships, so the constraint
fires before any dependent rows exist. And it runs in `REQUIRES_NEW`, so the losing request can
catch the violation, re-read the winner's channel, and return it as a success.

A group DM is a different key, therefore a different channel. Adding someone to a conversation
creates a new one rather than retroactively granting access to existing history — which is both
what people expect and the safe choice.

### Unread counters, and the one place that needs a dedupe ledger

`channel_members.unread_count` and `unread_mention_count` are maintained by the `message.created`
consumer. They are stored rather than computed because a sidebar of forty channels rendered by
counting messages newer than each watermark is forty queries, on every page load, for every user.

This is the only handler in the system that needs deduplication, and the contrast is the
lesson:

| Handler | Repeatable? | Needs a ledger? |
|---|---|---|
| "set `lastMessageAt` to X if X is newer" | Yes, by construction | No |
| "mark these memberships inactive" | Yes, by construction | No |
| "index this document by message id" | Yes, by construction | No |
| `unread_count = unread_count + 1` | **No** | **Yes** |
| "send this email" | **No** | **Yes** |

Reach for idempotency first. There is no way to phrase an increment that survives being applied
twice, so the event id is recorded in `workspace.processed_events` and claimed with
`INSERT … ON CONFLICT DO NOTHING` inside the handler's own transaction — so the claim and the
counters commit together, and a crash between them cannot mark an event done that was never
applied.

`ON CONFLICT` rather than catching `DuplicateKeyException`: in Postgres a constraint violation
marks the transaction rollback-only, so every statement after it fails with "current transaction
is aborted" even if the exception was caught.

### Mute means mute

The unread increment skips muted members; the mention increment does not. Muting a channel
should quieten it, not silently swallow a request addressed to you by name. Incrementing and
hiding it in the UI would be worse than either — the badge would still light up on the user's
other devices.

---

## message-service

### The embedded outbox

The message and the event that announces it are **fields of the same document**:

```json
{
  "_id": "...",
  "channelId": "...",
  "text": "ship it",
  "pendingEvents": [
    { "eventId": "...", "topic": "message.created", "payload": { ... } }
  ]
}
```

A poller finds documents where `pendingEvents.0` exists, publishes, and clears the array.

This exists because the production Mongo is a **standalone**, with no multi-document
transactions. A separate `outbox` collection would need one, so the event lives inside the
document whose write it describes — and Mongo guarantees a single-document write is atomic.

The index on it is partial: `{'pendingEvents.0': {$exists: true}}`. That keeps it holding the
handful of documents with unpublished work rather than every message ever written, which is what
stops the poller getting slower every day the service runs.

The integration tests run against a plain standalone container rather than Testcontainers'
`MongoDBContainer`, which starts a single-node *replica set*. Testing against a replica set would
green-light code that cannot run where it is deployed.

### Idempotent send

A phone that loses its connection after the write but before the response **will** retry. The
client sends a `clientMsgId`; the service looks for an existing message with it and returns that
one if found.

The lookup and the insert are two operations, so six concurrent retries can all miss the lookup.
A unique partial index on `(channelId, clientMsgId)` decides the winner, and the losers catch
`DuplicateKeyException` and return the existing message.

That index used to exist only in `infra/mongo/init/01-init.js`, which runs once and only on an
empty data directory — so a restored backup, a managed Mongo, or a developer with a database dump
had no index and no idempotency. An integration test against a fresh container turned six
concurrent retries into **five messages**. `MongoIndexInitializer` now creates all seven indexes
at startup, idempotently.

### Cursor pagination on a pair

The cursor is base64 of `<epochMillis>:<messageId>`, and the query is:

```
createdAt < :ts OR (createdAt = :ts AND _id < :id)
```

A timestamp alone is wrong, and the failure is subtle: two messages posted in the same
millisecond either both appear on consecutive pages or neither does. The pair breaks the tie
deterministically.

Offset pagination is wrong for a different reason. With `OFFSET 50`, a message arriving between
two requests shifts everything down — the reader sees one message twice and never sees another.
A cursor describes a *position in the data*, not a count of rows skipped.

### Reactions, and the order of the update

A reaction array holds one entry per emoji, each with the set of users:

```json
"reactions": [{ "emoji": "tada", "userIds": ["u1", "u2"] }]
```

Adding a reaction is two possible operations: `$addToSet` into an existing entry, or `$push` a
new entry. The order matters.

Try `$addToSet` **first**; only `$push` if it matched nothing. Backwards, two simultaneous
first-reactions each create an entry, the emoji appears twice on the message forever, and no
amount of un-reacting fixes it.

### Delete scrubs, including the outbox

Soft delete — the row stays so threads keep their root and the removal is recorded — but the
content is genuinely cleared: `text`, `blocks`, `previousText`, `attachments`, `mentions`.

`previousText` is the one people forget. Without clearing it, deleting an edited message leaves
the pre-edit text, which is frequently the exact thing being deleted.

And the staged outbox events are redacted too. The outbox lives *inside* the document, so
clearing `text` while an unpublished `message.created` event held a copy left the plaintext in
Mongo until the publisher happened to drain it. It also governs what *leaves* the service:
publishing that event after the delete pushes content downstream that the user has already
removed.

An acceptance test caught this intermittently — on a fast machine the delete lands before the
poller runs, on a slow one it does not.

---

## realtime-service

### STOMP over WebSocket, and where authorisation happens

```
CONNECT    ── token in the frame → verified, principal attached to the session
SUBSCRIBE  ── destination checked against workspace-service → allowed or rejected
MESSAGE    ── server pushes; clients may not SEND to /topic at all
```

**Authentication is on the CONNECT frame, not the HTTP upgrade.** The browser WebSocket API
cannot set headers, so the token travels in the STOMP frame. The handshake itself is
unauthenticated by necessity; the session is useless until CONNECT succeeds.

**Authorisation is per-SUBSCRIBE.** This is the boundary most implementations miss. Verifying
the token on CONNECT proves who someone is; it says nothing about which channels they may
listen to. Without a check here, any authenticated user can subscribe to
`/topic/channel.{anyId}` and receive every message in a private channel.

**Clients cannot publish to a broadcast topic.** A direct SEND to `/topic/...` is rejected.
Messages are created over HTTP, which is where the access check and the outbox write live;
allowing a socket to publish would be a second, unguarded write path.

### Fan-out across instances

Each instance consumes Kafka under `realtime-service.{instanceId}` — a **per-instance group**, so
every instance receives every message and pushes it to whichever clients happen to be attached
to it.

This is the opposite of every other consumer in the system, and the reason is that there is no
shared broker between instances: a client's socket lives on exactly one pod. With a shared
group, a message would reach one instance and the clients on the other two would never see it.
That bug does not reproduce on a single replica, which is the worst property a bug can have.

`auto-offset-reset: latest`, because a realtime push about a message from an hour ago is noise.

### Presence is a TTL key, not a flag

`presence:{userId}` with a 60-second expiry, refreshed by a heartbeat, plus a
`presence:sessions:{userId}` counter for multiple devices.

A boolean flag set on connect and cleared on disconnect is wrong because **nothing runs when a
laptop lid closes**. No disconnect frame arrives, the flag stays true, and the user shows as
online for days. Expiry is the only mechanism that handles an absence rather than an event.

The counter exists so closing one of two tabs does not mark you offline.

Typing indicators go over Redis pub/sub rather than Kafka: they are worthless three seconds
later, so durability would be pure cost.

---

## api-gateway

### It strips what it sets

The gateway verifies the JWT and sets `X-User-Id` and `X-User-Email` for downstream services. It
**removes those headers from the inbound request first**.

Without that, a client sends `X-User-Id: <anyone>` and, on any route where the gateway's own
filter does not overwrite it, the downstream service believes it. Strip-then-set is one line and
the difference between a trusted header and a forgeable one.

### Two traps that cost real time

**It must not component-scan `com.slackclone.common`.** The gateway is reactive (WebFlux);
common-lib contains a servlet filter. Scanning it makes Spring try to load `GenericFilterBean`,
which is not on a reactive classpath, and the failure is a
`BeanDefinitionStoreException: I/O failure` that names a class the gateway does not use. The fix
is to scan only its own package and `@Import(JwtService.class)` explicitly.

**`InetSocketAddress.getAddress()` returns null behind a proxy.** The rate limiter keys on the
client address. Behind nginx the address is unresolved, `getAddress()` is null, and every request
NPEs — but only in a deployment with a proxy in front, which is to say never on a developer's
machine. Null-guarded, falling back to `getHostString()`.

---

## search-service (Python)

### The authorisation filter is applied before the query

The query is built as:

```
filter: workspaceId = X AND channelId IN (channels this user may read) AND isDeleted = false
query:  match on text
```

The channel list comes from workspace-service, and it is a **filter clause, not a
post-processing step**. Filtering results after the search is the version that leaks: the `total`
count still reflects the unfiltered match, so a non-member learns how many messages in a private
channel mention a term they guessed.

### No stemmer, deliberately

The analyser is `standard → lowercase → asciifolding → stop`. A stemmer would make "deploying"
match "deploy", which sounds desirable and is wrong for chat: it also makes "running" match "run"
and, in a technical workspace, mangles identifiers and product names into each other. Chat search
is mostly people looking for a phrase they remember, so exactness beats recall.

Ids are `keyword`, not `text`. This is the most useful distinction in OpenSearch: `text` is
analysed and is for prose, `keyword` is stored whole and is for filtering. A UUID typed as `text`
is tokenised on its hyphens, and a filter for it silently matches nothing.

### Deleted messages are tombstoned, not removed

A delete updates the document with `isDeleted: true` and empty text, using `doc_as_upsert`.
Removing it outright would be simpler and breaks on out-of-order delivery: a delete arriving
before the create it refers to would delete nothing, and the message would be searchable forever
once the create arrived.

---

## notification-service (Python)

### Four reasons not to send

In the order they are checked:

1. **Already handled.** The event id is claimed in `notification.sent_notifications` with a
   unique constraint. An email is the least forgiving thing to send twice — the user sees it, and
   no later correctness removes it from their inbox.
2. **They are online.** A live WebSocket means they watched the message arrive. Emailing anyway
   is what makes a product feel noisy.
3. **Cooldown.** Already told about this channel in the last five minutes. A thread with twenty
   replies that each mention you is twenty events; twenty emails is how a product teaches
   someone to turn notifications off for good.
4. **No address on file.** Either the registration event has not arrived or the account was
   deactivated.

### The claim happens before the send

Deliberately, because the failure modes are not symmetric. A crash between claiming and sending
loses one notification — recoverable, the message is still in the app. Claiming afterwards means
a crash between sending and claiming sends a second copy of an email the user has already read.

A *failed* send releases its claim, so a redelivery can retry. Without that, one refused SMTP
connection suppresses the notification permanently with nothing afterwards to indicate it.

### Two read models, and the ordering bug they hid

The service has no user table and no channel table of its own. It builds both from
`user.events` and `channel.events`, because calling auth-service per email would put
authentication on the critical path of sending mail.

The channel read model carried `workspace_name` denormalised, updated by `workspace.created`.
That applies the name to the channels that exist *at that moment* — and since a workspace is
created before anything in it, that was none of them. Every email read `mentioned you in
#general ()`.

The fix is a small `workspaces` table so a channel can resolve the name whenever it arrives,
which also makes the consumer correct under either order. That matters beyond this bug: the Kafka
key is the aggregate id, so workspaces and channels sit on independent partitions and the broker
guarantees nothing about their relative order. **A consumer that only works in one order is
wrong, not merely fragile.**

### Seeded data publishes no events

Related, and worth knowing generally: the dev seeds insert rows directly, so no application code
runs and no event is published. Every read model built from those events had never heard of the
seeded users or channels — emails said "Someone mentioned you in #cccccccc".

Both seeds now stage the same outbox events a real signup or channel creation produces, with
md5-derived ids so the backfill is idempotent. Publishing through the outbox rather than
special-casing the seed in each consumer is the point: consumers handle it through their normal
path, and it is the same approach a production backfill after adding a new consumer would use.

---

## Frontend

### Single-flight token refresh

Ten requests fail with 401 at once when the access token expires. Without coordination, ten
refresh calls go out — and since refresh tokens rotate, nine of them present an
already-redeemed token, which triggers reuse detection and logs the user out.

A module-level promise is the fix: the first 401 starts the refresh, the rest await the same
promise.

Refresh is attempted only on `401 + TOKEN_EXPIRED`, not on any 401. A 401 from a revoked session
must log the user out rather than loop.

### Optimistic send and reconciliation

The message renders immediately with a client-generated id, and is replaced when the server
replies. The same `clientMsgId` is what makes a retry idempotent server-side, so one value
solves both problems.

`crypto.randomUUID` is **not** used for it. It is undefined outside a secure context — plain
HTTP on anything other than `localhost` — so it threw before `fetch` was ever called. No request
appeared in any log, on any service, and the UI said "could not reach the server". `src/lib/id.ts`
uses `crypto.getRandomValues`, which is available everywhere.

### Scroll compensation

Loading older history prepends content, which moves everything down. `useLayoutEffect` captures
`scrollHeight` before the paint and restores the offset after it, so the view stays where the
reader left it. In `useEffect` the user sees a jump.

Auto-scroll on a new message only happens if the reader is within 60px of the bottom. Scrolling
someone away from what they are reading because somebody else typed is worse than a missed
scroll.

### API composition on the client

Services return user *ids*; the browser batch-fetches profiles and caches them. The alternative
is every service joining against auth-service's data, which is the coupling the boundaries exist
to prevent.

The batch fetch is also single-flighted — rendering a channel with forty authors must not
produce forty requests.
