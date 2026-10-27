# fw_guide

Runs in the player's game. This is the help screen. Type `/guide` to open it. Type `/guide` again, or press Escape, to close it.

The first subject is the large card on the left. The next ones fill the cards on the right. The line along the bottom describes the card you are on. Enter opens that subject. Escape steps back. On the front page, Escape closes help.

A subject opens as one of two screens.

A reading page is the Player screen. The title at the top is the card you opened. The writing is in up to three columns. Up and down scroll it. The corner says SCROLL and BACK.

A card row is the Roles screen. The pictures sit in a line, with a title and a short line under each one. Left and right move the red frame. Enter opens that card's reading page. Activities uses this screen.

Both are filled in from `client/pages.lua`. The first entry is the large card. `screen = 'article'` is the reading page. `screen = 'row'` is the card row.

A column is a list of blocks. A block is one of these:

- `heading` — the small capital line, such as Satchel.
- `body` — the paragraph under it.
- `rule` — a thin line between two paragraphs.
- `icons` — a row of circles. The names that draw are `heart`, `bolt`, `cross`, `bag`, and `swap`.
- `image` — a worn picture. `art` is the file name in `html/art`, without `.svg`. `caption` is the line under it.

A row card uses `title`, `blurb`, and `art`. Give it `columns` when Enter should open a reading page.

## Calls

Both run in the player's game.

### Open

```lua
--- Shows the help screen.
---@return nil
exports.fw_guide:Open()
```

```lua
exports.fw_guide:Open()
```

### Close

```lua
--- Hides the help screen.
---@return nil
exports.fw_guide:Close()
```

```lua
exports.fw_guide:Close()
```

`/guide` is a client command. It takes no arguments. A script should call `Open` or `Close` instead of running the command for the player.
