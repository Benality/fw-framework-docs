# fw_admin

Runs on the server. Ask this before you do something only an admin should do.

An admin is a player named in `permissions.cfg`. Being able to open the txAdmin website does not make them an admin. Setting their account label to `admin` in the database does not make them an admin either.

```
add_ace group.admin fw.admin allow
add_principal identifier.license:YOUR_LICENSE group.admin
```

`server.cfg` loads that file with the line `exec permissions.cfg`. After you edit it, restart the server, or type `exec permissions.cfg` in the server window. fw_admin reads `permissions.cfg` again on every check, so a player who is already in the game picks up the change right away. They do not have to leave and join again.

`fw.admin` passes every admin check, including a smaller one such as `fw.admin.kick`. A helper who should only kick gets that smaller permission, and does not get `fw.admin`.

Type `fw_admin` and a player number in the server window. It writes INFO lines with that player's ids so you can copy the `license:` line.

## Calls

### IsReady

Runs on the server. Says yes once fw_admin has finished starting: `fw_core` answered its own `IsReady` as yes, and fw_admin checked in with it.

You do not need to wait for this before calling `IsAdmin` or `HasPermission`. Those two already work as soon as fw_admin starts, even in the moment it is still waiting on `fw_core` underneath. `IsReady` is here for a script that only wants to confirm fw_admin itself started cleanly, for example after seeing `[fw_admin] ready` in the server window.

```lua
--- True once fw_admin has finished starting.
---@return boolean ready
exports.fw_admin:IsReady()
```

### IsAdmin

Runs on the server. Yes for the server window, and yes for a player who has `fw.admin`. No for everyone else, and no if you pass something that is not a real player number.

```lua
--- True when this player may use admin tools.
---@param source integer player number, or 0 for the server window
---@return boolean
exports.fw_admin:IsAdmin(source)
```

`source` is the player's number for this visit. Use `0` when the check is for the server window itself, not a player.

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
```

### HasPermission

Runs on the server. Yes when the player has `fw.admin`, or has the one smaller permission you name. A permission that does not start with `fw.admin.` is always no — this will not answer questions about any other server permission, admin or not.

```lua
--- True when the player has fw.admin, or the smaller permission you pass.
---@param source integer player number, or 0 for the server window
---@param permission string|nil such as 'fw.admin.kick'
---@return boolean
exports.fw_admin:HasPermission(source, permission)
```

The server window passes any `fw.admin.` permission you name. A player who only has `fw.admin.kick` passes this check for that name, and `IsAdmin` is still no for them.

```lua
if not exports.fw_admin:HasPermission(source, 'fw.admin.kick') then
    return
end
```

## Commands

| Command | Who | What happens |
| --- | --- | --- |
| `/admin` | A player in the game | Opens the admin menu when they have `fw.admin`. Anyone else gets a log line saying they are not an admin. |
| `/tpm` | A player in the game | Moves them to the waypoint on their map when they have `fw.admin`. Anyone else gets a log line saying they are not an admin. Set a waypoint first. If the ground under it cannot be found, the log says so and they stay where they are. |
| `/fw_admin` | The player who types it | The log says whether they are an admin. It does not print their ids. |
| `fw_admin` | Server window | Prints whether fw_admin is ready. |
| `fw_admin <serverId>` | Server window | Prints whether that player is an admin, and their ids, so you can copy the `license:` line into `permissions.cfg`. |
| `admin` (no slash) | Server window | Prints that `/admin` is for a player in the game. It does nothing else. |
| `tpm` | Server window | Prints that `/tpm` is for a player in the game. It does nothing else. |

`/admin` opens a menu from [fw_menu](fw-menu.md). It needs `fw.admin`. A smaller permission such as `fw.admin.kick` does not open it.

The first page has three entries.

- **Players.** Who is online. Search filters that list. Unban asks for a license. Choose a person, then go to them, bring them, send them back, watch them, freeze them, fill their health, revive them, give an item, give dollars, kick, or ban.
- **Self.** God mode, no clip, invisible, infinite ammo, fill health, fill health and stamina, and revive. No clip moves with W and S, up with Q, down with Z. Shift changes the speed. Delete stops it.
- **Developer tools.** Teleports, then copy coordinates. Teleports are the map waypoint, coordinates you type, Guarma, and the trip back. TP to Waypoint is the same move as `/tpm`. Copy coordinates puts x y z, a `vector3`, a `vector4`, or the heading on the clipboard. The same line is also written to the F8 log. The clipboard call is `exports.vorp_lib:copyToClipBoard`. If `vorp_lib` is not started, nothing is copied and the log says the clipboard is not available.

Go back on a page returns to the page that opened it.

Giving an item uses [fw_inventory](fw-inventory.md) `AddItem`. The count is from 1 to 500. Giving dollars uses [fw_money](fw-money.md) `AddCash`. The amount is from 1 to 100000. Both land in the satchel of the character they are playing. If that character is not selected, or the satchel has no room, nothing is given and the log says why.

A ban asks for the hours. `0` does not expire. The longest timed ban from the menu is 8760 hours, which is one year. The menu saves it with [fw_core](fw-core.md) `SetBan`, then removes them from the server. Unban asks for the license, including `license:` at the front. A hex id on its own is treated as a Rockstar license.

Bring remembers where that person was standing, on the server. Send back uses that spot. A second bring keeps the first spot. Guarma remembers where you were standing the same way. If there is no remembered spot, back from Guarma uses a mainland spot near Saint Denis.

The server checks the permission again before it does the thing. A menu left open by someone who loses admin will not let them teleport, kick, or give.

## Smaller permissions

`fw.admin` passes every check below. Give a helper only the line they should have. They still cannot open `/admin` without `fw.admin`. The check matters when their game asks the server directly.

| Permission | What it allows |
| --- | --- |
| `fw.admin.players` | The player list, go to, bring, send back, freeze, fill health, revive, and spectate. |
| `fw.admin.kick` | Kick. |
| `fw.admin.ban` | Ban and unban. |
| `fw.admin.give` | Give an item, or give dollars. |

Self tools, the waypoint, typed coordinates, Guarma, and copy coordinates need `fw.admin`.

```
add_ace group.helper fw.admin.kick allow
add_principal identifier.license:YOUR_LICENSE group.helper
```

## Messages

These are networked events, sent with `RegisterNetEvent` and `TriggerServerEvent` / `TriggerClientEvent` — not `AddEventHandler`. None of them are queued: the listening side has to already be running when the message is sent, or it is simply missed. fw_admin's own client and server scripts are always running together while the resource is started, so this mostly matters if your own script triggers one of them.

A new admin tool should check `IsAdmin` or `HasPermission` itself first, then do its own work. Prefer `AddItem`, `AddCash`, and `SetBan` when you are giving or banning from your own script. The events below belong to the menu. The server ignores an ask when the permission is no.

You can open the same menu from your own server script:

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
TriggerClientEvent('fw_admin:openMenu', source)
```

### fw_admin:openMenu

Networked, server to the player's game. Opens the admin menu for that player. Their game shows "The menu is not running." in the log instead if `fw_menu` is not started.

```lua
--- Networked. Trigger this from the server once you have already checked IsAdmin.
---@param source integer the player to open the menu for
TriggerClientEvent('fw_admin:openMenu', source)
```

### fw_admin:log

Networked, server to that player's game. The menu writes the same line in the server window and in their F8 log. `level` is `info`, `warn`, or `error`.

```lua
--- Networked. Sent by fw_admin with a line for that player's F8 log.
---@param message string
---@param level string|nil
TriggerClientEvent('fw_admin:log', source, message, level)
```

### Asks from the menu

Networked, the player's game to the server. `target` is the other player's number for this visit. The server reads where people are standing itself. Send back cannot be aimed at a spot the game invented.

| Event | Permission | Arguments |
| --- | --- | --- |
| `fw_admin:requestPlayers` | `fw.admin.players` | None. The reply is `fw_admin:playerList`. |
| `fw_admin:goTo` | `fw.admin.players` | `target` |
| `fw_admin:bring` | `fw.admin.players` | `target` |
| `fw_admin:sendBack` | `fw.admin.players` | `target` |
| `fw_admin:freeze` | `fw.admin.players` | `target`. Choose it again to let them move. |
| `fw_admin:heal` | `fw.admin.players` | `target`, `stamina`. `stamina` true also fills stamina. |
| `fw_admin:revive` | `fw.admin.players` | `target` |
| `fw_admin:spectate` | `fw.admin.players` | `target`. The same target again stops it. |
| `fw_admin:spectateLost` | `fw.admin` | None. Their game sends this when the person can no longer be watched. |
| `fw_admin:kick` | `fw.admin.kick` | `target` |
| `fw_admin:ban` | `fw.admin.ban` | `target`, `hours`. `0` does not expire. |
| `fw_admin:unban` | `fw.admin.ban` | `license` |
| `fw_admin:giveItem` | `fw.admin.give` | `target`, `name`, `count` |
| `fw_admin:giveCash` | `fw.admin.give` | `target`, `amount` |
| `fw_admin:self` | `fw.admin` | `tool`, and sometimes `false` to force it off. `tool` is `god`, `noclip`, `invis`, `ammo`, `heal`, `stamina`, or `revive`. |
| `fw_admin:teleportWaypoint` | `fw.admin` | None. |
| `fw_admin:teleportCoords` | `fw.admin` | `x`, `y`, `z` |
| `fw_admin:teleportGuarma` | `fw.admin` | None. |
| `fw_admin:teleportReturn` | `fw.admin` | None. |
| `fw_admin:copyCoords` | `fw.admin` | `kind`: `xyz`, `vector3`, `vector4`, or `heading`. |

```lua
--- Networked. Sent by the admin menu. The server ignores it unless that player may do it.
---@param target integer
TriggerServerEvent('fw_admin:goTo', target)
```

```lua
--- Networked. hours is a whole number. 0 does not expire.
---@param target integer
---@param hours integer
TriggerServerEvent('fw_admin:ban', target, hours)
```

```lua
--- Networked. false on the end forces the tool off. Delete does that for no clip.
---@param tool string
---@param force boolean|nil
TriggerServerEvent('fw_admin:self', tool, force)
```

### Replies to the player's game

Networked, server to one player. The menu sends these after the check. Another script should not trigger them. If you need to move, give, or ban, do that in your own script with your own check.

| Event | Who receives it | Arguments |
| --- | --- | --- |
| `fw_admin:playerList` | The admin | A list of `{ id, name }`, or false when they may not see it. |
| `fw_admin:doMove` | The player who is moving | `x`, `y`, `z`, `heading` |
| `fw_admin:doFreeze` | The player being frozen | `toggle` |
| `fw_admin:doHeal` | The player being healed | `stamina` |
| `fw_admin:doRevive` | The player being revived | None |
| `fw_admin:doSpectate` | The admin | `target`, or `0` to stop |
| `fw_admin:doSelf` | The admin | `tool`, `on` |
| `fw_admin:doTeleportWaypoint` | The admin | None |
| `fw_admin:doCopy` | The admin | `kind` |

```lua
--- Networked. Sent by the server after a player action it allowed.
---@param x number
---@param y number
---@param z number
---@param heading number|nil
TriggerClientEvent('fw_admin:doMove', source, x, y, z, heading)
```

### fw_admin:teleportWaypoint

Networked, player's game to the server. Asks to teleport that player to their own map waypoint. The server ignores it unless `IsAdmin` is yes for whoever sent it. The admin menu sends this itself. `/tpm` checks `fw.admin` and then does the same move.

```lua
--- Networked. Sent by the admin menu. The server ignores it unless IsAdmin is yes for the sender.
TriggerServerEvent('fw_admin:teleportWaypoint')
```

### fw_admin:doTeleportWaypoint

Networked, server to the player's game. Moves that player to their own map waypoint. It is sent back automatically after a `fw_admin:teleportWaypoint` from an admin. If that player has no waypoint set, or the ground under it cannot be found, their F8 log says so and nothing moves.

```lua
--- Networked. Sent automatically by fw_admin after a valid fw_admin:teleportWaypoint.
---@param source integer the player to move
TriggerClientEvent('fw_admin:doTeleportWaypoint', source)
```

## State bags

This resource does not set one.
