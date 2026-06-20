# luau-security-practices

This is just a guide post researched by like 20 AI models so, do the research yourself, sources are cited at the bottom.

---

## The basics you can't skip

Never trust anything that comes from a client. Every RemoteEvent, every RemoteFunction, treat it as hostile until you prove it's safe.

The server owns the truth. All game state, all decisions, live there. The client just shows what the server says.

**Avoid yielding in remote handlers when possible.** If a handler must do expensive or independent work, wrap *that* work in `task.spawn()`, not every handler by default.

| Universal `task.spawn()` | Selective `task.spawn()` |
|--------------------------|--------------------------|
| Harder to debug (stack traces split across threads) | Fast handlers stay synchronous and easy to trace |
| Uncontrolled concurrency | You decide what runs in parallel |
| Race conditions if shared state isn't guarded | Shared state stays in one thread unless you design for concurrency |
| Often unnecessary for fast checks | Expensive I/O, DataStore calls, or fire-and-forget side effects get their own thread |

```lua
RE.BuyItem.OnServerEvent:Connect(function(plr, itemId)
    -- Fast path: validate synchronously, reject bad requests immediately
    if typeof(itemId) ~= "string" then return end
    if not ShopCatalog[itemId] then return end

    -- Slow path: spawn only the work that yields
    task.spawn(function()
        processPurchase(plr, itemId)
    end)
end)
```

---

## Authentication vs authorization

These get conflated constantly. Both matter; they solve different problems.

| Term | Question it answers | Roblox example |
|------|---------------------|----------------|
| **Authentication** | *Who is this?* | `player.UserId`, session identity from Roblox |
| **Authorization** | *Are they allowed to do this?* | Staff list, rank check, owns-item check, cooldown, game-state gate |

Authentication is mostly free on Roblox, the platform tells you who fired the remote. Authorization is where games get exploited: assuming "they're a real player" means "they can do anything."

```lua
-- AuthN: Roblox gives you plr.UserId for free
-- AuthZ: you decide if this player may run this command right now
RE.AdminCmd.OnServerEvent:Connect(function(plr, cmd)
    if not isStaff(plr.UserId) then return end   -- authorization
    if not ALLOWED_CMDS[cmd] then return end     -- authorization (allow-list)
    runAdminCmd(plr, cmd)
end)
```

Never use `pcall` as a substitute for authorization. Catching errors doesn't tell you whether the caller was allowed to try.

---

## How client and server actually talk

Think of it like this: the client **asks** for something to happen, the server **does** it, or says no.

Bad example: letting the client decide numbers  
```lua
RE.GiveCash.OnServerEvent:Connect(function(plr, amt)
    plr.leaderstats.Cash += amt   -- instant exploit
end)
```

Good example: the client requests, the server checks and acts  
```lua
RE.RequestGem.OnServerEvent:Connect(function(plr, gemId)
    if typeof(gemId) ~= "string" then return end

    local part = workspace.Gems:FindFirstChild(gemId)
    if not part or part:GetAttribute("Claimed") then return end

    part:SetAttribute("Claimed", true)   -- claim before reward to avoid double-pickup races
    part:Destroy()

    plr.leaderstats.Gems += ServerConfig.GEM_REWARD   -- reward from server config, not client
end)
```

Multiple players firing the same gem at once is a real race. Claim the resource (attribute, server-side set, or `:Take()` pattern) *before* granting the reward.

---

## Server Authority (still early access)

If you're in the Server Authority beta, the server now **alone** can move a character. No more client-set velocity or position. That kills speed hacks, fly hacks, noclip out of the gate.

How to turn it on:
1. File → Beta Features → check **Server Authority Core API**
2. In Workspace set:
   * StreamingEnabled = true
   * UseFixedSimulation = true
   * NextGenerationReplication = true
3. Open Workspace → Server Authority → set AuthorityMode = Server

Remember: you can't publish yet while it's in beta, and the APIs might shift. Even with Server Authority on you still need to validate every input, don't skip sanity checks.

---

## Locking down your remotes

You can't stop an exploiter from firing a RemoteEvent, tools like RemoteSpy let them see and copy anything. **Remote names and locations are not security boundaries.** Determined exploiters will find runtime-created remotes too.

Real protection lives in:

* **Validation**, type, range, allow-list, game-state checks on every argument
* **Authorization**, who may call this, when, and under what conditions
* **Rate limiting**, server-side cooldowns and spam caps
* **Server authority**, the server decides outcomes; the client only requests

Creating remotes at runtime (instead of leaving them in ReplicatedStorage at publish time) may slightly slow down inexperienced attackers, but treat that as **organization and defense-in-depth**, not obscurity-as-security.

```lua
-- ServerScriptService/RemoteConfig.lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Remotes = {}

function Remotes.create(name)
    local r = Instance.new("RemoteEvent")
    r.Name = name
    r.Parent = ReplicatedStorage.Remotes
    return r
end

return Remotes
```

### Type checks, do them every time
```lua
RE.Move.OnServerEvent:Connect(function(plr, dir, speed)
    if typeof(dir) ~= "Vector3" or typeof(speed) ~= "number" then
        return   -- log if you want; kicking on bad types alone causes false positives
    end
    -- …
end)
```

### Table arguments, only touch what you need
```lua
RE.Buy.OnServerEvent:Connect(function(plr, payload)
    if typeof(payload) ~= "table" then return end
    if typeof(payload.ItemId) ~= "string" then return end

    local keyCount = 0
    for _ in payload do
        keyCount += 1
        if keyCount > 8 then return end   -- cap size; prefer # for arrays
    end
    -- …
end)
```

---

## Keep your important code hidden

Put every server-side script, anti-exploit logic, economy handlers, remote modules, inside **ServerScriptService**. Never replicate it.

ModuleScripts go there too; `require()` them from server code.

Luau compiles to bytecode which already gives you a bit of obscurity for free. If you want extra layers, tools like luax, luac, or bytenode exist, but **never lean on obfuscation as your only defense.**

If a script handles money, stats, or anything that matters, assume an attacker can read it and build accordingly.

---

## The "never trust the client" pattern in action

Never let the client decide a number that affects game state. Let the client say *what* they want to do, let the server decide *how much* or *if*.

Bad:
```lua
RE.Hit.OnServerEvent:Connect(function(plr, target, dmg)
    target.Humanoid:TakeDamage(dmg)   -- exploiter sends 9999
end)
```

Good:
```lua
RE.RequestHit.OnServerEvent:Connect(function(plr, targetName)
    if typeof(targetName) ~= "string" then return end

    local char = plr.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local target = workspace.Enemies:FindFirstChild(targetName)
    if not target then return end
    local thum = target:FindFirstChildOfClass("Humanoid")
    if not thum then return end

    local distance = (target.HumanoidRootPart.Position - hrp.Position).Magnitude
    if distance > 20 then return end   -- too far? ignore

    thum:TakeDamage(ServerConfig.BASE_DMG)   -- server value
end)
```

---

## Quick sanity checks you should actually use

| What to check | How |
|---------------|-----|
| Is it an Instance and inside workspace? | `typeof(obj)=="Instance" and obj:IsDescendantOf(workspace)` |
| Number in a sane range? | `<`, `>`, `math.clamp` |
| String short enough or in an allow list? | `string.len`, `table.find` |
| Table not absurdly large? | count keys or `#t` for arrays |
| No NaN or infinities? | `n == n` (NaN fails this), range checks against `math.huge` |
| Does this call make sense right now? | compare against a round-state flag |

```lua
local function isFinitePositive(n)
    return typeof(n) == "number"
        and n == n                          -- NaN check (NaN ~= NaN in IEEE 754)
        and n ~= math.huge and n ~= -math.huge
        and n > 0
end
```

---

## Rate limiting and server-side cooldown design

Client-side cooldowns are useless. Track timing yourself inside the remote handler.

```lua
local COOLDOWN = 0.2   -- seconds, tweak per remote
local lastCall = {}

game.Players.PlayerAdded:Connect(function(p)
    lastCall[p.UserId] = 0
end)
game.Players.PlayerRemoving:Connect(function(p)
    lastCall[p.UserId] = nil
end)

RE.SomeAction.OnServerEvent:Connect(function(plr, ...)
    local now = os.clock()      -- monotonic, immune to clock jumps
    if now - (lastCall[plr.UserId] or 0) < COOLDOWN then
        return                  -- silently ignore spam
    end
    lastCall[plr.UserId] = now
    -- …process the real request
end)
```

Design notes:

* **Per-remote, per-player**, a global cooldown on one action shouldn't block unrelated actions.
* **Silent ignore vs kick**, spam on gameplay remotes: ignore. Sustained abuse or impossible call rates: log and escalate.
* **Cooldown before work**, set the timestamp *after* validation passes, or attackers burn cooldowns on rejected junk.
* **Stateful actions**, purchases, trades, and ability casts need cooldown *and* authorization, not just a timer.

---

## Anti-spam telemetry

Rate limits stop floods; telemetry tells you *who* is probing and *what* they're hitting.

Log (at minimum):

* rejected requests, bad type, out of range, unauthorized
* sustained spam, same remote fired above threshold in a window
* honeypot hits, see below
* impossible sequences, buy before shop open, trade item you don't own

```lua
local function logSuspicious(plr, reason, detail)
    warn(string.format("[anti-exploit] %s (%d): %s %s", plr.Name, plr.UserId, reason, detail or ""))
    -- plug in your analytics / moderation webhook here
end
```

Use logs to tune thresholds and review before auto-banning. Kicking on the first bad packet creates false positives from buggy tools, plugin mistakes, internal testing, and malformed clients.

---

## Remote direction, organization, not a security boundary

Using separate remotes for Client→Server vs Server→Client can improve clarity and make misuse easier to spot during code review. Roblox does not require this pattern, and direction enforcement alone does not stop exploiters who fire your C2S remotes with bad payloads.

Treat this as **recommended organization**, not mandatory security:

```lua
local REMOTE_DIR = {
    JoinEvent = "C2S", PingEvent = "S2C", }

for name, dir in REMOTE_DIR do
    local r = ReplicatedStorage.Remotes:FindFirstChild(name)
    if r and dir == "S2C" then
        r.OnServerEvent:Connect(function(plr)
            logSuspicious(plr, "wrong-direction", name)
            -- log first; auto-kick only if you accept false-positive risk
        end)
    end
end
```

### Honeypot remotes

Drop a RemoteEvent with a tempting name like `GiveAdminPerms` into `ReplicatedStorage.Remotes`. On the server, **log** the attempt, UserId, timestamp, any arguments.

```lua
RE.GiveAdminPerms.OnServerEvent:Connect(function(plr, ...)
    logSuspicious(plr, "honeypot", "GiveAdminPerms")
    -- review logs before auto-kick/ban; false positives happen
end)
```

Honeypots are useful signals, not proof of malice. Pair with human review or graduated responses (flag → temp mute → kick) rather than instant permanent bans unless you're confident in your signal quality.

---

## Economic exploits and inventory duplication

Most real-world Roblox exploits target **economy**, not movement. Assume attackers will:

* fire purchase remotes with arbitrary item IDs
* replay "grant reward" flows
* race concurrent requests to double-collect
* desync client UI state from server inventory

Patterns that help:

**Server-owned ledger**, currency and items live in server tables or DataStores; the client displays a copy.

**Idempotent grants**, track transaction IDs or "already claimed" flags so the same action can't pay out twice.

```lua
local pendingClaims = {}   -- [userId][claimId] = true

local function claimOnce(plr, claimId, grantFn)
    local uid = plr.UserId
    pendingClaims[uid] = pendingClaims[uid] or {}
    if pendingClaims[uid][claimId] then return false end
    pendingClaims[uid][claimId] = true
    grantFn()
    return true
end
```

**Validate ownership before mutate**, never `table.insert(inventory, item)` because the client said so; check the server inventory first.

**Atomic-ish updates**, read inventory → validate → write inventory in one server flow; don't yield between check and grant without a lock/claim flag.

---

## Replay attacks

A replay is firing a *valid-looking* request again to get the benefit twice, re-triggering a quest complete, re-claiming a daily reward, re-selling the same item.

Mitigations:

* **One-time tokens**, server generates a nonce per action; client returns it; server consumes it
* **Monotonic counters**, server tracks last processed action ID per player
* **Time windows**, daily rewards keyed to server date + already-claimed set
* **State gates**, "quest complete" remote rejects if quest isn't in completable state

```lua
RE.CompleteQuest.OnServerEvent:Connect(function(plr, questId)
    if typeof(questId) ~= "string" then return end
    local progress = getProgress(plr)
    if progress[questId] ~= "ready" then return end   -- not completable? ignore replay
    progress[questId] = "done"
    grantQuestReward(plr, questId)
end)
```

---

## DataStore abuse

DataStores are a trust boundary between sessions and the live server. Treat every read/write as exploitable if triggered by client requests.

* **Never write on client request alone**, validate the change against server state first
* **Rate-limit saves**, burst writes hit limits and lose data; batch where possible
* **Session locking**, one active session per user for economy-critical games
* **Retry with backoff**, `pcall` + retry for genuine service errors, not as exploit detection
* **Version or migrate carefully**, dupes often come from reading stale cache, writing twice, or rollback after grant

```lua
local saveCooldown = {}
local SAVE_INTERVAL = 30

local function scheduleSave(plr)
    local uid = plr.UserId
    if saveCooldown[uid] then return end
    saveCooldown[uid] = true
    task.delay(SAVE_INTERVAL, function()
        saveCooldown[uid] = nil
        savePlayerData(plr)   -- write from server-authoritative state only
    end)
end
```

---

## Secure trading and permission systems

Trading is two-player authorization with a concurrency problem. Both sides must agree on the same trade snapshot; either side disconnecting mid-trade shouldn't dup items.

Sketch:

1. Server builds trade offer from **server inventories** only
2. Both players accept the same trade ID
3. Re-validate ownership and quantities at commit time
4. Swap atomically in one server function, no yield between remove and add
5. Log completed trades for dispute review

Permission / role management:

* Store roles server-side (group rank, config table, DataStore), never trust a client "isAdmin" flag
* Separate **commands** (what staff can run) from **ranks** (who they are)
* Audit log privileged actions
* Prefer Roblox group roles or a maintained allow-list over hard-coded UserIds in scattered scripts

```lua
local RolePermissions = {
    mod = { "kick", "mute" }, admin = { "kick", "mute", "ban", "giveItem" }, }

local function canRun(plr, action)
    local role = getServerRole(plr)   -- from group API or server config
    local perms = RolePermissions[role]
    return perms and table.find(perms, action) ~= nil
end
```

---

## Where things live, quick reference

| Folder | Visible to client? | Put what here |
|--------|--------------------|---------------|
| ServerScriptService | **No** | All anti-exploit code, economy logic, remote handlers |
| ReplicatedStorage | **Yes** | RemoteEvent/Function definitions, shared data, assets |
| PlayerScripts | Per-player: yes | LocalScripts, InputAction setup |
| StarterPlayerScripts | Per-player (on spawn) | Default client scripts |

If the client has a legitimate reason to read something, assume an attacker can read it. Keep security-critical stuff strictly in ServerScriptService.

---

## Proven patterns that actually work

**Distance check on any "interact with X" request**
```lua
local MAX_DIST = 14   -- adjust to feel right

local function closeEnough(plr, target)
    local char = plr.Character
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local thrp = target:FindFirstChild("HumanoidRootPart")
    if not hrp or not thrp then return false end
    return (hrp.Position - thrp.Position).Magnitude <= MAX_DIST
end
```

**Allow-list instead of raw equality**
```lua
local ALLOWED_TOOLS = {"sword", "shield", "potion"}

RE.PickUp.OnServerEvent:Connect(function(plr, toolName)
    if typeof(toolName) ~= "string" then return end
    if not table.find(ALLOWED_TOOLS, toolName) then return end
    -- …process
end)
```

**UserId gate for privileged remotes**
```lua
local STAFF_IDS = {1234567, 8901234}

RE.AdminCmd.OnServerEvent:Connect(function(plr)
    if not table.find(STAFF_IDS, plr.UserId) then
        return   -- log; reserve Kick for repeated or confirmed abuse
    end
end)
```

**Avoid using pcall as an exploit detector**  
`pcall` just catches errors; it doesn't tell you whether the client is tampering. Swallowing errors silently can actually hide abuse. Use `pcall` only for genuine safety (like DataStore calls), never as a security mechanism.

---

## Input Action System

The Input Action System (IAS) is the channel Roblox marks as trusted. Anything outside IAS (your own RemoteEvents) stays hostile.

Client side (LocalScript):
```lua
local CAS = game:GetService("ContextActionService")

CAS:BindAction("Shoot", function(name, state, inp)
    if state == Enum.UserInputState.Begin then
        ShootRE:FireServer(inp.Position)
    end
    return Enum.ContextActionResult.Sink
end, false, Enum.KeyCode.ButtonR2, Enum.KeyCode.MouseButton1)
```

Server side, still verify!
```lua
ShootRE.OnServerEvent:Connect(function(plr, aimPos)
    if typeof(aimPos) ~= "Vector3" then return end

    local char = plr.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local start = hrp.Position
    local dir = (aimPos - start).Unit

    ServerWeapon.Spawn(plr, start, dir)
end)
```

Treat IAS data like any other remote: run the same type, range, and sanity checks.

---

## Quick pre-publish glance

Glance at this before you hit Publish:

- Never trust the client, validate every Remote/Function argument
- Server is the source of truth, authoritative state lives server-side
- Avoid yielding in remote handlers; use `task.spawn()` selectively for expensive work
- Remote names/locations are not security, validate, authorize, rate-limit
- Type-check every argument with `typeof()`
- Validate table arguments: use only needed keys, cap size
- Separate C2S and S2C remotes for clarity (optional, not a security silver bullet)
- Honeypot remotes: log first; auto-ban only with review or graduated responses
- Rate-limit remotes on the server (`os.clock`, per-player, per-remote)
- Design cooldowns before grant, not just after spam
- Log suspicious traffic, tune thresholds from telemetry
- Guard economy: idempotent grants, claim flags, server-owned inventory
- Mitigate replays: state gates, one-time tokens, monotonic action IDs
- DataStore writes from server state only; rate-limit saves
- Trades: re-validate at commit, atomic swap, audit log
- Roles/permissions: server-side, allow-list actions, audit privileged use
- Keep anti-exploit/economy code in ServerScriptService only
- Use the Input Action System for trusted client input, still verify!
- Never use `pcall` as an exploit detection mechanism
- Apply distance, allow-list, and authorization patterns where relevant
- If targeting Server Authority, enable it per the beta steps; otherwise manually validate movement-related remotes

---

That's it. Go make your game less of a free-for-all. If something feels off, tweak it, test it, repeat. No magic, just solid habits.
