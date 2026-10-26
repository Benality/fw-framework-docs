# fw_admin

Version 1.1.0. Resource name `fw_admin`.

Source: `resources/[framework]/fw_admin`.

An in-game admin is a player whose identifiers are granted `fw.admin` in `permissions.cfg`. `server.cfg` loads that file with `exec permissions.cfg`, and `fw_admin` also reads the file itself on each check. The txAdmin web login is a different list. An account in `txData/admins.json` is not an in-game admin until `permissions.cfg` names one of that player's game identifiers.

`fw_players.role` is not consulted. It stays a label on the account (`user` by default). Setting it to `admin` in the database does not grant these checks.

On a successful start:

```
[fw_admin] ready
```

## permissions.cfg

File: `permissions.cfg` in the server data folder, next to `server.cfg`.

```
add_ace group.admin fw.admin allow
add_principal identifier.fivem:13478417 group.admin
add_principal identifier.discord:129445094406356993 group.admin
```

`fw.admin` means every admin check passes, including a narrower permission such as `fw.admin.kick`.

A moderator who should have one tool and not the rest gets that ace only, and does not get `fw.admin`:

```
add_ace group.mod fw.admin.kick allow
add_principal identifier.license:MOD_LICENSE group.mod
```

After an edit, restart the server, or run this in the server console:

```
exec permissions.cfg
```

A player who is already connected picks up a new principal on the next `exec`. They do not need to reconnect.

The console command `fw_admin [serverId]` prints that player's identifiers. Copy the `license:` line into `permissions.cfg` when the fivem or discord principal is not present on the game session.

## Exports

All exports are server-side.

| Export | Summary |
| --- | --- |
| [`IsReady()`](#isready) | Whether startup registration finished. |
| [`IsAdmin(source)`](#isadminsource) | Whether this server id holds `fw.admin`. |
| [`HasPermission(source, permission)`](#haspermissionsource-permission) | `fw.admin`, or one narrower `fw.admin.*` ace. |

Permission checks do not wait on `IsReady`. The ACE is applied by `exec`, before the resource starts.

### `IsReady()`

```lua
---@return boolean
exports.fw_admin:IsReady()
```

`true` after `fw_core` accepted the registration. `false` while core is still starting, and if registration failed.

### `IsAdmin(source)`

```lua
---@param source integer
---@return boolean
exports.fw_admin:IsAdmin(source)
```

| Source | Result |
| --- | --- |
| `0` | `true`. The server console is allowed. |
| A connected player with `fw.admin` | `true`. |
| Anyone else | `false`. |

```lua
if not exports.fw_admin:IsAdmin(source) then
    return
end
```

### `HasPermission(source, permission)`

```lua
---@param source integer
---@param permission string|nil
---@return boolean
exports.fw_admin:HasPermission(source, permission)
```

| Call | Result |
| --- | --- |
| Console (`source` `0`) | `true` for any permission. |
| Player with `fw.admin` | `true` for any permission. |
| Player with only `fw.admin.kick` | `true` for `'fw.admin.kick'`. `IsAdmin` is still `false`. |
| A permission that does not start with `fw.admin.` | `false`. This export will not answer for `command` or any other ACE. |

```lua
if not exports.fw_admin:HasPermission(source, 'fw.admin.kick') then
    return
end
```

## Command

| Command | Who | Effect |
| --- | --- | --- |
| `/admin` | An admin in the game | Opens the admin menu. Anyone else gets a chat line saying they are not an admin. |
| `/fw_admin` | The player who runs it | Chat says whether that player is an admin. It does not print identifiers. |
| `fw_admin` | Server console | Prints ready state and the usage line. |
| `fw_admin [serverId]` | Server console | Prints `admin=true` or `admin=false` and that player's identifiers. |

## Admin menu

`/admin` asks the server, and the server opens the menu only when `IsAdmin` is true. The menu is `fw_menu`.

The first entry is Developer Tools. Inside that, Teleports. Inside that, TP to Waypoint.

TP to Waypoint asks the server again. The server tells that client to move only if they are still an admin. The client reads the map waypoint with `IS_WAYPOINT_ACTIVE` (`0x202B1BBFC6AB5EE4`) and `GET_WAYPOINT_COORDS` (`0x29B30D07C3F7873B`). The screen fades out, the ped is frozen, and the client steps down from height 750, probing with `GET_GROUND_Z_AND_NORMAL_FOR_3D_COORD` (`0x2A29CA9A6319E6AB`). On a hit it loads collision around the ped, places them on that ground, then fades back in. With no waypoint, chat says to set one. If the ground cannot be found, the ped is put back where they started and the screen fades in.

## Who already calls it

`fw_clothing` admin outfit and wearable menu requests call `IsAdmin`. `fw_creator` `/deletechar` and `/secondchance` allow an admin through `IsAdmin` before they check `player.role` and the identifier list.
