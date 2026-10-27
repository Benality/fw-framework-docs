# fw_money

Runs on the server. Read how many dollars a player is carrying, give them dollars, or take dollars away.

The dollars are a real stack in the satchel. The number you spend is the balance stored for that character. `GetCash` is that balance. It is the dollars in the satchel. Dollars inside a money bag, a chest, or any other storage stay there until someone takes them out. They are not part of `GetCash`.

One pile holds at most 500 dollars. A larger amount is several piles. `AddCash` fills a pile up to 500, then starts another pile. Each new pile needs an empty cell. 1,200 dollars is two piles of 500 and one of 200.

`source` is the player's number for this visit. An amount is a whole number of dollars. Nothing, written as `nil`, means the call did not do it, and the second value is the reason.

A bank is a later script. It will take these piles and keep a balance of its own.

Ask `IsReady` first. The server can say `fw_money` has started a moment before the tables are ready. Keep asking for about two minutes.

## Calls

### IsReady

Runs on the server. Yes after the currency tables exist.

```lua
--- True when cash can be read and changed.
---@return boolean
exports.fw_money:IsReady()
```

```lua
while not exports.fw_money:IsReady() do
    Wait(100)
end
```

### GetCash

Runs on the server. The dollars in the satchel. The first time you ask, the balance is created at 0.

Nothing when no character is selected, or the balance cannot be read.

```lua
--- The dollars in this player's satchel.
--- Nothing when no character is selected.
---@param source integer
---@return integer|nil amount
---@return string|nil err
exports.fw_money:GetCash(source)
```

```lua
local dollars, err = exports.fw_money:GetCash(source)
if dollars == nil then
    print(err)
    return
end
```

### AddCash

Runs on the server. Adds dollars to the satchel and to the balance. Fills piles up to 500, then starts another pile.

False, and nothing changes, when the satchel has no room for the new piles. The amount has to be from 1 to 100000.

```lua
--- Adds dollars to the satchel.
--- False, and nothing changes, when the new piles do not fit.
---@param source integer
---@param amount integer
---@return boolean ok
---@return string|nil err
exports.fw_money:AddCash(source, amount)
```

```lua
local added, err = exports.fw_money:AddCash(source, 1200)
if not added then
    print(err)
    return
end
```

### RemoveCash

Runs on the server. Takes dollars out of the satchel piles and lowers the balance by the same amount.

False, and nothing changes, when the satchel does not hold that much. Dollars sitting in a money bag are not taken. The player has to move them into the satchel first. Dollars set aside in an open trade stay there, and the reason is `That item is in a trade.`

```lua
--- Takes dollars out of the satchel.
--- False, and nothing changes, when the satchel does not hold that much.
---@param source integer
---@param amount integer
---@return boolean ok
---@return string|nil err
exports.fw_money:RemoveCash(source, amount)
```

```lua
local paid, err = exports.fw_money:RemoveCash(source, 12)
if not paid then
    print(err)
    return
end
```

### GetCurrency

Runs on the server. The same as `GetCash` for a named currency. `cash` is the one that exists now. A later currency is another item with category `currency` and its own name.

```lua
--- The satchel balance for one currency.
--- Nothing when no character is selected, or the name is not a currency.
---@param source integer
---@param name string
---@return integer|nil amount
---@return string|nil err
exports.fw_money:GetCurrency(source, name)
```

```lua
local dollars, err = exports.fw_money:GetCurrency(source, 'cash')
```

### AddCurrency

Runs on the server. The same as `AddCash` for a named currency. False when no item uses that currency name.

```lua
--- Adds a currency to the satchel.
--- False, and nothing changes, when the new piles do not fit.
---@param source integer
---@param name string
---@param amount integer
---@return boolean ok
---@return string|nil err
exports.fw_money:AddCurrency(source, name, amount)
```

```lua
local added, err = exports.fw_money:AddCurrency(source, 'cash', 50)
```

### RemoveCurrency

Runs on the server. The same as `RemoveCash` for a named currency.

```lua
--- Takes a currency out of the satchel.
--- False, and nothing changes, when the satchel does not hold that much.
---@param source integer
---@param name string
---@param amount integer
---@return boolean ok
---@return string|nil err
exports.fw_money:RemoveCurrency(source, name, amount)
```

```lua
local paid, err = exports.fw_money:RemoveCurrency(source, 'cash', 50)
```

## What the player can do

The player can split a pile, give that pile, offer it in a trade, or drag it into a pouch whose categories include `currency`. The money bag item accepts `currency`. A herb pouch does not.

Drop is refused on the dollars themselves. The player puts them in a money bag, then drops the bag. The dollars stay inside that bag.

Discard is refused on the dollars. Discard is also refused on a bag that still holds dollars. The reason is `Empty the bag first.` Take the dollars out, or drop the bag.

`AddItem` and `RemoveItem` on the item `cash` change the same balance. A shop should still call `AddCash` and `RemoveCash`, so it does not have to know the item name.

## Reasons you can get back

| Reason | What it means |
| --- | --- |
| `Select a character first.` | That player has no character selected. |
| `Unknown currency.` | No item is marked with that currency name. |
| `Invalid amount.` | The amount was not a whole number from 1 to 100000. |
| `Not enough room.` | The satchel has no empty cell for a new pile. |
| `You do not have that much.` | The satchel balance is smaller than the amount. |
| `That item is in a trade.` | Those dollars are sitting in an open trade. |
| `Money is not ready.` | `fw_money` is not started. |
| `fw_money is not ready` | The tables are not ready yet. Ask `IsReady`. |
| `Could not read the cash.` | The balance could not be read. |
| `Could not save the cash.` | The balance could not be written. Nothing else was kept from that change. |

## Console command

| Command | Where | What it does |
| --- | --- | --- |
| `fw_givemoney <serverId> <amount>` | Server window, or `/fw_givemoney` by an in-game admin | Calls `AddCash` for that player. `serverId` is their number for this visit. `amount` is whole dollars, from 1 to 100000. A player who is not an admin gets `You are not an admin.` The server window, and that admin's console, get an INFO line when the dollars are given. |

```
fw_givemoney 1 200
```

## Calls the satchel uses

A shop uses the calls above. The satchel uses these when a pile moves between a character and a storage. Do not call them from a shop. The amount is the new balance, including 0.

`source` is not used here. `characterId` is `fw_characters.id`. `storageId` is the storage id from [fw_inventory](fw-inventory.md), such as `item:` plus a container id, or a chest id.

```lua
--- The character balance, without creating a row.
--- The second value is false when the row does not exist yet.
---@param characterId integer
---@param name string
---@return integer|nil amount
---@return boolean|string|nil existsOrErr
exports.fw_money:ReadCharacter(characterId, name)
```

```lua
--- The character balance, creating a zero row the first time.
---@param characterId integer
---@param name string
---@return integer|nil amount
exports.fw_money:EnsureCharacter(characterId, name)
```

```lua
--- Replaces the character balance.
---@param characterId integer
---@param name string
---@param amount integer
---@return boolean
exports.fw_money:SetCharacter(characterId, name, amount)
```

```lua
--- The storage balance. Zero when that storage has no row.
---@param storageId string
---@param name string
---@return integer|nil amount
exports.fw_money:GetStorage(storageId, name)
```

```lua
--- The storage balance, and whether a row exists.
---@param storageId string
---@param name string
---@return integer|nil amount
---@return boolean|string|nil existsOrErr
exports.fw_money:ReadStorage(storageId, name)
```

```lua
--- Replaces the storage balance. Zero removes the row.
---@param storageId string
---@param name string
---@param amount integer
---@return boolean
exports.fw_money:SetStorage(storageId, name, amount)
```

```lua
--- True when this storage still has a currency amount above zero.
--- Also true when the balance cannot be read, so a bag is not thrown away by mistake.
---@param storageId string
---@return boolean
exports.fw_money:HoldsCurrency(storageId)
```

```lua
--- Deletes every currency row for a storage that is being removed.
---@param storageId string
---@return nil
exports.fw_money:ForgetStorage(storageId)
```

The balance is written before the satchel is saved. If the satchel save fails, the previous balance is put back. The next time the character is selected, a balance that does not match the piles wins, and the piles are rebuilt in stacks of 500. When the satchel is too full to place every dollar, the bag is left as it was and the server window shows a warning.
