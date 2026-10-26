# fw_menu

Version 1.0.0. Client only. Resource name `fw_menu`.

Source: `resources/[framework]/fw_menu`. The screen layout comes from the RSG menu base. There is no server script, no `rsg-core` call, and no version check against GitHub.

Other resources open menus through the export. `fw_menu` does not know about players, characters, or items.

## Export

Client side.

```lua
---@return table MenuData
local MenuData = exports.fw_menu:GetMenuData()
```

The table is the live menu API. Call it once from the resource that opens menus.

The same table is passed to `fw_menu:getData` on the client:

```lua
TriggerEvent('fw_menu:getData', function(MenuData)
end)
```

## `MenuData.Open`

```lua
MenuData.Open(menuType, namespace, name, data, submit, cancel, change, close)
```

| Argument | Type | Meaning |
| --- | --- | --- |
| `menuType` | string | `default`. That is the only built-in type. |
| `namespace` | string | Resource or feature name. Pair it with `name` so two menus do not collide. |
| `name` | string | Menu name inside that namespace. |
| `data` | table | What the screen shows. Fields below. |
| `submit` | function or nil | Called with `(data, menu)` when the player confirms an item. |
| `cancel` | function or nil | Called with `(data, menu)` when the player backs out. |
| `change` | function or nil | Called with `(data, menu)` when the highlight or a slider moves. |
| `close` | function or nil | Called with no arguments after the menu has closed. |

`data`:

| Field | Type | Meaning |
| --- | --- | --- |
| `title` | string | Header. |
| `subtext` | string | Line under the header. |
| `elements` | table | Items, in order. |
| `isGrid` | boolean | Icon grid instead of a list. |
| `disableMovement` | boolean | Freezes the player while this menu is open. Nested menus share one freeze and release it when the last one closes. |
| `lockInventory` | boolean | Sets local state `fwInventoryBusy` to `true` while this menu is open, and back to `false` when the last locked menu closes. |

An element:

| Field | Meaning |
| --- | --- |
| `label` | Text on the row. |
| `value` | Value passed back on submit. Any Lua value the NUI can send. |
| `desc` | Description shown for the highlighted row. |
| `type` | Omit it for a normal row. `slider` for a slider. |
| `min`, `max`, `hop` | Slider range and step. `hop` defaults to 1 when omitted by the caller. |
| `options` | List of labels for a slider that is not a number range. `value` is the index, starting at 0. |
| `image` | Grid icon name when `isGrid` is true. |

`Open` returns the menu table. The callbacks receive that same table as `menu`.

```lua
local MenuData = exports.fw_menu:GetMenuData()

MenuData.Open('default', 'myshop', 'main', {
    title = 'Shop',
    subtext = 'Select an item',
    disableMovement = true,
    elements = {
        { label = 'Bread', value = 'bread', desc = 'Fresh bread' },
        { label = 'Amount', type = 'slider', value = 1, min = 1, max = 10, hop = 1 },
    },
}, function(data, menu)
    print(data.current.value)
    menu.close()
end, function(data, menu)
    menu.close()
end)
```

## Menu methods

These exist on the `menu` value passed into the callbacks, and on the table from `MenuData.GetOpened`.

| Method | Arguments | Effect |
| --- | --- | --- |
| `close` | none | Closes this menu and runs its `close` callback. |
| `refresh` | none | Sends the current elements to the screen again. |
| `setElement` | `index`, `key`, `value` | Changes one field on one element. `index` starts at 1. |
| `setElements` | element array | Replaces the element list. |
| `setTitle` | `title` | Changes the header. |
| `update` | `query`, `newData` | On every element whose fields match `query`, copies the fields from `newData`. |
| `addNewElement` | element table | Appends an element. |
| `removeElement` | `query` | Removes every element whose fields match `query`. |
| `removeElementByValue` | `value`, `stop` | Removes elements with that `value`. `stop` true removes only the first match. |
| `removeElementByIndex` | `index` | Removes that element. `index` starts at 1. |

## Other `MenuData` functions

| Function | Arguments | Returns |
| --- | --- | --- |
| `IsOpen` | `menuType`, `namespace`, `name` | `true` when that menu is open. |
| `GetOpened` | `menuType`, `namespace`, `name` | The open menu, or nil. |
| `GetOpenedMenus` | none | Array of open menus. |
| `Close` | `menuType`, `namespace`, `name` | Closes that menu. |
| `CloseAll` | none | Closes every open menu. |
| `ReOpen` | menu table | Opens a menu that was closed, using the table `GetOpenedMenus` returned. |

Closing the NUI from the close control fires the client event `fw_menu:closed` with no arguments.

## Controls

| Key | Action |
| --- | --- |
| Enter | Confirm the highlighted item. |
| Backspace | Cancel. |
| Up / Down | Move the highlight. |
| Left / Right | Change a slider. |

The control thread waits 200 ms while no menu is open, and runs every frame while one is open.
