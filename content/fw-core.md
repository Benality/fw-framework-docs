# fw_core

Server-only. Version `1.1.0`. API major `1`.

Source: `resources/[framework]/fw_core`.

`fw_core` parses its manifest version, creates `fw_players` if that table is missing, and admits the connecting player. It has no client script. Gameplay resources register here so a future API break fails at startup with a named error.

Manifest version is `MAJOR.MINOR.PATCH`. The API version is `MAJOR`.

- A new minor version may add exports. Callers of API `1` keep working.
- A patch version does not change the API.
- A new major version is a different API. This core does not keep shims for the old major.
- `RegisterResource` takes one positive integer. It succeeds only when that integer equals the running major.
- A lower integer means the caller must be updated.
- A higher integer means this core is older than the caller.
- There is no version-range syntax.

`1.1.0` is API `1`. The player exports were added in `1.1.0` without changing the major.

## Startup

`IsReady` stays false until the manifest version is `MAJOR.MINOR.PATCH` and `sql/players.sql` has been applied. A bad version or a failed table setup logs the error and leaves the resource not ready. Connecting players are then refused.

The resource yields inside `MySQL.ready.await` before it becomes ready. FXServer still reports the resource as started at that yield. A dependent that calls `IsReady` once and gives up will run too early. Wait, as `fw_characters` does, up to 120 seconds.

On success:

```
[fw_core][info] ready version=1.1.0 api=1
```

Then the local event `fw_core:ready` runs with `(version, apiVersion)`. A resource started later by `ensure` should call `IsReady` itself. It has already missed that event.

Debug lines are off unless the convar is `1`, `true`, or `yes`. The convar is read when a line is written.

```
set fw_core_debug true
```

Startup, registration, and errors print without the convar.

## Exports

All exports are server-side. Export numbers may arrive as whole floats. The implementation uses `math.tointeger`, so `1` and `1.0` are the integer `1`. `1.5` is rejected where an integer is required.

### IsReady

```lua
---@return boolean
exports.fw_core:IsReady()
```

`true` after the version parsed and `fw_players` is usable. `false` while starting, after a failed database setup, and as soon as `fw_core` stops.

```lua
if not exports.fw_core:IsReady() then
    return
end
```

### GetVersion

```lua
---@return string|nil
exports.fw_core:GetVersion()
```

The manifest version, such as `'1.1.0'`. `nil` when the manifest version is missing or not `MAJOR.MINOR.PATCH`. A database failure does not clear it. `IsReady()` can be false while this still returns `'1.1.0'`.

### GetApiVersion

```lua
---@return integer|nil
exports.fw_core:GetApiVersion()
```

The major as an integer, currently `1`. `nil` in the same case as `GetVersion`. Use this instead of parsing the version string.

### RegisterResource

```lua
---@param requiredApiVersion integer
---@return boolean success
---@return string|nil err
exports.fw_core:RegisterResource(requiredApiVersion)
```

Records that the calling resource was written for `requiredApiVersion`. Call it once from that resource after core is ready.

| Result | Meaning |
| --- | --- |
| `true, nil` | Registered. |
| `false, 'fw_core is not ready'` | Core has not finished startup. |
| `false, 'RegisterResource must be called from another resource'` | The caller was `fw_core` itself, or `GetInvokingResource()` was empty. |
| `false, '<resource>: required API version must be a positive integer'` | The argument was missing, fractional, zero, negative, or not a number. |
| `false, 'fw_core API 1 is incompatible with required API 2; <resource> needs a newer core'` | The required integer is higher than the running API. |
| `false, 'fw_core API 1 is incompatible with required API <n>; update <resource> for API 1'` | The required integer is lower than the running API. |

Every failure is also printed as `[fw_core][error]`. The caller decides whether to stop.

Calling it again from the same resource replaces the stored requirement and logs `updated registration`. Stopping the resource removes the registration. There is no unregister export.

```lua
local ok, err = exports.fw_core:RegisterResource(1)
if not ok then
    print(err)
    return
end
```

### GetPlayer

```lua
---@param source integer
---@return table|nil player
exports.fw_core:GetPlayer(source)
```

Copy of the connected account for that server id. The read is from memory. It does not query the database.

Returns `nil` when that source is not attached. That includes the whole `playerConnecting` deferral. The record exists from `playerJoining` onward, which is also when `fw_core:playerLoaded` fires. An offline account returns `nil`. This export does not load a row by license.

```lua
local player = exports.fw_core:GetPlayer(source)
if player then
    print(('source %d is account %d'):format(source, player.id))
end
```

### GetPlayerById

```lua
---@param playerId integer
---@return table|nil player
exports.fw_core:GetPlayerById(playerId)
```

Same copy as `GetPlayer`, found by `fw_players.id`. Online players only. An id that exists in the table and is offline returns `nil`.

### GetSourceByPlayerId

```lua
---@param playerId integer
---@return integer|nil source
exports.fw_core:GetSourceByPlayerId(playerId)
```

The connected server id for an account id, or `nil` when that account is offline.

```lua
local target = exports.fw_core:GetSourceByPlayerId(playerId)
if target then
    TriggerClientEvent('my_resource:notify', target)
end
```

## Player record

`GetPlayer` and `GetPlayerById` return this shape. `playerLoaded` and `playerDropped` pass the same copy.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | integer | `fw_players.id`. Other tables should store this. |
| `source` | integer | Current server id. It can change across reconnects. |
| `license` | string | `license:` or `license2:` value, including the prefix. Max 64 bytes. |
| `steam` | string or nil | `steam:` identifier when present and at most 40 bytes. |
| `discord` | string or nil | `discord:` identifier when present and at most 40 bytes. |
| `name` | string | Player name, control characters removed, trimmed, clipped to 64 UTF-8 characters. Empty becomes `unknown`. |
| `role` | string | Account role. New rows use `user`. Max 16 characters in the column. |
| `characterSlots` | integer | How many characters `fw_characters` may create. Column default is 5. |
| `warnings` | integer | Stored count. Core does not act on it. |
| `banned` | boolean | Always `false` on a connected player. A banned license never finishes connect. |
| `bannedUntil` | integer | Always `0` on a connected player. |

Changing the returned table does not change the cache or the database.

`role`, `warnings`, `character_slots`, and ban columns are not writable through an export. Change them in the table. `role`, `warnings`, and a ban are read again on the next connection. `fw_characters` re-reads `character_slots` at the moment a character is created, so a slot change applies to the next create without a reconnect.

## Connect, join, drop

Identity is `GetPlayerIdentifierByType(source, 'license')`. If that is missing, core uses `license2`. The stored value keeps the prefix. One license cannot hold two server slots. The later connection is dropped with `This account is already connected.`

`playerConnecting` captures `source` before the first `Wait`. The player is not in `GetPlayer` during this phase. Core answers the deferral with one of:

| Message | When |
| --- | --- |
| `The server database is not available.` | Schema setup failed. |
| `The server is still starting. Try again in a moment.` | Schema setup has not finished. |
| `Rockstar license was not found. Restart RedM and try again.` | No usable license, or the license is longer than 64 bytes. |
| `You are banned from this server.` | `banned` is set and `banned_until` is `0` (no expiry). |
| `You are banned until YYYY-MM-DD HH:MM UTC.` | `banned` is set and `banned_until` is still in the future. |
| `Could not load your player.` | The admit query threw. |

An expired ban (`banned_until` greater than 0 and less than or equal to `os.time()`) is cleared on that connection, and the player is admitted.

`playerJoining` attaches the cache, sets the state bag, and fires `fw_core:playerLoaded`. A failure drops the player with the same class of message.

`playerDropped` fires `fw_core:playerDropped` with the copy, drops the cache, and updates `last_seen`. The event runs before the cache is cleared.

There is no export to ban, warn, or change role from Lua yet. Those columns exist so an admin tool can update the row.

## Local events

Server events. They are not network events.

```lua
AddEventHandler('fw_core:ready', function(version, apiVersion)
    -- version: string, such as '1.1.0'
    -- apiVersion: integer, such as 1
end)

AddEventHandler('fw_core:playerLoaded', function(source, player)
    -- player: copy of the account record
end)

AddEventHandler('fw_core:playerDropped', function(source, player)
    -- player: copy captured before the cache entry was removed
end)
```

## State bag

`Player(source).state.fwPlayerId` is the account id. It replicates.

```lua
local playerId = LocalPlayer.state.fwPlayerId
```

A failed write retries on a generation counter. A newer attach or drop cancels a retry that belonged to the previous value, so an old retry cannot write a stale id.

## Console command

```
fw_health
```

Restricted to the server console (`source == 0` and the command is restricted). A player who runs it gets no output.

With core ready, one connected account, and `fw_characters` registered:

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

`debug` follows the convar at the moment the command runs. `players` is the in-memory count. Each registration line is the resource name and the API it required.

## Files

| File | Purpose |
| --- | --- |
| `fxmanifest.lua` | Version `1.1.0`, dependency `oxmysql`, server scripts only. |
| `sql/players.sql` | `CREATE TABLE IF NOT EXISTS fw_players`. |
| `server/players.lua` | Connect, cache, state bag, schema. |
| `server/main.lua` | Version, registration, health command, exports. |
