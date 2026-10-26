# Project Territory framework

This folder is the development reference for the resources that exist today. Resource READMEs cover install. These pages cover the APIs, parameters, return values, events, and the tables a new resource should use.

Current resources, in start order:

| Resource | Version | Role |
| --- | --- | --- |
| `oxmysql` | 2.12.3 | Database pool. Creates the named database when it is missing. |
| `fw_core` | 1.1.0, API 1 | Account for the connecting license. |
| `fw_menu` | 1.0.0 | Client menu other resources open. No server script. |
| `fw_admin` | 1.1.0 | In-game admin checks and the admin menu. Registers with core API 1. |
| `fw_characters` | 1.0.0 | Character rows. Registers with core API 1. |
| `fw_creator` | 1.0.0 | Character menu. Registers with core API 1. |
| `fw_clothing` | 1.0.0 | Worn clothes, outfits, and wearable state. Registers with core API 1. |
| `fw_inventory` | 1.0.0 | Character satchel grid and item definitions. Registers with core API 1. |
| `fw_satchel` | 2.0.0 | Tetris satchel screen. Client only. |

`server.cfg` starts them after the RedM map resources:

```
ensure oxmysql
ensure fw_core
ensure fw_menu
ensure fw_admin
ensure fw_characters
ensure fw_creator
ensure fw_clothing
ensure fw_inventory
ensure fw_satchel
```

The folder `resources/[framework]/` is only a category. The resource name is the directory name: `fw_core`, `fw_menu`, `fw_admin`, `fw_characters`, `fw_creator`, `fw_clothing`, `fw_inventory`, `fw_satchel`.

## Pages

- [fw_core](fw-core.md) — readiness, API registration, the connected player, connect and drop events.
- [fw_menu](fw-menu.md) — the client menu other resources open.
- [fw_admin](fw-admin.md) — who counts as an in-game admin.
- [fw_characters](fw-characters.md) — create, read, update, select, and delete a character.
- [fw_creator](fw-creator.md) — how the menu writes a character, and the events the menu still uses.
- [fw_clothing](fw-clothing.md) — worn clothes, saved outfits, and wearable state on the character row.
- [fw_inventory](fw-inventory.md) — item definitions, the satchel grid, and the satchel screen.
- [Database](database.md) — `fw_players`, `fw_characters`, `fw_inventory`, `fw_items`, `fw_storage`, and what a later resource should key off.

## Who owns what

`fw_core` creates the account during connect. `fw_characters` creates the character when something calls `CreateCharacter`. `fw_creator` is that something for players. It also selects the character, saves the chosen spawn, and writes appearance and barber. `fw_clothing` writes the `clothing` document on that same row. `fw_inventory` stores that character's satchel in its own table. `fw_satchel` is the screen for it.

Money and jobs are not columns and not resources yet. When those resources exist, they should store `fw_characters.id`. The account id to store, when the record belongs to the account rather than one character, is `fw_players.id`.

Returned player and character tables are copies. Writing into the table a export gave you does not change the server record. Call the export that saves that field.

## Rules for the next resource

- Lua, `lua54`, RedM `game 'rdr3'`, and the `rdr3_warning` line.
- Call `exports.fw_core:RegisterResource(1)` once at startup, after `exports.fw_core:IsReady()` is true. `fw_characters` waits up to 120 seconds because FXServer marks `fw_core` started while it is still opening the database.
- Use exports for the API other resources call.
- Use local server events for lifecycle (`AddEventHandler`, not a network event).
- Use a network event only when the client must talk to the server.
- Keep the decision on the server. The client can ask; the server checks that the source owns the row.
- Do not read another resource's tables. Call its exports.
- Do not hold the table an export returned and treat it as live state. Read again when you need a fresh copy.

A resource that only needs the account looks like this:

```lua
CreateThread(function()
    local timeoutAt = GetGameTimer() + 120000
    while not exports.fw_core:IsReady() do
        if GetGameTimer() >= timeoutAt then
            print('[my_resource] fw_core is not ready')
            return
        end
        Wait(100)
    end

    local ok, err = exports.fw_core:RegisterResource(1)
    if not ok then
        print('[my_resource] ' .. tostring(err))
        return
    end
end)

AddEventHandler('fw_core:playerLoaded', function(source, player)
    print(('source %d is account %d'):format(source, player.id))
end)
```

`RegisterResource` must run from that resource's own script. `GetInvokingResource()` supplies the name. There is no resource-name argument.

## What a successful start prints

```
[fw_core][info] ready version=1.1.0 api=1
[fw_admin] ready
[fw_characters][info] registered fw_characters api=1
[fw_characters][info] ready version=1.0.0
[fw_creator] ready
[fw_clothing] ready. Purchases are not charged until a money resource exists.
[fw_inventory] ready. 5 items.
```

`fw_core` prints its own `registered` line for each resource. If the database in `mysql_connection_string` does not exist yet, oxmysql prints that it created the database and then the core line above.

In the server console:

```
fw_health
fw_characters
```

Both commands ignore a player who types them. They are restricted to the console.

## Reference copy

`.library/trp_creator`, `.library/trp_clothing`, and `.library/rsg-menubase-main` are the original resources. They are not ensured. `fw_creator`, `fw_clothing`, and `fw_menu` are the copies that talk to this framework. Change those, and leave the library copies as the originals.
