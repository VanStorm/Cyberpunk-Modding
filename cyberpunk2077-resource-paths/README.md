# cyberpunk2077-resource-paths.db

A SQLite database mapping FNV1a64 file hashes to human-readable resource paths for
Cyberpunk 2077 version 2.31 (including Phantom Liberty / EP1).

All paths have been verified against the real game archive indexes: only hashes
that exist as actual cooked files in the game archives are included.

## Stats

| Metric | Value |
|---|---|
| Game version | 2.31 |
| Verified paths | 544,540 |
| Unresolved hashes | 162 |
| Total archive files | 544,670 |
| Archives scanned | 32 base-game + EP1 |

## Background

Cyberpunk 2077 `.archive` files index resources by 64-bit FNV1a64 hashes with no
embedded path strings. Base-game archives contain no custom data sections at all,
meaning the only way to recover human-readable paths is to extract them from the
game's own resource files.

This database was built through multiple passes, then verified against the game archives:

1. **CR2W import extraction** -- every file in every base-game and EP1 archive was
   decompressed via Oodle Kraken and its CR2W import table parsed to collect all
   referenced resource paths. This alone yielded ~544K paths (~86% coverage).

2. **External source ingestion** -- additional paths ingested from REDmod tweak
   files, game scripts, and MlsetupBuilder `materialDB.json` and `tablemodels.json`
   (44K+ `.glb` model paths).

3. **Deep body scan** -- full regex scan of decompressed CR2W property/export data
   (beyond the imports table) found 1,685 additional paths embedded as string
   properties.

4. **WolvenKit JSON exports** -- 38K+ exported `.app`, `.ent`, `.scene`, and
   `.quest` files were scanned recursively for `DepotPath.$value` references,
   adding 26K+ further paths.

5. **Smart resolution** -- GPU-accelerated stem expansion: known path stems
   combined with known suffixes and extensions (e.g. `_d`, `_n`, `_r`, `_e`
   texture variants) were hashed in bulk and matched against the remaining
   unresolved set, adding ~25K texture and LOD variant paths.

6. **Verification pass** -- every collected path was normalized (backslash
   collapse, lowercase) and its FNV1a64 hash recomputed. Only paths whose hash
   exists in a real game archive index were retained. This step discarded ~207K
   paths from external sources that referenced files not present in the cooked
   game data, and silently corrected 32 paths with malformed separators.

## Schema

### paths (view)

Backward-compatible view always pointing to the active version's table. Use this
for queries unless you need to target a specific game version explicitly.

### paths_2310

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| path | TEXT | Human-readable resource path |

### versions

| Column | Type | Description |
|---|---|---|
| game_version | TEXT (PK) | Game version string, e.g. `2.31` |
| paths_table | TEXT | Name of the corresponding paths table |
| created | TEXT | UTC timestamp when the entry was created |

### unresolved

Hashes found in game archives that could not be resolved to a path.

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| source_archive | TEXT | Archive file the hash was found in |

## Hash algorithm

FNV-1a 64-bit, applied to the resource path encoded as UTF-8 bytes (no null
terminator). Paths use backslashes as separators, exactly as they appear in the
game's CR2W import tables.

```python
FNV_OFFSET = 0xCBF29CE484222325
FNV_PRIME  = 0x00000100000001B3

def fnv1a64(path: str) -> int:
    h = FNV_OFFSET
    for byte in path.encode("utf-8"):
        h ^= byte
        h = (h * FNV_PRIME) & 0xFFFFFFFFFFFFFFFF
    return h
```

Hashes are stored as signed int64 (SQLite has no unsigned 64-bit type). Convert
before querying:

```python
def to_signed64(value: int) -> int:
    if value >= 0x8000000000000000:
        return value - 0x10000000000000000
    return value

# Lookup example
hash_val = to_signed64(fnv1a64("base\\characters\\player\\player.ent"))
cursor.execute("SELECT path FROM paths WHERE hash = ?", (hash_val,))
```

## Path coverage

All paths start with one of the known root prefixes:

| Prefix | Count | Content |
|---|---|---|
| `base\` | 466,410 | Base game |
| `ep1\` | 77,558 | Phantom Liberty |
| `engine\` | 368 | Engine resources |
| `dlc\` | 172 | DLC content |

The `test\` and `user\` prefixes present in the raw database (CDPR developer
assets shipped in the game archives) were discarded by the verification pass, as
their hashes do not exist in any cooked archive index.

## Top file types

| Extension | Count |
|---|---|
| `.wem` | 119,857 |
| `.mesh` | 102,660 |
| `.json` | 73,292 |
| `.xbm` | 55,818 |
| `.streamingsector` | 26,355 |
| `.gidata` | 23,096 |
| `.ent` | 21,566 |
| `.mlsetup` | 17,734 |

## Relation to other hash databases

WolvenKit ships `red.kark`, a KARK-compressed SQLite database with 1,717,506 file
hashes mapped to archive names. It contains no path strings. This database provides
the path names for 544,540 of those hashes -- essentially complete coverage of all
hashes WolvenKit tracks for version 2.31.

The conflict checker project (red4lib) ships a `metadata-resources.csv` with
approximately 1.7M entries built from multiple game versions. Compared to it, this
database covers 11K more files specific to version 2.31.

## License

Released under CC BY 4.0. See LICENSE for details.

**Attribution:** Ultrapunk (https://github.com/Ultrapunk)

**Acknowledgements:**

- [WolvenKit](https://github.com/WolvenTeam/WolvenKit) -- WolvenKit's JSON export
  pipeline was used to extract `DepotPath` references from exported game files,
  contributing ~26K resolved paths. WolvenKit's `red.kark` dependency database
  provided authoritative archive-to-hash mappings used to identify unresolved
  hashes and their source archives.
- [MlsetupBuilder](https://github.com/Neurolinked/MlsetupBuilder) -- MlsetupBuilder's
  bundled `tablemodels.json` and `materialDB.json` data files contributed ~45K
  resource paths.

All path strings are derived from Cyberpunk 2077 game files. CD Projekt RED owns
all rights to the original game content. This database contains only file path
strings and their hashes, no game assets.
