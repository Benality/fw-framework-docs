# fw_stores

Runs in the player's game. Walk up to the clerk during open hours and press G. A short loading prompt plays while the counter is gathered, then the counter opens. Press Escape, or press Close, to leave it. Pressing Shop again closes it too.

Closing the counter puts the shelf back the way the store lists it. What you settled stays in your bag.

The clerk's goods are their own panel on the left, in the same journal style as the bag. Want and Offer are two panels between them and the bag. Want is what you are asking the clerk for. Offer is what you are putting up. None of those panels can be dragged. The bag on the right is the one you are carrying. Right-click, drag, split, use, give, and drop work the same way they do when you open the bag on its own. So do the pictures on the left of the bag, the clothes, the kit, and the gunbelt. The clerk keeps one pile of each thing. A pile with a number is a limited stock, and that number goes down when you buy it. A pile with no number is always in stock. Drag a pile into Want and choose how many. Canned beans from tier I can be up to 20 in one purchase. A hatchet is one, and a shovel is one. Those are as many as one pile in the satchel can hold. The rest of a limited pile stays on the counter. Shift-click a pile to move all of it onto Want or Offer. When Want already holds a purchase, Shift-click on dollars moves only the price still owed, and Shift-click on the trade goods moves only how many that purchase still needs. Otherwise Shift-click moves the whole pile. Ctrl-click moves 10. A counter pile that does not run out has no whole pile, so Shift does nothing there and Ctrl still takes 10. Click a pile on Want or Offer to put that whole pile back. Drag a pile from your bag into Offer. A pile the clerk does not buy is grey, in the bag and in an open pouch, and it will not move onto Offer. Dollars still move. So does something the clerk buys, and something that pays for a trade once you are trusted enough. A pile in an open pouch can go there too when the clerk takes it. The pile stays where it was. Drag a pile back out of Want to return it to the counter, or out of Offer to take it back. A counter pile will not sit in Offer, and a bag pile will not sit in Want.

Gold numbers are what you pay. A number with a plus is what the clerk pays you. You pay with a dollars pile on Offer. Drag Dollars from your bag, or from an open pouch, onto Offer. The count is how many dollars you are putting up, and that many leave the stack right then. The amount under Offer is the dollars sitting there. Settle spends the price and puts the rest back in the bag. Drag those dollars off Offer, or close the counter, and they come back too. What you put on Offer that the clerk buys is taken out of the bag. What you put on Want is placed in the bag. If the clerk owes you, those dollars are placed in the bag. A tin of beans is 5 dollars at Worths in Valentine and 4 dollars in the other towns. Put dollars on Offer to buy it. A sack of flour is 10 dollars. A hatchet is 80 dollars, and it stays locked until you are established with the clerk. Worths also sells a loaf of bread for 2 dollars. The clerk buys an egg for 2 dollars at Worths and 1 dollar in the other towns, once you are recognized. An apple, a potato, or an ear of corn is 1 dollar. A bundle of mint is 2 dollars. A jar of honey, a block of beeswax, a bundle of wool, a rabbit pelt, or a fish is 4 dollars. A deer hide is 8 dollars. On this visit everyone is unknown, so the counter takes dollars and does not buy goods.

On the left of the clerk's goods are the same round pictures as the bag. All shows every pile. Kit shows weapons and ammo together. Each other picture shows that one kind, such as food, medical, or containers. A pile that does not match goes grey and stays where it is. Press that picture again, or press All, to show every kind. Want and Offer are not filtered. The bag has its own pictures, the same ones you use when the counter is closed.

Above the clerk's goods is a mark of four small squares, then I, II, III, and IV. I is Unknown, II is Recognized, III is Trusted, and IV is Established. The four squares show every pile, packed to the top of the counter. A numeral shows only that standing, packed the same way. You can press a numeral up through your own standing. A higher one stays grey. With reputation off, every visit is Unknown, so I can be pressed, and II, III, and IV stay grey. A pile above your standing is locked on the full counter and cannot be bought. A tin of peaches is II. The counter scrolls when the shelf is taller than the page.

Goods, Work, and Services switch only the clerk's panel. Want, Offer, and the satchel stay where they are. Work says the clerk has no errand on this visit. Services says they are not mending or treating anyone on this visit.

## Opening the counter

Pressing Shop at the clerk opens that store, after a short loading prompt while the visit is built. That pause is `openPauseMs` on `FwStoresConfig`, 400 milliseconds unless you change it. The Shop prompt hides while that counter is on screen, and comes back when it closes. Start `fw_stores` after `fw_satchel` and `fw_inventory`, because the screen uses their pictures and the bag.

Another script on the player's game can open a named store. `storeId` is the store name, such as `ValentineGeneral` or `BlackwaterGeneral`. A missing name does nothing. The counter closes if that event runs again while it is already open.

```lua
--- Opens one store's counter beside the bag.
--- Nothing happens when that store is missing, or when the name is empty.
---@param storeId string
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

A type on `FwStoresConfig.types` is one kind of shop. `general` is the first. A new kind is another entry there: a label, a template name, a scenario, lines, a mark, and who may trade. A store that names that type receives those fields.

Write `type`, `name`, `location`, `model`, `voice`, and `hours` on the store. `type` is the kind, such as `general`. `name` is the words on the map mark. `hours` belongs to this shop and is not taken from the type. Leave `template`, `scenario`, `lines`, `takes`, `blip`, `access`, `pay`, `buy`, and `rows` off and the store uses its type. Write one of those fields when this shop should differ. A field you write replaces that field for this shop. A `blip` field you write replaces that one field, and the rest of the mark stays with the type. `template = false` skips the type's shelf and keeps only the `goods` and `buyback` written on this store.

`rows` on a type is how many rows that counter uses. Leave it off and the counter stays at `counter.rows`, which is 14. The gunsmith sets 20 so a rifle still packs. Raising the shared 14 would add empty rows to the general store. A store can write `rows` when that one shop should differ.

Leave `pay` and `buy` off a type and that shop uses the general rates. Write them on the type when that trade should differ. `access.sell` set to a word that is not a standing, such as `Closed`, means the clerk never buys. The gunsmith is set that way. The butcher, the trapper, and the bait shop set `sell` to `Unknown`, so a first visit can still sell. The doctor and the saloon keep selling at Recognized, the same as the general store. Barter stays closed on these shelves until a type has a real trade.

`goods` and `buyback` are the shelf. `access` says who may buy, sell, and barter. `pay` is the share of the shelf price. `buy` is the share of the buyback. `StoresConfig.BlackwaterGeneral.access` matches the general type until Blackwater writes its own.

The general type sets `template` to `general`, the shelf in `shared/templates.lua`. Then list only what differs. A line whose `item` is already on the template changes only the fields you write. A new `item` is added to that store alone. A `buyback` name you write replaces that price. A new name is added. One town's price does not change the template or another town.

Worths does that. The tin of beans stays on the shelf and becomes 5 dollars. Bread is not on the template, so Worths alone sells it, for 2 dollars. An egg buyback of 2 replaces the template price of 1. Keane's Saloon does the same kind of change for one drink: whiskey there is 8 dollars, and the rest of the saloon shelf stays as written.

```lua
ValentineGeneral = {
    type = 'general',
    name = 'Worths General Store',
    location = vector4(-324.16, 803.62, 117.88, 272.93),
    model = 'u_m_m_valgenstoreowner_01',
    voice = '0478_U_M_M_VALGENSTOREOWNER_01',
    hours = false,
    goods = {
        { item = 'consumable_can_beans', price = 5 },
        { item = 'consumable_bread', price = 2 },
    },
    buyback = {
        egg = 2,
    },
}
```

The other general stores, in Strawberry, Saint Denis, Armadillo, Rhodes, Blackwater, Tumbleweed, and Wallace Station, keep the general type's shelf.

`location` is where the clerk stands, a `vector4` with the heading last. `hours` is written on that store. It is `{ open, close }` or `false`. The first hour is included and the second is not, so `{ 6, 21 }` is 6:00 through 20:59. When the first hour is later, such as `{ 21, 6 }`, the shop stays open across midnight. `false` means always open. The general stores are open all day. Outside the hours you write, the clerk is gone, the Shop prompt is hidden, and a sale is refused with `The store is closed.` If the time resource is not running, the shop stays open.

`model` is the ped. `scenario` is the stance, such as `WORLD_HUMAN_SHOPKEEPER`. `voice` is that ped's voice bank. `lines` names the greet, the browse line when the counter opens, and the sale line when the trade goes through. `takes` is how many recordings a line has. A sale picks one of those and cuts off a line already playing.

The clerk appears within `spawnDistance` (50) while the shop is open. The prompt is within `promptDistance` (2.5). Each store's `blip.range` decides when its mark is on the map. `blipRange` (120) is that distance when a store does not set its own.

`blip` is that store's mark. Leave it off and the store uses its type's mark. The `name` on the store is the words on that mark. `style` is the blip style, such as `BLIP_STYLE_DEBUG_YELLOW`. `sprite` is the icon, such as `blip_ambient_npc`. `modifiers` is a list of blip modifiers, such as `BLIP_MODIFIER_MP_COLOR_6`. `color` is a color word, such as `yellow`, when you are not listing modifiers. `range` is how close the player must be, in metres, before the mark exists. The general type's mark uses `150`. The mark stays until the player is 20 metres past that, then it leaves the map. Come back inside the range and it returns. Leave `range` off a mark you write yourself and the store uses `blipRange` (120).

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

- `item` is the item name, such as `consumable_can_beans`. The label, the size, and the category come from that item.
- `price` is the dollars for one, before a standing changes it. The full list is at the end of this page.
- Leave `count` off and the pile stays in stock. One purchase can take up to a full stack. A tin of beans is 20. A hatchet is 1. A shovel is 1.
- `count` is a limited pile. That number goes down on this screen when it is bought.
- `standing` is who may buy it. Leave it off and anyone can. `Recognized` is tier II, `Trusted` is tier III, and `Established` is tier IV. A player below that standing sees the pile locked.
- `price` and `barter` can sit on the same pile. `price` is the dollars. `barter` is the other way to pay, once this visit may barter. The general shelf lists a tin of beans at 4 dollars and does not put a trade on it. Valentine writes the price as 5. A standing rate changes the dollars. It leaves the trade alone. A short stack does not pay for part of a trade.
- `prices` sets the dollars for one standing and skips the type rate. `prices = { Established = 1 }` means Established pays 1 dollar.

`buyback` is what the clerk pays for one item placed on Offer. A number is the list price. A table uses `price` as the list price, and a standing name when that standing should get a different amount.

```lua
egg = { price = 1, Established = 2 }
```

The general shelf lists a number for each good the clerk buys. An egg, an apple, a potato, or an ear of corn is 1 dollar. Valentine pays 2 dollars for an egg. Mint is 2. Honey, beeswax, wool, a rabbit pelt, or a fish is 4. A deer hide is 8.

On the type, `pay` is the share of `price` the player pays. `1` is the shelf price. `0.9` is nine tenths of it. `buy` is the share of the buyback the clerk pays. The result is a whole dollar. A shelf price of at least 1 dollar stays at least 1 dollar.

`access` on the type says who may buy, who may sell, and who may barter. Each line is a standing name. Leave a line off, or set `Unknown`, and everyone may do that. A word that is not a standing refuses that action. The general store lets everyone buy. Selling waits for Recognized. Barter waits for Trusted.

```lua
access = {
    buy = 'Unknown',
    sell = 'Recognized',
    barter = 'Trusted',
}
```

A pile's own `standing` still locks that pile. A tin of peaches stays Recognized even though buying, as an action, is open to everyone.

`FwStoresConfig.useReputation` starts as `false`. Every visit is then Unknown. Everyone can buy an unlocked pile. Selling and barter stay closed, because Unknown is below Recognized and Trusted. The rates and the access lines stay in the file for when you turn it on.

Turn `useReputation` on when a resource named `fw_reputation` is running. `fw_stores` asks it for this player's standing at this store type. `source` is the player's number for this visit. `storeType` is the type key, such as `'general'`.

```lua
---@param source integer
---@param storeType string
---@return string standing
exports.fw_reputation:GetStanding(source, storeType)
```

Return `Unknown`, `Recognized`, `Trusted`, or `Established`. If `fw_reputation` is missing, the call fails, or the word is anything else, the visit stays Unknown.

Settle takes Offer out of the bag and places Want in the bag. Dollars leave the stack when you put them on Offer, including dollars in a pouch you are still carrying. Settle puts back whatever the price did not use. Closing the counter puts the shelf back the way the store lists it, and puts those dollars back as well. The bag keeps what the clerk already handed over.

Another script on the player's game can open the same screen. `visit` is the clerk's goods for that store. `live` is set for you, so the bag on the right is the one the player is carrying.

```lua
TriggerEvent('fw_satchel:shopToggle', visit)
```

Running it again closes the bag. This event opens the screen. Settle is what changes the bag.

## List prices

The number is the list price. An Unknown visit pays that many dollars. Recognized pays 95 percent, Trusted 90 percent, and Established 85 percent, rounded to a whole dollar. A price of at least 1 dollar stays at least 1 dollar.

A later job should sit under these prices. A short chore pays about 1 or 2 dollars. A day of labor pays about 4 to 8 dollars, under a tin of food. A dangerous job can approach a small tool. It should stay well under a hatchet at 80 dollars.

The shelf is food the counter keeps, household goods, a short rack of garden seed, and a few tools. The clerk does not buy these goods back. The buyback list is farm and hunt goods, and those prices stay below the shelf.

### Rank I, anyone

| Item | List price |
| --- | --- |
| Bannock | 2 |
| Bar of Soap | 3 |
| Beef Jerky | 3 |
| Biscuits | 3 |
| Bread Roll | 1 |
| Candle | 2 |
| Chewing Gum | 1 |
| Cornbread | 2 |
| Crackers | 2 |
| Hard Candy | 1 |
| Oatcakes | 2 |
| Raisins | 1 |
| Roasted Peanuts | 2 |
| Salt | 2 |
| Sarsaparilla | 2 |
| Tin of Beans | 4 |
| Tin of Crackers | 2 |
| Washcloth | 2 |
| Wooden Spoon | 1 |

### Rank II, Recognized

| Item | List price |
| --- | --- |
| Bag of Rice | 8 |
| Bag of Salt | 4 |
| Bag of Sugar | 12 |
| Bandage | 3 |
| Bean Seed | 2 |
| Black Tea | 6 |
| Box of Bandages | 8 |
| Box of Biscuits | 5 |
| Cabbage Seed | 2 |
| Carrot Seed | 2 |
| Cloth | 6 |
| Coil of Rope | 6 |
| Corn Seed | 2 |
| Dill Pickles | 4 |
| Empty Bucket | 4 |
| Empty Jar | 2 |
| Horse Brush | 6 |
| Ink | 3 |
| Jar of Honey | 6 |
| Kitchen Towel | 3 |
| Lard | 5 |
| Lettuce Seed | 2 |
| Nails | 3 |
| Notebook | 4 |
| Oatmeal | 3 |
| Oats | 4 |
| Onion Vidalia Seed | 2 |
| Paper | 2 |
| Pen | 2 |
| Potato Seed | 2 |
| Pumpkin Seed | 2 |
| Sack of Cornmeal | 8 |
| Sack of Flour | 10 |
| Sack of Horse Feed | 8 |
| Spool of String | 4 |
| Sugar Cubes | 3 |
| Tea | 4 |
| Tin of Apricots | 6 |
| Tin of Carrots | 4 |
| Tin of Corn | 4 |
| Tin of Corned Beef | 7 |
| Tin of Kidney Beans | 5 |
| Tin of Peaches | 6 |
| Tin of Pears | 5 |
| Tin of Peas | 4 |
| Tin of Pork | 6 |
| Tin of Salmon | 8 |
| Tin of Sweet Corn | 4 |
| Tin of Vegetables | 5 |
| Tomato Seed | 2 |
| Tonic | 8 |
| Wallet | 10 |
| Wheat Seed | 2 |
| Yeast | 3 |

### Rank III, Trusted

| Item | List price |
| --- | --- |
| Coffee Beans | 15 |
| Coffee Pot | 28 |
| Ground Coffee | 18 |
| Hammer | 25 |
| Lamp Oil | 15 |
| Pepper | 8 |
| Skillet | 18 |
| Tobacco | 12 |

### Rank IV, Established

| Item | List price |
| --- | --- |
| Hatchet | 80 |
| Shovel | 60 |

The other shops use the same standing and the same rates. A firearm on the gunsmith shelf is at least 120 dollars, above the hatchet. Ammunition stays cheap. The gunsmith does not buy anything back. The butcher pays more than the general store for the same hide. The trapper pays more than the butcher. The bait shop pays more than the general store for a fish. With reputation off, rank II and up stay locked. The butcher, the trapper, and the bait shop still buy, because their sell line is Unknown.

### Gunsmith

| Standing | Item | List price |
| --- | --- | --- |
| I | Arrows | 2 |
| I | Pistol Ammo | 2 |
| I | Revolver Ammo | 2 |
| I | Varmint Rounds | 2 |
| I | Repeater Ammo | 3 |
| I | Rifle Ammo | 3 |
| I | Shotgun Shells | 3 |
| I | Hunting Knife | 12 |
| I | Lasso | 15 |
| II | Express Revolver, Pistol, Repeater, or Rifle Ammo | 6 |
| II | Binoculars | 70 |
| II | Bow | 90 |
| II | Sawed-Off Shotgun | 120 |
| II | Cattleman Revolver | 130 |
| II | Varmint Rifle | 140 |
| II | Double-Action Revolver | 150 |
| II | Carbine Repeater | 160 |
| II | Double-Barrel Shotgun | 170 |
| III | Throwing Knives | 40 |
| III | High Velocity or Split Point Ammo | 8 |
| III | Improved Binoculars | 120 |
| III | Schofield Revolver | 180 |
| III | Springfield Rifle | 190 |
| III | Volcanic Pistol | 200 |
| III | Pump Shotgun | 200 |
| III | Lancaster Repeater | 210 |
| IV | Fire Arrows | 12 |
| IV | Elephant Rifle Ammo | 12 |
| IV | Poison Arrows | 15 |
| IV | Incendiary Shotgun Shells | 18 |
| IV | Fire Bottle | 20 |
| IV | Explosive Ammo | 20 |
| IV | Dynamite Arrows | 25 |
| IV | Dynamite | 30 |
| IV | Improved Bow | 250 |
| IV | LeMat Revolver | 280 |
| IV | Navy Revolver | 300 |
| IV | Bolt Action Rifle | 320 |
| IV | Semi-Auto Pistol | 340 |
| IV | Repeating Shotgun | 340 |
| IV | Mauser Pistol | 360 |
| IV | Semi-Auto Shotgun | 380 |
| IV | Carcano Rifle | 400 |
| IV | M1899 Pistol | 420 |
| IV | Rolling Block Rifle | 450 |
| IV | Elephant Rifle | 500 |

The same five gunsmiths sell this shelf: Kuo Chao & Co. in Saint Denis, Rhodes, Schultz in Annesburg, Tumbleweed, and Valentine.

### Butcher

The butcher sells raw meat. Cooked cuts stay off the shelf.

| Standing | Item | List price |
| --- | --- | --- |
| I | Stringy Meat | 2 |
| I | Game Meat | 3 |
| II | Pork | 5 |
| II | Mutton | 5 |
| II | Beef | 6 |
| II | Venison | 6 |
| II | Plump Bird | 6 |
| III | Prime Beef | 12 |
| III | Mature Venison | 12 |
| III | Big Game | 14 |

Anyone may sell here. The clerk pays 1 dollar for stringy meat or game, 2 for pork, mutton, beef, venison, or a plump bird, 4 for prime beef or mature venison, and 5 for big game. A rabbit pelt is 6 dollars. A deer hide is 12. The general store still pays 4 for a rabbit pelt and 8 for a deer hide.

### Trapper

The hunt kit uses the same prices as the gunsmith, so the two shelves agree.

| Standing | Item | List price |
| --- | --- | --- |
| I | Tin of Worms | 2 |
| I | Hunting Knife | 12 |
| II | Binoculars | 70 |
| II | Bow | 90 |

Anyone may sell here. A rabbit pelt is 8 dollars. A deer hide is 16. Legendary parts and sewn clothes wait.

### Bait and tackle

One shop, in Lagras.

| Standing | Item | List price |
| --- | --- | --- |
| I | Tin of Worms | 2 |
| I | Tin of Crickets | 2 |
| I | Herbivore Bait | 3 |
| I | Predator Bait | 4 |
| II | Fishing Rod | 25 |
| II | Potent Herbivore Bait | 8 |
| II | Potent Predator Bait | 10 |

Anyone may sell a fish here, for 6 dollars. The general store pays 4.

### Doctor

The general store keeps the bandage and the plain tonic. The stronger medicines are on this shelf. The doctor sells the bandage and the tonic as well.

| Standing | Item | List price |
| --- | --- | --- |
| I | Bandage | 3 |
| I | Box of Bandages | 8 |
| I | Tonic | 8 |
| II | Herbal Tonic | 12 |
| II | Herbal Medicine | 14 |
| II | Snake Oil | 15 |
| III | Burn Salve | 16 |
| III | Juniper Balm | 18 |
| III | Restorative | 22 |
| IV | Health Serum | 35 |
| IV | Antivenom | 40 |

Selling waits for Recognized. A bundle of mint is 4 dollars. A ginseng root is 6. The general store pays 2 for mint. Treatment stays off this shelf.

### Saloon

| Standing | Item | List price |
| --- | --- | --- |
| I | Beer | 2 |
| I | Beef Stew | 4 |
| II | Whiskey | 6 |
| II | Venison Stew | 6 |
| III | Bottle of Whiskey | 14 |
| III | Bottle of Brandy | 16 |

Keane's in Valentine charges 8 dollars for whiskey. The other saloons charge 6. The clerk does not buy drinks back.

### Barber

| Standing | Item | List price |
| --- | --- | --- |
| I | Hair Tonic | 3 |
| I | Pomade | 4 |
| II | Shaving Tonic | 5 |
| II | Beard Pomade | 6 |

The clerk does not buy these back. A haircut is not on the counter.
