# luau-security-practices

Practical, opinionated Roblox–Luau security guide: what to lock down, why, and how to do it in 10 minutes or less.

<!-- TOC placeholder — sections start below -->

1. [Absolute Foundations](#absolute-foundations)
2. [The Client-Server Model](#the-client-server-model)
3. [Server Authority](#server-authority)
4. [Lock Down RemoteEvents / RemoteFunctions](#lock-down-remoteevents--remotefunctions)
5. [Copy-Proof Your Important Scripts](#copy-proof-your-important-scripts)
6. [Data Integrity: Never Trust The Client](#data-integrity-never-trust-the-client)
7. [Sanity / Edge-Case Checking](#sanity--edge-case-checking)
8. [Rate Limiting](#rate-limiting)
9. [Remote Direction Enforcing](#remote-direction-enforcing)
10. [Instances: ReplicatedStorage vs ServerScriptService vs PlayerScripts](#instances-replicatedstorage-vs-serverscriptservice-vs-playerscripts)
11. [Anti-Exploit Patterns](#anti-exploit-patterns)
12. [Input Action System (IAS)](#input-action-system-ias)

---

## Absolute Foundations

These are non-negotiable. They are framework basics, not advanced tricks.

- **FilteringEnabled is on.** It ships enabled now; nothing replicates client → server without an explicit RemoteEvent/RemoteFunction. If you are still using "Experimental Mode", convert immediately.
- **Never trust the client.** Not even a little. Any data received via a RemoteEvent/RemoteFunction is hostile until verified.
- **ServerOwner = source of truth.** The server holds state — the client only displays it.
- **Use `task.spawn()` inside remote handlers** to prevent yielding/queue-exhaustion panics if a handler accidentally yields.

---

## The Client–Server Model

### Roles, one sentence each
| Side | Job |
|------|-----|
| Server | All game logic, validaton, authoritative state |
| Client | Input collection, visual effects, mock-lag prediction |

Everything client-created vanishes on cleanup. Anything replicated must be created on the server.

### Action, not request
The canonical pattern: client **requests** an action, the server **performs** it or **refuses** it.

```lua
-- ❌ BAD — client dictates outcome
RE.AddCash.OnServerEvent:Connect(function(player, amount)
    player.leaderstats.Cash.Value += amount  -- definitely gonna have a bad time
end)

-- ✅ GOOD — client requests, server enforces
RE.RequestPickup.OnServerEvent:Connect(function(player, pickupId)
    local part = workspace:FindFirstChild(pickupId)
    if not part then return end
    part:Destroy()  -- server owns the destroy
    player.leaderstats.Cash.Value += 5  -- value came from server, not client
end)
```

---

## Server Authority

Roblox's Server Authority system is in early access. It eliminates common categories of exploits (speedhacks, noclip, fly, direct position/velocity manipulation) by making the **server the only thing that can move a character**.

### What one sentence
Server authority lets the server validate all player input and all movement math. Clients no longer "own" their position and cannot set it directly.

### Why it matters
- Speedhacks require client to set velocity directly → blocked
- Flyhacks require teleport/weld to invisible parts → blocked
- Noclip requires disabling collision locally → blocked at intersection with replication

### Enabling it
1. File > Beta Features > **Server Authority Core API**
2. Workspace → `StreamingEnabled = true`, `UseFixedSimulation = true`, `NextGenerationReplication = true`
3. Workspace → Server Authority → `AuthorityMode = Server`

⚠️ In early access — not publishable. APIs are unstable.

### With/Without Server Authority
- **Without Server Authority:** you must manually reject all movement-based remotes
- **With Server Authority:** you can't touch the character physically at all; only server-set inputs move it

Either way, always validate your inputs.

---

## Lock Down RemoteEvents / RemoteFunctions

### The hard truth
There is **no way** to stop a client from firing a RemoteEvent. Tools like RemoteSpy let exploiters read, copy, and fire anything.  
Security happens in two places:
1. **Where remotes exist** (hide them)
2. **What they accept** (validate everything)

### Create remotes server-side
Instead of placing RemoteEvents in ReplicatedStorage, create them from a ModuleScript in ServerScriptService and parent them to a hidden folder at runtime.

```lua
-- ServerScriptService/RemoteConfig.luau
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Remotes = {}

function Remotes.create(name, isClientToServer)
    local remote = Instance.new("RemoteEvent")
    remote.Name = name
    remote.Parent = ReplicatedStorage.Remotes -- hidden container
    -- direction bookkeeping happens here if you want (see §9)
    return remote
end

return Remotes
```

The key win: RemoteSpy shows nothing in ReplicatedStorage at publish time. Exploiters cannot discover what you named your remotes without runtime inspection (still bypassable — defense in depth).

### Type checking — the absolute minimum
Use `typeof()` on every argument, every time.

```lua
CacheInputRE.OnServerEvent:Connect(function(player, key: string, state: boolean)
    if typeof(key) ~= "string" or typeof(state) ~= "boolean" then
        player:Kick("Invalid action parameters")
        return
    end
    -- process
end)
```

### Table argument validation
Tables are where exploits most often hide. Two rules:
- Validate **each index you actually use**, don't iterate blindly over the whole table
- Reject extremely large tables; cap length with `table.maxn` for arrays

```lua
BuyItemRE.OnServerEvent:Connect(function(player, payload)
    if typeof(payload) ~= "table" then return end
    if typeof(payload.ItemId) ~= "string" then return end
    if #payload > 10 then return end  -- bizarrely large? reject
    -- proceed
end)
```

---

## Copy-Proof Your Important Scripts

- **Server-side scripts only live in ServerScriptService** — never replicated.
- **Move ModuleScripts into ServerScriptService** before implementation, use them via `require()` from server scripts.
- **Luau is compiled bytecode**, not plain Lua — it offers mild obfuscation without runtime cost.
- **Obfuscation tools** add another layer (e.g. luax, luac, bytenode ship workflows) but never rely on obfuscation as the security boundary.

Rule of thumb: if a script contains anti-exploit or critical economy logic, assume the client can see it and design accordingly.

---

## Data Integrity: Never Trust The Client

This pattern prevents 80% of all client exploit impact.

```lua
-- ❌ NEVER: server lets client name the target
RE.FireAt.OnServerEvent:Connect(function(player, targetInstance, damage)
    targetInstance.Humanoid:TakeDamage(damage)
end)

-- ✅ ALWAYS: server looks up the target itself, clamps from server config
RE.RequestAttack.OnServerEvent:Connect(function(player, targetName)
    local target = workspace.Enemies:FindFirstChild(targetName)
    if not target then return end

    local dist = (target.HumanoidRootPart.Position -
                  player.Character.HumanoidRootPart.Position).Magnitude
    if dist > 25 then return end  -- long-range kill blocked

    target.Humanoid:TakeDamage(ServerConfig.BASE_DAMAGE)  -- server value
end)
```

**No sensitive number ever travels "up" from client → server.** The client only says *"I want to do the thing"*, and the server decides *"you're allowed, here's how it resolves"*.

---

## Sanity / Edge-Case Checking

Tied heavily to §5, but expanded here.

| Check | Tool | When |
|-------|------|------|
| Is it an Instance / model, is it descendant of workspace? | `type`, `.IsDescendantOf()` | User sent an object |
| Is number between 0 and sane max? | `<`, `math.clamp` | Dmg, speed, count |
| Is string short enough / in allowlist? | `string.len`, `table.find` | Keys, names |
| Is table length bounded? | `table.maxn` | Inventory arrays |
| No NaN / `math.huge` / negative? | `~isnan`, range comparison | Matrix math inputs |
| Call occurs in expected state? | Zustand / server context check | Round is active |

```lua
-- NaN check snippet worth having on hand
local function isSafeNumber(n: number): boolean
    return typeof(n) == "number"
        and not (nan(n) or n == math.huge or n == -math.huge)
        and n > 0
end
```

---

## Rate Limiting

**The only valid rate limiter runs inside the remote handler, on the server, using your own tracked data.** Client-told cooldowns are a joke.

```lua
local COOLDOWN_S = 0.2
local RemoteCooldown = {}

game.Players.PlayerAdded:Connect(function(p)
    RemoteCooldown[p.UserId] = 0
end)
game.Players.PlayerRemoving:Connect(function(p)
    RemoteCooldown[p.UserId] = nil
end)

RE.OnServerEvent:Connect(function(player, ...)
    local now = os.clock()
    if now - RemoteCooldown[player.UserId] < COOLDOWN_S then
        return  -- silently ignore
    end
    RemoteCooldown[player.UserId] = now
    -- ... process
end)
```

Why `os.clock()` over `tick()`? `tick()` is wall-clock; `os.clock()` is monotonically increasing — no NTP drift resets throw off the cooldown.

### Table-based alternative (no per-player storage)
```lua
local Waiters = {}

RE.OnServerEvent:Connect(function(player, ...)
    if Waiters[player.UserId] then return end
    Waiters[player.UserId] = true
    task.spawn(function()
        task.wait(COOLDOWN_S)
        Waiters[player.UserId] = nil
    end)
    -- ... process
end)
```

---

## Remote Direction Enforcing

Every RemoteEvent has exactly one job: C2S or S2C. Never both.

```lua
local REMOTE_DIRECTIONS = {
    PlayerJoinCE = "C2S",
    PingRE       = "S2C",
    -- ...
}

for name, dir in REMOTE_DIRECTIONS do
    local remote = ReplicatedStorage.Remotes:FindFirstChild(name)
    if remote then
        remote.OnServerEvent:Connect(function(player, ...)
            if dir == "S2C" then
                player:Kick("Direction violation")  -- fired C2S on S2C
            end
            -- process
        end)
    end
end
```

Combined with a "honeypot" trick: add a RemoteEvent named something tempting like `GrantAdminRE` to ReplicatedStorage; on server, make it directly call `player:Kick()` + log. The first thing every exploiter does is fire every remote; this catches them before your gameplay logic even starts.

---

## Instances: ReplicatedStorage vs ServerScriptService vs PlayerScripts

| Location | Visible to client? | When to put what there |
|----------|--------------------|-----------------------|
| **ServerScriptService** | **No** | All anti-exploit code, economy logic, remote handlers |
| **ReplicatedStorage** | **Yes** | RemoteEvent/Function definitions, shared data tables, assets |
| **PlayerScripts** | Per-player: yes | LocalScripts, InputAction setup |
| **StarterPlayerScripts** | Per-player yes (on spawn) | Default client scripts |

Core rule: if the client has any use for it, assume exploiters can read it. Anti-exploit logic belongs server-only.

---

## Anti-Exploit Patterns

### 1. Distance-sanity on any "interact with X" request
```lua
local MAX_INTERACT_DIST = 12

local function raidDistanceValid(player: Player, target: Instance): boolean
    local char = player.Character
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local thrp = target:FindFirstChild("HumanoidRootPart")
    if not hrp or not thrp then return false end
    return (hrp.Position - thrp.Position).Magnitude <= MAX_INTERACT_DIST
end
```

### 2. Allowed-values allowlist instead of raw comparison
```lua
local ALLOWED_ITEMS = {"sword", "potion", "shield"}

RE.BuyItem.OnServerEvent:Connect(function(player, itemName)
    if not table.find(ALLOWED_ITEMS, itemName) then return end
    -- safe to process
end)
```

### 3. Admin/privileged remotes — UserId gate
```lua
local ADMIN_IDS = {123456, 789012}

RE.AdminCommand.OnServerEvent:Connect(function(player)
    if not table.find(ADMIN_IDS, player.UserId) then
        player:Kick("Unexpected admin remote access")
    end
end)
```

### 4. Walking away from `pcall` as "security"
`pcall` swallows errors. Never use `pcall` as a detection mechanism. Errors inside `pcall` do not indicate tampering — they indicate something went wrong, and silently absorbing them lets exploits pass under the radar.

### 5. The honeypot remote
```lua
-- A deliberately tempting remote that no real player would ever find:
local HONEYPOT = Instance.new("RemoteEvent")
HONEYPOT.Name = "GiveAdminPermissions"
HONEYPOT.Parent = ReplicatedStorage.Remotes
HONEYPOT.OnServerEvent:Connect(function(player)
    -- directly to moderation: log + kick/ban
    player:Kick("Remote firewall triggered")
    -- log to DataStore/discord webhook etc
end)
```

---

## Input Action System (IAS)

### What it is
The Input Action System (IAS) is the client-side input event system that feeds trusted data into the server. Outside IAS, data is cancer.

```lua
-- client (LocalScript)
local ContextActionService = game:GetService("ContextActionService")

local InputAction = ContextActionService:BindAction("PlayerShoot", function(actionName, inputState, input)
    if inputState == Enum.UserInputState.Begin then
        ShootRE:FireServer(input.Position)
    end
    return Enum.ContextActionResult.Sink
end, false, Enum.KeyCode.ButtonR2, Enum.KeyCode.MouseButton1)
```

### Server-side handling pattern
```lua
ShootRE.OnServerEvent:Connect(function(player, aimPos: Vector3)
    if typeof(aimPos) ~= "Vector3" then return end
    if typeof(player.Character) ~= "Model" then return end

    -- Range clamp, do not trust client's exact vector
    local state = ServerProjectileHandler.spawn(
        player,
        player.Character.HumanoidRootPart.Position,
        (aimPos - player.Character.HumanoidRootPart.Position).Unit
    )
end)
```

Important: even "trusted" IAS inputs must be range-checked, like any remote. Sanity checks don't go away.

---

## Quick Checklist (print-mode, run before publish)

```
[ ] FilteringEnabled: true (no Experimental Mode)
[ ] Server-side state exists and is authoritative
[ ] Every RemoteEvent has type checks + sanity checks
[ ] RemoteEvent table args length-capped + index-validated
[ ] Rate limiter is active (≤ 0.2s per-player)
[ ] Remote events are NOT in ReplicatedStorage by default
[ ] Economy values never come from client; server computes them
[ ] Sensitive remotes have UserId gate
[ ] Honeypot remote(s) installed
[ ] Sensitive scripts live in ServerScriptService only
[ ] pcall is used for error safety, NOT as exploit detection
[ ] Input Action System is wired up for anything touching gameplay
```

---

## Sources and further reading
- [Best way to protect RemoteEvents against exploiters? — DevForum](https://devforum.roblox.com/t/best-way-to-protect-remoteevents-against-exploiters/1314739)
- [How to secure remote events better from exploiters? — DevForum](https://devforum.roblox.com/t/how-to-secure-remote-events-better-from-exploiters/554772)
- [A Comprehensive Guide To Airtight Remote Security — DevForum](https://devforum.roblox.com/t/a-comprehensive-guide-to-airtight-remote-security/3079489)
- [How to secure your RemoteEvent and RemoteFunction — DevForum](https://devforum.roblox.com/t/how-to-secure-your-remoteevent-and-remotefunction/3345363)
- [Stop discouraging client side security — boyned's blog](https://blog.boyned.com/articles/stop-discouraging-client-side-security/)
- [Server Authority: What Developers Need to Know — Last Level Studios](https://lastlevel.co.uk/blog/server-authority-what-developers-need-to-know)
- [Server Authority: How to Begin? — DevForum](https://devforum.roblox.com/t/server-authority-how-to-begin/4139185)
- [Early Access Server Authority — DevForum](https://devforum.roblox.com/t/early-access-server-authority/3983188)

---

*Built from first-party Roblox docs, DevForum consensus, and community synthesis. Last updated June 2026.*