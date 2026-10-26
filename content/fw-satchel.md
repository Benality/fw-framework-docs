# fw_satchel

Runs in the player's game. This is the bag screen. It draws the grid from [fw_inventory](fw-inventory.md) and the loadout from [fw_weapons](fw-weapons.md).

It has no calls of its own, and it does not set a state bag. Give an item, open a chest, or define an item with the calls on the inventory page. Read the equipped weapons with `GetKit` on the weapons page.

The player opens the bag with B or `/satchel`. Escape or Tab closes it. Closing the bag also closes an open chest or pouch.

`fwInventoryBusy` on [fw_menu](fw-menu.md) is how another menu asks this screen to stay closed.

## Messages the screen uses

These go between the bag screen and the server. A new script should use the call named in the last column. Sending the screen's own message skips the checks that call already does, or fights the screen for the same drag.

Your script hears a message only if it is already running.

### From the server to the player's game

Heard with `RegisterNetEvent`.

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

Sent with `TriggerServerEvent`. The bag screen is the thing that sends these.

| Message | What it does | What you should use |
| --- | --- | --- |
| `fw_inventory:open` | Asks the server for the bag. | The player presses B |
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

A shop that hands the player a rope does this. It does not send `fw_inventory:move`:

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'rope', 1)
if not ok then
    print(err)
    return
end
```
