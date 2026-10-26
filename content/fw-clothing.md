# fw_clothing

Worn clothes, saved outfits, and wearable state. Version `1.0.0`. Registers with `fw_core` API `1`.

Source: `resources/[framework]/fw_clothing`.

The shop menu and the `murphy_clothing` / `murphy_clothes` event names come from the Murphy clothing resource. This copy does not create `murphy_clothes`, `murphy_outfits`, or `murphy_wearable` tables — everything is stored as one JSON document on `fw_characters.clothing`, written through `exports.fw_characters:SetClothing`.

Depends on `fw_core` and `fw_characters`. `fw_creator` must keep `Config.murphy_clothing = true` — the creator then fires the `murphy_clothing` events this resource listens for. See [fw_creator](fw-creator.md#config).

On success:

```
[fw_clothing] ready. Purchases are not charged until a money resource exists.
```

After a skin apply, `fw_creator` fires `murphy_clothing:loadclothes`; this resource handles that event and dresses the ped from the character's `clothing` document. The skin guard asks `exports.fw_clothing:AreClothesStillApplied()` to confirm the dress-up stuck.

There is no money resource yet. Shop prices still appear and a purchase succeeds, but nothing is deducted — see [Money, items, and jobs](#money-items-and-jobs). Job-locked categories stay hidden because there is no job on the character.

## The `clothing` document

```lua
{
    clothes = {
        hats = { model = 12, texture = 1 },
    },
    outfitId = "1234567", -- "0" when no saved outfit is worn
    outfits = {
        {
            outfitId = "1234567",
            name = "Town clothes",
            clothes = {},
            singleitems = {},
            price = 0,
            gender = "male", -- or "female"
        },
    },
    wearable = {
        ["1234567"] = {}, -- skin table saved with that outfit
    },
}
```

| Key | Meaning |
| --- | --- |
| `clothes` | What the ped is currently wearing. Categories in `Config.EssentialsCategories` are stripped before every save — hair and beard stay on the `barber` document, not here. |
| `outfitId` | The id of the saved outfit currently worn, or `"0"` for none. |
| `outfits` | Outfits this character has saved. An outfit id is a unique 7-digit string per character. Deleting an outfit removes it from this array — no row is left behind. |
| `wearable[outfitId]` | The wearable state saved alongside that outfit. |

There is no offline write — every write requires the online player who currently has that character selected. `SetClothesForChar` returns `false` when the target character isn't the one the player has selected right now.

## Server exports

| Export | Summary |
| --- | --- |
| [`IsBandanaEquipped(source)`](#isbandanaequippedsource) | Whether a mask/bandana category is worn. |
| [`GetPlayerClothes(source)`](#getplayerclothessource) | Worn clothes, standardized to asset hashes. |
| [`GetPlayerTemperatureProtection(source)`](#getplayertemperatureprotectionsource) | Warm/cold item counts and protection level. |
| [`SetClothesForChar(charid, clothes, outfitId)`](#setclothesforcharcharid-clothes-outfitid) | Writes worn clothes for an online, selected character. |

---

### `IsBandanaEquipped(source)`

**Signature**

```lua
---@param source integer
---@return boolean
exports.fw_clothing:IsBandanaEquipped(source)
```

**Returns** `true` when the worn `clothes` include `masks` or `masks_large` with `model > 0`.

---

### `GetPlayerClothes(source)`

**Signature**

```lua
---@param source integer
---@return table
exports.fw_clothing:GetPlayerClothes(source)
```

**Returns** a category map standardized to asset hashes for integrations like `jo_libs`: `{ [category] = { hash, drawable, albedo, normal, material, palette, tint0, tint1, tint2 } }`. `{}` when the character has nothing saved, or the category/model has no matching entry in `shared/murphy_assets.lua` for the ped's detected gender.

---

### `GetPlayerTemperatureProtection(source)`

**Signature**

```lua
---@param source integer
---@return table
exports.fw_clothing:GetPlayerTemperatureProtection(source)
```

**Returns**

```lua
{
    warmItems = 0,       -- count of worn categories flagged "warm" in Config.ClothingTemperature
    coldItems = 0,       -- count flagged "cold"
    coldProtection = 3,  -- Config.TemperatureProtection.cold[warmItems], default 3
    heatProtection = 3,  -- Config.TemperatureProtection.heat[coldItems], default 3
}
```

---

### `SetClothesForChar(charid, clothes, outfitId)`

**Signature**

```lua
---@param charid integer|string
---@param clothes table
---@param outfitId integer|string|nil
---@return boolean success
---@return string|nil err
exports.fw_clothing:SetClothesForChar(charid, clothes, outfitId)
```

Persists worn clothes for the online player who currently has `charid` selected, then re-applies them on their client. The official write path for another resource — e.g. a job script dressing a character in a uniform.

**Parameters**

| Name | Type | Required | Description |
| --- | --- | --- | --- |
| `charid` | `integer \| string` | yes | `fw_characters.id`. Must belong to the character currently selected by an online player. |
| `clothes` | `table` | yes | `{ [category] = { model = <catalog index>, texture = <number or {palette, tint0, tint1, tint2}> } }` — the murphy index format, catalog indexes from `shared/murphy_assets.lua`. Essential categories (skin, hair, beard, …) are stripped automatically; hair and beard belong to `exports.murphy_barber:SetHairForChar` instead. |
| `outfitId` | `integer \| string \| nil` | no | Defaults to `0` (no preset) when omitted. |

**Returns**

| Type | Description |
| --- | --- |
| `true` | Saved and re-applied. Fires `murphy_clothing:clothesUpdated` and tells that client to run `murphy_clothing:loadclothes`. |
| `false, err` | See Errors. |

**Errors**

| `err` value | Condition |
| --- | --- |
| `'SetClothesForChar: charid is required'` | `charid` is `nil` or `''`. |
| `'SetClothesForChar: clothes must be a table { category = { model, texture } }'` | `clothes` is not a table. |
| `"SetClothesForChar: invalid entry for category '<category>' (expected { model = <number>, texture = <number or table> })"` | An entry's `model` is missing or not a number. |
| `"SetClothesForChar: invalid texture for category '<category>'"` | `texture` is present but not a number or table. |
| `'SetClothesForChar: that character is not selected by an online player'` | No connected player currently has `charid` as their active character. |
| `'SetClothesForChar: could not save clothing'` | The underlying `SetClothing` write failed. |

## Client export

| Export | Summary |
| --- | --- |
| [`AreClothesStillApplied()`](#areclothesstillapplied) | Whether this client's last-applied clothes are still on the ped. |

### `AreClothesStillApplied()`

**Signature**

```lua
---@return boolean
exports.fw_clothing:AreClothesStillApplied()
```

Runs on the client that owns the ped. **Returns** `true` when every category this resource last applied is still worn; `false` when clothes were never loaded this session, or at least one category is missing — used by `fw_creator`'s skin guard to detect a skin reset that stripped clothing.

## Callbacks the menu uses

These names stay `murphy_clothing:...` so the existing shop UI can call them unchanged. The server reads and writes the `clothing` document directly — it does not query a separate clothing table.

| Callback | Parameters | Returns |
| --- | --- | --- |
| `murphy_clothing:GetCurrentClothes` | `source` | Worn `clothes` table, outfit id. `{}` and `0` when the character has no clothes. |
| `murphy_clothing:GetCurrentClothesCharid` | `source, characterId` | Same, for a character id owned by that player. |
| `murphy_clothing:GetCurrentOutfitList` | `source` | Array of `{ outfit_id, name, price, gender, clothes, singleitems }`. `clothes` and `singleitems` are JSON strings, matching the legacy row shape. |
| `murphy_clothing:GetOutfit` | `source, id` | Clothes table, gender. |
| `murphy_clothing:SaveOutfit` | `source, clothes, name, fee, male, payment?` | `true, outfitId` on success, `false, outfitId` / `false, nil` otherwise. Charges cash by default, or gold when `payment = { currency = "gold", gold = n }` and `Config.OutfitGold.enabled`. |
| `murphy_clothing:ModifyOutfit` | `source, clothes, id, fee, payment?` | Replaces that outfit's clothes after the same price check as `SaveOutfit`. |
| `murphy_clothing:DeleteOutfit` | `source, id` | Removes that outfit from the character. |
| `murphy_clothing:GiveOutfit` | `source, id, name, price, payment?` | Price check for recovering an outfit already owned. |
| `murphy_clothing:BuySingleItem` | `source, clothesTable, price, isMale` | `true`/`false`. Splits payment across cash, gold, and item currencies per `Config.ItemCurrency`, verifying full affordability before taking anything. |
| `murphy_clothing:SaveWearable` / `GetWearable` | `source, id, data?` / `source, id, characterId?` | Wearable state for one outfit id. |
| `murphy_clothing:GetCharIdentifier` | `source` | Active `fw_characters.id`. |
| `murphy_clothing:GetCharJob` | `source` | `nil, 0` — job-locked shop categories stay hidden until a job resource exists. |
| `murphy_clothing:GetCharMoney` | `source` | A stand-in balance — see [Money, items, and jobs](#money-items-and-jobs). |
| `murphy_clothing:IsBandanaEquipped` | `source` | Same check as the `IsBandanaEquipped` export, exposed as a callback for the menu. |
| `murphy_clothing:GetPlayerTemperatureProtection` | `source` | Same shape as the `GetPlayerTemperatureProtection` export. |

## Network events

| Event | Direction | Payload | What it does |
| --- | --- | --- | --- |
| `murphy_clothing:loadclothes` | server → client | none | Client re-dresses the ped from the saved document. Fired by `fw_creator` after a skin apply. |
| `murphy_clothing:SaveClothesCreator` | — | — | Fired during character creation's clothing step. |
| `murphy_clothing:ResetClothesMenuCreator` | — | — | Resets the in-creator clothing menu. |
| `murphy_clothing:OpenClothesMenuCreator` | — | — | Opens the in-creator clothing menu. |
| `murphy_clothes:ApplyClothesToCharid` | — | — | Applies a specific character's clothes to a ped. |
| `murphy_clothes:Loadlowerbody` / `Loadupperbody` | — | — | Partial body/clothing reload. |
| `murphy_clothes:updateclothes` | client → server | `clothes, outfitid` | Strips essential categories, saves through `FwClothing.SetCurrent`, fires `murphy_clothing:clothesUpdated` on success. Silently dropped if the character isn't loaded yet. |
| `murphy_clothes:retrieveClothes` | server local | `charid, callback` | Callback receives `{ charid, clothes, outfit_id }` (`clothes` as a JSON string), or `false` when that character isn't selected by an online player. |
| `murphy_clothes:retrieveWearable` | server local | `charid, outfit_id, callback` | Callback receives `{ charid, outfit_id, skin }` (`skin` as a JSON string), or `false` when there's no saved wearable state. |
| `murphy_clothing:clothesUpdated` | server local | `source, charid, clothes` | Fired after every successful clothes write — from the menu, or from `SetClothesForChar`. |
| `murphy_clothing:admin:requestOutfitMenu` / `requestWearableMenu` | client → server | none | Opens the outfit/wearable management menu — requires `fw_players.role` of `admin` or `superadmin` via `GetCharGroup`. |

`murphy_clothes:updateclothes` ignores a raw category list from an untrusted source — essential categories (`Config.EssentialsCategories`) are always stripped server-side before the save, regardless of what the client sent.

## Money, items, and jobs

Money and jobs are still not resources. `fw_inventory` exists, and this shop does not call it yet:

- `GetCharMoney` returns `100000000`; `RemoveCurrency` / `AddCurrency` do nothing — a purchase always succeeds and nothing is taken.
- `GetCharGold` returns `0`; `Config.OutfitGold.enabled` is `false`, so the gold payment option stays off.
- `GiveItem` returns `true` and creates nothing; `RemoveItem` / `GetItemAmount` are no-ops. `Config.OutfitItem` is `nil` and `Config.SingleItemCategory` is empty, so the shop never tries to hand out an item.
- `GetCharJob` / `GetCharJobGrade` return `nil, 0` — job-locked categories (`Config.JobLocked`) stay hidden.

When a money or job resource exists, replace `GetCharMoney`, `RemoveCurrency`, `AddCurrency`, `GetCharGold`, `RemoveGold`, `GetCharJob`, and `GetCharJobGrade` in `server/adapters/FWAdapter.lua`. Point `GiveItem`, `RemoveItem`, and `GetItemAmount` at [fw_inventory](fw-inventory.md) when a shop should put an item in the satchel.

## Commands

| Command | Default | Who | Effect |
| --- | --- | --- | --- |
| `/loadclothes` | `Config.LoadClothesCommand` | The player who runs it | Reloads worn clothes onto the current ped. |
| `/ClothesManagement` | `Config.ClothesManagement.Command` | `fw_players.role` of `admin` or `superadmin` | Opens the clothes management menu. |

Shops use the native prompt in `Config.Shops`; the prompt thread sleeps until the player is within 2 meters.

## Config

`shared/config.lua`. `Config.framework` must stay `'fw'` — that is what loads the adapter.

The starter outfit in [`fw_creator`](fw-creator.md#config) still dresses a character that has no saved clothes of its own; it does not write `clothing`. Once the player saves clothes here, the starter outfit leaves that character alone.

## Files

| File | Purpose |
| --- | --- |
| `fxmanifest.lua` | Version `1.0.0`. Dependencies `fw_core`, `fw_characters`. |
| `server/adapters/FWAdapter.lua` | Account/character lookups, startup registration, money/item/job stand-ins. |
| `server/callbacks.lua` | Outfit, wearable, and currency callbacks; `SetClothesForChar`, `GetPlayerClothes`, temperature, bandana exports. |
| `server/clothing_store.lua` | Internal read/write of the `clothing` document (`FwClothing.*`). |
| `server/server.lua` | Admin menu gating, shop routing-bucket instancing. |
| `shared/config.lua` | Prices, categories, temperature, currency, and shop config. |
| `client/function.lua`, `client/client.lua` | Applying clothes to the ped, `AreClothesStillApplied`. |
