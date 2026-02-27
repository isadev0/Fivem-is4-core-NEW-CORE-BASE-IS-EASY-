# is4-core

> **Enterprise-grade, standalone FiveM framework — built from scratch.**  
> No ESX. No QBCore. No compromises.

**Version:** 4.0.0 &nbsp;|&nbsp; **Author:** Isa &nbsp;|&nbsp; **License:** Private &nbsp;|&nbsp; **oxmysql** required

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Requirements & Installation](#requirements--installation)
4. [Database Setup](#database-setup)
5. [Core Systems](#core-systems)
6. [API Layer](#api-layer)
7. [Module Reference](#module-reference)
8. [Writing Your Own Module](#writing-your-own-module)
9. [Event Bus Reference](#event-bus-reference)
10. [Config Philosophy](#config-philosophy)
11. [Performance & Design Notes](#performance--design-notes)

---

## Overview

`is4-core` is a fully custom FiveM server framework. It was written to replace ESX/QBCore with a cleaner, faster, and more modular architecture. Every system — from database writes to client events — has been designed with performance and maintainability in mind.

**Core principles:**

- **Loose coupling.** Modules never call each other directly. All inter-module communication goes through the Event Bus.
- **RAM-first data layer.** All player data is held in memory after load. The database is only written to in periodic batch saves (every 5 minutes) or on player disconnect. This eliminates per-action SQL queries.
- **Fault isolation.** Every API call is wrapped in `pcall`. A broken module cannot crash the server.
- **Rate-limited network.** All client → server communication is capped at 50 events/second per player to prevent event flooding and exploit attempts.
- **Multi-character support.** Players are identified by `license:XXXX:char:N`, allowing up to 2 characters per player by default (configurable).

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│                  LAYER 1: CORE                   │
│  event_bus  ·  module_loader  ·  datastore        │
│  player_manager  ·  api_registry  ·  network      │
│  permission  ·  profiler  ·  logger               │
├──────────────────────────────────────────────────┤
│                  LAYER 2: API                    │
│  jobs · items · money · weapons                   │
│  vehicles · players · notifications               │
├──────────────────────────────────────────────────┤
│                LAYER 3: MODULES                  │
│  is4-players · is4-hud · is4-needs               │
│  is4-garages · is4-housing · is4-motels          │
│  is4-policejob · is4-ambulancejob · is4-mechanic  │
│  is4-gangs · is4-crypto · is4-phone · is4-chat   │
│  is4-system · is4-anticheat · is4-others          │
│  is4-advancedresources                            │
├──────────────────────────────────────────────────┤
│             LAYER 4: CLIENT / SERVER             │
│  main.lua · modules_loader.lua · cron.lua         │
│  ui.lua · animations.lua · commands.lua           │
├──────────────────────────────────────────────────┤
│               LAYER 5: ASSETS                    │
│  /docs · /assets · /ui                            │
└──────────────────────────────────────────────────┘
```

The framework boots in strict order: Core → API → Modules. No module can be called before its declared dependencies are loaded.

---

## Requirements & Installation

**Dependencies:**
- [oxmysql](https://github.com/overextended/oxmysql) — must be started before is4-core

**`server.cfg` load order:**
```
ensure oxmysql
ensure is4-core
```

**Folder placement:**
```
/resources/
  └── is4-core/
        ├── fxmanifest.lua
        ├── is4-core.sql
        ├── core/
        ├── api/
        ├── modules/
        ├── server/
        └── client/
```

---

## Database Setup

Import `is4-core.sql` into your MySQL/MariaDB database before starting the server. The file is included in the resource root.

```bash
mysql -u root -p your_database < is4-core.sql
```

The SQL file creates the following tables:

| Table | Purpose |
|---|---|
| `players` | Core player record — money (JSON), job, inventory, metadata, status |
| `characters` | Identity data — name, DOB, gender, bloodtype, fingerprint |
| `inventories` | Extended inventory store for stashes, vehicles, drops |
| `bank_transactions` | Full transaction history (deposits, withdrawals, transfers, salary) |
| `player_vehicles` | Owned vehicles — plate, model, mods (JSON), fuel, damage, garage state |
| `properties` | Housing ownership — price tier, locks, stash contents, furniture |
| `gangs` | Gang registry — name, label, boss identifier, bank balance |
| `gang_members` | Gang membership — identifier, gang, grade |
| `societies` | Job society bank accounts (police, ambulance, mechanic, etc.) |
| `crypto_wallets` | Crypto coin balances per player |
| `anticheat_logs` | Flagged events with player info and coordinates |
| `server_logs` | Framework-level log output (INFO / WARN / ERROR / TRACE) |

> **Note:** The `identifier` format for the `players` table is `license:XXXX:char:N` (e.g. `license:abc123:char:1`). Characters are stored separately in the `characters` table and joined at load time.

---

## Core Systems

Each file in `core/` has a single responsibility.

### `core/config.lua`
Global framework settings. Adjust before deployment.

```lua
Config.Framework = {
    Lang = "en",
    Debug = true,         -- Set false in production
    LogLevel = 2,         -- 0=Error 1=Warn 2=Info 3=Trace
    MaintenanceMode = false
}

Config.System = {
    AutoSaveInterval = 60000 * 5,  -- 5 minute batch save
    RateLimitEnabled = true,
    MaxApiRequestsPerSec = 50
}

Config.Economy = {
    StartingCash = 500,
    StartingBank = 5000
}
```

---

### `core/event_bus.lua`
The nervous system of the framework. All module-to-module communication goes through here.

```lua
-- Subscribe
IS4.Events.on("is4-core:moneyAdded", function(payload)
    print(payload.source, payload.amount)
end)

-- Publish
IS4.Events.emit("is4-core:moneyAdded", { source = src, amount = 500 })

-- One-time subscription
IS4.Events.once("module:loaded", function(name) end)

-- Unsubscribe
IS4.Events.off("eventName", handlerRef)
```

Every listener is wrapped in `pcall`. A bad listener cannot kill other listeners or the framework.

---

### `core/datastore.lua`
Handles all database I/O. Uses `oxmysql` (MySQL.Async) under the hood.

**Key functions:**

```lua
-- Check if player exists; INSERT if not, update last_seen if yes
IS4.Datastore.EnsurePlayer(identifier, license, callback)

-- Load player data from DB (returns parsed Lua table)
IS4.Datastore.LoadPlayer(identifier, callback)

-- Save a player object to DB (only if isDirty = true)
IS4.Datastore.SavePlayer(player)

-- Save all dirty players in RAM (called by cron every 5 min)
IS4.Datastore.SaveAllDirty()

-- Raw query wrappers
IS4.Datastore.Query(query, params, callback)        -- execute
IS4.Datastore.FetchSingle(query, params, callback)  -- fetchSingle
IS4.Datastore.FetchAll(query, params, callback)     -- fetchAll
```

> Direct `MySQL.Async` calls are also valid inside modules. The Datastore wrappers are provided for convenience and logging.

---

### `core/player_manager.lua`
Manages the in-memory player cache. Returns player objects with built-in dirty tracking.

```lua
-- Create (called after DB load, during character select)
local player = IS4.PlayerManager.CreatePlayer(source, identifier, dbData)

-- Get a loaded player by server source ID
local player = IS4.PlayerManager.GetPlayer(source)

-- Get all loaded players
local players = IS4.PlayerManager.GetPlayers()
```

**Player object methods:**

```lua
player.get(key)                    -- Get any field
player.set(key, value)             -- Set any field, marks isDirty, emits event
player.addMoney(type, amount)      -- type: "cash" | "bank"
player.removeMoney(type, amount)   -- Returns false if insufficient funds
player.giveItem(itemName, amount)
player.removeItem(itemName, amount) -- Returns false if insufficient
```

All mutations set `player.isDirty = true`, which queues the player for the next batch save.

---

### `core/network.lua`
Wraps all client ↔ server communication with rate limiting.

```lua
-- Server: register a callback that clients can trigger
Core.Network.RegisterServerCallback("myEvent", function(src, data)
    return someResult
end)

-- Client: trigger that callback
Core.Network.TriggerServer("myEvent", data)
```

Requests exceeding `MaxApiRequestsPerSec` are silently dropped server-side.

---

### `core/permission.lua`
Role-based access control. Roles in order of power: `god > admin > mod > user`.

```lua
-- Check role
IS4.Permission.HasRole(source, "admin")

-- Check named permission (e.g. "inventory.give", "admin.kick")
IS4.Permission.HasPermission(source, "admin.kick")

-- Assign
IS4.Permission.SetRole(source, "mod")
```

---

### `core/profiler.lua`
Automatic lag spike detection. Wraps any function and logs a warning if it exceeds 15ms.

```lua
IS4.Profiler.Measure("myHeavyFunction", function()
    -- ...
end)
-- Output: [Profiler] Lag Spike Detected: myHeavyFunction took 23ms
```

---

### `core/logger.lua`
Structured logging with level filtering.

```lua
IS4.Logger.Info("Player loaded")
IS4.Logger.Warn("Rate limit hit for source 12")
IS4.Logger.Error("Failed to save player: abc123")
IS4.Logger.Trace("DB query executed in 2ms")  -- Only shown at LogLevel 3
```

---

### `core/module_loader.lua`
Reads `manifest.lua` files from each module, resolves the dependency graph, and boots modules in the correct order.

```lua
-- Example manifest.lua (inside your module)
return {
    name = "is4-mymodule",
    version = "1.0.0",
    dependencies = { "datastore", "players" },
    client = true,
    server = true
}
```

If a dependency is missing or fails, the dependent module will not load and a clear error is logged.

---

## API Layer

The `/api` folder exposes simplified, safe-to-call functions for use inside modules and external scripts.

```lua
-- Get the core object
local Core = exports['is4-core']:GetCore()

-- Or call exports directly
exports['is4-core']:AddMoney(source, 5000)
exports['is4-core']:GiveItem(source, "water", 3)
```

| File | Exported Functions |
|---|---|
| `api/items.lua` | `CreateItem(name, opts)`, `GiveItem(src, name, amount)`, `RemoveItem(src, name, amount)`, `GetItemData(name)` |
| `api/money.lua` | `AddMoney(src, amount)`, `RemoveMoney(src, amount)`, `TransferMoney(from, to, amount)` |
| `api/weapons.lua` | `CreateWeapon(name, opts)`, `GiveWeapon(src, name, ammo)`, `RemoveWeapon(src, name)` |
| `api/vehicles.lua` | `CreateVehicle(model, opts)`, `SpawnVehicle(src, model, coords)`, `DeleteVehicle(src)` |
| `api/jobs.lua` | `CreateJob(name, opts)`, `SetPlayerJob(src, jobName, grade)` |
| `api/players.lua` | `GetPlayerData(src)`, `GetIdentity(src)`, `SetPlayerMetadata(src, key, value)` |
| `api/notifications.lua` | `SendNotification(src, text, type)` |

---

## Module Reference

### is4-players — Character System
Handles player connection, character selection (up to 2 characters), creation, deletion, and world spawn.

**Flow:**
1. `playerConnecting` → `EnsurePlayer()` defers the connection, creates a DB row if the player is new
2. Client receives character list from `characters` table via `fetchCharacters` callback
3. Player selects a character → `selectCharacter` event → DB JOIN query loads full data → `PlayerManager.CreatePlayer()` populates RAM cache
4. Client is teleported to `Config.StartingCoords` and `is4-core:clientSpawned` is emitted

**Key config:** `StartingCoords`, `MaxCharacters = 2`

---

### is4-hud — NUI HUD
HTML/CSS/JS NUI overlay showing health, armor, hunger, thirst, cash, and bank balance.

- Stats update on Event Bus signals only — no polling loops
- Health/Armor read every 500ms (GTA native requirement)
- Money animations triggered by `is4-core:moneyAdded` / `moneyRemoved`

---

### is4-needs — Hunger & Thirst
Depletes hunger/thirst on a configurable tick interval. Applies starvation damage at 0. Integrates with the items system to restore stats on food/drink use.

**Key config:** `HungerDepletion`, `ThirstDepletion`, `StatusTickRate`, `FoodItems`

---

### is4-garages — Garage System
Stores and retrieves player vehicles. Tracks plate, model, mods, fuel, damage, and garage location in `player_vehicles`.

**States:** `0 = stored`, `1 = out`, `2 = impounded`

---

### is4-housing — Property System
Buy/sell properties, enter/exit interiors, share keys, and access stash storage. Police can access properties based on `PoliceAccess` config.

**Key config:** `PriceMultiplier`, `PoliceAccess`, `AllowWeaponStorage`, property definitions

---

### is4-motels — Motel System
Hourly room rentals for players without permanent housing. Rooms expire automatically with player notification.

**Key config:** `RentPricePerHour = $200`, `MaxRentHours = 24`, `StorageSlots = 20`

---

### is4-policejob — Police Job
Full law enforcement system with duty toggle, cuffing, escort, jailing, and fines.

| Server Event | Action |
|---|---|
| `toggleDuty` | Toggle on-duty status |
| `cuff` / `uncuff` | Restrain/release a player |
| `escort` | Force target to follow the officer |
| `putInVehicle` | Place cuffed player in nearest vehicle's back seat |
| `jail` | Teleport to Bolingbroke with countdown timer |
| `fine` | Deduct from target's bank account |

**Key config:** `Ranks`, `Armory`, `Jail.MinTime`, `Jail.MaxTime`

---

### is4-ambulancejob — EMS / Ambulance Job
Revive, heal, hospital respawn, and pharmacy items. Includes a bleedout countdown on death.

| Server Event | Action |
|---|---|
| `reportDeath` | Notify all on-duty EMS |
| `revive` | Revive a downed player |
| `heal` | Apply partial healing |
| `hospitalRespawn` | $2,500 pay-to-respawn if no EMS available |
| `buyItem` | Purchase medical supplies from pharmacy |

---

### is4-mechanic — Mechanic Job
Vehicle repair (body, engine, full), respray, and tow truck dispatch.

**Pricing config:** `BodyRepair = $500`, `EngineRepair = $750`, `FullRepair = $1000`, `Respray = $250`

---

### is4-gangs — Gang System
Create gangs, manage members (max 20), deposit/withdraw from shared gang bank. Boss-only withdrawal.

---

### is4-crypto — Cryptocurrency
Mine crypto with a `cryptostick` item, exchange for cash, or transfer P2P. Default: 1 crypto = $500.

---

### is4-phone — Phone System
SMS between players, 911 dispatch (notifies all on-duty police/EMS with a map blip), and bank transfers.

---

### is4-chat — Chat System (NUI)
Proximity and global chat with multiple channel prefixes.

| Prefix | Channel | Scope |
|---|---|---|
| *(none)* | LOCAL | Proximity (20m) |
| `/ooc` | OOC | Global |
| `/me` | ME | Proximity |
| `/do` | DO | Proximity |
| `/ad` | AD | Global |
| `/twit` | TWIT | Global |

Max 50 messages in DOM. Auto fade-out after 15 seconds.

---

### is4-system — Server Management
Time/weather sync and admin commands (`/setweather`, `/settime`, `/kick`, `/announce`). Time advances 10 in-game minutes every 5 real minutes, synced to all clients.

---

### is4-anticheat — Anti-Cheat
Event-driven anomaly detection. No continuous loops.

| Check | Threshold | Action |
|---|---|---|
| Money injection | `amount > $100,000` in one event | Warning → 3 warnings = kick |
| Godmode | `health > 200` (reported every 10s) | 2 flags → kick |
| Speed hack | `speed > 500 km/h` (reported every 10s) | Flag logged |

---

### is4-advancedresources — Performance Optimization
Reduces NPC and vehicle density to 30%, disables ambient emergency dispatches, removes garbage trucks, random boats, and trains.

---

### is4-others — RP Interactions
Small RP animations triggered by chat commands.

| Command | Action |
|---|---|
| `/point` | Toggle pointing animation |
| `/handsup` | Hands up (surrender) |
| `/crouch` | Toggle crouch (clipset swap) |
| `/carry` | Carry the nearest player on your back |

---

## Writing Your Own Module

### 1. Folder structure
```
/modules/is4-mymodule/
├── manifest.lua
├── config.lua
├── client/
│   └── main.lua
├── server/
│   └── main.lua
└── data/
```

### 2. `manifest.lua`
```lua
return {
    name    = "is4-mymodule",
    version = "1.0.0",
    dependencies = { "datastore", "players" },  -- loaded before this module
    client  = true,
    server  = true
}
```

### 3. Server-side boilerplate
```lua
local Core = exports['is4-core']:GetCore()

-- Listen to the Event Bus
Core.Events.on("is4-core:playerLoaded", function(payload)
    local src = payload.source
    -- do something when any player loads
end)

-- Register a callback the client can call
Core.Network.RegisterServerCallback("is4-mymodule:getData", function(src, args)
    local player = IS4.PlayerManager.GetPlayer(src)
    if not player then return nil end
    return { money = player.money }
end)

-- Direct DB access
IS4.Datastore.FetchAll("SELECT * FROM my_table WHERE owner = ?", { "identifier" }, function(rows)
    for _, row in ipairs(rows) do
        print(row.id)
    end
end)
```

### 4. Client-side boilerplate
```lua
local Core = exports['is4-core']:GetCore()

-- Run logic after the player has fully spawned
Core.Events.on("is4-core:clientSpawned", function()
    -- safe to start threads, draw markers, etc.
end)

-- Call a server callback
Core.Network.TriggerServer("is4-mymodule:getData", {}, function(result)
    if result then
        print("Cash:", result.money.cash)
    end
end)
```

---

## Event Bus Reference

| Event Name | Payload | Fired When |
|---|---|---|
| `is4-core:playerLoaded` | `{ source, identifier }` | Player RAM cache is created |
| `is4-core:clientSpawned` | — | Client finishes spawning into the world |
| `is4-core:moneyAdded` | `{ source, type, amount, total }` | Money added to a player |
| `is4-core:moneyRemoved` | `{ source, type, amount, total }` | Money removed from a player |
| `is4-core:itemAdded` | `{ source, item, amount }` | Item given to a player |
| `is4-core:itemRemoved` | `{ source, item, amount }` | Item taken from a player |
| `is4-core:vehicleSpawned` | `{ source, model, plate }` | A vehicle is spawned for a player |
| `is4-core:playerUpdate:{key}` | `{ source, new, old }` | Any `player.set(key, value)` call |
| `datastore:playerSaved` | `identifier` | Player successfully written to DB |
| `module:loaded` | `moduleName` | A module finishes loading |
| `core:modulesLoaded` | `{ count }` | All modules have finished loading |

---

## Config Philosophy

Avoid on/off switches. Define **behavior**, not boolean flags.

```lua
-- ❌ Bad
weaponsEnabled = true

-- ✅ Good
Config.WeaponRules = {
    MaxCarried        = 3,
    DurabilityEnabled = true,
    DegradationRate   = 0.01,
    RepairEnabled     = true,
    LicenseRequired   = true
}
```

This makes configs self-documenting and avoids conditional spaghetti in module logic.

---

## Performance & Design Notes

| Feature | Detail |
|---|---|
| **RAM cache** | All active player data lives in `IS4.PlayerManager._players`. Zero SQL on gameplay actions. |
| **Batch save** | Only `isDirty` players are written. A server with 32 players doing nothing = 0 writes. |
| **Rate limiting** | Client → Server capped at 50 req/s per player. Excess is silently dropped. |
| **Profiler** | Any operation over 15ms emits a `[Profiler] Lag Spike` warning. Use in development. |
| **NPC density** | Reduced to 30% via `is4-advancedresources`. Significant FPS gain on lower-end hardware. |
| **Fault isolation** | Every Event Bus listener and API call runs in `pcall`. One broken module ≠ server crash. |
| **Restart cleanup** | On boot, `server/main.lua` resets all `status = 1` rows to `status = 0` to handle dirty state from unexpected shutdowns. |
| **Identifier format** | `license:XXXX:char:N` — separates license identity from character slots cleanly at the DB level. |

---

> Built from scratch by **Isa**. If you're reading this, you have the source — treat it with care.
# Fivem-is4-core-NEW-CORE-BASE-IS-EASY-
Best Easy Core &lt;3 ^^ 1.0
