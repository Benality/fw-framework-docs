# Database

Server database name: `TerritoryFRP`. The connection string is the `mysql_connection_string` convar in `server.cfg`. Use `utf8mb4`.

If that database does not exist, oxmysql creates it with `utf8mb4` / `utf8mb4_unicode_ci` and connects again. The name must match `^[A-Za-z0-9_]+$`. That behavior is a local change in `resources/[standalone]/oxmysql/dist/build.js`. Replacing oxmysql with an upstream build removes the auto-create. The tables below are still created by the framework resources on startup.

Windows error text may show the database name in lowercase (`territoryfrp`). The configured name is `TerritoryFRP`.

`fw_core` and `fw_characters` run their SQL with `CREATE TABLE IF NOT EXISTS`. They then check one expected column (`license` on `fw_players`, `clothing` on `fw_characters`). If a table with that name already exists and the column is missing, the resource logs the mismatch and does not alter the table.

Do not put inventory, money, or jobs on these tables. A later resource should add its own table and store `fw_characters.id` or `fw_players.id`.

## fw_players

Created by `fw_core` from `resources/[framework]/fw_core/sql/players.sql`. One row per Rockstar license. `fw_core` is the writer for connect-time columns. There is no Lua export yet for role, warnings, slots, or bans. Change those in the table.

```sql
CREATE TABLE IF NOT EXISTS `fw_players` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `license` varchar(64) NOT NULL,
  `steam` varchar(40) DEFAULT NULL,
  `discord` varchar(40) DEFAULT NULL,
  `name` varchar(64) NOT NULL DEFAULT '',
  `role` varchar(16) NOT NULL DEFAULT 'user',
  `character_slots` tinyint unsigned NOT NULL DEFAULT 5,
  `warnings` smallint unsigned NOT NULL DEFAULT 0,
  `banned` tinyint(1) NOT NULL DEFAULT 0,
  `banned_until` int unsigned NOT NULL DEFAULT 0,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_seen` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_fw_players_license` (`license`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| Column | Lua field | Written when | Meaning |
| --- | --- | --- | --- |
| `id` | `id` | Insert | Account id. Store this on child tables. |
| `license` | `license` | Insert | `license:…` or `license2:…`, prefix included. Unique. Max 64 bytes. |
| `steam` | `steam` | Every successful admit | `steam:…` up to 40 bytes, or NULL. |
| `discord` | `discord` | Every successful admit | `discord:…` up to 40 bytes, or NULL. |
| `name` | `name` | Every successful admit | Display name, clipped to 64 characters. |
| `role` | `role` | Default `user` on insert | Account role. Not a job. Read into memory at connect. The creator's ped permission and admin commands use it. |
| `character_slots` | `characterSlots` | Default `5` on insert | Max characters for this account. `CreateCharacter` reads this column again at create time. |
| `warnings` | `warnings` | Default `0` | Stored only. No enforcement. |
| `banned` | `banned` | Checked at connect | `1` refuses the connection. A connected player's in-memory copy shows `false`, because a banned license never attaches. |
| `banned_until` | `bannedUntil` | Checked at connect | Unix time. `0` with `banned = 1` means the ban does not expire. A time in the past clears `banned` on the next connect. |
| `created_at` | — | Insert | Not included on the Lua player record. |
| `last_seen` | — | Admit, and again on drop | Not included on the Lua player record. |

Indexes: primary key `id`, unique `license`. Lookups by steam or discord are not indexed. Add an index when a feature queries those columns.

`ON DELETE CASCADE` from `fw_characters.player_id` means deleting an account deletes its characters.

## fw_characters

Created by `fw_characters` from `resources/[framework]/fw_characters/sql/characters.sql`. One row per character. The Lua API is in [fw_characters](fw-characters.md). Do not `UPDATE` this table from another resource.

```sql
CREATE TABLE IF NOT EXISTS `fw_characters` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `player_id` int unsigned NOT NULL,
  `slot` tinyint unsigned NOT NULL,
  `first_name` varchar(50) NOT NULL,
  `last_name` varchar(50) NOT NULL,
  `nickname` varchar(50) DEFAULT NULL,
  `gender` enum('male','female') NOT NULL,
  `birth_date` date DEFAULT NULL,
  `description` text NOT NULL,
  `appearance` json DEFAULT NULL,
  `clothing` json DEFAULT NULL,
  `barber` json DEFAULT NULL,
  `x` float NOT NULL,
  `y` float NOT NULL,
  `z` float NOT NULL,
  `heading` float NOT NULL,
  `health` smallint unsigned NOT NULL DEFAULT 600,
  `health_core` tinyint unsigned NOT NULL DEFAULT 100,
  `stamina_core` tinyint unsigned NOT NULL DEFAULT 100,
  `is_dead` tinyint(1) NOT NULL DEFAULT 0,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_played` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_fw_characters_player_slot` (`player_id`, `slot`),
  CONSTRAINT `fk_fw_characters_player`
    FOREIGN KEY (`player_id`) REFERENCES `fw_players` (`id`)
    ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
```

| Column | Lua field | Meaning |
| --- | --- | --- |
| `id` | `id` | Character id. The id inventory, money, and jobs should store later. |
| `player_id` | `playerId` | Owner. `fw_players.id`. |
| `slot` | `slot` | 1 through that account's `character_slots`. Unique per account. Deleting a character frees the slot. Omitting `slot` on create uses the lowest free one. |
| `first_name` | `firstName` | 1–50 characters. |
| `last_name` | `lastName` | 1–50 characters. |
| `nickname` | `nickname` | NULL when unset. |
| `gender` | `gender` | `male` or `female`. |
| `birth_date` | `birthDate` | `YYYY-MM-DD` or NULL. Year must sit in `FwCharactersConfig` (1800–1899). |
| `description` | `description` | Lore, up to 2000 characters. Always sent on insert, including `''`. |
| `appearance` | `appearance` | Body and face JSON. NULL until `SetAppearance`. The creator stores the menu ped-data table here. |
| `clothing` | `clothing` | Worn clothes JSON. NULL today. The creator does not write it. |
| `barber` | `barber` | Hair, overlays, and presets. Shape is in [fw_creator](fw-creator.md). |
| `x`, `y`, `z` | `x`, `y`, `z` | Last position. Insert uses `defaultSpawn`. The creator then saves the chosen spawn. |
| `heading` | `heading` | Degrees. |
| `health` | `health` | 0–65535. Default 600. |
| `health_core` | `healthCore` | 0–100. Default 100. |
| `stamina_core` | `staminaCore` | 0–100. Default 100. |
| `is_dead` | `isDead` | `0` or `1`. Lua uses `true` / `false`. |
| `created_at` | `createdAt` | Insert time. |
| `last_played` | `lastPlayed` | Select, clear, or player drop. NULL until the first selection. |

The only secondary index is unique `(player_id, slot)`. Listing one account uses that index. The JSON columns are not indexed. `ROW_FORMAT=DYNAMIC` keeps a large document off the main row.

`SaveState` updates position and health only. It does not rewrite `appearance`, `clothing`, or `barber`. Each of those documents must encode to at most 262144 bytes.

`GetCharacters` does not select the JSON. It selects `(appearance IS NOT NULL) AS has_appearance` and the same for clothing and barber, and returns those as `hasAppearance`, `hasClothing`, and `hasBarber`.

## What the next table should look like

A gameplay resource that hangs data on a character should follow this shape:

- Its own table, created by that resource.
- A column `character_id` that stores `fw_characters.id`.
- A foreign key to `fw_characters(id)` with `ON DELETE CASCADE` when the data must disappear with the character.
- No copy of the character's name, gender, or appearance. Read those through `exports.fw_characters`.

An account-level table uses `player_id` → `fw_players.id` the same way.

The character id to use is the one from `CreateCharacter`, `GetActiveCharacter`, or `Player(source).state.fwCharacterId`. The account id is `player.id` from `GetPlayer`, or `Player(source).state.fwPlayerId`.
