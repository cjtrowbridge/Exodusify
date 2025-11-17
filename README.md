# Exodusify

Tools and notebooks to help migrate away from Spotify onto a local-library player (like the Innioasis Y1) using:

- Spotify playlist exports (via Exportify)
- A tagged local library under `Downloaded/`
- A Jupyter notebook that builds:
	- Dated shopping lists of missing tracks
	- An orphaned-tracks list (tracks not in any playlist)
	- (Later) device-ready playlists for the Y1

The core logic lives in a Jupyter notebook you can run inside this repo.

## Repository layout

- `exportify-master/` – Third-party Exportify tool (JavaScript + HTML) used to export your Spotify playlists and liked songs as CSV.
- `spotify_playlists/` – Exportify output: one `.csv` per playlist, including `Liked_Songs.csv` and `Your_Top_Songs_YYYY.csv`.
- `Downloaded/` – Your local audio files, generally organized by artist or artist-combo folders.
- `shopping_lists/` – Created by the notebook; contains dated shopping-list CSVs.
- `README.md` – This file.
- `exodusify_workflow.ipynb` – The primary notebook described below.

Open `exodusify_workflow.ipynb` in the repo root to drive the workflow described below.

## 1. Environment and library setup

First, ensure you have Python 3 installed and available on your PATH.

Recommended Python libraries:

- `pandas` – CSV handling and data manipulation (already required by Exportify’s `requirements.txt`).
- `numpy` – Used by `pandas` and for numeric operations.
- `mutagen` – Reads audio metadata (ID3, FLAC, etc.) and duration from files in `Downloaded/`.
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

The Jupyter notebook in the repo root (for example, `exodusify_workflow.ipynb`) will orchestrate three main tasks:

1. Index and canonicalize your downloaded library.
2. Build dated shopping lists of tracks present in Spotify playlists but missing from your downloads.
3. Build an orphaned-tracks list of local tracks that are not referenced by any playlist.

### 3.1 Configuration & helpers

In the first notebook cells, you define paths and helper functions, e.g.:

- Paths:
	- `DOWNLOAD_ROOT = "Downloaded"`
	- `SPOTIFY_PLAYLISTS = "spotify_playlists"`
	- `SHOPPING_LIST_DIR = "shopping_lists"`
- Import core libraries (`os`, `pathlib`, `datetime`, `pandas`, `mutagen`, `unidecode`).
- Define string-normalization helpers to create **canonical keys** for matching:
	- Lowercase, strip punctuation and extra spaces.
	- Normalize accents (using `unidecode`).
	- Optionally strip decorations like `(feat. ...)`, `- remaster`, etc.

These helpers are used consistently for both Spotify CSV data and local audio files so that slightly different spellings still match.

### 3.2 Indexing `Downloaded/` (local library index)

The notebook walks the `Downloaded/` directory and builds an in-memory index of all audio files:

- For each audio file (e.g. `.mp3`, `.flac`, `.m4a`, etc.):
	- Use `mutagen` to read tags (`artist`, `title`, `album`) and duration.
	- Fall back to folder/filename heuristics when tags are missing.
	- Build canonical `artist_canonical` and `title_canonical` using the helpers.
- Store this in a `pandas.DataFrame` with columns such as:
	- `file_path` – Relative path from `Downloaded/`.
	- `artist_raw`, `title_raw`, `album_raw`.
	- `artist_canonical`, `title_canonical`.
	- `duration_ms`.

You can optionally persist this index to disk (e.g. `library_index.csv`) so you have an auditable snapshot and avoid rebuilding it from scratch every time, but for initial development the notebook can keep it in memory.

Rekordbox fits naturally *before* this step if you want to use it as a tag editor: you can point Rekordbox at `Downloaded/`, fix metadata there, then let the notebook read the cleaned tags via `mutagen`.

### 3.3 Loading Spotify playlists

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

### 3.4 Matching Spotify tracks to local files

Once both datasets are canonicalized, the notebook matches Spotify tracks against the local index:

- Join Spotify rows to the local index on:
	- `artist_canonical`
	- `title_canonical`
- Optionally use duration as a secondary check (e.g. accept only matches with `abs(spotify_duration_ms - local_duration_ms) <= 3000`).

After this step each Spotify row will either:

- Have a `file_path` (meaning the track already exists in `Downloaded/`), or
- Have `NaN` for `file_path` (meaning it is missing locally).

### 3.5 Generating dated shopping lists

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

### 3.6 Generating an orphaned-tracks list

The notebook can also compute an **orphaned list**: tracks that exist in `Downloaded/` but are not referenced by any Spotify playlist in `spotify_playlists/`.

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

### 3.7 Later: device-ready playlists (Innioasis Y1)

The notebook design also supports a longer-term step: generating playlist files that the Innioasis Y1 can read (typically `.m3u` or `.m3u8`):

- For each Spotify playlist CSV:
	- Walk its rows, using the same match results to get `file_path` for tracks that exist locally.
	- Compute a relative path suitable for the Y1 (e.g. relative to a `Music/` directory on your SD card).
	- Skip tracks that are still missing (they will remain only on the shopping list).
	- Write each playlist as a `.m3u` file with one path per line.

This step can live in later cells of the same notebook, once the shopping list and orphaned list workflow feels solid.

## Project roadmap

This section outlines planned phases for Exodusify. It is intentionally high-level so the notebook and scripts can evolve without constant README edits.

### Phase 1 – Notebook MVP (current)

- [x] Document environment setup and required Python libraries.
- [x] Define a Jupyter notebook workflow that:
	- Indexes `Downloaded/` with canonical artist/title keys.
	- Loads and merges `spotify_playlists/` CSVs.
	- Matches Spotify tracks to local files.
	- Generates timestamped shopping lists of missing tracks.
	- Generates timestamped orphaned-track lists.
- [x] Create the initial `exodusify_workflow.ipynb` notebook with the sections described above.

### Phase 2 – Persistence and performance

- [ ] Persist the local library index to disk as CSV (e.g. `library_index.csv`) so full rescans of `Downloaded/` are optional and auditable.
- [ ] Add incremental update logic (only rescan new/changed files under `Downloaded/`).
- [x] Add summary metrics cells in the notebook (counts of tracks, playlists, missing tracks, orphans).

### Phase 3 – Better matching and normalization

- [ ] Tighten normalization rules for canonical keys (handling "feat.", remasters, mixes, etc.).
- [ ] Optional: integrate MusicBrainz (`musicbrainzngs`) as a lookup source to normalize Artist/Title/Album where tags are messy.
- [ ] Optional: integrate AcoustID/Chromaprint for fingerprint-based identification of hard-to-tag files.
- [ ] Expose match confidence in the notebook (e.g. exact vs fuzzy match, duration-tolerant match).

### Phase 4 – Device-ready playlists for Innioasis Y1

- [ ] Decide and document the on-device directory layout (e.g. mirror `Downloaded/` under `Music/`).
- [ ] Implement notebook cells that:
	- For each Spotify playlist, build a list of resolved local `file_path`s.
	- Compute relative paths appropriate for the Y1.
	- Write `.m3u` (or `.m3u8`) playlist files, skipping tracks that are still missing.
- [ ] Add a short "Syncing to Y1" section to the README with copy commands and gotchas.

### Phase 5 – UX and tooling polish

- [ ] Add convenience functions/cells to quickly regenerate:
	- The latest shopping list.
	- The latest orphaned list.
	- All playlists for the Y1.
- [ ] Optional: factor out notebook logic into a small Python module/CLI so you can run common tasks without opening Jupyter.
- [ ] Optional: add simple visualizations in the notebook (e.g. most-referenced artists, playlist coverage of your library).