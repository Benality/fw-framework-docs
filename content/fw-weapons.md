# fw_weapons

Runs on the server and in the player's game. This is the weapon kit: the guns and tools the character has equipped, separate from the bag.

The bag still holds the item. Equipping takes it out of the bag and puts it on a slot. Unequipping puts it back.

The slots are `revolver_1`, `revolver_2`, `longarm`, `bow`, `lasso`, `knife`, `lantern`, and `binoculars`.

A sidearm needs a gunbelt. The second sidearm needs an offhand holster. `fw_weapons` reads those from [fw_clothing](fw-clothing.md). If the belt comes off, the guns in those slots go back to the bag. If the bag is full, the player sees `Make room in the satchel first.`

`source` on this page is the player's number for this visit. Nothing is written as `nil`.

## Calls

### GetKit

Runs on the server. A snapshot of the kit for the character this player is playing. You get nothing when they have no character selected, or the kit is still loading. Changing the snapshot does not save.

```lua
--- The equipped slots, or nothing when this player has no kit yet.
---@param source integer
---@return table|nil kit
exports.fw_weapons:GetKit(source)
```

Each slot is `false` when it is empty. When it holds something, it has `name`, `count`, `metadata`, `hash`, `kit`, and `rotated`. `name` is the item name. `kit` is the kind of slot it belongs on, such as `revolver` or `melee`.

A sheriff script can refuse to start a duel when the player has no sidearm:

```lua
local kit = exports.fw_weapons:GetKit(source)
if kit == nil or kit.revolver_1 == false then
    return
end
print(kit.revolver_1.name)
```

### GetKit

Runs in the player's game. The same snapshot, for the local player. It can be empty until the server has sent the kit.

```lua
--- The equipped slots on this player's game.
---@return table kit
exports.fw_weapons:GetKit()
```

```lua
local kit = exports.fw_weapons:GetKit()
if kit.longarm ~= false then
    print(kit.longarm.name)
end
```

## Messages

A message on the server is heard with `AddEventHandler`. A message from the server to the player's game is heard with `RegisterNetEvent`. A message from the player's game to the server is sent with `TriggerServerEvent`. Your script hears one only if it is already running. Put the listener at the top of your script.

### fw_weapons:updated

Stays in the player's game. Sent after the kit on this computer changes. `kit` is a snapshot. Ask `GetKit` when you need a fresh one.

A holster script can watch for a rifle being put on the back:

```lua
---@param kit table
AddEventHandler('fw_weapons:updated', function(kit)
    if kit.longarm ~= false then
        print(kit.longarm.name)
    end
end)
```

### fw_weapons:kit

Sent from the server to the player's game. This is the kit they should show. `fw_weapons` already applies it and then sends `fw_weapons:updated`. Listen here only if you are drawing your own loadout.

```lua
---@param kit table
RegisterNetEvent('fw_weapons:kit', function(kit)
end)
```

### fw_weapons:rejected

Sent from the server to the player's game. `reason` is a short line, such as `That does not fit there.` or `Put on a gunbelt first.` The satchel shows it. Listen here if your own screen should show that line.

```lua
---@param reason string
RegisterNetEvent('fw_weapons:rejected', function(reason)
    print(reason)
end)
```

### fw_weapons:wear

Sent from the player's game to the server. `fw_weapons` sends it when the gunbelt or the offhand holster changes. `belt` and `offhand` are yes or no.

Another script should not send this. Change the clothes with [fw_clothing](fw-clothing.md). The kit follows.

```lua
---@param belt boolean
---@param offhand boolean
TriggerServerEvent('fw_weapons:wear', belt, offhand)
```

### fw_weapons:equip

Sent from the player's game to the server. The satchel sends this when the player drops a bag item onto a loadout slot.

`placementId` is the `id` from [GetItems](fw-inventory.md). `slotId` is one of the slot names above. The item's `kit` field has to match that slot. A revolver goes on `revolver_1` or `revolver_2`. A rifle goes on `longarm`.

If it fails, the player's game gets `fw_weapons:rejected`. Common reasons:

| Reason | When |
| --- | --- |
| `Select a character first.` | This player is not playing a character. |
| `That item is not in the satchel.` | That placement id is not in the bag. |
| `That does not fit there.` | That item's kit does not match the slot. |
| `Put on a gunbelt first.` | A sidearm slot was used with no gunbelt. |
| `Put on another holster first.` | `revolver_2` was used with no offhand holster. |
| `The kit is busy.` | Another equip is still finishing. |

A camp script can put the cattleman already in the bag onto the hip:

```lua
local items = exports.fw_inventory:GetItems(source)
for index = 1, #items do
    if items[index].name == 'weapon_revolver_cattleman' then
        TriggerClientEvent('my_camp:equip', source, items[index].id)
        break
    end
end
```

In the player's game, after that message:

```lua
RegisterNetEvent('my_camp:equip', function(placementId)
    TriggerServerEvent('fw_weapons:equip', placementId, 'revolver_1')
end)
```

Giving a weapon the player does not already hold is `AddItem` on [fw_inventory](fw-inventory.md). They equip it from the loadout after that.

### fw_weapons:unequip

Sent from the player's game to the server. Puts that slot back into the bag.

`slotId` is the slot name. `x` and `y` are the bag square to aim for, with `0, 0` at the top left. `rotated` turns the item if that item can turn. If that square does not fit, the item is placed anywhere that does. If the bag cannot take it, nothing moves, and the reason is `Make room in the satchel first.`

```lua
TriggerServerEvent('fw_weapons:unequip', 'knife', 0, 0, false)
```

### fw_weapons:move

Sent from the player's game to the server. Swaps two slots. Both names have to be real slots, and each item has to be allowed on the slot it is moving to.

```lua
TriggerServerEvent('fw_weapons:move', 'revolver_1', 'revolver_2')
```

## State bags

This resource does not set one. Read the kit with `GetKit`.
