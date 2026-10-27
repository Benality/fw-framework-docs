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
| `/admin` | A player in the game | Opens the admin menu if they are an admin. Anyone else gets a chat line saying they are not an admin. |
| `/fw_admin` | The player who types it | Chat says whether they are an admin. It does not print their ids. |
| `fw_admin` | Server window | Prints whether fw_admin is ready. |
| `fw_admin <serverId>` | Server window | Prints whether that player is an admin, and their ids, so you can copy the `license:` line into `permissions.cfg`. |
| `admin` (no slash) | Server window | Prints that `/admin` is for a player in the game. It does nothing else. |

`/admin` opens a menu from [fw_menu](fw-menu.md). The first entry is Developer Tools, then Teleports, then TP to Waypoint. The server checks `IsAdmin` again before it moves the player, so a menu left open by someone who loses admin will not let them teleport.

## Messages

These are networked events, sent with `RegisterNetEvent` and `TriggerServerEvent` / `TriggerClientEvent` — not `AddEventHandler`. None of them are queued: the listening side has to already be running when the message is sent, or it is simply missed. fw_admin's own client and server scripts are always running together while the resource is started, so this mostly matters if your own script triggers one of them.

A new admin tool should check `IsAdmin` or `HasPermission` itself first, then do its own work. You can open the same menu from your own server script:

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
TriggerClientEvent('fw_admin:openMenu', source)
```

### fw_admin:openMenu

Networked, server to the player's game. Opens the admin menu for that player. Their game shows "The menu is not running." instead if `fw_menu` is not started.

```lua
--- Networked. Trigger this from the server once you have already checked IsAdmin.
---@param source integer the player to open the menu for
TriggerClientEvent('fw_admin:openMenu', source)
```

### fw_admin:teleportWaypoint

Networked, player's game to the server. Asks to teleport that player to their own map waypoint. The server ignores it unless `IsAdmin` is yes for whoever sent it. The admin menu sends this itself — another script does not need to.

```lua
--- Networked. Sent by the admin menu. The server ignores it unless IsAdmin is yes for the sender.
TriggerServerEvent('fw_admin:teleportWaypoint')
```

### fw_admin:doTeleportWaypoint

Networked, server to the player's game. Moves that player to their own map waypoint. It is sent back automatically after a `fw_admin:teleportWaypoint` from an admin. If that player has no waypoint set, or the ground under it cannot be found, their game shows a chat line instead and nothing moves.

```lua
--- Networked. Sent automatically by fw_admin after a valid fw_admin:teleportWaypoint.
---@param source integer the player to move
TriggerClientEvent('fw_admin:doTeleportWaypoint', source)
```

Neither `fw_admin:teleportWaypoint` nor `fw_admin:doTeleportWaypoint` is meant to be triggered from another script. They exist for the admin menu's own Teleports screen.

## State bags

This resource does not set one.
