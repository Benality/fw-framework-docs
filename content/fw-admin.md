# fw_admin

Server. Ask this before you do something only an admin should do.

An admin is a player named in `permissions.cfg`, not someone who can open txAdmin, and not a row whose `role` column says `admin`.

```
add_ace group.admin fw.admin allow
add_principal identifier.license:YOUR_LICENSE group.admin
```

`server.cfg` loads that file with `exec permissions.cfg`. After you edit it, restart the server or run `exec permissions.cfg` in the server console. A player who is already connected picks up the change. They do not have to reconnect.

`fw.admin` passes every admin check, including a narrower one such as `fw.admin.kick`. A moderator who should only kick gets that narrower permission and does not get `fw.admin`.

The console command `fw_admin <serverId>` prints that player's identifiers so you can copy the `license:` line.

## Server exports

### IsReady

Server. True after this resource has registered with `fw_core`. You do not need to wait for it before an admin check. The permission file is loaded first.

```lua
--- True after fw_admin has registered with fw_core.
---@return boolean ready
exports.fw_admin:IsReady()
```

### IsAdmin

Server. True for the server console, and true for a player who has `fw.admin`. False for everyone else.

```lua
--- True when this player may use admin tools.
---@param source integer server id, or 0 for the server console
---@return boolean
exports.fw_admin:IsAdmin(source)
```

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
```

### HasPermission

Server. True when the player has `fw.admin`, or has the one permission you name. A permission that does not start with `fw.admin.` is always false. This will not answer for other server permissions.

```lua
--- True when the player has fw.admin, or the narrower permission you pass.
---@param source integer server id, or 0 for the server console
---@param permission string|nil such as 'fw.admin.kick'
---@return boolean
exports.fw_admin:HasPermission(source, permission)
```

The console is allowed for any permission. A player who only has `fw.admin.kick` passes this check for that name, and `IsAdmin` is still false.

```lua
if not exports.fw_admin:HasPermission(source, 'fw.admin.kick') then
    return
end
```

## Commands

| Command | Who | What happens |
| --- | --- | --- |
| `/admin` | An admin in the game | Opens the admin menu. Anyone else gets a chat line saying they are not an admin. |
| `/fw_admin` | The player who runs it | Chat says whether they are an admin. It does not print identifiers. |
| `fw_admin` | Server console | Prints whether the resource is ready. |
| `fw_admin <serverId>` | Server console | Prints whether that player is an admin, and their identifiers. |

`/admin` opens a `fw_menu` menu. The first entry is Developer Tools, then Teleports, then TP to Waypoint. The server checks `IsAdmin` again before it moves the player.
