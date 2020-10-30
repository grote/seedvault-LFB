# Overview

This is a design document for Seedvault Storage backup.
It is heavily inspired by borgbackup, but simplified and adapted to the Android context.

The aim is to efficiently backup media files from Android's `MediaStore`
and other files from external storage.
Apps and their data are explicitly out of scope
as this is handled already by the Android backup system.
Techniques introduced here might be applied to app backups in the future.

# Data structures

## Local caches

### Files cache

This cache is needed to quickly look up if a file has changed and if we have all of its chunks.
It will probably implemented as a sqlite-based Room database.

Contents:

* URI (stripped by scheme and authority?) (`String` with index for fast lookups)
* file size (`Long`)
* last modified in milliseconds (`Long`)
* generation added (MediaStore only) (`Long`)
* generation modified (MediaStore only) (`Long`)
* list of chunk IDs representing the file's contents
* last seen in milliseconds (`Long`) or TTL counter (`Integer`)

If the file's size, last modified timestamp (and generation) is still the same,
it is considered to not have changed.
In that case, we check that all file content chunks are (still) present in storage.

If the file has not changed and all chunks are present,
the file is not read/chunked/hashed again.
Only a file metadata item is added to backup metadata.

As the cache grows over time, we need a way to evict files eventually.
This could happen by checking a last seen timestamp or by using a TTL counter
or maybe even a boolean flag that gets checked after a successful pass over all files.
A flag might not be ideal if the user adds/removes folder as backup targets.

### Chunks cache

This is used to determine whether we already have a chunk,
to count references to it and also for statistics.

It could be implemented as a (joined?) table in the same database as the files cache.

* chunk ID (probably a hash/mac)
* reference count
* size

If the reference count of a chunk reaches `0`,
we can delete it from storage as it isn't used by a backup anymore.

When making a backup pass and hitting the files cache,
we need to check that all chunks are still available on storage.

## Backup Archive

The backup archive contains metadata about a single backup
and is written to the storage after a successful backup run.

* version - the backup version
* name - a name of the backup
* files - a list of all files in this backup
  * URI (includes authority and storage volume)
  * relative path
  * name
  * file size
  * ordered list of chunk IDs (to re-assemble the file)
* total size - sum of the size of all files, for stats
* timeStart - when the backup run was started
* timeEnd - when the backup run was finished

# Operations

## Making a backup

A backup run is ideally (automatically) triggered when

* the device is charging and connected to an un-metered network in case network storage is used
* a storage medium is plugged in and the user confirmed the run in case removable storage is used

Files to be backed up are listed based on the user's preference
using Android's `MediaProvider` and `ExternalStorageProvider`.
Tests on real world devices have shown ~200ms list times for `MediaProvider`
and `~10sec` for *all* of `ExternalStorageProvider`
(which is unlikely to happen, because the entire storage volume cannot be selected on Android 11).

All files will be processed with every backup run.
If a file is found in the cache, it is checked
if its content-modification-indicating attributes have not been modified
and all its chunks are still present in the backup storage.
We might be able to speed up the latter check by initially retrieving a list of all chunks.

For present unchanged files, an entry will be added to the backup metadata
and the TTL in the files cache updated.
If a file is not found in cache an entry will be added for it.
New and modified files will be put through a chunker
which splits up larger files into smaller chunks.
Initially, the chunker might just return a single chunk,
the file itself, to simplify the operation.

A chunk is hashed (with a key / MACed),
then (compressed and) encrypted (with authentication) and written to backup storage,
if it is not already present.
New chunks get added to the chunks cache.
Only after the backup has completed and the backup archive was written,
the reference counters of the included chunks will be incremented.

When all chunks of a file have either been written or were present already,
the file item is added to the backup archive with its list of chunk IDs and other metadata.

When all files have been processed, the backup archive is finalized and written to storage.

If the backup fails, a new run is attempted at the next opportunity creating a new backup archive.
Chunks uploaded during the failed run should still be available in storage
and the cache with reference count `0` providing an auto-resume.

After a successful backup, chunks that still have reference count `0`
can be deleted from storage and cache without risking to delete chunks that will be needed later.

## Removing old backups

Ideally, the user can decide how many backups should be kept based on available storage capacity.
These could be a number in the yearly/monthly/weekly/daily categories.
However, initially, we might simply auto-prune backups older than a month,
if there have been at least 3 backups within that month (or some similar scheme).

After doing a successful backup run, is a good time to prune old backups.
To determine which backups to delete, the backup archives need to be downloaded and inspected.
Maybe their file name can be the `timeStart` timestamp to help with that task.
If a backup is selected for deletion, the reference counter of all included chunks is decremented.
The backup archive file and chunks with reference count of `0` are deleted from storage.

## Restoring from backup

When the user wishes to restore a backup, they select the backup archive that should be used.
The selection can be done based on time and name.
We go through the list of files in the archive,
download, authenticate, decrypt (and decompress) each chunk of the file
and re-assemble the file this way.
Once we have the original chunk,
we might want to re-calculate the chunk ID to check if it is as expected
to prevent an attacker from swapping chunks.
This could also be achieved by including the chunk ID in the authenticated encryption (e.g. AEAD).
The re-assembled file will be placed into the same directory under the same name
with its attributes (e.g. lastModified) restored as much as possible.

If a file already exists with the that name and path,
we check if the file is identical to the one we want to restore
(by relying on file metadata or re-computing chunk IDs)
and move to the next if it is indeed identical.
If it is not identical, we could rely on Android's Storage Access Framework
to automatically give it a `(1)` suffix when writing it to disk.
Normally, restores are expected to happen to a clean file system anyway.

However, if a restore fails, the above behavior should give us a seamless auto-resume experience.
The user can re-try the restore and it will quickly skip already restored files
and continue to download the ones that are still missing.

After all files have been written to a directory,
we might want to attempt to restore its metadata (and flags?) as well.

TODO: figure out how to get the directories of `MediaStore` files.

# Out-of-Scope

The following features would be nice to have,
but are considered out-of-scope of the current design for time and budget reasons.

* compression (we initially assume that most files are already sufficiently compressed)
* external secret-less corruption checks that would use checksums over encrypted data
* supporting different backup clients backing up to the same storage

# Known issues

## Changes to files can not be detected reliably

Changes can be detected using file size and lastModified timestamps.
These have only a precision of seconds,
so we can't detect a changes happening within a second of a first change.
Also other apps can reset the lastModified timestamp
preventing us from registering a change if the file size doesn't change.
On Android 11, media files have a generation counter that gets incremented when files changes
to help with this issue.
However, files on external storage still don't have anything similar
and usually also don't trigger `ContentObserver` notifications.
