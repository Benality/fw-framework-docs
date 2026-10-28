# fw_stores

Runs in the player's game. Walk up to the clerk in Valentine during open hours and press G. A short loading prompt plays while the counter is gathered, then the counter opens. Type `/shop` to open the same counter from anywhere. Type `/shop` again, press Escape, or press Close, to leave it.

The counter stock is a sample. Closing it puts those piles back. Your bag does change when you settle.

The clerk's goods are their own panel on the left, in the same journal style as the bag. Want and Offer are two panels between them and the bag. Want is what you are asking the clerk for. Offer is what you are putting up. None of those panels can be dragged. The bag on the right is the one you are carrying. Right-click, drag, split, use, give, and drop work the same way they do when you open the bag on its own. So do the pictures on the left of the bag, the clothes, the kit, and the gunbelt. The clerk keeps one pile of each thing. A pile with a number is a limited stock, and that number goes down when you buy it. A pile with no number is always in stock. Drag a pile into Want and choose how many. Canned beans from tier I can be up to 20 in one purchase. A hatchet is one, and a shovel is one. Those are as many as one pile in the satchel can hold. The rest of a limited pile stays on the counter. Shift-click a pile to move all of it onto Want or Offer. When Want already holds a purchase, Shift-click on dollars moves only the price still owed, and Shift-click on the trade goods moves only how many that purchase still needs. Otherwise Shift-click moves the whole pile. Ctrl-click moves 10. A counter pile that does not run out has no whole pile, so Shift does nothing there and Ctrl still takes 10. Click a pile on Want or Offer to put that whole pile back. Drag a pile from your bag into Offer. A pile the clerk does not buy is grey, in the bag and in an open pouch, and it will not move onto Offer. Dollars still move. So does something the clerk buys, and something that pays for a trade once you are trusted enough. A pile in an open pouch can go there too when the clerk takes it. The pile stays where it was. Drag a pile back out of Want to return it to the counter, or out of Offer to take it back. A counter pile will not sit in Offer, and a bag pile will not sit in Want.

Gold numbers are what you pay. A number with a plus is what the clerk pays you. You pay with a dollars pile on Offer. Drag Dollars from your bag, or from an open pouch, onto Offer. The count is how many dollars you are putting up, and that many leave the stack right then. The amount under Offer is the dollars sitting there. Settle spends the price and puts the rest back in the bag. Drag those dollars off Offer, or close the counter, and they come back too. What you put on Offer that the clerk buys is taken out of the bag. What you put on Want is placed in the bag. If the clerk owes you, those dollars are placed in the bag. Canned beans are 4 dollars. Put dollars on Offer to buy them. Once you are trusted enough to trade, four eggs on Offer pay for them instead, and those dollars come back. The clerk buys an egg, an apple, a potato, or an ear of corn for 1 dollar once you are recognized. A bundle of mint is 2 dollars. A jar of honey, a block of beeswax, a bundle of wool, a rabbit pelt, or a fish is 4 dollars. A deer hide is 8 dollars. On this visit everyone is unknown, so the counter takes dollars and does not buy goods.

On the left of the clerk's goods are the same round pictures as the bag. All shows every pile. Kit shows weapons and ammo together. Each other picture shows that one kind, such as food, medical, or containers. A pile that does not match goes grey and stays where it is. Press that picture again, or press All, to show every kind. Want and Offer are not filtered. The bag has its own pictures, the same ones you use when the counter is closed.

Above the clerk's goods is a mark of four small squares, then I, II, III, and IV. I is Unknown, II is Recognized, III is Trusted, and IV is Established. The four squares show every pile, packed to the top of the counter. A numeral shows only that standing, packed the same way. You can press a numeral up through your own standing. A higher one stays grey. This sample stays Unknown, so I can be pressed, and II, III, and IV stay grey. A pile above your standing is locked on the full counter and cannot be bought. Canned peaches are II.

Goods, Work, and Services switch only the clerk's panel. Want, Offer, and the satchel stay where they are. Work says the clerk has no errand on this visit. Services says they are not mending or treating anyone on this visit.

## Command

`/shop` is a client command. It takes no arguments. It opens the store named `ValentineGeneral` from `shared/config.lua` beside the bag. Pressing Shop at the clerk opens that store the same way, after a short loading prompt of about 400 milliseconds while the visit is built. The Shop prompt hides while that counter is on screen, and comes back when it closes. Start `fw_stores` after `fw_satchel` and `fw_inventory`, because the screen uses their pictures and the bag.

Another script on the player's game can open a named store. `storeId` is the store name, such as `ValentineGeneral` or `BlackwaterGeneral`. Nothing, or an empty word, opens `ValentineGeneral`. The counter closes if that event runs again while it is already open.

```lua
--- Opens one store's counter beside the bag.
--- Nothing happens when that store is missing.
---@param storeId string|nil
---@return nil
AddEventHandler('fw_stores:open', function(storeId)
    -- The bag screen keeps the mouse.
end)
```

```lua
TriggerEvent('fw_stores:open', 'ValentineGeneral')
```

## The store list

Write stores in `resources/[framework]/fw_stores/shared/config.lua`. A type is the kind of shop, such as `general`. A store is one counter of that type. Set `type = 'general'` on the store.

A store is a named field on `StoresConfig`. That same table is `FwStoresConfig.stores`. The name is how you read one store.

```lua
StoresConfig.BlackwaterGeneral.blip
StoresConfig.BlackwaterGeneral.goods
StoresConfig.BlackwaterGeneral.access
```

`location`, `blip`, `hours`, `model`, `scenario`, `voice`, `lines`, and `takes` are the place. Write those on the store. `goods` and `buyback` are the shelf. `access` says who may buy, sell, and barter. `pay` is the share of the shelf price. `buy` is the share of the buyback. Leave `access`, `pay`, and `buy` off and the store uses the type. Write them on the store when that town should differ. `StoresConfig.BlackwaterGeneral.access` is the general type's access until Blackwater writes its own.

Set `template = 'general'` to use the shelf in `shared/templates.lua`. Then list only what differs. A line whose `item` is already on the template changes only the fields you write. Canned beans at 5 dollars keep the four-egg trade. A new `item` is added to that store alone. A `buyback` name you write replaces that price. A new name is added. One town's price does not change the template or another town.

```lua
ValentineGeneral = {
    type = 'general',
    template = 'general',
    -- goods = {
    --     { item = 'canned_beans', price = 5 },
    --     { item = 'bread', price = 2 },
    -- },
    -- buyback = { egg = 2 },
}
```

The general stores in Valentine, Strawberry, Saint Denis, Armadillo, Rhodes, Blackwater, Tumbleweed, and Wallace Station use that shelf. `/shop` opens `ValentineGeneral`.

`location` is where the clerk stands, a `vector4` with the heading last. `hours` is `{ open, close }` or `false`. The first hour is included and the second is not, so `{ 6, 21 }` is 6:00 through 20:59. When the first hour is later, such as `{ 21, 6 }`, the shop stays open across midnight. `false` means always open. Outside those hours the clerk is gone, the Shop prompt is hidden, and a sale is refused with `The store is closed.` If the time resource is not running, the shop stays open.

`model` is the ped. `scenario` is the stance, such as `WORLD_HUMAN_SHOPKEEPER`. `voice` is that ped's voice bank. `lines` names the greet, the browse line when the counter opens, and the sale line when the trade goes through. `takes` is how many recordings a line has. A sale picks one of those and cuts off a line already playing.

The clerk appears within `spawnDistance` (50) while the shop is open. The prompt is within `promptDistance` (2.5). Each store's `blip.range` decides when its mark is on the map. `blipRange` (120) is that distance when a store does not set its own.

`blip` is that store's mark. Put it on every store. `name` is the words on the mark. `style` is the blip style, such as `BLIP_STYLE_DEBUG_YELLOW`. `sprite` is the icon, such as `blip_ambient_npc`. `modifiers` is a list of blip modifiers, such as `BLIP_MODIFIER_MP_COLOR_6`. `color` is a color word, such as `yellow`, when you are not listing modifiers. `range` is how close the player must be, in metres, before the mark exists. The Valentine store uses `150`. The mark stays until the player is 20 metres past that, then it leaves the map. Come back inside the range and it returns. Leave `range` off and the store uses `blipRange` (120).

`card` set to `false`, or left out, means the mark has no card. A table is the card that opens when the player hovers that mark. Selecting it keeps the card open. `type` has to be `map_card_mission`. `title` is the large line. `subtitle` is the smaller line. `description` is the body, and `~n~` starts a new line. `footer` is the region line after the player selects the mark. Hovering does not change that line. `txd` is the picture's texture dictionary, such as `map_card_val_generalstore`. `txn` is the texture name inside it. The store sample uses `image`.

```lua
blip = {
    name = 'General Store',
    style = 'BLIP_STYLE_DEBUG_YELLOW',
    sprite = 'blip_ambient_npc',
    range = 150,
    modifiers = { 'BLIP_MODIFIER_MP_COLOR_6' },
    card = false,
},
```

```lua
location = vector4(-324.16, 803.62, 117.88, 272.93),
hours = { 6, 21 },
model = 'u_m_m_valgenstoreowner_01',
scenario = 'WORLD_HUMAN_SHOPKEEPER',
voice = '0478_U_M_M_VALGENSTOREOWNER_01',
lines = { greet = 'WELCOME', browse = 'TAKE_YOUR_TIME', sale = 'FAREWELL_SALE' },
takes = { sale = 5 },
```

Each line in `goods` is one pile on the counter.

- `item` is the item name, such as `canned_beans`. The label, the size, and the category come from that item.
- `price` is the dollars for one, before a standing changes it.
- Leave `count` off and the pile stays in stock. One purchase can take up to a full stack. Canned beans are 20. A hatchet is 1. A shovel is 1.
- `count` is a limited pile. That number goes down on this screen when it is bought.
- `standing` is who may buy it. Leave it off and anyone can. `Recognized` is tier II, `Trusted` is tier III, and `Established` is tier IV. A player below that standing sees the pile locked.
- `price` and `barter` can sit on the same pile. `price` is the dollars. `barter` is the other way to pay, once this visit may barter. Canned beans are 4 dollars, or four eggs. A standing rate changes the dollars. It leaves the trade alone. Eggs that pay for the beans are not also sold. A short stack does not pay for part of the beans.
- `prices` sets the dollars for one standing and skips the type rate. `prices = { Established = 1 }` means Established pays 1 dollar.

`buyback` is what the clerk pays for one item placed on Offer. A number is the list price. A table uses `price` as the list price, and a standing name when that standing should get a different amount.

```lua
egg = { price = 1, Established = 2 }
```

The general shelf lists a number for each good the clerk buys. An egg, an apple, a potato, or an ear of corn is 1 dollar. Mint is 2. Honey, beeswax, wool, a rabbit pelt, or a fish is 4. A deer hide is 8.

On the type, `pay` is the share of `price` the player pays. `1` is the shelf price. `0.9` is nine tenths of it. `buy` is the share of the buyback the clerk pays. The result is a whole dollar. A shelf price of at least 1 dollar stays at least 1 dollar.

`access` on the type says who may buy, who may sell, and who may barter. Each line is a standing name. Leave a line off, or set `Unknown`, and everyone may do that. A word that is not a standing refuses that action. The general store lets everyone buy. Selling waits for Recognized. Barter waits for Trusted.

```lua
access = {
    buy = 'Unknown',
    sell = 'Recognized',
    barter = 'Trusted',
}
```

A pile's own `standing` still locks that pile. Canned peaches stay Recognized even though buying, as an action, is open to everyone.

`ConfigUseReputation` starts as `false`. Every visit is then Unknown. Everyone can buy an unlocked pile. Selling and barter stay closed, because Unknown is below Recognized and Trusted. The rates and the access lines stay in the file for when you turn it on.

Turn `ConfigUseReputation` on when a resource named `fw_reputation` is running. `fw_stores` asks it for this player's standing at this store type. `source` is the player's number for this visit. `storeType` is the type key, such as `'general'`.

```lua
---@param source integer
---@param storeType string
---@return string standing
exports.fw_reputation:GetStanding(source, storeType)
```

Return `Unknown`, `Recognized`, `Trusted`, or `Established`. If `fw_reputation` is missing, the call fails, or the word is anything else, the visit stays Unknown.

Settle takes Offer out of the bag and places Want in the bag. Dollars leave the stack when you put them on Offer, including dollars in a pouch you are still carrying. Settle puts back whatever the price did not use. Closing the counter puts the sample stock back, and puts those dollars back as well. The bag keeps what the clerk already handed over.

Another script on the player's game can open the same screen. `visit` is the clerk's goods, in the shape `/shop` builds. `live` is set for you, so the bag on the right is the one the player is carrying.

```lua
TriggerEvent('fw_satchel:shopToggle', visit)
```

Running it again closes the bag. Nothing is written to the bag.
