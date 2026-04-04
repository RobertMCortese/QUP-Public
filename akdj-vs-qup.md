# Rotation Logic Comparison: ml_akdj (AutoKDJ) vs QUP Karaoke Engine

This document provides a detailed analysis of the karaoke rotation algorithms in both the original ml_akdj Winamp plugin (C++) and the QUP Karaoke Engine (Java), highlighting where they align, where they diverge, and what has been fixed.

---

## Architecture Overview

### ml_akdj
- Written in C++ as a Winamp media library plugin
- Rotation state managed by `CKaraokeRotation` class using linked lists (`stSongListEntry`, `stSingerListEntry`)
- Song queue is a singly-linked list; insertion walks the list to find the correct position
- Singer data stored in `stSingerEntry` structs pointed to by each song entry
- State persisted to XML files (`akdjsong.xml`, `akdjsing.xml`)

### QUP
- Written in Java 21
- Rotation state split across `RotationEngine` (singer management) and `PlaylistEngine` (queue management)
- Song queue is an `ArrayList<PlaylistEntry>` with index-based access
- Singer data tracked via `RotationEntry` objects looked up by singer hash
- State persisted via Java serialization to `.xml` files

---

## Priority Chain Comparison

Both systems walk the playlist from top to bottom, checking each existing entry against the new entry to determine the insertion point. The first matching rule causes a `break` (insert here). The priority chains are compared below in order of evaluation.

| Priority | ml_akdj | QUP | Notes |
|----------|---------|-----|-------|
| 1 | Song already played -- skip | Song already played -- skip | Identical |
| 2 | Lowest priority (standby) -- insert before | Standby entry -- insert before | Identical |
| 3 | 5-ticket bump -- insert before | *Not implemented* | ml_akdj had tiered ticket bumps |
| 4 | KDJ override / OUT_OF_ROTATION -- skip | KJ override -- skip | Identical |
| 5 | 4-ticket bump -- insert before | *Not implemented* | |
| 6 | **Wait time lock** -- skip | **Wait time lock** -- skip | *Key difference -- see below* |
| 7 | 3-ticket bump -- insert before | *Not implemented* | |
| 8 | Dynamic queue lock -- skip | Dynamic queue lock -- skip | Identical logic |
| 9 | Relative queue lock -- skip | Relative queue lock -- skip | Identical logic |
| 10 | 2+ ticket bump (by count) -- insert before | *Not implemented* | QUP uses bribe system instead |
| 11 | **New singer preferred** -- insert before | **New singer preferred** -- insert before | *Different implementation -- see below* |
| 12 | Target rotation less than current -- insert before | Target rotation less than current -- insert before | Identical |
| 13 | Same rotation, rotation index comparison | Same rotation, rotation index comparison | Similar with minor differences |

---

## Key Difference 1: Max Wait Time Lock

This is the most significant behavioral difference between the two systems, and was the source of a long-standing bug in QUP that has now been fixed.

### ml_akdj Behavior (Correct)

ml_akdj uses TWO mechanisms to ensure only a singer's **next song** gets the wait time lock:

**Mechanism A: singerRepeatValue check**
```cpp
if ((m_maximumSingerWaitTime > 0) &&
    (song->singerRepeatValue == song->singer->numberOfSongsSung + 1) &&
    (GetWaitTimeInSeconds(entryIndex) > m_maximumSingerWaitTime))
{
    return QUEUE_LOCK_TIME;
}
```
The `singerRepeatValue` is the ordinal position of this song among the singer's total entries. `numberOfSongsSung + 1` is "the next song to be sung." So this condition only matches the singer's immediate next song.

**Mechanism B: timeEntered zeroing**

The `_updateSingerRepeatValues()` method zeroes out `timeEntered` on ALL songs from a singer except their first unplayed entry. Since `GetWaitTimeInSeconds()` returns 0 for entries with `timeEntered=0`, subsequent songs can never trigger the wait time lock.

This belt-and-suspenders approach means: if Bob queues 3 songs and has been waiting 40 minutes past the max, only his first song is locked. His second and third songs are freely bumpable by new singers, rotation, or anyone else.

### QUP Behavior (Original -- Bug)

QUP's original `isSongEntryQueueLocked()` had NO singer-repeat check:
```java
if ((!NO_MAX_SINGER_WAIT_TIME && MAX_SINGER_WAIT_TIME > 0) &&
    (now - timeEntered > MAX_SINGER_WAIT_TIME * 60))
{
    return QUEUE_LOCK_TIME;
}
```
This locked ALL entries for a singer once any of them exceeded the max wait time. If Bob queued 3 songs, all 3 were locked -- nobody could jump past any of them. This negated the dynamic rotation.

### QUP Behavior (Fixed)

The fix adds `isSingersFirstUnplayedEntry()`:
```java
if ((!NO_MAX_SINGER_WAIT_TIME && MAX_SINGER_WAIT_TIME > 0) &&
    (now - timeEntered > MAX_SINGER_WAIT_TIME * 60) &&
    isSingersFirstUnplayedEntry(ple.singerHash, entryIndex))
{
    return QUEUE_LOCK_TIME;
}
```
This scans the queue above the current position; if the same singer has an earlier unplayed entry, this one is not their "next up" song and does not get the time lock.

**Note:** QUP does NOT zero out `timeEntered` on subsequent entries like ml_akdj does. The `isSingersFirstUnplayedEntry()` check alone is sufficient because it directly answers the question "is this the singer's next song?" without needing to manipulate timestamps. Both approaches produce the same result.

---

## Key Difference 2: New Singer Preference

### ml_akdj: Graduated Preference Factor (0-100)

ml_akdj uses `m_newSingerPreferenceFactor` (0-100) with a formula:
```cpp
else if ((isNewSinger) &&
         (currSinger->numberOfSongsSung != 0) &&
         (currSinger->rotationIndex) >= (numSingers-1) * (100-m_newSingerPreferenceFactor) / 100)
```
At factor 100, new singers jump ahead of everyone who has already sung. At factor 50, new singers only jump ahead of singers in the bottom half of the rotation order. At 0, new singer preference is effectively disabled.

For the same-rotation tiebreaker, ml_akdj checks `singerRepeatValue == 1` (first song ever entered) with factor > 50:
```cpp
if ((m_newSingerPreferenceFactor > 50) &&
    (currSongListEntry->songEntry.singerRepeatValue == 1) &&
    (newSinger->numberOfSongsEntered != 0))
```

### QUP: Binary On/Off

QUP uses a simple boolean `NEW_SINGERS_PREFERRED`:
```java
else if ((newPlaylistEntry.isNewSinger()) &&
         (currentRotationEntry.numberOfSongsSung() != 0) &&
         (KaraokeConfigPane.NEW_SINGERS_PREFERRED == true))
```
When on, new singers always jump ahead of anyone who has sung. There is no graduated threshold based on rotation index position. The same-rotation tiebreaker uses `numberOfSongsSung >= 1` and `numberOfSongsInQueue == 1`.

---

## Key Difference 3: Ticket / Bribe System

### ml_akdj: Tiered Ticket Bumps

ml_akdj's ticket system is deeply integrated into the priority chain with four distinct tiers:
- **5 tickets**: Jump to the very top (after played songs only)
- **4 tickets**: Jump after KDJ overrides
- **3 tickets**: Jump after wait-time-locked entries
- **2+ tickets**: Jump after queue locks, ordered by ticket count

Tickets are per-song-entry (`ticketBumpValue`) and checked at multiple points during the priority walk. Higher-tier tickets interrupt the normal priority chain at specific points.

### QUP: Bribe Credits System

QUP has a bribe system (`BribeConfigPane`, `bribeValue` on `PlaylistEntry`) but it is NOT integrated into the rotation priority chain. Bribes are handled separately and do not participate in the `getPlaylistInsertPoint()` algorithm. The bribe system primarily affects singer ordering through a separate UI mechanism rather than through algorithmic playlist insertion.

---

## Rotation Round Calculation

Both systems use identical logic for calculating a singer's target rotation:

```
// Both ml_akdj and QUP:
if (new singer with 0 songs sung and 0 in queue):
    targetRotation = currentRotation  // virgins go right in

else:
    rv = currentRotation + numberOfSongsInQueue
    if (rotationLastEntered >= rv): rv = rotationLastEntered + 1
    if (rv > maxRotationsQueued + 1): rv = maxRotationsQueued + 1
    targetRotation = rv
```

The rotation round advances when a song is dequeued (played). Both systems use this to determine which "round" a singer belongs to, then compare rounds during the priority walk.

---

## Queue Lock Systems

Both systems implement identical queue lock mechanisms:

### Dynamic Lock
Locks the first N entries in the queue when the queue exceeds a threshold size.
- ml_akdj: `m_songsQueued > m_queueLockThreshhold && entryIndex < m_queueLockValue`
- QUP: `currentPlaylist.size() > DYN_LOCK_MIN_QUEUED && entryIndex < DYN_LOCK_NUM_SONGS`

### Relative Lock
Locks a percentage of the top of the queue.
- ml_akdj: `((entryIndex+1) * 100) / m_songsQueued <= m_queueLockPercentage`
- QUP: `((entryIndex+1) * 100) / currentPlaylist.size() <= RELATIVE_LOCK`

### Wait Time Lock
Locks entries that have exceeded the max singer wait time. Both use wall-clock elapsed time since the song was queued.
- ml_akdj: Only the singer's next song (via `singerRepeatValue` check + `timeEntered` zeroing)
- QUP (fixed): Only the singer's first unplayed entry (via `isSingersFirstUnplayedEntry()`)

---

## Features in ml_akdj Not Present in QUP

| Feature | Description |
|---------|-------------|
| Tiered ticket bumps (5/4/3/2+) | Four priority levels integrated into the insertion chain |
| Graduated new singer factor | 0-100 scale vs QUP's binary on/off |
| Max singer repeats | Hard cap on songs a singer can queue (`m_maxSingerRepeats`) |
| Singer repeat value tracking | Each song entry knows its ordinal position among the singer's entries |
| timeEntered propagation | Only the first queued song carries the timestamp; others are zeroed |
| Debug file output | Writes detailed rotation decisions to `akdjdbug.txt` with timestamps |
| Queued songs history | Tracks which songs have been sung to prevent re-queueing |
| Song exchange | Swap two songs' positions within the same singer's entries |
| Placeholder singer | Special singer entry for non-rotation songs (filler, KDJ picks) |

## Features in QUP Not Present in ml_akdj

| Feature | Description |
|---------|-------------|
| Cloud queue (phone queueing) | Singers queue from phones via web interface |
| Singer notifications | PWA push notifications for queue position |
| Bribe/credit system | Financial incentive system (separate from rotation) |
| Filler integration | Filler tracks queued directly into the karaoke playlist |
| Song replace/change | Cloud-initiated song replacement |
| Closing time estimation | Calculates remaining songs before closing based on queue lengths |
| KJ override lock type | Separate lock type for manually repositioned entries |

---

## Summary of Current State

After recent fixes, QUP's rotation algorithm is now functionally equivalent to ml_akdj for the core behaviors:
- Rotation round calculation is identical
- Queue lock systems (dynamic, relative, wait time) are identical
- Wait time lock correctly protects only the singer's next song (was a bug, now fixed)
- New singer preference works similarly but with less granularity (binary vs graduated)

The main behavioral gap is the absence of tiered ticket bumps in QUP's insertion algorithm. The bribe system exists but operates outside the rotation priority chain.
