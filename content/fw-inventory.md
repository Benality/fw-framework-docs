# fw_inventory

Version 1.0.0. Server resource. The satchel screen is `fw_satchel` 2.0.0.

One row per character in `fw_inventory`. That row stores the placed grid as one JSON document. Item types are separate rows in `fw_items`. `fw_satchel` draws the grid and sends a move, use, or discard. The server commits it or rejects it.

The grid stays one document because a satchel is small: ten by ten cells, plus whatever a backpack adds, still a few kilobytes. A move rewrites that one row. The whole layout is saved together, so two placements cannot overlap from a half-written update. Splitting each placement into its own row would not make opening or dragging faster at this size.

`fw_items` is the catalog. Adding a type is one insert, and it does not rewrite anyone's satchel. Giving a player an item still updates that player's grid document through `AddItem`.

Weapons are item types with `weapon` set. They never share a footprint. Equipping, firing, ammo, stashes, and cash are not part of this resource.

## Table

```sql
CREATE TABLE IF NOT EXISTS `fw_inventory` (
  `character_id` int unsigned NOT NULL,
  `grid` json NOT NULL,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`character_id`),
  CONSTRAINT `fk_fw_inventory_character`
    FOREIGN KEY (`character_id`) REFERENCES `fw_characters` (`id`)
    ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
```

`character_id` is `fw_characters.id`. Deleting the character deletes the satchel.

The grid document:

| Field | Meaning |
| --- | --- |
| `width` | Cells across. A new satchel is 10. This is the drawn size: the base plus every bonus. |
| `height` | Cells down. A new satchel is 10. |
| `baseWidth`, `baseHeight` | Size before bonuses. A new satchel is 10 by 10. |
| `bonus` | Named extras, such as `backpack = { width = 0, height = 5 }`. |
| `nextId` | Next placement id. Not sent to the satchel screen. |
| `items` | Placements. |

Each placement:

| Field | Meaning |
| --- | --- |
| `id` | String id, unique inside that satchel. |
| `name` | Item name, such as `bread`. |
| `x`, `y` | Top-left cell. `0, 0` is the top left. |
| `rotated` | `true` when a rotatable item is turned. |
| `count` | How many share this footprint. |
| `metadata` | Table stored with the stack. Different metadata does not share a stack. |

The encoded document must be at most 262144 bytes. A larger write is refused and the previous grid stays.

The row is loaded when `fw_characters:selected` fires, and written after a successful add, remove, move, discard, split, or bonus change, or when the character is cleared. Exports return copies.

A satchel saved as the old 10 by 6 default becomes 10 by 10 the next time it loads. Placements keep their cells.

## Storage

A chest, wagon, horse, or container item is one row in `fw_storage`. The caller chooses the id. The grid document has the same shape as a satchel. `groups` is a JSON list, or `NULL` when every group is allowed. The encoded grid stays at or under 262144 bytes.

```sql
CREATE TABLE IF NOT EXISTS `fw_storage` (
  `id` varchar(64) NOT NULL,
  `grid` json NOT NULL,
  `groups` json NULL,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
```

The first `OpenStorage` creates the row. Dropping a container item deletes `item:<containerId>`. World props do not get a row until something opens them.

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
    rope = {
        label = 'Rope',
        group = 'tools',
        width = 3,
        height = 1,
        stack = 1,
        rotate = true,
        description = 'A coil of rope. It can be turned to fit.',
    },
}
```

| Field | Default | Meaning |
| --- | --- | --- |
| `label` | The name | Text in the satchel. Clipped to 48 characters. |
| `description` | `''` | Side panel. Clipped to 200 characters. |
| `width`, `height` | Required, 1–12 | Cell footprint before turning. |
| `group` | Required | One of `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `weapons`, `materials`. |
| `stack` | `1` | How many copies share one footprint. `1` means each copy is its own placement. Max 1000. |
| `weapon` | `false` | When true, the item never stacks. `stack` is stored as 1. |
| `rotate` | `false` | The player can turn it so width and height swap. |
| `kit` | none | Which satchel slot can hold it: `revolver`, `pistol`, `longarm`, `bow`, `lasso`, `melee`, or `lantern`. A kit item does not stack. |
| `hash` | none | Weapon name, such as `WEAPON_REVOLVER_CATTLEMAN`. Required when `kit` is set. |
| `container` | none | Opens a second grid when the item is used from the satchel. `stack` is stored as 1. |

`fw_items_reload` reads the folder again, writes those types into `fw_items`, then loads every row in `fw_items`. A type that exists only in the table stays. A placed item whose name is gone stays in the grid. It is drawn as 1 by 1 and cannot be used until the name exists again.

An admin tool, or a row inserted by hand, uses the same table. After the insert, run `fw_items_reload`. `RegisterItem` writes the row itself and does not need that command.

```sql
INSERT INTO fw_items (name, label, description, width, height, stack, weapon, item_group, rotate)
VALUES ('canteen', 'Canteen', 'A full canteen.', 1, 2, 1, 0, 'tools', 1);
```

`weapon` and `rotate` are `1` or `0`. `item_group` is one of the group names. `name` is the primary key. The picture is `images/canteen.png`, the same name as the item.

A new `.png` in `images/` is not served until `ensure fw_inventory`.

Sample items shipped in `items/default.lua`: `bread` (food, 1x1, stack 10), `bandage` (medical, 1x1, stack 5), `rope` (tools, 3x1, can turn), and `herb_pouch` (tools, 2x2, opens a 4x4 grid that accepts `herbs`).

`items/weapons.lua` is the weapon catalog. Each name is the game weapon in lowercase, and `hash` is that same name in uppercase. Give one with `fw_giveitem 1 weapon_lasso`. Revolvers and pistols are 2x1, longarms and bows are 1x4, a lasso or melee weapon is 1x2, binoculars are 2x1, and a lantern is 2x2. Throwables, cameras, and the fishing rod are in the satchel and are not kit slots. The shop list spelled two of these wrong; the catalog uses `WEAPON_KIT_BINOCULARS` and `WEAPON_KIT_CAMERA_ADVANCED`. A Davy lantern uses the lantern slot. Binoculars use the binoculars slot.

The kit is a sheet behind the left side of the satchel. It stays hidden until the Kit button above Sort is pressed, then the whole sheet slides out. Press the button again and it slides back. Each slot is a square. Drag a kit item onto a slot to equip it, or drag it back onto the grid to put it away. `fw_weapons` stores what is equipped. There are two revolver slots (`revolver` and `pistol`), one longarm slot, one bow slot, one lasso slot, one knife slot for every `melee` item, one lantern slot, and one binoculars slot. A revolver or a knife stays equipped only while a gunbelt is worn. A second revolver also needs another holster worn: `holsters_left`, `holsters_crossdraw`, or `holsters_center`. Taking the gunbelt off returns the revolvers and the knife to the satchel. Taking off that extra holster returns the second revolver. A longarm and a bow can both be equipped. The equipped weapon is given to the character. Ammunition and wear are not handled yet.

`fw_giveitem 1 weapon_lasso` gives the lasso from that catalog. A weapon name that is not in the file is still created when it matches `weapon_lasso*`, `weapon_revolver_*`, `weapon_pistol_*`, `weapon_rifle_*`, `weapon_repeater_*`, `weapon_shotgun_*`, `weapon_sniperrifle_*`, `weapon_bow*`, `weapon_kit_binoculars*`, or any `weapon_melee_*` name that contains `lantern` (lantern slot) or is other melee. Throwables, cameras, and the fishing rod have to be in the catalog.

`animals` is the hunting group: meat, pelts, beaks, claws. `materials` is crafting supplies.

## Images

Item art is one square PNG named after the item, such as `fw_inventory/images/bread.png`. The files you have at 96×96 are the right size. The satchel scales that square to the largest size that fits the item's footprint and keeps it centered. A 1×1 fills the cell. A wider or taller item keeps the same square, fitted to the shorter side. There is no image column on `fw_items`. If the file is missing, the item is drawn with its name. Use a transparent background. The satchel draws the cell frame. Do not paint that into the picture.

The canvas below is optional. It is the slot at 1440p, doubled, for a picture drawn to fill one footprint edge to edge. A square file does not need these sizes. Pixels on an axis are `152 × cells − 32`. A 1×1 is 120×120. Each extra column adds 152 pixels of width. Each extra row adds 152 pixels of height.

Footprints run from 1×1 through 6×8. A turned item uses this same file in the swapped slot. The picture is not rotated.

| Width | 1 tall | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 120×120 | 120×272 | 120×424 | 120×576 | 120×728 | 120×880 | 120×1032 | 120×1184 |
| 2 | 272×120 | 272×272 | 272×424 | 272×576 | 272×728 | 272×880 | 272×1032 | 272×1184 |
| 3 | 424×120 | 424×272 | 424×424 | 424×576 | 424×728 | 424×880 | 424×1032 | 424×1184 |
| 4 | 576×120 | 576×272 | 576×424 | 576×576 | 576×728 | 576×880 | 576×1032 | 576×1184 |
| 5 | 728×120 | 728×272 | 728×424 | 728×576 | 728×728 | 728×880 | 728×1032 | 728×1184 |
| 6 | 880×120 | 880×272 | 880×424 | 880×576 | 880×728 | 880×880 | 880×1032 | 880×1184 |

## Exports

Call these from a server script after `fw_inventory` is ready. `AddEventHandler('fw_inventory:ready', function() ... end)` runs once the table and the item files are loaded. This resource calls `exports.fw_core:RegisterResource(1)` itself.

### RegisterItem

```lua
exports.fw_inventory:RegisterItem({
    name = 'bread',
    label = 'Bread',
    group = 'food',
    width = 1,
    height = 1,
    stack = 10,
    description = 'A loaf of bread.',
})
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `def` | `table` | yes | Same fields as an item file, plus `name`. |

Returns `true`, or `false, reason`. The row is written to `fw_items` before the satchel can use the name. Connected clients receive the new definition immediately. No server restart.

A later `fw_items_reload` updates that row when a file defines the same name. A row with no file is left in place.

### RegisterUsable

```lua
exports.fw_inventory:RegisterUsable('bread', function(source, item)
    exports.fw_inventory:RemoveItem(source, 'bread', 1)
end)
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `name` | `string` | yes | An item that already exists. |
| `callback` | `function` | yes | `function(source, item)`. |

`item` is a copy: `id`, `name`, `label`, `description`, `count`, `x`, `y`, `rotated`, `metadata`. Changing it does not change the satchel. Nothing is consumed unless the callback calls `RemoveItem`.

Returns `true`, or `false, reason`. If that resource stops, its usable callbacks are removed. Starting it again can register them again. That does not need a server restart.

### AddItem

```lua
local ok, err = exports.fw_inventory:AddItem(source, 'rope', 1)
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Player server id. The active character's satchel must be loaded. |
| `name` | `string` | yes | Registered item name. |
| `count` | `integer` | no | Defaults to 1. Max 500 per call. |
| `metadata` | `table` | no | Stored on new stacks. A stack only grows when the metadata matches. |

Finds room before it changes anything. Stacks that are not full, and are not weapons, are filled first. The rest is placed in the first free rectangle, scanning left to right, top to bottom. A rotatable item is tried turned if the first orientation does not fit. If the whole count does not fit, nothing is added. A weapon is never added onto another weapon. Two of the same weapon are two placements.

Returns `true`, or `false, reason`. On success the open satchel is refreshed.

### RemoveItem

```lua
local ok, err = exports.fw_inventory:RemoveItem(source, 'bread', 2)
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Player server id. |
| `name` | `string` | yes | Item name. |
| `count` | `integer` | yes | How many to remove. |

Takes from the last placement of that name first. If the satchel does not hold that many, nothing is removed.

Returns `true`, or `false, reason`.

### GetItems

```lua
local items, err = exports.fw_inventory:GetItems(source)
```

Returns a copy of the placements, or `nil, reason`. `items[n].count` is the stack. The array is not the live satchel.

### CanFit

```lua
local ok, err = exports.fw_inventory:CanFit(source, 'lantern', 1)
```

Same arguments as `AddItem` after `source`. Returns `true` when `AddItem` would succeed, or `false, reason`. It does not change the satchel.

### SetGridBonus

```lua
local ok, err = exports.fw_inventory:SetGridBonus(source, 'backpack', 0, 5)
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Player server id. The active character's satchel must be loaded. |
| `key` | `string` | yes | Bonus name, such as `backpack`. Lowercase letters, digits, and underscores, up to 32 characters. |
| `extraWidth` | `integer` | yes | Columns this bonus adds. `0` adds none. |
| `extraHeight` | `integer` | yes | Rows this bonus adds. `0` adds none. |

The drawn grid is the 10 by 10 base plus every bonus. A second call with the same key replaces that bonus. It does not stack on itself. Two different keys both apply. The grid cannot pass 24 by 24.

`0, 0` removes that key. `ClearGridBonus(source, key)` does the same.

```lua
local ok, err = exports.fw_inventory:ClearGridBonus(source, 'backpack')
```

Shrinking is refused while an item sits in a cell that would disappear. The satchel stays the larger size and the reason is `Make room in the satchel first.` Nothing is moved or dropped.

Returns `true`, or `false, reason`. On success the open satchel is refreshed.

### IsReady

Server export. True after the tables and the item files have loaded.

```lua
---@return boolean ready
exports.fw_inventory:IsReady()
```

```lua
if not exports.fw_inventory:IsReady() then
    return
end
```

`fw_inventory:ready` is the local event for the same moment. It is not queued. A listener that starts after the event has already fired should call `IsReady` instead.

### GetDefinition

Server export. Returns a copy of one item type, or `nil` when the name is not in the catalog.

```lua
---@param name string
---@return table|nil definition
exports.fw_inventory:GetDefinition(name)
```

```lua
local def = exports.fw_inventory:GetDefinition('herb_pouch')
if def == nil then
    return
end
```

The copy includes `name`, `label`, `description`, `width`, `height`, `stack`, `weapon`, `group`, `rotate`, `kit`, `hash`, and `container` when the item file defines one. Changing the table does not change the catalog.

### PlaceAt

Server export. Puts one new placement on an exact cell. It does not search for room.

```lua
---@param source integer
---@param name string
---@param count integer
---@param metadata table|nil
---@param x integer
---@param y integer
---@param rotated boolean
---@return boolean ok
---@return string|nil idOrErr
exports.fw_inventory:PlaceAt(source, name, count, metadata, x, y, rotated)
```

| Argument | Type | Required | Meaning |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Player server id. The active character's satchel must be loaded. |
| `name` | `string` | yes | Registered item name. |
| `count` | `integer` | no | Defaults to 1. Cannot pass that item's `stack`. |
| `metadata` | `table` | no | Stored on the placement. A container item receives `metadata.containerId` here. |
| `x`, `y` | `integer` | yes | Top-left cell. `0, 0` is the top left. |
| `rotated` | `boolean` | no | `true` turns it. Refused when the item cannot turn. |

The cells must be empty. Returns `true, id`, or `false, reason`. `That does not fit.` means the rectangle is outside the grid or another placement overlaps it. On success the open satchel is refreshed.

```lua
local ok, idOrErr = exports.fw_inventory:PlaceAt(source, 'rope', 1, nil, 0, 0, false)
if not ok then
    print(idOrErr)
    return
end
```

### PullForKit

Server export. Takes one placement out of the satchel for a kit slot. `fw_weapons` calls this when a player equips. Other resources should not need it.

```lua
---@param source integer
---@param id string
---@param displaced table|nil
---@return table|nil item
---@return string|nil err
exports.fw_inventory:PullForKit(source, id, displaced)
```

`id` is the placement id from `GetItems`. `displaced` is the item coming off that kit slot, with `name`, `count`, `metadata`, and `rotated`. It is placed on the cell the pulled item left. If that cell does not fit, it is placed anywhere that does. If neither works, nothing is pulled and the reason is `Make room in the satchel first.`

Returns the taken placement, or `nil, reason`. On success the open satchel is refreshed.

```lua
local item, err = exports.fw_inventory:PullForKit(source, placementId, nil)
if item == nil then
    print(err)
    return
end
```

### OpenStorage

Opens a second grid beside the satchel. The first call creates the row. A later call can grow it. The player must have a character selected.

```lua
---@param source integer
---@param id string
---@param params table
---@return boolean ok
---@return string|nil err
exports.fw_inventory:OpenStorage(source, id, params)
```

`id` is 1–64 characters: a letter or digit, then letters, digits, underscore, colon, or hyphen. A chest uses a stable id such as `chest:valentine_1`. A container item uses `item:` plus the placement's `metadata.containerId`.

`params.width` and `params.height` are the grid, from 1 to 24. `params.label` is the title, clipped to 48 characters. Omitted, it is `Storage`. `params.groups` is a list of item groups. Omit it to accept anything. An empty list accepts nothing. An unknown group returns `unknown group`.

Shrinking is refused while an item sits in a cell that would disappear. The reason is `Make room in there first.` One player holds a storage at a time. A second open returns `Someone is using that.` Opening a different storage closes the one this player already has. Drop, character clear, resource stop, and closing the satchel release the hold.

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

Returns `true`, or `false, reason`. On success the satchel opens with the second grid.

### CloseStorage

Closes the second grid and leaves the satchel open.

```lua
---@param source integer
---@return boolean closed
exports.fw_inventory:CloseStorage(source)
```

```lua
exports.fw_inventory:CloseStorage(source)
```

Returns `true` when this player had a storage open. Returns `false` when they did not.

## Container items

A container item is a placement that owns a storage row. `container` on the item file:

| Field | Meaning |
| --- | --- |
| `label` | Title of the second grid. Defaults to the item label. Clipped to 48 characters. |
| `width`, `height` | Grid size, 1–24. |
| `groups` | Groups the grid accepts. Omit it to accept anything. |

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

The first time that placement is created it receives `metadata.containerId`. Moving the pouch, including into another storage, keeps that id, so the herbs stay with it. Dropping it, or removing the whole placement, deletes the row `item:<containerId>`. Using the pouch in the satchel calls `OpenStorage` for that id. Using it while it sits in the open storage is refused with `Take it out first.`

`container` is not a column on `fw_items`. The item file is the source. `fw_items_reload` copies it back onto the catalog.

## Satchel

`fw_satchel` opens with B (`INPUT_OPEN_SATCHEL_MENU`) or `/satchel`. Escape or Tab puts it away.

The grid is the character's `width` by `height`. Drag a placement. It follows the cursor, and the cells under it turn green when it fits and red when it does not. The item moves the moment it is dropped. The server then confirms that move. A rejected drop puts the item back, and the line above the grid says why. While the item is held, R turns it when `rotate` is true. Dropping a stack exactly on the same item, with room left under that item's `stack`, combines them. A weapon never combines. `stack` 1 never combines.

Right-click a placement for Use, Drop, or Split. Drop removes that stack. Split peels half of a stack into another placement. Double-click uses it. If nothing registered a usable, the line above the grid says it cannot be used.

The icons on the left of the grid filter by group. The chosen group stays bright. Everything else is dimmed, and it stays where it was. The top icon shows everything. The icon at the bottom of that column packs the satchel from the top left in group order: ammo, animals, documents, food, herbs, horse, medical, tools, valuables, weapons, then materials. Larger items in a group are placed first. Stacks are not merged. If the new layout cannot fit, nothing moves.

A storage open adds a second panel to the left of the satchel, sized to that storage's own grid. The satchel and the loadout stay where they are. When the loadout is open, the storage panel sits to the left of it. Drag a placement onto the other grid to move it. Dropping it on the same footprint of the same item, with the same metadata and room under `stack`, combines them. A storage with `groups` refuses anything outside that list. The line above the grid says `That does not go in there.` The satchel itself stays unrestricted. The container item that owns the open storage cannot be placed into it. Close on that panel hides it and leaves the satchel open. Closing the satchel releases the storage. Right-click stays on the satchel. Double-click on the storage grid is refused until the item is back in the satchel.

## Commands

| Command | Who | Effect |
| --- | --- | --- |
| `/satchel` | The player | Opens or closes the satchel. |
| `fw_giveitem <serverId> <name> [count]` | Server console | `AddItem` for that player. |
| `fw_items_reload` | Server console, or an admin in the game | Writes `items/` into `fw_items`, loads every item row, and pushes definitions. |

## Events

| Event | Side | When |
| --- | --- | --- |
| `fw_inventory:ready` | Server, local (`AddEventHandler`) | After the table and item files are loaded. Not queued. A listener that starts later calls `IsReady`. |
| `fw_characters:selected` | Listens | Loads or creates that character's row. |
| `fw_characters:cleared` | Listens | Saves and drops the in-memory satchel. |

Network events `fw_inventory:open`, `fw_inventory:move`, `fw_inventory:use`, `fw_inventory:discard`, `fw_inventory:split`, `fw_inventory:sort`, `fw_inventory:storageClose`, `fw_inventory:storageMove`, and `fw_inventory:transfer` are the satchel screen talking to the server. Other resources should use the exports. `fw_inventory:storageClosed` is the server telling that screen to hide the second grid.
