# Exodusify

Tools and notebooks to help migrate away from Spotify onto a local-library player (like the Innioasis Y1) using:

- Spotify playlist exports (via Exportify)
- A tagged local library under `Music/`
- A Jupyter notebook that builds:
	- Dated shopping lists of missing tracks
	- An orphaned-tracks list (tracks not in any playlist)
	- Playlist coverage stats and unique-missing counts
	- Device-ready `.m3u8` playlists for the Innioasis Y1

The core logic lives in a Jupyter notebook you can run inside this repo.

## Repository layout

- `exportify-master/` – Third-party Exportify tool (JavaScript + HTML) used to export your Spotify playlists and liked songs as CSV.
- `spotify_playlists/` – Exportify output: one `.csv` per playlist, including `Liked_Songs.csv` and `Your_Top_Songs_YYYY.csv`.
- `Music/` – Your local audio files, generally organized by artist or artist-combo folders.
- `Add/` – Staging folder for fresh downloads. Playlist-specific dropboxes now live under `Add/To Playlist/<Playlist Name>`.
- `Playlists/` – Device-ready `.m3u8` files built by the export cell.
- `shopping_lists/` – Created by the notebook; contains dated shopping-list CSVs.
- `README.md` – This file.
- `exodusify_workflow.ipynb` – The primary notebook described below.

Open `exodusify_workflow.ipynb` in the repo root to drive the workflow described below.

## 1. Environment and library setup

First, ensure you have Python 3 installed and available on your PATH.

Recommended Python libraries:

- `pandas` – CSV handling and data manipulation (already required by Exportify’s `requirements.txt`).
- `numpy` – Used by `pandas` and for numeric operations.
- `mutagen` – Reads audio metadata (ID3, FLAC, etc.) and duration from files in `Music/`.
- `unidecode` – Normalizes text by removing accents (e.g. `Beyoncé` → `Beyonce`).

From the repo root, create/activate a virtual environment, then install dependencies:

```powershell
cd "c:\Users\CJ\Desktop\Exodusify"
python -m pip install --upgrade pip
python -m pip install pandas numpy mutagen unidecode
```

## 2. Exporting playlists from Spotify

You only need to do this once on your way out of Spotify. Export your playlists on the Exportify website.

## 3. Running the Exodusify notebook

The Jupyter notebook in the repo root (for example, `exodusify_workflow.ipynb`) orchestrates the full pipeline:

1. Pull any fresh MP3s from `Add/`, infer tags, and place them into the canonical `Music/Artist/Album/Title` tree while guarding against duplicates.
2. Index and canonicalize everything under `Music/`.
3. Build dated shopping lists of tracks present in Spotify playlists but missing from your `Music/` library.
4. Build an orphaned-tracks list of local tracks that are not referenced by any playlist.
5. Generate device-ready playlists for the Innioasis Y1.

### 3.1 Configuration & helpers

In the first notebook cells, you define paths and helper functions, e.g.:

- Paths:
	- `MUSIC_ROOT = "Music"`
	- `SPOTIFY_PLAYLISTS = "spotify_playlists"`
	- `SHOPPING_LIST_DIR = "shopping_lists"`
	- `ADD_ROOT = "Add"`
- Import core libraries (`os`, `pathlib`, `datetime`, `pandas`, `mutagen`, `unidecode`).
- Define string-normalization helpers to create **canonical keys** for matching:
	- Lowercase, strip punctuation and extra spaces.
	- Normalize accents (using `unidecode`).
	- Optionally strip decorations like `(feat. ...)`, `- remaster`, etc.

These helpers are used consistently for both Spotify CSV data and local audio files so that slightly different spellings still match.

### 3.2 Processing new additions in `Add/`

Before rescanning the library, drop any freshly downloaded MP3s into the `Add/` folder. Playlist-specific dropboxes live inside `Add/To Playlist/<Playlist Name>` (the notebook auto-creates these to mirror your Exportify CSVs), but you can also drop loose files directly under `Add/`. Cell 3 in the notebook:

- Recursively finds `.mp3` files under `Add/` (future-friendly constant `SUPPORTED_IMPORT_SUFFIXES` gates supported formats).
- Reads tags with `mutagen` to derive artist/album/title; filename fallbacks keep the flow moving when tags are sparse.
- Tracks which playlist staging folder a file came from (based on the `Add/To Playlist/<Playlist Name>` prefix) so it can report how many tracks were added for each playlist snapshot.
- Normalizes those fields with `safe_path_component`/`canonicalize_string` and constructs the destination `Music/Artist/Album/Title.mp3` path.
- Checks for two collision cases before moving the file:
	- **Same canonical title already in the destination album folder.** This catches situations where old rips had prefixes like track numbers. The notebook logs `skipped_duplicate_title`, prints the existing file path, and leaves the staged file untouched so you can delete or retag it manually before trying again.
	- **Same filename already present.** Logged as `skipped_exists`.
- Moves non-conflicting files into `Music/` and prunes any now-empty directories left in `Add/`.

Re-run this cell whenever you place new downloads into `Add/`. If it reports duplicates, clean up the offending files in `Music/` (or rename the staged file) before rerunning.

### 3.3 Indexing `Music/` (local library index)

The notebook walks the `Music/` directory and builds an in-memory index of all audio files:

- For each audio file (e.g. `.mp3`, `.flac`, `.m4a`, etc.):
	- Use `mutagen` to read tags (`artist`, `title`, `album`) and duration.
	- Fall back to folder/filename heuristics when tags are missing.
	- Build canonical `artist_canonical` and `title_canonical` using the helpers.
- Store this in a `pandas.DataFrame` with columns such as:
	- `file_path` – Relative path from `Music/`.
	- `artist_raw`, `title_raw`, `album_raw`.
	- `artist_canonical`, `title_canonical`.
	- `duration_ms`.

You can optionally persist this index to disk (e.g. `library_index.csv`) so you have an auditable snapshot and avoid rebuilding it from scratch every time, but for initial development the notebook can keep it in memory.

Rekordbox fits naturally *before* this step if you want to use it as a tag editor: you can point Rekordbox at `Music/`, fix metadata there, then let the notebook read the cleaned tags via `mutagen`.

### 3.4 Loading Spotify playlists

The notebook then loads all CSV files from `spotify_playlists/`:

- Reads each `.csv` with `pandas.read_csv`.
- Adds a `playlist_name` column inferred from the filename.
- Concatenates them into a single `DataFrame` with columns like:
	- `Track Name`
	- `Artist Name(s)`
	- `Album Name`
	- `Duration (ms)`
	- `playlist_name`
- Adds helpful flags:
	- `is_liked` for rows from `Liked_Songs.csv`.
	- `is_top_songs` for rows from any `Your_Top_Songs_YYYY.csv` file.

The notebook then builds `artist_canonical` and `title_canonical` columns for Spotify tracks using the same normalization helpers as for the local library index. Typically you use the first artist listed in `Artist Name(s)` as the primary artist for matching, while keeping the full string for display.

### 3.5 Matching Spotify tracks to local files

Once both datasets are canonicalized, the notebook matches Spotify tracks against the local index:

- Join Spotify rows to the local index on:
	- `artist_canonical`
	- `title_canonical`
- Optionally use duration as a secondary check (e.g. accept only matches with `abs(spotify_duration_ms - local_duration_ms) <= 3000`).

After this step each Spotify row will either:

- Have a `file_path` (meaning the track already exists in `Music/`), or
- Have `NaN` for `file_path` (meaning it is missing locally).

### 3.6 Generating dated shopping lists

To build a shopping list of tracks that you still need to acquire:

1. Filter the matched Spotify DataFrame to rows where `file_path` is missing.
2. Group by `(artist_canonical, title_canonical)` to deduplicate across playlists.
3. For each unique missing track, aggregate:
	 - `Playlists_Count` – how many playlists the track appears in.
	 - `Playlists` – the set or comma-separated list of playlist names.
	 - `Is_Liked` – whether any occurrence comes from `Liked_Songs.csv`.
	 - `Is_Top_Songs` – whether any occurrence comes from a `Your_Top_Songs_YYYY.csv` file.
4. Create an output DataFrame with user-friendly columns, for example:
	 - `Artist`
	 - `Title`
	 - `Album`
	 - `Duration_ms`
	 - `Playlists_Count`
	 - `Playlists`
	 - `Is_Liked`
	 - `Is_Top_Songs`
5. Compute a timestamp string inside the notebook:

	 ```python
	 from datetime import datetime
	 stamp = datetime.now().strftime("%Y-%m-%d-%H-%M-%S")
	 ```

6. Ensure `shopping_lists/` exists, then save the shopping list as:

	 - `shopping_lists/shopping_list_{stamp}.csv`

7. Sort the DataFrame before saving (e.g. by `Playlists_Count` descending, then `Is_Liked`, then `Artist`, `Title`) so each CSV is easy to skim.

Because the filenames are timestamped in `YYYY-MM-DD-HH-MM-SS` format, they are naturally sortable; the lexicographically last file is your most recent shopping list.

### 3.7 Generating an orphaned-tracks list

The notebook can also compute an **orphaned list**: tracks that exist in `Music/` but are not referenced by any Spotify playlist in `spotify_playlists/`.

Steps:

1. From the matched Spotify DataFrame, collect the set of canonical keys `(artist_canonical, title_canonical)` that appear in any playlist.
2. From the local library index, take all canonical keys for downloaded tracks.
3. Compute the set difference:
	 - `orphan_keys = local_keys - playlist_keys`
4. Filter the local library index to rows whose canonical key is in `orphan_keys`.
5. Save this as a separate CSV, for example:
	 - `shopping_lists/orphaned_tracks_{stamp}.csv`

This file represents tracks you own that don’t belong to any current playlist snapshot. You can use it to:

- Discover music you may want to add to playlists.
- Decide whether to keep or archive rarely used tracks.

### 3.8 Reviewing playlist coverage

Cell 8 in the notebook summarizes overall progress so you can prioritize work without digging into raw DataFrames:

- Per-playlist totals of tracks, matched files, missing files, and percent complete (sorted by completion).
- Flags showing whether each playlist snapshot came from `Liked_Songs.csv` or a `Your_Top_Songs_YYYY.csv` export.
- Global aggregates plus the number of **unique missing tracks** across every playlist, deduplicated on canonical artist/title keys.

Re-run the earlier cells to refresh `matched_df`, then execute Cell 8 whenever you want an updated dashboard view.

### 3.9 Exporting Innioasis Y1 playlists

Cell 9 now writes device-ready `.m3u8` playlists under `Playlists/` so you can copy them straight to the Innioasis SD card:

1. Ensure Cells 1–7 (and the playlist stats if desired) have been re-run so `matched_df` reflects the latest library + Spotify snapshots.
2. Ensure the export cell points at the correct on-device base folder (default `../Music/`).
3. Run Cell 9. For each playlist Exportify produced:
	- Tracks with a resolved `file_path` are sorted by their Spotify `Position` (if present) and written as EXTINF entries.
	- Missing tracks are skipped, but the summary table shows how many were omitted per playlist.
	- Paths inside each `.m3u8` are relative (e.g. `../Music/Artist/Album/Track.mp3`) so the playlists continue to work when copied next to the `Music/` directory on the SD card.

Check the printed summary for the count of playlists exported and their destination filenames, then copy both the audio folders and matching `.m3u8` files to the Y1.

## Project roadmap

This section outlines planned phases for Exodusify. It is intentionally high-level so the notebook and scripts can evolve without constant README edits.

### Phase 1 – Notebook MVP (current)

- [x] Document environment setup and required Python libraries.
- [x] Define a Jupyter notebook workflow that:
	- Indexes `Music/` with canonical artist/title keys.
	- Loads and merges `spotify_playlists/` CSVs.
	- Matches Spotify tracks to local files.
	- Generates timestamped shopping lists of missing tracks.
	- Generates timestamped orphaned-track lists.
- [x] Create the initial `exodusify_workflow.ipynb` notebook with the sections described above.

### Phase 2 – Persistence and performance

- [ ] Persist the local library index to disk as CSV (e.g. `library_index.csv`) so full rescans of `Music/` are optional and auditable.
- [ ] Add incremental update logic (only rescan new/changed files under `Music/`).
- [x] Add summary metrics cells in the notebook (counts of tracks, playlists, missing tracks, orphans).

### Phase 3 – Better matching and normalization

- [ ] Tighten normalization rules for canonical keys (handling "feat.", remasters, mixes, etc.).
- [ ] Optional: integrate MusicBrainz (`musicbrainzngs`) as a lookup source to normalize Artist/Title/Album where tags are messy.
- [ ] Optional: integrate AcoustID/Chromaprint for fingerprint-based identification of hard-to-tag files.
- [ ] Expose match confidence in the notebook (e.g. exact vs fuzzy match, duration-tolerant match).

### Phase 4 – Device-ready playlists for Innioasis Y1

- [ ] Decide and document the on-device directory layout (e.g. mirror `Music/` under the device's `/Music`).
- [x] Implement notebook cells that:
	- For each Spotify playlist, build a list of resolved local `file_path`s.
	- Compute relative paths appropriate for the Y1.
	- Write `.m3u` (or `.m3u8`) playlist files, skipping tracks that are still missing.
- [x] Add a short "Syncing to Y1" section to the README with copy commands and gotchas.

### Phase 5 – UX and tooling polish

- [ ] Add convenience functions/cells to quickly regenerate:
	- The latest shopping list.
	- The latest orphaned list.
	- All playlists for the Y1.
- [ ] Optional: factor out notebook logic into a small Python module/CLI so you can run common tasks without opening Jupyter.
- [ ] Optional: add simple visualizations in the notebook (e.g. most-referenced artists, playlist coverage of your library).