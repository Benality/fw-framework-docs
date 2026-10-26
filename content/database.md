# Database

Do not read or write these tables from your resource. Call the export. This page is which id to store on a table you own, and what the columns mean when an export gives you a copy.

The database name in `server.cfg` is `TerritoryFRP`. If it does not exist yet, oxmysql creates it and connects again. The name can only contain letters, numbers, and underscores. Replacing oxmysql with an untouched copy removes that auto-create. The framework still creates the tables below when each resource starts.

If a table already exists and is missing the column that resource expects, the resource logs the mismatch and does not change the table. `fw_items` is the exception: missing `weapon` and `item_group` columns are added.

Do not put money or jobs on these tables. A later resource adds its own table and stores a character id or an account id. See [A table of your own](#a-table-of-your-own).

## `fw_players`

Created by `fw_core` from `resources/[framework]/fw_core/sql/players.sql`. One row per Rockstar license. `fw_core` is the writer for connect-time columns — there is no Lua export yet for role, warnings, slots, or bans; change those directly in the table.

**Schema**

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

**Columns**

| Column | Lua field | Written when | Meaning |
| --- | --- | --- | --- |
| `id` | `id` | Insert | Account id. Store this on child tables. |
| `license` | `license` | Insert | `license:…` or `license2:…`, prefix included. Unique. Max 64 bytes. |
| `steam` | `steam` | Every successful admit | `steam:…` up to 40 bytes, or NULL. |
| `discord` | `discord` | Every successful admit | `discord:…` up to 40 bytes, or NULL. |
| `name` | `name` | Every successful admit | Display name, clipped to 64 characters. |
| `role` | `role` | Default `'user'` on insert | Account role. Not a job. Read into memory at connect. The creator's ped permission and admin commands use it. |
| `character_slots` | `characterSlots` | Default `5` on insert | Max characters for this account. `CreateCharacter` re-reads this column at create time. |
| `warnings` | `warnings` | Default `0` | Stored only. No enforcement. |
| `banned` | `banned` | Checked at connect | `1` refuses the connection. A connected player's in-memory copy shows `false`, because a banned license never attaches. |
| `banned_until` | `bannedUntil` | Checked at connect | Unix time. `0` with `banned = 1` means the ban does not expire. A time in the past clears `banned` on the next connect. |
| `created_at` | — | Insert | Not included on the Lua player record. |
| `last_seen` | — | Admit, and again on drop | Not included on the Lua player record. |

**Indexes**: primary key `id`, unique `license`. Lookups by steam or discord are not indexed — add one if a feature queries those columns.

**Relationships**: `ON DELETE CASCADE` from `fw_characters.player_id` — deleting an account deletes its characters.

**Calls**: [`fw_core`](fw-core.md). The fields you get back are the [account record](fw-core.md#account-record).

## `fw_characters`

Created by `fw_characters` from `resources/[framework]/fw_characters/sql/characters.sql`. One row per character. Do not `UPDATE` this table from another resource — use the exports.

**Schema**

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

**Columns**

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
| `clothing` | `clothing` | Worn clothes, outfits, and wearable state. NULL until `fw_clothing` saves it. The shape is on [fw_clothing](fw-clothing.md). |
| `barber` | `barber` | Hair, overlays, and presets. The shape is on [fw_creator](fw-creator.md). |
| `x`, `y`, `z` | `x`, `y`, `z` | Last position. Insert uses `defaultSpawn`; the creator then saves the chosen spawn. While the character is online, a newer sample lives in the `fw_characters` server cache and is what a reconnect reads until the cache is flushed into these columns. |
| `heading` | `heading` | Degrees. |
| `health` | `health` | 0–65535. Default 600. |
| `health_core` | `healthCore` | 0–100. Default 100. |
| `stamina_core` | `staminaCore` | 0–100. Default 100. |
| `is_dead` | `isDead` | `0` or `1`. Lua uses `true` / `false`. |
| `created_at` | `createdAt` | Insert time. |
| `last_played` | `lastPlayed` | Select, clear, or player drop. NULL until the first selection. |

**Indexes**: primary key `id`, unique `(player_id, slot)` — listing one account uses this index. The JSON columns are not indexed. `ROW_FORMAT=DYNAMIC` keeps a large document off the main row.

**Write paths**: `SaveState` updates position and health only — it never rewrites `appearance`, `clothing`, or `barber`. Each of those documents must encode to at most 262144 bytes. `GetCharacters` never selects the JSON columns; it selects `(appearance IS NOT NULL) AS has_appearance` (and the same for clothing/barber) and returns those as `hasAppearance`, `hasClothing`, `hasBarber`.

**Calls**: [`fw_characters`](fw-characters.md). The fields you get back are the [character record](fw-characters.md#character-record).

## A table of your own

A resource that stores something for a character should follow this shape:

- Its own table, created by that resource.
- A column `character_id` that stores `fw_characters.id`.
- A foreign key to `fw_characters(id)` with `ON DELETE CASCADE` when the data must disappear with the character.
- No copy of the character's name, gender, or appearance — read those through `exports.fw_characters`.

An account-level table uses `player_id` → `fw_players.id` the same way.

The character id to use is the one from `CreateCharacter`, `GetActiveCharacter`, or `Player(source).state.fwCharacterId`. The account id is `player.id` from `GetPlayer`, or `Player(source).state.fwPlayerId`.

## `fw_inventory`

Created by `fw_inventory` from `resources/[framework]/fw_inventory/sql/inventory.sql`. One row per character. The grid document and the item fields are in [fw_inventory](fw-inventory.md).

```sql
CREATE TABLE IF NOT EXISTS `fw_inventory` (
  `character_id` int unsigned NOT NULL,
  `grid` json NOT NULL,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`character_id`),
  CONSTRAINT `fk_fw_inventory_character`
    FOREIGN KEY (`character_id`) REFERENCES `fw_characters` (`id`)
    ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
```

| Column | Meaning |
| --- | --- |
| `character_id` | `fw_characters.id`. Primary key. Deleting the character deletes the row. |
| `grid` | JSON document: `width`, `height`, `baseWidth`, `baseHeight`, `bonus`, `nextId`, and `items`. At most 262144 bytes. `width` and `height` are the drawn size. |
| `updated_at` | Set by the database when the row is written. |

## `fw_items`

Created by `fw_inventory` from `resources/[framework]/fw_inventory/sql/items.sql`. One row per item type. This is not a satchel. Placements stay in `fw_inventory.grid`.

```sql
CREATE TABLE IF NOT EXISTS `fw_items` (
  `name` varchar(32) NOT NULL,
  `label` varchar(48) NOT NULL,
  `description` varchar(200) NOT NULL DEFAULT '',
  `width` tinyint unsigned NOT NULL,
  `height` tinyint unsigned NOT NULL,
  `stack` smallint unsigned NOT NULL DEFAULT 1,
  `weapon` tinyint(1) NOT NULL DEFAULT 0,
  `item_group` varchar(16) NOT NULL DEFAULT '',
  `rotate` tinyint(1) NOT NULL DEFAULT 0,
  `image` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

| Column | Meaning |
| --- | --- |
| `name` | Primary key. Lowercase letters, digits, and underscores. The name `AddItem` uses. |
| `label` | Satchel text. |
| `description` | Side panel. |
| `width`, `height` | Cell footprint before turning. |
| `stack` | Copies that share one footprint. `1` means each copy is placed on its own. |
| `weapon` | `1` when the item never stacks. A weapon is always one placement. |
| `item_group` | Filter and sort group. `ammo`, `animals`, `documents`, `food`, `herbs`, `horse`, `medical`, `tools`, `valuables`, `weapons`, or `materials`. |
| `rotate` | `1` when the player can turn it. |
| `image` | Removed on start when the column is still there. The picture is `fw_inventory/images/<name>.png`. |

Files in `items/` are written into this table on start and on `fw_items_reload`. A row inserted for an admin tool stays until it is deleted. The field list and the insert example are in [fw_inventory](fw-inventory.md). The `container` block on a pouch is not a column. It stays on the item file and is merged back onto the catalog when the files load.

## `fw_storage`

Created by `fw_inventory` from `resources/[framework]/fw_inventory/sql/storage.sql`. One row per chest, wagon, horse, or container item. The id is chosen by the caller. Opening and moving items is in [fw_inventory](fw-inventory.md).

```sql
CREATE TABLE IF NOT EXISTS `fw_storage` (
  `id` varchar(64) NOT NULL,
  `grid` json NOT NULL,
  `groups` json NULL,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=DYNAMIC;
```

| Column | Meaning |
| --- | --- |
| `id` | Caller id, such as `chest:valentine_1` or `item:` plus a container id. |
| `grid` | Same document shape as a satchel: `width`, `height`, `baseWidth`, `baseHeight`, and `items`. At most 262144 bytes. |
| `groups` | JSON list of item groups this row accepts. `NULL` accepts every group. |
| `updated_at` | Set by the database when the row is written. |
