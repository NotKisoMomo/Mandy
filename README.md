```
 ███╗   ███╗ █████╗ ███╗   ██╗██████╗ ██╗   ██╗
 ████╗ ████║██╔══██╗████╗  ██║██╔══██╗╚██╗ ██╔╝
 ██╔████╔██║███████║██╔██╗ ██║██║  ██║ ╚████╔╝ 
 ██║╚██╔╝██║██╔══██║██║╚██╗██║██║  ██║  ╚██╔╝  
 ██║ ╚═╝ ██║██║  ██║██║ ╚████║██████╔╝   ██║   
 ╚═╝     ╚═╝╚═╝  ╚═╝╚═╝  ╚═══╝╚═════╝    ╚═╝   
```

<div align="center">

[![version](https://img.shields.io/badge/version-1.0.0--alpha-black?style=for-the-badge&labelColor=0d0d0d)](https://github.com/NotKisoMomo/Mandy)
[![stability](https://img.shields.io/badge/stability-alpha-6C3EF4?style=for-the-badge&labelColor=0d0d0d)](https://github.com/NotKisoMomo/Mandy)
[![license](https://img.shields.io/badge/license-MIT-white?style=for-the-badge&labelColor=0d0d0d)](https://github.com/NotKisoMomo/Mandy/blob/main/LICENSE)
[![roblox](https://img.shields.io/badge/roblox-luau-red?style=for-the-badge&labelColor=0d0d0d)](https://create.roblox.com)
[![plinko](https://img.shields.io/badge/by-plinko%20labs-6C3EF4?style=for-the-badge&labelColor=0d0d0d)](https://github.com/NotKisoMomo)

*Type-safe. Buffer-serialized. Async-first. Built for Roblox.*

</div>

---

```
  reliable ---------------------> [Unlazy]   RemoteEvent
  unreliable -------------------> [Lazy]     UnreliableRemoteEvent
  request-response -------------> [Resolver] -> Thread
                                                  |
                                          :Next(pkg, resolve)
                                          :Toss(err, resolve)
                                          :Conclude(original)
                                          :Retry(n)
                                          :Collapse(reason?)
```

---

## What is Mandy

Mandy is a next-generation networking library for Roblox. It owns its full stack -- buffer serialization, type validation, async Thread chains, per-server signature security, bidirectional streaming, auto-replicating state, cross-server messaging, lag compensation, and a plugin system with full lifecycle hooks.

All traffic is multiplexed through 8 remotes regardless of how many mandates are defined:

```
Runtime:
  __Mandy_Lazy                UnreliableRemoteEvent
  __Mandy_Unlazy              RemoteEvent
  __Mandy_Resolver            RemoteFunction

Diagnostic:
  __Mandy_Lazy_Diagnostic     UnreliableRemoteEvent
  __Mandy_Unlazy_Diagnostic   RemoteEvent
  __Mandy_Resolver_Diagnostic RemoteFunction

Infrastructure:
  __Mandy_Bootstrap           RemoteEvent
  __Mandy_Infra               RemoteEvent
```

Both sides define the same mandate. The rest is handled.

---

## Terminology

| Term | Meaning |
|------|---------|
| `Mandate` | a defined networking control -- what `Mandy.Mandate()` returns |
| `Package` | the data sent over the wire |
| `Parses` | the shape field inside a mandate config |
| `Subscription` | a listener connection returned by `Subscribe` |
| `Term` | any callback |
| `Patch` | an intersection -- `:also()` |
| `Thread` | the Thenable async pipeline |
| `Marks` | security flags on a player |
| `Imprint` | a `Mandy.Snapshot()` dump |
| `Session` | the bootstrapped security state after `Signed` fires |
| `Recording` | a `Mandy.Record()` handle |

---

## Installation

```lua
local Mandy = require(game:GetService("ReplicatedStorage"):WaitForChild("Mandy"))
```

---

## Type System

Every constructor returns a `TypeDef` -- fluent, chainable, compiles to a buffer type on the wire.

```lua
Mandy.uint8():bet(0, 100)
Mandy.string():nullable()
Mandy.vec3():also(Mandy.any())     -- Patch -- intersection &
Mandy.string():or(Mandy.uint8())   -- union |
Mandy.string():not(Mandy.uint8())  -- negation ~
```

### Primitives
```lua
Mandy.string()    Mandy.number()    Mandy.bool()      Mandy.buff()
Mandy.float32()   Mandy.float64()   Mandy.uint8()     Mandy.uint16()
Mandy.uint32()    Mandy.int8()      Mandy.int16()     Mandy.int32()
Mandy.any()       Mandy.never()     Mandy.unknown()   Mandy.nil_()
Mandy.None        -- zero-payload -- use as Parses = Mandy.None
```

### Roblox Types
```lua
Mandy.vec2()        Mandy.vec3()           Mandy.vec2int16()
Mandy.cframe()      Mandy.color3()         Mandy.udim()
Mandy.udim2()       Mandy.rect()           Mandy.region3()
Mandy.numberRange() Mandy.numberSequence() Mandy.colorSequence()
Mandy.tweenInfo()   Mandy.enumItem()
Mandy.inst()        Mandy.inst("Part")
```

### Composites
```lua
Mandy.struct({ Name = Mandy.string(), Health = Mandy.uint8() })
Mandy.array(Mandy.string())
Mandy.tuple(Mandy.string(), Mandy.uint8())
Mandy.map(Mandy.string(), Mandy.float32())
Mandy.record(Mandy.string())
Mandy.lazy(function() return Mandy.struct({ ... }) end)
Mandy.shape({ ... })
Mandy.partial(s)              Mandy.pick(s, { "Name" })
Mandy.omit(s, { "Health" })   Mandy.merge(sA, sB)
```

### Modifiers

| Modifier | Description |
|----------|-------------|
| `:or(t)` | union \| |
| `:also(t)` | Patch & |
| `:not(t)` | negation |
| `:nullable()` | may be nil |
| `:default(v)` | substitute if nil |
| `:where(fn)` | custom predicate |
| `:literal(v)` | must equal exactly |
| `:enum({ })` | must be one of |
| `:min(n)` / `:max(n)` | numeric bounds |
| `:bet(min, max)` | between |
| `:step(n)` | multiple of n |
| `:minLength(n)` / `:maxLength(n)` | string length |
| `:pattern(str)` | Lua pattern |
| `:minSize(n)` / `:maxSize(n)` | collection size |
| `:tag(str)` | nominal brand |
| `:describe(str)` | error label |

### Custom Types
```lua
local UserId = Mandy.defineType({
    Mask     = Mandy.uint32,
    Validate = function(value)
        return type(value) == "number" and value > 0
    end,
})

UserId():bet(1, 999999)
```

---

## Defining Mandates

```lua
-- Global -- no handle
Mandy.Define("PlayerHit", {
    Type      = Mandy.Unlazy,
    Modifiers = { Mandy.Linger, Mandy.Distinct },
    Security  = { RateLimit = { MaxPerSecond = 10 }, Validate = true },
    Parses    = {
        Origin = Mandy.vec3(),
        Damage = Mandy.uint8():bet(0, 100),
        Tags   = Mandy.array(Mandy.string()):nullable(),
    },
})

-- Mandate handle
local Damage = Mandy.Mandate("Damage", {
    Type    = Mandy.Resolver,
    Timeout = 5,
    Retries = 2,
    Parses  = { Amount = Mandy.uint8():bet(0, 100), Origin = Mandy.vec3() },
})

-- Handle from existing mandate
local Damage = Mandy.CreateHandle("Damage")

-- Zero-payload
Mandy.Define("Ping", { Type = Mandy.Unlazy, Parses = Mandy.None })

-- Bulk
Mandy.Load({
    { Name = "Chat", Config = { Type = Mandy.Unlazy, Parses = { Text = Mandy.string() } } },
    { Name = "Move", Config = { Type = Mandy.Lazy,   Parses = { Pos  = Mandy.vec3()   } } },
})
```

### Mandate Modifiers

| Modifier | Behavior |
|----------|----------|
| `Mandy.Linger` | Caches last package -- new subscriptions receive it immediately |
| `Mandy.Distinct` | Only fires if package changed |
| `Mandy.Debounce(n)` | Coalesces rapid posts into one |
| `Mandy.Throttle(n)` | Max one emission per n seconds |
| `Mandy.Buffer(n)` | Accumulates n packages then fires as array |

---

## Mandate Handle

```lua
local Damage = Mandy.Mandate("Damage", { ... })

-- Server
Damage.Subscribe(function(player, package, resolve, drop) end)
Damage.PostAll({ Amount = 25 })
Damage.PostExclude(player, { Amount = 25 })

-- Client
Damage.Post({ Amount = 25 })
    :Next(function(package, resolve) resolve(package.Applied) end)
    :Conclude(function(original) end)

Damage.Validate({ Amount = 25 })
Damage:Destroy()
```

---

## Bin

```lua
local Combat = Mandy.Bin("Combat", {
    Security = { RateLimit = { MaxPerSecond = 10 } },
    Mandates = {
        Hit        = { Type = Mandy.Unlazy, Parses = { Damage = Mandy.uint8() } },
        Projectile = { Type = Mandy.Lazy,   Parses = { Origin = Mandy.vec3() } },
    },
})

Combat.Define("GetStats", { Type = Mandy.Resolver, Timeout = 5, Parses = { UserId = Mandy.uint32() } })
local Hit = Combat.CreateHandle("Hit")

Combat.Subscribe("Hit", function(player, package, resolve, drop) end)
Combat.Post("Hit", player, { Damage = 25 })
Combat.PostAll("Hit", { Damage = 25 })
Combat.PostExclude("Hit", player, { Damage = 25 })

Combat.Has("Hit")      Combat.List()        Combat.Stats("Hit")
Combat.Pause("Hit")    Combat.Resume("Hit") Combat.Mute("Hit")
Combat.Unmute("Hit")   Combat.Remove("Hit") Combat:Destroy()
```

---

## Subscribing

```lua
local sub = Mandy.Subscribe("PlayerHit", function(player, package, resolve, drop)
    applyDamage(player, package.Damage)
end)

Mandy.Once("PlayerHit", function(player, package, resolve, drop) end)

local data           = Mandy.Await("PlayerHit")
local ok, data, err  = Mandy.AwaitSafe("PlayerHit")
```

### Subscription Object
```lua
sub:Disconnect()  sub.Disconnect()  sub()
sub.Data

sub:Next(function(package, resolve) end)
sub:Toss(function(err, resolve) end)
sub:Conclude(function(original) end)
sub:Retry(n)
sub:Collapse(reason?)
```

---

## Posting

```lua
-- Server
Mandy.Post("PlayerHit", player, { Origin = Vector3.new(), Damage = 25 })
Mandy.PostAll("PlayerHit", { Damage = 25 })
Mandy.PostExclude("PlayerHit", player, { Damage = 25 })

-- Client
Mandy.Post("PlayerHit", { Damage = 25 })

-- Batch
Mandy.Batch(function()
    Mandy.Post("PlayerHit",    player, { Damage = 25 })
    Mandy.Post("StatusEffect", player, { Tag = "Burn" })
end)
```

---

## Async -- Thread

```lua
Mandy.Post("GetStats", { UserId = 123 })
    :Next(function(package, resolve) resolve(package.Level) end)
    :Toss(function(err, resolve) resolve(0) end)
    :Conclude(function(original) print(original.Level) end)
    :Retry(2)
    :Collapse(reason?)
```

```
:Next(fn)          on resolve -- fn(package, resolve) -- must call resolve to continue
:Toss(fn)          on reject  -- fn(err, resolve)     -- call resolve to recover
:Conclude(fn)      always fires -- receives original unmodified package
:Retry(n)          retry up to n times on reject
:Collapse(reason?) cancel thread + auto-resolve
```

### Combinators
```lua
Mandy.All({ t1, t2 })     Mandy.Race({ t1, t2 })   Mandy.Any({ t1, t2 })
Mandy.Batch({ t1 }, n)    Mandy.Waterfall({ fn1 }) Mandy.Settle({ t1, t2 })
```

### Sync Utilities
```lua
Mandy.Resolve(value)  Mandy.Reject(reason)  Mandy.Try(fn)
Mandy.Await(name)     Mandy.AwaitSafe(name) Mandy.Expect(name)
```

---

## Token

```lua
local Token = Mandy.Token()

Token:Next(function(package, resolve)
    warn("cancelled:", package.Reason)
    resolve(package.Reason)
end)

Token:Next()              -- reactive ping
Token:Cancel(reason?)
Token.Cancelled           -- boolean
Token.Reason              -- string?

Mandy.Post("GetStats", { UserId = 123 }, { Token = Token })
Token:Destroy()
```

---

## Collective -- Streaming

```lua
local Stream = Mandy.Collective("MapData", {
    Type    = Mandy.Unlazy,
    Timeout = 10,
    Parses  = { Chunk = Mandy.array(Mandy.uint8()), Index = Mandy.uint16() },
})

Stream:Next({ Chunk = data, Index = 1 })

-- Server
Stream:Next(function(player, package, resolve, drop)
    if done then resolve({ Ok = true }) else drop("bad chunk") end
end)

-- Client
Stream:Next(function(package, resolve, drop) resolve(result) end)
    :Conclude(function(original) print("done") end)

Stream:Pause()   Stream:Resume()
Stream:Drop("reason")
Stream:Destroy()
```

```
states:  Active -> Resolved  /  Dropped
```

---

## Mirror
```lua
local Health = Mandy.Mirror("PlayerHealth", {
    Delta  = true,
    Parses = { Value = Mandy.uint8():bet(0, 100) },
})

Health.Set(player, { Value = 75 })   -- server
Health.Get(player)
Health.Subscribe(function(package, resolve, drop) end)  -- client
Health.Get()
```

## Sync
```lua
local Inventory = Mandy.Sync("Inventory", {
    Parses  = { Items = Mandy.array(Mandy.string()) },
    Resolve = function(player, current, proposed) return proposed end,
})
```

## Bridge
```lua
local ServerMsg = Mandy.Bridge("ServerChat", {
    Parses = { ServerId = Mandy.string(), Message = Mandy.string() },
})
ServerMsg.Post({ ServerId = game.JobId, Message = "hello" })
ServerMsg.Subscribe(function(package, resolve, drop) end)
```

---

## Security

### Signature Derivation
```
EPOCH = os.time({ year=1970, month=1, day=1, hour=0, min=0, sec=0 })
seed  = 42 ^ (os.clock() / (os.time() - EPOCH))
rng   = Random.new(seed)
Raw   = 64x rng:NextInteger(1, 126)
Pass1 = Dott.encode(Raw)
Pass2 = Dott.encode(Pass1)
        clamp Dott codes (01-70) -> a-z (01-26)
Sig   = translated:sub(1, 32)
```

### Setup
```lua
Mandy.Secure({
    RateLimit = { MaxPerSecond = 20, MaxPerMinute = 200 },
    Nonce     = true,
    Sequence  = true,
    OnFlag    = function(player, reason) end,
})

Mandy.Signed(function() end)
```

### Verification Flow
```
Receive
  - Signature   mismatch     -> marks + drop
  - Nonce       seen before  -> marks + drop
  - Sequence    seq <= last  -> marks + drop
  - Timestamp   outside +-5s -> marks + drop
  - RateLimit   exceeded     -> marks + drop
  - Cross       Drop()       -> drop
  - Validation  shape fail   -> marks + drop
  - Subscriber  ok
```

### Marks
```lua
Mandy.Flag(player, "reason")
Mandy.Unflag(player)
local marks = Mandy.Flags(player)
for _, mark in marks do print(mark.Reason, mark.At) end
```

### Advanced Security
```lua
Mandy.Honeypot("FakeMandate")       -- any post auto-marks player
Mandy.Drift({ OnDetect = fn })      -- clock manipulation detection
Mandy.Fingerprint({ OnFlag = fn })  -- behavioral anomaly detection
Mandy.Shadow(player, fn)            -- duplicate packages to observer
Mandy.Lockdown(player)              -- freeze mandate processing
Mandy.Seal("PlayerHit")             -- lock mandate -- no further modifications
Mandy.Surge(player, 5)              -- lift rate limits for n seconds

Mandy.RateLimit("PlayerHit", { MaxPerSecond = 5, MaxPerMinute = 30 })
Mandy.Mute("PlayerHit", player)
Mandy.Disconnect(player)
Mandy.Flush(player)
Mandy.StatsFor(player)
```

---

## Cross -- Middleware
```lua
Mandy.Cross(function(packet, next)
    if packet.Package.Damage > 100 then packet.Drop() return end
    next()
end)
```
`packet` fields: `Name` `Package` `Player` `Direction` `At`

---

## Latency + Network Compensation

### Latency Measurement
```lua
Mandy.Echo(player)           -- round-trip ms
Mandy.Echo()                 -- global average
Mandy.Ping(player)           -- one-way estimated ms
Mandy.Jitter(player)         -- variance ms
Mandy.PingHistory(player, 10) -- last n samples
Mandy.PingSmooth(player)     -- smoothed rolling average
```

### Connection Quality
```lua
local quality = Mandy.Quality(player)
-- { Ping, Jitter, PacketLoss, Stability, Grade }
-- Grade: "Excellent" "Good" "Fair" "Poor"

Mandy.PacketLoss(player)     -- 0-1
Mandy.Stability(player)      -- 0-1

Mandy.OnDegrade(player, function(quality) end)
Mandy.OnRecover(player, function(quality) end)
```

### Clock Synchronization
```lua
Mandy.ServerTime()           -- authoritative server timestamp
Mandy.ClientTime(player)     -- estimated client clock
Mandy.Offset(player)         -- clock offset in seconds
Mandy.Sync.Clock()           -- force re-sync all clients
```

### Lag Compensation
```lua
local state = Mandy.Rewind(player, 0.1)
-- { Position, Orientation, Velocity, At }

local snapshot = Mandy.Rollback(player, timestamp)

local window = Mandy.Window(player)
-- { MaxSeconds, Recommended }

local result = Mandy.Reconcile(player, clientPos, serverPos)
-- { Corrected, Delta, Final }
```

### Query -- Spatial + Mathematical
```lua
-- Part-based
Mandy.Query(player, timestamp, {
    Method = "Part",
    Target = target,
})
-- { Hit, Part, Position }

-- Hitscan ray
Mandy.Query(player, timestamp, {
    Method    = "Ray",
    Origin    = origin,
    Direction = direction,
    MaxDist   = 100,
})
-- { Hit, Position, Distance, Normal }

-- Sphere overlap
Mandy.Query(player, timestamp, {
    Method = "Sphere",
    Center = origin,
    Radius = 5,
})
-- { Hit, Parts }

-- Box overlap
Mandy.Query(player, timestamp, {
    Method = "Box",
    CFrame = cf,
    Size   = Vector3.new(4, 6, 4),
})
-- { Hit, Parts }

-- Pure distance
Mandy.Query(player, timestamp, {
    Method  = "Distance",
    Origin  = origin,
    Target  = targetPos,
    MaxDist = 10,
})
-- { Hit, Distance }
```

### Hit Validation
```lua
local result = Mandy.Validate.Hit(shooter, target, timestamp, origin, direction)
-- { Valid, Distance, Position, Latency }

local result = Mandy.Validate.Reach(player, target, maxDist, timestamp)
-- { Valid, Distance, Delta }
```

### Prediction + Interpolation
```lua
Mandy.Interpolate(player, timestamp)   -- Vector3
Mandy.Extrapolate(player, 0.1)         -- Vector3
Mandy.Snapshots(player, 10)
-- { { Position, Velocity, At }, ... }
```

### Adaptive Systems
```lua
Mandy.Adapt("PlayerPosition", player)  -- auto-switch Unlazy/Lazy by quality
Mandy.TickRate(player)                 -- recommended send rate hz
Mandy.Budget.Player(player)            -- { BytesPerSecond, Priority }
```

### Full Combat Example
```lua
-- Client
local timestamp = Mandy.ServerTime()
Mandy.Post("ShootEvent", {
    Origin    = gun.Position,
    Direction = direction,
    At        = timestamp,
})

-- Server
Mandy.Subscribe("ShootEvent", function(player, package, resolve, drop)
    local window = Mandy.Window(player)

    if Mandy.ServerTime() - package.At > window.MaxSeconds then
        drop("stale timestamp")
        return
    end

    local result = Mandy.Validate.Hit(
        player, target, package.At, package.Origin, package.Direction
    )

    if not result.Valid then
        Mandy.Flag(player, "invalid hit")
        drop("hit validation failed")
        return
    end

    applyDamage(target, 25)
    resolve({ Ok = true, Position = result.Position })
end)
```

---

## Advanced Features

### Routing
```lua
Mandy.Chorus({ player1, player2 }, "RoundStart", { At = tick() })
Mandy.When("PlayerHit", function(player, package)
    return player.Team ~= package.Target.Team
end, function(player, package) end)
Mandy.Region("ZoneEffect", workspace.Zone1, { Radius = 50 })
Mandy.Team("TeamAlert", redTeam, { Message = "Base under attack" })
Mandy.Alias("PlayerHit", "DamagePlayer")
Mandy.Pipeline("CombatFlow", { "PlayerHit", "ApplyDamage", "BroadcastDeath" })
Mandy.Lease(player, "PurchaseItem", { Expires = 10 })
Mandy.Marker.Issue(player, "PurchaseItem", { Expires = 10 })
Mandy.Handoff(stream, newPlayer)
Mandy.Converge({ "p1Ready", "p2Ready" }, function(packages) end)
Mandy.Dispatch("InternalEvent", data)
Mandy.Phase("Combat", { "PlayerHit", "Damage" })
Mandy.Enter("Combat")    Mandy.Exit("Combat")
Mandy.Bind("ZoneChat", workspace.Zone1)
```

### State
```lua
local Timer = Mandy.Anchor("RoundTimer", { Value = Mandy.float32() })
Timer.Set(60)    Timer.Get()

Mandy.Consensus("GlobalEconomy", { Parses = { Gold = Mandy.uint32() }, Quorum = 3 })

local delta = Mandy.Diff("PlayerState", pkgA, pkgB)
-- { Health = { From = 100, To = 75 } }

Mandy.Rollback(player, imprint)
Mandy.Epoch("PlayerHit")
```

### Cross-Server
```lua
Mandy.Cluster("GameServers", { Role = "Game" })
Mandy.Migrate(player, targetJobId)
Mandy.Postit({ Role = "Lobby", Capacity = 50 })
local lobby = Mandy.Find({ Role = "Lobby" })
```

### Performance
```lua
Mandy.Budget({ Critical = 4096, High = 2048, Normal = 1024, Low = 512 })
Mandy.Priority("PlayerHit", "Critical")
Mandy.Pool("PlayerPosition", { Size = 128 })
Mandy.Coalesce("PlayerPosition", true)
Mandy.Compress("MapData", true)
```

### Schema + Validation
```lua
local ok, err = Mandy.Pass({ Origin = Mandy.vec3() }, { Origin = Vector3.new() })
local ok, err = Mandy.Validate("PlayerHit", package)
local t       = Mandy.Infer({ Origin = Vector3.new(), Damage = 25 })

Mandy.Evolve("PlayerHit", {
    [1] = { Damage = Mandy.uint8() },
    [2] = { Damage = Mandy.uint8(), Origin = Mandy.vec3() },
    Migrate = {
        [1] = function(pkg) return { Damage = pkg.Damage, Origin = Vector3.zero } end,
    },
})

Mandy.Define("PlayerHit", { Type = Mandy.Unlazy, Conform = true, Parses = { Damage = Mandy.uint8():bet(0, 100):default(0) } })
local value  = Mandy.Narrow(package.Status, Mandy.string())
local schema = Mandy.Schema("PlayerHit")
Mandy.Explain("PlayerHit")
```

### Observability
```lua
Mandy.Forecast("PlayerHit", function(player, projection)
    print(projection.SecondsUntilLimit, projection.CurrentRate)
end)

Mandy.Probe("PlayerHit", { Interval = 5 })

local heatmap  = Mandy.Heatmap()
local timeline = Mandy.Timeline({ Mandate = "PlayerHit", Player = player })

Mandy.Audit({ Persist = true, DataStore = "MandyAudit" })

Mandy.Alert({
    Metric    = "DropRate",
    Mandate   = "PlayerHit",
    Threshold = 0.1,
    Term      = function(data) end,
})

Mandy.Notice({
    Enabled  = true,
    OnPacket = function(event) print(event.Direction, event.Name, event.Size) end,
    OnDrop   = function(event) end,
    OnFlag   = function(event) end,
    OnError  = function(event) end,
})

local imprint = Mandy.Snapshot()
```

### Recording + Replay
```lua
local rec = Mandy.Record(player)
rec:Stop()
rec:Replay(mockPlayer)
```

---

## Contract + Presence
```lua
Mandy.Contract({
    Request  = "GetStats",
    Response = "StatsResult",
    Within   = 5,
    OnBreach = function(player) end,
})

Mandy.Presence({
    OnJoin      = function(player) end,
    OnLeave     = function(player, reason) end,
    OnReconnect = function(player) end,
})

Mandy.Heartbeat({ Interval = 5, OnSilent = function(player) end })
```

---

## Stash -- Offline Delivery
```lua
Mandy.Stash(player, "RewardGranted", { Amount = 100 }, { Expires = 600 })
```

---

## Plugin System
```lua
return {
    Name    = "MyPlugin",
    Version = "1.0.0",
    OnLoad  = function(mandy)
        mandy.Cross(function(packet, next) next() end)
        mandy.Define("Plugin.Event", { Type = mandy.Lazy, Parses = { At = mandy.float64() } })
    end,
    OnDefine    = function(name, config) end,
    OnPost      = function(name, data) end,
    OnSubscribe = function(name, fn) end,
    OnPacket    = function(packet, next) next() end,
    OnFlag      = function(player, reason) end,
    OnUnload    = function() end,
}
```

```lua
local plugin = Mandy.Plugin(require(ReplicatedStorage.MyPlugin))
plugin:Unload()
```

---

## Cleanup + Janitor
```lua
Mandy.Remove("PlayerHit")   Mandy.Destroy()
p:Destroy()   bin:Destroy()   stream:Destroy()   token:Destroy()
plugin:Unload()
sub:Disconnect() / sub.Disconnect() / sub()

local jan = Mandy.Janitor()
jan:Add(Mandy.Mandate("Hit", { ... }))
jan:Add(Mandy.Subscribe("PlayerHit", fn))
jan:Add(function() end)
jan:Bind(player.Character)
jan:Remove(item)
jan:Cleanup()
```

---

## Dev Utilities
```lua
Mandy.Version
Mandy.Dev(true)
Mandy.Mock()
Mandy.Assert("PlayerHit", data)
Mandy.Validate("PlayerHit", data)
Mandy.Pass(parsesOrName, data)
Mandy.Simulate("PlayerHit", player, data)
Mandy.Replay(imprint)
Mandy.Has("PlayerHit")      Mandy.Get("PlayerHit")
Mandy.List()                Mandy.ListBin("Combat")
Mandy.Active("PlayerHit")
Mandy.Pause("PlayerHit")    Mandy.Resume("PlayerHit")
Mandy.Mute("PlayerHit")     Mandy.Unmute("PlayerHit")
Mandy.Stats("PlayerHit")    Mandy.Stats()
Mandy.Reset("PlayerHit")    Mandy.ResetAll()
Mandy.Extend(fn)
Mandy.Lifecycle({ OnBoot = fn, OnReady = fn, OnDestroy = fn })
```

---

## Exported Types

```lua
export type TypeDef = {
    _value      : string,
    _nullable   : boolean,
    _default    : any?,
    _unions     : { TypeDef }?,
    _intersects : { TypeDef }?,
    _nots       : { TypeDef }?,
    _where      : ((value: any) -> boolean)?,
    _literal    : any?,
    _enum       : { any }?,
    _min        : number?,
    _max        : number?,
    _step       : number?,
    _minLength  : number?,
    _maxLength  : number?,
    _pattern    : string?,
    _minSize    : number?,
    _maxSize    : number?,
    _tag        : string?,
    _describe   : string?,
}

export type MandateConfig = {
    Type      : typeof(Mandy.Unlazy) | typeof(Mandy.Lazy) | typeof(Mandy.Resolver),
    Parses    : { [string]: TypeDef } | typeof(Mandy.None),
    Modifiers : { any }?,
    Security  : { RateLimit: { MaxPerSecond: number?, MaxPerMinute: number? }?, Validate: boolean? }?,
    Conform   : boolean?,
    Timeout   : number?,
    Retries   : number?,
    Delta     : boolean?,
}

export type Thread = {
    Next     : (fn: (package: any, resolve: (value: any) -> ()) -> ()) -> Thread,
    Toss     : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thread,
    Conclude : (fn: (original: any) -> ()) -> Thread,
    Retry    : (n: number) -> Thread,
    Collapse : (reason: string?) -> (),
}

export type Subscription = {
    Disconnect : () -> (),
    Data       : any?,
    Next       : (fn: (package: any, resolve: (value: any) -> ()) -> ()) -> Thread,
    Toss       : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thread,
    Conclude   : (fn: (original: any) -> ()) -> Thread,
    Retry      : (n: number) -> Thread,
    Collapse   : (reason: string?) -> (),
}

export type Token = {
    Cancelled : boolean,
    Reason    : string?,
    Cancel    : (reason: string?) -> (),
    Next      : (fn: ((package: any, resolve: (value: any) -> ()) -> ())?) -> Thread,
    Toss      : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thread,
    Conclude  : (fn: (original: any) -> ()) -> Thread,
    Collapse  : (reason: string?) -> (),
    Destroy   : () -> (),
}

export type Mandate = {
    Subscribe   : (fn: (player: Player?, package: any, resolve: ((any) -> ())?, drop: ((string?) -> ())?) -> ()) -> Subscription,
    Post        : (playerOrData: any, data: any?) -> Thread?,
    PostAll     : (data: any) -> (),
    PostExclude : (player: Player, data: any) -> (),
    Validate    : (data: any) -> (boolean, string?),
    Destroy     : () -> (),
}

export type Bin = {
    Define        : (name: string, config: MandateConfig) -> (),
    CreateHandle  : (name: string) -> Mandate,
    Subscribe     : (name: string, fn: (...any) -> ()) -> Subscription,
    Post          : (name: string, playerOrData: any, data: any?) -> Thread?,
    PostAll       : (name: string, data: any) -> (),
    PostExclude   : (name: string, player: Player, data: any) -> (),
    Has           : (name: string) -> boolean,
    List          : () -> { string },
    Stats         : (name: string) -> { [string]: number },
    Pause         : (name: string) -> (),
    Resume        : (name: string) -> (),
    Mute          : (name: string) -> (),
    Unmute        : (name: string) -> (),
    Remove        : (name: string) -> (),
    Destroy       : () -> (),
}

export type Janitor = {
    Add     : (item: any) -> (),
    Remove  : (item: any) -> (),
    Bind    : (instance: Instance) -> (),
    Cleanup : () -> (),
}

export type PluginHandle = {
    Unload : () -> (),
}

export type Mark = {
    Reason : string,
    At     : number,
}

export type Quality = {
    Ping       : number,
    Jitter     : number,
    PacketLoss : number,
    Stability  : number,
    Grade      : "Excellent" | "Good" | "Fair" | "Poor",
}

export type QueryResult = {
    Hit      : boolean,
    Position : Vector3?,
    Distance : number?,
    Normal   : Vector3?,
    Parts    : { BasePart }?,
    Part     : BasePart?,
}

export type ValidationResult = {
    Valid    : boolean,
    Distance : number?,
    Position : Vector3?,
    Latency  : number?,
    Delta    : number?,
}
```

---

<div align="center">

*Built by [Plinko Labs](https://github.com/NotKisoMomo)*

</div>
