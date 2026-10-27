# fw_inventory

Runs on the server. Give a player an item, take one away, add a new kind of item, or open a second grid such as a chest or a pouch.

`fw_satchel` is the screen the player sees. It draws their bag and, when one is open, the chest or pouch beside it. Your script does not open that screen itself, except with `OpenStorage`, which opens both.

Ask `IsReady` first. If you also listen for `fw_inventory:ready` and your script starts later, you missed that message. Ask `IsReady` instead.

A second grid is the same kind of squares as the player's bag. There are two ways to open one. A chest, a safety deposit box, a wagon, or a horse bag is a storage your script registers, then opens. A pouch is an item the player uses. Those are different jobs. Both are below.

## A chest, a wagon, or a horse

Register the storage when your script starts. The items stay in the database. The registration itself is remembered only while your script is running. When your script stops, the spot in the world goes away, and a restart does not leave a second prompt.

`source` is the player's number for this visit.

```lua
exports.fw_inventory:RegisterStorage({
    id = 'chest:valentine_1',
    label = 'Chest',
    width = 8,
    height = 6,
    categories = { 'materials', 'tools' },
    acceptWeapons = false,
    adminOnly = false,
    shared = true,
    characters = nil,
    locations = {
        { coords = vector3(-308.0, 776.0, 118.0), distance = 2.0 },
    },
})
```

Hold at that spot opens the chest. A horse bag or a wagon has no `locations`. Your horse or wagon script calls `OpenRegistered` when the player is there.

- `id` is how the chest is remembered. `chest:valentine_1` is one chest. `chest:valentine_2` is a different chest. Use lowercase letters, digits, underscore, colon, and hyphen. Start with a letter or a digit. No spaces. Keep it to 64 characters or fewer. Do not start it with `item:`. That prefix is the storage inside a pouch.
- `shared = true` is one set of items for everyone, saved under that `id`. A town chest.
- `shared = false` is one set of items per character, saved as that `id` plus `:character:` plus the character's id. A safety deposit box, a horse bag, or a wagon. Register it once. Each character gets their own contents the first time they open it. The finished name still has to fit in 64 characters.
- `characters`, on a shared storage, is the list of character ids allowed to open it. Leave it out, or pass nothing, and any character can open it. A personal storage ignores this list. The character id comes from `fw_characters`.
- `adminOnly = true` means only a player `fw_admin` counts as an admin can open it. Anyone else gets `You cannot open that.` It still follows `shared`. A shared admin chest is one grid. A personal admin chest is one grid per admin. Leave it out and it is false.
- `locations` is optional. Each entry is `coords` and `distance`. With no locations, nothing is placed in the world.
- `width` and `height` are how many squares, from 1 to 24. The example is 8 across and 6 down.
- `label` is the title the player sees, and the words on the prompt. Leave it out and the title is `Storage`.
- The bag opens at the same time, with the chest beside it. The player drags items between them.
- The player must already be playing a character. Otherwise you get `Select a character first.`

A script that registered a storage opens it with `OpenRegistered`, so the personal name, the character list, and `adminOnly` are applied. `OpenStorage` is the low-level open. Use it for a storage you did not register. A direct `OpenStorage` does not add `:character:`, does not check the character list, and does not check `adminOnly`.

One person can have a chest open at a time. The next person gets `Someone is using that.` One player can have one of these open. Opening a second one closes the first.

`CloseStorage` hides the chest and leaves the bag open. Closing the bag on the player's screen also closes the chest — that happens on the player's screen, not on this page.

To make a chest larger later, register it again with the same name and a bigger width or height, or call `OpenStorage` with that same name. The items stay. Making it smaller, while an item sits on a square that would disappear, comes back as `Make room in there first.` Nothing is moved.

### Which items can go in

Every item has a category. The categories are `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `currency`, `weapons`, `materials`, and `container`. A pouch uses `container`. Dollars use `currency`. The balance a shop spends is on [fw_money](fw-money.md).

- Leave `categories` out, and every category is allowed, except weapons when `acceptWeapons` is false.
- Pass a list, and only those categories can be dragged in. Herbs and tools: `categories = { 'herbs', 'tools' }`.
- An empty list, `categories = {}`, takes nothing.
- A name that is not in the list above comes back as `unknown category`.
- Something other than a list of strings, such as a number or a single string, comes back as `Invalid storage categories.`
- `acceptWeapons` is its own yes or no. It is not worked out from the category list. Leave it out on `RegisterStorage` and it is false, so a gun stays out. `true` lets weapons in. A weapon is an item whose category is `weapons`, or one that has a `kit` or a weapon `hash`. This is checked when the item is dragged in, including when that open came from a registration. A direct `OpenStorage` that was never registered still accepts weapons, so an older caller keeps working. A registered storage that refuses a gun shows `Weapons do not go in there.`

When the player drags the wrong kind of item, they see `That does not go in there.` The bag itself takes every category.

A pouch, or any other item that has a `container` block, cannot be put into a chest, a wagon, a horse bag, or another pouch. The player sees `A container cannot go in there.` Taking that pouch back out into the bag is allowed. Handing one to another player is allowed.

```lua
exports.fw_inventory:OpenRegistered(source, 'chest:valentine_1')
```

## A pouch, or any item that holds other items

A pouch is an item in the bag. When the player uses it, a second grid opens, the same kind of grid as a chest. You do not call `OpenStorage` for a pouch. The item does that.

You add the pouch the same way you add any other item: a file in `items/`, or `RegisterItem`. The extra part is `container`.

```lua
local ok, err = exports.fw_inventory:RegisterItem({
    name = 'seed_pouch',
    label = 'Seed Pouch',
    category = 'container',
    width = 2,
    height = 2,
    stack = 1,
    description = 'A small pouch for seeds.',
    container = {
        label = 'Seed pouch',
        width = 4,
        height = 4,
        categories = { 'herbs' },
    },
})
if not ok then
    print(err)
    return
end
```

The same fields work in an item file. The name is the key, so you leave `name` out of the file. The [item file](#item-file) section shows that shape.

`items/container.lua` already ships with a pouch named `herb_pouch`, 2 by 2 in the bag with a 6-by-10 grid inside that only takes herbs. You can give that one to a player without registering anything, the same way you'd give any other item that ships in that folder.

What you pass:

- `category` on the pouch itself is `container`. That is the filter on the left of the bag. The list inside `container.categories` is what the pouch accepts.
- `width` and `height` on the item are how many squares the pouch takes in the bag. Here that is 2 by 2.
- `container.width` and `container.height` are how many squares are inside the pouch, from 1 to 24. Here that is 4 by 4.
- `container.label` is the title of that second grid.
- `container.categories` follows the same rules as a chest. Leave it out and the pouch takes anything. `{ 'herbs' }` takes herbs only. A pouch still cannot hold another pouch. The player sees `A container cannot go in there.`
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

The player opens the bag with B or `/satchel`, then right-clicks the pouch and chooses Open. The pouch grid appears beside the bag. They drag items in. Close on that panel hides the pouch and leaves the bag open.

Drag a pile onto the pouch picture in the bag to put the whole pile inside while the pouch stays closed. The pile fills an open pile of the same item, then the first empty square. Split it first when you want to put in less. A pouch that does not take that kind of item says `That does not go in there.` Another pouch says `A container cannot go in there.` A full pouch says `Not enough room.` A pile set aside for a trade stays in the bag, and the reason is `That item is in a trade.` Opening the pouch and dragging into its panel still works. If that pouch is already the one open beside the bag, the panel shows the pile. Any other open chest stays as it was.

A use function from `RegisterUsable` is skipped for a pouch. The pouch menu shows Open, and Open shows the pouch.

Using a pouch while it is sitting inside another open chest or pouch is refused. The player sees `Take it out first.` Put it back in the bag, then use it.

Discard throws the pouch away and deletes what was inside it. `RemoveItem` does the same. Moving it does not. A pouch that still holds dollars refuses with `Empty the bag first.` Take the dollars out first, or drop the bag. Dollars themselves refuse Discard with `You cannot throw money away.`

Dollars in a pouch you are carrying still count on `GetCash`. Dropping that pouch moves those dollars onto the pouch. They count again when someone picks the pouch up.

Drop sets the pouch on the ground in front of the player. The contents stay, and the pouch keeps the same size and the same categories. Hold Open at the bag to look inside. Hold Pick up to put it back in a satchel that has room. Anyone nearby can do either. A restart leaves the pouch where it was. Bread, ammo, and guns have Discard only. They do not get Drop. Dollars do not get Drop either. The player puts them in a money bag, then drops that bag. The reason on the dollars themselves is `Put that in a bag first.`

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
    category = 'food',
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

`usable` is `true` or left out. Left out means the item is not usable, so the right-click menu leaves Use off. Set `usable = true` when the player should see Use. `RegisterUsable` turns Use on as well, for as long as that callback stays registered.

`context` is a list of extra right-click rows. Each entry is `{ id = 'sharpen', label = 'Sharpen' }`, or a short id string such as `'sharpen'`. The id is lowercase letters, digits, and underscores, up to 32 characters, and it starts with a letter. `use`, `open`, `give`, `drop`, `discard`, and `split` are already taken. The label is the words on the row, clipped to 32 characters. Leave `label` out and the id is shown.

```lua
local ok, err = exports.fw_inventory:RegisterItem({
    name = 'knife',
    label = 'Knife',
    category = 'tools',
    width = 1,
    height = 2,
    usable = true,
    context = {
        { id = 'sharpen', label = 'Sharpen' },
    },
})
```

When the player picks Sharpen, the server sends `fw_inventory:contextOption`. That message is local. Hear it with `AddEventHandler`. It is not queued, so the listener has to be running already. `source` is the player's number for this visit. `item` is a copy. `option` is the id, here `sharpen`.

```lua
AddEventHandler('fw_inventory:contextOption', function(source, item, option)
    if item.name ~= 'knife' or option ~= 'sharpen' then
        return
    end
end)
```

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

If your resource stops, its use callbacks are removed, and Use leaves the menu for those items unless the item itself has `usable = true`. Start it again and register them again.

Use is on the menu when `usable` is true, or when this callback is registered. Double-click uses that item too. A pouch shows Open instead, and double-click leaves the pouch closed.

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

The copy can include `name`, `label`, `description`, `width`, `height`, `stack`, `weapon`, `category`, `rotate`, `kit`, `hash`, `container`, `currency`, `image`, `chamber`, `rounds`, and `ammo`. `image` is the picture file in `fw_inventory/images`, without `.png`, when it is not the item name. `currency` is set on a pile of money, such as `cash` for dollars. Shops spend that balance through [fw_money](fw-money.md). `category` is the kind of item, such as `food` or `tools`. `chamber` is which rounds a gun takes, such as `revolver` or `rifle`. `rounds` is how many that gun holds. `ammo` is set on a box or a bundle of arrows. `key` is the game ammo name, such as `AMMO_REVOLVER`. `fits` is the same word as the gun's `chamber`. `max` is the most of that kind the character can hold. `refill` is how many one use adds. `instant` is yes for arrows, which apply as soon as they are used.

```lua
local def = exports.fw_inventory:GetDefinition('herb_pouch')
if def == nil then
    return
end
```

### GetCatalog

Runs on the server. A copy of every item type, keyed by name. Same fields as `GetDefinition`. Changing the copy does not change the list.

```lua
--- Every item type, keyed by name.
---@return table catalog
exports.fw_inventory:GetCatalog()
```

```lua
local catalog = exports.fw_inventory:GetCatalog()
local rounds = catalog.ammo_revolver
if rounds == nil then
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

Ammo is a box, or a bundle of arrows. `stack` is the most of that item the satchel can hold. Giving more than that fails with `You cannot carry any more of those.` Using the item from the satchel puts `refill` rounds on the character, up to `max`. A box takes about two seconds. The bag stays open while the revolver-cylinder loading prompt shows at the bottom right. Backspace cancels it and keeps the box. Arrows apply right away. The box is not used up when the character is already at `max`. A shop sells one box:

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'ammo_revolver', 1)
if not ok then
    print(err)
    return
end
```

The player uses it from the satchel. The rounds stay on the character when they log out, and they come back when that character is selected again. Draw the gun and press reload. That is on the [weapons](fw-weapons.md) page.

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

### TakeCurrency

Runs on the server. Takes dollars off one stack. `itemId` is that stack's id. `storageId` is the pouch when the stack is inside one you are still carrying, such as `item:` followed by the pouch id. Leave `storageId` empty when the stack is loose in the satchel. The count comes off that stack, and the character's dollar total drops by the same amount. If either change cannot be saved, neither one is kept.

A pouch on the ground, or a chest, is refused. So is a stack that is not dollars, and a stack that is sitting in a trade.

```lua
--- Takes dollars off one stack.
--- False and a reason when that stack cannot pay the count.
---@param source integer
---@param itemId string
---@param count integer
---@param storageId string|nil pouch id, or nothing when the stack is loose
---@return boolean ok
---@return string|nil err
exports.fw_inventory:TakeCurrency(source, itemId, count, storageId)
```

```lua
local ok, err = exports.fw_inventory:TakeCurrency(source, pileId, 9, 'item:' .. pouchId)
if not ok then
    print(err)
    return
end
```

### TakePlacement

Runs on the server. Takes a count off one stack that is not dollars. `itemId` is that stack's id. `storageId` is the pouch when the stack is inside one you are still carrying. Leave `storageId` empty when the stack is loose in the satchel. Dollars are refused. Use `TakeCurrency` for those, because the dollar total has to drop with the stack.

A pouch on the ground, or a chest, is refused. So is a stack that is sitting in a trade. If the stack does not hold that many, nothing is taken.

```lua
--- Takes a count off one stack that is not dollars.
--- False and a reason when that stack cannot give the count.
---@param source integer
---@param itemId string
---@param count integer
---@param storageId string|nil pouch id, or nothing when the stack is loose
---@return boolean ok
---@return string|nil err
exports.fw_inventory:TakePlacement(source, itemId, count, storageId)
```

```lua
local ok, err = exports.fw_inventory:TakePlacement(source, pileId, 2, 'item:' .. pouchId)
if not ok then
    print(err)
    return
end
```

### RemoveItemDeferred

Runs on the server. Takes that many of the name out of the satchel, the same way `RemoveItem` does, and the bag screen updates right away. The database write waits.

The waiting writes go out together about every five seconds. The satchel is also written when the player leaves, when the character is cleared, and when `fw_inventory` stops. A crash inside that wait can put the items back.

```lua
--- Removes items now and writes the satchel on the next flush.
--- False and a reason when the player does not hold that many.
---@param source integer
---@param name string
---@param count integer
---@return boolean ok
---@return string|nil err
exports.fw_inventory:RemoveItemDeferred(source, name, count)
```

```lua
local ok, err = exports.fw_inventory:RemoveItemDeferred(source, 'dried_meat', 1)
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
--- False and a reason when the id, size, or categories are rejected, or someone else has it open.
---@param source integer
---@param id string
---@param params table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:OpenStorage(source, id, params)
```

`id` is the name from the chest section. `params.width` and `params.height` are the squares, from 1 to 24. `params.label` is the title. `params.categories` is the list of categories, or leave it out. `params.acceptWeapons` defaults to yes on this call. Pass `false` when this open should refuse weapons. A storage you registered should be opened with `OpenRegistered` instead, so the personal name, the character list, and `adminOnly` are applied.

| `err` | When |
| --- | --- |
| `fw_inventory is not ready` | The item list has not loaded yet. Ask `IsReady` first. |
| `Select a character first.` | This player is not playing a character. |
| `player is not loaded` | `source` is not a player who is in the server. |
| `Invalid storage.` | The name has a space, a capital letter, or some other character that is not allowed. Or `params` was left out. |
| `Invalid storage size.` | Width or height is outside 1–24. |
| `unknown category` | A category name is not in the list in the chest section. |
| `Invalid storage categories.` | `params.categories` was something other than a list of category names. |
| `Make room in there first.` | A later open tried to make the grid smaller while an item sits on a square that would disappear. |
| `Someone is using that.` | Another player already has this one open. |

A container item cannot be placed into the storage. The player sees `A container cannot go in there.` Taking it back out into the bag is allowed.

```lua
local ok, err = exports.fw_inventory:OpenStorage(source, 'chest:valentine_1', {
    label = 'Chest',
    width = 8,
    height = 6,
    categories = { 'materials', 'tools' },
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

### RegisterStorage

Runs on the server. Call it when your script starts. The fields are in [A chest, a wagon, or a horse](#a-chest-a-wagon-or-a-horse). The registration is dropped when your script stops. The items stay saved.

```lua
--- Remembers a storage until the calling resource stops.
--- False and a reason when the id, size, categories, character list, or locations are rejected.
---@param params table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:RegisterStorage(params)
```

`params.id` follows the name rules in the chest section, and it must not start with `item:`. `params.shared` left out means one shared grid. `params.acceptWeapons` left out means weapons stay out. `params.adminOnly` left out means anyone can open it. `params.characters` left out means any character, on a shared storage. `params.locations` left out means no spot in the world.

| `err` | When |
| --- | --- |
| `Invalid storage.` | The name is missing, too long, or starts with `item:`. Or `params` was left out. |
| `Invalid storage size.` | Width or height is outside 1–24. |
| `unknown category` | A category name is not in the list in the chest section. |
| `Invalid storage categories.` | `params.categories` was something other than a list of category names. |
| `Invalid character list.` | `params.characters` was not a list of character ids. |
| `Invalid storage locations.` | A spot was missing `coords`. |

```lua
local ok, err = exports.fw_inventory:RegisterStorage({
    id = 'bank:valentine',
    label = 'Safety deposit box',
    width = 8,
    height = 6,
    shared = false,
    locations = {
        { coords = vector3(-308.0, 776.0, 118.0), distance = 2.0 },
    },
})
if not ok then
    print(err)
    return
end
```

That box is personal. Each character gets their own items the first time they hold the prompt.

### OpenRegistered

Runs on the server. Opens a storage you registered. It checks the character, the character list, `adminOnly`, and `shared`, then opens the grid. The player must be playing a character.

```lua
--- Opens a registered storage for this player.
--- False and a reason when they cannot open it, or the storage was not registered.
---@param source integer
---@param id string
---@return boolean ok
---@return string|nil err
exports.fw_inventory:OpenRegistered(source, id)
```

`id` is the name you passed to `RegisterStorage`, without the `:character:` ending. A personal storage adds that ending for you.

| `err` | When |
| --- | --- |
| `fw_inventory is not ready` | The item list has not loaded yet. Ask `IsReady` first. |
| `Select a character first.` | This player is not playing a character. |
| `Finish the trade first.` | This player is in a trade. |
| `Invalid storage.` | Nothing is registered under that name, or the personal name would be longer than 64 characters. |
| `You cannot open that.` | `adminOnly` is set and this player is not an admin, or `characters` is set and this character is not on the list. |
| `Someone is using that.` | Another player already has this grid open. |

```lua
local ok, err = exports.fw_inventory:OpenRegistered(source, 'bank:valentine')
if not ok then
    print(err)
    return
end
```

A horse or a wagon registers with `shared = false` and no `locations`, then calls this when the player is at that horse or wagon.

### UnregisterStorage

Runs on the server. Removes one registration. The prompt goes away. The saved items stay, so registering the same name again opens the same chest.

```lua
--- Removes one registration. True when it was registered.
---@param id string
---@return boolean removed
exports.fw_inventory:UnregisterStorage(id)
```

```lua
exports.fw_inventory:UnregisterStorage('chest:valentine_1')
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

`id` is the placement id from `GetItems`. `displaced` is the item leaving that weapon slot, with `name`, `count`, `metadata`, and `rotated`. It is put back on the cell that was freed. If that cell does not fit, it is placed anywhere that does. If neither works, nothing is taken and the reason is `Make room in the satchel first.` A placement set aside with `SetHold` is not taken. The reason is `That item is in a trade.`

### SetHold

Runs on the server. Marks placements that must stay in the satchel. Discard, drop, use, equip, sort, and moving one into a chest all refuse with `That item is in a trade.` Move and split still work on placements that are not in the list. Pass nothing, or an empty list, to clear the hold.

While any placement is held, opening a chest or a pouch comes back as `Finish the trade first.`

```lua
--- Marks placements that a trade is holding. Nil or an empty list clears the hold.
--- False and a reason when a listed placement is not in the satchel.
---@param source integer
---@param ids string[]|nil
---@return boolean ok
---@return string|nil err
exports.fw_inventory:SetHold(source, ids)
```

`source` is the player's number for this visit. `ids` are placement ids from `GetItems`. The same id twice comes back as `Invalid hold.`

```lua
local items = exports.fw_inventory:GetItems(source)
local ok, err = exports.fw_inventory:SetHold(source, { items[1].id })
if not ok then
    print(err)
    return
end

exports.fw_inventory:SetHold(source, nil)
```

### CommitTrade

Runs on the server. Gives each player's listed placements to the other player. Both satchels change, or neither does. The items land anywhere they fit. They do not keep the cells they had in an offer grid.

Call this only when you already know both offers are the items you mean to move. If a placement is gone, or its count or metadata changed, nothing moves and the reason is `That offer changed.` If the other satchel cannot hold what is coming in, nothing moves and the reason is `Not enough room.` or `You cannot carry any more of those.`

```lua
--- Swaps the listed placements between two satchels.
--- False and a reason when either offer does not match the satchel, or either satchel cannot hold what it would receive.
---@param sourceA integer
---@param sourceB integer
---@param offerA table
---@param offerB table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:CommitTrade(sourceA, sourceB, offerA, offerB)
```

Each offer is a list. Each entry needs `id`, `name`, `count`, and `metadata` for one whole placement. An empty list gives nothing. `sourceA` and `sourceB` are the two players' numbers. They must be different, and both must be playing a character.

```lua
local ok, err = exports.fw_inventory:CommitTrade(sourceA, sourceB, offerA, offerB)
if not ok then
    print(err)
    return
end
```

[fw_trade](fw-trade.md) is the screen that locks two offers and then calls this.

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

### fw_inventory:changed

Local, on the server. Sent after the satchel is saved, with the player's number for this visit (`source` on other pages). Hear it if your script needs that same moment.

```lua
---@param source integer
AddEventHandler('fw_inventory:changed', function(source)
    local items = exports.fw_inventory:GetItems(source)
end)
```

### What the bag screen sends the server

These are networked (`RegisterNetEvent` on the server, sent from the player's game with `TriggerServerEvent`). `fw_satchel` sends these when the player drags, uses, sorts, or drops. A script with its own bag screen could send them too, but for giving or taking items from your own script, use the calls above instead — `AddItem`, `RemoveItem`, and so on, not these.

`token` is a value you make up and send with the request. It comes back unchanged on `fw_inventory:updated` or `fw_inventory:rejected`, so a screen that fires several requests at once can tell which response answers which request. Leave it out, or pass anything, if you don't need that.

| Event | Parameters | What it does |
| --- | --- | --- |
| `fw_inventory:open` | none | Opens the bag for the sender. Answered with `fw_inventory:opened`, or `fw_inventory:rejected` when the item list is not ready or the player has no character. |
| `fw_inventory:move` | `id, x, y, rotated, token` | Moves a placement already in the bag to a new cell, or turns it. `id` is the placement id. `rotated` is `true`, `false`, or left out to keep its current facing. |
| `fw_inventory:sort` | `token` | Rearranges the whole bag by item category, then size. |
| `fw_inventory:discard` | `id, count, token` | Throws away up to `count` of that placement. `count` defaults to 1. The last one removes the placement, and deletes the container it owns, if it had one. |
| `fw_inventory:drop` | `id, token` | Sets a pouch on the ground. `id` is the placement. Only an item with a `container` block can be dropped. The contents stay. `That cannot be dropped.` is anything else. `That item is in a trade.` is a placement set aside for a trade. |
| `fw_inventory:split` | `id, count, token` | Cuts `count` off a stackable placement onto an empty cell of its own. `count` is a whole number from 1 up to one less than what is already there, and the item must be able to stack. `Not enough room.` means the bag has no free cell of that size. |
| `fw_inventory:use` | `id, token, side` | Uses that placement, or opens a pouch. The bag sends this for Use, and for Open on a pouch. `side` is `'storage'` when the placement is sitting in an open chest or pouch rather than the bag — that is refused with `Take it out first.` Opens a pouch's storage, or runs its `RegisterUsable` callback, or rejects with `This cannot be used.` Three pouches can be open at once. A fourth is refused with `Close a pouch first.` |
| `fw_inventory:context` | `id, option, token` | The player picked a custom right-click row. `id` is the placement. `option` is the id from `context` on the item. `That action is not on this item.` means that id was not registered. `That item is in a trade.` means the pile is set aside. On success the server sends the local message `fw_inventory:contextOption`. |
| `fw_inventory:storageClose` | `id` | Closes one pouch when `id` is that pouch. Leave `id` out to close every pouch the sender has open. Answered with `fw_inventory:storageClosed`. |
| `fw_inventory:storageMove` | `storageId, id, x, y, rotated, token` | Moves a placement inside the pouch named by `storageId`, not the bag. `id` is the placement. |
| `fw_inventory:transfer` | `fromSide, id, x, y, rotated, token, storageId, toStorageId` | Moves a placement between the bag and one open pouch, or from one open pouch to another. `fromSide` is `'satchel'` or `'storage'`. `storageId` is the pouch. When the pile moves from one open pouch onto another, `storageId` is the pouch it leaves and `toStorageId` is the pouch it lands in. |
| `fw_inventory:deposit` | `id, containerId, token` | Puts the whole satchel pile `id` into the pouch placement `containerId`. The pouch can stay closed. The pile fills an open pile of the same item, then the first empty square. `Not enough room.` means the pouch cannot hold it. `That does not go in there.` means the pouch does not take that category. `A container cannot go in there.` means the pile is itself a pouch. `That item is in a trade.` means one of the two placements is set aside. |
| `fw_inventory:openRegistered` | `id` | Opens a registered storage for the sender. `id` is the name from `RegisterStorage`. Answered with `fw_inventory:opened`, or `fw_inventory:rejected` with `You cannot open that.` and the other reasons from `OpenRegistered`. The world prompt sends this. A horse script can call `OpenRegistered` instead. |
| `fw_inventory:requestStorageLocations` | none | Asks for the spots that have a prompt. Answered with `fw_inventory:storageLocations` for the sender. |
| `fw_inventory:requestNearbyDrops` | none | Asks which pouches are within 100 meters of the sender. Answered with `fw_inventory:dropped` only when that set changed. The same set gets no reply. |
| `fw_inventory:openDropped` | `id` | Opens a pouch on the ground. `id` is the ground pouch from `fw_inventory:dropped`. Answered with `fw_inventory:opened`, or `fw_inventory:rejected`. `That is gone.` means someone already picked it up. `You are too far away.` means the player is not next to it. |
| `fw_inventory:pickupDropped` | `id` | Puts that ground pouch into the sender's satchel when there is room. The contents come with it. `Not enough room.` leaves it on the ground. |

```lua
RegisterNetEvent('fw_inventory:use', function(id, token, side)
    -- fw_satchel already does this. Shown so a custom screen knows the shape.
end)
```

### What the server sends the bag screen

These are networked (`RegisterNetEvent` on the player's game, sent from the server with `TriggerClientEvent`). The screen in [fw_satchel](fw-satchel.md) listens for these. A script with its own bag screen can listen too. Giving an item from your own script is still `AddItem`, not one of these.

| Event | Parameters | What it carries |
| --- | --- | --- |
| `fw_inventory:opened` | `payload` | `payload.grid` is the bag. `payload.definitions` and `payload.categories` are every item type and the category names. `payload.storage` is the list of pouches that player has open. It is empty when none are open. |
| `fw_inventory:updated` | `grid, token, storage` | The bag after a change, the `token` from the request that caused it (or nothing, when this page's own calls caused the change), and the list of open pouches. A pouch that left the list should come off the screen. |
| `fw_inventory:definitions` | `definitions, categories` | Sent to everyone. Every item type and the category names, after `RegisterItem`, a reload, or a new weapon type being created automatically. |
| `fw_inventory:rejected` | `reason, token` | A short reason, such as `That does not go in there.`, and the `token` from the request that failed. |
| `fw_inventory:close` | none | Hide the bag. Sent when the character is deselected, the player leaves, or this resource stops. |
| `fw_inventory:storageClosed` | `id` | Hide one pouch when `id` is set. Hide every open pouch when `id` is left out. The bag stays. |
| `fw_inventory:storageLocations` | `list` | Sent to one player, or to everyone, when a storage is registered or removed. Each entry has `id`, `label`, and `locations`. `locations` is `{ x, y, z, distance }`. `fw_inventory` places a Hold prompt from this list. |
| `fw_inventory:dropped` | `change` | The reply to the player who asked with `fw_inventory:requestNearbyDrops`, and only when a pouch was added or removed within 100 meters of them. `change.add` is one pouch, with `id`, `label`, `x`, `y`, `z`, and `heading`. `change.remove` is the id to stop drawing. `fw_inventory` draws that one bag, with Hold to open and Hold to pick up. |

```lua
RegisterNetEvent('fw_inventory:rejected', function(reason, token)
    print(reason)
end)
```

## Console commands

| Command | Where | What it does |
| --- | --- | --- |
| `fw_items_reload` | Server console, or `/fw_items_reload` by an in-game admin | Reads `items/` again and re-saves every item. A player who is not an admin gets `You are not an admin.` in the log. An admin gets an INFO line for how many items were reloaded. |
| `fw_giveitem <serverId> <name> [count]` | Server console only | Calls `AddItem` for that player. A player who types it gets a warning in the log. The server window gets the same warning. |
| `fw_gridbonus <serverId> <key> <extraWidth> <extraHeight>` | Server console only | Calls `SetGridBonus` for that player. Same console-only behavior as `fw_giveitem`. |

## State bags

This resource does not set one. The bag screen reads `fwInventoryBusy` from [fw_menu](fw-menu.md) when a menu wants the bag kept closed.

## Item file

Put a Lua file in `resources/[framework]/fw_inventory/items/`. The file returns a map. The key is the item name: lowercase letters, digits, and underscores, up to 32 characters.

```lua
return {
    dried_meat = {
        label = 'Dried Meat',
        category = 'food',
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
| `category` | Required | `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `currency`, `weapons`, `materials`, or `container`. A pouch uses `container`. Dollars use `currency`. |
| `stack` | 1 | How many copies share one footprint. Maximum 1000. |
| `weapon` | false | When true, the item never stacks. |
| `rotate` | false | The player can turn it. |
| `kit` | none | Which weapon slot can hold it: `revolver`, `pistol`, `longarm`, `bow`, `lasso`, `melee`, `lantern`, `binoculars`, or `fishingrod`. |
| `hash` | none | Weapon name, such as `WEAPON_REVOLVER_CATTLEMAN`. Required when `kit` is set. |
| `container` | none | Opens a second grid when the player right-clicks the item and chooses Open. Stack is forced to 1. |
| `usable` | false | `true` puts Use on the right-click menu. Left out, or `false`, leaves Use off unless `RegisterUsable` has a callback for this name. A pouch still shows Open. |
| `context` | none | Extra right-click rows. A list of `{ id = 'sharpen', label = 'Sharpen' }`, or id strings. The player’s choice arrives as `fw_inventory:contextOption`. |
| `ammo` | none | A box or a bundle of arrows. `key` is the game ammo name, such as `AMMO_REVOLVER`. `fits` is which guns take it: `revolver`, `pistol`, `repeater`, `rifle`, `elephant`, `shotgun`, `arrow`, or `varmint`. `max` is the most of that kind on the character, from 1 to 999. `refill` is how many one use adds, and it cannot be higher than `max`. `instant` skips the open wait. Arrows set it. `stack` is how many of the item the satchel can hold. |
| `chamber` | filled in for shipped guns | Which `fits` value this gun accepts. Set it yourself on a new gun. |
| `rounds` | filled in for shipped guns | How many rounds that gun holds. A Cattleman is 6. A bow is 20. From 1 to 100. |

Shipped guns already have `chamber` and `rounds`. You only set those two when you add a gun the list does not know. Shipped ammo is in `items/ammo.lua`. A revolver box adds 12, up to 200. An explosive box adds 10, and 10 is the cap. Arrows add 10, up to 40, and they apply as soon as they are used. Fire, poison, and dynamite arrows add fewer. The satchel holds 20 of any one ammo item.

If the item's name, or the `hash` you set, matches a known RDR3 weapon such as `weapon_revolver_cattleman` or `WEAPON_BOW`, you can leave `kit`, `hash`, `width`, `height`, and `rotate` out entirely — they are worked out for you from that name. The shipped `items/weapons.lua` sets them by hand anyway, so the shape is easy to see and change.

The picture is `fw_inventory/images/<name>.png`. Set `image` on the item file when the picture has another name. Dollars use `bill_stack_single`. Leave off the `.png`. A new picture is not served until `ensure fw_inventory`.

### Picture size

`fw_satchel` draws the picture inside a box sized to the item's `width` and `height` in cells, and shrinks or grows the picture to fit that box without stretching it — so a picture of roughly the right shape always looks right, even if its pixels don't match the table below exactly.

For a crisp picture at the satchel's normal size, one cell is 48 pixels, with a 2-pixel gap between cells. A footprint of `n` cells in one direction comes out to `n * 48 + (n - 1) * 2` pixels in that direction:

| width \ height | 1 | 2 | 3 | 4 |
| --- | --- | --- | --- | --- |
| **1** | 48x48 | 48x98 | 48x148 | 48x198 |
| **2** | 98x48 | 98x98 | 98x148 | 98x198 |
| **3** | 148x48 | 148x98 | 148x148 | 148x198 |
| **4** | 198x48 | 198x98 | 198x148 | 198x198 |

A `rope`, 3 wide and 1 tall, is `148x48`. A `herb_pouch`, 2 by 2, is `98x98`.

The satchel's cell can grow up to 88 pixels on a tall window, so a picture made at double the table above holds up better on a bigger screen. There's no setting for this — it's just a size to draw at.

`fw_items_reload` reads `items/` again, as described in [Console commands](#console-commands). This resource always loads `ammo.lua`, `container.lua`, `currency.lua`, `food.lua`, `medical.lua`, `tools.lua`, and `weapons.lua`, plus any other `.lua` file it finds in that folder, in alphabetical order. The file name is the list you open. Each item still has its own `category`. If two files define the same name, the one that sorts later wins.

## Container items

The walkthrough is [A pouch](#a-pouch-or-any-item-that-holds-other-items). This is the item that ships in `items/container.lua`. The name is the key, so `name` is not repeated inside.

```lua
herb_pouch = {
    label = 'Herb Pouch',
    category = 'container',
    width = 2,
    height = 2,
    stack = 1,
    description = 'A pouch for herbs.',
    container = {
        label = 'Herb pouch',
        width = 6,
        height = 10,
        categories = { 'herbs' },
    },
}
```

`container` lives in the item file, or in the `RegisterItem` call. It is not a column you fill in by hand. A reload of the item files puts it back.

## Satchel screen

`fw_satchel` opens with B or `/satchel`. Escape or Tab closes it and releases any open storage. A trade uses that same screen. Escape declines the trade. That is on [fw_trade](fw-trade.md).

Drag a placement. The cells under it turn green when it fits and red when it does not. R turns it when `rotate` is true. Dropping a stack on the same item, with the same metadata and room left in `stack`, combines them. A weapon never combines.

Right-click a satchel placement for Use, Give, or Discard. A pile of more than one also offers Split in half and Split amount. Half cuts the pile in two, and an odd count leaves the extra one on the original pile. Amount is a slider from 1 up to one less than the pile. Either cut lands on an empty cell of its own. A pouch also has Drop. Dollars have no Discard. Discard throws the whole stack away. Drop sets that pouch on the ground. Give hands that whole stack to a nearby player. That is on [fw_trade](fw-trade.md). Double-click uses it. If nothing registered a use callback, the line under the grid says it cannot be used.

A storage open adds a second panel to the left of the satchel, only as large as that storage's grid. The satchel and the weapon loadout stay where they are. When the loadout is open, the storage panel sits to the left of it. Close on that panel hides it and leaves the satchel open. Double-click on the storage grid is refused until the item is back in the satchel.
