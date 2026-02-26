# RS-SDK — Copilot Agent Instructions

RS-SDK is a RuneScape automation toolkit. You control bots via MCP tools (`execute_code`, `list_bots`, `disconnect_bot`) that are available through the MCP server configured in `.vscode/mcp.json`.

## Architecture

```
Your Code → MCP (execute_code) → BotManager → BotSDK → Gateway (WS) → BotClient (browser) → Game Server
```

- **MCP Server** (`mcp/server.ts`): Exposes tools. Auto-connects bots on first use.
- **BotManager** (`mcp/api/index.ts`): Manages bot connections. Reads credentials from `bots/{name}/bot.env`.
- **BotSDK** (`sdk/`): Low-level protocol. State queries, raw actions.
- **BotActions** (`sdk/actions.ts`): High-level domain-aware actions that wait for effects.
- **Gateway** (`server/gateway/`): WebSocket bridge between SDK and browser client.

## Quick Start

1. Create a bot: `bun bots/create-bot.ts mybot`
2. (Optional) Start local server: see README for `server/engine`, `server/webclient`, `server/gateway`
3. Use the `execute_code` MCP tool with `bot_name: "mybot"` and your TypeScript code

## MCP Tools Available

### `execute_code`
Execute TypeScript on a connected bot. Globals: `bot` (BotActions), `sdk` (BotSDK).
```json
{"bot_name": "mybot", "code": "await bot.chopTree(); return sdk.getInventory();", "timeout": 5}
```
Returns: console output + result + formatted world state.

### `list_bots`
List all connected bots with connection status.

### `disconnect_bot`
Disconnect a bot: `{"name": "mybot"}`

## Bot API (High-Level) — `bot.*`

Use these for most tasks. They wait for effects to complete.

### Movement
- `bot.walkTo(x, z, tolerance?)` — Walk to coordinates

### Gathering
- `bot.chopTree(target?)` — Chop tree, wait for logs
- `bot.mineRock(target?)` — Mine rock, wait for ore
- `bot.fish(target?)` — Fish at spot, wait for catch

### Combat
- `bot.attackNpc(target, timeout?)` — Attack and wait for kill
- `bot.eatFood(food)` — Eat food from inventory

### Items & Inventory
- `bot.pickupItem(target)` — Pick up ground item
- `bot.equipItem(item)` / `bot.unequipItem(item)`
- `bot.dropItem(item)`
- `bot.useItemOnLoc(item, loc)` / `bot.useItemOnNpc(item, npc)` / `bot.useItemOnItem(item1, item2)`

### Banking
- `bot.openBank(timeout?)` — Walk to banker and open bank
- `bot.depositItem(item, amount?)` / `bot.depositAll()`
- `bot.withdrawItem(item, amount?)`

### Shopping
- `bot.openShop(target?)` — Talk to shopkeeper
- `bot.buyFromShop(item, amount?)` / `bot.sellToShop(item, amount?)`

### Crafting
- `bot.smithAtAnvil(product, options?)`
- `bot.fletchLogs(product?)` / `bot.craftLeather(product?)`
- `bot.cookFood(food?, range?)`
- `bot.burnLogs(logs?)`

### Other
- `bot.openDoor(target?)` — Open doors/gates
- `bot.buryBones(bones)` — Bury bones
- `bot.castSpell(spellName, target?)` — Cast magic spell
- `bot.dismissBlockingUI()` — Close dialogs/level-ups

All methods return `{success: boolean, message?: string, ...}`. Always check `result.success`.

## SDK API (Low-Level) — `sdk.*`

Direct state access and raw protocol commands.

### State Queries (synchronous, from cache)
- `sdk.getState()` → full world state snapshot
- `sdk.getInventory()` → `InventoryItem[]`
- `sdk.findInventoryItem(pattern)` → find by name/regex
- `sdk.getNearbyNpcs()` / `sdk.findNearbyNpc(pattern)`
- `sdk.getNearbyLocs()` / `sdk.findNearbyLoc(pattern)` — "locs" are interactable world objects (trees, rocks, banks, etc.)
- `sdk.getGroundItems()` / `sdk.findGroundItem(pattern)`
- `sdk.getSkill(name)` → `{name, level, baseLevel, experience}`
- `sdk.getEquippedItems()`

### Raw Actions (resolve on server acknowledgment, NOT effect completion)
- `sdk.sendWalk(x, z, running?)`
- `sdk.sendInteractLoc(x, z, locId, option)`
- `sdk.sendInteractNpc(npcIndex, option)`
- `sdk.sendTakeGroundItem(x, z, itemId)`
- `sdk.sendUseItem(slot, option)`

### Utility
- `sdk.sendScreenshot()` — returns base64 screenshot
- `sdk.waitForCondition(predicate, timeout?)` — wait for state match
- `sdk.findPath(startX, startZ, endX, endZ)` — pathfinding

## Key Gotchas

### PlayerState has NO hp field
Use `sdk.getSkill('hitpoints')` for current HP. `player.combat.inCombat` tells you if in combat.

### Locations ("locs") are world objects, not coordinates
Trees, rocks, banks, doors — these are all "locs". Use `sdk.findNearbyLoc(/^tree$/i)` to find them.

### Walking
- The map uses a custom coordinate system. Always use `bot.walkTo()` or `sdk.sendWalk()`.
- For long distances, break into waypoints — you can only walk within view distance per tick.
- Open doors/gates BEFORE walking through them: `await bot.openDoor()` then `await bot.walkTo(x, z)`.

### Banking
- Lumbridge does NOT have a bank. Closest banks: Draynor Village, Varrock.
- Always use `bot.openBank()` which handles walking to the banker.
- Deposit/withdraw only work while bank is open.

### Combat
- Dark Wizards south of Varrock (around x:3222, z:3394) are dangerous at low levels. Avoid or prepare food.
- `bot.attackNpc()` waits for the kill. Always have food for tough fights.
- Check `sdk.getSkill('hitpoints').level` during combat to know when to eat.

### Chat is OFF by default
Bot env has `SHOW_CHAT=false` to prevent prompt injection via in-game chat. Only enable if needed.

### Inventory is 28 slots
Always check inventory space before gathering. Use `sdk.getInventory().length < 28` or bank when full.

## Example: Woodcutting Loop

```typescript
// Chop trees until inventory full, bank, repeat
while (true) {
  const inv = sdk.getInventory();
  if (inv.length >= 28) {
    await bot.openBank();
    await bot.depositAll();
    // Walk back to trees
    await bot.walkTo(3180, 3452);
  }
  const result = await bot.chopTree();
  if (!result.success) {
    console.log('Chop failed:', result.message);
    await bot.walkTo(3180, 3452); // Reposition
  }
}
```

## Example: Combat Training

```typescript
// Kill goblins, eat food when low HP
while (true) {
  const hp = sdk.getSkill('hitpoints');
  if (hp && hp.level < 10) {
    const food = sdk.findInventoryItem(/shrimp|bread|meat/i);
    if (food) await bot.eatFood(food.name);
    else { console.log('Out of food!'); break; }
  }
  const result = await bot.attackNpc(/goblin/i);
  if (!result.success) {
    console.log('No goblins nearby');
    await new Promise(r => setTimeout(r, 2000));
  }
}
```

## File Reference
- `mcp/api/bot.ts` — Full BotActions method list with descriptions
- `mcp/api/sdk.ts` — Full BotSDK method list with types
- `sdk/actions.ts` — BotActions implementation
- `sdk/types.ts` — All TypeScript types
- `learnings/*.md` — Detailed game knowledge per skill (banking, combat, cooking, fishing, mining, smithing, walking, woodcutting, etc.)
