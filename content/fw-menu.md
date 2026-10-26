# fw_menu

Runs in the player's game. Open a menu on their screen. This does not know about accounts, characters, or items.

## Get the menu

### GetMenuData

Runs in the player's game. This is the menu. Ask for it once, in the script that opens menus.

```lua
--- The menu for this player's game.
---@return table MenuData
local MenuData = exports.fw_menu:GetMenuData()
```

The same menu is also handed to the message `fw_menu:getData`. That message stays in the player's game. Your script hears it only if it is already running. Put the listener at the top of your script.

```lua
AddEventHandler('fw_menu:getData', function(MenuData)
end)
```

## Open a menu

Opens a list on the player's screen. You get back a handle. The functions you pass in receive that same handle as `menu`.

```lua
--- Opens a menu. You get back the handle the functions below receive as menu.
---@param menuType string use 'default'
---@param namespace string your script or feature name
---@param name string menu name inside that name
---@param data table title, rows, and options
---@param submit function|nil runs with (data, menu) when the player confirms
---@param cancel function|nil runs with (data, menu) when the player backs out
---@param change function|nil runs with (data, menu) when the highlight or a slider moves
---@param close function|nil runs with no arguments after the menu has closed
---@return table menu
MenuData.Open(menuType, namespace, name, data, submit, cancel, change, close)
```

`menuType` is `default`. That is the only built-in kind. Use `namespace` and `name` together so two menus do not get mixed up.

`data`:

| Field | Meaning |
| --- | --- |
| `title` | Header. |
| `subtext` | Line under the header. |
| `elements` | Rows, in order. |
| `isGrid` | Icon grid instead of a list. |
| `disableMovement` | The player cannot walk while this menu is open. Nested menus share one freeze and release it when the last one closes. |
| `lockInventory` | While this menu is open, the satchel should stay closed. It sets `LocalPlayer.state.fwInventoryBusy` to true, and back to false when the last menu that asked for this has closed. |

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

The functions you passed to `Open` receive `menu`. `GetOpened` gives you the same kind of handle.

| Call | What it does |
| --- | --- |
| `menu.close()` | Closes this menu and runs the close function you gave `Open`. |
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

## When the player closes the menu

`fw_menu:closed` stays in the player's game. Your script hears it only if it is already running. It has no extra information. It is sent when the player backs out of the screen.

## Keys

Enter confirms the highlighted row. Backspace cancels. Up and Down move the highlight. Left and Right change a slider.
