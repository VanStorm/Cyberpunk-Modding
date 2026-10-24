# cyberpunk2077-resource-paths.db

> **Note:** `cyberpunk2077-resource-paths.db` is the legacy 2.31 database, kept
> for compatibility with tools that already reference it. The latest database is
> distributed as `cyberpunk2077-resource-paths.7z` and contains 2.5 data with the
> updated schema described below.

A SQLite database mapping FNV1a64 file hashes to human-readable resource paths for
Cyberpunk 2077 (including Phantom Liberty / EP1).

All paths have been verified against the real game archive indexes. Paths confirmed
in the cooked archives are in the `paths` table. Paths that resolve correctly but
are not present in any archive (cut content, dev-only assets) are preserved in the
`unshipped` table.

## Stats

| Metric | Value |
|---|---|
| Game version | 2.5 |
| Verified paths | 552,514 |
| Unshipped paths | 206,278 |
| Unresolved hashes | 956 |
| Total archive files | 553,470 |
| Archives scanned | content + EP1 |

## Background

Cyberpunk 2077 `.archive` files index resources by 64-bit FNV1a64 hashes with no
embedded path strings. Base-game archives contain no custom data sections at all,
meaning the only way to recover human-readable paths is to extract them from the
game's own resource files.

This database was built through multiple passes, then verified against the game archives:

1. **CR2W import extraction** -- every file in every base-game and EP1 archive was
   decompressed via Oodle Kraken and its CR2W import table parsed to collect all
   referenced resource paths. This alone yielded ~556K paths (~80% coverage).

2. **External source ingestion** -- additional paths ingested from MlsetupBuilder
   `materialDB.json` and `tablemodels.json` (44K+ `.glb` model paths) and
   WolvenKit's `red.kark` dependency database.

3. **Deep body scan** -- full regex scan of decompressed CR2W property and export
   data (beyond the imports table) found additional paths embedded as string
   properties.

4. **WolvenKit JSON exports** -- a set of exported game files was scanned for
   `DepotPath.$value` references, adding further paths.

5. **Smart resolution** -- GPU-accelerated stem expansion and sibling inference:
   known path stems combined with known suffixes and extensions (texture variants,
   LOD levels) hashed in bulk and matched against the remaining unresolved set.

6. **Token recombination** -- GPU-accelerated per-directory token recombination:
   filename tokens extracted from resolved paths recombined (1-6 tokens) and
   matched against remaining unresolved hashes, including cross-directory token
   augmentation and numeric range expansion.

7. **Verification pass** -- every collected path was normalized (backslash
   collapse, lowercase) and its FNV1a64 hash recomputed. Paths whose hash exists
   in a real game archive index go into the `paths` table. Paths with a valid hash
   that are not in any archive go into the `unshipped` table (cut content,
   dev-only assets, or files renamed before shipping).

## Schema

### paths

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| path | TEXT | Human-readable resource path |

Also has index `idx_paths_path` on `path` for reverse lookups (path to hash).

### unshipped

Valid resource paths whose hash does not appear in any cooked game archive. These
represent cut content, development-only assets, or files that were renamed or
merged before shipping. They are real CDPR resource paths, but the files are not
present in the shipped game and cannot be extracted.

Also accessible via the `unshipped_paths` view.

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| path | TEXT | Resource path not present in any archive |

### unresolved

Hashes found in game archives that could not be resolved to a path string.

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| source_archive | TEXT | Archive file the hash was found in |

### phantom_hashes

Archive index entries with no stored data segments. These are declared in the
archive index but have no file content -- likely procedural resources, DLC stubs,
or runtime-generated entries. Empty for version 2.5.

| Column | Type | Description |
|---|---|---|
| hash | INTEGER (PK) | FNV1a64 hash, stored as signed int64 |
| source_archive | TEXT | Archive file the hash was found in |

### versions

| Column | Type | Description |
|---|---|---|
| game_version | TEXT (PK) | Game version string, e.g. `2.5` |
| flattened_at | TEXT | UTC timestamp when the verified DB was produced |
| paths_added | INTEGER | Paths written from this version |
| unshipped_added | INTEGER | Unshipped paths from this version |

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

# Reverse lookup (path to hash)
cursor.execute("SELECT hash FROM paths WHERE path = ?", ("base\\characters\\player\\player.ent",))
```

## Path coverage

All paths start with one of the known root prefixes:

| Prefix | Count | Content |
|---|---|---|
| `base\` | 474,346 | Base game |
| `ep1\` | 77,629 | Phantom Liberty |
| `engine\` | 367 | Engine resources |
| `dlc\` | 172 | DLC content |

## Top file types

| Extension | Count |
|---|---|
| `.wem` | 122,436 |
| `.mesh` | 105,377 |
| `.json` | 74,071 |
| `.xbm` | 56,397 |
| `.streamingsector` | 26,384 |
| `.gidata` | 23,097 |
| `.ent` | 22,002 |
| `.mlsetup` | 17,867 |
| `.anims` | 13,991 |
| `.mi` | 10,579 |

## Relation to other hash databases

WolvenKit ships `red.kark`, a KARK-compressed SQLite database with 1,717,506 file
hashes mapped to archive names. It contains no path strings. This database provides
the path names for 552,514 of those hashes -- essentially complete coverage of all
hashes WolvenKit tracks.

## License

Released under CC BY 4.0. See LICENSE for details.

**Attribution:** VanStorm (https://github.com/VanStorm/)

**Acknowledgements:**

- [WolvenKit](https://github.com/WolvenKit/) -- WolvenKit's JSON export
  pipeline was used to extract `DepotPath` references from exported game files.
  WolvenKit's `red.kark` dependency database provided authoritative archive-to-hash
  mappings used to identify unresolved hashes and their source archives.
- [MlsetupBuilder](https://github.com/Neurolinked/MlsetupBuilder) -- MlsetupBuilder's
  bundled `tablemodels.json` and `materialDB.json` data files contributed resource
  paths.

All path strings are derived from Cyberpunk 2077 game files. CD Projekt RED owns
all rights to the original game content. This database contains only file path
strings and their hashes, no game assets.
