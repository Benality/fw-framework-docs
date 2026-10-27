# vorp_lib

Runs in the player's game. This page is the loading prompt, the revolver cylinder at the bottom right. The game has one of these. Starting it again only changes the words.

`vorp_lib` has to be started before the script that calls it.

## SpinnerStart

```lua
--- Shows the loading prompt. A string is the words. A table may also set how long it stays, in milliseconds.
--- No duration means it stays until SpinnerStop. Nothing comes back.
---@param data string|table|nil
---@return nil
exports.vorp_lib:SpinnerStart(data)
```

```lua
exports.vorp_lib:SpinnerStart('Opening')
```

```lua
exports.vorp_lib:SpinnerStart({ text = 'Opening', duration = 2000 })
```

`text` is the line next to the cylinder. `duration` is how many milliseconds until it goes away on its own. Leave `duration` out when your own code decides when it is finished, then call `SpinnerStop`.

## SpinnerStop

```lua
--- Hides the loading prompt.
---@return nil
exports.vorp_lib:SpinnerStop()
```

```lua
exports.vorp_lib:SpinnerStop()
```

Call this when the wait is over, including when the player cancels it. Stopping when nothing is showing is safe.
