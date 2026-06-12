# TownBuilder RPG Bot — Design

This doc defines the architecture, command set, and conversation flows for the
Telegram bot described in `docs/general.md`. Scope is intentionally limited to
the features in the General doc — no quests, no PvP, no inter-group
competition.

## 1. Architecture

```
Telegram group chat
        │
        │  HTTP (Updates)
        ▼
┌─────────────────────────┐
│  grammY bot (long poll) │
│  built via @agntdev/    │
│  bot-toolkit            │
│  (createBot + sessions) │
└──────────┬──────────────┘
           │
           │  in-process calls
           ▼
┌─────────────────────────┐
│  TownService            │
│  - getOrCreateTown      │
│  - addGold              │
│  - addFood              │
│  - listBuildings        │
│  - spend (construct)    │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Persistence (KV-style) │
│  key: chat:<chatId>     │
│  val: Town JSON         │
│  { gold, food,          │
│    unlocked: [..] }     │
└─────────────────────────┘
```

- **Entry point**: `src/index.ts` exports `makeBot()` per the toolkit contract.
- **Runtime**: Node.js 18+, long polling for dev, webhook-ready for prod.
- **Session**: per-chat state for multi-step `/spend` flow (`awaiting_building`
  step). No per-user session needed for v1.
- **Persistence**: chat-keyed record. One town per group chat, identified by
  `ctx.chat.id`. Default storage = in-memory `MemorySessionStorage` for dev;
  swap for a real adapter before publish.
- **Bot token**: `process.env.BOT_TOKEN`. Never in source.

### Data model

```ts
interface Town {
  chatId: number;
  gold: number;          // +1 per non-bot message
  food: number;          // +1 per reaction added
  unlocked: string[];    // building keys already constructed
  createdAt: number;     // unix ms
}

const BUILDINGS: Building[] = [
  { key: "town_hall",  name: "Town Hall", cost: { gold: 5, food: 0 } },
  { key: "bakery",     name: "Bakery",    cost: { gold: 3, food: 5 } },
  { key: "market",     name: "Market",    cost: { gold: 10, food: 5 } },
  { key: "barracks",   name: "Barracks",  cost: { gold: 15, food: 10 } },
];

interface Building {
  key: string;
  name: string;
  cost: { gold: number; food: number };
}
```

## 2. Full Command Set

| Command | Scope | Purpose |
|---|---|---|
| `/start` | group | Welcome message; auto-creates the town's record for this chat. |
| `/help`  | group | Show available commands. |
| `/town`  | group | Show current resources and constructed buildings. |
| `/spend` | group | Open the build menu to spend resources on a building. |

Group-only. In private chats the bot replies with "This bot works in group
chats."

Resource gains are **passive** — no command. They happen automatically on
non-bot messages and reactions.

## 3. Conversation / UX Flows

### 3.1 First-time setup (`/start` in a group)

```
User:  /start
Bot:   🏰 Welcome to TownBuilder!
       This group's town has been founded.
       Gold +1 per message, Food +1 per reaction.
       Use /spend to build something.
```

If the town already exists:

```
Bot:   🏰 The town is still standing.
       Gold: 12  Food: 7
       Use /spend to build something.
```

### 3.2 Passive resource gain

- Any non-bot message in the group: `town.gold += 1`.
- Any reaction added to a message: `town.food += 1`.
- Bot's own messages and bot-added reactions are ignored (filter on
  `from.is_bot` and `message.from.is_bot`).
- No notification to chat on every gain — that would spam. Resource totals
  surface only in `/town` and `/spend` outputs.

### 3.3 `/town` — view status

```
Bot:   🏰 Town Status
       Gold:  12
       Food:   7
       Built:  Town Hall, Bakery
```

### 3.4 `/spend` — build menu (the only multi-step flow)

Step 1 — show resources + building list as inline keyboard:

```
Bot:   💰 Gold: 12   🍞 Food: 7
       Pick a building to construct:
       [ Town Hall  (5g)        ]
       [ Bakery     (3g, 5f)   ]
       [ Market     (10g, 5f)  ]
       [ Barracks   (15g, 10f) ]
       [ Cancel ]
```

Session step: `awaiting_building`.

Step 2 — handle callback:

- **Affordable + not built** → deduct cost, append to `unlocked`, reply in
  group:
  ```
  Bot:   🏗️ The Bakery has been constructed!
  ```
  Clear session step.
- **Already built** → toast: `"Already built."`, keep menu open.
- **Not enough resources** → toast: `"Not enough resources."`, keep menu
  open.
- **Cancel** → clear session, delete the menu message.

### 3.5 `/help`

```
Bot:   Commands:
       /town  — show resources & buildings
       /spend — open build menu
       /help  — this message
```

## 4. Event handlers (non-command)

These are not commands but Update listeners wired at bot setup:

| Trigger | Effect |
|---|---|
| `message` (non-bot, in group) | `town.gold += 1` |
| `message_reaction` (non-bot user) | `town.food += 1` |
| `new_chat_members` (bot added) | Send `/start`-equivalent welcome |
| `left_chat_member` (bot removed) | Optionally log; town record is dropped only on explicit `/reset` (out of v1 scope — see Non-Goals) |

## 5. Out of scope (matches General non-goals)

- Inter-group competition
- Random events / quests
- PvP
- Real-money transactions
- Complex building interactions beyond the construction notification
- Player economy / trading

## 6. Testability

- All flows reachable from a chat-level Update.
- `/spend` flow is the only stateful one — covered by a `BotSpec` that
  triggers the command, taps a building callback, and asserts the reply.
- Resource gain is covered by sending a message Update and asserting the
  next `/town` shows incremented gold.
