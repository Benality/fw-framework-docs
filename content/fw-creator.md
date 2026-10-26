# fw_creator

The character menu. Players use it to make a character and to play one. Another script should not call into this menu. Create and read characters with [fw_characters](fw-characters.md).

`Config.framework` in `shared/config.lua` must stay `fw`. Anything else and character select never answers.

The number of characters an account can create is `characterSlots` on the account, not `Config.Slots` in this menu.

## Where the menu saves things

| What the player enters | Where it goes |
| --- | --- |
| Name | `firstName` and `lastName` |
| Gender | `male` or `female` |
| Birthday | `birthDate` as `YYYY-MM-DD` |
| Lore | `description` |
| Chosen spawn | `SaveState`, right after the character is selected |
| Body and face | `appearance` |
| Hair and makeup | `barber` |
| Clothes | [fw_clothing](fw-clothing.md), not this menu |

`Config.murphy_clothing` must stay true while `fw_clothing` is running.

## Body document

`SetAppearance` stores the table the menu already built. These are the keys other code reads:

| Key | Meaning |
| --- | --- |
| `gender` | `Male` or `Female`. |
| `skintone` | Skin tone index. |
| `pedmodel.model` | Model name, such as `mp_male`. |
| `pedmodel.outfit` | Outfit preset on that model. |
| `head`, `upperbody`, `lowerbody` | Body indexes. |

Replace the document by sending the whole table. Do not write one key by itself and expect the rest to remain unless you read the current document first and send it back with your change.

## Hair document

```lua
{
    hairstyle = {},
    overlays = {},
    permanentoverlay = {},
    outfitId = '1230456',
    presets = {
        {
            hairstyle = {},
            overlays = {},
            name = 'Base',
            outfitId = '1230456',
            price = 0,
            gender = 'male',
        },
    },
}
```

The first save during character creation uses the preset name `Base`.

## Health sent to the menu

When a character is selected, the client receives:

```lua
{
    hOuter = 600, -- character.health
    hInner = 100, -- character.healthCore
    sOuter = 100, -- always 100; there is no second stamina field
    sInner = 100, -- character.staminaCore
}
```

That message is `murphy_creator:HealthFromCore`. The server sends it to the player's game. This menu does not save health when the player leaves. Use `SaveState` on [fw_characters](fw-characters.md) for that.

## Messages the menu uses

These keep the old `murphy_creator` names so the screen did not have to be renamed. A new script should listen to the messages on [fw_characters](fw-characters.md) instead.

These go between the player's game and the server. Your script hears one only if it is already running.

| Event | Direction | What it does |
| --- | --- | --- |
| `murphy_creator:getCharacters` | Client to server | Asks for the character list. |
| `murphy_creator:LaunchCharSelect` | Server to client | Opens the list. |
| `murphy_creator:CharSelectFailed` | Server to client | A reason string. The client prints it and tries once more. |
| `fw_creator:selectCharacter` | Client to server | Selects that character id, then sends health. |
| `murphy_creator:deleteCharacter` | Client to server | The argument is the index in the list last sent to this player, not a character id. |
| `murphy_creator:HealthFromCore` | Server to client | The health table above. |

`fw_characters:selected` is the event to use when your resource needs to know a character was chosen. A spawn script listens there, not on `fw_creator:selectCharacter`:

```lua
AddEventHandler('fw_characters:selected', function(source, character)
    print(character.id)
end)
```

`murphy_creator:HealthFromCore` is the health table above. The server sends it to the player's game. Listen only if you are drawing your own health bar from that menu:

```lua
RegisterNetEvent('murphy_creator:HealthFromCore', function(health)
    print(health.hOuter)
end)
```

The menu also uses these. A new script should keep using [fw_characters](fw-characters.md). They are listed so an older script can be matched to a name.

| Message | Direction | What it does |
| --- | --- | --- |
| `murphy_creator:GetHealthValues` | Player's game to server | Asks for the health table. The answer is `murphy_creator:HealthFromCore`. |
| `murphy_creator:OpenCharSelect` | Either way | Opens the character list again. |
| `murphy_creator:LaunchCreator` | Player's game | Opens the body editor for a new character. |
| `murphy_creator:deleteCharacterClient` | Server to player's game | The list index that was just deleted. |
| `murphy_creator:SecondChance` | Server to player's game | Opens the body editor again. `/secondchance` sends this. |
| `fw_creator:characterSelected` | Player's game | The menu's own note that a character was picked. Then it loads the body. Prefer `fw_characters:selected`. |
| `murphy_creator:loadskin` | Player's game | Puts the saved body back on. `/mc` does this. |
| `murphy_creator:loadskinvorp` | Player's game | The same reload, used after the model is already the right one. |
| `murphy_creator:localExpressionsReady` | Player's game | The face sliders are ready to share. The menu publishes them. |
| `murphy_creator:client:skinApplied` | Player's game | The body finished applying. The argument is the character in the game. |
| `murphy_creator:client:characterCreated` | Player's game | The new character was just saved by the menu. |
| `murphy_barber_creator:loadbarberoverlay` | Player's game | Puts the saved hair and makeup back on. |
| `murphy_barber_creator:loadbarberoverlayOnCharacter` | Player's game | Same, for a character id you pass. |
| `murphy_barber_creator:updatehairstyle` | Player's game to server | The barber saves hair. A new script uses `SetBarber` on fw_characters. |
| `murphy_barber_creator:clotheitem` | Player's game | The barber applies one hair piece. |
| `murphy_creator:triggerCallback` | Server to player's game | The menu's private question. A new script does not send this. |
| `murphy_creator:responseCallback` | Player's game to server | The answer to that question. A new script does not send this. |

A script that needs to know the body is on them listens for the apply message:

```lua
AddEventHandler('murphy_creator:client:skinApplied', function()
    exports.fw_creator:ReassertLocalExpressions()
end)
```

## Calls in the player's game

Another script should still create and read characters with [fw_characters](fw-characters.md). These calls are for a script that changes the body the player is looking at, such as an animal form.

### SetAppearancePaused

Runs in the player's game. Pass `true` before you change them into something that is not their character, such as a wolf. The character menu stops painting the human face on top. Pass `false` when they are themselves again.

This also sets `LocalPlayer.state.murphy_appearance_paused`. Only this player's game can read that value.

```lua
--- Pauses face and clothes updates on this player's game.
---@param paused boolean
exports.fw_creator:SetAppearancePaused(paused)
```

```lua
exports.fw_creator:SetAppearancePaused(true)
```

### IsAppearancePaused

Runs in the player's game. Yes while that pause is on.

```lua
--- True while appearance updates are paused on this computer.
---@return boolean
exports.fw_creator:IsAppearancePaused()
```

```lua
if exports.fw_creator:IsAppearancePaused() then
    return
end
```

### IsPedModelCharacter

Runs in the player's game. Yes when the loaded character is a custom model, not `mp_male` or `mp_female`. A barber script uses this so it does not paint skin and makeup onto a copied face.

```lua
--- True when this character is not mp_male or mp_female.
---@return boolean
exports.fw_creator:IsPedModelCharacter()
```

```lua
if exports.fw_creator:IsPedModelCharacter() then
    return
end
```

### GetCachedExpressions

Runs in the player's game. The face sliders, or nothing when none are loaded. The keys are the expression names the game uses. Changing the table does not change the face. Call `ReassertLocalExpressions` after you want them drawn again.

```lua
--- The face sliders, or nothing when none are loaded.
---@return table|nil expressions
exports.fw_creator:GetCachedExpressions()
```

```lua
local expressions = exports.fw_creator:GetCachedExpressions()
if expressions == nil then
    return
end
```

### PublishLocalExpressions

Runs in the player's game. Copies the current face sliders into `murphy_expressions` so other players' games can draw the same face. The menu already does this. Call it after your script changes those sliders.

```lua
--- Shares the local face sliders with other players.
exports.fw_creator:PublishLocalExpressions()
```

```lua
exports.fw_creator:PublishLocalExpressions()
```

### ClearLocalExpressions

Runs in the player's game. Clears `murphy_expressions`. Call it when this player leaves the character, so the next person nearby does not keep the old face.

```lua
--- Clears the shared face sliders.
exports.fw_creator:ClearLocalExpressions()
```

```lua
exports.fw_creator:ClearLocalExpressions()
```

### PublishLocalFaceComps

Runs in the player's game. Shares the eyes and teeth so a player who walks up later still sees them. The menu already does this.

```lua
--- Shares eyes and teeth with other players.
exports.fw_creator:PublishLocalFaceComps()
```

```lua
exports.fw_creator:PublishLocalFaceComps()
```

### ReassertLocalExpressions

Runs in the player's game. Draws the saved face sliders on the local character again. Call it right after your script rebuilds that character. You get no while appearance is paused, while there are no sliders, or while the character is missing.

```lua
--- Draws the saved face sliders again. False when there is nothing to draw.
---@return boolean ok
exports.fw_creator:ReassertLocalExpressions()
```

```lua
exports.fw_creator:ReassertLocalExpressions()
```

### GetMurphyAssets

Runs in the player's game. The clothing and barber picture list. A normal script does not need it. Read worn clothes with [fw_clothing](fw-clothing.md).

```lua
--- The clothing and barber picture list.
---@return table assets
exports.fw_creator:GetMurphyAssets()
```

```lua
local assets = exports.fw_creator:GetMurphyAssets()
```

## State bags

| Name | Who can read it | What it is |
| --- | --- | --- |
| `murphy_appearance_paused` | This player's game only | Yes while `SetAppearancePaused(true)` is on. |
| `PlayerIsInCharacterShops` | The server and every player's game | Yes while this player is in the character menu. No after they pick a character. |
| `murphy_expressions` | The server and every player's game | The face sliders. Written by `PublishLocalExpressions`. |
| `murphy_face_comps` | The server and every player's game | Eyes and teeth. Written by `PublishLocalFaceComps`. |
| `murphyLocale` | Everyone, on the global state | A language name, if another resource sets `GlobalState.murphyLocale`. This menu reads it. It does not set it. |

A script that should not teleport someone who is still picking a character:

```lua
if Player(source).state.PlayerIsInCharacterShops then
    return
end
```

On the player's own game:

```lua
if LocalPlayer.state.murphy_appearance_paused then
    return
end
```

## Commands

| Command | Who | What it does |
| --- | --- | --- |
| `/mc` | The player | Reloads their body on the client. |
| `/deletechar [characterId]` | An admin, or a role or identifier listed in config | Deletes the caller's character. The server console cannot delete, because delete needs the owning player. Someone else's id returns `character not found`. |
| `/secondchance [serverId]` | The same people. The console may run it. | Opens the body menu again for that player. |

Who may delete is `Config.CharacterDeletion` in `shared/config.lua`. When `restrictToRoles` is false, every player may delete.
