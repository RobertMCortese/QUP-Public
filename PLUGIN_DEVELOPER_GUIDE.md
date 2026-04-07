# QUP Karaoke Engine Plugin Developer Guide

## Overview

QUP Karaoke Engine supports plugins via a thin API JAR (`qup-plugin-api.jar`) that provides interfaces for extending the application without accessing its internals. Plugins are loaded at runtime from `/plugins/*.jar` using Java's ServiceLoader mechanism.

## Plugin types

### QUPPlugin (base)
For general-purpose plugins that add functionality to QUP. Examples: music video to karaoke converter, auto-pitch correction, custom visualizers.

### SongStorePlugin (extends QUPPlugin)
For plugins that connect to remote song stores for browsing, purchasing, and importing karaoke tracks. Examples: Party Tyme SongShop, Sunfly Direct, Karaoke-Version.

## Getting started

### 1. Create a new Java project

Add `qup-plugin-api.jar` to your classpath. Do NOT include QUP's main JAR or any of its internal classes.

### 2. Implement the interface

For a song store plugin:

```java
package com.example.mystore;

import com.qupkaraoke.plugin.*;
import javax.swing.JPanel;
import java.util.List;
import java.util.ArrayList;

public class MyStorePlugin implements SongStorePlugin {

    private PluginContext context;
    private List<CatalogEntry> catalog = new ArrayList<>();

    @Override public String getId()          { return "com.example.mystore"; }
    @Override public String getName()        { return "My Song Store"; }
    @Override public String getVersion()     { return "1.0.0"; }
    @Override public String getAuthor()      { return "Your Name"; }
    @Override public String getDescription() { return "Browse and purchase songs from My Store"; }

    @Override
    public void init(PluginContext context) {
        this.context = context;

        // Register a tab in the main media browser (KJ sees this)
        context.registerMediaTab("My Store", buildMediaTab());

        // Optionally register a kiosk tab (singers see this)
        // context.registerKioskTab("My Store", buildKioskTab());

        // Load cached catalog if available
        List<CatalogEntry> cached = getCachedCatalog();
        if (!cached.isEmpty()) {
            context.registerCatalog(cached);
        }
    }

    @Override
    public JPanel getConfigPanel() {
        // Return a settings panel for the cpanel tree
        // MUST use light theming: white bg, black text, blue accents
        return null;
    }

    @Override
    public int fetchCatalog() {
        // Download catalog from your vendor API
        // Parse into CatalogEntry objects
        // Cache to getPluginDataDir()
        // Call context.registerCatalog(entries)
        return catalog.size();
    }

    @Override
    public boolean onPurchaseRequested(CatalogEntry entry, String singerHash) {
        // Handle purchase:
        //   1. Download the file to context.getPluginDataDir()
        //   2. Create a SongImport with metadata + file path
        //   3. Call context.importSong(songImport)
        //   4. Optionally call context.queueSong(singerHash, songHash)
        return false;
    }

    @Override public int getDownloadProgress()              { return -1; }
    @Override public List<CatalogEntry> getCachedCatalog()  { return catalog; }
    @Override public boolean requiresKJApproval()           { return true; }
    @Override public void shutdown()                         {}

    private JPanel buildMediaTab() {
        // Build your store's browse/search/purchase UI
        // This appears as a tab next to "Karaoke" and "Filler"
        return new JPanel();
    }
}
```

### 3. Register with ServiceLoader

Create the file:
```
META-INF/services/com.qupkaraoke.plugin.QUPPlugin
```

Contents (one line, your fully qualified class name):
```
com.example.mystore.MyStorePlugin
```

### 4. Build and deploy

Build your plugin as a JAR:
```bash
javac -cp qup-plugin-api.jar -d build src/com/example/mystore/*.java
jar cf mystore-plugin.jar -C build .
```

Place the JAR in QUP's `/plugins/` directory. QUP discovers it on startup.

## API reference

### PluginContext methods

| Method | Description |
|--------|-------------|
| `importSong(SongImport)` | Import a downloaded song file into QUP's media database |
| `isSongInLibrary(externalId)` | Check if a song is already owned (avoids re-downloading) |
| `getSongHashByExternalId(id)` | Look up a song's media DB hash by store ID |
| `registerCatalog(entries)` | Register purchasable songs for kiosk search results |
| `clearCatalog()` | Remove this plugin's catalog entries |
| `registerMediaTab(title, panel)` | Add a tab to the KJ's main media browser |
| `registerKioskTab(title, panel)` | Add a tab to the singer kiosk |
| `queueSong(singerHash, songHash)` | Queue an imported song for a singer |
| `getPluginDataDir()` | Persistent storage directory for this plugin |
| `getLogger()` | Namespaced logger for this plugin |
| `showNotification(msg, isError)` | Show a status bar message |
| `showConfirmDialog(title, msg)` | Show a modal OK/Cancel dialog |
| `getAppVersion()` | Current QUP version string |
| `getAppMajorVersion()` | Current QUP major version number |

### Data classes

**SongImport** -- for importing a file into the media database:
- `artist`, `title`, `manufacturer`, `cdId`, `externalId`, `format`, `file`

**CatalogEntry** -- a purchasable song from a store:
- `artist`, `title`, `songId`, `purchaseUrl`, `vendor`, `format`, `price`

## Purchase flow patterns

### Pattern 1: URL-redirect (Party Tyme / OpenKJ style)

```
Singer searches in kiosk -> finds store song -> requests it
  -> KJ sees request in media tab -> clicks "Buy"
  -> Plugin opens purchaseUrl in browser
  -> KJ completes purchase on vendor website
  -> File downloads to a watched directory
  -> Plugin detects file, calls importSong() + queueSong()
```

### Pattern 2: API-mediated (credit-based)

```
Singer searches in kiosk -> finds store song -> requests it
  -> KJ sees request in media tab -> clicks "Buy"
  -> Plugin calls vendor API to claim credit + download
  -> Plugin polls for download progress (getDownloadProgress())
  -> On completion, calls importSong() + queueSong()
```

### Pattern 3: KJ direct purchase

```
KJ clicks plugin's media tab -> browses/searches store catalog
  -> KJ clicks "Buy" on a song
  -> Plugin handles purchase + download + import
  -> Song appears in local library immediately
```

## Rules and constraints

1. **No QUP internals** -- plugins MUST only use the `com.qupkaraoke.plugin` API. Accessing QUP classes directly will break on updates.

2. **Light theming for cpanel** -- any JPanel returned by `getConfigPanel()` must use white backgrounds, black text, blue accents. The media tab and kiosk tab can use any styling.

3. **ASCII only** -- all source files must be pure ASCII. Non-ASCII characters cause build failures on some platforms.

4. **Thread safety** -- `fetchCatalog()` and `onPurchaseRequested()` run on background threads. UI updates must use `SwingUtilities.invokeLater()`. All PluginContext methods are thread-safe.

5. **Graceful degradation** -- if the plugin can't reach its vendor API, it should fall back to cached catalog data silently. Don't block QUP startup.

6. **File cleanup** -- downloaded files in `getPluginDataDir()` are the plugin's responsibility to manage. QUP does not garbage-collect plugin data.

## Example: OpenKJ SongShop integration

The OpenKJ catalog API at `db.openkj.org/apigetsongs_v2` returns:

```json
{
    "command": "getsongs",
    "songs": [
        {
            "artist": "*NSYNC",
            "title": "Bye Bye Bye",
            "songid": "PH00176",
            "url": "https://www.partytyme.net/songshop/...",
            "vendor": "Party Tyme Hd",
            "type": "mp3g",
            "price": 2.49
        }
    ]
}
```

A plugin would:
1. Fetch this JSON in `fetchCatalog()`
2. Map each song to a `CatalogEntry`
3. Call `context.registerCatalog(entries)`
4. Build a searchable media tab with a results table and "Buy" button
5. On purchase, open the `url` in the system browser or handle download directly
6. After download, call `context.importSong()` with the mp3g zip file
