# miowawa.de Bot Guide

Build bots for [miowawa.de](https://miowawa.de) in **JavaScript** or **Python**. A bot connects with a token over Socket.IO (real-time) and REST (profile, rooms).

- [1. Get a token & go online](#1-get-a-token--go-online)
- [2. Name, picture, room name](#2-name-picture-room-name)
- [3. Create a room](#3-create-a-room)
- [4. Receive & send messages](#4-receive--send-messages)
- [5. Colored messages](#5-colored-messages)
- [6. Slash commands](#6-slash-commands)
- [7. Room settings](#7-room-settings)
- [8. Playing: words for a syllable](#8-playing-words-for-a-syllable)
- [9. All events](#9-all-events)
- [Examples (JS + Python)](#examples)

---

## 1. Get a token & go online

1. A site admin opens **Admin panel → Bots**, enters a **name** and a **color**, and creates the bot.
2. The token (`miowawabot_...`) is shown **once**. Copy it.
3. Keep it secret: put it in a `.env` file, never in Git.

```env
MIOWAWA_URL=https://miowawa.de
MIOWAWA_BOT_TOKEN=miowawabot_xxxxxxxx
```

Connect with the token in the Socket.IO `auth` object. REST calls use the header `x-bot-token`. Both also work while the site is in maintenance mode.

```js
// JavaScript
const { io } = require("socket.io-client");
const socket = io(process.env.MIOWAWA_URL, { auth: { botToken: process.env.MIOWAWA_BOT_TOKEN } });
socket.on("connect", () => console.log("online"));
```

```python
# Python
import socketio, os
sio = socketio.Client()
sio.connect(os.environ["MIOWAWA_URL"], auth={"botToken": os.environ["MIOWAWA_BOT_TOKEN"]})
```

Install: `npm i socket.io-client dotenv` / `pip install "python-socketio[client]" websocket-client requests`

## 2. Name, picture, room name

`PUT /api/me` (header `x-bot-token`):

| Field | Rules |
|---|---|
| `name` | Display name |
| `avatar` | `data:image/png;base64,...` (png / jpeg / webp, max 200,000 characters) |
| `roomName` | Name of rooms this bot creates, max 20 characters |

```js
await fetch(`${URL}/api/me`, {
  method: "PUT",
  headers: { "content-type": "application/json", "x-bot-token": TOKEN },
  body: JSON.stringify({ name: "Kitsu", roomName: "[EN] KitsuBot", avatar: "data:image/png;base64,..." }),
});
```

```python
requests.put(f"{URL}/api/me", headers={"x-bot-token": TOKEN},
             json={"name": "Kitsu", "roomName": "[EN] KitsuBot"})
```

## 3. Create a room

`POST /api/rooms` with `{ "visibility": "public" }` (or `"private"`) returns `{ "code": "ABCD" }`. The room gets the bot's `roomName`. Max **3 rooms per host**. The creator is the **host**.

Join a room (also needed for existing rooms) and sit down to play:

```js
const { code } = await (await fetch(`${URL}/api/rooms`, {
  method: "POST",
  headers: { "content-type": "application/json", "x-bot-token": TOKEN },
  body: JSON.stringify({ visibility: "public" }),
})).json();

socket.emit("room:join", { code }, (res) => {
  // res: room, people, messages, you, game ...
  socket.emit("room:seat", { seated: true }); // play instead of spectating
});
```

`GET /api/rooms/:code` returns info about a room. Leave with `socket.emit("room:leave")`. Save the room code (e.g. in a JSON file) to reuse the room after a restart.

## 4. Receive & send messages

```js
socket.on("chat:message", (m) => console.log(m.name, m.text));
socket.emit("chat:send", { text: "Hello!" });
```

```python
@sio.on("chat:message")
def on_msg(m): print(m["name"], m["text"])
sio.emit("chat:send", {"text": "Hello!"})
```

Bot messages have no rate limit and no character limit (max 20 lines) and show the 🤖 "MIOWAW.DE Bot" icon. System messages (e.g. `joined the room`) also arrive as `chat:message`, so you can send welcome messages.

## 5. Colored messages

Bot messages are automatically shown in the **bot color** you chose in the Admin panel (Bots). To change the color, change it there. No markup is needed.

## 6. Slash commands

The server only knows `/c` and `/suicide`. A bot implements its own commands by reading chat messages that start with `/`:

```js
socket.on("chat:message", (m) => {
  if (!m.text?.startsWith("/")) return;
  const [cmd, ...args] = m.text.slice(1).split(" ");
  if (cmd === "ping") socket.emit("chat:send", { text: "pong" });
  if (cmd === "say") socket.emit("chat:send", { text: args.join(" ") });
});
```

```python
@sio.on("chat:message")
def cmd(m):
    t = m.get("text") or ""
    if not t.startswith("/"): return
    name, *args = t[1:].split(" ")
    if name == "ping": sio.emit("chat:send", {"text": "pong"})
```

## 7. Room settings

Only the **host** can change settings, and only while the game is `waiting` / in countdown.

```js
socket.emit("game:settings", {
  startLives: 2, maxLives: 3, minPlayers: 2, maxPlayers: 16,
  minTurn: 5, wppMode: false, wpp: 500, promptAge: 0, bonus: 0, lang: "en",
});
socket.emit("room:settings", { visibility: "public" });
socket.emit("game:startNow");
```

Languages: `en, de, fr, es, it, pt-br, nah`. Difficulty is fixed to beginner. If you get `Only the host can change the rules.`, the bot is not host: create the room with the bot instead of joining someone else's.

Host-only: `room:host`, `room:moderator`, `room:kick`, `room:ban`, `room:unban`.

## 8. Playing: words for a syllable

The site does **not** give the bot a dictionary. Bring your own word list (one word per line, lower case), e.g. `words-en.txt`.

```js
const words = require("fs").readFileSync("words-en.txt", "utf8").split("\n");
const used = new Set();
function pick(syl) {
  const list = words.filter((w) => w.includes(syl) && !used.has(w));
  return list[Math.floor(Math.random() * list.length)];
}

socket.on("game:state", (g) => {
  // g.currentPlayerId === your id and g.syllable is set -> your turn
  if (g.currentPlayerId !== myId || !g.syllable) return;
  const w = pick(g.syllable);
  if (!w) return;
  used.add(w);
  socket.emit("game:typing", { text: w });
  socket.emit("game:word", { word: w });
});
```

Reset `used` when a new round starts. Wrong or already used words are rejected (`game:word` gets a rejection state), then try the next word. Your own id comes from the `room:join` ack (`you`).

## 9. All events

**Bot → server**

| Event | Payload |
|---|---|
| `room:join` | `{ code }` (ack: room, people, messages, you, game) |
| `room:leave` | – |
| `room:seat` | `{ seated: true/false }` |
| `game:startNow` | – |
| `game:settings` | `{ startLives, maxLives, minPlayers, maxPlayers, minTurn, wppMode, wpp, promptAge, bonus, lang }` (host) |
| `room:settings` | `{ visibility }` (host) |
| `game:word` | `{ word }` |
| `game:typing` | `{ text }` |
| `chat:send` | `{ text }` |
| `room:host` `room:moderator` `room:kick` `room:ban` `room:unban` | host only |
| `me:refresh` | reload profile |

**Server → bot**

`game:state` · `game:typing` · `game:word` · `game:explode` · `game:life` · `game:birthday` · `room:update` · `room:person` · `chat:message` · `rooms:list` · `rooms:stats` · `online` · `room:kicked` · `site:banned`

## Examples

### JavaScript (`bot.js`)

```js
require("dotenv").config();
const { io } = require("socket.io-client");
const fs = require("fs");
const URL = process.env.MIOWAWA_URL, TOKEN = process.env.MIOWAWA_BOT_TOKEN;
const H = { "content-type": "application/json", "x-bot-token": TOKEN };
const words = fs.readFileSync("words-en.txt", "utf8").split("\n").filter(Boolean);
const used = new Set();
let myId = null;

(async () => {
  await fetch(`${URL}/api/me`, { method: "PUT", headers: H,
    body: JSON.stringify({ name: "MyBot", roomName: "[EN] MyBot" }) });
  const { code } = await (await fetch(`${URL}/api/rooms`, { method: "POST", headers: H,
    body: JSON.stringify({ visibility: "public" }) })).json();

  const socket = io(URL, { auth: { botToken: TOKEN } });
  socket.on("connect", () => {
    socket.emit("room:join", { code }, (res) => {
      myId = res.you?.id;
      socket.emit("game:settings", { startLives: 2, maxLives: 3, minPlayers: 2, maxPlayers: 16, minTurn: 5, lang: "en" });
      socket.emit("room:seat", { seated: true });
    });
  });
  socket.on("chat:message", (m) => {
    if (m.text === "/ping") socket.emit("chat:send", { text: "pong" });
  });
  socket.on("game:state", (g) => {
    if (g.currentPlayerId !== myId || !g.syllable) return;
    const w = words.find((x) => x.includes(g.syllable) && !used.has(x));
    if (!w) return;
    used.add(w);
    socket.emit("game:word", { word: w });
  });
})();
```

### Python (`bot.py`)

```python
import os, requests, socketio
from dotenv import load_dotenv
load_dotenv()
URL, TOKEN = os.environ["MIOWAWA_URL"], os.environ["MIOWAWA_BOT_TOKEN"]
H = {"x-bot-token": TOKEN}
words = [w.strip() for w in open("words-en.txt", encoding="utf8") if w.strip()]
used, me = set(), {"id": None}

requests.put(f"{URL}/api/me", headers=H, json={"name": "MyBot", "roomName": "[EN] MyBot"})
code = requests.post(f"{URL}/api/rooms", headers=H, json={"visibility": "public"}).json()["code"]

sio = socketio.Client()

@sio.event
def connect():
    def joined(res):
        me["id"] = (res.get("you") or {}).get("id")
        sio.emit("game:settings", {"startLives": 2, "maxLives": 3, "minPlayers": 2,
                                   "maxPlayers": 16, "minTurn": 5, "lang": "en"})
        sio.emit("room:seat", {"seated": True})
    sio.emit("room:join", {"code": code}, callback=joined)

@sio.on("chat:message")
def on_chat(m):
    if (m.get("text") or "") == "/ping":
        sio.emit("chat:send", {"text": "pong"})

@sio.on("game:state")
def on_state(g):
    if g.get("currentPlayerId") != me["id"] or not g.get("syllable"): return
    w = next((x for x in words if g["syllable"] in x and x not in used), None)
    if w:
        used.add(w)
        sio.emit("game:word", {"word": w})

sio.connect(URL, auth={"botToken": TOKEN})
sio.wait()
```

> Field names such as `currentPlayerId` and `you.id` may differ slightly: print the `room:join` ack and the first `game:state` once (`console.log` / `print`) and adjust.

## Tips

- Run with pm2: `pm2 start bot.js --name my-bot`.
- Keep the token in `.env`. If it leaks, an admin should delete the bot and create a new one.
- Be nice: don't spam rooms.
