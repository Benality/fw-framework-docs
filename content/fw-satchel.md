# fw_satchel

Runs on the player's game. There is no code on the server. This is the bag screen. It draws the grid from [fw_inventory](fw-inventory.md) and the loadout from [fw_weapons](fw-weapons.md).

It has no exports and no state bag. Give an item, open a chest, or define an item with the calls on the inventory page. Read the equipped weapons with `GetKit` on the weapons page. This page exists so you know which messages the screen is already listening for, and which call to use instead of sending your own.

The player opens the bag with `B`, or by typing `/satchel`. Escape or Tab closes it. Closing the bag also closes an open chest or pouch. During a trade, Escape or Tab declines the trade instead, and the bag closes when the trade ends. That flow is on [fw_trade](fw-trade.md).

Right-click a placement in the bag for Use, Give, Drop, or Split. Give hides the bag, shows markers on nearby players, and hands that whole stack to the one they click. The bag comes back after. Split is only there when the stack is more than one. Give is not on the menu during a trade.

## Trade screen

`fw_trade` asks this screen to show the bag with two offer grids beside it. The messages stay in the player's game. Your script should not send them. Start a trade with the prompt or `/trade`, described on [fw_trade](fw-trade.md). `/tradeui` opens the same screen with nobody else in it, so you can look at the layout. Escape closes that preview.

| Message | What the screen does with it |
| --- | --- |
| `fw_satchel:tradeBegin` | Opens the bag, or switches the open bag, to the trade layout. The argument is the current trade view. |
| `fw_satchel:tradeNotice` | Shows a short reason under the grid. The trade stays open. |
| `fw_satchel:tradeEnd` | Closes the bag. |
| `fw_satchel:shopToggle` | Opens the sample counter beside the bag, or closes the bag if that counter is already open. The argument is the clerk's goods. `fw_stores` sends it from `/shop`. The bag is the one the player is carrying. |
| `fw_satchel:shopVisible` | The screen sends this. `true` means the counter is on screen. `false` means it just closed. Listen with `AddEventHandler`. It is not queued. Do not send it yourself. |

```lua
--- The counter is on screen when visible is true, and gone when it is false.
---@param visible boolean
---@return nil
AddEventHandler('fw_satchel:shopVisible', function(visible)
    -- Hide a prompt while the counter is up.
end)
```

Right-click a filled loadout slot for Inspect and Clean. That menu is only on the loadout, not on an item sitting in the bag. Inspect draws the gun and plays the inspect hold, then the bag closes. A knife, lasso, lantern, binoculars, bow, or fishing pole cannot be inspected. Clean does nothing yet.

The gunbelt sits on the left of those slots. Each kind on the character is an icon the size of one satchel cell, with the count in the corner. Hover an icon to read the name. A kind at zero is left off. Nothing on the belt shows `Nothing on the belt.` Drag a box or a bundle of arrows from the bag onto the gunbelt to load one. The gunbelt stays closed until you press its button. The Kit filter on the left keeps weapons and ammo visible together.

Drag a gunbelt icon onto the satchel, or right-click it and choose Unload, to put one full box back in the bag. Those rounds come off that kind only. The bag stays open. If fewer than a full box are left, the menu says Empty. Empty clears the leftover and does not give an item. Dragging a short stack onto the satchel shows `Not a full box.` and leaves the rounds in place. If the bag cannot hold the box, nothing changes and the player sees `You cannot carry any more of those.`

Use an ammo box or a bundle of arrows from the bag. That puts rounds on the character. The bag stays open. A box shows the revolver-cylinder loading prompt at the bottom right for about two seconds. Backspace cancels it and keeps the box. Draw the gun and press reload. Firing spends those rounds. The bag is not charged again. That is on [fw_weapons](fw-weapons.md). Inspect uses `Inspect` on that same page. Read the belt with `GetBelt` on that page.

`fwInventoryBusy` on [fw_menu](fw-menu.md) is how another menu asks this screen to stay closed.

## The command

`/satchel` is a client command, typed in chat or the console. It toggles the screen: closed if open, otherwise it asks the server to open it. It takes no arguments and gives nothing back. A script should not run it for the player — trigger the underlying calls on [fw_inventory](fw-inventory.md) instead, as shown below.

## Messages the screen uses

These go between the bag screen and the server. A new script should use the call named in the last column instead of sending or listening for the message itself. Sending the screen's own message skips the checks that call already does, or fights the screen for the same drag.

**Message** here means a networked event — sent with `TriggerServerEvent` or `TriggerClientEvent`, heard with `RegisterNetEvent`. Your script hears one only if it is already running when the message is sent. A script that starts after the message went out missed it; there is no queue to catch up on later.

### From the server to the player's game

Heard with `RegisterNetEvent`. These are sent by [fw_inventory](fw-inventory.md) and [fw_weapons](fw-weapons.md), not by this resource.

| Message | What the screen does with it | What you should use |
| --- | --- | --- |
| `fw_inventory:opened` | Shows the bag, and a chest or pouch when one was opened. | `OpenStorage` |
| `fw_inventory:updated` | Redraws the bag after a move. | `AddItem`, `RemoveItem`, `GetItems` |
| `fw_inventory:definitions` | Learns item names, sizes, and pictures. | `RegisterItem`, `GetDefinition` |
| `fw_inventory:rejected` | Shows the reason under the grid. | Read the reason your call returned |
| `fw_inventory:close` | Hides the bag. | Let the player close it |
| `fw_inventory:storageClosed` | Hides one pouch when the message includes its id, or every open pouch when it does not. The bag stays up. | `CloseStorage` |
| `fw_weapons:updated` | Redraws the loadout. | `GetKit` |
| `fw_weapons:rejected` | Shows why an equip failed. | Read `fw_weapons:rejected` on the weapons page |

`fw_inventory:storageClosed` is the one a script that draws its own chest panel would listen for:

```lua
RegisterNetEvent('fw_inventory:storageClosed', function(id)
    -- id is one pouch. Nothing means every open pouch closed. The bag screen already hides them.
end)
```

### From the player's game to the server

Sent with `TriggerServerEvent`. The bag screen is the thing that sends these, in response to the player dragging or clicking in the UI. A script building its own panel should call the matching export on [fw_inventory](fw-inventory.md) or [fw_weapons](fw-weapons.md) instead of sending the message directly — the export does the same validation the screen relies on.

| Message | What it does | What you should use |
| --- | --- | --- |
| `fw_inventory:open` | Asks the server for the bag. | The player presses `B` or types `/satchel` |
| `fw_inventory:move` | Moves an item inside the bag. | The player drags it |
| `fw_inventory:storageMove` | Moves an item inside one open pouch. The first value is that pouch's id. | The player drags it |
| `fw_inventory:transfer` | Moves an item between the bag and a pouch, or from one open pouch to another. | The player drags it across |
| `fw_inventory:deposit` | Puts a whole pile into a pouch in the bag. The pouch can stay closed. The pile needs a free square inside, or room on an open pile of the same item. | The player drags the pile onto the pouch picture |
| `fw_inventory:use` | Uses the item. A pouch opens. Anything else runs `RegisterUsable`. | `RegisterUsable`, or the pouch's `container` block |
| `fw_inventory:discard` | Drops part or all of a stack. | `RemoveItem` |
| `fw_inventory:split` | Cuts some off a pile onto an empty square of its own. | The player chooses Split in half, or Split amount |
| `fw_inventory:sort` | Packs the bag. | The player chooses sort |
| `fw_inventory:storageClose` | Closes one pouch when an id is sent, or every open pouch when it is not. The bag stays open. | `CloseStorage` |
| `fw_weapons:equip` | Puts a bag item onto a weapon slot. | See [fw_weapons](fw-weapons.md) |
| `fw_weapons:unequip` | Puts a weapon slot back in the bag. | See [fw_weapons](fw-weapons.md) |
| `fw_weapons:move` | Swaps two weapon slots. | See [fw_weapons](fw-weapons.md) |

A shop that hands the player a rope does this. It does not send `fw_inventory:move`, and it does not open the bag:

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'rope', 1)
if not ok then
    print(err)
    return
end
```

## Clothing while the bag is open

The bag screen also shows a row of worn clothing (hat, coat, gloves, and so on) so the player can toggle a piece on or off without leaving the screen. It reads and changes that through [fw_clothing](fw-clothing.md)'s own exports. There is nothing here for another script to call — if you need to read or change what a character is wearing, use fw_clothing directly.
