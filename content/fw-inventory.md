# fw_inventory

Runs on the server. Give a player an item, take one away, add a new kind of item, or open a second grid such as a chest or a pouch.

`fw_satchel` is the screen the player sees. It draws their bag and, when one is open, the chest or pouch beside it. Your script does not open that screen itself, except with `OpenStorage`, which opens both.

Ask `IsReady` first. If you also listen for `fw_inventory:ready` and your script starts later, you missed that message. Ask `IsReady` instead.

## Calls

### IsReady

Runs on the server. Yes after the item list has loaded.

```lua
--- True when inventory calls can be used.
---@return boolean ready
exports.fw_inventory:IsReady()
```

```lua
if not exports.fw_inventory:IsReady() then
    return
end
```

### RegisterItem

Runs on the server. Adds an item type. The same fields as an item file, plus `name`. Connected players get the new definition without a restart.

```lua
--- Adds or replaces an item type.
--- False and a reason when the name or a field is rejected, or the row cannot be saved.
---@param def table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:RegisterItem(def)
```

```lua
local ok, err = exports.fw_inventory:RegisterItem({
    name = 'bread',
    label = 'Bread',
    group = 'food',
    width = 1,
    height = 1,
    stack = 10,
    description = 'A loaf of bread.',
})
if not ok then
    print(err)
    return
end
```

A later reload from `items/` updates a name that also has a file. A name that exists only from this call is left in place.

### RegisterUsable

Runs on the server. Runs your function when the player uses that item from the satchel. The item is not removed unless you call `RemoveItem`.

```lua
--- Registers a use callback for an item that already exists.
--- False and a reason when the item is missing or the callback is not a function.
---@param name string
---@param callback fun(source: integer, item: table)
---@return boolean ok
---@return string|nil err
exports.fw_inventory:RegisterUsable(name, callback)
```

`item` is a copy: `id`, `name`, `label`, `description`, `count`, `x`, `y`, `rotated`, `metadata`. Changing it does not change the satchel.

If your resource stops, its use callbacks are removed. Start it again and register them again.

```lua
exports.fw_inventory:RegisterUsable('bread', function(source, item)
    local ok, err = exports.fw_inventory:RemoveItem(source, 'bread', 1)
    if not ok then
        print(err)
    end
end)
```

### GetDefinition

Runs on the server. A snapshot of one kind of item, or nothing when that name is not in the list. Changing what you got back does not change the list.

```lua
--- One item type, or nil when the name is unknown.
---@param name string
---@return table|nil definition
exports.fw_inventory:GetDefinition(name)
```

The copy can include `name`, `label`, `description`, `width`, `height`, `stack`, `weapon`, `group`, `rotate`, `kit`, `hash`, and `container`.

```lua
local def = exports.fw_inventory:GetDefinition('herb_pouch')
if def == nil then
    return
end
```

### AddItem

Runs on the server. Puts items into the satchel of the character this player is playing. It finds room first. If the whole count does not fit, nothing is added.

```lua
--- Adds items to the active character's satchel.
--- False and a reason when the item is unknown, the player has no character, or it does not fit.
---@param source integer
---@param name string
---@param count integer|nil
---@param metadata table|nil
---@return boolean ok
---@return string|nil err
exports.fw_inventory:AddItem(source, name, count, metadata)
```

`count` defaults to 1. The maximum for one call is 500. Stacks that already match the name and metadata are filled first. A weapon never stacks on another weapon. A container item receives its own `metadata.containerId` on each new placement.

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'rope', 1)
if not ok then
    print(err)
    return
end
```

### RemoveItem

Runs on the server. Takes that many of the name out of the satchel. If the player does not hold that many, nothing is removed. Removing a whole container placement deletes the storage that belongs to it.

```lua
--- Removes items from the active character's satchel.
--- False and a reason when the player does not hold that many.
---@param source integer
---@param name string
---@param count integer
---@return boolean ok
---@return string|nil err
exports.fw_inventory:RemoveItem(source, name, count)
```

```lua
local ok, err = exports.fw_inventory:RemoveItem(source, 'bread', 2)
if not ok then
    print(err)
    return
end
```

### GetItems

Runs on the server. A snapshot of what is in the bag. You get nothing, and a reason, when the player has no character selected.

```lua
--- The placements in the active character's satchel.
---@param source integer
---@return table|nil items
---@return string|nil err
exports.fw_inventory:GetItems(source)
```

Each placement has `id`, `name`, `x`, `y`, `rotated`, `count`, and `metadata`. `x` and `y` are the top-left cell. `0, 0` is the top left.

```lua
local items, err = exports.fw_inventory:GetItems(source)
if items == nil then
    print(err)
    return
end
```

### CanFit

Runs on the server. True when `AddItem` would succeed for that name and count. It does not change the satchel.

```lua
--- True when that many of the item would fit.
---@param source integer
---@param name string
---@param count integer|nil
---@param metadata table|nil
---@return boolean ok
---@return string|nil err
exports.fw_inventory:CanFit(source, name, count, metadata)
```

```lua
local ok, err = exports.fw_inventory:CanFit(source, 'lantern', 1)
if not ok then
    print(err)
    return
end
```

### PlaceAt

Runs on the server. Puts one new placement on an exact cell. It does not look for another spot.

```lua
--- Places one stack on an exact cell.
--- False and a reason when the cell is taken, outside the grid, or the item cannot turn.
---@param source integer
---@param name string
---@param count integer|nil
---@param metadata table|nil
---@param x integer
---@param y integer
---@param rotated boolean|nil
---@return boolean ok
---@return string|nil idOrErr
exports.fw_inventory:PlaceAt(source, name, count, metadata, x, y, rotated)
```

`count` cannot be higher than that item's stack. On success the second value is the new placement id. `That does not fit.` means the rectangle is outside the grid or overlaps something else.

```lua
local ok, idOrErr = exports.fw_inventory:PlaceAt(source, 'rope', 1, nil, 0, 0, false)
if not ok then
    print(idOrErr)
    return
end
```

### SetGridBonus

Runs on the server. Adds columns or rows to the satchel under a name you choose, such as `backpack`. The same name replaces that bonus. It does not stack on itself. Two different names both apply. The grid cannot pass 24 by 24.

```lua
--- Adds or replaces a named size bonus on the satchel.
--- False and a reason when shrinking would cut off an item.
---@param source integer
---@param key string
---@param extraWidth integer
---@param extraHeight integer
---@return boolean ok
---@return string|nil err
exports.fw_inventory:SetGridBonus(source, key, extraWidth, extraHeight)
```

`key` is lowercase letters, digits, and underscores, up to 32 characters. `0, 0` removes that bonus.

Shrinking is refused while an item sits in a cell that would disappear. The reason is `Make room in the satchel first.` Nothing is moved.

```lua
local ok, err = exports.fw_inventory:SetGridBonus(source, 'backpack', 0, 5)
if not ok then
    print(err)
    return
end
```

### ClearGridBonus

Runs on the server. Removes one named bonus. Same refusal as shrinking with `SetGridBonus`.

```lua
--- Removes a named size bonus.
---@param source integer
---@param key string
---@return boolean ok
---@return string|nil err
exports.fw_inventory:ClearGridBonus(source, key)
```

```lua
local ok, err = exports.fw_inventory:ClearGridBonus(source, 'backpack')
if not ok then
    print(err)
end
```

### OpenStorage

Runs on the server. Opens a second grid beside the satchel. The first call creates it. A later call can make it larger. The player must be playing a character.

```lua
--- Opens a storage grid for this player.
--- False and a reason when the id, size, or groups are rejected, or someone else has it open.
---@param source integer
---@param id string
---@param params table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:OpenStorage(source, id, params)
```

`id` is 1–64 characters. It starts with a letter or digit. After that, letters, digits, underscore, colon, and hyphen are allowed. A chest uses a stable id such as `chest:valentine_1`. A pouch uses `item:` plus the placement's `metadata.containerId`.

`params.width` and `params.height` are the grid, from 1 to 24. `params.label` is the title. Leave it out and the title is `Storage`. `params.groups` is a list of item groups. Leave it out and the storage accepts anything. An empty list accepts nothing. An unknown group returns `unknown group`.

| `err` | When |
| --- | --- |
| `Invalid storage.` | The id or the parameter table is not usable. |
| `Invalid storage size.` | Width or height is outside 1–24. |
| `unknown group` | A group name is not in the item catalog. |
| `Make room in there first.` | A later open tried to shrink the grid while an item sits in a cell that would disappear. |
| `Someone is using that.` | Another player already has this storage open. |

One player holds a storage at a time. Opening a different one closes the one this player already has. Dropping, clearing the character, stopping the resource, and closing the satchel all release it.

Dragging an item into a storage that lists groups refuses anything outside that list. The player sees `That does not go in there.` The satchel itself accepts every group. The pouch that owns the open storage cannot be placed into it.

```lua
local ok, err = exports.fw_inventory:OpenStorage(source, 'chest:valentine_1', {
    label = 'Chest',
    width = 8,
    height = 6,
    groups = { 'materials', 'tools' },
})
if not ok then
    print(err)
    return
end
```

### CloseStorage

Runs on the server. Closes the second grid and leaves the satchel open. True when this player had one open. False when they did not.

```lua
--- Closes the storage this player has open.
---@param source integer
---@return boolean closed
exports.fw_inventory:CloseStorage(source)
```

```lua
exports.fw_inventory:CloseStorage(source)
```

### PullForKit

Runs on the server. `fw_weapons` uses this when a player equips a weapon. Other scripts should use `RemoveItem`.

```lua
--- Takes one placement out of the satchel for a weapon slot.
--- Nil and a reason when the placement is missing or the item coming off the slot does not fit back.
---@param source integer
---@param id string
---@param displaced table|nil
---@return table|nil item
---@return string|nil err
exports.fw_inventory:PullForKit(source, id, displaced)
```

`id` is the placement id from `GetItems`. `displaced` is the item leaving that weapon slot, with `name`, `count`, `metadata`, and `rotated`. It is put back on the cell that was freed. If that cell does not fit, it is placed anywhere that does. If neither works, nothing is taken and the reason is `Make room in the satchel first.`

## Messages

`fw_inventory:ready` stays on the server. It is sent once, after the item list has loaded. Your script hears it only if it is already running. If your script starts later, you missed it. Ask `IsReady`.

```lua
AddEventHandler('fw_inventory:ready', function()
end)
```

`fw_characters:selected` loads or creates that character's bag. `fw_characters:cleared` saves it and forgets it. Both are messages on the server, described on [fw_characters](fw-characters.md). Your script hears them only if it is already running.

The satchel screen talks to the server with its own messages (`fw_inventory:open`, `move`, `use`, `discard`, `split`, `sort`, `storageClose`, `storageMove`, `transfer`). Do not send those from your script. Use the calls on this page.

`fw_inventory:storageClosed` is sent from the server to the player's game. It tells the satchel screen to hide the chest or pouch. Listen for it only if you are drawing that screen yourself.

## Item file

Put a Lua file in `resources/[framework]/fw_inventory/items/`. The file returns a map. The key is the item name: lowercase letters, digits, and underscores, up to 32 characters.

```lua
return {
    bread = {
        label = 'Bread',
        group = 'food',
        width = 1,
        height = 1,
        stack = 10,
        description = 'A loaf of bread.',
    },
}
```

| Field | Default | Meaning |
| --- | --- | --- |
| `label` | The name | Text in the satchel. Clipped to 48 characters. |
| `description` | empty | Line in the satchel. Clipped to 200 characters. |
| `width`, `height` | Required, 1–12 | How many cells the item covers before it is turned. |
| `group` | Required | `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `weapons`, or `materials`. |
| `stack` | 1 | How many copies share one footprint. Maximum 1000. |
| `weapon` | false | When true, the item never stacks. |
| `rotate` | false | The player can turn it. |
| `kit` | none | Which weapon slot can hold it: `revolver`, `pistol`, `longarm`, `bow`, `lasso`, `melee`, or `lantern`. |
| `hash` | none | Weapon name, such as `WEAPON_REVOLVER_CATTLEMAN`. Required when `kit` is set. |
| `container` | none | Opens a second grid when the item is used from the satchel. Stack is forced to 1. |

The picture is `fw_inventory/images/<name>.png`. A new picture is not served until `ensure fw_inventory`.

`fw_items_reload` reads `items/` again. The server console can run it. An admin in the game can run `/fw_items_reload`.

`fw_giveitem <serverId> <name> [count]` is a server console command. It calls `AddItem`.

## Container items

A pouch is an item that owns a storage row. The first time that placement is created it gets `metadata.containerId`. Moving the pouch keeps that id, so the contents stay with it. Dropping it, or removing the whole placement, deletes the row `item:<containerId>`.

```lua
herb_pouch = {
    label = 'Herb Pouch',
    group = 'tools',
    width = 2,
    height = 2,
    stack = 1,
    description = 'A pouch for herbs.',
    container = {
        label = 'Herb pouch',
        width = 4,
        height = 4,
        groups = { 'herbs' },
    },
}
```

`container.label` is the title of the second grid. `width` and `height` are 1–24. `groups` is optional. Leave it out and the pouch accepts anything.

Using the pouch in the satchel opens that storage. Using it while it is sitting inside the open storage is refused with `Take it out first.`

`container` is not a database column. It lives in the item file. A reload copies it back onto the catalog.

## Satchel screen

`fw_satchel` opens with B or `/satchel`. Escape or Tab closes it and releases any open storage.

Drag a placement. The cells under it turn green when it fits and red when it does not. R turns it when `rotate` is true. Dropping a stack on the same item, with the same metadata and room left in `stack`, combines them. A weapon never combines.

Right-click a satchel placement for Use, Drop, or Split. Double-click uses it. If nothing registered a use callback, the line under the grid says it cannot be used.

A storage open adds a second panel to the left of the satchel, only as large as that storage's grid. The satchel and the weapon loadout stay where they are. When the loadout is open, the storage panel sits to the left of it. Close on that panel hides it and leaves the satchel open. Double-click on the storage grid is refused until the item is back in the satchel.
