# fw_characters

Server-only API for one character row. Version `1.0.0`. Registers with `fw_core` API `1`.

Source: `resources/[framework]/fw_characters`.

This resource owns `fw_characters`. It does not open a menu, spawn a ped, or move the player — `fw_creator` does that and calls these exports. Any later resource that needs the active character should call these exports too, not read the table directly.

## Startup

`IsReady()` is `false` until config loads, `fw_core` is ready, `sql/characters.sql` has been applied, and `RegisterResource(1)` succeeds. The wait for core is 120 seconds; if it expires, the log is `fw_core is not ready` and the character table is not created.

On success:

```
[fw_characters][info] ready version=1.0.0
```

Debug lines need:

```
set fw_characters_debug true
```

Creation, deletion, startup, and errors print regardless of the convar.

## Exports

| Export | Summary |
| --- | --- |
| [`IsReady()`](#isready) | Whether the resource is ready to serve calls. |
| [`GetCharacters(source)`](#getcharacterssource) | The lightweight list of an account's characters. |
| [`GetCharacter(source, characterId)`](#getcharactersource-characterid) | A full character record, by id. |
| [`GetActiveCharacter(source)`](#getactivecharactersource) | The full record of the currently selected character. |
| [`CreateCharacter(source, data)`](#createcharactersource-data) | Inserts a new character row. |
| [`UpdateCharacter(source, characterId, data)`](#updatecharactersource-characterid-data) | Changes identity fields on an existing row. |
| [`SelectCharacter(source, characterId)`](#selectcharactersource-characterid) | Makes a character the active one for a source. |
| [`ClearCharacter(source)`](#clearcharactersource) | Deselects the active character for a source. |
| [`SetAppearance(source, characterId, value)`](#setappearance-setclothing-setbarber) | Replaces the appearance document. |
| [`SetClothing(source, characterId, value)`](#setappearance-setclothing-setbarber) | Replaces the clothing document. |
| [`SetBarber(source, characterId, value)`](#setappearance-setclothing-setbarber) | Replaces the barber document. |
| [`SaveState(source, state)`](#savestatesource-state) | Writes position and health for the active character. |
| [`DeleteCharacter(source, characterId)`](#deletecharactersource-characterid) | Deletes an owned row. |

Every export that takes a `characterId` checks that `source` is an attached `fw_core` player and that the row's `player_id` is that account. A missing row and someone else's row both return `'character not found'`. When the resource is not ready, every export returns `nil, 'fw_characters is not ready'`.

---

### `IsReady()`

**Signature**

```lua
---@return boolean
exports.fw_characters:IsReady()
```

**Parameters** — none.

**Returns**

| Type | Description |
| --- | --- |
| `boolean` | `true` once config, `fw_core`, the schema, and registration all succeeded. `false` otherwise, including during the startup wait. |

---

### `GetCharacters(source)`

**Signature**

```lua
---@param source integer
---@return table[]|nil characters
---@return string|nil err
exports.fw_characters:GetCharacters(source)
```

Every character on that account, ordered by `slot`. Documents (`appearance`, `clothing`, `barber`) are omitted — see [Character record](#character-record) for the `has*` flags that say whether each exists.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Server id of the connected player. |

**Returns**

| Type | Description |
| --- | --- |
| `table[]` | List of character records (no documents), ordered by slot. |
| `nil, 'player is not loaded'` | `source` is not an attached player. |

**Example**

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

---

### `GetCharacter(source, characterId)`

**Signature**

```lua
---@param source integer
---@param characterId integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetCharacter(source, characterId)
```

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Server id of the connected player. |
| `characterId` | `integer` | yes | `fw_characters.id` to read. Must be owned by `source`'s account. |

**Returns**

| Type | Description |
| --- | --- |
| `table` | Full copy of the record, including `appearance`, `clothing`, and `barber`. |
| `nil` | See Errors. |

**Errors**

| `err` value | Condition |
| --- | --- |
| `'player is not loaded'` | `source` is not an attached player. |
| `'character id must be a positive integer'` | `characterId` is missing, fractional, or below 1. |
| `'character not found'` | No row for that id on that account. |

---

### `GetActiveCharacter(source)`

**Signature**

```lua
---@param source integer
---@return table|nil character
---@return string|nil err
exports.fw_characters:GetActiveCharacter(source)
```

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Server id of the connected player. |

**Returns**

| Type | Description |
| --- | --- |
| `table` | Full copy of the character currently selected for `source`. |
| `nil, nil` | The resource is ready but nobody is selected. |
| `nil, err` | The resource is not ready. |

The active record lives in memory. `SaveState` and the document setters update it in place; identity updates update it only when the active id is the one being edited.

---

### `CreateCharacter(source, data)`

**Signature**

```lua
---@param source integer
---@param data table
---@return integer|nil characterId
---@return string|nil err
exports.fw_characters:CreateCharacter(source, data)
```

Inserts a row and returns the new id. It does **not** select the character — call `SelectCharacter` afterward if it should become active. Position and health on the new row come from config (`FwCharactersConfig.defaultSpawn`), not from `data`.

**Parameters**

`source: integer` — server id of the connected player.

`data: table` —

| Field | Required | Rule |
| --- | --- | --- |
| `firstName` | yes | String, 1–50 UTF-8 characters. Trimmed. No NUL or control characters. |
| `lastName` | yes | Same as `firstName`. |
| `gender` | yes | String, trimmed and lowercased. `male` or `female`; `Male` is accepted. |
| `birthDate` | no | `YYYY-MM-DD` inside the config year range, or omit. Must be a real calendar date, leap days included. Empty string stores NULL. |
| `nickname` | no | 1–50 characters, or `''` to store NULL. |
| `description` | no | Up to 2000 characters. Newlines allowed. Omitted becomes `''`. |
| `slot` | no | Integer from 1 through that account's `character_slots`. Omitted uses the lowest free slot. |
| `appearance` | no | Table, encoded JSON ≤ 262144 bytes. |
| `clothing` | no | Same limit. |
| `barber` | no | Same limit. |

The slot limit is read from `fw_players.character_slots` at create time (falling back to the connected player's cached `characterSlots` if that read fails).

**Returns**

| Type | Description |
| --- | --- |
| `integer` | The new `fw_characters.id`. |
| `nil, err` | See Errors. |

**Errors**

| `err` value | Condition |
| --- | --- |
| `'player is not loaded'` | `source` is not attached. |
| `'character data must be a table'` | `data` is not a table. |
| `'first name must be text'` | `firstName` is not a string. Same shape for last name, nickname, description. |
| `'first name contains unsupported characters'` | NUL, broken UTF-8, or a control character. |
| `'first name must be 1 to 50 characters'` | Empty or too long (description uses 2000). |
| `'gender must be male or female'` | Missing or any other value. |
| `'birth date must be YYYY-MM-DD'` | Present and not that shape. |
| `'birth date must be a real date from 1800 to 1899'` | Out of the config's year range, or not a real day. |
| `'appearance must be a table'` | The field is present and not a table (same for `clothing`, `barber`). `nil` means "do not set." |
| `'could not encode appearance'` | `json.encode` failed. |
| `'appearance is too large'` | Encoded size over 262144 bytes. |
| `'character slot limit reached'` | The account already has `character_slots` rows, or the limit is below 1. |
| `'character slot is not available'` | The requested slot is out of range, taken, or hit the unique key. |
| `'could not create the character'` | The insert failed for another reason, or the new id could not be read. |

**Example**

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

Fires `fw_characters:created` with `(source, character)` after the insert — the character is a full copy and is not yet active.

---

### `UpdateCharacter(source, characterId, data)`

**Signature**

```lua
---@param source integer
---@param characterId integer
---@param data table
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:UpdateCharacter(source, characterId, data)
```

Updates only the identity fields present in `data`; omitted fields are left unchanged. Never touches documents, position, or health.

**Parameters**

`source: integer`, `characterId: integer` — must be owned by `source`'s account.

`data: table` —

| Field | Effect when present |
| --- | --- |
| `firstName`, `lastName`, `gender` | Replaced. Same validation as `CreateCharacter`. |
| `nickname` | Replaced. `''` sets NULL. |
| `birthDate` | Replaced. `''` sets NULL. |
| `description` | Replaced. `''` stores an empty string. |

**Returns**

| Type | Description |
| --- | --- |
| `true` | Row updated (or already matched the given values — `changedRows == 0` still counts as success). |
| `nil, err` | See Errors. |

**Errors**

| `err` value | Condition |
| --- | --- |
| `'nothing to update'` | `data` contains none of the updatable fields. |
| `'character not found'` | The id is not owned by that account. |

Fires `fw_characters:updated` with `(source, characterId)` — the event does not carry the new values; call `GetCharacter` for those. If `characterId` is the active character, the in-memory copy is updated too.

---

### `SelectCharacter(source, characterId)`

**Signature**

```lua
---@param source integer
---@param characterId integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SelectCharacter(source, characterId)
```

Loads an owned character into the active slot, sets state bag `fwCharacterId`, and touches `last_played`. Does **not** move the ped or change routing buckets — the caller teleports.

**Parameters**

`source: integer`, `characterId: integer` — must be owned by `source`'s account.

**Returns**

`boolean|nil success`, `string|nil err` — `'character not found'` when not owned; otherwise the same errors as other id-taking exports.

Selecting the id that's already active returns `true` without firing anything. Selecting a different id first clears the previous one (`fw_characters:cleared` with the old id), then fires `fw_characters:selected` with `(source, character)` — a full copy.

---

### `ClearCharacter(source)`

**Signature**

```lua
---@param source integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:ClearCharacter(source)
```

Clears the active character for `source`.

**Parameters**

`source: integer`.

**Returns**

| Type | Description |
| --- | --- |
| `true` | Cleared, or there was nothing to clear. |
| `nil, 'player is not loaded'` | `source` is not a positive integer. |

Fires `fw_characters:cleared` with `(source, characterId)` only when a character was actually active. Sets `fwCharacterId` to `nil` and touches `last_played`. This does not require the account to still be in the `fw_core` cache — it's also the path used while a player is disconnecting.

---

### `SetAppearance`, `SetClothing`, `SetBarber`

**Signature**

```lua
---@param source integer
---@param characterId integer
---@param value table|nil
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SetAppearance(source, characterId, value)
```

`SetClothing` and `SetBarber` take identical arguments — the third argument is the document for that column.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `source` | `integer` | yes | Server id of the connected player. |
| `characterId` | `integer` | yes | Must be owned by `source`'s account. |
| `value` | `table \| nil` | yes | A table is encoded and stored (≤ 262144 bytes). `nil` sets the column to NULL. No keys are required inside the table — `fw_creator` defines the keys it writes; see [fw_creator](fw-creator.md). |

**Returns**

`boolean|nil success`, `string|nil err` — same error set as `CreateCharacter`'s document fields: not a table, encode failure, too large, character not found, player not loaded, bad id.

Fires `fw_characters:updated` with `(source, characterId)`. If this character is active, the in-memory copy of that document is replaced. (`SaveState` is the only write that does *not* fire `updated`.)

**Example**

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

---

### `SaveState(source, state)`

**Signature**

```lua
---@param source integer
---@param state table
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:SaveState(source, state)
```

Writes position and health for the **active character only**. Never touches the JSON columns, and never fires `fw_characters:updated`.

**Parameters**

`source: integer`.

`state: table` — all fields optional, but position is all-or-nothing:

| Field | Rule |
| --- | --- |
| `x`, `y`, `z`, `heading` | If any one is present, all four are required and must be finite numbers. |
| `health` | Integer 0–65535. |
| `healthCore` | Integer 0–100. |
| `staminaCore` | Integer 0–100. |
| `isDead` | Boolean. |

There is no stamina-outer column — only `health`, `healthCore`, and `staminaCore` are stored.

**Returns**

`boolean|nil success`, `string|nil err`.

**Errors**

| `err` value | Condition |
| --- | --- |
| `'state must be a table'` | |
| `'player is not loaded'` | |
| `'no active character'` | `SelectCharacter` has not been called for this source. |
| `'position requires x, y, z, and heading'` | A partial position. |
| `'health must be an integer from 0 to 65535'` | |
| `'healthCore must be an integer from 0 to 100'` | |
| `'staminaCore must be an integer from 0 to 100'` | |
| `'isDead must be true or false'` | |
| `'nothing to save'` | `state` had none of these fields. |
| `'character not found'` | The active row disappeared. |

**Example**

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

---

### `DeleteCharacter(source, characterId)`

**Signature**

```lua
---@param source integer
---@param characterId integer
---@return boolean|nil success
---@return string|nil err
exports.fw_characters:DeleteCharacter(source, characterId)
```

Deletes an owned row. `appearance`, `clothing`, and `barber` go with it. If the row was active, it's cleared first (`fw_characters:cleared` fires), then `fw_characters:deleted` fires with `(source, characterId)`. The freed slot can be used by the next `CreateCharacter`.

**Parameters**

`source: integer`, `characterId: integer` — must be owned by `source`'s account.

**Returns**

`boolean|nil success`, `string|nil err` — same ownership/validation errors as the other id-taking exports.

---

## Character record

`GetCharacter` and `GetActiveCharacter` return a copy of this record. `GetCharacters` returns the same fields minus `appearance`, `clothing`, and `barber` (replaced by the `has*` flags).

| Field | Type | Meaning |
| --- | --- | --- |
| `id` | `integer` | `fw_characters.id`. The id other resources should store. |
| `source` | `integer` | Server id the row was loaded for. |
| `playerId` | `integer` | `fw_players.id` of the owner. |
| `slot` | `integer` | Slot on that account, 1 through `character_slots`. |
| `firstName` | `string` | 1–50 characters. |
| `lastName` | `string` | 1–50 characters. |
| `nickname` | `string \| nil` | Optional, 1–50 characters. `nil` when the column is NULL. |
| `gender` | `string` | `male` or `female`. |
| `birthDate` | `string \| nil` | `YYYY-MM-DD`, or `nil`. |
| `description` | `string` | Lore. Empty string when left blank. Max 2000 characters. |
| `x`, `y`, `z` | `number` | Last saved position. |
| `heading` | `number` | Last saved facing, in degrees. |
| `health` | `integer` | Entity health, 0–65535. New characters use `defaultHealth` (600). |
| `healthCore` | `integer` | 0–100. Default 100. |
| `staminaCore` | `integer` | 0–100. Default 100. |
| `isDead` | `boolean` | |
| `createdAt` | `string` | Database timestamp. |
| `lastPlayed` | `string \| nil` | Set on select, clear, or drop. `nil` until the first selection. |
| `hasAppearance` | `boolean` | The appearance column is non-NULL. |
| `hasClothing` | `boolean` | The clothing column is non-NULL. |
| `hasBarber` | `boolean` | The barber column is non-NULL. |
| `appearance` | `table \| nil` | Present on the full record only. |
| `clothing` | `table \| nil` | Present on the full record only. |
| `barber` | `table \| nil` | Present on the full record only. |

A document that is stored but cannot be decoded becomes `nil` and an error is logged — the rest of the row is still returned. The record is always a copy: changing it does not change the database; call `UpdateCharacter`, `SetAppearance`, `SetClothing`, `SetBarber`, or `SaveState`.

## Config

`server/config.lua`, table `FwCharactersConfig`. Read once at startup — a bad value aborts startup and the resource stays not ready.

| Field | Current value | Rule |
| --- | --- | --- |
| `defaultSpawn.x` | `-174.39` | Finite number. Used only when `CreateCharacter` inserts a row. |
| `defaultSpawn.y` | `633.78` | Finite number. |
| `defaultSpawn.z` | `114.09` | Finite number. |
| `defaultSpawn.heading` | `0.0` | Finite number. |
| `defaultHealth` | `600` | Integer 0–65535. |
| `defaultHealthCore` | `100` | Integer 0–100. |
| `defaultStaminaCore` | `100` | Integer 0–100. |
| `minBirthYear` | `1800` | Integer 1–9999, not above `maxBirthYear`. |
| `maxBirthYear` | `1899` | Integer 1–9999. |

Confirm the spawn coordinates in game. `fw_creator` overwrites them with the player's chosen spawn immediately after selecting the new character.

## Local events

Server events. Listeners run before the export returns.

```lua
AddEventHandler('fw_characters:ready', function(version)
    -- version: manifest string, e.g. '1.0.0'
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
    -- identity or document save, not SaveState. The JSON is not in the event.
end)

AddEventHandler('fw_characters:deleted', function(source, characterId)
end)
```

A resource started after `ready` has already fired should call `IsReady` directly.

On `playerDropped`: an active character is cleared, `last_played` is updated, and `fw_characters:cleared` fires. `fw_core:playerDropped` also fires — the order between the two handlers follows resource start order. On resource stop: active rows get `last_played` updated, but `cleared` is **not** fired for that path.

## State bag

`Player(source).state.fwCharacterId` is the selected character id, or `nil` when none is selected. It replicates. The account id remains `fwPlayerId`.

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

Restricted to the server console — a player gets no output.

```
fw_characters
fw_char_list <serverId>
fw_char_create <serverId> <firstName> <lastName> <male|female> [YYYY-MM-DD]
fw_char_select <serverId> <characterId>
fw_char_delete <serverId> <characterId>
```

| Command | Output |
| --- | --- |
| `fw_characters` | Prints `ready` and how many sources have an active character. |
| `fw_char_list` | Prints `id`, `slot`, name, and gender for that server id. |
| `fw_char_create` | Single-word names only (the console splits on spaces); the export accepts spaces inside `firstName`. Does not select the new character. |

`serverId` is the live server id from `fw_health`, after `playerJoining`. A character can be created only for a connected player.

## Files

| File | Purpose |
| --- | --- |
| `fxmanifest.lua` | Version `1.0.0`. Dependencies `oxmysql` and `fw_core`. |
| `server/config.lua` | Spawn, health defaults, birth years. |
| `sql/characters.sql` | Table definition. |
| `server/main.lua` | Schema, exports, events, console commands. |
