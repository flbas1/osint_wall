# TODO

Chris writes the code; Claude keeps this file and [README.md](README.md) up to date.
The steps are a suggested order, so reorder them freely.

## Build steps

### 1. Minimal tray app
- [ ] `uv run earthwall2` starts a PySide6 app showing a tray icon.
- [ ] The icon is a dot drawn with Pillow (`get_tray_icon`), converted to a `QIcon`
      through `ImageQt`.
- [ ] The right-click menu has **Quit** (plus a "Show Config" placeholder).
- [ ] Logging writes to a file (needed once it runs under `pythonw`).

### 2. Status dot
- [ ] `TrayApp` class: three icons built once, swapped by `set_status(color)`.
- [ ] Temporary test menu items (Green / Red / Yellow). Remove them once real status
      changes exist.
- [ ] Add a tooltip.
- [ ] Add a custom `logging.Handler` that turns the dot red on `WARNING`/`ERROR`.

### 3. Config
- [ ] Choose the YAML library: **ruamel.yaml**.
- [ ] `uv add ruamel.yaml`.
- [ ] New `config.py`: `DEFAULTS`, `config_path()`, `load_config()`, `save_config(cfg)`.
- [ ] Find the right folder through `QStandardPaths`, with `mkdir`. (Written; set the app
      name in `run()` before first use.)
- [ ] Missing file: write the defaults. (Written, not yet run end-to-end.)
- [ ] `merge(defaults, file)`, where the defaults define the shape. All six test cases pass
      (2026-11-06).
- [ ] Broken file: print "bad yaml" (path plus line/column) and exit non-zero. Never
      overwrite the file.
- [ ] `run()` loads the config and passes it to `TrayApp`.

### 4. Projections
- [ ] Write the projection base class `Projection` (`ABCMeta`): shared `__init__(center_lon)`
      and `wrap_lon()`, with abstract `forward`, `inverse` and `aspect`
- [ ] Implement `Equirectangular(center_lon)`: normalised 0–1 coordinates, aspect 2.
- [ ] Make both methods work on numpy arrays. All 12 checks pass (2026-11-06), including
      round trips with Miami and Hong Kong centres, three cities as arrays, and the
      abstract base refusing to instantiate.
- [ ] Optional: turn the checks into `tests/test_projection.py` with pytest.
- [ ] Later: a `valid` mask from `inverse` for projections with off-Earth pixels (the globe).

### 5. Base map
- [ ] `render.draw_map(projection, width)` uses `inverse` to build the day map from the Blue
      Marble image (`earthwall2\data\bluemarble_200411.jpg`), nearest pixel. Checked
      2026-11-06: centre 0 matches the NASA image exactly; Miami and Hong Kong centres look
      right. 0.3 s at 1920 wide, 0.7 s at 3840.
- [ ] Optional: smoother sampling (bilinear) instead of nearest pixel.
- [ ] Known: a thin bright line at the very top on the 180° meridian is in the NASA image
      itself (its last columns), not a bug in `draw_map`.
- [ ] Read the screen size, scaling and taskbar margin from `QScreen`, and do the layout
      from them.

### 6. Sunlight
- [ ] `sun.subsolar_point(when)` gives the (lat, lon) under the sun (Claude wrote it: pure
      USNO formulas). Checked against both solstices, the equinox and 3 Nov.
- [ ] Night image: `data\blackmarble_2016_5400x2700.jpg` (NASA Black Marble 2016, shrunk
      once from 13500×6750 to match the day image).
- [ ] Lighting in `render.draw_map(projection, width, when)` (Claude wrote it at Chris's
      request). The sun's height at every pixel, a smooth fade from +3° to −6°
      (`TWILIGHT_TOP` / `TWILIGHT_BOTTOM`), and a blend of day with Black Marble night.
      Checked 2026-11-06: "now" lights the Americas with Europe and Asia at night; on
      21 June the Arctic is lit all round; on 21 December it is dark. About 2 s at 3840 wide.
- [ ] Look choices made as placeholders, so change them if you like: night is Black Marble
      as-is (faint blue Earth plus lights), and the day side is not dimmed toward the
      terminator.
- [ ] Optional speed-up: load the two images once instead of on every redraw (most of the 2 s).

### 7. Wallpaper (Windows)
Scope: primary monitor only (`device_name: primary` = the monitor at (0, 0)).

- [ ] Turn off the old `earthwall` scheduled task: **disabled** 2026-11-06 (still
      registered; `Enable-ScheduledTask earthwall` turns it back on).
- [ ] `uv add comtypes`.
- [ ] `wallpaper_windows.py` (Claude wrote it: the COM boilerplate). Provides
      `connected_monitors()`, `primary_monitor_id()`, `get_wallpaper(id)` and
      `set_wallpaper(id, path)`. Tested: 3 connected monitors found, disconnected ones
      skipped, and setting works.
- [ ] `wallpaper.py` (Claude wrote it at Chris's request): the `Desktop` ABC
      (`primary_screen`, `screen_size`, `get_current`, `set`, `restore`), `WindowsDesktop`,
      `get_desktop()` choosing by `sys.platform`, and `remember_original()` with the crash
      guard.
- [ ] `render.draw_screen(projection, screen_size)`: the map full width on a black
      screen-sized canvas, map at the top and the band below for now.
- [ ] Wired into `run()`: load config → record the original (saved if it changed) →
      `draw_screen` → save `earthwall.png` next to `config.yml` → `set`; restore on
      `aboutToQuit`.
- [ ] Tested 2026-11-06 (a test copy quit itself after 6 s): during the run the primary
      showed `earthwall.png` (3840×2160); after Quit it was restored. The crash guard keeps
      the old original when the screen already shows our image.
- [ ] The first recorded original was the old app's image. Fixed 2026-11-06: the primary
      was set back to `C:\Windows\Web\Wallpaper\Windows\img0.jpg` and `original` cleared,
      so the next run records `img0.jpg`.
- [ ] `save_config` writes the merged config, so `config.yml` now lists every default
      (`check:`, `source:`, `location:` and so on). Fine, or only save the original?
- [ ] Draws once at startup; the redraw every minute is step 8.
- [ ] Test whether restore also happens on sign-out.

### 8. Redraw loop and marquee
- [ ] Redraw every `REDRAW_SECONDS` (60) with a `QTimer` inside the app (not a scheduled
      task). The dot turns yellow while drawing and green when done. A failure turns it red,
      with the reason in the tooltip, and stops redraws (v1 red rule). Claude wrote it.
- [ ] Measured 2026-11-06: about 1.7 s per redraw at 3840×2160. The NASA images are now
      decoded once (`render._load`, cached).
- [ ] Marquee band: `draw_screen(..., band_position, band_text)`. The position comes from
      the monitor's marquee view (`top` puts the map at the bottom). For now it shows a
      placeholder line (date and time) until feeds exist.
- [ ] Tested with a 4 s timer for 11 s: 3 redraws, dots Yellow→Green ×3, `original`
      recorded as `img0.jpg`, and it was restored on Quit.
- [ ] Replace the blind 60 s timer with **event-driven redraws**: redraw when a feed on this
      monitor reports new data, coalescing near-simultaneous updates into one redraw, **and at
      least every 10–15 minutes** so the sun keeps moving.
- [ ] Marquee has two layouts. Layout 1 is on the map (logo and title, no link). Layout 2
      is its own window that scrolls messages away every `interval_seconds` (logo and
      hyperlink). Layout 2's rotation costs no redraw.
- [ ] Layout 1 (on the map): does the title rotate through items (each change is a
      wallpaper redraw), or show only the newest item until the feed updates?
- [ ] Layout 2 config: scroll direction (`up` / `left`) in the YAML; remember each window's
      position and size between runs?
- [ ] Layout 2 windows can belong to virtual monitors, so several can exist at once (e.g. 5),
      each movable anywhere, including over the map.
- [ ] Layout 2 window decisions (2026-11-06):
  - a **normal window** (comes to the front when clicked, goes behind when you switch away);
  - **position and size saved** into that monitor's entry in `config.yml` and reused at start;
  - scrolls **up** by default, with `scroll: up|left` in the marquee view;
  - a **text badge** with the source name (BBC, NHC) instead of a logo for now; real logos
    are a later step.
- [ ] Built (Claude, 2026-11-06): `marquee_window.py`, `MarqueeWindow`. A source badge
      (BBC red, NHC blue, others grey) plus the title as a link (`setOpenExternalLinks`). Every
      `interval_seconds` the message slides away **up** or **left** (`scroll`). Moving or
      resizing saves `window: {x, y, width, height}` into that marquee view, 1 s after the
      last move.
  - `tray.py`: `screen_for_position(n)` (1 = primary, others by left edge). Every *other*
    connected monitor entry with a `position` gets a window per marquee view; entries
    whose screen isn't connected are skipped. Their feeds join the service's list. A
    window waiting for data shows the first item as soon as it arrives.
  - Config: monitors have `position` (the primary is `position: 1`); marquee views have
    `scroll` and `window`. Chris's `config.yml` gained a monitor 2 entry ("asus right",
    hurricane + BBC, every 30 s).
- [ ] Tested: the window opened on "ASUS VP28U (1)", the screen at (3840, 0); it showed
      "waiting…", then NHC Pena, then BBC stories; a move was saved (x 4020, y 130) and
      the interval stayed 30. Both badges were checked by screenshot.
- [ ] Not checked by hand: clicking a link (it should open the browser), and how the slide
      animation looks (only the before and after were captured).
- [ ] While the wallpaper redraws (about 1.7 s, on the GUI thread), the marquee pauses. Another
      reason to move drawing to a background thread.
- [ ] Map views on monitors other than the primary are skipped for now (wallpaper is
      primary-only).
- [ ] Source logos: where they come from (bundled image files, or downloaded once, e.g.
      the site's favicon) and where the feed catalogue points to them.
- [ ] Step 9 split: **Chris** writes the `Feed` ABC and `BBCFeed` (`feeds.py`). **Claude**
      writes the plumbing (`feed_service.py`: background fetches, timers, the "feed updated"
      signal, the red rule).
- [ ] Headlines in the band, rotating every `interval_seconds` (needs the BBC feed, step 9).
- [ ] With `position: bottom` the band text must sit clear of the taskbar (use `QScreen`
      `availableGeometry`). `top` is fine today.
- [ ] The UI pauses for about 1.7 s per redraw (drawing runs on the GUI thread). Move it to a
      background thread later if that is noticeable.

### 9. Feeds
**Contract** (agreed 2026-11-06): a `Feed` has `name`, `update_minutes`, and
`fetch(client: httpx.Client) -> DataFrame | None`. The DataFrame has columns `title`, `url`,
`source`, `time`, `lat`, `lon`; `None` means nothing changed (ETag 304). Errors just raise.

- [ ] **Claude:** `feed_service.py`. `FeedService(QObject)` with `add(feed)`, `start()`,
      `stop()`, `refresh(name)`, `refresh_all()`, `latest(name)`, `fetched_at(name)` and
      `failed()`. It has one shared `httpx.Client` (User-Agent set), and fetches on a
      4-thread pool so the GUI never waits. Signals: `feed_updated(name)` and
      `feed_failed(name, reason)`. A DataFrame is replaced whole, only on the GUI thread. The
      red rule is built in: a failing feed stops.
- [ ] Tested 2026-11-06 with stand-in feeds (not Chris's code). A changing feed: 3 fetches →
      3 updates. A 304-style feed: 1 update, then quiet. An erroring feed: 1 failure, then
      stopped. All fetches ran off the GUI thread, and signals arrived on it.
- [ ] **Chris:** `feeds.py`, the `Feed` ABC and `BBCFeed` (RSS → 6-column DataFrame).
      Checked 2026-11-06: 25 headlines with real times.
- [ ] **Step 4 decided: B.** BBC sends no ETag and no Last-Modified, so `BBCFeed.fetch`
      always returns its items, and the wallpaper redraws every BBC `update_minutes` (5).
      ETag handling waits for a feed that supports it (e.g. AP News).
- [ ] **Claude:** wired into `tray.run()`.
  - Feeds are built from `feed_catalog` for the names the primary monitor's views use, via
    `FEED_TYPES` (`{"bbc": BBCFeed}`); `hurricane` is skipped until written.
  - Redraws happen 2 s after a feed update (coalesced), plus at least every
    `REFRESH_MINUTES` (10). The blind 60 s timer is gone.
  - The newest headline across the marquee's feeds goes in the band as "BBC:  title".
  - `feed_failed` turns the dot red, with the reason. Red is now sticky in `set_status`
    (yellow and green can't cover it), and `set_red()` was added.
  - The service stops on quit, and the original wallpaper is restored.
- [ ] Tested 2026-11-06. Real BBC: a redraw at 0.2 s ("waiting for headlines…"), then at
      4.7 s with the real headline; the original was restored on quit. A failing stand-in
      feed gave a red dot with "feed 'bbc' stopped: ConnectionError…".
- [ ] Hurricanes: `feeds.Hurricanes` (Claude filled in Chris's started class at his
      request). It reads NHC `CurrentStorms.json` and returns one row per storm: title such
      as "Tropical Storm Pena, 40 kt", the public-advisory URL, source `NHC`, lastUpdate,
      lat and lon. Registered in `FEED_TYPES` as `hurricane`.
  - NHC **does** send an ETag. It is used when the catalogue entry has `check: etag`
    (`config.yml`'s `hurricane` has `check:` empty today, so it's off until you set it).
  - Tested 2026-11-06: live data gives 1 storm (Pena, 11.3, −150.1); a second fetch with
    the ETag gives `None` (304); NHC's sample file gives 4 storms. With no storms, the
    DataFrame still has the six columns.
- [ ] Storms **on the map** (Claude, 2026-11-06): `render.draw_screen(..., markers=...)` places
      each item with `projection.forward`, as a red dot with a white ring and a label (dark
      outline so it reads on snow). `tray.map_markers()` collects every item with lat/lon
      from the map view's feeds.
- [ ] **Past track** (Chris asked: "current location and the historical map"):
      `Hurricanes` adds an extra **`track`** column, a list of (lat, lon), oldest first,
      read from NHC's best-track KMZ (a zip holding a KML with a `<Point>` per
      observation). It's drawn as a yellow line with dots, leading into the marker, and a
      segment crossing the map's left/right seam is skipped.
  - The contract's six columns are unchanged; `track` is optional, and views that don't
    know it ignore it.
  - Tested: live Pena has 18 points, from 7.7°N 129.8°W to its position now. NHC's 2023
    sample has 4 storms with tracks. Their *finished* best tracks run past the sample's
    "now", so the lines jump back. That's an artefact of the old sample, not live behaviour.
- [ ] Labels can overlap: neighbouring storms (Gert / Franklin), and a label drawn over its
      own track when the storm moves west. Possible fix: put the label on the side away
      from the track.
- [ ] **Hurricanes split into three feeds** (Chris's design; Claude wrote it, 2026-11-06).
      Each is its own `feed_catalog` entry, so views pick any combination:
  - `hurricane` → `Hurricanes`: where each storm is now (`lat`/`lon` → a marker).
  - `hurricane_track` → `HurricaneTracks`: past track (`geometry` = one line, ending at now).
  - `hurricane_cone` → `HurricaneCones`: forecast cone (polygon) plus forecast points,
    together (Chris's choice).
  - They share `_NHCFeed`: `CurrentStorms.json` with the ETag when `check: etag`, plus the
    common columns. `kmz_shapes()` reads any NHC KMZ into (kind, points).
  - Extra columns: **`id`** (NHC's storm id, the same across the three) and **`geometry`**
    (a list of (kind, points), kind `line` / `polygon` / `point`). The six contract
    columns are unchanged.
  - Render: `draw_screen(..., markers=..., shapes=...)` draws cones translucent, then
    lines, then forecast points, then markers on top. A shape that crosses the map's
    left/right edge is skipped.
  - Config: the two new catalogue entries were added to `DEFAULTS`, and the default map view
    lists all three. Chris's `config.yml` map view was updated to list them too.
- [ ] Tested: live Pena gives 1 marker, a track of 19 points, a cone polygon and 7 forecast
      points; NHC's sample gives 4 storms. A second cone fetch with the ETag gives `None`.
      The real app ran for 20 s: BBC and all three hurricane feeds came together in **one**
      redraw.
- [ ] Worldwide storms (typhoons, cyclones) are not covered by NHC. Add another source later
      (e.g. NASA EONET or GDACS).
- [ ] Fires.
- [ ] Plane tracking.

### 9b. Query layer
- [ ] Write query functions over the store (`get_hurricanes()`, `storm_track(name)`,
      `find_flight(...)`, `fires_near(...)`, `headlines()`…). Names are placeholders.
- [ ] Write clear docstrings and type hints, because they become the MCP tool descriptions.
- [ ] Include fetch and observation times, and report a stopped (red) feed explicitly.

### 10. Tray menu, full version
- [ ] Status lines.
- [ ] Open settings…
- [ ] Open dashboard.
- [ ] Restore default background.
- [ ] Update all now.
- [ ] View log.
- [ ] Quit.

### 11. Settings window (control box)
- [ ] A `QDialog` that edits the config and saves it.

### 12. Dashboard
- [ ] `uv add dash`.
- [ ] Run it on a background thread, bound to `127.0.0.1`, with `debug=False`.
- [ ] Plotly charts and map from the DataFrames, refreshed with `dcc.Interval`.

### 12b. MCP server
- [ ] `uv add mcp`.
- [ ] Run `MCPServer` with the streamable HTTP transport on a background thread (with
      its own asyncio loop), bound to `127.0.0.1`.
- [ ] Each tool is a thin wrapper over the query function of the same name. Convert
      DataFrames to small JSON-friendly records, including timestamps and NaN handling.
- [ ] Register it in Claude Code as an HTTP server and ask it where a storm is.

### 13. Start at login
- [ ] Pick the mechanism, then set it up.

## Open questions

- [ ] **Broken config file under `pythonw`:** there is no console, so how does the user
      see "bad yaml"? (A message box, or the log only?) Not needed until autostart.
- [ ] **Start at login:** Startup folder, the Registry `Run` key, or Task Scheduler at log on?
- [ ] **Fetch threading:** one thread per feed with the sync client, or asyncio tasks with
      `httpx.AsyncClient`?
- [ ] **Data sources:** fires, planes and news (hurricanes probably NHC).
- [ ] **"Custom" plane tracking:** specific flights, airlines, a region, or something else?
- [ ] **Marquee vs redraw timing:** one setting, or require redraw interval ≤ headline
      interval?
- [ ] **Long headlines:** shrink the font, truncate with "…", or wrap onto two lines?
- [ ] **Ultrawide screens** (wider than the map's shape allows): fit the map by height and
      leave side strips, or crop the poles?
- [ ] **Both views disabled:** allowed (the app just gathers data and shows status), or
      should the settings window refuse it?
- [ ] **When config changes take effect:** right away on save, or on restart (Dash cannot
      be stopped cleanly while running)?
- [ ] **Monitors:** primary only, or more?
- [ ] **Common item shape for the marquee:** what every feed must provide. At least
      `title` and `url`; probably also `time` and `source`, and `lat`/`lon` when it has
      a place. As required DataFrame columns, or as a small `dataclass`?
- [ ] **Feed service interface:** `count()`, `item(x)` and `items()` methods, or let it
      behave like a list (`len(feed)`, `feed[ ]`, `for item in feed`) through Python's
      special methods `__len__`, `__getitem__` and `__iter__`?
- [ ] **Which feeds are active:** the union of the feeds named by currently connected
      monitors, recalculated on `screenAdded` / `screenRemoved`.
- [ ] **MCP when no monitor shows a feed:** MCP is a virtual monitor (`device_name: mcp`),
      always connected, with no wallpaper. Its feeds stay active.
- [ ] **Virtual monitor shape:** how it is marked in the config (reserved name, or a flag
      such as `virtual: true`), and whether it has `views` or just `feeds`.
- [ ] **Dash:** a virtual monitor as well, the same as MCP.
- [ ] **Unknown monitor plugged in:** left alone.
- [ ] **Feed catalogue:** kept, renamed `feed_catalog`.
- [ ] **Monitor identity:** both. A config entry matches by `position` (layout: primary = 1,
      then by left edge) or by `device_id` (the `IDesktopWallpaper` ID).
- [ ] **Both match the same screen:** if one entry says `device_id: <this ASUS>` and another
      says `position: 1`, which wins? (Suggestion: `device_id`, because it is more specific.)
- [ ] **A monitor left of the primary** (negative x): is it numbered before 1, or after the
      others?
- [ ] **`DEFAULTS`:** the default monitor's `device_name: primary` becomes `position: 1`
      (with `device_name` kept as a label).
- [ ] **Qt and monitors:** match a `QScreen` (for `screenAdded` / `screenRemoved`) to a
      `device_id` and to a position.
- [ ] **Config shape for monitors and feeds:** a `monitors:` list (real device `id`
      plus a friendly `name`, `view: map|marquee`, `projection`, `feeds`, optional
      `location` and `zoom`) and a top-level `feeds:` section (source and update interval).
      Rewrite the sample in a real `.yml` file first. `merge` replaces lists whole, so
      per-monitor defaults need a `MONITOR_DEFAULTS` merged into each list item.
- [ ] **Feed passwords** (e.g. AP News): keep them out of `config.yml`. Use the `keyring`
      package (Windows Credential Manager) and name only the account in the config.
- [ ] **Query layer return type:** return DataFrames and let each adapter convert them
      (Dash uses them as they are, MCP converts to records), or return plain records or
      dataclasses?
- [ ] **Does the wallpaper read through the query layer too,** or straight from the store?
- [ ] **When red clears:** postponed. v1 keeps it red until close.

## Future ideas

- **A different view on each monitor** (Chris's idea): the config maps each monitor to
  what it shows. Needs:
  - a stable monitor key (the `IDesktopWallpaper` ID, model + port), not a number;
  - quiet skipping of monitors that are not connected (office vs home);
  - a check on render time, since each extra image costs another render.

- Flip the map for the southern hemisphere.
- Smarter red handling: retries before counting a failure, and acknowledging errors.
- A Mercator projection as a second strategy.
- macOS and Linux implementations of the OS interface.