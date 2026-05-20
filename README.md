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
[![bytenet](https://img.shields.io/badge/transport-bytenet-6C3EF4?style=for-the-badge&labelColor=0d0d0d)](https://github.com/ffrostfall/ByteNet)
[![plinko](https://img.shields.io/badge/by-plinko%20labs-6C3EF4?style=for-the-badge&labelColor=0d0d0d)](https://github.com/NotKisoMomo)

*Buffer-serialized. Type-safe. Async-first. Built for Roblox.*

</div>

---

```
  reliable ──────────────────────────────► [Unlazy]  RemoteEvent
  unreliable ────────────────────────────► [Lazy]    UnreliableRemoteEvent  
  request-response ──────────────────────► [Resolver] → Thenable chain
                                                            │
                                                    :Next(pkg, resolve)
                                                    :Toss(err, resolve)
                                                    :Conclude(original)
                                                    :Retry(n)
                                                    :Collapse(reason?)
```

---

## what is mandy

Mandy is a next-generation networking library for Roblox. ByteNet powers the transport layer -- everything travels as compact binary buffers. On top of that sits a fluent runtime type system, a middleware-style async Thenable chain, per-server signature security, bidirectional streaming via Collectives, auto-replicating Mirror state, cross-server Bridge messaging, and a plugin system with full lifecycle hooks.

Both sides define the same control. The rest is handled.

---

## installation

```lua
local Mandy = require(game:GetService("ReplicatedStorage"):WaitForChild("Mandy"))
```

---

## core concepts

```
control      named channel -- defined on both server and client before use
TypeDef      fluent type object -- maps to a ByteNet primitive on the wire
             validation runs on top after deserialization
kind         Unlazy / Lazy / Resolver -- determines the underlying remote
subscriber   server: (Player, Package, Resolve, Drop)
             client: (Package, Resolve, Drop)
             Resolve + Drop are only non-nil on Resolver kind
```

---

## type system

Every type constructor returns a `TypeDef` -- a fluent object with chainable modifiers.

```lua
Mandy.uint8()                         -- base TypeDef
Mandy.uint8():bet(0, 100)             -- constrained
Mandy.string():nullable()             -- nilable
Mandy.vec3():also(Mandy.any())        -- intersection  &
Mandy.string():or(Mandy.uint8())      -- union         |
Mandy.string():not(Mandy.uint8())     -- negation      ~
```

### primitives

```lua
Mandy.string()      Mandy.number()      Mandy.bool()
Mandy.float32()     Mandy.float64()     Mandy.buff()
Mandy.uint8()       Mandy.uint16()      Mandy.uint32()
Mandy.int8()        Mandy.int16()       Mandy.int32()
Mandy.any()         Mandy.never()       Mandy.unknown()
Mandy.nil_()        Mandy.None          -- zero-payload -- use as Parses = Mandy.None
```

### roblox types

```lua
Mandy.vec2()          Mandy.vec3()          Mandy.vec2int16()
Mandy.cframe()        Mandy.color3()        Mandy.udim()
Mandy.udim2()         Mandy.rect()          Mandy.region3()
Mandy.numberRange()   Mandy.numberSequence() Mandy.colorSequence()
Mandy.tweenInfo()     Mandy.enumItem()
Mandy.inst()          -- any Instance
Mandy.inst("Part")    -- narrowed to classname
```

### composites

```lua
Mandy.struct({ Name = Mandy.string(), Health = Mandy.uint8() })
Mandy.array(Mandy.string())
Mandy.tuple(Mandy.string(), Mandy.uint8())
Mandy.map(Mandy.string(), Mandy.float32())
Mandy.record(Mandy.string())
Mandy.lazy(function() return Mandy.struct({ ... }) end)   -- recursive types
Mandy.shape({ ... })         -- exact -- no extra keys allowed
Mandy.partial(existingStruct)
Mandy.pick(existingStruct, { "Name" })
Mandy.omit(existingStruct, { "Health" })
Mandy.merge(structA, structB)
```

### modifiers

| modifier | description |
|----------|-------------|
| `:or(typeDef)` | union \| |
| `:also(typeDef)` | intersection & |
| `:not(typeDef)` | negation |
| `:nullable()` | value may be nil |
| `:default(value)` | substitute if nil |
| `:where(fn)` | custom predicate |
| `:literal(value)` | must equal exactly |
| `:enum({ ... })` | must be one of |
| `:min(n)` / `:max(n)` | numeric bounds |
| `:bet(min, max)` | between -- shorthand |
| `:step(n)` | must be multiple of n |
| `:minLength(n)` / `:maxLength(n)` | string length |
| `:pattern(str)` | Lua pattern match |
| `:minSize(n)` / `:maxSize(n)` | collection size |
| `:tag(str)` | nominal brand |
| `:describe(str)` | human-readable error label |

### custom types

```lua
local UserId = Mandy.defineType({
    Mask     = Mandy.uint32,    -- ByteNet primitive on the wire
    Validate = function(value)
        return type(value) == "number" and value > 0
    end,
})

-- first-class TypeDef -- all modifiers work
Mandy.Define("PlayerJoin", {
    Type   = Mandy.Unlazy,
    Parses = {
        Id   = UserId():bet(1, 999999),
        Name = Mandy.string():minLength(3):maxLength(20),
    },
})
```

---

## defining controls

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

-- Resolver -- adds Timeout + Retries
Mandy.Define("GetStats", {
    Type    = Mandy.Resolver,
    Timeout = 5,
    Retries = 2,
    Parses  = { UserId = Mandy.uint32() },
})

-- zero-payload
Mandy.Define("Ping", {
    Type   = Mandy.Unlazy,
    Parses = Mandy.None,
})

-- bulk define
Mandy.Load({
    { Name = "Chat", Config = { Type = Mandy.Unlazy, Parses = { Text = Mandy.string() } } },
    { Name = "Move", Config = { Type = Mandy.Lazy,   Parses = { Pos  = Mandy.vec3()   } } },
})
```

### packet modifiers

| modifier | behavior |
|----------|----------|
| `Mandy.Linger` | caches last emission -- new subscribers receive it immediately |
| `Mandy.Distinct` | only fires if value changed from last emission |
| `Mandy.Debounce(n)` | coalesces rapid posts into one after n seconds |
| `Mandy.Throttle(n)` | max one emission per n seconds |
| `Mandy.Buffer(n)` | accumulates n emissions then fires as array |

---

## packet handle

`Mandy.Packet` is identical to `Mandy.Define` but returns a handle -- colocates definition and usage.

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

Damage.PostAll({ Amount = 25, Origin = hit })
Damage.PostExclude(player, { Amount = 25, Origin = hit })

-- client
Damage.Post({ Amount = 25, Origin = hit })
    :Next(function(package, resolve)
        resolve(package.Applied)
    end)
    :Conclude(function(original)
        print("original:", original.Amount)
    end)

Damage.Validate({ Amount = 25, Origin = Vector3.new() })
Damage:Destroy()
```

---

## bin

A live named registry. Accepts an initial `Packets` table and a `Security` config that cascades to all controls. Controls can be added or removed at any time after creation.

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

Combat.Define("GetStats", {
    Type    = Mandy.Resolver,
    Timeout = 5,
    Parses  = { UserId = Mandy.uint32() },
})

Combat.Subscribe("Hit", function(player, package, resolve, drop)
    applyDamage(player, package.Damage)
end)

Combat.Post("Hit", player, { Damage = 25 })
Combat.PostAll("Hit", { Damage = 25 })
Combat.PostExclude("Hit", player, { Damage = 25 })

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

## subscribing

```lua
-- persistent
local sub = Mandy.Subscribe("PlayerHit", function(player, package, resolve, drop)
    applyDamage(player, package.Damage)
end)

-- one-shot -- auto-disconnects after first fire
Mandy.Once("PlayerHit", function(player, package, resolve, drop)
    print("first hit:", package.Damage)
end)

-- yields thread
local data = Mandy.Await("PlayerHit")

-- safe -- never throws
local ok, data, err = Mandy.AwaitSafe("PlayerHit")
```

### subscription object

```lua
sub:Disconnect()    -- method
sub.Disconnect()    -- dot
sub()               -- call -- all three disconnect

sub.Data            -- populated after first emission

-- Thenable surface -- resolves on next emission
sub:Next(function(package, resolve) resolve(package.Damage) end)
sub:Toss(function(err, resolve) end)
sub:Conclude(function(original) end)
sub:Retry(n)
sub:Collapse(reason?)
```

---

## posting

```lua
-- server
Mandy.Post("PlayerHit", player, { Origin = Vector3.new(), Damage = 25 })
Mandy.PostAll("PlayerHit", { Origin = Vector3.new(), Damage = 25 })
Mandy.PostExclude("PlayerHit", player, { Origin = Vector3.new(), Damage = 25 })

-- client
Mandy.Post("PlayerHit", { Origin = root.Position, Damage = 25 })

-- batch -- flushes as one transport payload
Mandy.Batch(function()
    Mandy.Post("PlayerHit",    player, { Damage = 25 })
    Mandy.Post("StatusEffect", player, { Tag = "Burn" })
    Mandy.Post("PlayerHit",    player, { Damage = 10 })
end)
```

---

## async -- thenable

`Mandy.Post` on a `Resolver` control returns a `Thenable`. The chain is middleware-style -- each `:Next` must call `resolve` to continue downstream.

```lua
Mandy.Post("GetStats", { UserId = 123 })
    :Next(function(package, resolve)
        resolve(package.Level)
    end)
    :Next(function(level, resolve)
        resolve(level + 10)
    end)
    :Toss(function(err, resolve)
        resolve(0)              -- recover + continue
    end)
    :Conclude(function(original)
        print("original level:", original.Level)
    end)
    :Retry(2)
    :Collapse(reason?)
```

```
:Next(fn)          on resolve -- fn(package, resolve) -- must call resolve to continue
:Toss(fn)          on reject  -- fn(err, resolve)     -- call resolve to recover
:Conclude(fn)      always fires after Next -- receives original unmodified package
:Retry(n)          retry up to n times on reject before Toss fires
:Collapse(reason?) cancel thread + auto-resolve
```

### combinators

```lua
Mandy.All({ t1, t2, t3 })       -- resolves when all resolve
Mandy.Race({ t1, t2, t3 })      -- first to resolve or reject wins
Mandy.Any({ t1, t2, t3 })       -- first to resolve wins
Mandy.Batch({ t1, t2 }, n)      -- resolves n at a time
Mandy.Waterfall({ fn1, fn2 })   -- sequential -- output feeds next
Mandy.Settle({ t1, t2 })        -- waits for all -- never rejects
```

### sync utilities

```lua
Mandy.Resolve(value)
Mandy.Reject(reason)
Mandy.Try(fn)
Mandy.Await(name)
Mandy.AwaitSafe(name)
Mandy.Expect(name)
```

---

## token -- cancellation

`Mandy.Token()` is itself Thenable -- its chain fires on cancel. Both caller and returner participate.

```lua
local Token = Mandy.Token()

Token:Next(function(package, resolve)
    warn("cancelled:", package.Reason)
    resolve(package.Reason)
end)
:Conclude(function(original)
    print("token done:", original.Reason)
end)

Token:Next()    -- reactive ping -- no callback -- fires chain

Mandy.Post("GetStats", { UserId = 123 }, { Token = Token })
    :Next(function(package, resolve)
        resolve(package.Level)
    end)

Token:Cancel(reason?)

Token.Cancelled    -- boolean
Token.Reason       -- string?

-- returner side
Mandy.Subscribe("GetStats", function(player, package, resolve, drop)
    drop("invalid request")
end)

Token:Destroy()
```

---

## collective -- streaming

Typed bidirectional streaming. Pass a table to send -- pass a function to listen. Each chunk validated against `Parses`. Stream can be resolved (completed) or dropped (terminated).

```lua
local Stream = Mandy.Collective("MapData", {
    Type    = Mandy.Unlazy,
    Timeout = 10,
    Parses  = {
        Chunk = Mandy.array(Mandy.uint8()),
        Index = Mandy.uint16(),
    },
})

-- send
Stream:Next({ Chunk = data, Index = 1 })
Stream:Next({ Chunk = data, Index = 2 })

-- listen -- server
Stream:Next(function(player, package, resolve, drop)
    if package.Index == finalIndex then
        resolve({ Ok = true })
        return
    end
    drop("malformed chunk")
end)

-- listen -- client
Stream:Next(function(package, resolve, drop)
    resolve(compiledResult)
end)
:Conclude(function(original)
    print("stream concluded")
end)

Stream:Drop("reason")
Stream:Destroy()
```

```
states:  Active  →  Resolved  /  Dropped
         once Resolved or Dropped -- no further chunks fire
```

---

## mirror

Server-owned state that auto-replicates to all clients. Mutations propagate automatically -- no manual `PostAll`. Supports `Delta = true` -- only changed fields replicate.

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

## sync

Bidirectional shared state. Server is authoritative -- client proposes, server accepts or rejects via `Resolve`.

```lua
local Inventory = Mandy.Sync("Inventory", {
    Parses  = { Items = Mandy.array(Mandy.string()) },
    Resolve = function(player, current, proposed)
        return proposed    -- return current to reject
    end,
})
```

---

## bridge

Cross-server messaging via `MessagingService` -- fully typed and validated like any other Mandy control.

```lua
local ServerMsg = Mandy.Bridge("ServerChat", {
    Parses = {
        ServerId = Mandy.string(),
        Message  = Mandy.string(),
    },
})

ServerMsg.Post({ ServerId = game.JobId, Message = "hello from this server" })

ServerMsg.Subscribe(function(package, resolve, drop)
    print(package.ServerId, package.Message)
end)
```

---

## contract

Enforces that a request is always followed by a response within a time window.

```lua
Mandy.Contract({
    Request  = "GetStats",
    Response = "StatsResult",
    Within   = 5,
    OnBreach = function(player)
        Mandy.Flag(player, "contract breached")
    end,
})
```

---

## presence

Built-in player connection tracking.

```lua
Mandy.Presence({
    OnJoin      = function(player) end,
    OnLeave     = function(player, reason) end,
    OnReconnect = function(player) end,
})
```

---

## security

Opt-in. Call `Mandy.Secure()` to activate. A per-server signature is derived on boot, sent to each client via a bootstrap remote, and verified on every subsequent packet. Packets sent before bootstrap completes are queued and flushed in order after `Mandy.Signed` fires.

### signature derivation

```
EPOCH  = os.time({ year=1970, month=1, day=1, hour=0, min=0, sec=0 })
delta  = os.time() - EPOCH
seed   = 42 ^ (os.clock() / delta)
rng    = Random.new(seed)

Raw    = 64x rng:NextInteger(1, 126)
Pass1  = Dott.encode(Raw)
Pass2  = Dott.encode(Pass1)
        clamp each char Dott code (01-70) → a-z range (01-26)
Sig    = translated:sub(1, 32)
```

### setup

```lua
Mandy.Secure({
    RateLimit = { MaxPerSecond = 20, MaxPerMinute = 200 },
    Nonce     = true,
    Sequence  = true,
    OnFlag    = function(player, reason)
        warn(player.Name .. " -- " .. reason)
    end,
})

Mandy.Signed(function()
    print("bootstrap complete -- safe to post")
end)
```

### verification flow

```
Receive
  → Signature     mismatch       → flag + drop
  → Nonce         seen before    → flag + drop
  → Sequence      seq ≤ last     → flag + drop
  → Timestamp     outside ±5s   → flag + drop
  → RateLimit     exceeded       → flag + drop
  → Cross chain   Drop() called  → drop
  → Validation    shape fail     → flag + drop
  → Subscriber    ✓
```

### rate limiting

```lua
Mandy.RateLimit("PlayerHit", { MaxPerSecond = 5, MaxPerMinute = 30 })
```

### flags

```lua
Mandy.Flag(player, "attempted exploit")

local flags = Mandy.Flags(player)
for _, flag in flags do
    print(flag.Reason, flag.At)
end

Mandy.Unflag(player)
```

### per-player

```lua
Mandy.Mute("PlayerHit", player)
Mandy.Disconnect(player)
Mandy.Flush(player)
Mandy.StatsFor(player)
```

---

## cross -- middleware

Global middleware. Stacks in registration order. Each handler must call `next()` or `packet.Drop()`.

```lua
Mandy.Cross(function(packet, next)
    if packet.Name == "PlayerHit" and packet.Package.Damage > 100 then
        packet.Drop()
        return
    end
    next()
end)
```

`packet` fields: `Name` `Package` `Player` `Direction` `At`

---

## notice -- observability

```lua
Mandy.Notice({
    Enabled  = true,
    OnPacket = function(event)
        print(event.Direction, event.Name, event.Size, event.At)
    end,
    OnDrop   = function(event) warn("dropped:", event.Name) end,
    OnFlag   = function(event) warn("flagged:", event.Player.Name, event.Reason) end,
    OnError  = function(event) warn("error:",   event.Reason) end,
})

local snapshot = Mandy.Snapshot()
```

---

## plugin system

Authored as a plain table -- consumed via `Mandy.Plugin()`. The `mandy` reference in `OnLoad` is a full Mandy handle -- plugins can define controls, subscribe, post, and add middleware.

```lua
-- MyPlugin.lua
return {
    Name    = "MyPlugin",
    Version = "1.0.0",

    OnLoad = function(mandy)
        mandy.Cross(function(packet, next)
            next()
        end)
        mandy.Define("Plugin.Event", {
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

```lua
local plugin = Mandy.Plugin(require(ReplicatedStorage.MyPlugin))
plugin:Unload()
```

---

## cleanup + janitor

```lua
-- explicit
Mandy.Remove("PlayerHit")
Mandy.Destroy()

p:Destroy()
bin:Destroy()
stream:Destroy()
token:Destroy()
plugin:Unload()
sub:Disconnect() / sub.Disconnect() / sub()

-- janitor
local jan = Mandy.Janitor()

jan:Add(Mandy.Packet("Hit", { ... }))
jan:Add(Mandy.Subscribe("PlayerHit", fn))
jan:Add(Mandy.Token())
jan:Add(function() end)

jan:Bind(player.Character)    -- auto-cleans when instance destroyed
jan:Remove(item)              -- untrack without destroying
jan:Cleanup()
```

---

## dev utilities

```lua
Mandy.Version
Mandy.Dev(true)
Mandy.Assert("PlayerHit", data)
Mandy.Validate("PlayerHit", data)
Mandy.Simulate("PlayerHit", player, data)
Mandy.Replay(snapshot)
Mandy.Has("PlayerHit")
Mandy.Get("PlayerHit")
Mandy.List()
Mandy.ListBin("Combat")
Mandy.Active("PlayerHit")
Mandy.Pause("PlayerHit")
Mandy.Resume("PlayerHit")
Mandy.Mute("PlayerHit")
Mandy.Unmute("PlayerHit")
Mandy.Stats("PlayerHit")
Mandy.Stats()
Mandy.Reset("PlayerHit")
Mandy.ResetAll()
Mandy.Extend(fn)
```

---

## api reference

| function | returns | description |
|----------|---------|-------------|
| `Mandy.Define(name, config)` | `void` | register a control globally |
| `Mandy.Load(controls)` | `void` | bulk define |
| `Mandy.Packet(name, config)` | `PacketHandle` | define + return handle |
| `Mandy.Bin(name, config?)` | `Bin` | named control registry |
| `Mandy.Collective(name, config)` | `Collective` | streaming channel |
| `Mandy.Mirror(name, config)` | `Mirror` | auto-replicating server state |
| `Mandy.Sync(name, config)` | `Sync` | bidirectional shared state |
| `Mandy.Bridge(name, config)` | `Bridge` | cross-server messaging |
| `Mandy.Contract(config)` | `void` | enforced request/response |
| `Mandy.Presence(config)` | `void` | player connection tracking |
| `Mandy.Subscribe(name, fn)` | `Subscription` | persistent listener |
| `Mandy.Once(name, fn)` | `Subscription` | one-shot listener |
| `Mandy.Await(name)` | `data` | yield until next packet |
| `Mandy.AwaitSafe(name)` | `ok, data, err` | safe yield |
| `Mandy.Expect(name)` | `data` | yield -- errors on reject |
| `Mandy.Post(name, playerOrData, data?)` | `Thenable?` | send packet |
| `Mandy.PostAll(name, data)` | `void` | broadcast -- server only |
| `Mandy.PostExclude(name, player, data)` | `void` | broadcast except one |
| `Mandy.Batch(fn)` | `void` | flush multiple posts in one frame |
| `Mandy.Validate(name, data)` | `boolean, string?` | validate payload |
| `Mandy.Token()` | `Token` | cancellation token |
| `Mandy.Janitor()` | `Janitor` | batch cleanup |
| `Mandy.Secure(config)` | `void` | activate security layer |
| `Mandy.Signed(fn)` | `void` | fires when bootstrap completes |
| `Mandy.Cross(fn)` | `void` | add global middleware |
| `Mandy.RateLimit(name, config)` | `void` | per-control rate limit |
| `Mandy.Flag(player, reason)` | `void` | manual flag |
| `Mandy.Unflag(player)` | `void` | clear flags |
| `Mandy.Flags(player)` | `FlagEntry[]` | flag history |
| `Mandy.Notice(config)` | `void` | observability hooks |
| `Mandy.Snapshot()` | `Snapshot` | point-in-time dump |
| `Mandy.Plugin(plugin)` | `PluginHandle` | register plugin |
| `Mandy.Extend(fn)` | `void` | wrap core methods globally |
| `Mandy.Remove(name)` | `void` | unregister control |
| `Mandy.Destroy()` | `void` | tear down everything |

---

## exported types

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

<div align="center">

*Built by [Plinko Labs](https://github.com/NotKisoMomo)*

</div>
