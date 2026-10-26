# fw_menu

Client. Open a menu on the player's screen. This resource does not know about accounts, characters, or items.

## Client export

### GetMenuData

Client. The menu API. Call it once from the resource that opens menus.

```lua
--- The menu API for this client.
---@return table MenuData
local MenuData = exports.fw_menu:GetMenuData()
```

The same table is passed to the local client event `fw_menu:getData`. Listen with `AddEventHandler`. It is not saved. Your script must already be running when the event fires.

```lua
AddEventHandler('fw_menu:getData', function(MenuData)
end)
```

## Open a menu

```lua
--- Opens a menu and returns the menu handle passed into the callbacks.
---@param menuType string use 'default'
---@param namespace string your resource or feature name
---@param name string menu name inside that namespace
---@param data table title, elements, and options
---@param submit function|nil called with (data, menu) when the player confirms
---@param cancel function|nil called with (data, menu) when the player backs out
---@param change function|nil called with (data, menu) when the highlight or a slider moves
---@param close function|nil called with no arguments after the menu has closed
---@return table menu
MenuData.Open(menuType, namespace, name, data, submit, cancel, change, close)
```

`menuType` is `default`. That is the only built-in type. Pair `namespace` and `name` so two menus do not collide.

`data`:

| Field | Meaning |
| --- | --- |
| `title` | Header. |
| `subtext` | Line under the header. |
| `elements` | Rows, in order. |
| `isGrid` | Icon grid instead of a list. |
| `disableMovement` | The player cannot walk while this menu is open. Nested menus share one freeze and release it when the last one closes. |
| `lockInventory` | Sets `LocalPlayer.state.fwInventoryBusy` to true while this menu is open, and back to false when the last locked menu closes. `fw_satchel` should refuse to open while that is true. |

A row:

| Field | Meaning |
| --- | --- |
| `label` | Text on the row. |
| `value` | What you get back when the player confirms. |
| `desc` | Text shown for the highlighted row. |
| `type` | Leave it out for a normal row. `slider` for a slider. |
| `min`, `max`, `hop` | Slider range and step. `hop` defaults to 1. |
| `options` | Labels for a slider that is not a number range. `value` is the index, starting at 0. |
| `image` | Icon name when `isGrid` is true. |

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

## Menu handle

The callbacks receive `menu`. `GetOpened` returns the same kind of table.

| Call | What it does |
| --- | --- |
| `menu.close()` | Closes this menu and runs its close callback. |
| `menu.refresh()` | Sends the current rows to the screen again. |
| `menu.setElement(index, key, value)` | Changes one field. `index` starts at 1. |
| `menu.setElements(elements)` | Replaces the row list. |
| `menu.setTitle(title)` | Changes the header. |
| `menu.update(query, newData)` | On every row whose fields match `query`, copies the fields from `newData`. |
| `menu.addNewElement(element)` | Adds a row. |
| `menu.removeElement(query)` | Removes every row whose fields match `query`. |
| `menu.removeElementByValue(value, stop)` | Removes rows with that `value`. `stop` true removes only the first. |
| `menu.removeElementByIndex(index)` | Removes that row. `index` starts at 1. |

## Other calls

| Call | Returns |
| --- | --- |
| `MenuData.IsOpen(menuType, namespace, name)` | True when that menu is open. |
| `MenuData.GetOpened(menuType, namespace, name)` | The open menu, or nil. |
| `MenuData.GetOpenedMenus()` | A list of open menus. |
| `MenuData.Close(menuType, namespace, name)` | Closes that menu. |
| `MenuData.CloseAll()` | Closes every open menu. |
| `MenuData.ReOpen(menu)` | Opens a menu that was closed, using a table from `GetOpenedMenus`. |

## Events

`fw_menu:closed` is a local client event. Listen with `AddEventHandler`. It fires when the player closes the screen with the back control. It has no arguments. It is not saved. Your script must already be running.

## Keys

Enter confirms the highlighted row. Backspace cancels. Up and Down move the highlight. Left and Right change a slider.
