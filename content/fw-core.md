# fw_core

Server-only core API. Version `1.1.0`. API major `1`.

Source: `resources/[framework]/fw_core`.

`fw_core` parses its manifest version, creates `fw_players` if that table is missing, and admits the connecting player. It has no client script. Gameplay resources register with it so a future API break fails at startup with a named error, instead of failing silently later.

## API versioning

Manifest version is `MAJOR.MINOR.PATCH`. The API version is `MAJOR`.

| Rule | Detail |
| --- | --- |
| Minor bump | May add exports. Callers of the current API major keep working. |
| Patch bump | Never changes the API. |
| Major bump | A different API. No compatibility shims are kept for the old major. |
| `RegisterResource` | Takes one positive integer. Succeeds only when it equals the running major — there is no range syntax. |

`1.1.0` is API `1`. The player exports (`GetPlayer`, `GetPlayerById`, `GetSourceByPlayerId`) were added in `1.1.0` without changing the major.

## Startup

`IsReady()` stays `false` until the manifest version is a valid `MAJOR.MINOR.PATCH` and `sql/players.sql` has been applied. A bad version or a failed table setup logs the error and leaves the resource not ready; connecting players are then refused.

The resource yields inside `MySQL.ready.await` before it becomes ready, while FXServer already reports it as started. A dependent that calls `IsReady()` once and gives up will run too early — poll it, as `fw_characters` does, for up to 120 seconds.

On success:

```
[fw_core][info] ready version=1.1.0 api=1
```

The local event `fw_core:ready` fires with `(version, apiVersion)` right after. A resource started later by `ensure` has already missed that event and should call `IsReady()` directly instead.

Debug lines are off unless this convar is `1`, `true`, or `yes` (checked at the moment each line is written):

```
set fw_core_debug true
```

Startup, registration, and errors print regardless of the convar.

## Exports

All exports are server-side. Where an argument is documented as `integer`, a whole-number float also works — the implementation normalizes with `math.tointeger`, so `1` and `1.0` both resolve to `1`, while `1.5` is rejected.

| Export | Summary |
| --- | --- |
| [`IsReady()`](#isready) | Whether the resource is ready to serve calls. |
| [`GetVersion()`](#getversion) | The manifest version string. |
| [`GetApiVersion()`](#getapiversion) | The manifest major version as an integer. |
| [`RegisterResource(requiredApiVersion)`](#registerresourcerequiredapiversion) | Declares API compatibility for the calling resource. |
| [`GetPlayer(source)`](#getplayersource) | The connected account record for a server id. |
| [`GetPlayerById(playerId)`](#getplayerbyidplayerid) | The connected account record for an account id. |
| [`GetSourceByPlayerId(playerId)`](#getsourcebyplayeridplayerid) | The server id for a connected account id. |

---

### `IsReady()`

**Signature**

```lua
---@return boolean
exports.fw_core:IsReady()
```

**Parameters** — none.

**Returns**

| Type | Description |
| --- | --- |
| `boolean` | `true` once the manifest version parsed and `fw_players` is usable. `false` while starting, after a failed database setup, and immediately after `fw_core` stops. |

**Example**

```lua
if not exports.fw_core:IsReady() then
    return
end
```

---

### `GetVersion()`

**Signature**

```lua
---@return string|nil
exports.fw_core:GetVersion()
```

**Parameters** — none.

**Returns**

| Type | Description |
| --- | --- |
| `string` | The manifest version, e.g. `'1.1.0'`. |
| `nil` | Only when the manifest version string is missing or not shaped `MAJOR.MINOR.PATCH`. A database failure does **not** clear this — `IsReady()` can be `false` while this still returns `'1.1.0'`. |

**Example**

```lua
print(exports.fw_core:GetVersion()) --> '1.1.0'
```

---

### `GetApiVersion()`

**Signature**

```lua
---@return integer|nil
exports.fw_core:GetApiVersion()
```

**Parameters** — none.

**Returns**

| Type | Description |
| --- | --- |
| `integer` | The manifest major version, e.g. `1`. |
| `nil` | Same condition as `GetVersion()` — an invalid manifest version. |

Prefer this over parsing the string from `GetVersion()` yourself.

**Example**

```lua
print(exports.fw_core:GetApiVersion()) --> 1
```

---

### `RegisterResource(requiredApiVersion)`

**Signature**

```lua
---@param requiredApiVersion integer
---@return boolean success
---@return string|nil err
exports.fw_core:RegisterResource(requiredApiVersion)
```

Records that the calling resource was written for `requiredApiVersion`. Call it once from that resource's own startup, after core is ready. The caller is read via `GetInvokingResource()` — there is no resource-name argument, and calling this from `fw_core` itself always fails.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `requiredApiVersion` | `integer` | yes | The API major this resource was written for. Must be a positive whole number; floats with no fraction (`1.0`) are accepted. |

**Returns**

| Type | Description |
| --- | --- |
| `success: boolean` | `true` when registration succeeded. |
| `err: string \| nil` | `nil` on success; an error string otherwise (see Errors). |

**Errors**

| `err` value | Condition |
| --- | --- |
| `'fw_core is not ready'` | Core has not finished startup. |
| `'RegisterResource must be called from another resource'` | The caller was `fw_core` itself, or `GetInvokingResource()` was empty. |
| `'<resource>: required API version must be a positive integer'` | The argument was missing, fractional, zero, negative, or not a number. |
| `'fw_core API 1 is incompatible with required API 2; <resource> needs a newer core'` | The required integer is higher than the running API. |
| `'fw_core API 1 is incompatible with required API <n>; update <resource> for API 1'` | The required integer is lower than the running API. |

Every failure is also printed as `[fw_core][error]` by `fw_core` itself — the caller doesn't need to log it again, just decide whether to `error()` or continue.

Calling this again from the same resource replaces the stored requirement and logs `updated registration`. Stopping the resource removes the registration automatically; there is no unregister export.

**Example**

```lua
local ok, err = exports.fw_core:RegisterResource(1)
if not ok then
    print(err)
    return
end
```

---

### `GetPlayer(source)`

**Signature**

```lua
---@param source integer
---@return table|nil player
exports.fw_core:GetPlayer(source)
```

Returns a copy of the connected account for that server id, read from memory — this never queries the database and never loads a row by license.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Server id of the connected player. |

**Returns**

| Type | Description |
| --- | --- |
| `table` | A copy of the [player record](#player-record). |
| `nil` | `source` is not attached — this includes the entire `playerConnecting` deferral. The record exists from `playerJoining` onward (same moment `fw_core:playerLoaded` fires). |

**Example**

```lua
local player = exports.fw_core:GetPlayer(source)
if player then
    print(('source %d is account %d'):format(source, player.id))
end
```

---

### `GetPlayerById(playerId)`

**Signature**

```lua
---@param playerId integer
---@return table|nil player
exports.fw_core:GetPlayerById(playerId)
```

Same record and copy semantics as `GetPlayer`, looked up by `fw_players.id` instead of server id.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `playerId` | `integer` | yes | `fw_players.id` of the account. |

**Returns**

| Type | Description |
| --- | --- |
| `table` | A copy of the [player record](#player-record). |
| `nil` | That account is offline. This is an online-players lookup, not a database read — an id that exists in the table but is offline still returns `nil`. |

**Example**

```lua
local player = exports.fw_core:GetPlayerById(1)
```

---

### `GetSourceByPlayerId(playerId)`

**Signature**

```lua
---@param playerId integer
---@return integer|nil source
exports.fw_core:GetSourceByPlayerId(playerId)
```

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `playerId` | `integer` | yes | `fw_players.id` of the account. |

**Returns**

| Type | Description |
| --- | --- |
| `integer` | The connected server id for that account. |
| `nil` | That account is offline. |

**Example**

```lua
local target = exports.fw_core:GetSourceByPlayerId(playerId)
if target then
    TriggerClientEvent('my_resource:notify', target)
end
```

---

## Player record

`GetPlayer`, `GetPlayerById`, `fw_core:playerLoaded`, and `fw_core:playerDropped` all pass this same shape. It is always a **copy** — changing the returned table does not change the cache or the database.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | `integer` | `fw_players.id`. Other tables should store this. |
| `source` | `integer` | Current server id. Changes across reconnects. |
| `license` | `string` | `license:` or `license2:` value, prefix included. Max 64 bytes. |
| `steam` | `string \| nil` | `steam:` identifier when present and at most 40 bytes. |
| `discord` | `string \| nil` | `discord:` identifier when present and at most 40 bytes. |
| `name` | `string` | Player name: control characters removed, trimmed, clipped to 64 UTF-8 characters. Empty becomes `'unknown'`. |
| `role` | `string` | Account role. New rows use `'user'`. Max 16 characters. |
| `characterSlots` | `integer` | How many characters `fw_characters` may create. Column default is 5. |
| `warnings` | `integer` | Stored count. `fw_core` does not act on it. |
| `banned` | `boolean` | Always `false` on a connected player — a banned license never finishes connect. |
| `bannedUntil` | `integer` | Always `0` on a connected player. |

`role`, `warnings`, `character_slots`, and the ban columns are not writable through any export — change them directly in the table. `role`, `warnings`, and a ban are re-read on the account's *next connection*. `fw_characters` re-reads `character_slots` at the moment a character is created, so a slot change applies to the next create without requiring a reconnect.

## Connect, join, drop lifecycle

Identity comes from `GetPlayerIdentifierByType(source, 'license')`, falling back to `license2` if that's missing. One license cannot hold two server slots — the later connection is dropped with `This account is already connected.`

`playerConnecting` answers the deferral with one of:

| Message | Condition |
| --- | --- |
| `The server database is not available.` | Schema setup failed. |
| `The server is still starting. Try again in a moment.` | Schema setup has not finished. |
| `Rockstar license was not found. Restart RedM and try again.` | No usable license, or the license is longer than 64 bytes. |
| `You are banned from this server.` | `banned` is set and `banned_until` is `0` (no expiry). |
| `You are banned until YYYY-MM-DD HH:MM UTC.` | `banned` is set and `banned_until` is still in the future. |
| `Could not load your player.` | The admit query threw. |

An expired ban (`banned_until` > 0 and ≤ `os.time()`) is cleared on that connection and the player is admitted.

`playerJoining` attaches the cache, sets the state bag, and fires `fw_core:playerLoaded`. A failure here drops the player with the same class of message. `playerDropped` fires `fw_core:playerDropped` with the copy (before the cache entry is cleared), drops the cache, and updates `last_seen`.

There is no export to ban, warn, or change role from Lua. Those columns exist for an admin tool to update directly.

## Local events

Server events — not network events.

```lua
AddEventHandler('fw_core:ready', function(version, apiVersion)
    -- version: string, e.g. '1.1.0'
    -- apiVersion: integer, e.g. 1
end)

AddEventHandler('fw_core:playerLoaded', function(source, player)
    -- player: copy of the player record
end)

AddEventHandler('fw_core:playerDropped', function(source, player)
    -- player: copy captured before the cache entry was removed
end)
```

## State bag

`Player(source).state.fwPlayerId` is the account id. It replicates to the client:

```lua
local playerId = LocalPlayer.state.fwPlayerId
```

A failed write retries on a generation counter; a newer attach or drop cancels any retry that belonged to the previous value, so a stale retry can never overwrite a current id.

## Console command

```
fw_health
```

Restricted to the server console (`source == 0`); a player who runs it gets no output.

```
[fw_core] fw_health
  ready: true
  version: 1.1.0
  api: 1
  debug: true
  players: 1
    id=1 source=4 Player name
  registrations: 1
    fw_characters api=1
```

`debug` reflects the convar at the moment the command runs. `players` is the in-memory connected count. Each registration line is the resource name and the API major it required.

## Files

| File | Purpose |
| --- | --- |
| `fxmanifest.lua` | Version `1.1.0`, dependency `oxmysql`, server scripts only. |
| `sql/players.sql` | `CREATE TABLE IF NOT EXISTS fw_players`. |
| `server/players.lua` | Connect, cache, state bag, schema. |
| `server/main.lua` | Version, registration, health command, exports. |
