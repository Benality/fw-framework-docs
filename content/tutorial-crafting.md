# Tutorial: a crafting resource

This builds one resource, `fw_crafting`, from nothing. A player types `/craft` and, if they are carrying 3 Plant Fiber, it becomes 1 Rope. One recipe, kept small on purpose. The patterns here are the same ones you would use for ten recipes, or for giving items from any other kind of resource.

Read [Write a script](README.md#write-a-script) first if you have not. This tutorial is that pattern, with a real feature built on top of it.

## What it needs running first

`fw_crafting` calls [fw_core](fw-core.md), [fw_characters](fw-characters.md), and [fw_inventory](fw-inventory.md), and uses [vorp_lib](vorp-lib.md) for logging. All four have to already be in `server.cfg` above it:

```
ensure vorp_lib
ensure fw_core
ensure fw_characters
ensure fw_inventory
ensure fw_crafting
```

## The folder

```
resources/[framework]/fw_crafting/
    fxmanifest.lua
    client/main.lua
    server/main.lua
```

## The manifest

```lua
fx_version 'cerulean'
game 'rdr3'

rdr3_warning 'I acknowledge that this is a prerelease build of RedM, and I am aware my resources *will* become incompatible once RedM ships.'

name 'fw_crafting'
author 'Project Territory'
description 'Craft Rope from Plant Fiber'
version '1.0.0'

lua54 'yes'

dependencies {
    'fw_core',
    'fw_characters',
    'fw_inventory',
    'vorp_lib',
}

shared_scripts {
    '@vorp_lib/import.lua',
}

client_scripts {
    'client/main.lua',
}

server_scripts {
    'server/main.lua',
}
```

`dependencies` does not start those resources for you. It only stops `fw_crafting` from starting before them. The `ensure` order above still matters.

There is no `fw_crafting` database table. The recipe is just logic; the satchel that holds the items is already saved by `fw_inventory`. A resource that only moves items other resources already track does not need a table of its own.

## Add the ingredient

Rope already exists — it is in `fw_inventory`'s own item list. Plant Fiber does not, so `fw_crafting` registers it with [RegisterItem](fw-inventory.md#registeritem) the first time it starts:

```lua
local itemOk, itemErr = exports.fw_inventory:RegisterItem({
    name = 'plant_fiber',
    label = 'Plant Fiber',
    category = 'materials',
    width = 1,
    height = 1,
    stack = 20,
    description = 'Stripped fiber from a yucca or similar plant. Good for cordage.',
})
if not itemOk then
    print('[fw_crafting] ' .. tostring(itemErr))
end
```

`category` has to be one already in use: `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `currency`, `weapons`, `materials`, or `container`. Calling `RegisterItem` again on every restart is normal — it replaces the same definition, it does not fail on the second call.

This item has no `image`, so the satchel screen falls back to a blank tile. To give it a picture, drop a PNG named after the item, `plant_fiber.png`, into `fw_inventory/images/`. That is an asset, not a code change, so it does not touch anything `fw_inventory` owns in script.

## The recipe and the crafting logic

Keep recipes in a table, even with only one in it:

```lua
local RECIPES = {
    rope = {
        result = 'rope',
        resultCount = 1,
        ingredients = {
            plant_fiber = 3,
        },
    },
}
```

Crafting touches the satchel twice — take the ingredients, then add the result — and those are two separate calls to `fw_inventory`. Nothing stops a player from being one item short on the second call. Handle both directions:

- Check every ingredient count **before** removing any of them. A recipe with two ingredients should never take the first and then fail on the second.
- If adding the result fails — the satchel is full — give the ingredients back. The player should not lose materials for a craft that did not produce anything.

```lua
--- How many of name the player is carrying, from a GetItems snapshot.
---@param items table[]
---@param name string
---@return integer
local function countOf(items, name)
    local total = 0
    for index = 1, #items do
        if items[index].name == name then
            total = total + (items[index].count or 0)
        end
    end
    return total
end

--- Takes every ingredient for a recipe, or none of them.
---@param source integer
---@param recipe table
---@return boolean ok
---@return string|nil err
local function takeIngredients(source, recipe)
    local items, err = exports.fw_inventory:GetItems(source)
    if items == nil then
        return false, err
    end

    for name, needed in pairs(recipe.ingredients) do
        if countOf(items, name) < needed then
            return false, 'You do not have enough materials.'
        end
    end

    local taken = {}
    for name, needed in pairs(recipe.ingredients) do
        local ok, removeErr = exports.fw_inventory:RemoveItem(source, name, needed)
        if not ok then
            for takenName, takenCount in pairs(taken) do
                exports.fw_inventory:AddItem(source, takenName, takenCount)
            end
            return false, removeErr
        end
        taken[name] = needed
    end

    return true, nil
end

--- Runs one craft attempt. Ingredients come back if the result cannot fit.
---@param source integer
---@param recipeName string
---@return boolean ok
---@return string message
local function craft(source, recipeName)
    local recipe = RECIPES[recipeName]
    if recipe == nil then
        return false, 'That is not something you know how to make.'
    end

    local took, takeErr = takeIngredients(source, recipe)
    if not took then
        return false, takeErr or 'Could not take the materials.'
    end

    local added, addErr = exports.fw_inventory:AddItem(source, recipe.result, recipe.resultCount)
    if not added then
        for name, needed in pairs(recipe.ingredients) do
            exports.fw_inventory:AddItem(source, name, needed)
        end
        return false, addErr or 'Could not carry the result.'
    end

    return true, ('Crafted %d %s.'):format(recipe.resultCount, recipe.result)
end
```

`GetItems` and `RemoveItem` already say no when the player has no character selected, or does not hold enough — `fw_crafting` does not repeat those checks itself. Read the call before you copy it elsewhere; do not assume what it rejects.

## The server event

The client only ever sends a recipe **name**, never a count or an item. Counts come from the `RECIPES` table on the server, where the player cannot change them.

```lua
RegisterNetEvent('fw_crafting:craft', function(recipeName)
    local src = source

    if not ready then
        TriggerClientEvent('fw_crafting:result', src, false, 'Crafting is not ready yet.')
        return
    end

    local ok, message = craft(src, recipeName)
    TriggerClientEvent('fw_crafting:result', src, ok, message)
end)
```

`local src = source` reads the player number off the event itself, not off an argument. A `TriggerServerEvent('fw_crafting:craft', recipeName)` call cannot forge who it is — `source` is set by FXServer before your handler runs. If this line were instead a `source` argument the client passed in, any player could craft on behalf of any other player's inventory by sending someone else's number.

## The rest of server/main.lua

This is the same shape as every other `fw_*` resource: wait for what you depend on, register the item, check in, then say you are ready.

```lua
local Logger = Import('logger').Logger
local LOG_OPTIONS = { prefix = 'fw_crafting' }

local ready = false

local RECIPES = {
    rope = {
        result = 'rope',
        resultCount = 1,
        ingredients = {
            plant_fiber = 3,
        },
    },
}

local function log(message, level)
    if level == 'info' then
        Logger:Info(message, LOG_OPTIONS)
        return
    end
    Logger:Error(message, LOG_OPTIONS)
end

local function coreReady()
    local ok, isReady = pcall(function() return exports.fw_core:IsReady() end)
    return ok and isReady == true
end

local function charactersReady()
    local ok, isReady = pcall(function() return exports.fw_characters:IsReady() end)
    return ok and isReady == true
end

local function inventoryReady()
    local ok, isReady = pcall(function() return exports.fw_inventory:IsReady() end)
    return ok and isReady == true
end

-- countOf, takeIngredients, craft go here, as above

RegisterNetEvent('fw_crafting:craft', function(recipeName)
    -- as above
end)

--- Gives test materials from the server console, or to another player by number.
--- Not for players: RegisterCommand's third argument restricts it to an ace.
RegisterCommand('fw_crafting_give', function(source, args)
    local target = tonumber(args[1]) or source
    exports.fw_inventory:AddItem(target, 'plant_fiber', 10)
end, true)

local function waitUntil(check, label)
    if check() then
        return true
    end
    local timeoutAt = GetGameTimer() + 120000
    while not check() do
        if GetGameTimer() >= timeoutAt then
            log(label .. ' did not become ready')
            return false
        end
        Wait(100)
    end
    return true
end

if not waitUntil(coreReady, 'fw_core') then return end
if not waitUntil(charactersReady, 'fw_characters') then return end
if not waitUntil(inventoryReady, 'fw_inventory') then return end

local registered, registerErr = exports.fw_core:RegisterResource(1)
if not registered then
    log(registerErr or 'fw_core registration failed')
    return
end

local itemOk, itemErr = exports.fw_inventory:RegisterItem({
    name = 'plant_fiber',
    label = 'Plant Fiber',
    category = 'materials',
    width = 1,
    height = 1,
    stack = 20,
    description = 'Stripped fiber from a yucca or similar plant. Good for cordage.',
})
if not itemOk then
    log(itemErr or 'could not register plant_fiber')
end

ready = true
log('ready.', 'info')
```

`fw_crafting_give` is restricted the same way `fw_core`'s own `fw_health` command is: the server console can always run it, and a player can only run it in-game if you grant the ace. Console is enough for testing. If you do want it in-game for admins, add this to `permissions.cfg`:

```
add_ace group.admin command.fw_crafting_give allow
```

`fw_crafting` keeps no state of its own between restarts — no table in memory keyed by player, nothing to walk on `onResourceStop`. When a resource only calls out to other resources and keeps nothing itself, there is nothing to clean up. That will not stay true once a resource tracks its own state, and `fw_core`'s own `onResourceStop` handler, which forgets a resource's registration when it stops, is what that cleanup looks like.

## client/main.lua

The client sends the command and prints whatever the server decides happened. It does not touch items, and it does not decide whether the craft worked.

```lua
local Logger = Import('logger').Logger
local LOG_OPTIONS = { prefix = 'fw_crafting' }

--- Writes one line with vorp_lib on this player's game. level is error or info.
---@param message string
---@param level string|nil
---@return nil
local function say(message, level)
    if level == 'info' then
        Logger:Info(message, LOG_OPTIONS)
        return
    end
    Logger:Error(message, LOG_OPTIONS)
end

RegisterCommand('craft', function(_, args)
    local recipe = args[1] or 'rope'
    TriggerServerEvent('fw_crafting:craft', recipe)
end, false)

RegisterNetEvent('fw_crafting:result', function(ok, message)
    say(message, ok and 'info' or nil)
end)
```

This is the same `say` helper [fw_trade](fw-trade.md) uses for its own rejections. There is no on-screen toast in this framework yet — feedback is a line in the player's own F8 console. Match that instead of inventing a second way to talk to the player.

`args[1]` lets `/craft rope` name a recipe. With one recipe it is not load-bearing, but it means adding a second recipe to `RECIPES` does not also require a second command.

## Try it

```
ensure vorp_lib
ensure fw_core
ensure fw_characters
ensure fw_inventory
ensure fw_crafting
```

Start the server. The console should show:

```
[fw_core][info] ready version=1.1.0 api=1
[fw_characters][info] ready version=1.0.0
[fw_inventory] ready. 6 items.
[fw_crafting] ready.
```

In the server console, give yourself fiber and craft:

```
fw_crafting_give <your server id>
```

In the player's game, open the satchel and confirm 10 Plant Fiber are there, then type:

```
/craft
```

The console (F8) should print `Crafted 1 Rope.`, and the satchel should show 7 Plant Fiber and 1 Rope. Typing `/craft` again with fewer than 3 Plant Fiber should print `You do not have enough materials.` and change nothing.

## Where to go from here

- More recipes are more entries in `RECIPES`. The crafting logic does not change.
- Right now any player can type `/craft` from anywhere. A real crafting station would check distance to a prop, or an animation finishing, before calling `craft`. That check belongs on the server, next to the `RECIPES` lookup — not on the client, which can only ever be a suggestion.
- A player crafting as fast as they can press Enter is not covered here. A small per-player cooldown, checked in the same place `ready` is checked, is the next thing to add before this goes near real players.
- If the crafted item should do something when used — like `dried_meat` does in [fw_inventory](fw-inventory.md#registerusable) — register that separately with `RegisterUsable`. Crafting an item and using an item are two different calls.
