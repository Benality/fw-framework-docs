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

`fw_characters:selected` is the event to use when your resource needs to know a character was chosen.

## Commands

| Command | Who | What it does |
| --- | --- | --- |
| `/mc` | The player | Reloads their body on the client. |
| `/deletechar [characterId]` | An admin, or a role or identifier listed in config | Deletes the caller's character. The server console cannot delete, because delete needs the owning player. Someone else's id returns `character not found`. |
| `/secondchance [serverId]` | The same people. The console may run it. | Opens the body menu again for that player. |

Who may delete is `Config.CharacterDeletion` in `shared/config.lua`. When `restrictToRoles` is false, every player may delete.
