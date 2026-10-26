# fw_weapons

Runs on the server and in the player's game. This is the weapon kit: the guns and tools the character has equipped, separate from the bag.

The bag still holds the item. Equipping takes it out of the bag and puts it on a slot. Unequipping puts it back.

The slots are `revolver_1` (right revolver), `revolver_2` (left revolver), `longarm`, `bow`, `lasso`, `knife`, `lantern`, `binoculars`, and `fishingrod` (fishing pole).

A sidearm needs a gunbelt, and so does the knife. The left revolver also needs the offhand holster, and the right revolver has to be filled first. Dragging a gun onto the empty left slot shows `Equip the right revolver first.` Taking the right revolver off while the left one is still there shows `Take the left revolver off first.` `fw_weapons` reads the belt and holster from [fw_clothing](fw-clothing.md). If the gunbelt comes off, the right revolver, the left revolver, and the knife all go back to the bag. If just the offhand holster comes off, only the left revolver goes back. If the bag is full, the player sees `Make room in the satchel first.`

`source` on this page is the player's number for this visit. Nothing is written as `nil`.

## Calls

### GetKit

Runs on the server. A snapshot of the kit for the character this player is playing. You get nothing when this player has not selected a character yet, when the kit is still loading right after they did, or when `fw_weapons` itself has not started yet. Changing the snapshot does not save.

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

Runs in the player's game. The same snapshot, for the local player.

```lua
--- The equipped slots on this player's game.
---@return table kit
exports.fw_weapons:GetKit()
```

Before the server has sent a kit, this can be an empty table with no slot keys at all yet, not a table where every slot is `false`. Check with `type(...) == 'table'` rather than only `~= false`, so you don't try to read a field off nothing:

```lua
local kit = exports.fw_weapons:GetKit()
if type(kit.longarm) == 'table' then
    print(kit.longarm.name)
end
```

### Inspect

Runs in the player's game. Draws the gun in that loadout slot and plays the inspect hold. The bag closes first, so the player can see it. You get no when that slot is empty, when the thing in it is not a gun, or when an inspect is already playing.

A knife, lasso, lantern, binoculars, bow, or fishing pole cannot be inspected this way. Clean is on the same right-click menu, and it does nothing yet. Unload puts the rounds back in the bag. That is described under `fw_weapons:unload`.

```lua
--- Draws the gun in this loadout slot and plays the inspect hold.
--- False when the slot is empty, the weapon is not a gun, or an inspect is already playing.
---@param slotId string
---@return boolean started
exports.fw_weapons:Inspect(slotId)
```

`slotId` is a loadout slot name, such as `revolver_1` or `longarm`. The right-click menu on a filled loadout slot calls this for Inspect.

```lua
local started = exports.fw_weapons:Inspect('revolver_1')
if not started then
    return
end
```

## Messages

A message on the server is heard with `AddEventHandler`. A message from the server to the player's game is heard with `RegisterNetEvent`. A message from the player's game to the server is sent with `TriggerServerEvent`. Your script hears one only if it is already running. Put the listener at the top of your script.

### fw_weapons:updated

Local event in the player's game, raised with `TriggerEvent` and heard with `AddEventHandler`. Sent after the kit on this computer changes, including the first time it arrives from the server. `kit` is a snapshot. Ask `GetKit` when you need a fresh one.

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

Networked event, sent from the server to the player's game with `TriggerClientEvent` and heard there with `RegisterNetEvent`. This is the kit they should show. `fw_weapons` already applies it and then sends `fw_weapons:updated`. Listen here only if you are drawing your own loadout.

```lua
---@param kit table
RegisterNetEvent('fw_weapons:kit', function(kit)
end)
```

### fw_weapons:rejected

Networked event, sent from the server to the player's game with `TriggerClientEvent` and heard there with `RegisterNetEvent`. `reason` is a short line. The satchel shows it. Listen here if your own screen should show that line.

```lua
---@param reason string
RegisterNetEvent('fw_weapons:rejected', function(reason)
    print(reason)
end)
```

This is sent whenever `fw_weapons:wear`, `fw_weapons:equip`, `fw_weapons:unequip`, `fw_weapons:move`, `fw_weapons:load`, or `fw_weapons:unload` (described below) could not be done. The possible lines are:

| Reason | When |
| --- | --- |
| `Select a character first.` | This player is not playing a character. |
| `The kit is still loading.` | A character was just selected and the saved kit has not finished loading yet. Try again shortly. |
| `That item is not in the satchel.` | The placement id passed to `fw_weapons:equip` is not in the bag. |
| `That does not fit there.` | The slot name is not one of the nine listed above, or the item's `kit` does not match the slot it was aimed at. |
| `That slot is empty.` | `fw_weapons:unequip` or `fw_weapons:move` named a slot that has nothing on it. |
| `Put on a gunbelt first.` | `revolver_1`, `revolver_2`, or `knife` was used with no gunbelt worn. |
| `Put on another holster first.` | `revolver_2` was used with no offhand holster worn. |
| `Equip the right revolver first.` | The left slot would end up holding a gun while the right slot stays empty. |
| `Take the left revolver off first.` | The right slot would be cleared while the left slot still holds a gun. |
| `Make room in the satchel first.` | The bag has no space for the item that needs to go back into it. |
| `The kit is busy.` | Another equip, unequip, or move for this player has not finished yet. Try again shortly. |
| `Could not save the kit.` | The database could not store the change. Rare. The action did not happen. |
| `Unload it first.` | The gun already holds a different kind of ammo. |
| `That is already full.` | The gun is at its own round limit. |
| `That is empty.` | Unload was used on a gun with no rounds in it. |
| `You cannot carry any more of those.` | Unload would put the satchel over that ammo's stack. |
| `Those have to use the same rounds.` | The other gun of the same name already holds a different kind. |

### fw_weapons:wear

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. `fw_weapons` sends it itself when the gunbelt or the offhand holster changes. `belt` and `offhand` are yes or no.

Another script should not send this. Change the clothes with [fw_clothing](fw-clothing.md). The kit follows.

```lua
---@param belt boolean
---@param offhand boolean
TriggerServerEvent('fw_weapons:wear', belt, offhand)
```

### fw_weapons:equip

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. The satchel sends this when the player drops a bag item onto a loadout slot.

`placementId` is the `id` from [GetItems](fw-inventory.md). `slotId` is one of the slot names above. The item's `kit` field has to match that slot. A revolver goes on `revolver_1` (the right hip) or `revolver_2` (the left hip). The right slot has to already hold a gun before the left slot will take one. `revolver_1`, `revolver_2`, and `knife` all need a gunbelt worn first, and `revolver_2` also needs the offhand holster. A rifle goes on `longarm`. A fishing pole goes on `fishingrod`. The item name for the shipped pole is `weapon_fishingrod`, and its `kit` is `fishingrod`.

If it fails, the player's game gets `fw_weapons:rejected` with one of the reasons listed above.

```lua
---@param placementId integer
---@param slotId string
TriggerServerEvent('fw_weapons:equip', placementId, slotId)
```

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

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. Puts that slot back into the bag.

`slotId` is the slot name. It has to already hold something, and taking the right revolver off while the left one is still in place is refused with `Take the left revolver off first.` `x` and `y` are the bag square to aim for, with `0, 0` at the top left. `rotated` turns the item if that item can turn. If that square does not fit, the item is placed anywhere that does. If the bag cannot take it, nothing moves, and the reason is `Make room in the satchel first.`

```lua
---@param slotId string
---@param x integer
---@param y integer
---@param rotated boolean
TriggerServerEvent('fw_weapons:unequip', 'knife', 0, 0, false)
```

### fw_weapons:move

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. Swaps two slots.

`fromId` and `toId` both have to be real slots, and each cannot be empty. The item leaving `fromId` has to be allowed on `toId`, and if `toId` already holds something, that item has to be allowed back on `fromId` too — a swap only goes through when both items fit both ways. Moving into `revolver_1`, `revolver_2`, or `knife` needs the same gunbelt (and, for `revolver_2`, offhand holster) as equipping does.

```lua
---@param fromId string
---@param toId string
TriggerServerEvent('fw_weapons:move', 'revolver_1', 'revolver_2')
```

Swaps the two revolvers when both slots already hold a gun. Moving the right revolver onto an empty left slot is refused with `Equip the right revolver first.`

### fw_weapons:load

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. Moves loose rounds from one pile in the bag into the gun in that loadout slot.

`placementId` is the pile's id from the bag. `slotId` is the loadout slot, and that slot has to already hold a gun. The rounds have to match that gun. Revolver rounds go in a revolver. Rifle rounds go in a rifle. The elephant rifle and the varmint rifle each take their own rounds. A bow takes arrows.

The gun only holds so many. A Cattleman holds 6. A Winchester holds 14. A Springfield holds 1. A bow holds 20. Dragging a pile loads up to that number and leaves the rest in the bag. Dragging the same kind again fills whatever room is left. Dragging a different kind, such as Express while Normal is still in the gun, is refused with `Unload it first.`

Two copies of the same gun, such as two Cattlemen, share one ammo count. They have to be loaded with the same rounds. Shots come out of the right-hand gun's count first.

The loaded count is stored on the gun. It stays there when the gun is put back in the bag. `GetKit` shows it on that slot's `metadata`: `ammo` is the item name, `ammoLabel` is the words the player sees, `rounds` is how many are in the gun.

```lua
---@param placementId string
---@param slotId string
TriggerServerEvent('fw_weapons:load', placementId, 'revolver_1')
```

A shop does not send this. A shop gives the pile with `AddItem`. The player loads the gun by dragging that pile onto the loadout slot.

### fw_weapons:unload

Networked event, sent from the player's game to the server with `TriggerServerEvent` and heard there with `RegisterNetEvent`. Takes every round out of that loadout gun and puts them back in the bag, so a different kind can be loaded.

`slotId` is the loadout slot. The right-click menu on a filled loadout slot calls this for Unload. An empty gun answers `That is empty.` If the bag is already holding that ammo's full stack, nothing comes out, and the reason is `You cannot carry any more of those.`

```lua
---@param slotId string
TriggerServerEvent('fw_weapons:unload', 'revolver_1')
```

## State bags

This resource does not set one. Read the kit with `GetKit`.
