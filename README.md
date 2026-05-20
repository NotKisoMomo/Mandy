# Mandy -- Roblox Networking Library

[![Static Badge](https://img.shields.io/badge/build-v1.0.0--alpha-black?style=for-the-badge)](https://github.com/NotKisoMomo/Mandy)
[![Static Badge](https://img.shields.io/badge/stability-alpha-orange?style=for-the-badge)](https://github.com/NotKisoMomo/Mandy)
[![Static Badge](https://img.shields.io/badge/by-Plinko%20Labs-6C3EF4?style=for-the-badge)](https://github.com/NotKisoMomo)

A next-generation type-safe networking library for Roblox. Buffer-serialized transport via ByteNet, fluent runtime type system, async-first with a clean Thenable surface, built-in security with per-player signature derivation, and a comprehensive API that covers everything from fire-and-forget packets to bidirectional streaming, cross-server messaging, and auto-replicating state.

---

## Table of Contents

- [Installation](#installation)
- [File Structure](#file-structure)
- [Core Concepts](#core-concepts)
- [Type System](#type-system)
- [Defining Controls](#defining-controls)
- [Packet Handle](#packet-handle)
- [Bin](#bin)
- [Subscribing](#subscribing)
- [Posting](#posting)
- [Async -- Thenable](#async----thenable)
- [Token -- Cancellation](#token----cancellation)
- [Collective -- Streaming](#collective----streaming)
- [Mirror](#mirror)
- [Sync](#sync)
- [Bridge](#bridge)
- [Contract](#contract)
- [Presence](#presence)
- [Security](#security)
- [Cross -- Middleware](#cross----middleware)
- [Notice -- Observability](#notice----observability)
- [Plugin System](#plugin-system)
- [Cleanup + Janitor](#cleanup--janitor)
- [Dev Utilities](#dev-utilities)
- [API Reference](#api-reference)
- [File Structure](#file-structure)
- [Exported Types](#exported-types)

---

## Installation

Place the `Mandy` ModuleScript in `ReplicatedStorage` and require it on both server and client:

```lua
local Mandy = require(game:GetService("ReplicatedStorage"):WaitForChild("Mandy"))
```

---

## File Structure

```
Mandy.lua
├── Types.lua
├── Signal.lua
├── Promise.lua
├── Shared.lua
├── Signature.lua
├── Transport.lua
├── Registry.lua
├── Thenable.lua
├── Collective.lua
├── Mirror.lua
├── Sync.lua
├── Bridge.lua
├── Token.lua
├── Janitor.lua
├── Notice.lua
├── Plugin.lua
├── Presence.lua
└── Contract.lua
```

`Mandy.lua` is the top-level ModuleScript. All dependencies are direct children.

---

## Core Concepts

### Controls
Named communication channels with a defined shape, transmission type, and optional modifiers. Both server and client define the same control before use.

### Types
Every field in `Parses` is a `TypeDef` -- a fluent object returned by a Mandy type constructor. TypeDefs carry the base type, constraints, and modifiers. They map to ByteNet buffer primitives on the wire -- validation runs on top after deserialization.

### Transmission Types
Three kinds -- declared via `Type` in the define config:

| Constant | Remote | Behavior |
|----------|--------|----------|
| `Mandy.Unlazy` | `RemoteEvent` | Reliable, ordered, guaranteed delivery |
| `Mandy.Lazy` | `UnreliableRemoteEvent` | Fast, unordered, may drop |
| `Mandy.Resolver` | Internal RF equivalent | Reliable request-response -- `Post` returns a Thenable |

### Subscriber Signatures
Callbacks always receive `(Player, Package, Resolve, Drop)` on the server and `(Package, Resolve, Drop)` on the client. `Resolve` and `Drop` are only non-nil on `Resolver` kind.

---

## Type System

Mandy's type system is a fluent builder. Every constructor returns a `TypeDef` object with chainable modifier methods.

```lua
Mandy.uint8()                    -- base TypeDef
Mandy.uint8():bet(0, 100)        -- constrained
Mandy.string():nullable()        -- nilable
Mandy.vec3():also(Mandy.any())   -- intersection
Mandy.string():or(Mandy.uint8()) -- union
```

### Primitive Types

```lua
Mandy.string()
Mandy.number()
Mandy.float32()
Mandy.float64()
Mandy.uint8()
Mandy.uint16()
Mandy.uint32()
Mandy.int8()
Mandy.int16()
Mandy.int32()
Mandy.bool()
Mandy.buff()
Mandy.any()
Mandy.never()
Mandy.unknown()
Mandy.nil_()
Mandy.None       -- explicit zero-payload -- use as Parses = Mandy.None
```

### Roblox Types

```lua
Mandy.vec2()
Mandy.vec3()
Mandy.vec2int16()
Mandy.cframe()
Mandy.color3()
Mandy.udim()
Mandy.udim2()
Mandy.rect()
Mandy.region3()
Mandy.numberRange()
Mandy.numberSequence()
Mandy.colorSequence()
Mandy.tweenInfo()
Mandy.enumItem()
Mandy.inst()           -- any Instance
Mandy.inst("Part")     -- narrowed to specific classname
```

### Composite Types

```lua
Mandy.struct({
    Name   = Mandy.string(),
    Health = Mandy.uint8(),
})

Mandy.array(Mandy.string())
Mandy.tuple(Mandy.string(), Mandy.uint8())
Mandy.map(Mandy.string(), Mandy.float32())
Mandy.record(Mandy.string())
```

### Advanced Type Constructors

```lua
Mandy.lazy(function()
    return Mandy.struct({ Children = Mandy.array(Mandy.lazy(NodeType)):nullable() })
end)

Mandy.shape({ Name = Mandy.string() })        -- exact -- no extra keys
Mandy.partial(existingStruct)                  -- all fields become nullable
Mandy.pick(existingStruct, { "Name" })         -- subset
Mandy.omit(existingStruct, { "Health" })       -- minus fields
Mandy.merge(structA, structB)                  -- combine two structs
```

### TypeDef Modifiers

```lua
-- Logic
:or(typeDef)          -- union |
:also(typeDef)        -- intersection &
:not(typeDef)         -- negation

-- Nullability
:nullable()
:default(value)

-- Refinement
:where(fn)            -- fn(value) -> boolean
:literal(value)
:enum({ ... })

-- Numeric
:min(n)
:max(n)
:bet(min, max)        -- between
:step(n)

-- String
:minLength(n)
:maxLength(n)
:pattern(str)

-- Collection
:minSize(n)
:maxSize(n)

-- Meta
:tag(str)
:describe(str)
```

### Custom Types

```lua
local UserId = Mandy.defineType({
    Mask     = Mandy.uint32,
    Validate = function(value)
        return type(value) == "number" and value > 0
    end,
})

Mandy.Define("PlayerJoin", {
    Type   = Mandy.Unlazy,
    Parses = {
        Id   = UserId():bet(1, 999999),
        Name = Mandy.string():minLength(3):maxLength(20),
    },
})
```

`Mask` -- the ByteNet primitive the custom type serializes to on the wire. All TypeDef modifiers work on custom types.

---

## Defining Controls

```lua
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

-- Resolver kind -- adds Timeout + Retries
Mandy.Define("GetStats", {
    Type    = Mandy.Resolver,
    Timeout = 5,
    Retries = 2,
    Parses  = {
        UserId = Mandy.uint32(),
    },
})

-- zero-payload
Mandy.Define("Ping", {
    Type   = Mandy.Unlazy,
    Parses = Mandy.None,
})
```

### Modifiers

Declared in the `Modifiers` array -- composable:

| Modifier | Behavior |
|----------|----------|
| `Mandy.Linger` | Caches last emission -- new subscribers receive it immediately |
| `Mandy.Distinct` | Only fires if value changed from last emission |
| `Mandy.Debounce(n)` | Coalesces rapid posts into one after n seconds |
| `Mandy.Throttle(n)` | Max one emission per n seconds |
| `Mandy.Buffer(n)` | Accumulates n emissions then fires as array |

### Modifier Conflict Resolution

| Conflict | Resolution |
|----------|------------|
| `Linger + Buffer` | Caches the last flushed array |
| `Distinct + Buffer` | Distinct check runs on the flushed array |
| `Debounce + Throttle` | Debounce first -- throttle gates output |

### Bulk Define

```lua
Mandy.Load({
    {
        Name   = "Chat",
        Config = { Type = Mandy.Unlazy, Parses = { Text = Mandy.string() } },
    },
    {
        Name   = "Move",
        Config = { Type = Mandy.Lazy,   Parses = { Pos  = Mandy.vec3() } },
    },
})
```

---

## Packet Handle

`Mandy.Packet` is functionally identical to `Mandy.Define` but returns a handle object -- colocates definition and usage.

```lua
local Damage = Mandy.Packet("Damage", {
    Type   = Mandy.Resolver,
    Parses = {
        Amount = Mandy.uint8():bet(0, 100),
        Origin = Mandy.vec3(),
    },
})

-- server
Damage.Subscribe(function(player, package, resolve, drop)
    resolve({ Ok = true, Applied = package.Amount })
end)

-- server broadcast
Damage.PostAll({ Amount = 25, Origin = hit })
Damage.PostExclude(player, { Amount = 25, Origin = hit })

-- client
Damage.Post({ Amount = 25, Origin = hit })
    :Next(function(package, resolve)
        resolve(package.Applied)
    end)
    :Conclude(function(original)
        print("Original:", original.Amount)
    end)

Damage.Validate({ Amount = 25, Origin = Vector3.new() })
Damage:Destroy()
```

---

## Bin

A live named registry of controls. Accepts an initial `Packets` table and a `Security` config that cascades to all controls. Controls can be added or removed at any time after creation.

```lua
local Combat = Mandy.Bin("Combat", {
    Security = { RateLimit = { MaxPerSecond = 10 } },
    Packets  = {
        Hit = {
            Type   = Mandy.Unlazy,
            Parses = { Damage = Mandy.uint8() },
        },
        Projectile = {
            Type   = Mandy.Lazy,
            Parses = { Origin = Mandy.vec3(), Direction = Mandy.vec3() },
        },
    },
})

-- add after creation
Combat.Define("GetStats", {
    Type    = Mandy.Resolver,
    Timeout = 5,
    Parses  = { UserId = Mandy.uint32() },
})

-- use
Combat.Subscribe("Hit", function(player, package, resolve, drop)
    applyDamage(player, package.Damage)
end)

Combat.Post("Hit", player, { Damage = 25 })
Combat.PostAll("Hit", { Damage = 25 })
Combat.PostExclude("Hit", player, { Damage = 25 })

-- inspect + control
Combat.Has("Hit")
Combat.List()
Combat.Stats("Hit")
Combat.Pause("Hit")
Combat.Resume("Hit")
Combat.Mute("Hit")
Combat.Unmute("Hit")
Combat.Remove("Hit")
Combat:Destroy()
```

---

## Subscribing

```lua
-- persistent -- returns Subscription
local sub = Mandy.Subscribe("PlayerHit", function(player, package, resolve, drop)
    applyDamage(player, package.Damage)
end)

-- one-shot -- auto-disconnects after first fire
Mandy.Once("PlayerHit", function(player, package, resolve, drop)
    print("First hit:", package.Damage)
end)

-- yields current thread -- returns data directly
local data = Mandy.Await("PlayerHit")

-- safe variant -- never throws
local ok, data, err = Mandy.AwaitSafe("PlayerHit")
```

### Subscription Object

```lua
sub:Disconnect()   -- method form
sub.Disconnect()   -- dot form
sub()              -- call form -- all three disconnect

sub.Data           -- populated after first emission (useful after Await)

-- Thenable surface -- resolves on next emission
sub:Next(function(package, resolve)
    resolve(package.Damage)
end)
sub:Toss(function(err, resolve) end)
sub:Conclude(function(original) end)
sub:Retry(n)
sub:Collapse(reason?)
```

---

## Posting

```lua
-- server
Mandy.Post("PlayerHit", player, { Origin = Vector3.new(), Damage = 25 })
Mandy.PostAll("PlayerHit", { Origin = Vector3.new(), Damage = 25 })
Mandy.PostExclude("PlayerHit", player, { Origin = Vector3.new(), Damage = 25 })

-- client
Mandy.Post("PlayerHit", { Origin = root.Position, Damage = 25 })

-- batched -- all posts inside flush as one transport payload
Mandy.Batch(function()
    Mandy.Post("PlayerHit", player, { Damage = 25 })
    Mandy.Post("StatusEffect", player, { Tag = "Burn" })
    Mandy.Post("PlayerHit", player, { Damage = 10 })
end)
```

---

## Async -- Thenable

`Mandy.Post` on a `Resolver` control returns a `Thenable`. The chain is middleware-style -- each `:Next` must explicitly call `resolve` to continue.

```lua
Mandy.Post("GetStats", { UserId = 123 })
    :Next(function(package, resolve)
        resolve(package.Level)
    end)
    :Next(function(level, resolve)
        resolve(level + 10)
    end)
    :Toss(function(err, resolve)
        resolve(0)        -- recover and continue
    end)
    :Conclude(function(original)
        -- always fires with original unmodified package
        print("Original level:", original.Level)
    end)
    :Retry(2)
    :Collapse(reason?)    -- cancels thread + auto-resolves
```

### Thenable Methods

| Method | Behavior |
|--------|----------|
| `:Next(fn)` | Fires on resolve -- `fn(package, resolve)` -- must call `resolve` to continue |
| `:Toss(fn)` | Fires on reject -- `fn(err, resolve)` -- can recover by calling `resolve` |
| `:Conclude(fn)` | Always fires after `:Next` resolves -- receives original unmodified package |
| `:Retry(n)` | Retries up to n times on reject before `:Toss` fires |
| `:Collapse(reason?)` | Cancels the thread the Thenable is on -- auto-resolves |

### Thenable Combinators

```lua
Mandy.All({ t1, t2, t3 })
Mandy.Race({ t1, t2, t3 })
Mandy.Any({ t1, t2, t3 })
Mandy.Batch({ t1, t2 }, n)
Mandy.Waterfall({ fn1, fn2 })
Mandy.Settle({ t1, t2 })
```

### Sync Utilities

```lua
Mandy.Resolve(value)
Mandy.Reject(reason)
Mandy.Try(fn)
Mandy.Await(name)
Mandy.AwaitSafe(name)
Mandy.Expect(name)
```

---

## Token -- Cancellation

`Mandy.Token()` is itself Thenable -- its chain fires when cancelled. Both caller and returner can cancel.

```lua
local Token = Mandy.Token()

Token:Next(function(package, resolve)
    warn("Cancelled:", package.Reason)
    resolve(package.Reason)
end)
:Conclude(function(original)
    print("Token done:", original.Reason)
end)

-- reactive ping -- no callback -- fires chain
Token:Next()

-- pass into Post
Mandy.Post("GetStats", { UserId = 123 }, { Token = Token })
    :Next(function(package, resolve)
        resolve(package.Level)
    end)

-- caller cancels
Token:Cancel(reason?)

-- check state
Token.Cancelled  -- boolean
Token.Reason     -- string?

-- returner drops inside Subscribe
Mandy.Subscribe("GetStats", function(player, package, resolve, drop)
    drop("Invalid request")
end)

Token:Destroy()
```

---

## Collective -- Streaming

Typed bidirectional streaming. Buffers flow in both directions -- each chunk validated against `Parses`. The stream can be resolved (completed) or dropped (terminated).

```lua
local Stream = Mandy.Collective("MapData", {
    Type    = Mandy.Unlazy,
    Timeout = 10,           -- auto-drop if no chunk within n seconds
    Parses  = {
        Chunk = Mandy.array(Mandy.uint8()),
        Index = Mandy.uint16(),
    },
})

-- send a chunk (pass a table)
Stream:Next({ Chunk = data, Index = 1 })
Stream:Next({ Chunk = data, Index = 2 })

-- listen (pass a function) -- server
Stream:Next(function(player, package, resolve, drop)
    if package.Index == finalIndex then
        resolve({ Ok = true })
        return
    end
    drop("Malformed chunk")
end)

-- listen -- client
Stream:Next(function(package, resolve, drop)
    resolve(compiledResult)
end)
:Next(function(package, resolve)
    resolve(package)
end)
:Conclude(function(original)
    print("Stream concluded")
end)

-- drop externally
Stream:Drop("Timeout")
Stream:Destroy()
```

Stream states: `Active` `Resolved` `Dropped`. Once `Resolved` or `Dropped` no further chunks fire.

---

## Mirror

Server-owned state that auto-replicates to clients. Mutations on the server propagate automatically -- no manual `PostAll` needed. Supports delta encoding -- only changed fields replicate.

```lua
local Health = Mandy.Mirror("PlayerHealth", {
    Delta  = true,
    Parses = { Value = Mandy.uint8():bet(0, 100) },
})

-- server
Health.Set(player, { Value = 75 })
Health.Get(player)

-- client -- read only + reactive
Health.Subscribe(function(package, resolve, drop)
    updateHealthBar(package.Value)
end)

Health.Get()    -- last known value
```

---

## Sync

Bidirectional shared state. Server is authoritative -- client proposes changes, server accepts or rejects via `Resolve`.

```lua
local Inventory = Mandy.Sync("Inventory", {
    Parses  = { Items = Mandy.array(Mandy.string()) },
    Resolve = function(player, current, proposed)
        return proposed   -- return current to reject
    end,
})
```

---

## Bridge

Cross-server typed messaging via `MessagingService` -- fully validated like a normal Mandy control.

```lua
local ServerMsg = Mandy.Bridge("ServerChat", {
    Parses = {
        ServerId = Mandy.string(),
        Message  = Mandy.string(),
    },
})

ServerMsg.Post({ ServerId = game.JobId, Message = "Hello from this server" })

ServerMsg.Subscribe(function(package, resolve, drop)
    print(package.ServerId, package.Message)
end)
```

---

## Contract

Enforces a required relationship between two controls -- a request must always be followed by a response within a time window.

```lua
Mandy.Contract({
    Request  = "GetStats",
    Response = "StatsResult",
    Within   = 5,
    OnBreach = function(player)
        Mandy.Flag(player, "Contract breached")
    end,
})
```

---

## Presence

Built-in player connection state tracking.

```lua
Mandy.Presence({
    OnJoin      = function(player)
        print(player.Name .. " joined")
    end,
    OnLeave     = function(player, reason)
        print(player.Name .. " left -- " .. reason)
    end,
    OnReconnect = function(player)
        print(player.Name .. " reconnected")
    end,
})
```

---

## Security

Mandy's security layer is opt-in -- call `Mandy.Secure()` to activate. Once active, a per-server signature is derived, sent to each client via a bootstrap remote, and verified on every subsequent packet. Packets sent before bootstrap completes are queued and flushed in order after `Mandy.Signed` fires.

### Signature Derivation

```lua
local EPOCH  = os.time({ year = 1970, month = 1, day = 1, hour = 0, min = 0, sec = 0 })
local delta  = os.time() - EPOCH
local seed   = 42 ^ (os.clock() / delta)
local rng    = Random.new(seed)

local Raw = ""
for i = 1, 64 do
    Raw = Raw .. rng:NextInteger(1, 126)
end

local Pass1     = Dott.encode(Raw)
local Pass2     = Dott.encode(Pass1)
-- clamp each char's Dott code (01-70) to a-z range (01-26)
local Signature = translated:sub(1, 32)
```

### Setup

```lua
-- server only
Mandy.Secure({
    RateLimit = { MaxPerSecond = 20, MaxPerMinute = 200 },
    Nonce     = true,
    Sequence  = true,
    OnFlag    = function(player, reason)
        warn(player.Name .. " -- " .. reason)
    end,
})

Mandy.Signed(function()
    print("Bootstrap complete -- safe to post")
end)
```

### Packet Verification Flow

```
Receive
  → Signature check     -- mismatch → flag + drop
  → Nonce check         -- seen before → flag + drop
  → Sequence check      -- seq <= last seen → flag + drop
  → Timestamp window    -- outside ±5s → flag + drop
  → Rate limit          -- exceeded → flag + drop
  → Cross chain         -- packet.Drop() called → drop
  → Parses validation   -- shape mismatch → flag + drop
  → Subscriber fires
```

### Rate Limiting

```lua
-- global -- set in Mandy.Secure

-- per-control override
Mandy.RateLimit("PlayerHit", {
    MaxPerSecond = 5,
    MaxPerMinute = 30,
})

-- per-define override
Mandy.Define("PlayerHit", {
    Type     = Mandy.Unlazy,
    Security = { RateLimit = { MaxPerSecond = 5 }, Validate = true },
    Parses   = { Damage = Mandy.uint8():bet(0, 100) },
})
```

### Flags

```lua
Mandy.Flag(player, "Attempted exploit")

local flags = Mandy.Flags(player)
for _, flag in flags do
    print(flag.Reason, flag.At)
end

Mandy.Unflag(player)
```

### Per-player Controls

```lua
Mandy.Mute("PlayerHit", player)
Mandy.Disconnect(player)
Mandy.Flush(player)
Mandy.StatsFor(player)
```

---

## Cross -- Middleware

Global middleware layer. Runs before subscribers fire on every packet. Multiple `Cross` calls stack in registration order. Each handler must call `next()` or `packet.Drop()`.

```lua
Mandy.Cross(function(packet, next)
    if packet.Name == "PlayerHit" and packet.Package.Damage > 100 then
        packet.Drop()
        return
    end
    next()
end)

Mandy.Cross(function(packet, next)
    if not Mandy.Has(packet.Name) then
        packet.Drop()
        return
    end
    next()
end)
```

`packet` fields: `Name` `Package` `Player` `Direction` `At`

---

## Notice -- Observability

Live traffic monitor. Zero cost when disabled. Works alongside `Mandy.Snapshot()` for point-in-time inspection.

```lua
Mandy.Notice({
    Enabled  = true,
    OnPacket = function(event)
        print(event.Direction, event.Name, event.Size, event.At)
    end,
    OnDrop   = function(event)
        warn("Dropped:", event.Name, event.Player and event.Player.Name)
    end,
    OnFlag   = function(event)
        warn("Flagged:", event.Player.Name, "--", event.Reason, "at", event.At)
    end,
    OnError  = function(event)
        warn("Error:", event.Reason)
    end,
})

local snapshot = Mandy.Snapshot()
```

`event` fields: `Direction` `Name` `Package` `Player` `Size` `At`

---

## Plugin System

Plugins hook into the full Mandy lifecycle. Authored as a plain table, consumed via `Mandy.Plugin()`. The `kimo` reference passed into `OnLoad` is a full Mandy handle -- plugins can define controls, subscribe, post, and add middleware.

### Authoring

```lua
-- AnticheatPlugin.lua
return {
    Name    = "AnticheatPlugin",
    Version = "1.0.0",

    OnLoad = function(mandy)
        mandy.Cross(function(packet, next)
            if packet.Package.Velocity and packet.Package.Velocity.Magnitude > 200 then
                packet.Drop()
                mandy.Flag(packet.Player, "Speed exploit")
                return
            end
            next()
        end)

        mandy.Define("Plugin.Heartbeat", {
            Type   = mandy.Lazy,
            Parses = { At = mandy.float64() },
        })
    end,

    OnDefine    = function(name, config) end,
    OnPost      = function(name, data) end,
    OnSubscribe = function(name, fn) end,
    OnPacket    = function(packet, next) next() end,
    OnFlag      = function(player, reason) end,
    OnUnload    = function() end,
}
```

### Consuming

```lua
local AnticheatPlugin = require(ReplicatedStorage.AnticheatPlugin)

local plugin = Mandy.Plugin(AnticheatPlugin)

-- unload later -- cleans up all controls, subscriptions, and hooks the plugin registered
plugin:Unload()
```

Multiple plugins stack in registration order:

```lua
Mandy.Plugin(AnticheatPlugin)
Mandy.Plugin(LoggerPlugin)
Mandy.Plugin(DevToolsPlugin)
```

---

## Cleanup + Janitor

### Explicit Cleanup

```lua
Mandy.Remove("PlayerHit")      -- unregister a single control
Mandy.ResetAll()               -- reset all traffic stats
Mandy.Destroy()                -- tear down everything

local p = Mandy.Packet(...)
p:Destroy()

local bin = Mandy.Bin("Combat", { ... })
bin:Destroy()

local stream = Mandy.Collective(...)
stream:Destroy()

local token = Mandy.Token()
token:Destroy()

local plugin = Mandy.Plugin(...)
plugin:Unload()

local sub = Mandy.Subscribe(...)
sub:Disconnect()
sub()
```

### Janitor

```lua
local jan = Mandy.Janitor()

jan:Add(Mandy.Packet("Hit", { ... }))
jan:Add(Mandy.Subscribe("PlayerHit", fn))
jan:Add(Mandy.Token())
jan:Add(Mandy.Bin("Combat", { ... }))
jan:Add(function()
    print("custom cleanup")
end)

-- bind to Instance -- auto-cleans when destroyed
jan:Bind(player.Character)

-- remove a specific item without destroying it
jan:Remove(item)

-- clean everything
jan:Cleanup()
```

`jan:Add()` accepts anything with `Destroy`, `Unload`, `Disconnect`, or `Remove` -- or a raw cleanup function.

---

## Dev Utilities

```lua
Mandy.Version                           -- current version string

Mandy.Dev(true)                         -- verbose logging -- every packet printed

Mandy.Assert("PlayerHit", data)         -- throws if data fails Parses validation
Mandy.Validate("PlayerHit", data)       -- returns (boolean, string?)
Mandy.Simulate("PlayerHit", player, data) -- fire packet locally without network
Mandy.Replay(snapshot)                  -- replay a Mandy.Snapshot() dump

Mandy.Has("PlayerHit")                  -- boolean
Mandy.Get("PlayerHit")                  -- control config
Mandy.List()                            -- all registered control names
Mandy.ListBin("Combat")                 -- all controls under a Bin
Mandy.Active("PlayerHit")              -- is control currently active

Mandy.Pause("PlayerHit")               -- suspend -- packets queue
Mandy.Resume("PlayerHit")              -- flush queue + resume
Mandy.Mute("PlayerHit")               -- drop all incoming silently
Mandy.Unmute("PlayerHit")

Mandy.Stats("PlayerHit")               -- { Sent, Received, Dropped, Flagged, BytesIn, BytesOut }
Mandy.Stats()                          -- global totals
Mandy.Reset("PlayerHit")              -- reset stats for one control
Mandy.ResetAll()

Mandy.Extend(fn)                        -- safely wrap core methods globally
```

---

## API Reference

### Top-level

| Function | Returns | Description |
|----------|---------|-------------|
| `Mandy.Define(name, config)` | `void` | Register a control globally |
| `Mandy.Load(controls)` | `void` | Bulk define from table |
| `Mandy.Packet(name, config)` | `PacketHandle` | Define + return handle |
| `Mandy.Bin(name, config?)` | `Bin` | Named control registry |
| `Mandy.Collective(name, config)` | `Collective` | Streaming channel |
| `Mandy.Mirror(name, config)` | `Mirror` | Auto-replicating server state |
| `Mandy.Sync(name, config)` | `Sync` | Bidirectional shared state |
| `Mandy.Bridge(name, config)` | `Bridge` | Cross-server messaging |
| `Mandy.Contract(config)` | `void` | Enforced request/response |
| `Mandy.Presence(config)` | `void` | Player connection tracking |
| `Mandy.Subscribe(name, fn)` | `Subscription` | Listen for packets |
| `Mandy.Once(name, fn)` | `Subscription` | One-shot listener |
| `Mandy.Await(name)` | `data` | Yield until next packet |
| `Mandy.AwaitSafe(name)` | `ok, data, err` | Safe yield |
| `Mandy.Expect(name)` | `data` | Yield -- errors on reject |
| `Mandy.Post(name, playerOrData, data?)` | `Thenable?` | Send packet |
| `Mandy.PostAll(name, data)` | `void` | Broadcast -- server only |
| `Mandy.PostExclude(name, player, data)` | `void` | Broadcast except one player |
| `Mandy.Batch(fn)` | `void` | Flush multiple posts in one frame |
| `Mandy.Validate(name, data)` | `boolean, string?` | Validate payload |
| `Mandy.Token()` | `Token` | Cancellation token |
| `Mandy.Janitor()` | `Janitor` | Batch cleanup |
| `Mandy.Secure(config)` | `void` | Activate security layer |
| `Mandy.Signed(fn)` | `void` | Fires when bootstrap completes |
| `Mandy.Cross(fn)` | `void` | Add global middleware |
| `Mandy.RateLimit(name, config)` | `void` | Per-control rate limit |
| `Mandy.Flag(player, reason)` | `void` | Manual flag |
| `Mandy.Unflag(player)` | `void` | Clear flags |
| `Mandy.Flags(player)` | `FlagEntry[]` | Flag history |
| `Mandy.Notice(config)` | `void` | Observability hooks |
| `Mandy.Snapshot()` | `Snapshot` | Point-in-time registry dump |
| `Mandy.Plugin(plugin)` | `PluginHandle` | Register plugin |
| `Mandy.Extend(fn)` | `void` | Wrap core methods globally |
| `Mandy.Remove(name)` | `void` | Unregister control |
| `Mandy.Destroy()` | `void` | Tear down everything |

### Transmission Types

| Constant | Remote | Behavior |
|----------|--------|----------|
| `Mandy.Unlazy` | `RemoteEvent` | Reliable -- ordered -- guaranteed |
| `Mandy.Lazy` | `UnreliableRemoteEvent` | Fast -- unordered -- may drop |
| `Mandy.Resolver` | Internal equivalent | Reliable -- request-response -- returns Thenable |

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

export type ControlConfig = {
    Type      : typeof(Mandy.Unlazy) | typeof(Mandy.Lazy) | typeof(Mandy.Resolver),
    Parses    : { [string]: TypeDef } | typeof(Mandy.None),
    Modifiers : { any }?,
    Security  : { RateLimit: { MaxPerSecond: number?, MaxPerMinute: number? }?, Validate: boolean? }?,
    Timeout   : number?,
    Retries   : number?,
    Delta     : boolean?,
}

export type Thenable = {
    Next     : (fn: (package: any, resolve: (value: any) -> ()) -> ()) -> Thenable,
    Toss     : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thenable,
    Conclude : (fn: (original: any) -> ()) -> Thenable,
    Retry    : (n: number) -> Thenable,
    Collapse : (reason: string?) -> (),
}

export type Subscription = {
    Disconnect : () -> (),
    Data       : any?,
    Next       : (fn: (package: any, resolve: (value: any) -> ()) -> ()) -> Thenable,
    Toss       : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thenable,
    Conclude   : (fn: (original: any) -> ()) -> Thenable,
    Retry      : (n: number) -> Thenable,
    Collapse   : (reason: string?) -> (),
}

export type Token = {
    Cancelled : boolean,
    Reason    : string?,
    Cancel    : (reason: string?) -> (),
    Next      : (fn: ((package: any, resolve: (value: any) -> ()) -> ())?) -> Thenable,
    Toss      : (fn: (err: any, resolve: (value: any) -> ()) -> ()) -> Thenable,
    Conclude  : (fn: (original: any) -> ()) -> Thenable,
    Collapse  : (reason: string?) -> (),
    Destroy   : () -> (),
}

export type PacketHandle = {
    Subscribe   : (fn: (player: Player?, package: any, resolve: ((any) -> ())?, drop: ((string?) -> ())?) -> ()) -> Subscription,
    Post        : (playerOrData: any, data: any?) -> Thenable?,
    PostAll     : (data: any) -> (),
    PostExclude : (player: Player, data: any) -> (),
    Validate    : (data: any) -> (boolean, string?),
    Destroy     : () -> (),
}

export type Bin = {
    Define      : (name: string, config: ControlConfig) -> (),
    Subscribe   : (name: string, fn: (...any) -> ()) -> Subscription,
    Post        : (name: string, playerOrData: any, data: any?) -> Thenable?,
    PostAll     : (name: string, data: any) -> (),
    PostExclude : (name: string, player: Player, data: any) -> (),
    Has         : (name: string) -> boolean,
    List        : () -> { string },
    Stats       : (name: string) -> { [string]: number },
    Pause       : (name: string) -> (),
    Resume      : (name: string) -> (),
    Mute        : (name: string) -> (),
    Unmute      : (name: string) -> (),
    Remove      : (name: string) -> (),
    Destroy     : () -> (),
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

export type FlagEntry = {
    Reason : string,
    At     : number,
}
```

---

*Built by [Plinko Labs](https://github.com/NotKisoMomo)*
