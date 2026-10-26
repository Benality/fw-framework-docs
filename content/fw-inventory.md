# fw_inventory

Runs on the server. Give a player an item, take one away, add a new kind of item, or open a second grid such as a chest or a pouch.

`fw_satchel` is the screen the player sees. It draws their bag and, when one is open, the chest or pouch beside it. Your script does not open that screen itself, except with `OpenStorage`, which opens both.

Ask `IsReady` first. If you also listen for `fw_inventory:ready` and your script starts later, you missed that message. Ask `IsReady` instead.

A second grid is the same kind of squares as the player's bag. There are two ways to open one. A chest, wagon, or horse is a place your script opens by name. A pouch is an item the player uses. Those are different jobs. Both are below.

## A chest, a wagon, or a horse

You do not register a chest. You pick a name for it, and you call `OpenStorage` with that same name every time the player should see it. The first open creates it, empty, at the size you pass. Later opens with that same name show the same items.

`source` is the player's number for this visit.

```lua
local function openValentineChest(source)
    local ok, err = exports.fw_inventory:OpenStorage(source, 'chest:valentine_1', {
        label = 'Chest',
        width = 8,
        height = 6,
    })
    if not ok then
        print(err)
        return
    end
end
```

Call that from your own script at the moment the player opens the chest. This page does not watch the world for you. A wagon and a horse are the same call, with their own names, such as `wagon:1` or `horse:12`.

- The name is how the chest is remembered. `chest:valentine_1` is one chest. `chest:valentine_2` is a different chest. Use the same name every time you want the same items.
- Use lowercase letters, digits, underscore, colon, and hyphen. Start with a letter or a digit. No spaces. Keep it to 64 characters or fewer. `Chest 1` is rejected with `Invalid storage.`
- `chest:`, `wagon:`, and `horse:` are not enforced by this resource — nothing stops you from naming it something else — but using them keeps your own script's names easy to tell apart. Do not start a name with `item:`. That prefix is used internally for the storage inside a pouch, and a name you pick by hand could collide with one.
- `width` and `height` are how many squares, from 1 to 24. The example is 8 across and 6 down.
- `label` is the title the player sees. Leave it out and the title is `Storage`.
- The bag opens at the same time, with the chest beside it. The player drags items between them.
- The player must already be playing a character. Otherwise you get `Select a character first.`

One person can have a chest open at a time. The next person gets `Someone is using that.` One player can have one of these open. Opening a second one closes the first.

`CloseStorage` hides the chest and leaves the bag open. Closing the bag on the player's screen also closes the chest — that happens on the player's screen, not on this page.

To make a chest larger later, call `OpenStorage` again with the same name and a bigger width or height. The items stay. Making it smaller, while an item sits on a square that would disappear, comes back as `Make room in there first.` Nothing is moved.

### Which items can go in

Every item has a group. The groups are `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `weapons`, and `materials`.

- Leave `groups` out, and the chest takes any item.
- Pass a list, and only those groups can be dragged in. Herbs and tools: `groups = { 'herbs', 'tools' }`.
- An empty list, `groups = {}`, takes nothing.
- A name that is not in the list above comes back as `unknown group`.
- Something other than a list of strings, such as a number or a single string, comes back as `Invalid storage groups.`

When the player drags the wrong kind of item, they see `That does not go in there.` The bag itself takes every group.

```lua
exports.fw_inventory:OpenStorage(source, 'wagon:1', {
    label = 'Wagon',
    width = 10,
    height = 8,
    groups = { 'materials', 'tools' },
})
```

## A pouch, or any item that holds other items

A pouch is an item in the bag. When the player uses it, a second grid opens, the same kind of grid as a chest. You do not call `OpenStorage` for a pouch. The item does that.

You add the pouch the same way you add any other item: a file in `items/`, or `RegisterItem`. The extra part is `container`.

```lua
local ok, err = exports.fw_inventory:RegisterItem({
    name = 'seed_pouch',
    label = 'Seed Pouch',
    group = 'tools',
    width = 2,
    height = 2,
    stack = 1,
    description = 'A small pouch for seeds.',
    container = {
        label = 'Seed pouch',
        width = 4,
        height = 4,
        groups = { 'herbs' },
    },
})
if not ok then
    print(err)
    return
end
```

The same fields work in an item file. The name is the key, so you leave `name` out of the file. The [item file](#item-file) section shows that shape.

`items/default.lua` already ships with a pouch named `herb_pouch`, 2 by 2 in the bag with a 6-by-10 grid inside that only takes herbs. You can give that one to a player without registering anything, the same way you'd give any other item that ships in that folder.

What you pass:

- `width` and `height` on the item are how many squares the pouch takes in the bag. Here that is 2 by 2.
- `container.width` and `container.height` are how many squares are inside the pouch, from 1 to 24. Here that is 4 by 4.
- `container.label` is the title of that second grid.
- `container.groups` follows the same rules as a chest. Leave it out and the pouch takes anything. `{ 'herbs' }` takes herbs only.
- A pouch always holds a count of 1. Two pouches cannot share one square, because each one has its own contents.

A picture named `seed_pouch.png` goes in `fw_inventory/images/`. A new picture shows up after `ensure fw_inventory`.

Give it to the player with `AddItem`:

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'herb_pouch', 1)
if not ok then
    print(err)
    return
end
```

Each pouch gets its own contents the first time it is used. Two pouches are two different storages. Moving a pouch around the bag keeps the same contents.

The player opens the bag with B or `/satchel`, then double-clicks the pouch, or right-clicks it and chooses Use. The pouch grid appears beside the bag. They drag items in. Close on that panel hides the pouch and leaves the bag open.

A use function from `RegisterUsable` is not called for a pouch. Using it opens the pouch instead.

Using a pouch while it is sitting inside another open chest or pouch is refused. The player sees `Take it out first.` Put it back in the bag, then use it.

Dropping the pouch, or removing the whole pouch with `RemoveItem`, deletes what was inside it. Moving it does not.

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

Runs on the server. Adds an item type, or replaces one that already exists. The same fields as an item file, plus `name`. Connected players get the new definition without a restart.

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
    name = 'dried_meat',
    label = 'Dried Meat',
    group = 'food',
    width = 1,
    height = 1,
    stack = 20,
    description = 'Smoked and dried. Keeps for a long time.',
})
if not ok then
    print(err)
    return
end
```

A later reload from `items/` updates a name that also has a file. A name that exists only from this call is left in place, because it was already written to the database the first time you registered it.

If the name you register, or the `hash` you give it, looks like a known RDR3 weapon such as `WEAPON_REVOLVER_SCHOFIELD`, you can leave `kit`, `hash`, `width`, `height`, and `rotate` out. They are filled in for you.

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
exports.fw_inventory:RegisterUsable('dried_meat', function(source, item)
    local ok, err = exports.fw_inventory:RemoveItem(source, 'dried_meat', 1)
    if not ok then
        print(err)
    end
end)
```

A pouch, or any item with a `container`, never reaches this callback. Using it opens the pouch instead, as described in [A pouch](#a-pouch-or-any-item-that-holds-other-items).

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

If `name` is not a registered item, but it looks like a known RDR3 weapon, such as `weapon_revolver_cattleman`, this resource creates that item type for you on the spot and then gives it. Any other unknown name fails with `Unknown item.` — register it first with `RegisterItem` or a file in `items/`.

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
local ok, err = exports.fw_inventory:RemoveItem(source, 'dried_meat', 2)
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

Runs on the server. Puts one new placement on an exact cell. It does not look for another spot, and it does not combine with a stack that is already sitting there.

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
--- False and a reason when the key or sizes are invalid, the total would pass 24, or shrinking would cut off an item.
---@param source integer
---@param key string
---@param extraWidth integer
---@param extraHeight integer
---@return boolean ok
---@return string|nil err
exports.fw_inventory:SetGridBonus(source, key, extraWidth, extraHeight)
```

`key` is lowercase letters, digits, and underscores, up to 32 characters. `0, 0` removes that bonus.

| `err` | When |
| --- | --- |
| `Invalid bonus.` | `key` has characters outside lowercase letters, digits, and underscore, is longer than 32 characters, or `extraWidth`/`extraHeight` is not a whole number from 0 to 24. |
| `The satchel cannot grow any further.` | Adding this bonus on top of the base size and every other named bonus would ask for more than 24 columns or 24 rows, even though the satchel itself is capped at 24. Use a smaller bonus. |
| `Make room in the satchel first.` | Shrinking or removing a bonus would leave an item sitting partly off the grid. Nothing is moved. |

```lua
local ok, err = exports.fw_inventory:SetGridBonus(source, 'backpack', 0, 5)
if not ok then
    print(err)
    return
end
```

### ClearGridBonus

Runs on the server. Removes one named bonus. The same `Invalid bonus.` and `Make room in the satchel first.` reasons from `SetGridBonus` apply here.

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

Runs on the server. Opens a chest, wagon, or horse, as described in [A chest, a wagon, or a horse](#a-chest-a-wagon-or-a-horse). The first call creates it. A later call with the same name opens the same items, and can make the grid larger. The player must be playing a character.

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

`id` is the name from the chest section. `params.width` and `params.height` are the squares, from 1 to 24. `params.label` is the title. `params.groups` is the list of groups, or leave it out.

| `err` | When |
| --- | --- |
| `fw_inventory is not ready` | The item list has not loaded yet. Ask `IsReady` first. |
| `Select a character first.` | This player is not playing a character. |
| `player is not loaded` | `source` is not a player who is in the server. |
| `Invalid storage.` | The name has a space, a capital letter, or some other character that is not allowed. Or `params` was left out. |
| `Invalid storage size.` | Width or height is outside 1–24. |
| `unknown group` | A group name is not in the list in the chest section. |
| `Invalid storage groups.` | `params.groups` was something other than a list of group names. |
| `Make room in there first.` | A later open tried to make the grid smaller while an item sits on a square that would disappear. |
| `Someone is using that.` | Another player already has this one open. |

The pouch that owns the open storage cannot be placed into it.

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

A message on the server is heard with `AddEventHandler`. A message from the server to the player's game is heard with `RegisterNetEvent`. A message from the player's game to the server is sent with `TriggerServerEvent`. Your script hears one only if it is already running. Put the listener at the top of your script.

### fw_inventory:ready

Local, on the server. Sent once, after the item list has loaded. If your script starts later, you missed it. Ask `IsReady`.

```lua
AddEventHandler('fw_inventory:ready', function()
    exports.fw_inventory:RegisterUsable('dried_meat', function(source, item)
        exports.fw_inventory:RemoveItem(source, 'dried_meat', 1)
    end)
end)
```

`fw_characters:selected` loads or creates that character's bag. `fw_characters:cleared` saves it and forgets it. Both are messages on the server, described on [fw_characters](fw-characters.md).

### What the bag screen sends the server

These are networked (`RegisterNetEvent` on the server, sent from the player's game with `TriggerServerEvent`). `fw_satchel` sends these when the player drags, uses, sorts, or drops. A script with its own bag screen could send them too, but for giving or taking items from your own script, use the calls above instead — `AddItem`, `RemoveItem`, and so on, not these.

`token` is a value you make up and send with the request. It comes back unchanged on `fw_inventory:updated` or `fw_inventory:rejected`, so a screen that fires several requests at once can tell which response answers which request. Leave it out, or pass anything, if you don't need that.

| Event | Parameters | What it does |
| --- | --- | --- |
| `fw_inventory:open` | none | Opens the bag for the sender. Answered with `fw_inventory:opened`, or `fw_inventory:rejected` when the item list is not ready or the player has no character. |
| `fw_inventory:move` | `id, x, y, rotated, token` | Moves a placement already in the bag to a new cell, or turns it. `id` is the placement id. `rotated` is `true`, `false`, or left out to keep its current facing. |
| `fw_inventory:sort` | `token` | Rearranges the whole bag by item group, then size. |
| `fw_inventory:discard` | `id, count, token` | Drops up to `count` of that placement from the bag. `count` defaults to 1. Dropping the last one removes the placement, and the container it owns, if it had one. |
| `fw_inventory:split` | `id, count, token` | Splits `count` off a stackable placement into a new stack. `count` must be less than what's already there, and the item must be able to stack. |
| `fw_inventory:use` | `id, token, side` | Uses that placement. `side` is `'storage'` when the placement is sitting in an open chest or pouch rather than the bag — that is refused with `Take it out first.` Opens a pouch's storage, or runs its `RegisterUsable` callback, or rejects with `This cannot be used.` |
| `fw_inventory:storageClose` | none | Closes whatever chest or pouch the sender has open, if any. Answered with `fw_inventory:storageClosed`. |
| `fw_inventory:storageMove` | `id, x, y, rotated, token` | Moves a placement inside the open chest or pouch, not the bag. |
| `fw_inventory:transfer` | `fromSide, id, x, y, rotated, token` | Moves a placement between the bag and the open chest or pouch. `fromSide` is `'satchel'` or `'storage'` — which side the item is coming from. `x, y, rotated` describe where it lands on the other side. |

```lua
RegisterNetEvent('fw_inventory:use', function(id, token, side)
    -- fw_satchel already does this. Shown so a custom screen knows the shape.
end)
```

### What the server sends the bag screen

These are networked (`RegisterNetEvent` on the player's game, sent from the server with `TriggerClientEvent`). The screen in [fw_satchel](fw-satchel.md) listens for these. A script with its own bag screen can listen too. Giving an item from your own script is still `AddItem`, not one of these.

| Event | Parameters | What it carries |
| --- | --- | --- |
| `fw_inventory:opened` | `payload` | `payload.grid` is the bag. `payload.definitions` and `payload.groups` are every item type and the group names. `payload.storage` is the chest or pouch, when one was opened at the same time, or nothing. |
| `fw_inventory:updated` | `grid, token, storage` | The bag after a change, the `token` from the request that caused it (or nothing, when this page's own calls caused the change), and the open chest or pouch, when there is one. |
| `fw_inventory:definitions` | `definitions, groups` | Sent to everyone. Every item type and the group names, after `RegisterItem`, a reload, or a new weapon type being created automatically. |
| `fw_inventory:rejected` | `reason, token` | A short reason, such as `That does not go in there.`, and the `token` from the request that failed. |
| `fw_inventory:close` | none | Hide the bag. Sent when the character is deselected, the player leaves, or this resource stops. |
| `fw_inventory:storageClosed` | none | Hide the chest or pouch. The bag stays. |

```lua
RegisterNetEvent('fw_inventory:rejected', function(reason, token)
    print(reason)
end)
```

## Console commands

| Command | Where | What it does |
| --- | --- | --- |
| `fw_items_reload` | Server console, or `/fw_items_reload` by an in-game admin | Reads `items/` again and re-saves every item. A player who is not an admin and types `/fw_items_reload` gets `You are not an admin.` in chat. |
| `fw_giveitem <serverId> <name> [count]` | Server console only | Calls `AddItem` for that player. Typing it as a player, or in chat, does nothing — the console check happens before anything else runs. |
| `fw_gridbonus <serverId> <key> <extraWidth> <extraHeight>` | Server console only | Calls `SetGridBonus` for that player. Same console-only behavior as `fw_giveitem`. |

## State bags

This resource does not set one. The bag screen reads `fwInventoryBusy` from [fw_menu](fw-menu.md) when a menu wants the bag kept closed.

## Item file

Put a Lua file in `resources/[framework]/fw_inventory/items/`. The file returns a map. The key is the item name: lowercase letters, digits, and underscores, up to 32 characters.

```lua
return {
    dried_meat = {
        label = 'Dried Meat',
        group = 'food',
        width = 1,
        height = 1,
        stack = 20,
        description = 'Smoked and dried. Keeps for a long time.',
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
| `kit` | none | Which weapon slot can hold it: `revolver`, `pistol`, `longarm`, `bow`, `lasso`, `melee`, `lantern`, or `binoculars`. |
| `hash` | none | Weapon name, such as `WEAPON_REVOLVER_CATTLEMAN`. Required when `kit` is set. |
| `container` | none | Opens a second grid when the item is used from the satchel. Stack is forced to 1. |

If the item's name, or the `hash` you set, matches a known RDR3 weapon such as `weapon_revolver_cattleman` or `WEAPON_BOW`, you can leave `kit`, `hash`, `width`, `height`, and `rotate` out entirely — they are worked out for you from that name. The shipped `items/weapons.lua` sets them by hand anyway, so the shape is easy to see and change.

The picture is `fw_inventory/images/<name>.png`. A new picture is not served until `ensure fw_inventory`.

`fw_items_reload` reads `items/` again, as described in [Console commands](#console-commands). This resource always loads `default.lua` and `weapons.lua`, plus any other `.lua` file it finds in that folder, in alphabetical order. If two files define the same name, the one that sorts later wins.

## Container items

The walkthrough is [A pouch](#a-pouch-or-any-item-that-holds-other-items). This is the item that ships in `items/default.lua`. The name is the key, so `name` is not repeated inside.

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
        width = 6,
        height = 10,
        groups = { 'herbs' },
    },
}
```

`container` lives in the item file, or in the `RegisterItem` call. It is not a column you fill in by hand. A reload of the item files puts it back.

## Satchel screen

`fw_satchel` opens with B or `/satchel`. Escape or Tab closes it and releases any open storage.

Drag a placement. The cells under it turn green when it fits and red when it does not. R turns it when `rotate` is true. Dropping a stack on the same item, with the same metadata and room left in `stack`, combines them. A weapon never combines.

Right-click a satchel placement for Use, Drop, or Split. Double-click uses it. If nothing registered a use callback, the line under the grid says it cannot be used.

A storage open adds a second panel to the left of the satchel, only as large as that storage's grid. The satchel and the weapon loadout stay where they are. When the loadout is open, the storage panel sits to the left of it. Close on that panel hides it and leaves the satchel open. Double-click on the storage grid is refused until the item is back in the satchel.
