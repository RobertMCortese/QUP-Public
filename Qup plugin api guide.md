# QUP Karaoke Engine — Plugin API Developer Guide

**API Version:** 1.0  
**Engine Version:** 2.0.14+  
**Java:** 21+

---

## Overview

The QUP Plugin API allows third-party developers to extend QUP Karaoke Engine with new functionality without modifying the core application. Plugins are self-contained JAR files that QUP discovers and loads at startup.

### What you can build

- **Song Stores** — browse, purchase, and import songs from online catalogs (e.g. Party Tyme, Sunfly)
- **Playback Listeners** — react to song start/stop events (e.g. OBS recording, analytics, social media posting)
- **Media Providers** — supply background video or other media to the CDG renderer (e.g. YouTube music video backgrounds)
- **General Plugins** — add config panels, custom tools, or any Swing UI to QUP

### Architecture

```
your-plugin.jar
  |-- com/yourcompany/pluginname/YourPlugin.class
  |-- META-INF/services/com.qupkaraoke.plugin.QUPPlugin
```

QUP discovers plugins via Java's `ServiceLoader` mechanism. Each plugin JAR is loaded in its own `URLClassLoader`, receives a `PluginContext` providing a controlled API surface, and has its config panel automatically registered in QUP's settings tree.

---

## Quick Start

### 1. Set up your project

```
my-plugin/
  src/com/mycompany/myplugin/MyPlugin.java
  META-INF/services/com.qupkaraoke.plugin.QUPPlugin
  build.bat
```

### 2. Create the ServiceLoader descriptor

**`META-INF/services/com.qupkaraoke.plugin.QUPPlugin`**
```
com.mycompany.myplugin.MyPlugin
```

### 3. Implement the plugin

```java
package com.mycompany.myplugin;

import com.qupkaraoke.plugin.*;
import javax.swing.*;
import java.util.logging.Logger;

public class MyPlugin implements QUPPlugin {

    private PluginContext context;
    private Logger log;

    @Override public String getId()          { return "com.mycompany.myplugin"; }
    @Override public String getName()        { return "My Plugin"; }
    @Override public String getVersion()     { return "1.0.0"; }
    @Override public String getAuthor()      { return "Your Name"; }
    @Override public String getDescription() { return "Does something cool"; }

    @Override
    public void init(PluginContext ctx) {
        this.context = ctx;
        this.log = ctx.getLogger();
        log.info("My plugin initialized!");
    }

    @Override
    public JPanel getConfigPanel() {
        // Return a settings panel for QUP's config tree, or null
        JPanel p = new JPanel();
        p.setBackground(java.awt.Color.WHITE);
        p.add(new JLabel("Hello from My Plugin"));
        return p;
    }

    @Override
    public void shutdown() {
        log.info("My plugin shutting down");
    }
}
```

### 4. Build

```bat
@echo off
set JAVA_HOME=C:\Program Files\Java\jdk-21
set JAVAC="%JAVA_HOME%\bin\javac"
set JAR="%JAVA_HOME%\bin\jar"
set API_JAR=C:\dev2\QUPKaraokeEngine_64\libs\qup-plugin-api.jar

if not exist build mkdir build
%JAVAC% -cp %API_JAR% -d build src\com\mycompany\myplugin\*.java
xcopy /s /y META-INF build\META-INF\ >nul
%JAR% cf my-plugin.jar -C build .
```

### 5. Deploy

Copy `my-plugin.jar` to the QUP `plugins/` directory and restart QUP.

---

## Plugin Types

### QUPPlugin (base)

All plugins implement `QUPPlugin`. This gives you:

- Lifecycle management (`init`, `shutdown`)
- A config panel in QUP's settings tree
- Access to `PluginContext` for interacting with QUP

### SongStorePlugin

Extends `QUPPlugin` for plugins that sell or import songs from online catalogs.

```java
public class MySongStore implements SongStorePlugin {

    @Override
    public int fetchCatalog() {
        // Download catalog from vendor API
        // Cache locally in context.getPluginDataDir()
        // Call context.registerCatalog(entries)
        return entries.size();
    }

    @Override
    public boolean onPurchaseRequested(CatalogEntry entry, String singerHash) {
        // Handle purchase: download file, call context.importSong()
        // Optionally auto-queue: context.queueSong(singerHash, songHash)
        return true;
    }

    @Override
    public int getDownloadProgress() { return -1; } // 0-100 or -1

    @Override
    public List<CatalogEntry> getCachedCatalog() {
        // Return entries from local cache (no network)
        return cachedEntries;
    }

    @Override
    public boolean requiresKJApproval() { return true; }
}
```

**Key methods:**

| Method | Purpose |
|--------|---------|
| `fetchCatalog()` | Download and register song catalog. Runs on background thread. |
| `onPurchaseRequested(entry, singerHash)` | Handle purchase + download + import. Background thread. |
| `getDownloadProgress()` | Return 0-100 for progress bars, -1 if idle. |
| `getCachedCatalog()` | Return locally cached catalog entries (no network). |
| `requiresKJApproval()` | If true, singer requests are queued for KJ approval. |

### PlaybackListenerPlugin

Extends `QUPPlugin` for plugins that react to song playback events.

```java
public class MyListener implements PlaybackListenerPlugin {

    @Override
    public void onSongStarted(PlaybackEvent event) {
        System.out.println("Now playing: " + event.getSingerName()
            + " - " + event.getSongTitle());
    }

    @Override
    public void onSongEnded() {
        System.out.println("Song finished");
    }

    @Override
    public void onNextSongPrefetch(PlaybackEvent event) {
        // Optional: pre-fetch resources for the next song
    }

    // -- Video provider (optional) --

    @Override
    public boolean isVideoProvider() { return false; }

    @Override
    public File getVideoForSong(String songHash) { return null; }
}
```

**Callbacks:**

| Callback | When | Thread |
|----------|------|--------|
| `onSongStarted(event)` | Song playback begins | Background |
| `onSongEnded()` | Song playback finishes (EOF) | Background |
| `onNextSongPrefetch(event)` | QUP knows the next queued song | Background |
| `getVideoForSong(hash)` | CDG player needs a background video | Playback (return fast!) |

**Video providers:** If your plugin provides video files (e.g. music video backgrounds), override `isVideoProvider()` to return `true` and implement `getVideoForSong()`. QUP calls this on the playback thread, so return a cached `File` immediately or `null` if not yet available. Use `onSongStarted` / `onNextSongPrefetch` to download videos in the background.

---

## PluginContext API Reference

The `PluginContext` is your plugin's gateway to QUP. It's passed to `init()` and should be stored as a field.

### Song Import

```java
// Import a downloaded song file into QUP's media database
boolean success = context.importSong(songImport);

// Check if a song already exists by external store ID
boolean exists = context.isSongInLibrary("PH79809");

// Look up the media database hash after import
String hash = context.getSongHashByExternalId("PH79809");
```

### Catalog Registration

```java
// Register purchasable songs for kiosk/search results
context.registerCatalog(catalogEntries);

// Clear before re-registering
context.clearCatalog();
```

### UI Tab Registration

```java
// Add a tab to the main media browser (KJ-facing)
context.registerMediaTab("My Store", myBrowsePanel);

// Add a tab to the kiosk (singer-facing)
context.registerKioskTab("Song Store", myKioskPanel);
```

### Queue Integration

```java
// Queue a song for a singer after purchase
context.queueSong(singerHash, songHash);
```

### File & Config

```java
// Get persistent data directory: plugins/data/{pluginId}/
File dataDir = context.getPluginDataDir();

// Get a namespaced logger
Logger log = context.getLogger();
```

### UI Notifications

```java
// Status bar message
context.showNotification("Download complete!", false);
context.showNotification("Purchase failed", true);  // error style

// Modal confirm dialog
boolean ok = context.showConfirmDialog("Confirm", "Delete this item?");
```

### App Info

```java
String version = context.getAppVersion();     // "2.0.14"
int major = context.getAppMajorVersion();     // 2
```

### Skin Colors

```java
// Match QUP's current theme
Map<String, Color> colors = context.getSkinColors();
Color bg     = colors.get("panelBg");
Color bgDark = colors.get("panelBgDark");
Color border = colors.get("panelBorder");
Color text   = colors.get("textPrimary");
Color dim    = colors.get("textDim");
Color active = colors.get("textActive");
Color accent = colors.get("accent");
```

---

## Data Classes

### PlaybackEvent

Passed to `PlaybackListenerPlugin` callbacks.

| Field | Type | Description |
|-------|------|-------------|
| `singerName` | String | Singer's display name |
| `singerHash` | String | Singer's database hash |
| `songTitle` | String | Song title |
| `songArtist` | String | Song artist |
| `songHash` | String | Song's media database hash |
| `songPath` | String | Path to the song's directory |
| `songFilename` | String | Song filename |
| `songLength` | int | Duration in seconds |
| `pitchTweak` | int | Key change applied (-12 to +12) |
| `tempoTweak` | int | Tempo adjustment percentage |

### SongImport

Passed to `context.importSong()`.

| Field | Type | Description |
|-------|------|-------------|
| `artist` | String | Song artist name |
| `title` | String | Song title |
| `manufacturer` | String | Disc manufacturer (e.g. "Party Tyme Hd") |
| `cdId` | String | Disc/track ID (e.g. "PH79809") |
| `externalId` | String | Store-specific ID for deduplication |
| `format` | String | File format ("mp3g", "mp4", "cdg") |
| `file` | File | Path to the downloaded file on disk |

**Note:** Use `setCdId()` (not `setCdID()`).

### CatalogEntry

Purchasable song from a remote store.

| Field | Type | Description |
|-------|------|-------------|
| `artist` | String | Song artist |
| `title` | String | Song title |
| `songId` | String | Store-specific ID |
| `purchaseUrl` | String | Direct purchase URL, or null |
| `vendor` | String | Vendor name (e.g. "Party Tyme Hd") |
| `format` | String | File format |
| `price` | double | Price in USD |

---

## Plugin Lifecycle

```
1. QUP starts
2. Core components initialize (database, rotation, playlist, player)
3. PluginLoader scans plugins/*.jar
4. For each JAR:
   a. URLClassLoader created
   b. ServiceLoader discovers QUPPlugin implementations
   c. init(PluginContext) called
   d. getConfigPanel() called — panel added to Settings > Plugins > {name}
   e. Media tabs / kiosk tabs registered if applicable
5. QUP is ready
6. During operation:
   - Song start → PluginLoader.notifySongStarted(event)
   - Song end   → PluginLoader.notifySongEnded()
   - Prefetch   → PluginLoader.notifyNextSongPrefetch(event)
7. QUP shuts down
   a. shutdown() called on each plugin
   b. ClassLoaders closed
```

---

## Config Panel Guidelines

Plugin config panels appear in QUP's settings tree under **Plugins > {Plugin Name}**.

**Required styling:**
- White background (`Color.WHITE`)
- Black text (`Color.BLACK`)
- Standard Swing components (no custom L&F)
- Do NOT apply dark themes — config panels are always light

**Recommended layout:**
- Use `BoxLayout` (Y_AXIS) for vertical stacking
- 12px inset padding
- Bold 14pt title at top
- Group related fields with `BorderFactory.createTitledBorder()`
- Save button at the bottom

---

## File Structure

```
QUPKaraokeEngine_64/
  plugins/
    songshop-plugin.jar        <- plugin JARs go here
    musicvideo-plugin.jar
    obs-plugin.jar
    data/
      com.qupkaraoke.songshop/     <- per-plugin data directories
        songshop.properties
        catalog_cache.json
        downloads/
      com.qupkaraoke.musicvideo/
        musicvideo.properties
      com.qupkaraoke.obs/
        obs.properties
  libs/
    qup-plugin-api.jar         <- compile dependency for plugins
```

---

## Building the Plugin API JAR

Before building any plugin, you need `qup-plugin-api.jar`:

```bat
cd C:\dev2
build-plugin-api.bat
```

This compiles the 7 API interfaces from `src\com\qupkaraoke\plugin\` and outputs to `C:\dev2\QUPKaraokeEngine_64\libs\qup-plugin-api.jar`.

Re-run this whenever the API interfaces change.

---

## Existing Plugins

### SongShop (Party Tyme / OpenKJ)

**Type:** `SongStorePlugin`  
**ID:** `com.qupkaraoke.songshop`  
**Source:** `C:\dev2\songshop-plugin`

Provides in-app purchase of songs from the Party Tyme catalog via the OpenKJ API. Features include catalog browsing with search, one-click purchase, download with retry, and a lock/unlock system for protecting KJ credentials from DJs.

### Music Video Backgrounds

**Type:** `PlaybackListenerPlugin` (video provider)  
**ID:** `com.qupkaraoke.musicvideo`  
**Source:** `C:\dev2\musicvideo-plugin`

Searches YouTube via yt-dlp for music videos matching karaoke songs, downloads and caches them locally, and provides them to the CDG compositor as background video. Place `yt-dlp.exe` in the QUP directory or system PATH.

### OBS Recording

**Type:** `PlaybackListenerPlugin`  
**ID:** `com.qupkaraoke.obs`  
**Source:** `C:\dev2\obs-plugin`

Controls OBS Studio via WebSocket (v5 protocol) to automatically start/stop recording for each karaoke song. Supports custom filename formatting, scene switching (intro/singing/applause/idle), and lower third text source control.

---

## Tips

- **Thread safety:** `PluginContext` methods are thread-safe. Playback callbacks run on background threads. Use `SwingUtilities.invokeLater()` for UI updates from callbacks.
- **No QUP internals:** Plugins must only depend on `qup-plugin-api.jar`. Do not import any `QUPKaraokeEngine.*` classes.
- **Config persistence:** Use `context.getPluginDataDir()` + `java.util.Properties` for settings. The directory is created automatically.
- **Skin matching:** For media browser tabs, use `context.getSkinColors()` to match QUP's current theme. Config panels should always be light-themed.
- **Error handling:** Wrap all operations in try/catch. QUP catches plugin exceptions to prevent crashes, but unhandled errors will be logged as warnings.
- **ASCII only:** All source files must be pure ASCII. QUP's build pipeline (LiteSpeed hosting) fatally errors on non-ASCII characters.
