# Testing strategy

## Four layers, each catching what the one below cannot

| Layer | Count | Runs against | Catches | Time |
|---|---|---|---|---|
| Unit | 71 | One class, collaborators mocked | Logic and branch errors | ~5s |
| Integration (`*IT`) | 47 | One service + real Postgres/Mongo via Testcontainers | Wrong SQL, mapping bugs, transaction boundaries, constraint behaviour | ~90s |
| Acceptance | 325 checks, 10 suites | The whole stack over the network | Wiring, config, HTTP contracts, cross-service behaviour | ~10min |
| End-to-end | 12 | A real browser | Rendering, client state, realtime delivery | ~30s |

The layers are not a pyramid for its own sake. Each one exists because there is a specific class
of bug the others structurally cannot see.

```
                 ┌──────────────┐
                 │  E2E (12)    │  a real browser
                 ├──────────────┤
                 │ Acceptance   │  the whole stack, over a network
                 │ (325 checks) │
                 ├──────────────┤
                 │ Integration  │  one service + real databases
                 │    (47)      │
                 ├──────────────┤
                 │  Unit (71)   │  one class
                 └──────────────┘
```

## Unit tests

`mvn test`. Mocked collaborators, no Spring context, no database — the whole set runs in about
five seconds, which is what makes it reasonable to bind to a save action.

**What they are good at:** branch coverage over decisions. `MentionParserTest` is the clearest
example — 19 tests over one method, because that method decides who gets notified. Over-matching
emails a hundred people who were not addressed; under-matching means a mention highlights in the
UI and never reaches the person, which is worse, because they can see afterwards that they were
asked and never told.

**One deliberate exception to mocking.** `AuthServiceTest` uses a real `BCryptPasswordEncoder`
at cost factor 4 rather than a mock. A mocked encoder returns whatever `matches()` was told to
return, so the tests would keep passing if password checking were deleted outright — which is
the single most important thing in the class.

**What they cannot see:** anything about the seams. A mocked repository returns what the test
said, so a test asserting "the repository was called with these arguments" passes whether or not
the SQL behind it is valid.

## Integration tests

`mvn verify`. Named `*IT` so Failsafe runs them in the `verify` phase rather than Surefire
running them on every `test` — that split is what keeps the unit tests fast enough to run
constantly.

**Real databases, not H2.** H2 would start in a second and every test would be worthless. These
services depend on things H2 does not have: `citext` for case-insensitive uniqueness, partial
indexes, the `inet` type, `INSERT … ON CONFLICT DO NOTHING`, Mongo's single-document atomicity
and `$addToSet` semantics, and Postgres' specific constraint-violation messages that the
exception handler maps to error codes.

A suite that passes on H2 and fails in production is worse than no suite, because it produces
confidence instead of information.

**Mongo runs as a plain standalone**, not Testcontainers' `MongoDBContainer` — which starts a
single-node *replica set* and therefore supports multi-document transactions. Production is
standalone and does not, which is the whole reason the outbox is embedded in the message
document. Testing against a replica set would green-light code that cannot run where it is
deployed.

**What they found that unit tests could not:**

- A unique index that existed only in an init script, so six concurrent retries of one
  `clientMsgId` produced five messages.
- `userId IN ()` — invalid SQL in Postgres — reachable whenever a message mentioned only its
  author.
- The exact text of constraint-violation messages, which the error mapping depends on.

**JaCoCo gates line coverage at 70% per module**, and a module below it fails the build. Current:
auth-service 77.6%, workspace-service 73.9%, message-service 78.3%.

A number is a blunt instrument and worth being honest about: 70% is a floor that stops a module
quietly decaying, not a target that means anything on its own. It is possible to hit 90% with
tests that assert nothing useful. What the gate actually buys is that *adding a service with no
tests fails the build* — which is how workspace-service and message-service came to have any.

## Acceptance suites

`bash scripts/verify/run-all.sh`. Ten suites, 325 assertions, driving the running stack over
HTTP and WebSocket.

This is the layer that earns its keep most often, and the reason is structural: every layer above
runs inside one JVM with the framework configured by a test profile. The bugs that live in the
gaps *between* services and in configuration are invisible to all of them.

Everything in this list was found by this layer and by nothing else:

- A Kafka producer whose `delivery.timeout.ms` was below `linger.ms + request.timeout.ms`. The
  broker was healthy, the integration tests passed, and the outbox silently stopped publishing.
- An exception thrown after a write, rolling back the write it was meant to record — twice, in
  two different places. Both looked correct in isolation and both lost data in a running system.
- `crypto.randomUUID` undefined outside a secure context, so the client threw before `fetch` was
  called. No request appeared in any log, on any service.
- CORS rejecting a request whose Origin was the app's own hostname.
- A compression codec linked against glibc, loaded on an Alpine image.
- A null `InetSocketAddress` behind a reverse proxy — which only happens behind a reverse proxy.
- **A consumer with no producer.** workspace-service's `user.deactivated` handler had been
  written, reviewed and merged; auth-service published nothing. A deleted account stayed in every
  workspace and channel it had joined. Both halves read correctly in isolation, and no test that
  looks at one service can see it.
- A workspace name applied to the channels that existed at that moment — none, since a workspace
  precedes its channels. Every notification email read `mentioned you in #general ()`.
- An alert rule on a metric no service published, and a dashboard panel whose query matched
  nothing. Both render as silence, which is indistinguishable from everything being fine.

### The conventions, and the mistakes behind them

Each of these is in `scripts/verify/README.md` because a test suite got it wrong first.

**Assert on a count, not a substring.** The search API echoes the query back in its response
body, so grepping the body for the search term matched even with zero hits. The suite passed for
many runs while proving nothing.

**Prove the negative case reached the system.** "The replayed event did not double the counter"
passes just as happily when the replay was never delivered. The dedupe tests therefore also
assert the consumer *logged the skip*. Without that second assertion it is not a test, it is a
coin flip that always lands heads.

**Poll for a value, never sleep a guessed interval.** Anything crossing Kafka takes an outbox
tick plus a broker hop plus a consumer poll. A fixed `sleep 3` is either slower than necessary
or flaky on a loaded machine, and usually both.

**Make every assertion relative to a snapshot.** Mailpit keeps mail between runs, so "wait until
one message matches" returned instantly on a previous run's email — the suite then raced ahead of
its own email and the next section's baseline was wrong. One mistake, three unrelated-looking
failures.

**Reset what you change.** These run against a shared database. A suite that promotes a user to
ADMIN and leaves them there makes the next run's "a member cannot promote themselves" check fail
on a correct service.

**Be re-runnable.** Registering a fixed email passes exactly once and 409s forever after, taking
down every later test that needs the resulting token. One run reported nine failures that were
all the same stale row.

**Select containers by compose label, not by image.** `--filter ancestor=<tag>` looks obvious and
is a trap: rebuilding the image moves the tag, and a container already running from the old id no
longer matches its own tag. The filter silently returns nothing and the assertion fails claiming
an event was never processed while the log line is sitting right there.

## End-to-end tests

`bash e2e/run-in-docker.sh`. Playwright, in the official image so browser binaries and their
system libraries come with it — this works the same on a laptop and on CI without anyone
installing Chromium.

**Deliberately few.** Twelve tests, each going through the full stack: browser → nginx → gateway
→ services → databases → Kafka → WebSocket → browser. They are slow and the most likely layer to
be flaky, so the set is small and every one of them covers something no other layer can.

The two that justify the whole layer:

**Two independent browser contexts.** Separate cookie jars and separate localStorage, so they are
genuinely two users rather than two tabs sharing a session. Alice sends; Bob must receive without
reloading. If any link in the chain is broken this is the test that fails.

**A mention badge that survives a reload.** Keeping a badge correct while a tab is open needs
nothing but the incoming socket event — the client can count for itself. What it cannot do is
know the count after a refresh. So the assertion that matters is the one *after* `reload()`: a
version where the counter is only ever computed in the browser passes every line above it and
fails there.

### Selector discipline

Role and text based, never CSS classes or test ids. A selector on `.sidebar > div:nth-child(2)`
breaks when someone adds a wrapper, which teaches people that E2E failures are noise.

One real lesson: `getByText('Alice Anderson')` matched a single element when the seed data was
fresh and ten of them once Alice had posted a few messages, each labelled with her name. A
selector that depends on how much data happens to exist is one that fails on a Tuesday for no
reason anyone can explain. Scoped to the element holding the sign-out control instead.

---

## The two QA roles

The user asked for "2 testers". In a team this size that is two distinct roles rather than two
people doing the same thing, and the division is by *what they can see*.

### QA 1 — functional and contract

Owns the acceptance suites. Works from the API contract and the acceptance criteria on each
ticket, without reading the implementation first.

That last part is the discipline that makes the role worth having. Someone who has read the code
tests the paths the code takes; someone working from the contract tests the paths the contract
allows — and the gap between those two sets is where the bugs are. Every item in the list above
is from that gap.

Their deliverable for a ticket is a suite section with the ticket number, and their standing
question is "what does the API let me do that it should not?" Concretely:

- Can a non-member reach a private channel? (404, not 403 — a 403 confirms it exists)
- Does the same `clientMsgId` twice produce two messages? (and six at once?)
- Does a role change take effect immediately, or is it cached?
- What happens on the second call — the retry nobody tested?

### QA 2 — integration, resilience and non-functional

Owns the E2E suite, the realtime verification, and the failure-injection work. Tests the system
*as a system*: two users at once, a dependency removed, a clock skewed, an event replayed.

Their standing question is "what happens when a piece of this is broken?" Concretely:

- Stop Kafka. Does posting a message still work? (it must — events queue in the outbox)
- Stop workspace-service. Does message-service fail *closed*? (it must)
- Replay an event. Does the counter double? Does a second email go out?
- Kill a realtime instance. Do clients reconnect and resubscribe?
- Two browsers, same channel, simultaneous sends.

The second role is the one most teams skip, and it is the one that found the dedupe gaps, the
rollback bugs and the fan-out strategy error.

### How a ticket moves

```
In Progress ──> Code Review ──> QA ──> Done
                                 │
                                 └── QA 1 verifies acceptance criteria against the contract
                                     QA 2 verifies it under failure and alongside other users
```

A ticket does not reach Done on a passing pipeline alone. The pipeline proves no *regression*;
QA proves the thing in the ticket actually works, which are different claims.

### What QA does not do

Not manual regression testing. Every check either becomes a script in `scripts/verify` or a
Playwright test, and then runs forever. A QA role spent re-running last sprint's checks by hand
is a role that cannot look at this sprint's.

---

## Running it

```bash
mvn -f backend/pom.xml test                 # unit, ~5s
mvn -f backend/pom.xml verify               # + integration + coverage gate, ~2min
bash scripts/verify/run-all.sh              # acceptance, ~10min, needs the stack up
bash e2e/run-in-docker.sh                   # browser, ~30s
```

In CI the first two run on every push; the acceptance suites run against deployed staging, which
is where they belong — they need the whole system on a network, and that is what an environment
is. See [06 — DevOps](06-devops.md).

## What is not tested

- **No load or soak testing.** Nothing here establishes how many concurrent WebSocket
  connections one realtime instance holds, which is the number that would actually size the
  deployment. k6 or Gatling against staging is the next thing worth adding.
- **No chaos testing beyond manual container stops.** The resilience checks above are run by
  hand rather than on a schedule.
- **No contract tests between services.** The acceptance suites catch a broken contract after
  both sides are deployed together. Pact would catch it in the producer's own pipeline, before
  the consumer ever sees it — worth it once the two sides are owned by different teams.
- **No accessibility testing.** The components use roles and labels (which is partly why the E2E
  selectors can be role-based), but nothing asserts it. `axe-core` in the Playwright suite is a
  small addition.
- **No mutation testing.** The coverage gate measures whether a line ran, not whether anything
  would notice if it changed. PIT would answer the more useful question.
