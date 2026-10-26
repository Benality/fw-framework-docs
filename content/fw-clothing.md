# fw_clothing

Some calls run on the server. One runs in the player's game. Read the clothes a character is wearing, or replace them. The saved clothes live on the character. This resource writes them with `SetClothing` from [fw_characters](fw-characters.md).

The shop still uses the old `murphy_clothing` names. A new script should use the calls on this page.

There is no money resource yet. The shop shows prices and the purchase succeeds, but nothing is taken. Job-locked categories stay hidden because there is no job.

`fw_creator` must keep `Config.murphy_clothing` true.

## Clothing document

```lua
{
    clothes = {
        hats = { model = 12, texture = 1 },
    },
    outfitId = '1234567', -- '0' when no saved outfit is worn
    outfits = {},
    wearable = {},
}
```

`clothes` is what the ped is wearing. Hair and beard stay on the barber document, not here. `outfitId` is the saved outfit, or `'0'`.

A write only works while that player is online and has this character selected.

## Calls on the server

### IsBandanaEquipped

Runs on the server. Yes when the worn clothes include a mask with a model number above 0.

```lua
--- True when this player is wearing a mask or a large mask.
---@param source integer
---@return boolean
exports.fw_clothing:IsBandanaEquipped(source)
```

```lua
if exports.fw_clothing:IsBandanaEquipped(source) then
    -- hide the face from other players
end
```

### GetPlayerClothes

Runs on the server. The worn clothes, turned into the numbers the game uses to draw them. You get an empty result when nothing is saved, or when a clothing slot has no matching picture in the catalog.

```lua
--- Worn clothes as asset hashes, keyed by category.
---@param source integer
---@return table clothes
exports.fw_clothing:GetPlayerClothes(source)
```

Each category looks like `{ hash, drawable, albedo, normal, material, palette, tint0, tint1, tint2 }`.

```lua
local clothes = exports.fw_clothing:GetPlayerClothes(source)
local hat = clothes.hats
```

### GetPlayerTemperatureProtection

Runs on the server. How many warm pieces and cold pieces are worn, and the protection numbers from the clothing settings.

```lua
--- Warm and cold clothing counts for the active character.
---@param source integer
---@return table protection
exports.fw_clothing:GetPlayerTemperatureProtection(source)
```

```lua
{
    warmItems = 0,
    coldItems = 0,
    coldProtection = 3,
    heatProtection = 3,
}
```

```lua
local protection = exports.fw_clothing:GetPlayerTemperatureProtection(source)
print(protection.coldProtection)
```

### SetClothesForChar

Runs on the server. Saves clothes for the character that is selected right now, then puts them on that player. This is the call a job or uniform script should use.

```lua
--- Saves worn clothes for the online player who has this character selected.
--- False and a reason when the character is offline or an entry is not a model and texture.
---@param charid integer|string
---@param clothes table
---@param outfitId integer|string|nil
---@return boolean ok
---@return string|nil err
exports.fw_clothing:SetClothesForChar(charid, clothes, outfitId)
```

`charid` is `fw_characters.id`. `clothes` is `{ hats = { model = 12, texture = 1 } }`. `model` is the catalog index. `texture` is a number, or a table `{ palette, tint0, tint1, tint2 }`. Hair and beard are removed from the table before it is saved. Leave `outfitId` out to store no saved outfit.

| `err` | When |
| --- | --- |
| `SetClothesForChar: charid is required` | The character id is missing. |
| `SetClothesForChar: clothes must be a table { category = { model, texture } }` | `clothes` is not a table. |
| `SetClothesForChar: invalid entry for category '<category>' ...` | That category has no numeric `model`. |
| `SetClothesForChar: invalid texture for category '<category>'` | `texture` is not a number or a table. |
| `SetClothesForChar: that character is not selected by an online player` | Nobody online is playing that character. |
| `SetClothesForChar: could not save clothing` | `SetClothing` failed. |

```lua
local ok, err = exports.fw_clothing:SetClothesForChar(characterId, {
    hats = { model = 12, texture = 1 },
}, nil)
if not ok then
    print(err)
    return
end
```

## A call in the player's game

### AreClothesStillApplied

Runs in the player's game. Yes when the clothes this resource last put on them are still there. No if clothes were never loaded this visit, or one slot is missing. The character menu uses this after it applies a body, to see if the clothes were wiped off.

```lua
--- True when the last clothes this resource applied are still on the ped.
---@return boolean
exports.fw_clothing:AreClothesStillApplied()
```

```lua
if not exports.fw_clothing:AreClothesStillApplied() then
    TriggerEvent('murphy_clothing:loadclothes')
end
```

## Messages

`murphy_clothing:clothesUpdated` stays on the server. It is sent after clothes are saved, from the shop or from `SetClothesForChar`. Your script hears it only if it is already running. Put the listener at the top of your script.

```lua
---@param source integer
---@param charid integer
---@param clothes table
AddEventHandler('murphy_clothing:clothesUpdated', function(source, charid, clothes)
end)
```

`murphy_clothing:loadclothes` is sent from the server to the player's game. Their game puts the saved clothes back on. The character menu sends it after a body is applied. Your script hears it only if it is already running.

The shop's other `murphy_clothing` and `murphy_clothes` messages are the menu talking to itself. Use the calls above from a new script.

## Commands

| Command | Who | What it does |
| --- | --- | --- |
| `/loadclothes` | The player | Puts the saved clothes back on the ped. |
| `/ClothesManagement` | An account whose role is `admin` or `superadmin` | Opens the clothes management menu. |
