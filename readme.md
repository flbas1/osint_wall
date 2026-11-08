# earthwall2

A live world map for the desktop: a sunlit Earth with city lights on the night side,
hurricane tracks, tracked planes, fires and a news marquee. The data can be seen as the
desktop wallpaper, as a Dash dashboard in the browser, and through an MCP server that
lets Claude answer questions about it.

This is a **Python learning project**. Chris writes the Python and Claude guides and
maintains the docs (this README and [TODO.md](TODO.md)).

This file records the design decisions made so far. Open questions are in TODO.md.

---

## Platform

- Anything OS-specific sits behind a small interface with one implementation per OS,
  chosen at startup with `sys.platform` (`"win32"`, `"darwin"`, `"linux"`). This covers:
  setting the wallpaper, reading it, restoring it, and starting at login.
- Everything else (feeds, rendering, tray, settings, dashboard) must not depend on the OS.
- Nothing is hard-coded for one PC. Screen size, scaling and taskbar position are read
  at runtime (see [Layout](#layout)).

## Libraries

| Library | Used for | Installed |
|---|---|---|
| pandas | The data store. Each feed's latest data is a DataFrame | yes |
| numpy | Array maths for projections and lighting | yes |
| Pillow | Drawing the wallpaper bitmap and the tray-icon dots | yes |
| httpx | Fetching feeds (sync or async, decided later) | yes |
| PySide6 (Qt) | Tray icon, menu, settings window, screen info | yes |
| Dash | Dashboard web view | not yet |
| mcp | MCP server view (HTTP transport) | not yet |
| ruamel.yaml | Config file (keeps comments on save) | not yet |

## How it runs

- A **long-running tray app** built with PySide6. Qt's event loop owns the main thread.
- Started at login. Which mechanism is still open (see TODO).
- On Windows it runs under `pythonw.exe`, so there is no console window. `print()` goes
  nowhere, so all output goes through `logging` to a log file.

## Architecture

```
 monitors (config) each list the feeds they want
        |  union of every monitor's feeds
        v
 feed service: one global list of active feeds, each updating on its own timer
   - data store: latest raw pandas DataFrame per feed
   - query layer: e.g. get_hurricanes(), count, item[x], items
        |
   +----+--------------+-----------------+--------------------+
   v                   v                 v                    v
 wallpaper view      dashboard view    MCP view             marquee view
 (Pillow bitmap)     (Dash, browser)   (tools for Claude)   (Qt window, links)
```

- **The feed service owns all fetching.** The **active feeds** are the union of the
  feeds named by the monitors that are **connected right now**. A feed listed by two
  monitors is still fetched only once. At work the ASUS monitors' feeds run; at home the
  LG monitors' feeds run, and the work-only feeds stop.
- **Monitors come and go at runtime.** Qt's `QGuiApplication.screenAdded` and
  `screenRemoved` signals trigger the following:
  - **known monitor connected:** record its `original` wallpaper, take over its
    background, and add its feeds to the active set;
  - **monitor removed:** drop its feeds, unless another connected monitor still uses them.
  - Matching a `QScreen` to a config entry and to the `IDesktopWallpaper` ID is still
    open: `QScreen.name()` (`\\.\DISPLAY3`) changes between plug-ins, so
    model/serial (or the `AUS28B1` part of the Windows ID) is the likely key.
- **Every view is a consumer of the service, at the same level**, including the marquee.
  Views only query and never fetch.

- **Fetching and drawing are separate.** Each feed fetches on its own interval and
  replaces its DataFrame in the store. The views never wait on the network.
- **One query layer, shared by every view.** A function such as `get_hurricanes()` is
  written once. Dash calls it, and the MCP tool of the same name calls exactly the same
  function. Each view only adds a thin adapter:
  - Dash turns the result into a Plotly figure or table.
  - MCP turns it into JSON-friendly data.
- Query functions only read the store. They never fetch.
- Each query function's **docstring and type hints become the MCP tool's description**,
  so write them for a reader who has never seen the code.
- Shared data across threads: either use a lock, or always **replace** a whole
  DataFrame and never edit one in place.

## Feeds

| Feed | Update rule | Source |
|---|---|---|
| Hurricanes: three feeds. `hurricane` (position now), `hurricane_track` (past track), `hurricane_cone` (cone + forecast points) | own interval; ETag when `check: etag` (NHC sends one) | NHC `CurrentStorms.json`, plus each storm's best-track / cone / forecast-track KMZ |
| Plane tracking (custom) | own interval (config) | TBD. What "custom" means is still open |
| Fires | own interval (config) | TBD |
| News | conditional GET using ETag / `If-None-Match`. A `304` means nothing changed | TBD |

- Set the HTTP `User-Agent` once, on the client. `api.weather.gov` rejects requests
  without one.
- httpx does not cache. The app stores the last `ETag` itself.

## Views

The views (wallpaper, dashboard, MCP, marquee window) can all run at the same time.
Dash, MCP and the marquee read through the feed service's query layer. Whether the
wallpaper does too is still open (see TODO).

Each **monitor** in the config says what it shows (a map or a marquee) and which feeds
it wants. The config shape for monitors and feeds is being redesigned (see TODO).

### Wallpaper bitmap
- **Redrawn only when there is a reason**, never on a blind timer. A reason is one of the
  monitor's feeds reporting new data (the feed service signals "feed X updated").
  - Several feeds updating at about the same time cause **one** redraw (they are coalesced).
  - **At least every 10–15 minutes regardless**, so the day/night line keeps moving even
    when no feed has changed (the sun moves about 1° every 4 minutes). The exact number and
    whether it is a config setting are decided when it is coded.
  - Still open: whether rotating the marquee headline counts as a reason.
  - The interim 60-second `QTimer` from step 8 is replaced once the feed service exists.
- Cost measured 2026-11-06: about 1.7 s per redraw at 3840×2160, with no noticeable CPU spike.
- When the bitmap is disabled, nothing touches the wallpaper.

### Dashboard (Dash)
- Viewed in the **browser**.
- Bind to **`127.0.0.1` only, never `0.0.0.0`**, because this PC is on the company network.
- Runs on a background thread with `debug=False` (the reloader does not work off the
  main thread).
- The page refreshes itself with `dcc.Interval`.
- When the dashboard is disabled, the server never starts and no port is opened. The
  tray's "Open dashboard" item is hidden or greyed out.

### MCP server
- Lets Claude answer questions such as "where is that flight?" or "how is the hurricane
  tracking?" from the app's live data.
- Runs **inside the tray app**, so it reads the same data store directly. It uses MCP's
  **streamable HTTP** transport on a background thread with its own asyncio loop, and
  is only available while the app is running.
  - Not stdio: a stdio server is a separate process and cannot see the app's memory.
- Bind to **`127.0.0.1` only**. Switched on or off by a config boolean, on its own port.
- Uses the `mcp` package (2.3.0, class `MCPServer` from `mcp.server.mcpserver`, as in
  `C:\work\myTestPython\noaa.py`).
- Tools are thin wrappers over the query layer, with the same names. Their results:
  - are small, filtered records, never whole DataFrames;
  - include fetch and observation times, so answers can say how fresh they are;
  - say plainly when a feed is stopped (red) instead of returning an empty result.
- Registered in Claude Code as an HTTP MCP server pointing at the local URL.

## Map

- **Equirectangular** projection, so the full poles are shown. North is always up
  (flipping hemispheres is a possible future option).
- **Vertical:** fixed. The equator sits in the middle of the map, the North Pole at the
  top, the South Pole at the bottom.
- **Horizontal:** centred on the device's **longitude**. Moving the device just shifts
  the map left or right.
- **Sunlight:** the day side is lit to match where the sun is now. The night side shows
  city lights.

### Projections are swappable (strategy pattern)
- A projection is an object passed to `draw_map(...)`. Swapping `Equirectangular` for
  `Mercator` must not change anything else.
- Each projection provides:
  - **forward**: lat/lon → x/y. Used to place overlays such as tracks, planes and fires.
  - **inverse**: x/y → lat/lon. Used to build the base map: for each output pixel, which
    spot on Earth does it show?
  - its own limits (Mercator cannot reach the poles) and the height it needs for a
    given width (Equirectangular is 2:1).
- The centre longitude is given when the projection is created.
- Both methods accept **numpy arrays**, so one call handles a single point or every pixel.
- The config chooses a projection by name, using a dictionary from name to class.
- Recommended interface style: `abc.ABC` with `@abstractmethod` (closest to a C#
  abstract class). The alternative is `typing.Protocol`.

## Layout

- **One band** next to the map, used only for the **marquee**. The map sits against
  one edge and the band against the other.
- The band's edge is set in config (`top` / `bottom`).
- Everything is worked out from the screen at runtime, in physical pixels:

| Value | Source | This PC |
|---|---|---|
| Screen size | `QScreen` geometry × `devicePixelRatio()` | 3840×2160 (150% scaling) |
| Map height | asked of the projection for that width | 1920 |
| Band height | screen height − map height | 240 |
| Taskbar / menu-bar margin | `geometry()` − `availableGeometry()` | about 72 px at the bottom |

- The marquee must sit clear of the taskbar on Windows and of the menu bar (top) on macOS.
- Desktop icons sit over the left edge of the map. That is just something to be aware of.

## Marquee

- **A consumer of the feed service, at the same level as Dash and MCP.** It asks the
  service for items from one or more feeds and shows them one message at a time. It
  works for any feed: news ("BBC: humans beat AI!"), hurricanes ("NASA: 15 hurricanes",
  linking to nasa.gov), or anything added later. That is why the common item interface
  matters.

### Layout 1: on the map (a monitor with a map view and a marquee view)
- The message title is drawn **on the wallpaper**, in the band next to the map (as built
  in step 8).
- Shows the **source logo** with the title.
- **No hyperlink**: a wallpaper cannot be clicked.

### Layout 2: marquee only (a monitor with a marquee view and no map)
- A **small window** (Qt), not the wallpaper. Resizable and movable.
- Every `interval_seconds` (from the YAML) the current message **scrolls away** (up or
  left) and the next one replaces it.
- Each message shows the **source logo** and is a **hyperlink**: a click opens the source
  in the browser.
- Because it is a window, changing messages costs nothing. No wallpaper redraw is involved.
- **Not tied to a physical monitor.** A layout 2 marquee can belong to a **virtual monitor**
  (like MCP and Dash), so there can be several at once, e.g. 5 marquee windows, each with
  its own feeds. Being virtual, they are always "connected", so their feeds stay active.
- The windows can be **dragged anywhere**, including on top of the map wallpaper.
- Still open: should each window remember its position and size between runs? Should it
  sit just above the desktop (behind other apps, like a widget) or always on top?

### What every feed item must provide
- **title**: the text shown.
- **url**: where a click goes (layout 2).
- **source**: e.g. BBC, NASA, NOAA. The **logo** belongs to the source or feed, not to
  each item.
- probably **time**, and **lat/lon** for items with a place (the map needs those).

## Tray

- **Status dot** on the icon:
  - **green**: running normally;
  - **yellow**: generating a new image;
  - **red**: a feed had an error, or something else needs a look at the log.
- **Red rule (v1):** when a feed errors, that feed **stops** and the dot turns red. It
  stays red until the app is closed and starts green on the next launch.
- Status is shown **in addition to** logging. A custom `logging.Handler` watches log
  records (`WARNING` / `ERROR`) and drives the dot, so both come from the same events.
- **Tooltip:** short status text. Windows limits it to about 127 characters.
- **Right-click menu:**
  - status lines (greyed out);
  - Open settings…;
  - Open dashboard (only when the dashboard is enabled);
  - Restore default background;
  - Update all now;
  - View log;
  - Quit.
- Qt GUI updates happen on the GUI thread. Background threads send **signals**.

## Wallpaper restore

- BrandsMart pushes a default background. The app puts back **whatever was set before
  it started**.
- Restore happens:
  - on Quit and on close;
  - on sign-out (whether Qt's quit signal fires then still needs testing);
  - when the bitmap view is switched off;
  - from the tray menu.
- "Restore" only changes what is shown. The generated image file stays on disk.
- At startup, record the current wallpaper's **real path** in the config as that
  monitor's `original` (it is `None`/`null` until first recorded). This happens at every
  startup unless the current wallpaper is the app's own output.
  - Read it with Windows' `IDesktopWallpaper` COM interface (the `comtypes` package),
    which gives the real path for each monitor. `SystemParametersInfo` and the registry
    only return `...\Themes\TranscodedWallpaper`, Windows' cached copy, which is
    overwritten on every change.
  - The COM code lives in its own module (the OS layer), not in `config.py`.
  - Reference: `C:\work\python\earthwall\src\earthwall\wallpaper.py` (read only).
- **Crash guard:** if the current wallpaper is the app's own image, the app must have
  crashed last time. Keep the saved original and do not overwrite it.
- On this PC no Group Policy locks the wallpaper. Before the old `earthwall` ran, every
  monitor showed `C:\Windows\Web\Wallpaper\Windows\img0.jpg`.

## Config

- **YAML, read and written with `ruamel.yaml`.** It keeps comments and key order when
  the app saves, so hand-written notes in the file survive. Use its round-trip (`rt`) or
  `safe` mode only.
- **Location:** `QStandardPaths.writableLocation(AppConfigLocation)`. Set the application
  name before asking, and create the folder if it is missing.
- **Defaults live in code**, as a nested dictionary, and **they define the shape**. The
  merge walks the defaults and fills in the loaded file in place, which keeps the file's
  comments. For each default key:
  - the key is missing from the file: use the default;
  - the default is a dictionary and the file's value is a dictionary: merge recursively;
  - the default is a dictionary but the file's value is not (the shape changed): the
    file's value is wrong, so replace it with the default;
  - the default is a plain value: keep the file's value, whatever its type.
- **Missing file (first run):** write the defaults to the file, then carry on.
- **Broken file** (invalid YAML): the app **stops**, because it cannot fix a bad YAML
  file. It prints "bad yaml" with the file path and the parser's line/column to the
  console and exits with a non-zero code. The file is never overwritten.
  - Under `pythonw` (autostart) there is no console. A visible message for that case
    (for example a message box) is still open.
- The settings window (control box, opened from the tray) edits the config file.
- `run()` loads the config once and passes it to whatever needs it (no global).

### Shape (implemented in `config.py`)

```yaml
location: {lat: 25.76, lon: -80.19}     # general default; a map view can override it
dashboard: {enabled: false, port: 8050}
mcp:       {enabled: false, port: 8051}

feed_catalog:                           # how to fetch each feed, defined once, by name
  hurricane: {update_minutes: 240, source: noaa_east}
  bbc:       {check: etag, update_minutes: 5}

monitors:                               # a list: each monitor starts with "- "
  - device_name: primary
    original:                           # the wallpaper the app replaced (recorded by the app)
    views:                              # a list: a monitor can show several views
      - type: map
        projection: equirectangular
        feeds: [hurricane]
      - type: marquee
        position: top
        interval_seconds: 60
        feeds: [bbc]
```

- **`feed_catalog`** holds each feed's settings (source, interval, login) once, by name.
  It is a catalogue, not a to-do list. Views refer to feeds by name. The feed service
  fetches the union of the feeds named by the **connected** monitors.
- **How a config entry finds its monitor: either way.** Each entry in `monitors` matches by
  - **`position`** (1, 2, 3…): "the middle screen always shows hurricanes", the same layout
    wherever you are, with any monitor; or
  - **`device_id`** (the long `\\?\DISPLAY#AUS28B1#...UID45383...` ID): "the LG at home
    always shows Taiwan news", which follows the physical monitor and its port.
  - `device_name` is a friendly label for people only; the code never matches on it.
- **Position numbers come from the layout, never from hard-coded pixels:**
  - screen 1 is the **primary** (the monitor at (0, 0));
  - the others are numbered by their **left edge**, using the rectangles
    `wallpaper_windows.connected_monitors()` returns.
  - On the work desk that gives 1 = ASUS at (0, 0), 2 = ASUS at (3840, 0) and
    3 = laptop at (7680, 944).
  - Windows' own numbers (Display Settings "1, 2, 3", `\\.\DISPLAYn`, the COM list
    index) are **not** used: they disagree with each other and change between desks.
- **Unknown monitors** (matched by no entry) are left alone: their wallpaper is not touched.
- **MCP and Dash are monitors too:** virtual entries (for example `device_name: mcp` and
  `device_name: dash`) that are always "connected" and have no wallpaper. The feeds they
  name stay active even when no physical monitor shows them, so Claude can still answer
  "where's the hurricane?" at home, and the dashboard still has data.
  Exactly how a virtual monitor is written in the config is still open.
- **View types:** `map` (`projection`, optional `location` with `lat`/`lon`/`zoom`, `feeds`) and
  `marquee` (`position`, `interval_seconds`, `feeds`).
- **Defaults for list items:** `merge()` replaces lists whole, so `apply_item_defaults()`
  fills each feed from `FEED_DEFAULTS`, each monitor from `MONITOR_DEFAULTS`, and each view
  from `VIEW_DEFAULTS[type]`.
- **Shape errors stop the app**, like bad YAML: an unknown view `type`, `monitors` or
  `views` not being a list.
- Keys the app does not know (for example ones left over from an older shape) are kept
  as they are and ignored.
- Still open: what `device_name` means and how it maps to the Windows monitor ID (it
  could be a friendly name plus the real `id`), what `zoom` means, and keeping feed
  passwords in Windows Credential Manager (`login_account` names the entry).

## Related

- `C:\work\python\earthwall`: the first, Claude-built version. It is kept **only as a
  reference** and is not part of this project. Its scheduled task `earthwall` still
  sets the primary monitor's wallpaper every 15 minutes. It will fight with this app,
  so turn it off before this app starts setting the wallpaper.
- `C:\work\python\earthwall\data`: NASA Blue Marble (day, November) and Black Marble
  (night lights) images, both equirectangular and reusable here.