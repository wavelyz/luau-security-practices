# luau-security-practices

Practical, opinionated Roblox–Luau security guide: what to lock down, why, and how to do it in 10 minutes or less.

If you’ve ever wondered how to keep your Roblox game safe from exploiters without drowning in theory, this guide is for you. Think of it as a quick‑read cheat sheet that walks you through the most effective, battle‑tested practices—straight from the DevForum, official docs, and hard‑won experience.

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

These are the non‑negotiable basics that every Roblox developer should internalize before writing a single line of game logic.

- **Never trust the client.** Any data that arrives via a RemoteEvent or RemoteFunction is hostile until you verify it.
- **The server is the source of truth.** All authoritative state lives on the server; the client merely displays it.
- **Always wrap remote handlers in `task.spawn()`** to avoid yielding or queue‑exhaustion panics if something accidentally yields.

> *Note:* FilteringEnabled has been on by default for years and cannot be turned off, so you don’t need to worry about it—just assume client‑to‑server replication only happens through your explicit remotes.

---

## The Client–Server Model

### Who does what?

| Side | Responsibility |
|------|----------------|
| **Server** | Runs all game logic, validates input, holds the authoritative state |
| **Client** | Collects input, plays visual effects, runs local‑only predictions |

Anything the client creates on its own disappears when the player leaves; only server‑created instances replicate.

### The request‑action pattern

The cleanest way to think about communication is: the client **requests** an action, and the server **performs** it—or refuses it.

```lua
-- ❌ BAD – letting the client decide the outcome
RE.AddCash.OnServerEvent:Connect(function(player, amount)
    player.leaderstats.Cash.Value += amount  -- exploiter’s dream
end)

-- ✅ GOOD – client asks, server decides
RE.RequestPickup.OnServerEvent:Connect(function(player, pickupId)
    local part = workspace:FindFirstChild(pickupId)
    if not part then return end
    part:Destroy()               -- server owns the destroy
    player.leaderstats.Cash.Value += 5  -- reward comes from server config
end)
```

---

## Server Authority

Roblox’s Server Authority (still in early access) flips the script: the server becomes the **only** entity that can move a character. This shuts down whole classes of exploits—speedhacks, flyhacks, noclip—by removing the client’s ability to set position or velocity directly.

### What it means in one sentence
Server authority lets the server validate every piece of player input and every movement calculation; clients can only *predict* locally for smooth visuals.

### Why it matters
- **Speedhacks** require the client to set velocity → blocked
- **Flyhacks** rely on teleporting or welding to invisible parts → blocked
- **Noclip** tries to disable collision locally → blocked at the replication level

### Getting it running (early access)
1. Enable **Server Authority Core API** via *File > Beta Features*.
2. Set these Workspace properties:
   - `StreamingEnabled = true`
   - `UseFixedSimulation = true`
   - `NextGenerationReplication = true`
3. In Workspace → Server Authority, set `AuthorityMode = Server`.

⚠️ Remember: this is still early access—games using it cannot be published yet, and the APIs may change.

### With vs. without Server Authority
- **Without it:** you must manually reject any movement‑based remotes.
- **With it:** you can’t touch the character physically at all; only server‑driven inputs move it.

Either way, *always* validate your inputs—Server Authority doesn’t replace sanity checks.

---

## Locking Down RemoteEvents / RemoteFunctions

### The hard truth
You cannot stop a determined exploiter from firing a RemoteEvent; tools like RemoteSpy let them read, copy, and invoke anything. Real security happens in two places:

1. **Where the remotes live** (keep them hidden)
2. **What they accept** (validate every argument)

### Create remotes on the server
Instead of dropping RemoteEvents into ReplicatedStorage at publish time, generate them from a ModuleScript in ServerScriptService and parent them to a hidden folder at runtime.

```lua
-- ServerScriptService/RemoteConfig.luau
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local Remotes = {}

function Remotes.create(name, isClientToServer)
    local remote = Instance.new("RemoteEvent")
    remote.Name = name
    remote.Parent = ReplicatedStorage.Remotes   -- hidden container
    -- (optional) track direction here – see §9
    return remote
end

return Remotes
```

The payoff: RemoteSpy shows an empty `ReplicatedStorage.Remotes` at publish time, forcing exploiters to guess names or inspect at runtime—defense in depth.

### Type checking – the bare minimum
Validate **every** argument with `typeof()`, every single time.

```lua
CacheInputRE.OnServerEvent:Connect(function(player, key: string, state: boolean)
    if typeof(key) ~= "string" or typeof(state) ~= "boolean" then
        player:Kick("Invalid action parameters")
        return
    end
    -- …process the request
end)
```

### Table argument validation
Tables are where exploits love to hide malicious payloads. Two simple rules:
- Only validate the indices you actually **use**—don’t blindly iterate over the whole table.
- Reject absurdly large tables; cap length with `table.maxn` for arrays.

```lua
BuyItemRE.OnServerEvent:Connect(function(player, payload)
    if typeof(payload) ~= "table" then return end
    if typeof(payload.ItemId) ~= "string" then return end
    if #payload > 10 then return end   -- wildly oversized? drop it
    -- …process the purchase
end)
```

---

## Making Important Scripts Hard to Copy

- Keep **all server‑side code** in `ServerScriptService`—never replicated.
- Place **ModuleScripts** there too and `require()` them from server scripts.
- Luau compiles to bytecode, which already offers light obfuscation at zero runtime cost.
- If you want extra layers, tools like `luax`, `luac`, or `bytenode` exist—but never rely on obfuscation as your security boundary.

**Rule of thumb:** If a script handles anti‑exploit logic, economy, or anything critical, assume the client can see it and design accordingly.

---

## Data Integrity: Never Trust the Client

This single pattern stops roughly 80 % of typical exploit impact.

```lua
-- ❌ NEVER – letting the client name the target and damage
RE.FireAt.OnServerEvent:Connect(function(player, targetInstance, damage)
    targetInstance.Humanoid:TakeDamage(damage)
end)

-- ✅ ALWAYS – server looks up the target and uses its own values
RE.RequestAttack.OnServerEvent:Connect(function(player, targetName)
    local target = workspace.Enemies:FindFirstChild(targetName)
    if not target then return end

    local dist = (target.HumanoidRootPart.Position -
                  player.Character.HumanoidRootPart.Position).Magnitude
    if dist > 25 then return end   -- long‑range kill blocked

    target.Humanoid:TakeDamage(ServerConfig.BASE_DAMAGE)  -- server‑defined
end)
```

**Never** let a sensitive number (damage, currency, stats) travel *up* from client to server. The client only says *“I want to do the thing”* and the server decides *“you’re allowed, here’s how it resolves.”*

---

## Sanity / Edge‑Case Checking

Think of these as the extra locks on your door.

| Check | How to do it | When it matters |
|-------|--------------|-----------------|
| Is it an Instance/descendant of workspace? | `typeof(obj) == "Instance"` and `obj:IsDescendantOf(workspace)` | User sent an object |
| Is a number in a sane range? | `<`, `>`, `math.clamp` | Damage, speed, count |
| Is a string short enough or in an allowlist? | `string.len`, `table.find` | Keys, names, commands |
| Is a table length bounded? | `table.maxn` | Inventory arrays, lists |
| No NaN / `math.huge` / negative? | `~isnan`, explicit range checks | Any math input |
| Does the call fit the current game state? | Compare against a round‑state variable | Only allow during active rounds, not in lobby |

**NaN‑check helper** (worth copying into a utility module):

```lua
local function isSafeNumber(n: number): boolean
    return typeof(n) == "number"
        and not (nan(n) or n == math.huge or n == -math.huge)
        and n > 0
end
```

---

## Rate Limiting

The only rate limiter that actually works lives **inside** the remote handler, on the server, using your own tracked data. Client‑enforced cooldowns are trivial to bypass.

```lua
local COOLDOWN_S = 0.2          -- tweak per‑remote as needed
local RemoteCooldown = {}

game.Players.PlayerAdded:Connect(function(p)
    RemoteCooldown[p.UserId] = 0
end)
game.Players.PlayerRemoving:Connect(function(p)
    RemoteCooldown[p.UserId] = nil
end)

RE.OnServerEvent:Connect(function(player, ...)
    local now = os.clock()               -- monotonic, immune to NTP jumps
    if now - RemoteCooldown[player.UserId] < COOLDOWN_S then
        return                           -- silently ignore excess fires
    end
    RemoteCooldown[player.UserId] = now
    -- …process the legitimate request
end)
```

### Table‑based alternative (no per‑player storage)
```lua
local Waiters = {}

RE.OnServerEvent:Connect(function(player, ...)
    if Waiters[player.UserId] then return end
    Waiters[player.UserId] = true
    task.spawn(function()
        task.wait(COOLDOWN_S)
        Waiters[player.UserId] = nil
    end)
    -- …process
end)
```

---

## Enforcing Remote Direction

Every RemoteEvent should have **exactly one** job: either Client→Server **or** Server→Client—never both. Mixing directions is a dead giveaway of exploitation.

```lua
local REMOTE_DIRECTIONS = {
    PlayerJoinCE = "C2S",
    PingRE       = "S2C",
    -- add the rest of your remotes here
}

for name, dir in REMOTE_DIRECTIONS do
    local remote = ReplicatedStorage.Remotes:FindFirstChild(name)
    if remote then
        remote.OnServerEvent:Connect(function(player, ...)
            if dir == "S2C" then
                player:Kick("Direction violation")   -- fired C2S on an S2C‑only remote
            end
            -- …process legitimate C2S traffic
        end)
    end
end
```

### The honeypot trick
Add a deliberately tempting RemoteEvent—say, `GiveAdminPermissions`—to `ReplicatedStorage.Remotes`. On the server, make it immediately kick (or ban) and log the offender. Since most exploiters start by firing every remote they can find, this catches them before they even reach your real gameplay logic.

---

## Where Things Live: ReplicatedStorage vs ServerScriptService vs PlayerScripts

| Location | Visible to client? | What belongs here |
|----------|--------------------|-------------------|
| **ServerScriptService** | **No** | All anti‑exploit code, economy logic, remote handlers |
| **ReplicatedStorage** | **Yes** | RemoteEvent/Function definitions, shared data tables, assets |
| **PlayerScripts** | Per‑player: yes | LocalScripts, InputAction setup |
| **StarterPlayerScripts** | Per‑player (on spawn) | Default client scripts |

**Core rule:** If the client has any legitimate use for something, assume an exploiter can read it. Keep your security‑critical code strictly in `ServerScriptService`.

---

## Proven Anti‑Exploit Patterns

### 1. Distance sanity on any “interact with X” request
```lua
local MAX_INTERACT_DIST = 12   -- tweak to your game’s feel

local function raidDistanceValid(player: Player, target: Instance): boolean
    local char = player.Character
    if not char then return false end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local thrp = target:FindFirstChild("HumanoidRootPart")
    if not hrp or not thrp then return false end
    return (hrp.Position - thrp.Position).Magnitude <= MAX_INTERACT_DIST
end
```

### 2. Allow‑list instead of raw comparison
```lua
local ALLOWED_ITEMS = {"sword", "potion", "shield"}

RE.BuyItem.OnServerEvent:Connect(function(player, itemName)
    if not table.find(ALLOWED_ITEMS, itemName) then return end
    -- safe to process the purchase
end)
```

### 3. Admin/privileged remotes – UserId gate
```lua
local ADMIN_IDS = {123456, 789012}

RE.AdminCommand.OnServerEvent:Connect(function(player)
    if not table.find(ADMIN_IDS, player.UserId) then
        player:Kick("Unexpected admin remote access")
    end
end)
```

### 4. Avoid using `pcall` as a security mechanism
`pcall` merely catches errors; it does **not** tell you whether the client is tampering. Swallowing errors silently can actually hide exploits. Use `pcall` only for genuine error safety (e.g., HTTP requests), never as an exploit detector.

### 5. The honeypot remote (see §9)
A deliberately named remote that no honest player would ever encounter, tied directly to kick/ban/logic.

---

## Input Action System (IAS) – the trusted client path

The Input Action System is the **only** client‑to‑server channel that Roblox marks as trusted. Anything outside IAS (e.g., raw RemoteEvents you create yourself) must be treated as hostile.

### Client side (LocalScript)
```lua
local ContextActionService = game:GetService("ContextActionService")

local InputAction = ContextActionService:BindAction(
    "PlayerShoot",
    function(actionName, inputState, input)
        if inputState == Enum.UserInputState.Begin then
            ShootRE:FireServer(input.Position)
        end
        return Enum.ContextActionResult.Sink
    end,
    false,                -- not create touch button
    Enum.KeyCode.ButtonR2, Enum.KeyCode.MouseButton1
)
```

### Server side – still validate!
```lua
ShootRE.OnServerEvent:Connect(function(player, aimPos: Vector3)
    if typeof(aimPos) ~= "Vector3" then return end
    if typeof(player.Character) ~= "Model" then return end

    -- Even “trusted” IAS input needs range checks
    local launchPos = player.Character.HumanoidRootPart.Position
    local direction = (aimPos - launchPos).Unit

    local state = ServerProjectileHandler.spawn(player, launchPos, direction)
end)
```

**Bottom line:** treat IAS data like any other remote—apply the same type, range, and sanity checks.

---

## Quick Pre‑Publish Checklist

Run through this before you hit *Publish*:

```
[ ] Never trust the client – validate every Remote/Function argument
[ ] Server is the source of truth – authoritative state lives server‑side
[ ] Use task.spawn() inside remote handlers to avoid yield/queue issues
[ ] Hide remotes: create them server‑side, not in ReplicatedStorage at publish time
[ ] Type‑check every argument with typeof()
[ ] Validate table arguments: only use needed indices, cap length with table.maxn
[ ] Enforce remote direction (C2S or S2C only) – kick violators
[ ] Install a honeypot remote (e.g., GiveAdminPermissions) to catch exploiters early
[ ] Rate‑limit remotes on the server (≤0.2s per‑player, using os.clock)
[ ] Keep anti‑exploit/economy code in ServerScriptService only
[ ] Use the Input Action System for trusted client input – still verify!
[ ] Never use pcall as an exploit detection mechanism
[ ] Apply distance, allow‑list, and UserId‑gate patterns where relevant
[ ] Confirm Server Authority is enabled (if targeting early access) or manually validate movement remotes
```

---

## Sources & Further Reading

- [Best way to protect RemoteEvents against exploiters? — DevForum](https://devforum.roblox.com/t/best-way-to-protect-remoteevents-against-exploiters/1314739)
- [How to secure remote events better from exploiters? — DevForum](https://devforum.roblox.com/t/how-to-secure-remote-events-better-from-exploiters/554772)
- [A Comprehensive Guide To Airtight Remote Security — DevForum](https://devforum.roblox.com/t/a-comprehensive-guide-to-airtight-remote-security/3079489)
- [How to secure your RemoteEvent and RemoteFunction — DevForum](https://devforum.roblox.com/t/how-to-secure-your-remoteevent-and-remotefunction/3345363)
- [Stop discouraging client side security — boyned's blog](https://blog.boyned.com/articles/stop-discouraging-client-side-security/)
- [Server Authority: What Developers Need to Know — Last Level Studios](https://lastlevel.co.uk/blog/server-authority-what-developers-need-to-know)
- [Server Authority: How to Begin? — DevForum](https://devforum.roblox.com/t/server-authority-how-to-begin/4139185)
- [Early Access Server Authority — DevForum](https://devforum.roblox.com/t/early-access-server-authority/3983188)

---

*Built from first‑party Roblox docs, DevForum consensus, and community synthesis. Last updated June 2026.*