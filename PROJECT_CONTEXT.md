# Project context: Auction Game 2.0 (auction-game_2.0)

This file is meant to be uploaded to a new conversation with Claude to resume work on
this project if the current conversation gets interrupted. **Upload this file
together with your current `index.html`, `README.md`, and `config.json`** (config.json
optional/sensitive - you can skip it and just tell Claude your Supabase project is
already set up). Claude cannot see prior conversation history on its own; this file
is how continuity gets re-established.

Suggested opening message when you upload this: *"Here's the context file and current
state of my auction game project - please read it and let's continue."*

---

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

2. **Settings + multi-league support:** added a ⚙️ settings panel (player count 2-10,
   budget $5-$100, roster size 5-12, number of leagues 1-N). Multiple leagues render
   as separate cards in the lobby, each with its own team checklist; players from all
   selected leagues merge into one pool (deduped by file+name+team).

3. **Persistent color palette fix:** player colors are ordered so any two consecutive
   turn-order players are 108° apart on the color wheel (max separation for a 10-color
   rotation) - fixes a bug where e.g. "orange" and "red" ended up adjacent and looked
   like the same color.

4. **Lobby/game persistence (localStorage, local mode only):** lobby settings/names/
   league selections always persist across visits. An in-progress LOCAL game is saved
   too, and a refresh offers to resume exactly where you left off. Deliberate exits
   ("back to home"/"new game") clear the saved game state on purpose.

5. **Step-through auto-fill:** when only one player has an incomplete roster left, they
   get remaining players for $0 - but revealed one at a time with an "OK, next" button
   rather than instantly, so it still feels like part of the game.

6. **Online multiplayer (Supabase Realtime + Postgres + anonymous auth):**
   - One shared `rooms` table (`room_code` PK, `lobby` jsonb, `state` jsonb).
   - A Postgres function `claim_seat(...)` does an atomic "claim only if still empty"
     seat claim (prevents two people grabbing the same seat).
   - RLS policy allows anon read/write scoped only by knowing the room code (like a
     Kahoot PIN) - not by hiding the anon key, which is meant to be public.
   - Host creates a room -> gets a 5-char room code -> friends join with the code and
     claim a seat -> host starts once ready -> full game state lives in the `state`
     column and every client just mirrors it via a realtime subscription.
   - Turn-gating: only the client whose seat matches the active turn has working bid/
     pass controls; everyone else sees them disabled with "waiting for X...".
   - Originally used Firebase Realtime Database + Anonymous Auth; **switched to
     Supabase** per user request (wanted free + no Google-account requirement -
     Supabase allows GitHub/email signup).

7. **"New Round" instead of leaving the room:** summary screen now has "New Round"
   (keeps the same players/seats, empties rosters, refills budgets, rebuilds/reshuffles
   the pool) separate from "Leave to main menu". Online: only the host sees/can trigger
   New Round; guests see a "waiting for host" note. Verified via a simulated two-browser
   test that both a local round-2 and an online round-2 correctly preserve player
   identity and finish cleanly.

8. **Only the host types one name for online rooms; everyone else picks theirs on
   join.** (Superseded further by step 11 below - see note.)

9. **30-second turn timer:** shows a countdown; on timeout, auto-passes for a normal
   turn or auto-"opens at $1" for the very first bidder (reuses the exact same
   `handleAction('pass')` logic the manual buttons use, so there's no separate timeout
   code path to get out of sync). Fixed a follow-up issue where the game would stall if
   the *active* player's device went offline - now **any connected client** can carry
   out the timed-out action (safe because the resulting state transition is
   deterministic regardless of who submits it; manual off-turn clicks are still
   blocked, only automatic timeouts get this exception).

10. **Conclusion-screen money hidden:** the final summary (and the shared screenshot)
    no longer shows each player's remaining budget - only names/rosters. Live in-game
    view during bidding still shows budgets normally. (Also fixed a duplicate-`id`
    issue this uncovered: the summary screen clones the live dashboard, and now strips
    ids from the clone since duplicate DOM ids are invalid HTML.)

11. **Room sharing + deep link:** a "🔗 שתף קישור לחדר" button in the room lobby
    shares (native share sheet, or clipboard fallback) a link like
    `yoursite/index.html?room=ABCDE`. Opening that link auto-joins the room directly,
    skipping the mode-select/enter-code screens entirely.

12. **config.json externalized:** `SUPABASE_CONFIG` used to be hardcoded inside
    `index.html`; it's now loaded at boot from a separate `config.json` file, so
    regenerating/updating `index.html` in the future never wipes real credentials.

13. **Removed fixed player count for online rooms (supersedes step 8's framing):**
    online rooms always create 10 seats; anyone can join an open seat freely (no
    upfront "how many players" setting - that setting is now hidden entirely in online
    mode and only applies to local mode). Host's "Start game" button activates once at
    least 2 people have joined, and shows a live count; there's no need to fill all 10.

14. **Cosmetic:** title changed from "🏆 מכירה פומבית - בחירת ליגה" to "🏀 מכירה
    פומבית - שחקנים"; all 🏆 trophy emoji in the app replaced with 🏀 basketball.
    Turn timer display font size increased (0.9rem -> 1.5rem, bold) for visibility.

## Key architecture decisions worth knowing

- **Single HTML file, vanilla JS, no build step, no framework.** Two CDN dependencies:
  `html2canvas` (screenshot sharing) and `@supabase/supabase-js` (online sync, only
  actually used once `config.json` has real values).
- **Local mode and online mode share almost all game logic** (bidding/turn/auto-fill
  functions are identical); online mode just adds a persistence/sync layer
  (`persistState()` branches between `saveGameState()`/localStorage for local vs.
  `pushOnlineState()`/Supabase for online) and a turn-gating check in the render
  functions and in `handleAction`.
- **Full-state overwrites, not fine-grained diffs.** Every action pushes the entire
  game-state snapshot to the `state` column. Simple to reason about, fine at this
  scale (a few KB, casual party-game traffic).
- **No custom backend/server beyond Supabase's hosted Postgres + Realtime.** This
  means there's no true server-side authority - e.g. the turn-timer auto-action relies
  on *some* connected client's local clock firing it, not a server cron job. Accepted
  tradeoff for staying free/serverless; documented as a known limitation.
- **Testing approach used throughout:** since this is a static site with no real
  Supabase project available to Claude, most features were verified using Node +
  jsdom - loading `index.html` in a headless DOM, stubbing `fetch`/`supabase`/
  `html2canvas`, and in multi-client tests, running two independent jsdom windows
  against a small in-memory fake Postgres-like backend (implementing `.from().select/
  insert/update()`, `.rpc()`, and `.channel().on().subscribe()` to mimic Supabase's
  client API) to simulate two real browsers/devices talking to each other. This caught
  several real bugs before shipping (e.g. a duplicate-DOM-id issue, an off-turn-client
  desync risk). This testing harness is not part of the shipped files - it's scratch
  work done during development.

## Known limitations (intentionally accepted, not oversights)

- If the active player's device disconnects *and* no other client's timer happens to
  fire (e.g. everyone else also closed their tab), the game truly stalls - there's no
  server-side keepalive. Low risk for a casual friends game, but real.
- Room codes are the only access control (like a Kahoot PIN) - RLS allows any
  authenticated-anonymous user to read/write any room if they guess/find the code.
  Fine for casual use; `generateRoomCode()` could be lengthened for more entropy if
  ever needed.
- No true atomic transaction for general state pushes (only seat-claiming uses the
  atomic `claim_seat` RPC) - relies on the deterministic-transition property to make
  concurrent writes safe rather than true locking.

## If resuming: what to tell Claude

- Attach your current `index.html` (and `README.md` if you want doc changes too).
- Mention whether you've already got a working Supabase project/config.json, or if
  something about the online setup is broken/pending.
- Mention what you want to change/add next.
