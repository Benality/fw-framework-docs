# fw_stores

Runs in the player's game. Type `/shop` to open a sample store counter. Type `/shop` again, press Escape, or press Close, to leave it.

The counter is a sample. The dollars on Offer, and the piles on Want and Offer, change only on that screen. Closing it puts the sample back. Nothing is written to the bag you carry, and no real dollars move.

The clerk's goods are their own panel on the left, in the same journal style as the bag. Want and Offer are two panels between them and the bag. Want is what you are asking the clerk for. Offer is what you are putting up. None of those panels can be dragged. The bag on the right is the one you are carrying. Right-click, drag, split, use, give, and drop work the same way they do when you open the bag on its own. So do the pictures on the left of the bag, the clothes, the kit, and the gunbelt. The clerk keeps one pile of each thing. A pile with a number is a limited stock, and that number goes down when you buy it. A pile with no number is always in stock. Drag a pile into Want and choose how many. Bread from tier I can be up to 100 in one purchase, bandages 50, and revolver ammo 20. Those are as many as one pile in the satchel can hold. The rest of a limited pile stays on the counter. Drag a pile from your bag into Offer. A pile in an open pouch can go there too. The pile stays where it was. Drag a pile back out of Want to return it to the counter, or out of Offer to take it back. A counter pile will not sit in Offer, and a bag pile will not sit in Want.

Gold numbers are what you pay. A number with a plus is what the clerk pays you. The dollars on Offer are a sample. Settle spends those sample dollars. It does not put a purchase into your bag, and it does not take a sale out of it. Drag a pile from your bag onto Offer. A pile in an open pouch can go onto Offer without leaving the pouch. One knife is not sold for dollars. It asks for two bandages, and those bandages have to be in your bag when you settle. The bag still keeps them.

On the left of the clerk's goods are the same round pictures as the bag. All shows every pile. Kit shows weapons and ammo together. Each other picture shows that one kind, such as food, medical, or containers. A pile that does not match goes grey and stays where it is. Press that picture again, or press All, to show every kind. Want and Offer are not filtered. The bag has its own pictures, the same ones you use when the counter is closed.

Above the clerk's goods is a mark of four small squares, then I, II, III, and IV. I is Unknown, II is Recognized, III is Trusted, and IV is Established. The four squares show every pile, packed to the top of the counter. A numeral shows only that standing, packed the same way. You can press a numeral up through your own standing. A higher one stays grey. This sample stays Unknown, so I can be pressed, and II, III, and IV stay grey. A pile above your standing is locked on the full counter and cannot be bought. The medical bag is II.

Goods, Work, and Services switch only the clerk's panel. Want, Offer, and the satchel stay where they are. Work says the clerk has no errand on this visit. Services says they are not mending or treating anyone on this visit.

## Command

`/shop` is a client command. It takes no arguments. It asks the bag screen to open beside the counter. Start `fw_stores` after `fw_satchel` and `fw_inventory`, because the screen uses their pictures and the bag.

Another script on the player's game can open the same screen. `visit` is the clerk's goods, in the shape `/shop` builds. `live` is set for you, so the bag on the right is the one the player is carrying.

```lua
TriggerEvent('fw_satchel:shopToggle', visit)
```

Running it again closes the bag. Nothing is written to the bag.
