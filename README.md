# DungeonQuestNPC

A quest giver at the entrance of every instance, a turn-in NPC at the exit, and a return
portal beside him. Talk to the giver and every quest that instance has to offer is on one
page; clear the place, hand everything in to the NPC at the end, and click the portal to be
back at the entrance.

* AzerothCore, WotLK 3.3.5a
* needs [Eluna](https://github.com/ElunaLuaEngine/Eluna) with `Eluna.UseUnsafeMethods = 1`
  (that is the default — the module reads quest data straight from the world database)
* 48 instances, 463 quests

Three files, kept together:

| File | What it is |
|---|---|
| `dungeon-quest-npc.lua` | the module: the start menu and the return portal |
| `dungeon-quest-data.lua` | the dungeons, their two NPCs, their entrances and their quests |
| `dungeon-quest.sql` | the NPCs, the spawns and the turn-ins, for your world database |

## How the two NPCs work

**The start NPC** draws his own menu. Every quest that belongs to the instance is on it,
whether you can take it or not, with the reason in brackets when you cannot. One row,
`Take every available quest (n)`, takes everything you can have.

**The end NPC** is an ordinary quest turn-in NPC. The core draws that window itself, from the
`creature_questender` rows in `dungeon-quest.sql`, so you get the normal reward choice — and
your reward by mail when your bags are full.

**The return portal** is a game object standing next to the end NPC. Right clicking it sends
you back to the entrance of the map you are standing in; dungeons with several wings
(Coilfang) each keep their own entrance.

Nothing is written to `creature_queststarter`. The start menu is drawn by the script, so the
core never reads those rows, and writing them would put a "!" over the NPC's head computed by
the core's own rules instead of the module's.

## Install

```bash
# 1. the world database
mysql -u root -p acore_world < dungeon-quest.sql

# 2. the scripts - copy the whole folder into your server's lua_scripts directory
cp -r dungeon-quest /path/to/azerothcore-server/lua_scripts/

# 3. in game, or on the server console
.reload eluna
```

You should see one line at startup:

```
[DungeonQuestNPC] loaded - 47 start NPCs, 47 return portals, 48 dungeons, 463 quests,
23 tagged quests (141 handlers registered; quest details are read from the world database
on first use).
```

The SQL is safe to run more than once. It deletes its own rows before writing them, so
re-importing after an edit is the normal way to change things.

## The start menu

```
Take every available quest (7)
Wailing Caverns
[17] Deviate Hides
[18] Serpentbloom
[18] Smart Drinks
[22] Leaders of the Fang
Outside the instance
[16] Hamuul Runetotem
[16] Nara Wildmane
[18] Trouble at the Docks
Unavailable
[21] Deviate Eradication  (lvl 15)
```

Three sections, in this order: what you can take here, what you can take but finish somewhere
else, and what you cannot take. The first is headed with **the instance's own name**, in the
language of the player's client, so the menu says where you are without spending a row on a
title. A section with nothing in it is left out, heading included.

Inside a section the quests are ordered by level, except in `Unavailable`, where they are
grouped **by the reason** first and by level second.

Titles are colour coded by how far above or below your level the quest sits, like the normal
quest list, and translated into the player's own language. Two colours are about availability
rather than level: `trivial` for a quest you have out-levelled but can still take, `locked`
for one you cannot take at all. All of them are set in `Config.COLOURS`.

Long menus are paged, 16 rows at a time (`PAGE_SIZE`). A section heading is never the last
row of a page — it moves down to the page its quests are on. Clicking a quest keeps you on
the page you were reading.

Every row carries an icon, and the split is what the rows *mean*: a gossip bubble for a row
that does something when you click it, a grey dot for a row that is only there to be read —
the headings and the quests you cannot take. Both are set in `Config.ICONS`.

## Why a quest is not offered

The reason in brackets comes from the same rules that decide whether you may take the quest,
so it says exactly what is missing. They are kept short on purpose, because they share a
gossip line with the title:

| Reason | Meaning |
|---|---|
| `lvl 50` | `MinLevel` |
| `max lvl 50` | `MaxLevel` |
| `Horde only`, `Alliance only`, `Mage only` | `AllowableRaces` / `AllowableClasses` — a faction mask is named after the faction |
| `rep too low`, `rep too high` | reputation |
| `needs skill` | `RequiredSkillID` |
| `needs <quest>` | the chain prerequisite, by title |
| `needs one: <quest>, <quest>` | one of an exclusive group must be done |
| `conflicts: <quest>` | another quest of the exclusive group is active |
| `follow-up: <quest>` | `NextQuestID` |
| `finish <quest> first` | a quest that leads into this one is in the log |
| `superseded: <quest>`, `breadcrumb: <quest>` | breadcrumb pair |
| `in log`, `ready`, `done`, `failed` | quest state |
| `errand`, `outside` | not finished inside the instance |
| `breadcrumb`, `breadcrumb target`, `chain`, `raid`, `group`, `heroic`, `escort`, `repeatable`, `event`, `attunement` | a quest kind switched off in `Config.OFFER` |

Clicking a quest you cannot take tells you the same thing in chat, with the quest as a
clickable link so you can read the core's own tooltip for it.

## Settings

Everything is at the top of `dungeon-quest-npc.lua`, in the `Config` table.

| Setting | Default | Meaning |
|---|---|---|
| `PAGE_SIZE` | `16` | Rows per page (5–24). |
| `ENFORCE` | see below | Which requirements keep a quest out of reach. |
| `SHOW_UNAVAILABLE` | `true` | List every quest, with the reason in brackets. `false` = only what you can take, and no `Unavailable` section. |
| `SHOW_TAKE_ALL` | `true` | The `Take every available quest (n)` row at the top. |
| `ICONS` | `0` / `10` | Icon per kind of row: `quest`, `command`, `page`, `locked`, `heading`, `empty`. |
| `DISABLE` | `{}` | Dungeons to skip, by the slug in `dungeon-quest-data.lua`. |
| `NPC_TEXT_START` | `92000` | `npc_text` row with the body text of the menu (created by `dungeon-quest.sql`). |
| `PORTAL_BASE` | `9400000` | Entry base of the return portals; must match `dungeon-quest.sql`. |
| `CREATURE_ID_COLUMN` | `"id"` | Entry column of `creature`: `"id1"` on current AzerothCore, `"id"` on older builds. |
| `OFFER` | see below | Quest kinds the giver offers. `false` = listed greyed out, with the kind as the reason. |
| `HIDE` | `errand` only | Quest kinds left out of the menu entirely. Wins over `OFFER`. |
| `COLOURS` | see the file | Menu text colours, six hex digits each. Chat messages keep the client's own bright hues. |
| `DEBUG` | `false` | Print the module's log lines to the server console. |

### `Config.ENFORCE` — one switch per requirement

```lua
ENFORCE = {
    minLevel       = true,      -- MinLevel
    maxLevel       = false,     -- MaxLevel
    race           = true,      -- AllowableRaces
    class          = true,      -- AllowableClasses
    reputation     = true,      -- required reputation, at or above
    skill          = true,      -- RequiredSkillID

    previousQuest  = false,     -- PrevQuestID
    nextChain      = false,     -- NextQuestID
    prevChain      = false,     -- a quest that leads into this one
    breadcrumb     = false,     -- breadcrumb pairs

    exclusiveGroup = true,      -- never two quests of the same group
    requiredGroup  = false,     -- one of the group must already be done
}
```

The point of the module is that you walk in and take the whole instance in one go, so the
chain switches are **off** by default: you are offered the end of a chain without having done
the steps before it. `previousQuest`, `nextChain`, `prevChain` and `breadcrumb` put those
rules back. The rest are the requirements the core itself enforces, so turning one off means
the giver offers a quest the server would refuse — leave them on unless you have a reason.

### `Config.OFFER` / `Config.HIDE` — which kinds of quest

Every kind the database can describe is detected on its own, so this list follows your own
content and needs no maintenance:

| Kind | Detected from | Default |
|---|---|---|
| `breadcrumb` | `BreadcrumbForQuestId` | offered |
| `breadcrumbTarget` | another quest names this one in `BreadcrumbForQuestId` | offered |
| `chain` | `PrevQuestID` or `NextQuestID` | offered |
| `repeatable` | `SpecialFlags & 1` | offered |
| `event` | `SpecialFlags & 2` | offered |
| `raid` | `QuestInfoID` 62, or the raid flag | offered |
| `group` | `QuestInfoID` 1 | offered |
| `heroic` | `QuestInfoID` 85 | offered |
| `escort` | `QuestInfoID` 84 | **greyed out** |
| `errand` | nothing to kill, collect or reach, and every turn-in NPC stands outside | **hidden** |
| `outside` | tagged in `dungeon-quest-data.lua` — not completed inside the instance | offered |
| `attunement` | tagged in `dungeon-quest-data.lua` — attunement and key chains | **greyed out** |

`HIDE` drops a kind from the menu altogether; `OFFER = false` lists it greyed out with the
kind as the reason. Escorts are greyed out because they need another player's timeline, and
errands are hidden because they are not part of clearing the instance — both are one line to
change.

The last two are the exception to "detected": nothing in `quest_template` says *this quest is
finished outside the instance* or *this one is an attunement*, so they are listed by quest id
in `DQN.TAGS`, at the top of `dungeon-quest-data.lua`:

```lua
DQN.TAGS = {
    [959]   = "outside",
    [4742]  = "attunement",
}
```

Add your own the same way. A quest tagged `outside` gets the second section; a quest whose
kind is switched off in `OFFER` is greyed out with the kind as its reason, wherever it is.

## Choosing which dungeons

The configuration block at the top of `dungeon-quest.sql` switches whole dungeons on and off:

```sql
SET @DQN_WAILING_CAVERNS := 1;
```

Set a line to `0` and re-import to remove that dungeon's two NPCs again — templates, spawns,
quest links and models. Set it back to `1` and re-import to get them back. Nothing else has
to change: `dungeon-quest-data.lua` can stay as it is, because a dungeon whose NPCs are not
in the database is simply never spoken to.

`Config.DISABLE` in the module does the same thing at runtime, by slug, for when you cannot
re-import:

```lua
DISABLE = {
    ["wailing_caverns"] = true,
}
```

## If something looks wrong

The module says what it did when it loaded, and stays quiet afterwards. `Config.DEBUG = true`
prints one line per decision — which quests the filters hid, which entries were skipped and
why.

| At startup you see | What it means |
|---|---|
| `nothing of ours is in the database` | `dungeon-quest.sql` has not been imported, or every dungeon in it is switched off. |
| `N of M NPC/portal entries are not in the database` | Normal for a dungeon switched off in the SQL. `DEBUG` lists them. |
| `no entrance on file for ...` | That instance has no entrance coordinates, so its portal has nowhere to send you; it still spawns. |
| `ERROR: could not read quest data` | `WorldDBQuery` is unavailable — set `Eluna.UseUnsafeMethods = 1`. |
| `ERROR: dungeon-quest-data.lua did not load` | The data file is not next to the module in `lua_scripts`. |

In game: a quest that will not go into the log is reported as such, with the usual causes —
a full quest log, or a quest that is started by an item rather than by an NPC.

## Credits

The dungeon and quest lists come from
[Tntdruid's mod-dungeon-quest-npc](https://github.com/Tntdruid/mod-dungeon-quest-npc).
