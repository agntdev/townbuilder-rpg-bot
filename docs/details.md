# TownBuilder RPG Bot — Details Spec

Concrete per-command behaviour for the bot defined in `docs/design.md`.
This doc is the contract each `dev` task must implement against.

## Global behaviour

- **Scope**: group chats only. In a private chat the bot replies
  `"This bot works in group chats."` to every command and ignores all
  other updates.
- **Bot filter**: any Update whose `from.is_bot === true` (or whose
  `message.from.is_bot` for reactions) is a no-op. The bot never
  generates resources from its own messages or its own reactions.
- **Persistence key**: `town:<chatId>`. One town per group chat.
- **Idempotency**: re-invoking `/start` does not reset resources.
- **Errors**: unhandled errors go to `bot.catch()`; the user sees a
  short toast `"Something went wrong."` and the bot keeps polling.

## Resources

| Resource | Source | Increment | Notes |
|---|---|---|---|
| Gold  | every non-bot `message` in the group | `+1` per message | Service messages (member join/leave) are excluded. |
| Food  | every non-bot `message_reaction` update (any emoji added) | `+1` per reaction | A reaction being changed or removed does not grant food. |

Resource gains are silent — no reply sent on each message/reaction.

## Commands

### `/start`

- **Trigger**: `bot.command("start", ...)`.
- **Pre**: none.
- **Effect**:
  1. `getOrCreateTown(ctx.chat.id)` — create if missing.
  2. Reply with welcome block (see below).
- **Reply** (new town):
  ```
  🏰 Welcome to TownBuilder!
  This group's town has been founded.
  Gold +1 per message, Food +1 per reaction.
  Use /spend to build something.
  ```
- **Reply** (existing town):
  ```
  🏰 The town is still standing.
  Gold: <n>  Food: <n>
  Use /spend to build something.
  ```
- **Out of scope**: `/reset` is not implemented in v1.

### `/help`

- **Trigger**: `bot.command("help", ...)`.
- **Reply**:
  ```
  Commands:
  /town  — show resources & buildings
  /spend — open build menu
  /help  — this message
  ```

### `/town`

- **Trigger**: `bot.command("town", ...)`.
- **Effect**: load town, reply with status block.
- **Reply**:
  ```
  🏰 Town Status
  Gold:  <n>
  Food:  <n>
  Built: <comma-separated building names, or "nothing yet">
  ```

### `/spend`

The only stateful flow. Uses `ctx.session.step` (per-chat session).

- **Trigger**: `bot.command("spend", ...)`.
- **Pre**: `ctx.session.step` reset to `awaiting_building` (string
  literal, namespaced if other flows are added later).
- **Effect**: send a new message with the menu, persist the message id
  in session so a future `/spend` can edit it instead of spamming.
- **Menu message**:
  ```
  💰 Gold: <n>   🍞 Food: <n>
  Pick a building to construct:
  ```
  With an `InlineKeyboardMarkup`:
  - One button per building whose label is `"<Name>  (<costG>g<cF>f)"` and
    `callback_data = "build:<key>"`.
  - A final row with `"Cancel"` → `callback_data = "build:cancel"`.
  - Buttons are omitted (or shown as a single row with `"—"`) if there
    are no buildings.
- **Callback handler**: `bot.callbackQuery(/^build:/, ...)`.
  1. `answerCallbackQuery()` first (always).
  2. Branch on `data`:
     - `build:cancel` → `editMessageText("Build menu closed.")`, clear
       `ctx.session.step`.
     - `build:<key>` → look up `BUILDINGS`. If `town.unlocked.includes(key)`
       → toast `"Already built."` (no edit). If `town.gold < cost.gold ||
       town.food < cost.food` → toast `"Not enough resources."` (no edit).
       Otherwise:
       - Deduct cost, push key to `unlocked`, save town.
       - `editMessageText` the menu message to the construction
       announcement:
         ```
         🏗️ The <Name> has been constructed!
         ```
       - Clear `ctx.session.step`.
- **Cancel** can also be done implicitly by running `/spend` again — the
  new menu replaces the old one.

## Event handlers

| Source event | Filter | Effect |
|---|---|---|
| `message` (group, non-bot) | `ctx.chat.type ∈ {"group","supergroup"}` and `!ctx.from?.is_bot` and `!ctx.message?.service` | `town.gold += 1` |
| `message_reaction` (group) | non-bot reactor | `town.food += 1` |
| `new_chat_members` | includes this bot | send `/start`-equivalent welcome to the chat |
| `my_chat_member` status = `kicked`/`left` | n/a | log; do not delete the town record (recovery on re-add) |

## BotSpec test coverage (assigned to integration task)

The following dialogs must be covered by `tests/specs/*.json`:

1. `start_new.json` — first `/start` in a fresh group → welcome.
2. `start_existing.json` — second `/start` → status with current resources.
3. `town.json` — send a non-bot message, then `/town` → gold incremented.
4. `spend_affordable.json` — seed resources via `town.json` setup, run
   `/spend`, tap a building the town can afford → construction reply,
   next `/town` shows the building in `Built:`.
5. `spend_broke.json` — `/spend` with insufficient resources, tap a
   building → toast `"Not enough resources."`, no deduction.
6. `spend_cancel.json` — `/spend` → tap Cancel → menu closed message.
7. `private_chat_blocked.json` — `/start` in a private chat → blocked
   message.

These specs are the review gate for the dev → tests phase.

## Data types (for reference)

```ts
type ChatId = number;

interface Town {
  chatId: ChatId;
  gold: number;
  food: number;
  unlocked: string[];   // building keys
  createdAt: number;    // ms epoch
}

interface Building {
  key: string;          // e.g. "bakery"
  name: string;         // e.g. "Bakery"
  cost: { gold: number; food: number };
}

const BUILDINGS: readonly Building[] = [
  { key: "town_hall", name: "Town Hall", cost: { gold:  5, food:  0 } },
  { key: "bakery",    name: "Bakery",    cost: { gold:  3, food:  5 } },
  { key: "market",    name: "Market",    cost: { gold: 10, food:  5 } },
  { key: "barracks",  name: "Barracks",  cost: { gold: 15, food: 10 } },
] as const;
```
