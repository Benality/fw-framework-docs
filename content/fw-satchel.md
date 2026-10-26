# fw_satchel

Runs on the player's game. There is no code on the server. This is the bag screen. It draws the grid from [fw_inventory](fw-inventory.md) and the loadout from [fw_weapons](fw-weapons.md).

It has no exports, no events of its own, and no state bag. Give an item, open a chest, or define an item with the calls on the inventory page. Read the equipped weapons with `GetKit` on the weapons page. This page exists so you know which messages the screen is already listening for, and which call to use instead of sending your own.

The player opens the bag with `B`, or by typing `/satchel`. Escape or Tab closes it. Closing the bag also closes an open chest or pouch.

Right-click a filled loadout slot for Inspect, Clean, and Unload. That menu is only on the loadout, not on an item sitting in the bag. Inspect draws the gun and plays the inspect hold, then the bag closes. A knife, lasso, lantern, binoculars, or bow cannot be inspected. Clean and Unload are listed, and they do nothing yet. Another script plays the same inspect with `Inspect` on [fw_weapons](fw-weapons.md).

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
| `fw_inventory:storageClosed` | Hides the chest or pouch and leaves the bag up. | `CloseStorage` |
| `fw_weapons:updated` | Redraws the loadout. | `GetKit` |
| `fw_weapons:rejected` | Shows why an equip failed. | Read `fw_weapons:rejected` on the weapons page |

`fw_inventory:storageClosed` is the one a script that draws its own chest panel would listen for:

```lua
RegisterNetEvent('fw_inventory:storageClosed', function()
    -- Hide your chest panel. The bag screen already does this.
end)
```

### From the player's game to the server

Sent with `TriggerServerEvent`. The bag screen is the thing that sends these, in response to the player dragging or clicking in the UI. A script building its own panel should call the matching export on [fw_inventory](fw-inventory.md) or [fw_weapons](fw-weapons.md) instead of sending the message directly — the export does the same validation the screen relies on.

| Message | What it does | What you should use |
| --- | --- | --- |
| `fw_inventory:open` | Asks the server for the bag. | The player presses `B` or types `/satchel` |
| `fw_inventory:move` | Moves an item inside the bag. | The player drags it |
| `fw_inventory:storageMove` | Moves an item inside the open chest or pouch. | The player drags it |
| `fw_inventory:transfer` | Moves an item from the bag to the chest, or the other way. | The player drags it across |
| `fw_inventory:use` | Uses the item. A pouch opens. Anything else runs `RegisterUsable`. | `RegisterUsable`, or the pouch's `container` block |
| `fw_inventory:discard` | Drops part or all of a stack. | `RemoveItem` |
| `fw_inventory:split` | Splits a stack into two squares. | The player chooses Split |
| `fw_inventory:sort` | Packs the bag. | The player chooses sort |
| `fw_inventory:storageClose` | Closes the chest and leaves the bag open. | `CloseStorage` |
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
