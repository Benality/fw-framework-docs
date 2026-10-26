# fw_core

Server. This is the account for a player who has joined. Your resource reads that account here, and tells the server which API number it was written for.

There is no client script.

The version is `1.1.0`. The first number, `1`, is the API number. Pass that number to `RegisterResource`. A later `1.2.0` can add calls and your resource still works. A future `2.0.0` is a different API, and your resource must be updated.

## Server exports

### IsReady

Server. True after the account table is ready. False while the database is still opening, or if that setup failed.

```lua
--- True when other resources can call fw_core.
---@return boolean ready
exports.fw_core:IsReady()
```

```lua
if not exports.fw_core:IsReady() then
    return
end
```

The server can say `fw_core` has started before this returns true. Wait for it, as the example on the [overview](README.md) does.

### GetVersion

Server. The full version string, such as `1.1.0`. This does not check the database. It can return a version while `IsReady` is still false.

```lua
--- The version string from this resource, or nil if that string is not three numbers.
---@return string|nil version
exports.fw_core:GetVersion()
```

```lua
print(exports.fw_core:GetVersion())
```

### GetApiVersion

Server. The API number, such as `1`. Use this instead of splitting the version string yourself.

```lua
--- The API number, or nil if the version string is not three numbers.
---@return integer|nil apiVersion
exports.fw_core:GetApiVersion()
```

```lua
print(exports.fw_core:GetApiVersion())
```

### RegisterResource

Server. Tell `fw_core` that your resource was written for an API number. Call it once, from your own script, after `IsReady` is true.

```lua
--- Records that this resource was written for an API number.
--- False when core is not ready, the call did not come from another resource, or the number does not match.
---@param requiredApiVersion integer
---@return boolean ok
---@return string|nil err
exports.fw_core:RegisterResource(requiredApiVersion)
```

`requiredApiVersion` must be the running API number. `1` and `1.0` both work. `1.5`, `0`, and text do not. There is no range. Asking for `2` while the server is on `1` fails, and the error names both numbers.

| `err` | When |
| --- | --- |
| `fw_core is not ready` | The database setup has not finished. |
| `RegisterResource must be called from another resource` | The call came from `fw_core` itself. |
| `<resource>: required API version must be a positive integer` | The number was missing or not a whole number above 0. |
| `fw_core API 1 is incompatible with required API 2; <resource> needs a newer core` | You asked for a newer API than this server has. |
| `fw_core API 1 is incompatible with required API <n>; update <resource> for API 1` | You asked for an older API. |

`fw_core` also prints the error. You only need to stop your resource or keep going.

```lua
local ok, err = exports.fw_core:RegisterResource(1)
if not ok then
    print(err)
    return
end
```

Calling it again from the same resource replaces the stored number. Stopping your resource removes the registration. There is no separate unregister call.

### GetPlayer

Server. A copy of the account for a player who has finished joining. Nil during the connection check, and nil if that server id is not in the game.

```lua
--- The connected account, or nil when that player has not finished joining.
---@param source integer
---@return table|nil player
exports.fw_core:GetPlayer(source)
```

`source` is the player's server id. The table is a copy. Changing it does not save. The fields are in [Account record](#account-record).

```lua
local player = exports.fw_core:GetPlayer(source)
if player == nil then
    return
end
print(('source %d is account %d'):format(source, player.id))
```

### GetPlayerById

Server. The same account, looked up by account id instead of server id. Nil when that account is offline. This does not read the database.

```lua
--- The connected account for an account id, or nil when that account is offline.
---@param playerId integer
---@return table|nil player
exports.fw_core:GetPlayerById(playerId)
```

`playerId` is `fw_players.id`.

```lua
local player = exports.fw_core:GetPlayerById(1)
if player == nil then
    return
end
```

### GetSourceByPlayerId

Server. The server id for an account that is online. Nil when that account is offline. Use this when you stored an account id and need to send that player a client event.

```lua
--- The server id for an online account, or nil when that account is offline.
---@param playerId integer
---@return integer|nil source
exports.fw_core:GetSourceByPlayerId(playerId)
```

```lua
local target = exports.fw_core:GetSourceByPlayerId(playerId)
if target == nil then
    return
end
TriggerClientEvent('my_resource:notify', target)
```

## Account record

`GetPlayer`, `GetPlayerById`, `fw_core:playerLoaded`, and `fw_core:playerDropped` all use this shape. It is a copy.

| Field | Meaning |
| --- | --- |
| `id` | Account id. Store this when a row belongs to the account, not one character. |
| `source` | Server id. It changes when they reconnect. |
| `license` | Rockstar license, including the `license:` prefix. |
| `steam` | Steam id, or nil. |
| `discord` | Discord id, or nil. |
| `name` | The name they joined with. |
| `role` | Account label. New accounts are `user`. This is not an admin check. Use [fw_admin](fw-admin.md). |
| `characterSlots` | How many characters this account may create. |
| `warnings` | A stored count. `fw_core` does not act on it. |
| `banned` | Always false here. A banned license never finishes connecting. |
| `bannedUntil` | Always 0 here. |

There is no export to change role, warnings, slots, or a ban. Change those in the `fw_players` table. Role, warnings, and a ban are read again the next time that license connects. The character slot count is read again the next time a character is created.

A player is refused while connecting when the database is down, the server is still starting, RedM did not send a license, the license is banned, or the same license is already in the server.

## Events

These are local server events. Listen with `AddEventHandler`. They are not saved. Your script must already be running, except where a note says to call an export instead.

### fw_core:ready

Fires once, after the account table is ready. A resource started later has missed it. Call `IsReady`.

```lua
--- version is a string such as '1.1.0'. apiVersion is the API number, such as 1.
---@param version string
---@param apiVersion integer
AddEventHandler('fw_core:ready', function(version, apiVersion)
end)
```

### fw_core:playerLoaded

Fires when a player has finished joining. `player` is a copy of the account. Call `GetPlayer` again when you need a fresh one.

```lua
---@param source integer
---@param player table
AddEventHandler('fw_core:playerLoaded', function(source, player)
    print(player.id)
end)
```

### fw_core:playerDropped

Fires when that player leaves. `player` is a copy taken before the account is dropped from memory.

```lua
---@param source integer
---@param player table
AddEventHandler('fw_core:playerDropped', function(source, player)
end)
```

## Value on the player

`Player(source).state.fwPlayerId` is the account id. The client can read it:

```lua
local playerId = LocalPlayer.state.fwPlayerId
```

## Console command

`fw_health` prints whether core is ready, the version, the API number, how many players are connected, and which resources registered. It does nothing when a player types it.
