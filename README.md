# smoke-storagewars

Storage Wars style container auctions for civilians. ox_lib + framework/inventory/target agnostic via `smoke-bridge`.

## Dependencies

Required:

- [`ox_lib`](https://github.com/overextended/ox_lib)
- `smoke-bridge`(https://github.com/BigSmoke6000/smoke-bridge)
- One supported framework through `smoke-bridge`: `qbx_core`, `qb-core`, `es_extended`, or a custom bridge
- One supported inventory through `smoke-bridge`: `qs-inventory`, `ox_inventory`, `qb-inventory`, or a custom bridge

Interaction:

- `ox_target` or `qb-target`, unless `smoke-bridge` is set to `Config.Target = 'drawtext'`

Optional:

- [`interact-sound`](https://github.com/plunkettscott/interact-sound) - only needed if `Config.Voice.UseInteractSound = true`
- [`cd_easytime`](https://github.com/dsheedes/cd_easytime) - only needed if you want the auction schedule to follow synced in-game time

Start these before `smoke-storagewars` in `server.cfg`.

## Installation

1. Drop this resource in `[smoke-scripts]` and `ensure smoke-storagewars` in your server.cfg, after `ox_lib`, `smoke-bridge`, your framework, your inventory, and your target resource (or `ox_target`/`qb-target`, whichever `smoke-bridge/config.lua` is pointed at).
2. **Register the auctioneer's voice lines with `interact-sound`.** FiveM's NUI browser only serves files that are explicitly listed in a resource's `fxmanifest.lua` `files {}` block — anything missing from that list just 404s inside the NUI page with no error, and `Howl.play()` silently does nothing. The 8 voice line `.ogg` files ship in `interact-sound/client/html/sounds/auctioneer_*.ogg`, but you still need to declare them. Open `[standalone]/interact-sound/fxmanifest.lua` and make sure its `files {}` block includes all of these (they should already be there if you got this resource as a bundle - double check after any interact-sound update, since re-downloading it can overwrite the list):
   ```lua
   files {
       'client/html/index.html',
       'client/html/sounds/auctioneer_lotopen.ogg',
       'client/html/sounds/auctioneer_bid.ogg',
       'client/html/sounds/auctioneer_goingonce.ogg',
       'client/html/sounds/auctioneer_goingtwice.ogg',
       'client/html/sounds/auctioneer_sold.ogg',
       'client/html/sounds/auctioneer_unsold.ogg',
       'client/html/sounds/auctioneer_angry.ogg',
       'client/html/sounds/auctioneer_end.ogg'
   }
   ```
   Then **fully restart** `interact-sound` (`restart interact-sound` or a server restart) — `files {}` is only read when the resource starts, a refresh isn't enough.
3. If you ever swap in your own voice recordings, keep the same 8 filenames and they'll just work — no config or fxmanifest changes needed as long as the names match `Config.Voice.Lines` in `config.lua`.
4. To disable interact-sound voice entirely (ambient GTA barks only), set `Config.Voice.UseInteractSound = false` — step 2 becomes unnecessary.

## Bridge (framework / inventory / target swap)

This script no longer ships its own `bridge/` folder - framework/inventory/target access goes through the shared **`smoke-bridge`** resource instead (`ensure smoke-bridge` before this script in your server.cfg). `Config.Framework`/`Config.Inventory`/`Config.Target` are no longer set here - they live centrally in `smoke-bridge/config.lua` and are shared by every script that uses the bridge, defaulting to **auto-detect**:

```lua
-- smoke-bridge/config.lua
Config.Framework = 'auto'   -- 'auto' | 'qbx' | 'qb' | 'esx'
Config.Inventory = 'auto'   -- 'auto' | 'qs' | 'ox' | 'qb' | 'custom'
Config.Target = 'auto'      -- 'auto' | 'ox' | 'qb' | 'drawtext' | 'custom'
```

| Side | Function | Purpose |
| --- | --- | --- |
| server | `Bridge.GetCitizenId(src)` | stable character identifier |
| server | `Bridge.GetCharName(src)` | display name for bid chatter |
| server | `Bridge.GetMoney(src, account)` / `Bridge.RemoveMoney(...)` / `Bridge.AddMoney(...)` | payments and cash loot |
| server | `Bridge.GetSourceByCitizenId(cid)` | re-resolve a winner who relogged |
| server | `Bridge.CanCarryItem(src, item, amount)` / `Bridge.AddItem(...)` | loot payout |
| client | `Bridge.GetCitizenId()` | "is this my container?" checks |
| client | `Bridge.OnPlayerLoaded(cb)` | initial state sync |
| client | `Bridge.GetItemLabel(name)` | pretty names in the loot reveal |
| client | `Bridge.AddBoxZone(...)` / `Bridge.AddEntityZone(...)` / `Bridge.RemoveZone(...)` / `Bridge.RemoveEntityZone(...)` | auctioneer, containers, and loot props are all targetable through whichever target resource (or `drawtext` fallback) `smoke-bridge` is configured for |

See `smoke-bridge`'s own README for the full API surface and how to adapt a custom framework/inventory/target.

### Using a custom inventory

A ready-made template ships with `smoke-bridge` - you only fill in your inventory's exports:

1. Set `Config.Inventory = 'custom'` in `smoke-bridge/config.lua` (applies to every script using the bridge).
2. Open `smoke-bridge/server/inventory/custom.lua` and implement the two functions using your inventory's exports (commented examples are inside the file):
   - `Bridge.CanCarryItem(src, item, amount)` — return `true` if the player has space, `false` if not. If your inventory has no such check, return `true` and make sure `AddItem` returns `false` on failure instead.
   - `Bridge.AddItem(src, item, amount)` — give the item; return `true` **only** if it actually landed in the inventory. Loot that fails to give stays in the container so the winner can make room and search again — returning `true` on a silent failure loses the loot forever.
3. (Optional) Open `smoke-bridge/client/inventory/custom.lua` and implement `Bridge.GetItemLabel(name)` so the loot-reveal dialog shows pretty item names. If you skip this, raw item names are shown and everything still works.
4. Check every `item` in `Config.Rarities` exists in your inventory's item list, then restart `smoke-bridge` and this resource.

Until step 2 is done, the custom server bridge is intentionally safe: it gives nothing and prints a `[smoke-storagewars]` warning in the server console (tagged with whichever resource is actually calling it) so a half-configured swap can't eat loot silently.

A custom **framework** or **target** system works the same way — copy any file pair from `smoke-bridge/server/framework/` + `smoke-bridge/client/framework/` (or just `smoke-bridge/client/target/` - targeting has no server side), change its guard, implement the functions, and set `Config.Framework`/`Config.Target = 'yourname'`/`'custom'` in `smoke-bridge/config.lua`.

## How it works

0. **Daily schedule** — with `Config.Schedule.Enabled` (default on), auctions only auto-start between `StartHour` and `EndHour` on the **in-game clock** (default 7am-7pm), reading the time from `cd_easytime` if it's running, or the server's real-world clock hour otherwise. The first auction of the day fires as soon as the clock hits `StartHour`; while inside the window, sessions keep repeating every `Config.Auction.IntervalMinutes` same as before; once the clock passes `EndHour`, no new session starts (an already-running one is left to finish naturally, never cut off mid-bid). Set `Config.Schedule.Enabled = false` to go back to auctions repeating every `IntervalMinutes` around the clock with no daily window.
1. **Announcement** — the city (or just nearby players) gets a heads-up that an auction starts in `AnnounceMinutes`. A blip appears at the yard.
2. **Inspection phase** — containers spawn at the yard, and a countdown floats above the auctioneer's head ("Bidding starts in 1:23") until the first lot opens. Players can inspect each lot and see:
   - A **peek description** (what you can glimpse through the door — sometimes misleading, `Config.Hints.Accuracy`)
   - **Possible loot rarity odds** (Junk → Legendary percentages, biased toward the truth but noisy)
   - **Estimated value range** and **condition**
3. **Bidding** — lots run one at a time. Bid via ox_target on the live container, through the auctioneer's Auction Board, or hands-free with the **live bid UI**: stand within `Config.BidUI.Radius` of the auctioneer while a lot is live and a custom HUD panel pops up top-center of the screen — lot number, current bid with a flash animation on every new bid, a live self-ticking countdown (green → amber → red as it runs low), a **recent bids list** showing who bid what and in what order (NPCs shown in italics), and hotkeys to bid instantly — `E` minimum raise, `G` power bid, `H` custom amount (rebindable per player in Settings > Key Bindings > FiveM). The panel is `client/html/bidui.html` if you want to restyle it. Anti-snipe: bids in the final seconds reset the clock. The auctioneer calls "going once / going twice".
4. **Auctioneer voice** — the auctioneer ped audibly barks on key moments (lot open, bids, going once/twice, sold, unsold, deadbeats) using built-in GTA ambient speech, layered with real spoken lines played through your `interact-sound` resource. Voice lines ship with the script (`interact-sound/client/html/sounds/auctioneer_*.ogg`, generated with a free Microsoft neural voice via [edge-tts](https://github.com/rany2/edge-tts)) so voice works immediately — swap them for your own recordings any time by overwriting the same filenames listed in `Config.Voice.Lines`, no config changes needed. Set `Config.Voice.UseInteractSound = false` to fall back to ambient barks only.
5. **NPC bidders** — 2-4 NPC personalities show up per session (aggressive / conservative / sniper styles). They physically stand at the yard, raise their hands when bidding, talk trash in chat, and voice it too — built-in GTA ambient ped speech, so each NPC's own model supplies a matching voice with zero audio files needed. They bark a random line on every bid, cheer when they win a lot, and grumble when they lose (`Config.NPCVoice`). Their budgets are based on the same estimated value players see.
6. **Payment** — the winner pays instantly from `PaymentAccount`. If they can't cover the bid, they're banned from bidding for `DeadbeatCooldownMinutes` and the lot falls to the runner-up NPC.
7. **Looting** — the winner grinds the lock off (angle-grinder prop + progress bar). The true rarity is revealed and the loot **spawns as physical props laid out in a neat row directly in front of the container** — gold bars, money bags, cases, ammo boxes, weapon props (wraps into a second row further out if there are more than `Config.Pickup.MaxPerRow` items). Every container uses the same `Config.ContainerProp` model regardless of size/rarity, so nothing visually telegraphs what's inside. Each prop is picked up individually via ox_target with a short grab animation; the server validates every pickup (proximity, not-already-taken, carry weight). By default **anyone** can grab the props once the container is open — bring friends to help carry, or guard your haul from vultures. Set `Config.Pickup.WinnerOnly = true` to restrict pickups to the winner, or `Config.Pickup.Enabled = false` for the old straight-to-inventory flow. Unclaimed props despawn when the yard closes (`ClaimGraceMinutes`).

## Commands

- `/storageauction start` — start a session immediately (ace: `admin`)
- `/storageauction announce` — run the announcement flow, session starts after the warning period
- `/storageauction cancel` — cancel the running session (no refunds are needed; money is only taken when a lot is won)

## Setup notes

- **Adjust the coordinates** in `Config.Location` (auctioneer, bidder spots, container spots) to your preferred yard. Defaults are placed around the Terminal docks — verify them in-game and space container spots to fit the props (the large shipping container is ~12m long).
- **World props only spawn once you're within `Config.Location.SpawnDistance`** (default 90m) of the yard, and despawn again if you leave. This is deliberate: spawning them unconditionally the moment a session starts meant a player who was far away (or just joined) got containers before the game had streamed in collision at that location, so they floated. Raise `SpawnDistance` if your yard is visible/approachable from further away and you want props to appear sooner.
- **Check the loot tables** in `Config.Rarities` — every `item` must exist in your qs-inventory items list. Swap anything you don't have.
- **Pickup props**: `Config.Pickup.Models` maps items to the props that spawn on the ground (unlisted items use `default`, missing models fall back automatically). The lock-cutting grinder (`tr_prop_tr_grinder_01a`) is a Tuner DLC prop — set `Config.Pickup.CutProp = false` if your build doesn't have it (the animation still plays without it).
- Auto-scheduling is on by default (`Config.Auction.AutoSchedule`, every 90 minutes, gated to 7am-7pm in-game by `Config.Schedule`). Turn `AutoSchedule` off if you want admin/event-only auctions, or just turn `Config.Schedule.Enabled` off to keep the interval but drop the daily window.
- The daily-schedule window relies on `cd_easytime` (if installed) for the in-game hour; make sure it starts before `smoke-storagewars` in your server.cfg so auto-detection sees it. With zero players online there's no client to report the in-game hour, so it silently falls back to the server's real-world clock hour until someone connects.
- Sessions are in-memory only: a server restart mid-session drops unclaimed containers (money is only charged at the hammer, so nobody loses cash to a restart).
