# Project Territory

These pages are the calls another resource can use. Each one says which side it runs on, what you pass, what you get back, and shows a short example.

Install notes stay in each resource's `README.md`. Do not read another resource's database tables. Call the export.

## Start order

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

The folder `resources/[framework]/` is only a category. The resource name is the directory name, such as `fw_core`.

## What each resource is for

| Resource | You use it to |
| --- | --- |
| [fw_core](fw-core.md) | Read the connected account, and tell the server your resource was written for API 1. |
| [fw_menu](fw-menu.md) | Open a menu on the player's screen. |
| [fw_admin](fw-admin.md) | Check that a player is an in-game admin. |
| [fw_characters](fw-characters.md) | Create, read, select, and save a character. |
| [fw_creator](fw-creator.md) | See how the character menu stores a body and hair. Other resources should call `fw_characters`. |
| [fw_clothing](fw-clothing.md) | Read or replace the clothes the character is wearing. |
| [fw_inventory](fw-inventory.md) | Give items, open a chest or pouch, and define item types. |
| [Database](database.md) | See which id to store on your own table. |

`fw_core` creates the account when a player connects. `fw_characters` creates the character when something calls `CreateCharacter`. The character menu does that for players. `fw_clothing` saves worn clothes on that character. `fw_inventory` saves the satchel. `fw_satchel` is the screen.

Money and jobs do not exist yet. When they do, store `fw_characters.id` for something that belongs to one character, and `fw_players.id` for something that belongs to the account.

A table you get back from an export is a copy. Changing it does not save. Call the export that writes that field.

## Write a resource

Wait until `fw_core` is ready, then register. The server can mark `fw_core` as started while it is still opening the database, so poll for up to two minutes.

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

`RegisterResource` must run from your resource's own script. You do not pass the resource name.

`fw_core:playerLoaded` is a local server event. Use `AddEventHandler`. It is not saved for a script that starts later. If your resource starts after the player is already in, call `GetPlayer`.

## A good start looks like this

```
[fw_core][info] ready version=1.1.0 api=1
[fw_admin] ready
[fw_characters][info] ready version=1.0.0
[fw_creator] ready
[fw_clothing] ready. Purchases are not charged until a money resource exists.
[fw_inventory] ready. 5 items.
```

In the server console, `fw_health` and `fw_characters` print status. A player who types them gets nothing.
