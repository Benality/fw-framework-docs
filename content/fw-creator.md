# fw_creator

Player-facing character menu. Version `1.0.0`. Registers with `fw_core` API `1`.

Source: `resources/[framework]/fw_creator`.

The UI and client appearance code come from the Murphy creator. The server adapter — `server/adapters/FWAdapter.lua` and `server/callbacks.lua` — is the part that belongs to this framework: it calls `fw_core` and `fw_characters` exports and does not create its own `murphy_creator` or `murphy_barber` tables.

`.library/trp_creator` is the untouched original. Do not `ensure` it.

On success:

```
[fw_creator] ready
```

Printed after `fw_core` and `fw_characters` are both ready, then `RegisterResource(1)` succeeds. The wait is 120 seconds. Menu events are registered before that wait, so a player who connects during startup still hits the adapter — if the character API isn't ready yet, character select tells the client why.

`Config.framework` must stay `'fw'`. The adapter file is wrapped in that check; any other value loads no adapter, and character select never answers.

## Player flow

1. The client session starts. `client/event.lua` fires `murphy_creator:OpenCharSelect`.
2. The client asks the server for `murphy_creator:getCharacters`.
3. The server puts the player in an empty routing bucket and sends the character list.
4. The player creates a character or plays an existing one.
5. Play selects that character, sends stored health to the client, leaves the routing bucket, and the client moves the ped to the saved coordinates and applies `appearance`.

Creating a character is synchronous on the server from the menu's point of view:

1. `CreateCharacter` with name, gender, birth date, and description.
2. `SelectCharacter` so the new id is active before the callback returns.
3. `SaveState` with the spawn the player picked (`pedspawn` / `pedspawnheading`).
4. `SetAppearance` with the menu's ped data.
5. `SetBarber` from the preset save that follows.

If select fails, the new row is deleted and the callback returns `false`; the client opens character select again.

`CreateCharacter` itself still uses `FwCharactersConfig.defaultSpawn` for the insert — the spawn the player picked is the `SaveState` call that runs immediately after. A later reconnect uses that saved position.

## What is stored where

| Menu data | Where it goes |
| --- | --- |
| First name, last name | `first_name`, `last_name` |
| Gender `Male` / `Female` | `gender` `male` / `female` |
| Birth day, month, year | `birth_date` `YYYY-MM-DD` |
| Lore | `description` |
| Chosen spawn | `x`, `y`, `z`, `heading` via `SaveState` |
| Body, face, ped model | `appearance` |
| Hair, makeup, scars, named presets | `barber` |
| Worn clothes | Not written by `fw_creator` itself. `fw_clothing` owns `clothing` — see [fw_clothing](fw-clothing.md). |

The slot count shown in the menu is `player.characterSlots` from `fw_core` (`fw_players.character_slots`) — `Config.Slots` in the creator config is **not** the limit the create call enforces; change the column on the account instead.

Custom ped models follow `Config.PedPermission`: `default` applies first, then `role[player.role]`, then `identifier[player.license]`. The license value includes the `license:` or `license2:` prefix — a steam id in that map does not match. Delete permission is separate and checks every identifier from `GetPlayerIdentifiers`.

## The `appearance` document

`SetAppearance` stores the ped-data table the menu already built — the framework does not rename those keys. Fields the adapter and client rely on:

| Key | Used for |
| --- | --- |
| `gender` | `Male` or `Female`. The client picks the body set from this. |
| `skintone` | Index into the creator's skin-tone tables. `GetCharSkinTone` hashes the matching head texture. |
| `pedmodel.model` | Model name, e.g. `mp_male` or `mp_female`. |
| `pedmodel.outfit` | Outfit preset index on that model. |
| `head`, `upperbody`, `lowerbody` | Indexes into the skin-tone body tables. |

The rest of the table (sliders, eyes, teeth, overlays the menu keeps on the ped) is stored with those keys and applied by the client. Treat the document as owned by the menu — replace it by calling `SetAppearance` with the whole table the menu would save.

## The `barber` document

```lua
{
    hairstyle = {},          -- menu hair table, or {}
    overlays = {},           -- makeup, keyed by overlay name
    permanentoverlay = {},   -- scars and other permanent overlays, keyed by name
    outfitId = '1230456',    -- menu outfit id string from SavePreset, or a number from updatehairstyle
    presets = {
        {
            hairstyle = {},
            overlays = {},
            name = 'Base',
            outfitId = '1230456',
            price = 0,
            gender = 'male', -- or 'female'
        },
    },
}
```

`SavePreset` always writes `hairstyle`, `overlays`, `permanentoverlay`, and a new `outfitId`. It appends a preset when `name` is not empty and `hairstyle` has entries. The first save from character creation uses the name `Base`.

`updatehairstyle` splits overlays into makeup and permanent by these makeup names: `eyeliners`, `shadows`, `paintedmasks`, `blush`, `blush2`, `foundation`, `lipsticks`. Everything else with a `name` is permanent. That save keeps the existing `presets` array.

## Health the client receives

Selecting a character, and the event `murphy_creator:GetHealthValues`, send:

```lua
{
    hOuter = 600,  -- character.health
    hInner = 100,  -- character.healthCore
    sOuter = 100,  -- there is no stamina-outer column; this is always 100
    sInner = 100,  -- character.staminaCore
}
```

The client event is `murphy_creator:HealthFromCore`. The creator does not write health back when the player disconnects — `fw_characters:SaveState` is the call that would persist a later health change.

## Internal functions

Globals inside `fw_creator`. The menu callbacks call these. **Other resources should use `fw_characters` exports instead** — these exist only so the existing callback file can keep its call shape.

| Function | Summary |
| --- | --- |
| [`GetCharIdentifier(source)`](#getcharidentifiersource) | Active character id. |
| [`GetCharFirstname(source)`](#getcharfirstnamesource-getcharlastnamesource) | Active first name. |
| [`GetCharLastname(source)`](#getcharfirstnamesource-getcharlastnamesource) | Active last name. |
| [`GetCharGroup(source)`](#getchargroupsource) | Account role from `fw_core`. |
| [`CreateNewCharacter(source, data)`](#createnewcharactersource-data) | Create-character callback entry point. |
| [`CanPlayerDeleteCharacter(source)`](#canplayerdeletecharactersource) | Whether this player may delete. |

---

### `GetCharIdentifier(source)`

**Returns** `integer | nil` — the active character id, or `nil` when none is selected.

### `GetCharFirstname(source)`, `GetCharLastname(source)`

**Returns** `string | nil` — the active character's first/last name, or `nil` when none is selected.

### `GetCharGroup(source)`

**Returns** `string | nil` — the account's `role` from `fw_core`, or `nil`. This is the account role, not a job.

### `CreateNewCharacter(source, data)`

**Parameters** — `source: integer`; `data: table`, as the menu sends it:

| Field | Maps to |
| --- | --- |
| `firstname` | `firstName` |
| `lastname` | `lastName` |
| `gender` | `Male` or `Female`, stored as `male` or `female` |
| `birthday`, `birthmonth`, `birthyear` | `birthDate` when all three numbers are present |
| `lore` | `description`. Omitted becomes `''` |
| `pedspawn` or `coords` | `x`, `y`, `z` after select. A `vector3` or a table with `x`, `y`, `z` |
| `pedspawnheading` | `heading`. Missing uses `coords.heading` or `0` |

**Returns** `boolean` — `true` on success, `false` on failure. A second call for the same `source` while the first is still running also returns `false`.

### `CanPlayerDeleteCharacter(source)`

**Returns** `boolean` — `true` when this player may delete, per `Config.CharacterDeletion` (see [Network events](#network-events)).

## Character list entry

`murphy_creator:LaunchCharSelect` receives the list, whether custom peds are allowed, and the slot count. Each entry:

| Field | Source |
| --- | --- |
| `charIdentifier` | `fw_characters.id` |
| `slot` | Account slot. The list is sorted by this; the menu clicks the array index, and delete uses that index. |
| `firstname`, `lastname` | Name columns |
| `nickname` | Nickname or `''` |
| `gender` | `Male` or `Female` |
| `charDesc` | Description |
| `skin` | `{ sex = 'mp_male' }` or `{ sex = 'mp_female' }` |
| `components` | `{}` |
| `coords` | `{ x, y, z, heading }` |
| `isDead` | Boolean |
| `peddata` | `appearance`, or `{}` |
| `informations` | Rebuilt from the row — see below. |
| `age` | `Config.GameYear` minus the birth year, or `''` |
| `group` | Account role |
| `money`, `gold` | `0` |
| `job`, `grade` | `''` |

`informations` is rebuilt on read; it is not its own column.

```lua
{
    lore = '',          -- description
    birthday = '12',    -- day, no leading zero
    birthmonth = '4',
    birthyear = '1870',
    firstname = 'Jane',
    lastname = 'Doe',
}
```

Empty birth fields are `''` when `birth_date` is NULL.

## Server callbacks

Registered on the menu's callback module, `shared/Callbacks.lua`.

| Callback | Parameters | Returns |
| --- | --- | --- |
| `murphy_creator:GetCharInformations` | `source` | The `informations` table, or `{}` when no character is active. |
| `murphy_creator:CreateNewCharacter` | `source, data, peddata` | `true` or `false`. `peddata` is stored with `SetAppearance` when it is a table. |
| `murphy_creator:GetPedData` | `source, characterId?` | `appearance` for that id, or the active character when `characterId` is nil. `{}` when there is no document. |
| `murphy_creator:SavePreset` | `source, hairstyle, overlays, name, fee, male` | `true, outfitId` or `false, outfitId` / `false, nil`. |
| `murphy_creator:UpdatePedData` | `source, peddata` | `true` or `false`. Replaces `appearance` on the active character. `peddata` must be a non-empty table. |
| `murphy_barber_creator:GetCurrentOverlays` | `source, characterId?` | `overlays, permanentoverlay` |
| `murphy_barber_creator:GetCurrentHairs` | `source` | `hairstyle, outfitId, overlays, permanentoverlay` for the active character |
| `murphy_barber_creator:GetCurrentHairsOnCharacter` | `source, characterId` | Same, for that owned id |
| `murphy_barber_creator:GetCharSkinTone` | `source, male, characterId?` | Head-texture hash from `appearance.skintone`, or `0` |

`male` on the skin-tone callback chooses the `Male` or `Female` texture table — it does not read the character's `gender` column. `?` marks an argument that may be `nil`.

## Network events

The client still uses the `murphy_creator` names so the menu did not have to be rewritten. One new event selects the framework character.

| Event | Direction | Payload | What it does |
| --- | --- | --- | --- |
| `murphy_creator:getCharacters` | client → server | none | Builds the list, or fires `CharSelectFailed` with a reason. |
| `murphy_creator:LaunchCharSelect` | server → client | characters, pedPermission, slots | Opens the menu. |
| `murphy_creator:CharSelectFailed` | server → client | reason string | The client prints it and retries once. |
| `murphy_creator:OpenCharSelect` | client local, also server → client | none | Asks for the list again. |
| `fw_creator:selectCharacter` | client → server | characterId | `SelectCharacter`, leave the routing bucket, then `HealthFromCore`. |
| `murphy_creator:GetHealthValues` | client → server | none | Sends `HealthFromCore` for the active character. |
| `murphy_creator:HealthFromCore` | server → client | health table | See [Health the client receives](#health-the-client-receives). |
| `murphy_creator:deleteCharacter` | client → server | list index | Resolves that index in the list last sent to this player, then `DeleteCharacter`. |
| `murphy_creator:deleteCharacterClient` | server → client | list index | Drops that entry from the local list. |
| `murphy_creator:RemovePlayerFromInstance` | client → server | none | Routing bucket `0`. |
| `fw_creator:characterSelected` | server → client | none | Registered on the client; fires `murphy_creator:loadskin`. The select path currently loads the skin itself after `fw_creator:selectCharacter`. |
| `murphy_barber_creator:updatehairstyle` | client → server | hairstyle, overlays, outfitId | Writes the barber document on the active character. |
| `murphy_creator:retrievehairstyle` | server local | characterId, callback | Calls `callback` with the barber table, or `false`. |

`deleteCharacter` ignores a raw character id — the argument is the index in the list that `getCharacters` last sent to that source. A stale index reopens character select.

Delete permission is `Config.CharacterDeletion`. With `restrictToRoles` false, every player may delete. With it true, `Config.CharacterDeletion.identifier` is checked against the player's identifiers, then `Config.CharacterDeletion.role` against `player.role`.

## Commands

| Command | Default | Who | Effect |
| --- | --- | --- | --- |
| `/mc` | `Config.LoadSkinCommand` | The player who runs it | Reloads appearance on that client. |
| `/deletechar [characterId]` | `Config.Commands.deleteCharacter` | `fw_admin` admins, roles in `allowedRoles`, or identifiers in `allowedIdentifiers`. The console passes the role check but is then refused, because delete needs the owning player. | `DeleteCharacter` for the caller. Someone else's id returns `character not found`. |
| `/secondchance [serverId]` | `Config.Commands.secondChance` | Same admin, role, and identifier checks. The console may run it. | Opens the customization menu on that player (`murphy_creator:SecondChance`). |

Chat lines use the `chat:addMessage` event with the tag `fw_creator`.

## Config

File: `shared/config.lua`.

| Key | Current | Effect |
| --- | --- | --- |
| `Config.framework` | `'fw'` | Loads the framework adapter. |
| `Config.murphy_clothing` | `true` | `fw_clothing` saves worn clothes to the character. Must stay `true` while `fw_clothing` is ensured — see [fw_clothing](fw-clothing.md). |
| `Config.CreatorClothing.allowCustomization` | `false` | Hides the clothes pin. |
| `Config.CreatorClothing.starterOutfit.enabled` | `true` | After a skin load, a character with none of the listed clothing categories gets game outfit preset `3` (`male` and `female`). |
| `Config.UseSpawnSelector` | `true` | The menu asks where to spawn. `false` uses `Config.DefaultSpawnLocationId`. |
| `Config.DefaultSpawnLocationId` | `1` | Index into `SpawnLocation` in `client/settings/settings.lua`. |
| `Config.GameYear` | `1899` | Used only to show age in the character list. |
| `Config.LoadSkinCommand` | `'mc'` | Client command that reloads the skin. |
| `Config.DevMode` | `false` | `true` lets F2 request the character list again. |
| `Config.Debug` | `false` | Prints `[murphy_creator]` debug lines. |
| `Config.ExternalClothingEvents` | `{}` | Client events fired after a skin apply, for a clothing script that is not murphy_clothing. |
| `Config.PedPermission` | default true | Who may pick a custom ped. `role` and `identifier` override `default`. |
| `Config.CharacterDeletion` | enabled, not restricted | Who may delete from the menu. |
| `Config.Commands` | deletechar, secondchance | Admin commands above. |

`Config.Slots` is leftover menu config — the number of slots the player can create is `fw_players.character_slots`.

## Routing buckets

`server/function.lua`:

- `PutPlayerinInstance(serverId)` puts the player in a random empty bucket. Character select does this.
- `RemovePlayerFromInstance(serverId)` returns them to bucket `0`. Play and a successful create do this.

## Files that are the framework adapter

| File | Purpose |
| --- | --- |
| `server/adapters/FWAdapter.lua` | Account and character calls, character list, select, delete, commands. |
| `server/callbacks.lua` | Appearance and barber reads and writes. |
| `server/function.lua` | Routing buckets. |
| `client/adapters/FWAdapter.lua` | Menu flow. Play selects through `fw_creator:selectCharacter`, then moves the ped and loads the skin. |
| `client/event.lua` | Opens character select when the network session starts. |
| `shared/config.lua` | `Config.framework = 'fw'` and the switches above. |

The rest of `client/` and `ui/` is the menu. Keep the `murphy_creator` callback and event names when you change the menu, or update both sides together.

## Not done yet

- The starter outfit (`Config.CreatorClothing.starterOutfit`) is a client preset, not a saved document — it stands down once `fw_clothing` has saved real clothes for that character. Height is stored on `appearance.height` and applied on the client. The old VORP scale event is not sent.
- Health on the ped is not saved back to the row on disconnect.
- `Config.PersistVitals` is a leftover from other frameworks — this adapter does not run a health autosave.
- Inventory, money, and jobs are the zeros and empty strings in the character list.
