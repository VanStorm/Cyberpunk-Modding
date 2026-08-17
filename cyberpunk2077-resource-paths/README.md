# cyberpunk2077-resource-paths.db

A SQLite database mapping FNV1a64 file hashes to human-readable resource paths for
Cyberpunk 2077 version 2.31 (including Phantom Liberty / EP1).

## Stats

| Metric | Value |
|---|---|
| Game version | 2.31 |
| Resolved paths | 751,710 |
| Unresolved hashes | 162 |
| Coverage of game archives | 99.97% (544,508 of 544,670 files) |
| Archives scanned | 32 base-game + EP1 |

## Background

Cyberpunk 2077 `.archive` files index resources by 64-bit FNV1a64 hashes with no
embedded path strings. Base-game archives contain no custom data sections at all,
meaning the only way to recover human-readable paths is to extract them from the
game's own resource files.

This database was built through multiple passes:

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

All paths start with one of four known root prefixes:

| Prefix | Count | Content |
|---|---|---|
| `base\` | 659,119 | Base game |
| `ep1\` | 91,800 | Phantom Liberty |
| `engine\` | 388 | Engine resources |
| `dlc\` | 185 | DLC content |
| `test\` | 147 | CDPR dev test assets (shipped in archives) |
| `user\` | 50 | CDPR developer personal assets (shipped in archives) |

The `test\` and `user\` paths are real files present in the shipped game archives,
left over from development. They resolve correctly but are not meaningful for
modding purposes.

## Top file types

| Extension | Count |
|---|---|
| `.xbm` | 126,846 |
| `.wem` | 119,857 |
| `.mesh` | 107,512 |
| `.json` | 77,526 |
| `.glb` | 44,273 |
| `.anims` | 31,065 |
| `.mlmask` | 27,158 |
| `.streamingsector` | 26,355 |

## Relation to other hash databases

WolvenKit ships `red.kark`, a KARK-compressed SQLite database with 1,717,506 file
hashes mapped to archive names. It contains no path strings. This database
provides the path names for 544,412 of those hashes (the overlap between both
datasets), and resolves an additional 207,298 hashes not present in red.kark.

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
