# QUP Karaoke Engine — Plugin API Guide

**Version:** 2.0.19  
**Last Updated:** April 2026

---

## Overview

QUP Karaoke Engine supports plugins via a Java ServiceLoader architecture. Plugins are discovered at startup from JAR files in the `plugins/` directory. Each plugin implements one or more interfaces from the `com.qupkaraoke.plugin` package and interacts with QUP exclusively through the `PluginContext` API.

**Key Principles:**
- Plugins MUST NOT access QUP internals directly
- `PluginContext` is the only contract between plugin code and the QUP engine
- All `PluginContext` methods are thread-safe
- New API methods are added in a backwards-compatible way — existing plugins don't need recompilation

---

## Getting Started

### Project Structure

```
my-plugin/
├── src/com/qupkaraoke/plugins/myplugin/
│   └── MyPlugin.java
├── META-INF/services/
│   └── com.qupkaraoke.plugin.QUPPlugin
└── build.bat
```

### ServiceLoader Registration

Create `META-INF/services/com.qupkaraoke.plugin.QUPPlugin` containing your fully qualified class name:

```
com.qupkaraoke.plugins.myplugin.MyPlugin
```

### Compiling

Compile against `qup-plugin-api.jar` (and `miglayout-3.7.1.jar` if you use MigLayout for UI):

```bat
javac -cp "path/to/qup-plugin-api.jar;path/to/miglayout-3.7.1.jar" -d build src/com/qupkaraoke/plugins/myplugin/*.java
xcopy /s /y META-INF build\META-INF\
jar cf myplugin.jar -C build .
copy /y myplugin.jar path/to/QUP/plugins/
```

QUP's Gradle build also has a `compilePlugins` task that auto-compiles all plugins in `plugins-src/`.

---

## Interfaces

### QUPPlugin (required)

The core lifecycle interface. Every plugin must implement this.

```java
public interface QUPPlugin {
    String getId();           // Unique plugin ID (e.g. "com.example.myplugin")
    String getName();         // Human-readable name shown in config tree
    String getVersion();      // Version string (e.g. "1.0.0")
    String getAuthor();       // Author / vendor name
    String getDescription();  // One-line description

    void init(PluginContext context);   // Called once after loading
    JPanel getConfigPanel();            // Config panel for cpanel tree, or null
    void shutdown();                    // Called on QUP exit
}
```

**Config panel rules:** If `getConfigPanel()` returns a JPanel, it MUST use **light theming** (white background, black text, blue accents) to match QUP's control panel style.

### PlaybackListenerPlugin (optional)

Extends `QUPPlugin` for plugins that react to song playback events (e.g. OBS recording, music video backgrounds).

```java
public interface PlaybackListenerPlugin extends QUPPlugin {
    void onSongStarted(PlaybackEvent event);   // Song is about to play
    void onSongEnded();                        // Song finished

    // Optional (default no-ops):
    void onNextSongPrefetch(PlaybackEvent event);  // Pre-fetch next song's resources
    File getVideoForSong(String songHash);          // Provide background video for CDG
    boolean isVideoProvider();                      // Return true if this plugin provides videos
}
```

### SongStorePlugin (optional)

Extends `QUPPlugin` for plugins that provide a remote song store (e.g. Party Tyme, SongShop).

```java
public interface SongStorePlugin extends QUPPlugin {
    int fetchCatalog();                                          // Download/refresh catalog
    boolean onPurchaseRequested(CatalogEntry entry, String singerHash);  // Handle purchase
    int getDownloadProgress();                                   // 0-100 or -1
    List<CatalogEntry> getCachedCatalog();                       // Return cached entries
    boolean requiresKJApproval();                                // KJ must approve purchases?
}
```

---

## PluginContext API Reference

The `PluginContext` is passed to your plugin's `init()` method. This is your entire API surface for interacting with QUP.

### Song Import

| Method | Description |
|--------|-------------|
| `boolean importSong(SongImport song)` | Import a downloaded song file into QUP's media database |
| `boolean isSongInLibrary(String externalId)` | Check if a song exists by external store ID |
| `String getSongHashByExternalId(String externalId)` | Look up a song hash by external store ID |

### Song Query & Management

| Method | Description |
|--------|-------------|
| `List<SongInfo> queryAllSongs(boolean fillerOnly)` | Get all songs from karaoke DB (`false`) or filler DB (`true`) |
| `boolean removeSongByHash(String songHash)` | Remove a song from the DB by hash (file untouched) |
| `boolean reimportSong(String filePath, String artist, String title, String cdid, int importFormat)` | Re-import a song with new metadata. `importFormat` is 0-6 (see below) |
| `void refreshFileList()` | Refresh the main file list UI. Call once after batch operations |

**Import format types:**

| Value | Format |
|-------|--------|
| 0 | Unknown / not set |
| 1 | `\DISC\TRACK - ARTIST - TITLE.EXT` |
| 2 | `\DISC\TRACK - TITLE - ARTIST.EXT` |
| 3 | `\DISC-TRACK - ARTIST - TITLE.EXT` |
| 4 | `\DISC-TRACK - TITLE - ARTIST.EXT` |
| 5 | `\ARTIST - TITLE.EXT` |
| 6 | `\TITLE - ARTIST.EXT` |

### Catalog Registration

| Method | Description |
|--------|-------------|
| `void registerCatalog(List<CatalogEntry> entries)` | Register purchasable songs for kiosk/search |
| `void clearCatalog()` | Clear this plugin's catalog entries |

### UI Tab Registration

| Method | Description |
|--------|-------------|
| `void registerMediaTab(String tabTitle, JPanel panel)` | Add a tab to the main media browser |
| `void registerKioskTab(String tabTitle, JPanel panel)` | Add a tab to the kiosk song browser |

### Queue Integration

| Method | Description |
|--------|-------------|
| `boolean queueSong(String singerHash, String songHash)` | Queue a song for a singer |

### File & Config

| Method | Description |
|--------|-------------|
| `File getPluginDataDir()` | Persistent plugin data directory (`plugins/data/{pluginId}/`) |
| `Logger getLogger()` | Logger namespaced to this plugin |

### UI Notifications

| Method | Description |
|--------|-------------|
| `void showNotification(String message, boolean isError)` | Show a status bar notification |
| `boolean showConfirmDialog(String title, String message)` | Show a modal OK/Cancel dialog |

### App Info

| Method | Description |
|--------|-------------|
| `String getAppVersion()` | Current QUP version string (e.g. "2.0.19") |
| `int getAppMajorVersion()` | Current major version number (e.g. 2) |

### Skin / Theming

| Method | Description |
|--------|-------------|
| `Map<String, Color> getSkinColors()` | Current skin colors (see keys below) |
| `int getListFontSize()` | User's preferred font size for media tables (default 12) |

**Skin color keys:** `panelBg`, `panelBgDark`, `panelBorder`, `textPrimary`, `textDim`, `textActive`, `vfdOn`, `vfdFillerOn`, `accent`

---

## Data Classes

### SongInfo

Read-only song metadata returned by `queryAllSongs()`.

```java
public class SongInfo {
    String getArtist();
    String getTitle();
    String getCDID();          // Disc ID (e.g. "SC8090")
    String getTrackNr();       // Track number (e.g. "14")
    String getFilename();      // Includes extension (e.g. "song.zip")
    String getPath();          // Directory path
    String getExtension();     // File extension (e.g. "zip")
    String getHash();          // SHA1 hash (DB primary key)
    boolean isFiller();        // true = filler track
    int getImportFormat();     // 0-6, see format types above
    String getFullPath();      // Convenience: path + filename
}
```

### SongImport

Data object for importing songs via `importSong()`.

```java
public class SongImport {
    String getArtist() / setArtist(String)
    String getTitle() / setTitle(String)
    String getManufacturer() / setManufacturer(String)  // e.g. "Party Tyme Hd"
    String getCdId() / setCdId(String)                   // e.g. "PH79809"
    String getExternalId() / setExternalId(String)       // Store-specific ID for dedup
    String getFormat() / setFormat(String)                // e.g. "mp3g", "mp4"
    File getFile() / setFile(File)                        // Downloaded file on disk
}
```

### CatalogEntry

A purchasable song from a remote store, registered via `registerCatalog()`.

```java
public class CatalogEntry {
    String getArtist() / setArtist(String)
    String getTitle() / setTitle(String)
    String getSongId() / setSongId(String)          // Store-specific ID
    String getPurchaseUrl() / setPurchaseUrl(String) // Direct purchase URL, or null
    String getVendor() / setVendor(String)          // e.g. "Party Tyme Hd"
    String getFormat() / setFormat(String)           // e.g. "mp3g"
    double getPrice() / setPrice(double)             // USD
}
```

### PlaybackEvent

Song and singer info passed to `PlaybackListenerPlugin` callbacks.

```java
public class PlaybackEvent {
    String getSingerName()
    String getSingerHash()
    String getSongTitle()
    String getSongArtist()
    String getSongHash()
    String getSongPath()
    String getSongFilename()
    int getSongLength()       // seconds
    int getPitchTweak()
    int getTempoTweak()
}
```

---

## Theming Your Plugin

Media tab panels should match QUP's active skin. Use `getSkinColors()` and `getListFontSize()`:

```java
@Override
public void init(PluginContext ctx) {
    this.context = ctx;
    JPanel panel = new JPanel(new BorderLayout());
    JTable table = new JTable(model);

    // Apply skin colors
    Map<String, Color> colors = ctx.getSkinColors();
    table.setBackground(colors.get("panelBg"));
    table.setForeground(colors.get("textPrimary"));
    table.setGridColor(colors.get("panelBorder"));
    table.setSelectionBackground(colors.get("accent"));
    table.setSelectionForeground(Color.WHITE);

    // Apply user's preferred font size
    int fontSize = ctx.getListFontSize();
    table.setFont(new Font("SansSerif", Font.PLAIN, fontSize));
    table.setRowHeight(fontSize + 6);

    ctx.registerMediaTab("My Plugin", panel);
}
```

To update when the user changes skins, add a `HierarchyListener` that re-applies theme colors each time the tab becomes visible:

```java
panel.addHierarchyListener(e -> {
    if ((e.getChangeFlags() & HierarchyEvent.SHOWING_CHANGED) != 0 && panel.isShowing()) {
        applyTheme(table);
    }
});
```

---

## Batch Operations

When modifying multiple songs (import, rename, etc.), suppress per-operation UI refreshes for performance:

```java
// Don't call refreshFileList() inside the loop!
for (SongInfo song : selected) {
    context.removeSongByHash(song.getHash());
    context.reimportSong(newPath, artist, title, cdid, formatType);
}
// One refresh at the end
context.refreshFileList();
```

---

## Shared Resources

Plugins can share tools in `plugins/bin/` (e.g. yt-dlp, deno). Navigate from your plugin data dir:

```java
File sharedBin = context.getPluginDataDir()     // plugins/data/{id}/
    .getParentFile()                             // plugins/data/
    .getParentFile();                            // plugins/
File binDir = new File(sharedBin, "bin");        // plugins/bin/
```

Plugin-specific tools (Python, demucs, whisper, etc.) should go in your own `plugins/data/{id}/bin/`.

---

## Bundled Plugins

QUP ships with these plugins as reference implementations:

| Plugin | Type | Description |
|--------|------|-------------|
| **SongShop** | SongStorePlugin | Party Tyme catalog integration with AES decryption |
| **MusicVideo** | PlaybackListenerPlugin | YouTube background videos for CDG songs via yt-dlp |
| **OBS Recording** | PlaybackListenerPlugin | WebSocket control for per-song OBS recording |
| **KaraokeMaker** | QUPPlugin | AI-powered karaoke track creation (demucs + whisper + ffmpeg) |
| **FileManagement** | QUPPlugin | Batch rename/re-import with format detection and search/replace |

---

## Tips

- **Import `java.util.List`** explicitly in any plugin file that also imports `java.awt.*` (avoids ambiguity with `java.awt.List`)
- **Config panels use light theming** — white backgrounds, black text, blue accents — always
- **Media tab panels use dark theming** — read colors from `getSkinColors()`
- **Never block the EDT** — use background threads for network I/O, file operations, and batch DB operations
- **Plugin JARs in `plugins/` are tracked in git** — commit your built JAR alongside source
