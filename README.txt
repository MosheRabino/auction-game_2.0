# 🏆 Auction Game 2.0

A player auction game for friends - everyone gets a budget, then takes turns bidding
on players from a league (or several leagues) until every roster is full. At the end
you can share a screenshot of the results.

Everything runs in the browser, in one HTML file, with no server and nothing to install.

## How to run it

Since the game loads its data files (`Leagues.json` and the league files) via `fetch`,
you **can't** just open `index.html` directly from disk (`file://`) - the browser will
block those requests. You need to serve the folder through some static server. For example:

```bash
# with Python (already installed on almost every machine)
python3 -m http.server 8000
```

Then open in a browser: `http://localhost:8000/index.html`

Alternatively, publish the repo as GitHub Pages (Settings → Pages → Deploy from branch)
and play directly from the public link - that also makes it easy to send friends a link
to play together (each from their own device, per what's described below).

## File structure

```
index.html      - the whole game: HTML + CSS + JS, one file
Leagues.json     - list of the available league file names
PT_26.json       - example league file
GS_26.json       - example league file
```

### `Leagues.json` format

A simple list of file names, each one a league:

```json
["PT_26.json", "GS_26.json"]
```

### League file format (e.g. `PT_26.json`)

The first item in the array is the league's title (the key can be either `league` or
`laegue` for historical reasons - both are supported):

```json
[
  { "laegue": "Petah Tikva 2026" },
  { "name": "Tamir Suissa", "team": "Shirat Ganim" },
  { "name": "Guy Kraus",    "team": "Arzei HaLevanon" }
]
```

**To add a new league:** create a new JSON file in the same format, place it in the
same folder, and add its file name to the list in `Leagues.json`. The league will
automatically appear in the league picker on the main screen.

## Local mode vs. online mode

The first screen lets you choose between two modes:

- **📱 Local** - exactly as before: one device, passed around between players.
- **🌐 Online** - each player on their own device, anywhere. One person creates a room
  and gets a 5-character **room code**, sends it to friends, and each of them joins
  with the code and claims a seat. Once every seat is claimed, the room's creator
  (the host) clicks "Start game" and everyone moves into the auction simultaneously,
  each from their own device. At any given moment only the player whose turn it is
  can bid/pass - everyone else sees their buttons locked with a "waiting for ⁠..."
  message until the turn passes to them.

### Setting up Supabase (one-time, only needed for online mode)

Online mode uses [Supabase](https://supabase.com) (free) - a single Postgres table
plus realtime updates - to sync state between devices, with anonymous sign-in
(players don't register or enter an email/password, just a name and a room code).
Signing up for Supabase itself can be done with GitHub or a regular email - no Google
account required.

1. Go to [supabase.com](https://supabase.com) and sign in (GitHub or email), then
   **New project** (free).
2. **Authentication → Providers → Anonymous → enable it.**
3. **SQL Editor → New query**, paste and run the block below (creates the table, the
   access policy, the Realtime hookup, and the atomic "claim seat" function that
   prevents two people from claiming the same seat at once):

   ```sql
   -- the table holding every room
   create table rooms (
     room_code text primary key,
     lobby jsonb,
     state jsonb,
     updated_at timestamptz default now()
   );

   -- permissions: anyone (even anonymous) can read/write - the room code itself is
   -- the actual protection, just like a PIN code in a Kahoot-style game.
   alter table rooms enable row level security;

   create policy "allow anon read/write" on rooms
     for all
     to anon, authenticated
     using (true)
     with check (true);

   -- enables Realtime updates on the table, so every device gets changes instantly
   alter publication supabase_realtime add table rooms;

   -- atomic seat claim: only updates if the seat is still empty, so two people can't
   -- both "claim" the same seat at the same time
   create or replace function claim_seat(p_room_code text, p_seat_index int, p_name text, p_uid text)
   returns boolean
   language plpgsql
   as $$
   declare
     v_updated int;
   begin
     update rooms
       set lobby = jsonb_set(lobby, array['seats', p_seat_index::text], jsonb_build_object('name', p_name, 'uid', p_uid)),
           updated_at = now()
       where room_code = p_room_code
         and (lobby->'seats'->p_seat_index->>'uid') is null;
     get diagnostics v_updated = row_count;
     return v_updated > 0;
   end;
   $$;

   grant execute on function claim_seat(text, int, text, text) to anon, authenticated;
   ```

4. **Project Settings (⚙️) → API.** Copy the **Project URL** and the **anon public**
   key.
5. Open `index.html`, find `SUPABASE_CONFIG` (near the start of the `<script>` tag),
   and replace `url` and `anonKey` with the values you copied.

Worth knowing: the **anon** key is not a secret - it's normal and safe to embed it
directly in client code (and to publish it in a public GitHub repo). Security is based
on **Row Level Security** and on the room code itself not being guessable - not on
hiding the key. If you want stronger protection, you can lengthen the room code in
the code (`generateRoomCode` in `index.html`).

## Game settings

Clicking the ⚙️ settings button on the main screen opens a panel with:

| Setting | Range | Default |
|---|---|---|
| Number of players | 2–10 | 2 |
| Budget per player | $5–$100 | $20 |
| Roster size (players each person buys) | 5–12 | 5 |
| Number of leagues in the pool | 1 up to however many league files exist | 1 |

When you pick **more than one league**, a separate card opens in the lobby for each
league - with its own league selector and its own list of teams to check/uncheck. The
players from every selected league get mixed together into one pool for the auction.

## Persistence

**In local mode**, the game saves two kinds of information to the browser's
`localStorage`:

1. **Lobby settings** (player names, advanced settings, which leagues/teams are
   checked) - always saved, even after a game ends or after returning to the main
   screen, so you don't have to set everything up again each time.
2. **Active game state** - if the game closes by accident (page refresh, browser
   crash) mid-auction, the next time you open the page it will offer to restore
   exactly where you left off. A deliberate exit (the "back to main screen" or "new
   game" button) clears the saved game state, so you're never offered a restore for
   a game you already abandoned on purpose.

**In online mode**, the actual state lives in Supabase, not on your device, so
refreshing the page just reconnects and re-syncs automatically - no separate restore
prompt needed. To make it easy to return to the last room you were in, the game only
saves the room code locally (`auction_last_room_v1` in localStorage), and shows a
"back to last room" button on the main screen if one exists.

## Auto-fill at the end of the auction

Once only one player is left with an incomplete roster (everyone else has finished),
there's no point continuing the competitive auction - they get all the remaining
players for $0. This happens step by step: each time, the next player about to join
the roster is shown, and clicking "Confirm, next" locks them in and reveals the next
one in line, until the roster is full.

## Technology

Vanilla HTML + CSS + JavaScript, no framework. Two external dependencies, both via
CDN: `html2canvas` for the shared screenshot at the end of the game, and
`@supabase/supabase-js` for syncing online games (only loaded/used once online mode
is configured).