# Pazzion's LuaBoost — WoW 3.3.5a Lua Runtime Optimizer & Performance Suite

![WoW Version](https://img.shields.io/badge/WoW-3.3.5a%20(Build%2012340)-blue.svg)
![Server Compatibility](https://img.shields.io/badge/Server-Warmane%20%7C%20WotLK%20Private%20Servers-orange.svg)
![Type](https://img.shields.io/badge/Type-Performance%20%26%20Engine%20Optimizer-brightgreen.svg)
![Status](https://img.shields.io/badge/Status-Production%20Ready-success.svg)
![Taint Safe](https://img.shields.io/badge/Taint-100%25%20Safe-purple.svg)

**Pazzion's LuaBoost** is a comprehensive Lua runtime optimization suite and smart garbage collection (GC) manager engineered specifically for the **World of Warcraft: Wrath of the Lich King (3.3.5a)** client. Tested and calibrated on high-population servers like **Warmane** (Icecrown, Lordaeron, Onyxia), it eliminates micro-stutters, improves 1% low frame rates, accelerates loading screens, and shields the client from UI thrashing.

---

## The Problem in WoW 3.3.5a

The 3.3.5a client relies on Lua 5.1 with an uncalibrated, uncoordinated automatic garbage collector. In modern WotLK environments with demanding addon suites (TradeSkillMaster, ElvUI, WeakAuras, Details/Recount, DBM):
- **GC Spikes:** Lua memory accumulates until the client halts execution for a full garbage sweep, causing noticeable 1–2 second stutter during raids, arena matches, or Dalaran bank visits.
- **Loading Screen Freezes:** Dozens of addons fire redundant events simultaneously during zoning (`WORLD_MAP_UPDATE`, `BAG_UPDATE`, etc.), turning 3-second loading screens into 20-second hangs.
- **UI Engine Thrashing:** Addon unit frames spam native C engine calls (`SetValue`, `SetMinMaxValues`) 60+ times per second even when health, mana, or cast progress has not changed.
- **Heap Allocation Churn:** Addons constantly allocate and discard short-lived temporary tables, putting immense pressure on the Lua heap.

---

## Key Pillars & Architecture

### 1. Smart 4-Tier Incremental Garbage Collector (Smart GC)
LuaBoost replaces the game's erratic automatic garbage collector (`collectgarbage("stop")`) with a deterministic, frame-budgeted stepping system:
- **Combat Mode:** Step budget is minimized (~64 KB/frame) to prioritize rendering performance and maintain stable 1% low FPS during encounters.
- **Normal Mode:** Steady, balanced incremental sweeps (~256 KB/frame) preventing memory spikes during regular gameplay.
- **Idle Mode:** When resting in sanctuaries or AFK, step size increases up to 1024 KB/frame to aggressively flush accumulated heap without impacting responsiveness.
- **Loading Screen Mode:** Halts garbage collection to avoid freezing during asset streaming.
- **Emergency Protection:** Continuous threshold monitoring safely triggers controlled recovery before memory exhaustion can crash the 32-bit client.

### 2. SpeedyLoad (Zoning & Teleport Optimization)
SpeedyLoad suppresses noisy, high-frequency events while the loading screen is active:
- **Safe Mode (23 events):** Temporarily unregisters non-essential events without breaking any addon logic.
- **Aggressive Mode (35 events):** Further silences secondary UI notifications for maximum teleport and zoning speed.
- **Zero-Loss Re-registration:** Restores event listeners seamlessly upon `PLAYER_ENTERING_WORLD` and re-fires pending triggers.

### 3. UI Thrashing Protection (ThrashGuard)
- Intercepts frequent `StatusBar` API invocations: `SetValue`, `SetMinMaxValues`, and `SetStatusBarColor`.
- Caches widget state. If an addon attempts to apply identical values, LuaBoost bypasses the native engine call.
- **100% Taint-Safe:** Restricts hooks strictly to `StatusBar` objects; secure `FontString` and action button elements are left untouched.

### 4. Memory Pooling & Lua Enhancements
- **`GetTimeCached()`:** Caches `GetTime()` on a per-frame basis, eliminating thousands of redundant C-to-Lua lookups in tight `OnUpdate` loops.
- **Table Pool System:** Reusable table management (`LuaBoost_AcquireTable` / `LuaBoost_ReleaseTable`) to minimize garbage generation.
- **Tooltip Throttling:** Caps `SetSpell` and `SetHyperlink` refreshes at 10/sec, eliminating micro-freezes when mousing over raid frames or action bars.
- **Lua 5.0 Legacy Shims:** Built-in compatibility layer (`table.getn`, `table.setn`, `table.foreach`, `table.foreachi`) ensuring legacy WotLK addons run flawlessly without errors.

### 5. Built-in Diagnostics & Profiling
Built-in profiling utilities allow you to audit your entire addon lineup directly in-game:
- **FPS & Frame-Time Monitor:** Measures average framerate and 1% low drops over a 10-second sample (`/lb fps`).
- **Event Profiler:** Identifies which addons trigger high-volume event storms (`/lb events`).
- **Memory Leak Scanner:** Measures memory delta across all active addons over a 30-second window to pinpoint rogue scripts (`/lb memleak`).
- **Real-Time Heap Telemetry:** Live memory consumption and table pool status (`/lb gc`, `/lb pool`).

### 6. C-Level DLL Interop (Optional)
Supports bidirectional synchronization via global hooks (`_G.LUABOOST_DLL_*`) for clients paired with `wow_optimize.dll`.

---

## Directory Structure

> **Notice:** The addon directory MUST start with an exclamation mark (`!LuaBoost`). In World of Warcraft 3.3.5a, addon loading order is determined alphabetically. The `!` prefix guarantees that LuaBoost loads before all other addons (TSM, ElvUI, DBM), allowing memory shims and GC controls to initialize first.

```
World of Warcraft/
└── Interface/
    └── AddOns/
        └── !LuaBoost/
            ├── !LuaBoost.toc
            ├── LuaBoost.lua
            ├── enUS.lua
            ├── deDE.lua
            └── koKR.lua
```

---

## Installation

1. Navigate to the [Releases](https://github.com/wonderkidsem-official/Pazzions-LuaBoost-WotLK/releases) section or clone the repository.
2. Copy the **`!LuaBoost`** folder into your World of Warcraft directory:
   ```
   <WoW Directory>\Interface\AddOns\!LuaBoost
   ```
3. Ensure the folder is named **`!LuaBoost`** (with the exclamation mark) so it loads before your other addons.
4. Launch the game and confirm `!LuaBoost` is enabled in the in-game AddOn list (check **"Load out of date AddOns"** if prompted).

---

## In-Game Slash Commands

Access all controls using `/luaboost` or `/lb`:

| Command | Description |
| :--- | :--- |
| `/lb` | Displays current runtime status, memory footprint, and active modes |
| `/lb gc` | Detailed Garbage Collector stats, memory usage, and tier step sizes |
| `/lb toggle` | Enables or disables the Smart GC Manager |
| `/lb force` | Forces an immediate full two-pass garbage sweep |
| `/lb sl` | Toggles SpeedyLoad on/off |
| `/lb sl safe` | Sets SpeedyLoad to **Safe Mode** (23 events suppressed) |
| `/lb sl agg` | Sets SpeedyLoad to **Aggressive Mode** (35 events suppressed) |
| `/lb tg` | Displays UI Thrashing Protection metrics (skipped vs passed calls) |
| `/lb tg toggle` | Toggles UI Thrashing Protection on/off |
| `/lb tg reset` | Resets ThrashGuard hit-rate counters |
| `/lb pool` | Displays table pool efficiency metrics (acquired, released, created) |
| `/lb fps` | Runs a 10-second real-time framerate and 1% low benchmark |
| `/lb events` | Profiles event activity for 10 seconds to detect spammy addons |
| `/lb memleak` | Runs a 30-second memory leak delta scan across all loaded addons |
| `/lb updates` | Lists all registered periodic `OnUpdate` dispatchers |
| `/lb settings` | Opens the graphical Interface Options configuration panel |
| `/lb help` | Prints the slash command reference in the chat window |

---

## Companion Addons

For the optimal performance and trading setup on Warmane WotLK:
- [Pazzion's TradeSkillMaster 2 (Warmane Edition)](https://github.com/wonderkidsem-official/Pazzions-TradeSkillMaster-2-WotLK-Warmane-Edition) — Fully tuned TSM 2 suite with instant AuctionDB scanning, single-query group lookup, and TSMPerfBoost.

---

## Credits & License

- **Original LuaBoost Architecture:** Suprematist
- **Packaged, Calibrated & Maintained for Warmane WotLK:** Pazzion
- **License:** Open Source under Blizzard's UI Add-On Development Policy.
