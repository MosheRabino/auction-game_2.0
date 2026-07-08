# Project context: Auction Game 2.0 (auction-game_2.0)

This file is meant to be uploaded to a new conversation with Claude to resume work on
this project if the current conversation gets interrupted. **Upload this file
together with your current `index.html`** (and `README.md`/`config.json` if you want
doc changes too or have them handy - config.json is optional/sensitive, you can just
tell Claude your Supabase project is already set up). Claude cannot see prior
conversation history on its own; this file is how continuity gets re-established.

Suggested opening message when you upload this: *"Here's the context file and current
state of my auction game project - please read it and let's continue."*

**IMPORTANT - file version note:** partway through this project's history, a version
mismatch happened where Claude was given a stale/older copy of `index.html` for a few
turns and built on top of it, producing a shorter diff than expected. The fix was
re-uploading the actual current file. **Going forward, always upload the exact
`index.html` you're currently running** (the one downloaded from Claude's last
response in this project, or whatever you've since deployed) - don't assume an old
upload in a prior conversation still matches what you're playing with.

## What this project is

A browser-based fantasy-league player auction game for friends (Hebrew UI, RTL). Each
participant gets a budget and takes turns bidding on players pulled from a league (or
several leagues) until every roster is full. Originally a single-device "pass the
phone" game; now also supports playing online with each person on their own device.

Repo: `github.com/MosheRabino/auction-game_2.0`

## File structure

```
index.html      - the entire game/app: HTML + CSS + JS, one file, no build step
config.json     - Supabase project URL + anon key (kept separate so index.html can be
                   regenerated/updated without wiping your real credentials)
Leagues.json    - list of available league file names
PT_26.json      - example league file
GS_26.json      - example league file
README.md       - full setup/usage documentation
```

## Chronological history of what's been built (in order)

1. **Original state (pre-Claude):** a working local hotseat auction game. 2-4 players,
   fixed $20 budget, fixed 5-player rosters, single hardcoded league selection.
2. **Settings + multi-league support.**
3. **Persistent color palette fix** (108° spacing on a 10-color wheel).
4. **Lobby/game persistence (localStorage, local mode only).**
5. **Step-through auto-fill** for the last remaining incomplete roster, one player at a
   time via "OK, next" - see step 23 below for its 5s auto-timer.
6. **Online multiplayer (Supabase Realtime + Postgres + anonymous auth).**
7. **"New Round" instead of leaving the room.**
8. *(superseded by step 13)* Originally only the host typed a name up front.
9. **30-second turn timer**, auto-acts on timeout, any connected client can fire it.
10. **Conclusion-screen money hidden** in the shared summary/screenshot.
11. **Room sharing + deep link** (`?room=CODE` auto-joins).
12. **config.json externalized** from index.html.
13. **Removed fixed player count for online rooms** - always 10 seats, host starts once
    ≥2 have joined.
14. **Cosmetic:** 🏆→🏀, title/timer-font tweaks.

*(Steps 1-14 above are unchanged carryover from before this session - see below for
everything added in this session.)*

15. **Online color privacy model, iterated to its final form:**
    - Every player's dashboard card **always shows an outline in their own color**, in
      both local and online modes - pure identification, not a reveal of anything.
    - The **full color fill** (background tint, not just outline) is reserved
      differently per mode:
      - **Online:** only your own seat, always - so you can spot your own card at a
        glance regardless of whose turn it is.
      - **Local (pass-the-phone):** only whoever's turn it currently is - there's no
        single "you" to protect on a shared device, so this just tracks the turn like
        it always did.
    - The arena ("big screen") is untouched - it already changed color for whoever's
      turn it is, in both modes, before this session.
    - At the summary screen, everyone's true color comes back (via the color dot),
      since privacy no longer matters once the game's over.
    - Implementation lives in `playerColumnHtml()` (the dot) and `highlightColumn()`
      (the card border/fill).

16. **Room locks permanently once a game starts, until the host closes it:**
    - `attemptJoinRoom()` now checks: if the room's `state` already exists, only a
      uid that's already in `state.players` (a genuine reconnect, e.g. after a
      refresh) is let in. Anyone else gets "המשחק כבר התחיל - החדר נעול" and is
      turned away.
    - This lock is permanent for the room's whole lifetime, including every future
      "New Round" within it - there's no way to unlock a room short of the host
      closing it (step 17).

17. **Host leaving closes the room for everyone:**
    - `closeRoomAsHost()` does an actual `DELETE` of the room row in Supabase.
    - Every other connected client's realtime subscription gets the `DELETE` event
      (added a `payload.eventType === 'DELETE'` branch to the channel handler) and
      reacts with `handleRoomClosedByHost()` - alert "המארח עזב את החדר - המשחק
      הסתיים" then back to the main screen.
    - Wired into `resetToHome()` (leaving mid-game) and `leaveToMainMenu()` (leaving
      from the summary screen) - both check `amIHost` first and close the room
      instead of just reloading locally.

18. **Leaving the pre-game lobby frees your seat immediately (and safely):**
    - New atomic Postgres function `leave_seat(room_code, seat_index, uid)` - mirrors
      `claim_seat`'s "only if still empty" guarantee, but for freeing ("only clears a
      seat if it's still yours"). Two atomic single-seat writes on the same row can
      never lose each other's changes, so freeing a seat while someone else is mid-
      `joinOpenSeat()` is safe by construction - confirmed this reasoning explicitly
      with the user before building it.
    - `leaveLobby()` (replaces the old bare `location.reload()` on the lobby's "עזוב"
      button): host → `closeRoomAsHost()`; anyone else with a claimed seat → calls
      `leave_seat` (guarded by a new `iAmLeavingVoluntarily` flag so my own client
      doesn't mistake my own voluntary leave for being kicked - see step 19) then
      reloads.

19. **Host can kick anyone, both pre-game and at the post-game ready-check:**
    - New atomic Postgres function `host_kick_seat(room_code, seat_index, host_uid)` -
      same shape as `leave_seat` but gated on `hostUid` instead of seat ownership, so
      the host can clear *any* seat.
    - UI: a "✕" button next to every other claimed seat in the room lobby, and next
      to every other still-seated player in the new ready-check screen (step 21).
    - Detection on the kicked person's own client: the realtime channel handler keeps
      a live `currentLobby` cache and checks (by **uid**, not raw seat index, since
      seat indices get compacted differently once a game starts - `players[i].id` is
      the position within *claimed* seats, not the raw lobby.seats slot) whether I'm
      still present anywhere in `lobby.seats`; if not (and I didn't just leave
      voluntarily myself), `handleKickedByHost()` fires - alert "המארח הוציא אותך
      מהחדר" then home.

20. **Host-only "skip" button during any turn:**
    - New `hostSkipBtn` in the arena, visible only when `isOnline && amIHost`,
      available regardless of whose turn it is (own or someone else's).
    - `hostForceSkip()` reuses the exact same deterministic paths the 30s timer
      already used: `handleAction('pass', true)` for a normal bidding turn, or
      `confirmAutoFillPick(true)` (added a `force` param) during auto-fill - so it's
      safe no matter who triggers it, same reasoning as the existing timer.

21. **Ready-check before "New Round" + starting player is now random:**
    - Shared `state.readyUids` (uids who've clicked ready for the next round), reset
      to `[]` every time `hostStartOnlineGame()` runs (covers both the very first game
      and every subsequent New Round).
    - New atomic Postgres function `mark_ready(room_code, uid)` - jsonb array-append
      guarded by "not already present", so simultaneous ready-clicks from different
      players can't lose each other's entries (same atomic-RPC philosophy as
      `claim_seat`/`leave_seat`/`host_kick_seat`).
    - Summary screen: new `readyCheckContainer` shows "מוכנים: X/Y" plus each still-
      seated player's status; non-host players get a "מוכן לסיבוב הבא ✅" button
      (optimistic local update + the RPC call); the host's "New Round" button stays
      disabled until every currently-seated player (per `currentLobby`, so a kicked
      player doesn't block it) has marked ready - enforced both via the disabled
      button and a guard inside `startNewRound()` itself.
    - **Random starting player**: applied in `hostStartOnlineGame()`,  local
      `startGame()`, and local `startNewRound()` - every game and every round now
      picks a random `roundStarterIndex` instead of always seat/player 0. Verified via
      a jsdom smoke test (25 local games, confirmed the starting player's name varied
      rather than always being the same).

22. **Discussed but explicitly deferred - silent disconnect detection:** covered in
    depth why a truly silent disconnect (dead phone, closed tab, no signal) can't be
    detected by anything currently in the app - only a *voluntary* leave (someone
    clicking leave/kick) is covered by steps 16-21. A real fix would mean adding
    Supabase Realtime **Presence** (a `track()`/`join`/`leave` channel, a grace-period
    debounce to avoid false positives on ordinary refreshes, and accepting that
    total detection time is realistically 20-60s, not instant) - sized at roughly
    1.5-2x the work of everything in steps 16-21 combined, and not verifiable with the
    jsdom test harness (needs a real live test with an actual dropped connection).
    **Not built.** Revisit if silent disconnects turn out to be a real recurring
    problem in actual games, not before.
    - Also *not* built (explicitly superseded by the simpler ready-check/kick design
      above): the earlier idea of "grey out and lock out a mid-game leaver's controls,
      auto-fill their remaining roster slots for them." The host-skip button (step 20)
      now covers the practical case this was meant to solve (a stuck turn from someone
      who's gone) without needing to detect *why* they're stuck.

23. **Auto-fill step also has its own 5-second timer** (separate from the 30s bidding
    timer): if nobody clicks "אישור, הבא" within 5s, it self-confirms via
    `confirmAutoFillPick(true)` - same shared `turnTimerDisplay` element, same "any
    connected client can fire a deterministic action" reasoning as the bidding timer.
    New `ensureAutoFillTimer()`/`tickAutoFillTimer()`/`clearAutoFillTimer()` mirror the
    existing `ensureTurnTimer()` family exactly. Verified with a real 6-second wait in
    a jsdom test (not simulated) - the roster genuinely updated with no click.

## Key architecture decisions worth knowing

- **Single HTML file, vanilla JS, no build step, no framework.** Two CDN dependencies:
  `html2canvas` and `@supabase/supabase-js`.
- **Local mode and online mode share almost all game logic**; online mode adds a
  persistence/sync layer and turn-gating checks.
- **Full-state overwrites, not fine-grained diffs**, for the main game state (bids/
  turns/rosters) - simple, fine at this scale.
- **Every concurrency-sensitive mutation that ISN'T the main game state uses a
  dedicated atomic Postgres function**, not a fetch-then-write round trip - this is a
  deliberate, consistently-applied pattern: `claim_seat` (pre-existing), and now also
  `leave_seat`, `host_kick_seat`, `mark_ready` (all added this session). Each one only
  touches a single jsonb path, conditioned on an ownership/permission check in the
  `WHERE` clause, so concurrent calls can never silently clobber each other.
- **No server-side authority beyond Supabase's hosted Postgres + Realtime** - the
  turn-timer, auto-fill-timer, and host-skip actions all rely on "any connected
  client can carry out a fully-deterministic action," not a cron job.
- **Testing approach**: jsdom + a stubbed `fetch`/`supabase`/`html2canvas`, driving
  the real `index.html` in a headless DOM. This session specifically verified (for
  real, not just by reading the code): the random-starting-player distribution across
  25 local games, and the 5s auto-fill timer genuinely self-confirming after an actual
  6-second wait. Two-client realtime scenarios (host-close, kick-detection, ready-
  check races) were reasoned through carefully but **not** re-verified with the two-
  jsdom-window harness this session - worth a real live test with a friend before
  fully trusting the online-specific parts of steps 16-21.

## SQL setup required (must be run in Supabase's SQL Editor before this session's
## features will work - none of this can be run on your behalf, no live project here)

In addition to the original `claim_seat` function, three more atomic functions were
added this session:

```sql
-- Frees a seat, but only if it's still yours.
create or replace function leave_seat(p_room_code text, p_seat_index int, p_uid text)
returns boolean language plpgsql as $$
declare v_updated int;
begin
  update rooms
    set lobby = jsonb_set(lobby, array['seats', p_seat_index::text], jsonb_build_object('name', null, 'uid', null)),
        updated_at = now()
    where room_code = p_room_code
      and (lobby->'seats'->p_seat_index->>'uid') = p_uid;
  get diagnostics v_updated = row_count;
  return v_updated > 0;
end; $$;
grant execute on function leave_seat(text, int, text) to anon, authenticated;

-- Host-only: clears ANY seat.
create or replace function host_kick_seat(p_room_code text, p_seat_index int, p_host_uid text)
returns boolean language plpgsql as $$
declare v_updated int;
begin
  update rooms
    set lobby = jsonb_set(lobby, array['seats', p_seat_index::text], jsonb_build_object('name', null, 'uid', null)),
        updated_at = now()
    where room_code = p_room_code
      and (lobby->>'hostUid') = p_host_uid;
  get diagnostics v_updated = row_count;
  return v_updated > 0;
end; $$;
grant execute on function host_kick_seat(text, int, text) to anon, authenticated;

-- Marks a uid ready for the next round (atomic array-append, no duplicates, no lost writes).
create or replace function mark_ready(p_room_code text, p_uid text)
returns boolean language plpgsql as $$
declare v_updated int;
begin
  update rooms
    set state = jsonb_set(
          state, array['readyUids'],
          (coalesce(state->'readyUids', '[]'::jsonb) || to_jsonb(p_uid::text))
        ),
        updated_at = now()
    where room_code = p_room_code
      and not (coalesce(state->'readyUids', '[]'::jsonb) ? p_uid);
  get diagnostics v_updated = row_count;
  return v_updated > 0;
end; $$;
grant execute on function mark_ready(text, text) to anon, authenticated;
```

## Known limitations (intentionally accepted, not oversights)

- **Silent disconnects aren't detected** - only voluntary leaves/kicks are (see step
  22). A dead phone or closed tab just sits there until the host notices and uses the
  skip button or a kick.
- If the active player's device disconnects *and* nobody uses host-skip, a bidding
  turn still self-resolves via the existing 30s timer either way - so the game itself
  never truly stalls, even without presence detection.
- Room codes are still the only access control (unchanged from before this session).
- A room, once started, can only ever lose players (via leave/kick) for the rest of
  its life - there's no way to add a brand-new person mid-lifecycle, only for
  existing/reconnecting players to come back.

## If resuming: what to tell Claude

- Attach your **current, actually-deployed** `index.html` (see the file-version note
  at the top of this document - don't rely on an old upload from a prior chat).
- Mention whether you've already run the three new SQL functions above in Supabase.
- Mention what you want to change/add next - e.g. presence-based disconnect detection
  (step 22) is the most obvious next candidate if it's become an actual problem.
