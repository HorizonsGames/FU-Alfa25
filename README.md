# Fractured Universe — Multiplayer Backend

Accounts, persistent game state, and a live WebSocket relay for PvP and
boss battles. Built for Heroku + Heroku Postgres.

## What this is (and isn't)

**Is:**
- Real accounts (bcrypt-hashed passwords, session tokens) replacing the
  decorative client-side `auth.db`
- Persistent player state in Postgres (`state` JSONB column), replacing
  localStorage — same shape, so the client mostly just needs to send/receive
  a blob instead of reading/writing localStorage directly
- A WebSocket **relay** for live battle rooms: everyone in the same room
  sees everyone else's ship-state updates in real time. This is what makes
  a "boss at center, all contributing players around it" live view and live
  PvP possible — the server just needs one room per fight, and every
  contributing client's `ship_state` messages get broadcast to the rest.

**Isn't (yet):**
- **Not cheat-proof.** Each client still computes its own combat locally
  and broadcasts the result — same as the existing single-player
  `animateBattle()`. A modified client could lie about its ship-state.
  Making the server authoritative for combat resolution is the natural next
  phase, once this relay layer is proven out.
- **Not horizontally scalable as-is.** Room membership lives in one Node
  process's memory. Fine for a single Heroku dyno. If you ever run more
  than one web dyno, players on different dynos won't see each other's
  battles — you'd need to add Heroku Redis as a pub/sub layer between
  dynos. Noted in `src/ws/hub.js` too so it isn't a surprise later.
  For now: **keep this app at 1 web dyno.**
- **Doesn't touch the client yet.** This is the server + a connector module
  (`client/multiplayerClient.js`) you can wire into `index.html`
  incrementally. The 20k-line game client itself hasn't been modified —
  that's the next phase once you're happy with this foundation.

## Local development

```bash
cp .env.example .env        # fill in a real local Postgres URL + a random SESSION_SECRET
npm install
npm start                   # http://localhost:3000, WebSocket at /ws
```

Run `node smoke-test.js` any time to sanity-check the whole stack (accounts,
state save/load, guilds, and the WebSocket relay) without needing a real
Postgres — it spins up an in-memory one just for the test run.

## Deploying to Heroku

```bash
heroku create your-fu-backend
heroku addons:create heroku-postgresql:essential-0
heroku config:set SESSION_SECRET=$(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
heroku config:set ALLOWED_ORIGINS=https://your-game-url.example.com
git subtree push --prefix server heroku main   # if this server/ folder lives inside a bigger repo
# or, if this is its own repo:
git push heroku main
```

`DATABASE_URL` is set automatically by the Postgres addon — you don't set it
yourself. The schema (`src/schema.sql`) is applied automatically on boot,
every time the dyno starts, and is safe to re-run (`CREATE TABLE IF NOT
EXISTS`).

**Set `ALLOWED_ORIGINS` before you consider this production-ready.** Left
unset, the server allows requests from any origin, which is fine for local
testing but means any website could make authenticated calls on a logged-in
player's behalf once this is public.

## API reference

### REST (`/api/...`)

| Method | Path | Auth | Body | Notes |
|---|---|---|---|---|
| POST | `/api/auth/register` | — | `{username, password, email?, race?}` | Returns `{token, player}` |
| POST | `/api/auth/login` | — | `{username, password}` | Returns `{token, player}` |
| POST | `/api/auth/logout` | Bearer | — | Invalidates the session token |
| GET  | `/api/auth/me` | Bearer | — | Returns `{player}` |
| GET  | `/api/state` | Bearer | — | Returns `{state}` — the player's full save blob |
| PUT  | `/api/state` | Bearer | `{state}` | Overwrites the save blob (2MB cap) |
| POST | `/api/guilds` | Bearer | `{name, tag}` | Creates a guild, creator becomes leader |
| POST | `/api/guilds/:id/join` | Bearer | — | Joins a guild |
| GET  | `/api/guilds/:id/roster` | Bearer | — | Lists members |

Auth is `Authorization: Bearer <token>` on every protected route.

### WebSocket (`/ws?token=<sessionToken>`)

Connect, then send/receive JSON messages. See the full protocol writeup and
rationale in `src/ws/battleRoom.js`. Quick summary:

**Client → server:**
- `{type:'join_room', roomId, roomType}` — `roomType` is `'pvp'` or `'boss'`
- `{type:'leave_room', roomId}`
- `{type:'ship_state', roomId, payload}` — send every animation tick
- `{type:'battle_event', roomId, payload}` — discrete events (ship destroyed, victory)
- `{type:'chat', roomId, payload:{text}}`

**Server → client:**
- `{type:'room_joined', roomId, members:[{playerId, username}]}`
- `{type:'member_joined'|'member_left', roomId, playerId, username?}`
- `{type:'ship_state'|'battle_event'|'chat', roomId, playerId, username, payload}`
- `{type:'error', message}`

**Room ID convention:** `pvp:<matchId>` for a 1v1, `boss:<bossInstanceId>`
for a world/guild boss — every contributing player joins the same boss room,
which is exactly the mechanism for the "boss at center, everyone around it"
view: the server just relays; the client lays out whoever's in `members` in
a ring around the boss sprite.

## Suggested next steps (client-side, not started yet)

1. Swap `createPlayer`/`authenticatePlayer` in `index.html` for
   `MPClient.register`/`.login`, and swap localStorage save/load for
   `MPClient.saveState`/`.loadState`.
2. For a live boss fight: on entering the boss screen, call
   `mp.joinRoom('boss:' + bossId, 'boss')`, listen via `mp.onMemberJoined`
   to know who else is there, and call `mp.sendShipState(...)` from inside
   the existing battle animation loop instead of (or alongside) the current
   local-only simulation. Lay out `members` in a ring around the boss using
   the existing `drawShip` — the boss sprite goes at the canvas center.
3. For live PvP: same idea with a `pvp:<matchId>` room and exactly 2
   participants instead of a ring.
4. Once that's working end-to-end, consider moving combat resolution itself
   onto the server so it becomes the source of truth (closes the cheating
   gap noted above).
