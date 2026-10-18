# Parallel scenes roadmap

Status: proposal for maintainer review. This document describes an incremental
route to split-party play; it does not make parallel scenes available by
itself.

Open Dungeon Master currently has one live story context per campaign. That
simple invariant reaches farther than the visible scene: one DM queue, one
message stream, one active encounter, one floor, one structured scene tracker,
one ambience state, and one event replay cursor all assume that the party is
together. Parallel scenes must change those boundaries without changing how a
normal table joins or plays.

The order below is deliberate. Durable jobs, retry-safe tools and recoverable
event delivery must land before two storytellers are allowed to mutate one
world concurrently.

## Invariants

- A room code, `/join/CODE`, QR code and `odm://` invite continue to join a
  **campaign**, never a private scene. Authentication, seat limits and
  character ownership stay campaign-scoped.
- Each campaign always has one default story thread. Existing campaigns are
  migrated into it and behave exactly as they do today.
- Each campaign member has one persisted active thread. A returning player
  resumes it; a new player enters the campaign's designated arrival thread.
- Authorization and projection happen on the server. A client is never sent a
  private thread's narration, rolls, maps, encounters or DM status and asked to
  hide them.
- Campaign-global facts remain global. Thread-local narration and tactical
  state must name their thread explicitly.
- Older clients remain safe. They see their assigned thread as the table and
  receive an ordinary resync when the assignment changes; they do not gain a
  thread switcher until updated.
- “Story thread” is the internal name. It avoids colliding with the existing
  weather scene state, tactical-map scenes, scene tracker and transcript scene
  chunks. The UI may simply call these scenes.

## Phase 0: reliable AI work

Before parallel execution, make the current single queue recoverable:

1. Record mutating tool-call receipts so a retry cannot apply the same damage,
   item, currency, encounter, clock or world change twice.
2. Persist DM jobs and their queued, running, waiting, parked, recovered,
   failed, cancelled and completed states.
3. Commit narration and replayable events through a recoverable outbox.
4. Separate interactive narration from chapter summaries, compaction,
   embeddings, world ticks, TTS, pictures and maps.
5. Expose the authoritative job state in the campaign snapshot so browser,
   desktop and Android clients recover after sleep or network loss.

The durable job scope is `{ campaignId, storyThreadId? }`. The optional thread
is unused until Phase 3, which keeps these prerequisites independently useful.

## Phase 1: compatibility foundation

Add `story_threads` with one active default row per campaign and add a
persisted active thread to campaign membership. Backfill both in the database
migration. Do not add a campaign-creation setting: the default thread is the
safe default and a campaign only creates more threads when the DM splits the
party.

The campaign snapshot gains additive protocol fields:

- a parallel-scenes protocol version;
- the caller's active thread id;
- summaries of threads the caller may enter; and
- the active thread's revision.

All existing fields still describe the default/active thread, so older clients
continue to render one coherent table.

## Phase 2: membership and navigation

Add server-authorized operations to create, rename, close, merge and assign a
story thread. The DM and assistant DM may observe every thread; an ordinary
seat sees only threads it is permitted to enter.

The built-in web table gains a compact scene switcher only when more than one
authorized thread exists. The same component is bundled into desktop and
Android. The native repository needs compatibility and lifecycle tests, not a
second implementation of the switcher.

Moving a connected seat publishes a persisted assignment event and requests a
snapshot resync. Moving a sleeping or disconnected phone changes the same
database field; its next snapshot opens the assigned thread. No new invitation
is required.

## Phase 3: thread-scoped table state

Add a thread id to the state that can diverge while the party is split:

- messages and streaming narration;
- rolls and pending rolls;
- floor ownership and queued player intent;
- encounters, initiative and tactical maps;
- structured scene tracking, ambience and scene cards;
- DM turns, jobs, tool receipts and media work; and
- replayable events and snapshot projections.

Accounts, campaign membership, character libraries, campaign configuration,
the admitted ruleset and explicitly global lore/world facts remain campaign
scoped. Whispers retain their existing recipient authorization and additionally
name a thread only when their content belongs to one.

The event store records a thread scope. Snapshot, replay and live fan-out apply
the same authorization predicate so reconnect cannot reveal data that the live
stream would refuse.

## Phase 4: concurrent DM turns

Replace the campaign-wide serialization key with a story-thread key. One
thread remains serialized internally, but independent threads may narrate at
the same time.

Campaign-global mutable state receives a monotonic revision. A tool that read
global state includes that revision when it writes. A mismatch does not
silently overwrite the other thread: the job is parked for a deterministic
re-read/retry or DM review. The clients show “waiting on another scene” and the
resolution rather than leaving a spinner running.

Per-thread state does not pay this conflict cost. Rolls, local messages and a
thread's encounter use their own thread serialization.

## Phase 5: clocks, reunion and world simulation

Track thread-local elapsed time beside a campaign chronology. Advancing one
thread never silently advances another. Scheduled campaign-global events fire
only under an explicit chronology rule recorded with the event.

Merging threads produces a bounded catch-up summary for each returning seat,
then closes the source threads after their jobs settle. The merge is
idempotent: reconnect or retry cannot duplicate messages, encounters or world
mutations.

Only after these rules are stable do quests, factions, NPC movement, scheduled
events and background world simulation become thread-aware.

## Client and join compatibility matrix

| Client | Server | Behaviour |
| --- | --- | --- |
| Old | New | Joins with the same link; receives only its assigned thread as one table; assignment changes force a compatible resync. |
| New | Old | Detects no parallel-scenes protocol and renders the current single table with no switcher. |
| New | New | Restores the assigned thread, offers authorized switching and shows thread-scoped AI recovery state. |

Android acceptance covers joining by room code, web invite, QR and deep link;
backgrounding during queued, streaming and pending-roll states; tunnel address
changes; offline/online transitions; and process restart. The server snapshot,
not local UI memory, is authoritative after every resume.

## Delivery gates

Each phase ships independently and keeps single-thread campaigns unchanged.
Required tests include migration of a populated campaign, old/new client-server
pairings, crash injection around tool mutation and event publication, two-seat
privacy across snapshots and replay, simultaneous DM jobs, shared-world write
conflicts, mobile background/resume, and an idempotent reunion.

Server changes land before a native-client release that depends on them. The
client bundle pins a compatible server revision and retains the older-host
fallback already used by the portal.
