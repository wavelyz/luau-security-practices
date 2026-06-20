# luau-security-practices

This is just a guide post researched by like 20 AI models so, do the research yourself, sources are cited at the bottom.

---

## The basics you can’t skip

Never trust anything that comes from a client. Every RemoteEvent, every RemoteFunction, treat it as dangerous until you prove it’s safe.

The server owns the truth. All game state, all decisions, live there. The client just shows what the server says.

Wrap every remote handler in `task.spawn()`. If something yields by accident you won’t choke the queue.

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
    local part = workspace.Gems:FindFirstChild(gemId)
    if not part then return end
    part:Destroy()
    plr.leaderstats.Gems += 5   -- reward comes from server config
end)
```

---

## Server Authority (still early access)

If you’re in the Server Authority beta, the server now **alone** can move a character. No more client‑set velocity or position. That kills speed hacks, fly hacks, noclip out of the gate.

How to turn it on:
1. File → Beta Features → check **Server Authority Core API**
2. In Workspace set:
   * StreamingEnabled = true
   * UseFixedSimulation = true
   * NextGenerationReplication = true
3. Open Workspace → Server Authority → set AuthorityMode = Server

Remember: you can’t publish yet while it’s in beta, and the APIs might shift. Even with Server Authority on you still need to validate every input—don’t skip sanity checks.

---

## Locking down your remotes

You can’t stop an exploiter from firing a RemoteEvent, tools like RemoteSpy let them see and copy anything. Real protection lives in two places:
* where the remotes live (keep them hidden)
* what they accept (check every argument)

Make remotes on the server, not in ReplicatedStorage at publish time:
```lua
-- ServerScriptService/RemoteConfig.lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Remotes = {}

function Remotes.create(name, clientToServer)
    local r = Instance.new("RemoteEvent")
    r.Name = name
    r.Parent = ReplicatedStorage.Remotes   -- hidden folder
    return r
end

return Remotes
```

Now RemoteSpy shows an empty folder at publish time. Attackers have to guess names or dig at runtime—defense in depth.

### Type checks – do them every time
```lua
RE.Move.OnServerEvent:Connect(function(plr, dir: Vector3, speed: number)
    if typeof(dir) ~= "Vector3" or typeof(speed) ~= "number" then
        plr:Kick("Weird data")
        return
    end
    -- …
end)
```

### Table arguments – only touch what you need
```lua
RE.Buy.OnServerEvent:Connect(function(plr, payload)
    if typeof(payload) ~= "table" then return end
    if typeof(payload.ItemId) ~= "string" then return end
    if #payload > 8 then return end   -- too big? drop it
    -- …
end)
```

---

## Keep your important code hidden

Put every server‑side script, sanity checking logic, economy handlers, remote modules—inside **ServerScriptService**. Never replicate it.

ModuleScripts go there too; `require()` them from server code.

If a script handles money, stats, or anything that matters, assume an attacker can read it and build accordingly.

---

## The “never trust the client” pattern in action

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
    local target = workspace.Enemies:FindFirstChild(targetName)
    if not target then return end

    local distance = (target.HumanoidRootPart.Position -
                      plr.Character.HumanoidRootPart.Position).Magnitude
    if distance > 20 then return end   -- too far? ignore

    target.Humanoid:TakeDamage(ServerConfig.BASE_DMG)  -- server value
end)
```

---

## Quick sanity checks you should actually use

| What to check | How |
|---------------|-----|
| Is it an Instance and inside workspace? | `typeof(obj)=="Instance" and obj:IsDescendantOf(workspace)` |
| Number in a sane range? | `<`, `>`, `math.clamp` |
| String short enough or in an allow list? | `string.len`, `table.find` |
| Table not absurdly large? | `table.maxn` |
| No NaN or infinities? | explicit `~isnan` and range checks |
| Does this call make sense right now? | compare against a round‑state flag |

Tiny helper you might drop in a module:
```lua
local function isGoodNum(n)
    return typeof(n)=="number" and not (nan(n) or n==math.huge or n==-math.huge) and n>0
end
```

---

## Rate limiting – do it on the server

Client‑side cooldowns are useless. Track timing yourself inside the remote handler.

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
    if now - lastCall[plr.UserId] < COOLDOWN then
        return                  -- silently ignore spam
    end
    lastCall[plr.UserId] = now
    -- …process the real request
end)
```

If you don’t want a per‑player table you can use a flag + task.delay, but the per‑player version is straightforward and effective.

---

## Enforce remote direction – one way only

A RemoteEvent should only ever be Client→Server **or** Server→Client, never both. Mixing directions is a dead giveaway of exploitation.

```lua
local REMOTE_DIR = {
    JoinEvent = "C2S",
    PingEvent = "S2C",
    -- fill in the rest
}

for name, dir in REMOTE_DIR do
    local r = ReplicatedStorage.Remotes:FindFirstChild(name)
    if r then
        r.OnServerEvent:Connect(function(plr, ...)
            if dir == "S2C" then
                plr:Kick("Wrong way")   -- fired C2S on an S2C‑only remote
                return
            end
            -- …handle legitimate traffic
        end)
    end
end
```

### Honey pot trick
Drop a RemoteEvent with a tempting name like `GiveAdminPerms` into `ReplicatedStorage.Remotes`. On the server make it instantly kick (or ban) and log the offender. Most exploiters start by firing every remote they can find—this catches them before they even reach your real gameplay logic.

---

## Where things live – quick reference

| Folder | Visible to client? | Put what here |
|--------|--------------------|---------------|
| ServerScriptService | **No** | All anti‑exploit code, economy logic, remote handlers |
| ReplicatedStorage | **Yes** | RemoteEvent/Function definitions, shared data, assets |
| PlayerScripts | Per‑player: yes | LocalScripts, InputAction setup |
| StarterPlayerScripts | Per‑player (on spawn) | Default client scripts |

If the client has a legitimate reason to read something, assume an attacker can read it. Keep security‑critical stuff strictly in ServerScriptService.

---

## Proven patterns that actually work

**Distance check on any “interact with X” request**
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

**Allow‑list instead of raw equality**
```lua
local ALLOWED_TOOLS = {"sword", "shield", "potion"}

RE.PickUp.OnServerEvent:Connect(function(plr, toolName)
    if not table.find(ALLOWED_TOOLS, toolName) then return end
    -- …process
end)
```

**UserId gate for privileged remotes**
```lua
local STAFF_IDS = {1234567, 8901234}

RE.AdminCmd.OnServerEvent:Connect(function(plr)
    if not table.find(STAFF_IDS, plr.UserId) then
        plr:Kick("Nice try")
    end
end)
```

**Avoid using pcall as an exploit detector**  
`pcall` just catches errors; it doesn’t tell you whether the client is tampering. Swallowing errors silently can actually hide abuse. Use `pcall` only for genuine safety (like network calls), never as a security mechanism.

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

Server side – still verify!
```lua
ShootRE.OnServerEvent:Connect(function(plr, aimPos: Vector3)
    if typeof(aimPos) ~= "Vector3" then return end
    if typeof(plr.Character) ~= "Model" then return end

    -- Even IAS data needs range checks
    local start = plr.Character.HumanoidRootPart.Position
    local dir = (aimPos - start).Unit

    ServerWeapon.Spawn(plr, start, dir)
end)
```

Treat IAS data like any other remote: run the same type, range, and sanity checks.

---

## Quick pre‑publish glance

Glance at this before you hit Publish:

- Never trust the client – validate every Remote/Function argument
- Server is the source of truth – authoritative state lives server‑side
- Wrap remote handlers in task.spawn() to avoid yield/queue trouble
- Hide remotes: create them server‑side, not in ReplicatedStorage at publish time
- Type‑check every argument with typeof()
- Validate table arguments: use only needed indices, cap length with table.maxn
- Enforce remote direction (C2S **or** S2C only) – kick violators
- Install a honeypot remote (e.g., GiveAdminPerms) to catch exploiters early
- Rate‑limit remotes on the server (≤0.2 s per player, using os.clock)
- Keep anti‑exploit/economy code in ServerScriptService only
- Use the Input Action System for trusted client input – still verify!
- Never use pcall as an exploit detection mechanism
- Apply distance, allow‑list, and UserId‑gate patterns where relevant
- If targeting Server Authority, enable it per the beta steps; otherwise manually validate any movement‑related remotes

---

That’s it. Go make your game less of a free‑for‑all. If something feels off, tweak it, test it, repeat. No magic, just solid habits.
