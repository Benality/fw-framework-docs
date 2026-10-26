# fw_admin

Runs on the server. Ask this before you do something only an admin should do.

An admin is a player named in `permissions.cfg`. Being able to open the txAdmin website does not make them an admin. Setting their account label to `admin` in the database does not make them an admin either.

```
add_ace group.admin fw.admin allow
add_principal identifier.license:YOUR_LICENSE group.admin
```

`server.cfg` loads that file with the line `exec permissions.cfg`. After you edit it, restart the server, or type `exec permissions.cfg` in the server window. A player who is already in the game picks up the change. They do not have to leave and join again.

`fw.admin` passes every admin check, including a smaller one such as `fw.admin.kick`. A helper who should only kick gets that smaller permission, and does not get `fw.admin`.

Type `fw_admin` and a player number in the server window. It prints that player's ids so you can copy the `license:` line.

## Calls

### IsReady

Runs on the server. Says yes after this resource has told `fw_core` it is here. You do not need to wait for it before an admin check. The permission file is loaded first.

```lua
--- True after fw_admin has checked in with fw_core.
---@return boolean ready
exports.fw_admin:IsReady()
```

### IsAdmin

Runs on the server. Yes for the server window, and yes for a player who has `fw.admin`. No for everyone else.

```lua
--- True when this player may use admin tools.
---@param source integer player number, or 0 for the server window
---@return boolean
exports.fw_admin:IsAdmin(source)
```

`source` is the player's number for this visit. Use `0` when the command was typed in the server window.

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
```

### HasPermission

Runs on the server. Yes when the player has `fw.admin`, or has the one permission you name. A permission that does not start with `fw.admin.` is always no. This will not answer questions about other server permissions.

```lua
--- True when the player has fw.admin, or the smaller permission you pass.
---@param source integer player number, or 0 for the server window
---@param permission string|nil such as 'fw.admin.kick'
---@return boolean
exports.fw_admin:HasPermission(source, permission)
```

The server window is allowed for any permission. A player who only has `fw.admin.kick` passes this check for that name, and `IsAdmin` is still no.

```lua
if not exports.fw_admin:HasPermission(source, 'fw.admin.kick') then
    return
end
```

## Commands

| Command | Who | What happens |
| --- | --- | --- |
| `/admin` | An admin in the game | Opens the admin menu. Anyone else gets a chat line saying they are not an admin. |
| `/fw_admin` | The player who types it | Chat says whether they are an admin. It does not print their ids. |
| `fw_admin` | Server window | Prints whether this resource is ready. |
| `fw_admin <serverId>` | Server window | Prints whether that player is an admin, and their ids. |

`/admin` opens a menu from [fw_menu](fw-menu.md). The first entry is Developer Tools, then Teleports, then TP to Waypoint. The server checks `IsAdmin` again before it moves the player.
