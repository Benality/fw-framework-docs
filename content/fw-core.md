# fw_core

Runs on the server. This is the account for a player who has joined. Your script reads that account here.

There is no code on the player's game.

## Why you check in

`fw_core` has a version written as three numbers, such as `1.1.0`.

Your script only cares about the first number. Right now that number is `1`. It means "these are the calls on this page." The other two numbers can go up when something is added that an old script can ignore. You never pass those two numbers.

`RegisterResource(1)` is a check-in. You are saying "I was written while the first number was 1." You do not pass your script's name. The server already knows who called.

The check-in does two jobs:

- While this server is still on `1`, the call succeeds. Typing `fw_health` in the server window lists your script.
- If a later update changes the calls so a script written for `1` would break, the first number becomes `2`. Your script then gets a clear line in the server window, telling you to update it, instead of failing in the middle of someone's visit.

The other calls on this page still work if you skip the check-in. Do it anyway, once, after `IsReady` says yes. The example on the [first page](README.md) already does this. Copy that.

`GetVersion` and `GetApiVersion` are for looking. A normal script does not call them.

## Calls

### IsReady

Runs on the server. Ask this before you use anything else on this page.

It stays no until two things are done. `fw_core` has a normal version, such as `1.1.0`, and the player list has been created in the database. If the version is wrong, or the player list cannot be created, the server window shows an error. `IsReady` stays no. People trying to join are turned away.

The server can say `fw_core` has started a moment before this is true. Checking once is too early. Keep checking for about two minutes. The example on the [first page](README.md) does that.

```lua
--- True when other scripts can use fw_core.
---@return boolean ready
exports.fw_core:IsReady()
```

```lua
if not exports.fw_core:IsReady() then
    return
end
```

### GetVersion

Runs on the server. Gives the full version, such as `1.1.0`. A normal script does not need this. It does not check the database. It can give you a version while `IsReady` is still no.

```lua
--- The version, such as '1.1.0', or nothing if the version is not three numbers.
---@return string|nil version
exports.fw_core:GetVersion()
```

```lua
print(exports.fw_core:GetVersion())
```

### GetApiVersion

Runs on the server. Gives the first number of the version, such as `1`. A normal script does not need this. `RegisterResource` already compares that number for you.

```lua
--- The first number of the version, or nothing if the version is not three numbers.
---@return integer|nil apiVersion
exports.fw_core:GetApiVersion()
```

```lua
print(exports.fw_core:GetApiVersion())
```

### RegisterResource

Runs on the server. Check in, as described in [Why you check in](#why-you-check-in). Call it once, from your own script, after `IsReady` says yes.

```lua
--- Checks this script in for a set of calls.
--- False when fw_core is not ready, the call did not come from another script, or the number does not match.
---@param requiredApiVersion integer
---@return boolean ok
---@return string|nil err
exports.fw_core:RegisterResource(requiredApiVersion)
```

Pass `1`. That is the first number of the version. `1.0` also works. `1.5`, `0`, and words do not. Pass the number this server is on. Passing `2` while the server is on `1` fails, and the reason names both numbers.

In the lines below, `your-script` stands in for your script's folder name.

| What you get back | When |
| --- | --- |
| `fw_core is not ready` | The player list is not ready yet. Keep checking, as on the [first page](README.md). |
| `RegisterResource must be called from another resource` | The call came from `fw_core` itself. Put it in your own script. |
| `your-script: required API version must be a positive integer` | You did not pass a whole number above 0. Pass `1`. |
| `fw_core API 1 is incompatible with required API 2; your-script needs a newer core` | You asked for `2`, and this server is still on `1`. |
| `fw_core API 2 is incompatible with required API 1; update your-script for API 2` | A later server is on `2`, and your script still passes `1`. Update the script, then pass `2`. |

The same line is printed in the server window. If it fails, stop there and read the line. The example below does that.

```lua
local ok, err = exports.fw_core:RegisterResource(1)
if not ok then
    print(err)
    return
end
```

Calling it again from the same script replaces the stored number. Stopping your script removes the check-in. There is no separate call to undo it.

### GetPlayer

Runs on the server. Gives a snapshot of the account for a player who has finished joining. You get nothing while they are still connecting, and nothing if that player number is not in the game. Changing the snapshot does not save.

```lua
--- The connected account, or nothing when that player has not finished joining.
---@param source integer
---@return table|nil player
exports.fw_core:GetPlayer(source)
```

`source` is the player's number for this visit. The fields are in [Account](#account).

```lua
local player = exports.fw_core:GetPlayer(source)
if player == nil then
    return
end
print(('source %d is account %d'):format(source, player.id))
```

### GetPlayerById

Runs on the server. The same account, found by account id instead of player number. You get nothing when that person is offline. This does not look in the database for someone who is not in the game.

```lua
--- The connected account for an account id, or nothing when that person is offline.
---@param playerId integer
---@return table|nil player
exports.fw_core:GetPlayerById(playerId)
```

`playerId` is the account id, `fw_players.id`.

```lua
local player = exports.fw_core:GetPlayerById(1)
if player == nil then
    return
end
```

### GetSourceByPlayerId

Runs on the server. The player number for an account that is online. You get nothing when that person is offline. Use this when you saved an account id and need to send that person a message in their game.

```lua
--- The player number for an online account, or nothing when that person is offline.
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

## Account

`GetPlayer`, `GetPlayerById`, `fw_core:playerLoaded`, and `fw_core:playerDropped` all give you this. It is a snapshot. Changing it does not save.

| Field | Meaning |
| --- | --- |
| `id` | Account id. Save this when something belongs to the person, not to one character. |
| `source` | Player number for this visit. It changes when they join again. |
| `license` | Their Rockstar license, including `license:` at the front. |
| `steam` | Steam id, or nothing. |
| `discord` | Discord id, or nothing. |
| `name` | The name they joined with. |
| `role` | A label on the account. New accounts are `user`. This does not make them an admin. Use [fw_admin](fw-admin.md). |
| `characterSlots` | How many characters this account may create. |
| `warnings` | A stored count. Nothing happens because of it. |
| `banned` | Always false here. A banned license never finishes joining. |
| `bannedUntil` | Always 0 here. |

There is no call to change role, warnings, character slots, or a ban. Change those in the `fw_players` table. Role, warnings, and a ban are read again the next time that person joins. The character slot count is read again the next time a character is created.

A person is turned away while joining when the database is down, the server is still starting, RedM did not send a license, the license is banned, or that same license is already in the server.

## Messages

These stay on the server. They are not sent to the player's game. Your script hears one only if it is already running. Put the listener at the top of your script. If your script starts later, you missed the message. Ask the matching call instead, where one is listed.

### fw_core:ready

Sent once, when the player list is ready. A script that starts later missed it. Ask `IsReady`.

```lua
--- version is text such as '1.1.0'. apiVersion is the first number, such as 1.
---@param version string
---@param apiVersion integer
AddEventHandler('fw_core:ready', function(version, apiVersion)
end)
```

### fw_core:playerLoaded

Sent when a player has finished joining. `player` is a snapshot of the account. Ask `GetPlayer` again when you need a fresh one.

```lua
---@param source integer
---@param player table
AddEventHandler('fw_core:playerLoaded', function(source, player)
    print(player.id)
end)
```

### fw_core:playerDropped

Sent when that player leaves. `player` is a snapshot taken before the account is forgotten.

```lua
---@param source integer
---@param player table
AddEventHandler('fw_core:playerDropped', function(source, player)
end)
```

## A value the player's game can read

`Player(source).state.fwPlayerId` is the account id. In the player's game:

```lua
local playerId = LocalPlayer.state.fwPlayerId
```

## Server window

Type `fw_health` in the server window. It prints whether `fw_core` is ready, the version, how many players are connected, and which scripts registered. Nothing happens if a player types it in chat.
