# Project Territory

These pages are the actions your script can use. Each one says where it runs, what you give it, what you get back, and shows a short example.

How to install a resource stays in that resource's `README.md`. Do not open another resource's database tables yourself. Use the action on its page.

## Words used here

- **Server** — the computer running the game. Players do not see this code.
- **Player's game** — RedM on their computer. Some actions only work there.
- **Call** — an action another script can use. It looks like `exports.fw_core:IsReady()`.
- **Player number** — written as `source`. The server gives each connected player a number. That number changes every time they join.
- **Nothing** — written as `nil`. The call did not find what you asked for.
- **Message** — written as an event. Your script hears it only if it is already running. If your script starts later, you missed it.

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
| [fw_core](fw-core.md) | Read the connected account, and check in that your script was written for the current set of calls. |
| [fw_menu](fw-menu.md) | Open a menu on the player's screen. |
| [fw_admin](fw-admin.md) | Check that a player is an in-game admin. |
| [fw_characters](fw-characters.md) | Create, read, select, and save a character. |
| [fw_creator](fw-creator.md) | See how the character menu stores a body and hair. Other resources should call `fw_characters`. |
| [fw_clothing](fw-clothing.md) | Read or replace the clothes the character is wearing. |
| [fw_inventory](fw-inventory.md) | Give items, open a chest or pouch, and define item types. |
| [Database](database.md) | See which id to store on your own table. |

`fw_core` creates the account when a player connects. `fw_characters` creates the character when something calls `CreateCharacter`. The character menu does that for players. `fw_clothing` saves worn clothes on that character. `fw_inventory` saves the satchel. `fw_satchel` is the screen.

Money and jobs do not exist yet. When they do, store `fw_characters.id` for something that belongs to one character, and `fw_players.id` for something that belongs to the account.

What you get back is a snapshot. Changing it does not save. Use the call that writes that field.

## Write a script

Wait until `fw_core` is ready, then check in with `RegisterResource(1)`.

The check-in is how your script says "I was written for the current set of calls." Right now that set is number `1`. You do not pick a version, and you do not pass your script's name. If a later update changes the calls so an old script would break, the server window tells you to update yours, instead of failing in the middle of someone's visit. [fw_core](fw-core.md) explains this in full. The other calls still work if you skip it. Do it anyway.

The server can say `fw_core` has started while it is still creating the player list. If you ask once and stop, you asked too soon. Keep asking for about two minutes.

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

Put `RegisterResource` in your own script. You do not type your script's name. The server already knows who called.

`fw_core:playerLoaded` is a message on the server. It is not sent to the player's game. Your script hears it only if it is already running. If the player joined before your script started, ask `GetPlayer` instead.

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
