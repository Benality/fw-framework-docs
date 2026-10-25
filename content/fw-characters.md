# fw_characters

Server-only. Version `1.0.0`. Registers with `fw_core` API `1`.

Source: `resources/[framework]/fw_characters`.

This resource is the API for one character row. It does not open a menu, spawn a ped, or move the player. `fw_creator` does that and calls these exports. Any later resource that needs the active character should call these exports too.

`IsReady` is false until config loads, `fw_core` is ready, `sql/characters.sql` has been applied, and `RegisterResource(1)` succeeds. The wait for core is 120 seconds. If that wait expires, the log is `fw_core is not ready` and the character table is not created.

On success:

```
[fw_characters][info] ready version=1.0.0
```

Debug lines need:

```
set fw_characters_debug true
```

Creation, deletion, startup, and errors print without the convar.

## Character record

`GetCharacter` and `GetActiveCharacter` return a copy of this record. `GetCharacters` returns the same fields except `appearance`, `clothing`, and `barber`. The list still includes `hasAppearance`, `hasClothing`, and `hasBarber`.

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | integer | `fw_characters.id`. The id other resources should store. |
| `source` | integer | Server id the row was loaded for. |
| `playerId` | integer | `fw_players.id` of the owner. |
| `slot` | integer | Slot on that account, from 1 through `character_slots`. |
| `firstName` | string | 1–50 characters. |
| `lastName` | string | 1–50 characters. |
| `nickname` | string or nil | Optional, 1–50 characters. Nil when the column is NULL. |
| `gender` | string | `male` or `female`. |
| `birthDate` | string or nil | `YYYY-MM-DD`, or nil. |
| `description` | string | Lore. Empty string when the player left it blank. Max 2000 characters. |
| `x`, `y`, `z` | number | Last saved position. |
| `heading` | number | Last saved facing in degrees. |
| `health` | integer | Entity health, 0–65535. New characters use `defaultHealth` (600). |
| `healthCore` | integer | Health core, 0–100. Default 100. |
| `staminaCore` | integer | Stamina core, 0–100. Default 100. |
| `isDead` | boolean | |
| `createdAt` | string | Database timestamp. |
| `lastPlayed` | string or nil | Set when the character is selected, cleared, or the player drops. Nil until the first selection. |
| `hasAppearance` | boolean | The appearance column is non-NULL. |
| `hasClothing` | boolean | The clothing column is non-NULL. |
| `hasBarber` | boolean | The barber column is non-NULL. |
| `appearance` | table or nil | Present on the full record only. |
| `clothing` | table or nil | Present on the full record only. |
| `barber` | table or nil | Present on the full record only. |

A document that is stored but cannot be decoded becomes nil, and an error is logged. The row is still returned.

Changing the returned table does not change the database. Call `UpdateCharacter`, `SetAppearance`, `SetClothing`, `SetBarber`, or `SaveState`.

Every call that takes a character id checks that the server id is an attached `fw_core` player and that the row's `player_id` is that account. A missing row and someone else's row both return `character not found`.

When the resource is not ready, exports return `nil, 'fw_characters is not ready'`.

## Config

`server/config.lua`, table `FwCharactersConfig`. Read once at startup. A bad value aborts startup and the resource stays not ready.

| Field | Current value | Rule |
| --- | --- | --- |
| `defaultSpawn.x` | `-174.39` | Finite number. Used only when `CreateCharacter` inserts a row. |
| `defaultSpawn.y` | `633.78` | Finite number. |
| `defaultSpawn.z` | `114.09` | Finite number. |
| `defaultSpawn.heading` | `0.0` | Finite number. |
| `defaultHealth` | `600` | Integer 0–65535. |
| `defaultHealthCore` | `100` | Integer 0–100. |
| `defaultStaminaCore` | `100` | Integer 0–100. |
| `minBirthYear` | `1800` | Integer 1–9999, and not above `maxBirthYear`. |
| `maxBirthYear` | `1899` | Integer 1–9999. |

Confirm the spawn coordinates in game. `fw_creator` overwrites them with the spawn the player picked, after it selects the new character.

## Exports

### IsReady

```lua
---@return boolean
exports.fw_characters:IsReady()
```

### GetCharacters

```lua
---@param source integer
---@return table[]|nil characters
---@return string|nil err
exports.fw_characters:GetCharacters(source)
```

Every character on that account, ordered by `slot`. Documents are omitted. Flags say whether each document exists.

`nil, 'player is not loaded'` when `source` is not an attached player.

```lua
local list, err = exports.fw_characters:GetCharacters(source)
if not list then
    print(err)
    return
end

for index = 1, #list do
    local character = list[index]
    print(character.id, character.slot, character.firstName, character.hasAppearance)
end
```

### GetCharacter

```lua
---@param source integer
---@param characterId integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetCharacter(source, characterId)
```

Full copy, including the three documents, when that account owns `characterId`.

| Second return | When |
| --- | --- |
| `'player is not loaded'` | Source is not an attached player. |
| `'character id must be a positive integer'` | The id is missing, fractional, or below 1. |
| `'character not found'` | No row for that id on that account. |

### GetActiveCharacter

```lua
---@param source integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetActiveCharacter(source)
```

Full copy of the character selected for that source. `nil` when nobody is selected. The error string is set when the resource is not ready. A ready resource with no selection returns `nil` and no error.

The active record is memory. `SaveState` and document saves update it. Identity updates update it when the active id is the one being edited.

### CreateCharacter

```lua
---@param source integer
---@param data table
---@return integer|nil characterId
---@return string|nil err
exports.fw_characters:CreateCharacter(source, data)
```

Inserts a row and returns the new id. It does not select the character. Call `SelectCharacter` when that character should be the active one.

`data` fields:

| Field | Required | Rule |
| --- | --- | --- |
| `firstName` | yes | String, 1–50 UTF-8 characters. Trimmed. No NUL and no control characters. |
| `lastName` | yes | Same as `firstName`. |
| `gender` | yes | String. Trimmed and lowercased. `male` or `female`. `Male` is accepted. |
| `birthDate` | no | `YYYY-MM-DD` inside the config year range, or omit it. A real calendar date, including leap days. Empty string is stored as NULL. |
| `nickname` | no | 1–50 characters, or `''` to store NULL. |
| `description` | no | Up to 2000 characters. Newlines are allowed. Omitted becomes `''`. |
| `slot` | no | Integer from 1 through that account's `character_slots`. Omitted uses the lowest free slot. |
| `appearance` | no | Table, encoded JSON at most 262144 bytes. |
| `clothing` | no | Same limit. |
| `barber` | no | Same limit. |

Position and health on a new row come from config, not from `data`. There is no `x` field on create. After `SelectCharacter`, `SaveState` can move the character.

The slot limit is `SELECT character_slots FROM fw_players` at create time. If that read fails, the connected player's `characterSlots` is used.

| Error | When |
| --- | --- |
| `'player is not loaded'` | Source is not attached. |
| `'character data must be a table'` | `data` is not a table. |
| `'first name must be text'` | `firstName` is not a string. The same shape is used for last name, nickname, and description. |
| `'first name contains unsupported characters'` | NUL, a broken UTF-8 sequence, or a control character. |
| `'first name must be 1 to 50 characters'` | Empty or too long. Description uses 2000. |
| `'gender must be male or female'` | Missing or any other value. |
| `'birth date must be YYYY-MM-DD'` | Present and not that shape. |
| `'birth date must be a real date from 1800 to 1899'` | Out of range, or not a real day. The years are the config values. |
| `'appearance must be a table'` | The field is present and is not a table. Same for `clothing` and `barber`. `nil` means "do not set". |
| `'could not encode appearance'` | `json.encode` failed. |
| `'appearance is too large'` | Encoded size is over 262144 bytes. |
| `'character slot limit reached'` | The account already has `character_slots` rows, or the limit is below 1. |
| `'character slot is not available'` | The requested slot is out of range, taken, or the insert hit the unique key. |
| `'could not create the character'` | The insert failed for another reason, or the new id could not be read. |

```lua
local characterId, err = exports.fw_characters:CreateCharacter(source, {
    firstName = 'Jane',
    lastName = 'Doe',
    gender = 'female',
    birthDate = '1870-04-12',
    nickname = 'JD',
    description = 'A line of lore.',
})
if not characterId then
    print(err)
    return
end

local selected, selectErr = exports.fw_characters:SelectCharacter(source, characterId)
if not selected then
    print(selectErr)
end
```

`fw_characters:created` fires with `(source, character)` after the insert. The character argument is a full copy. The new character is not active yet.

### UpdateCharacter

```lua
---@param source integer
---@param characterId integer
---@param data table
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:UpdateCharacter(source, characterId, data)
```

Updates only the identity fields that are present. Omitted fields stay as they are. This does not change documents, position, or health.

| Field | Effect when present |
| --- | --- |
| `firstName`, `lastName`, `gender` | Replaced. Same validation as create. |
| `nickname` | Replaced. `''` sets NULL. |
| `birthDate` | Replaced. `''` sets NULL. |
| `description` | Replaced. `''` stores an empty string. |

`'nothing to update'` when `data` contains none of those fields.

`'character not found'` when the id is not owned by that account. An update that sets a column to the value it already has still counts as success when the row exists. `changedRows == 0` is not treated as a missing row.

`fw_characters:updated` fires with `(source, characterId)`. The event does not include the new values. Call `GetCharacter` if you need them. If this id is the active character, the memory copy is updated too.

### SelectCharacter

```lua
---@param source integer
---@param characterId integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SelectCharacter(source, characterId)
```

Loads that owned character into the active slot, sets state bag `fwCharacterId`, and touches `last_played`.

Selecting the id that is already active returns `true` and does not fire again.

Selecting a different id clears the previous one first (`fw_characters:cleared` with the old id), then fires `fw_characters:selected` with `(source, character)`. The character argument is a full copy.

This does not move the ped and does not change routing buckets. The caller teleports.

### ClearCharacter

```lua
---@param source integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:ClearCharacter(source)
```

Clears the active character for that source. Returns `true` when there was nothing to clear. Fires `fw_characters:cleared` with `(source, characterId)` only when a character was active. Sets `fwCharacterId` to nil and touches `last_played`.

`'player is not loaded'` when `source` is not a positive integer. Clear does not require the account to still be in the `fw_core` cache, because it is also the path used while a player is going away.

### SetAppearance, SetClothing, SetBarber

```lua
---@param source integer
---@param characterId integer
---@param value table|nil
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SetAppearance(source, characterId, value)
```

`SetClothing` and `SetBarber` take the same arguments. The third argument is the document for that column.

A table is encoded and stored. `nil` sets the column to NULL. The encoded JSON must be at most 262144 bytes. The resource does not require any keys inside the table. `fw_creator` defines the keys it writes; see [fw_creator](fw-creator.md).

Errors match create: not a table, encode failure, too large, character not found, player not loaded, bad id.

`fw_characters:updated` fires with `(source, characterId)`. `SaveState` is the call that does not fire `updated`. If this character is active, the memory copy of that document is replaced.

```lua
local ok, err = exports.fw_characters:SetAppearance(source, characterId, {
    gender = 'Female',
    skintone = 1,
})
if not ok then
    print(err)
end

-- Clear the clothing document.
exports.fw_characters:SetClothing(source, characterId, nil)
```

### SaveState

```lua
---@param state table
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SaveState(source, state)
```

Writes position and health for the active character only. It does not read or write the JSON columns. It does not fire `fw_characters:updated`.

`state` fields, all optional, but position is all or nothing:

| Field | Rule |
| --- | --- |
| `x`, `y`, `z`, `heading` | If any one is present, all four are required and must be finite numbers. |
| `health` | Integer 0–65535. |
| `healthCore` | Integer 0–100. |
| `staminaCore` | Integer 0–100. |
| `isDead` | Boolean. |

| Error | When |
| --- | --- |
| `'state must be a table'` | |
| `'player is not loaded'` | |
| `'no active character'` | `SelectCharacter` has not been called for this source. |
| `'position requires x, y, z, and heading'` | A partial position. |
| `'health must be an integer from 0 to 65535'` | |
| `'healthCore must be an integer from 0 to 100'` | |
| `'staminaCore must be an integer from 0 to 100'` | |
| `'isDead must be true or false'` | |
| `'nothing to save'` | The table had none of these fields. |
| `'character not found'` | The active row disappeared. |

```lua
local ok, err = exports.fw_characters:SaveState(source, {
    x = -174.39,
    y = 633.78,
    z = 114.09,
    heading = 90.0,
    health = 540,
    healthCore = 80,
    staminaCore = 100,
    isDead = false,
})
```

There is no stamina-outer column. Only `health`, `healthCore`, and `staminaCore` are stored.

### DeleteCharacter

```lua
---@param source integer
---@param characterId integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:DeleteCharacter(source, characterId)
```

Deletes that owned row. Appearance, clothing, and barber go with the row. If it was active, it is cleared first and `fw_characters:cleared` fires, then `fw_characters:deleted` fires with `(source, characterId)`.

The freed slot can be used by the next `CreateCharacter`.

## Local events

Server events. Listeners run before the export returns.

```lua
AddEventHandler('fw_characters:ready', function(version)
    -- version: manifest string, such as '1.0.0'
end)

AddEventHandler('fw_characters:created', function(source, character)
    -- character: full copy, not selected yet
end)

AddEventHandler('fw_characters:selected', function(source, character)
    -- character: full copy of the newly active character
end)

AddEventHandler('fw_characters:cleared', function(source, characterId)
end)

AddEventHandler('fw_characters:updated', function(source, characterId)
    -- identity or document save. Not SaveState. The JSON is not in the event.
end)

AddEventHandler('fw_characters:deleted', function(source, characterId)
end)
```

A resource started after `ready` has already fired should call `IsReady`.

On `playerDropped`, an active character is cleared, `last_played` is updated, and `fw_characters:cleared` fires. `fw_core:playerDropped` also fires. Order between the two handlers follows resource start order.

On resource stop, active rows get `last_played` updated. `cleared` is not fired for that stop path.

## State bag

`Player(source).state.fwCharacterId` is the selected character id, or nil when none is selected. It replicates. The account id remains `fwPlayerId`.

```lua
AddStateBagChangeHandler('fwCharacterId', nil, function(bagName, _, value)
    local player = GetPlayerFromStateBagName(bagName)
    if player == 0 then
        return
    end
    -- value is the character id, or nil
end)
```

After a resource restart, a still-connected player is restored from this state bag. A restore that fails clears the bag.

## Console commands

Restricted to the server console. A player gets no output.

```
fw_characters
fw_char_list <serverId>
fw_char_create <serverId> <firstName> <lastName> <male|female> [YYYY-MM-DD]
fw_char_select <serverId> <characterId>
fw_char_delete <serverId> <characterId>
```

`fw_characters` prints `ready` and how many sources have an active character.

`fw_char_list` prints `id`, `slot`, name, and gender for that server id.

`fw_char_create` takes single-word names because the console splits on spaces. The export accepts spaces inside `firstName`. The command does not select the new character.

`serverId` is the live server id from `fw_health`, after `playerJoining`. A character can be created only for a connected player.

## Files

| File | Purpose |
| --- | --- |
| `fxmanifest.lua` | Version `1.0.0`. Dependencies `oxmysql` and `fw_core`. |
| `server/config.lua` | Spawn, health defaults, birth years. |
| `sql/characters.sql` | Table definition. |
| `server/main.lua` | Schema, exports, events, console commands. |
