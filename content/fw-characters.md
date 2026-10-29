# fw_characters

Runs on the server. This is how you create a character, read one, and save where they are standing or a saved document on them. Do not write the `fw_characters` table yourself.

This does not open the character menu, and it does not move the player. The character menu does those things, then uses the calls on this page. Your script should use these calls too.

Ask `IsReady` first. If it says no, every other call on this page comes back empty, and the reason is `fw_characters is not ready`. `nil` on this page means nothing came back.

A call that takes a character id checks that this player owns that character. A missing character and someone else's character both come back as `character not found`.

What you get back is a snapshot. Changing it does not save. Use `UpdateCharacter`, `SetAppearance`, `SetClothing`, `SetBarber`, or `SaveState`.

## Calls

### IsReady

Runs on the server. Yes after startup has finished.

```lua
--- True when character calls can be used.
---@return boolean ready
exports.fw_characters:IsReady()
```

### GetCharacters

Runs on the server. Every character on this player's account, lowest slot first. The body, clothes, and hair are left out. `hasAppearance`, `hasClothing`, and `hasBarber` say whether each one was saved.

```lua
--- The character list for this player, or nil when the player is not in the server.
---@param source integer
---@return table[]|nil characters
---@return string|nil err
exports.fw_characters:GetCharacters(source)
```

`source` is the player's number for this visit. `player is not loaded` means that player has not finished joining.

```lua
local list, err = exports.fw_characters:GetCharacters(source)
if list == nil then
    print(err)
    return
end

for index = 1, #list do
    local character = list[index]
    print(character.id, character.firstName, character.hasAppearance)
end
```

### GetCharacter

Runs on the server. One full character, including body, clothes, and hair, when this player owns it.

```lua
--- One character owned by this player, or nil and a reason.
---@param source integer
---@param characterId integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetCharacter(source, characterId)
```

| `err` | When |
| --- | --- |
| `player is not loaded` | That player has not finished joining. |
| `character id must be a positive integer` | The id is missing or below 1. |
| `character not found` | No character with that id on this account. |

```lua
local character, err = exports.fw_characters:GetCharacter(source, characterId)
if character == nil then
    print(err)
    return
end
```

### GetActiveCharacter

Runs on the server. The character this player is playing right now. You get nothing, and no reason, when nobody is selected. You get nothing and a reason when this resource is not ready yet.

```lua
--- The character selected for this player.
--- Nil when no character is selected.
---@param source integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetActiveCharacter(source)
```

```lua
local character, err = exports.fw_characters:GetActiveCharacter(source)
if character == nil then
    print(err)
    return
end
```

Ask this again when you need a fresh copy. Do not keep the old one and treat it as live.

### CreateCharacter

Runs on the server. Creates a character and gives you the new id. It does not make that character the one they are playing. Use `SelectCharacter` for that.

Position and health on the new row come from the character config, not from `data`. The menu saves the spawn the player picked with `SaveState` right after.

```lua
--- Creates a character and returns the new id.
--- Nil and a reason when the name, gender, slot, or document is rejected.
---@param source integer
---@param data table
---@return integer|nil characterId
---@return string|nil err
exports.fw_characters:CreateCharacter(source, data)
```

| Field | Required | Rule |
| --- | --- | --- |
| `firstName` | yes | 1–50 characters. |
| `lastName` | yes | Same as the first name. |
| `gender` | yes | `male` or `female`. `Male` is accepted and stored in lowercase. |
| `birthDate` | no | `YYYY-MM-DD` between the configured years, or leave it out. `''` stores nothing. |
| `nickname` | no | 1–50 characters, or `''` to store nothing. |
| `description` | no | Up to 2000 characters. Leave it out and it is stored as empty. |
| `slot` | no | A free slot from 1 through this account's character limit. Leave it out and the lowest free slot is used. |
| `appearance`, `clothing`, `barber` | no | A table, or leave it out. The saved text cannot be larger than 262144 bytes. |

| `err` | When |
| --- | --- |
| `player is not loaded` | That player has not finished joining. |
| `character data must be a table` | `data` is not a table. |
| `first name must be text` | The name is not text. The same shape is used for the last name, nickname, and description. |
| `first name contains unsupported characters` | The name has a control character. |
| `first name must be 1 to 50 characters` | Empty or too long. Description uses 2000. |
| `gender must be male or female` | Missing or any other value. |
| `birth date must be YYYY-MM-DD` | Present and not that shape. |
| `birth date must be a real date from 1800 to 1899` | Outside the configured years, or not a real day. |
| `appearance must be a table` | Present and not a table. The same words are used for clothing and barber. |
| `could not encode appearance` | The table could not be saved as text. |
| `appearance is too large` | Over 262144 bytes. |
| `character slot limit reached` | This account already has as many characters as it is allowed. |
| `character slot is not available` | That slot is taken or out of range. |
| `could not create the character` | The insert failed. |

```lua
local characterId, err = exports.fw_characters:CreateCharacter(source, {
    firstName = 'Jane',
    lastName = 'Doe',
    gender = 'female',
    birthDate = '1870-04-12',
})
if characterId == nil then
    print(err)
    return
end

local selected, selectErr = exports.fw_characters:SelectCharacter(source, characterId)
if not selected then
    print(selectErr)
end
```

After a successful insert, the local event `fw_characters:created` fires with the new character. It is not selected yet.

### UpdateCharacter

Runs on the server. Changes name, gender, nickname, birth date, or description. Fields you leave out stay as they are. This does not change body, clothes, hair, position, or health.

```lua
--- Updates identity fields that are present in data.
--- Nil and a reason when there is nothing to update or the character is not owned.
---@param source integer
---@param characterId integer
---@param data table
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:UpdateCharacter(source, characterId, data)
```

`''` for nickname or birth date clears that field. `''` for description stores an empty description.

`nothing to update` means `data` had none of those fields. `character not found` means this player does not own that id.

The local event `fw_characters:updated` fires with the source and the character id. It does not include the new values. Call `GetCharacter`.

### SelectCharacter

Runs on the server. Makes this character the one the player is playing. It does not move them. Your script does that, using `x`, `y`, `z`, and `heading` from the character you get back.

```lua
--- Selects an owned character for this player.
---@param source integer
---@param characterId integer
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:SelectCharacter(source, characterId)
```

`character not found` means this player does not own that id. Selecting the character they are already playing returns true and fires nothing. Selecting a different one fires `fw_characters:cleared` for the old id, then `fw_characters:selected` with a copy of the new character.

### ClearCharacter

Runs on the server. The player is no longer playing a character. True when it cleared one, and true when none was selected. `player is not loaded` when `source` is not a player id.

```lua
--- Clears the character selected for this player.
---@param source integer
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:ClearCharacter(source)
```

`fw_characters:cleared` fires only when a character was actually selected. The player's game also receives `fw_characters:released` for that same character.

### SetAppearance, SetClothing, SetBarber

Runs on the server. Replaces one saved block on a character this player owns. Pass nothing (`nil`) to clear it. The same rules apply to all three calls.

```lua
--- Replaces the body document. Nil clears it.
---@param source integer
---@param characterId integer
---@param value table|nil
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:SetAppearance(source, characterId, value)
```

```lua
--- Replaces the clothing document. Nil clears it.
---@param source integer
---@param characterId integer
---@param value table|nil
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:SetClothing(source, characterId, value)
```

```lua
--- Replaces the hair and makeup document. Nil clears it.
---@param source integer
---@param characterId integer
---@param value table|nil
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:SetBarber(source, characterId, value)
```

The table is saved as text and cannot be larger than 262144 bytes. `appearance must be a table`, `could not encode appearance`, and `appearance is too large` use the document name. `character not found` means this player does not own that id.

`fw_characters:updated` fires. It does not include the document. Call `GetCharacter`.

The character menu decides the keys inside `appearance` and `barber`. See [fw_creator](fw-creator.md). Worn clothes should go through [fw_clothing](fw-clothing.md), which calls `SetClothing` for you.

```lua
local ok, err = exports.fw_characters:SetAppearance(source, characterId, {
    gender = 'Female',
    skintone = 1,
})
if not ok then
    print(err)
    return
end
```

### SaveState

Runs on the server. Saves where they are standing, and their health, for the character this player is playing. It does not change the name or the saved body, clothes, or hair. It does not send the `fw_characters:updated` message.

```lua
--- Saves position and health for the active character.
---@param source integer
---@param state table
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:SaveState(source, state)
```

Every field is optional, but position is all or nothing. If you send any of `x`, `y`, `z`, or `heading`, send all four.

| Field | Rule |
| --- | --- |
| `x`, `y`, `z`, `heading` | Numbers. Heading is degrees. |
| `health` | Whole number from 0 to 65535. |
| `healthCore` | Whole number from 0 to 100. |
| `staminaCore` | Whole number from 0 to 100. |
| `isDead` | True or false. |

| `err` | When |
| --- | --- |
| `state must be a table` | `state` is not a table. |
| `player is not loaded` | That number is not a player in the game. |
| `no active character` | No character is selected. |
| `position requires x, y, z, and heading` | Only part of the position was sent. |
| `health must be an integer from 0 to 65535` | Health is out of range. |
| `healthCore must be an integer from 0 to 100` | Health core is out of range. |
| `staminaCore must be an integer from 0 to 100` | Stamina core is out of range. |
| `isDead must be true or false` | `isDead` is not a boolean. |
| `nothing to save` | None of these fields were sent. |
| `character not found` | The selected character no longer exists. |

```lua
local ok, err = exports.fw_characters:SaveState(source, {
    x = -174.39,
    y = 633.78,
    z = 114.09,
    heading = 90.0,
    health = 540,
    isDead = false,
})
if not ok then
    print(err)
end
```

The game also reports where the ped is standing. Those reports update the coordinates `GetCharacter` returns, and they are written to the database on a timer, on disconnect, and when the character is cleared. `SaveState` writes immediately.

### DeleteCharacter

Runs on the server. Deletes a character this player owns. Body, clothes, and hair go with it. If it was selected, it is cleared first.

```lua
--- Deletes an owned character.
---@param source integer
---@param characterId integer
---@return boolean|nil ok
---@return string|nil err
exports.fw_characters:DeleteCharacter(source, characterId)
```

`character not found` means this player does not own that id. `fw_characters:cleared` fires when it was selected. `fw_characters:deleted` fires after the row is gone.

## Character record

| Field | Meaning |
| --- | --- |
| `id` | Character id. Store this when a row belongs to one character. |
| `source` | Server id the row was loaded for. |
| `playerId` | Account id of the owner. |
| `slot` | Slot on that account, starting at 1. |
| `firstName`, `lastName` | 1–50 characters. |
| `nickname` | Optional. Nil when empty. |
| `gender` | `male` or `female`. |
| `birthDate` | `YYYY-MM-DD`, or nil. |
| `description` | Lore. Empty string when blank. |
| `x`, `y`, `z`, `heading` | Last known position. A recent sample can be newer than the last database write. |
| `health` | 0–65535. A new character starts at 600. |
| `healthCore`, `staminaCore` | 0–100. A new character starts at 100. |
| `isDead` | True or false. |
| `createdAt` | When the row was created. |
| `lastPlayed` | When they last played it. Nil until the first selection. |
| `hasAppearance`, `hasClothing`, `hasBarber` | True when that document exists. |
| `appearance`, `clothing`, `barber` | The documents. Only on `GetCharacter` and `GetActiveCharacter`. |

`GetCharacters` leaves the three documents out.

A new character is placed at the default spawn in `server/config.lua` until something calls `SaveState`. Birth years must fall between `minBirthYear` and `maxBirthYear` in that file. Today that is 1800 through 1899.

## Messages

These stay on the server. They are not sent to the player's game. Your script hears one only if it is already running. Put the listener at the top of your script. If your script starts after `fw_characters:ready` was sent, you missed it. Ask `IsReady` instead.

The listeners for created, selected, cleared, updated, and deleted run before the call that caused them finishes. The snapshot in the message can be old a moment later. Ask `GetCharacter` when you need a fresh one.

### fw_characters:ready

Fires once, when the resource is ready. `version` is a string such as `1.0.0`.

```lua
---@param version string
AddEventHandler('fw_characters:ready', function(version)
end)
```

### fw_characters:created

The new character is a copy and is not selected yet.

```lua
---@param source integer
---@param character table
AddEventHandler('fw_characters:created', function(source, character)
end)
```

### fw_characters:selected

A snapshot of the character that just became the one they are playing. Ask `GetCharacter` when you need a fresh one.

```lua
---@param source integer
---@param character table
AddEventHandler('fw_characters:selected', function(source, character)
end)
```

### fw_characters:cleared

The player is no longer on that character. It is also sent when they leave, if one was selected. It is not sent when this resource stops.

```lua
---@param source integer
---@param characterId integer
AddEventHandler('fw_characters:cleared', function(source, characterId)
end)
```

### fw_characters:released

Sent to that player's game when the character is cleared, including a switch and a delete. It is not sent when they drop, because their game is already gone. Listen on the client. `characterId` is the character that just ended. Clear anything you were keeping for that character.

```lua
--- The character on this player's game just ended.
---@param characterId integer
---@return nil
RegisterNetEvent('fw_characters:released', function(characterId)
end)
```

### fw_characters:relog

Networked. The player's game sends it. `/relog` does that. The command name is `relogCommand` in `server/config.lua`.

The server refuses, and writes a warning, when no character is selected, they are already on the character list, the last switch was too recent (`relogCooldownMs`), they are in a trade, their ped is dead, or they are hogtied or cuffed. A success saves place, health, and cores, clears the character, writes an info line, and tells that player's game to open the character list.

```lua
--- Asks to return to the character list.
---@return nil
TriggerServerEvent('fw_characters:relog')
```

### fw_characters:leave

Networked. The player's game sends it. `/leave` does that. The command name is `leaveCommand` in `server/config.lua`.

The same save runs when a character is selected. Then the server drops them with `leaveReason`, which starts as `Left the server.` There is no wait between switches for this one. Someone on the character list, with no character selected, is dropped without a save.

```lua
--- Asks to save and return to the RedM menu.
---@return nil
TriggerServerEvent('fw_characters:leave')
```

### fw_characters:updated

Name or a document was saved. Not fired by `SaveState`. The new values are not in the event. Call `GetCharacter`.

```lua
---@param source integer
---@param characterId integer
AddEventHandler('fw_characters:updated', function(source, characterId)
end)
```

### fw_characters:deleted

The character id that was deleted.

```lua
---@param source integer
---@param characterId integer
AddEventHandler('fw_characters:deleted', function(source, characterId)
end)
```

## A value the player's game can read

`Player(source).state.fwCharacterId` is the selected character id, or nothing when none is selected. The player's game can read it. The account id is still `fwPlayerId` from [fw_core](fw-core.md).

On the server:

```lua
local characterId = Player(source).state.fwCharacterId
if characterId == nil then
    return
end
```

In the player's game:

```lua
local characterId = LocalPlayer.state.fwCharacterId
```

`fwHogtied` and `fwCuffed` are true while that ped is hogtied or cuffed. The player's game publishes them about once a second. A switch reads them. They go back to false when the character ends.

## Where the character is standing

These are networked events: `RegisterNetEvent` on the server, sent from the player's game with `TriggerServerEvent`. `fw_characters`'s own client script sends them while the player plays. Another script should save a position with `SaveState` instead. Do not send these yourself, and do not listen for them. A bad position is ignored.

| Message | What it does |
| --- | --- |
| `fw_characters:reportPosition` | The player's game tells the server `x`, `y`, `z`, and `heading` on a timer. |
| `fw_characters:commitPosition` | The player's game tells the server to store that position now, including when they are leaving. |

A jail script can store where they were standing before it moves them:

```lua
local character, err = exports.fw_characters:GetActiveCharacter(source)
if character == nil then
    print(err)
    return
end

local ok, saveErr = exports.fw_characters:SaveState(source, {
    x = character.x,
    y = character.y,
    z = character.z,
    heading = character.heading,
})
if not ok then
    print(saveErr)
end
```

## Server window

These do nothing when a player types them in chat.

| Command | What it does |
| --- | --- |
| `fw_characters` | Prints whether the resource is ready, and how many players have a character selected. |
| `fw_char_list <serverId>` | Lists that player's characters. |
| `fw_char_create <serverId> <firstName> <lastName> <male\|female> [YYYY-MM-DD]` | Creates one. Names cannot contain a space here. It does not select the new character. |
| `fw_char_select <serverId> <characterId>` | Selects that character. |
| `fw_char_delete <serverId> <characterId>` | Deletes that character. |

`serverId` is the player number printed by `fw_health`.
