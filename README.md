# Overview

This is a design document for Seedvault Storage backup.
It is heavily inspired by borgbackup, but simplified and adapted to the Android context.

The aim is to efficiently backup media files from Android's `MediaStore`
and other files from external storage.
Apps and their data are explicitly out of scope
as this is handled already by the Android backup system.
Techniques introduced here might be applied to app backups in the future.

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


# Cryptography

The goal here is to be as simple as possible while still being secure
meaning that we want to primarily conceal the content of the backups.
Certain trade-offs have to be made though,
so that for now we do not attempt to hide file sizes
which can especially be an issue for smaller files below the maximum chunk size.
E.g. an attacker with access to the encrypted backup storage might be able to infer
that the Snowden files are part of our backup.
We do however encrypt file names and paths.

## Master Key

Seedvault already uses [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
to give users a mnemonic recovery code and for generating deterministic keys.
The derived key has 512 bits
and Seedvault uses the first 256 bits as an AES key to encrypt app data (out of scope here).
This key's usage is limited by Android for encryption and decryption.
Therefore, the second 256 bits will be imported into Android's keystore for use with HMAC-SHA256,
so that this key can act as a master key we can deterministically derive additional keys from
by using HKDF ([RFC5869](https://tools.ietf.org/html/rfc5869)).
These second 256 bits must not be used for any other purpose in the future.
We use them for a master key to avoid users having to handle another secret.

For deriving keys, we are only using the HKDF's second 'expand' step,
because the Android Keystore does not give us access
to the key's byte representation (required for first 'extract' step) after importing it.
This should be fine as the input key material is already a cryptographically strong key
(see section 3.3 of RFC 5869).

## Choice of primitives

AES-GCM and SHA256 have been chosen,
because [both are hardware accelerated](https://en.wikichip.org/wiki/arm/armv8#ARMv8_Extensions_and_Processor_Features)
on 64-bit ARMv8 CPUs that are used in modern phones.
Our own tests against Java implementations of Blake2s, Blake3 and ChaCha20-Poly1305
have confirmed that these indeed offer worse performance by a few factors.

## Chunk ID calculation

We use a keyed hash instead of a normal hash for calculating the chunk ID
to not leak the file content via the public hash.
Using HMAC-SHA256 directly with the master key in Android's key store
resulted in terrible throughput of around 4 MB/sec.
Java implementations of Blake2s and Blake3 performed better,
but by far the best performance gave HMAC-SHA256
with a key we can hold the byte representation for in memory.

Therefore, we suggest to derive a dedicated key for chunk ID calculation from the master key
and keep it in memory for as long as we need it.
If an attacker is able to read our memory,
they have access to the entire device anyway
and there's no point anymore in protecting content indicators such as chunk hashes.

To derive the chunk ID calculation key,
we use HKDF with the UTF-8 byte representation of "Chunk ID calculation" as info input.

## Stream Encryption

Each file/chunk written to backup storage will be encrypted with a fresh key
to prevent issues with nonce/IV re-use of a single key.
The fresh stream key is randomly generated and encrypted with a key-wrapping key
according to [RFC 5649](https://tools.ietf.org/html/rfc5649)
using an 8 byte IV.
The key-wrapping key is derived from the master key by using HKDF with HMAC-SHA256
like the chunk ID calculation key above.
Here, the UTF-8 byte representation of "stream key wrapping" is used as info input.

When a stream is written to backup storage,
it starts with a header consisting of a single byte indicating the backup format version
followed by the encrypted stream key (40 bytes), so we can later re-derive the key for decryption.
After the first 41 header bytes, the ciphertext starts.

Instead of encrypting, authenticating and segmenting a cleartext stream ourselves,
we have chosen to employ the [tink library](https://github.com/google/tink) for that task.
Since it does not allow us to work with imported or derived keys,
we are only using its [AesGcmHkdfStreaming](https://google.github.io/tink/javadoc/tink-android/1.5.0/index.html?com/google/crypto/tink/subtle/AesGcmHkdfStreaming.html)
to delegate encryption and decryption of byte streams.
This follows the OAE2 definition as proposed in the paper
"Online Authenticated-Encryption and its Nonce-Reuse Misuse-Resistance"
([PDF](https://eprint.iacr.org/2015/189.pdf)).

It adds its own 40 byte header consisting of header length (1 byte), salt and nonce prefix.
Then it adds one or more segments, each up to 1 MB in size.
All segments are encrypted with yet another key that is derived by using HKDF
on our stream key with another internal random salt (32 bytes) and associated data as info
([documentation](https://github.com/google/tink/blob/master/docs/WIRE-FORMAT.md#streaming-encryption)).

A possible simplification could be to not use a new random stream key for each stream
(which needs to get wrapped and included in the header),
but to feed the derived key-wrapping key directly into AesGcmHkdfStreaming
and call it stream key instead.
Tink's AesGcmHkdfStreaming does a HKDF with the stream key as info
and a random salt to derive a fresh "file key" for each stream.

When writing files/chunks to backup storage,
the authenticated associated data (AAD) will contain the backup version as the first byte
(to prevent downgrade attacks)
followed by a second type byte depending on the type of file written:

* chunks: `0x00` as type byte and then the chunk ID
* backup archives: `0x01` as type byte and then the backup archive timestamp

The chunk ID and the backup archive timestamp get added
to prevent an attacker from renaming and swapping files/chunks.

# Data structures

## Local caches

### Files cache

This cache is needed to quickly look up if a file has changed and if we have all of its chunks.
It will probably be implemented as a sqlite-based Room database
which has shown promising performance in early tests.

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

The files cache is local only and will not be included in the backup.
After restoring from backup the cache needs to get repopulated on the next backup run
after re-generating the chunks cache.
The URIs of the restored files will most likely differ from the backed up ones.
When the `MediaStore` version changes,
the chunk IDs of all files will need to get recalculated as well.

### Chunks cache

This is used to determine whether we already have a chunk,
to count references to it and also for statistics.

It could be implemented as a (joined?) table in the same database as the files cache.

* chunk ID (probably a hash/mac)
* reference count
* size

If the reference count of a chunk reaches `0`,
we can delete it from storage as it isn't used by a backup anymore.

References are only stored in this local chunks cache.
If the cache is lost (or not available after restoring),
it can be repopulated by inspecting all backup archives
and setting the reference count to the number of backup archives a chunk is referenced from.

When making a backup pass and hitting the files cache,
we need to check that all chunks are still available on storage.

## Remote Files

All types of files written to backup storage have the following format:

    ┏━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━┓
    ┃ version byte ┃ encrypted stream key (40 bytes)  ┃ encrypted payload ┃
    ┗━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━━┛

### Backup Archive

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

TODO determine how this data will be serialized (e.g. JSON, protobuf, BDF)

All backup archives are stored in an `/backups` folder.
The filename is the timeStart timestamp.

### Chunks

The encrypted payload of chunks is just the chunk data itself.
All chunks are stored in one of 256 sub-folders
representing the first byte of the chunk ID encoded as a hex string.
The file name is the chunk ID encoded as a hex string.
This is similar to how git stores its repository objects
and to avoid having to store all chunks in a single directory.

# Out-of-Scope

The following features would be nice to have,
but are considered out-of-scope of the current design for time and budget reasons.

* compression (we initially assume that most files are already sufficiently compressed)
* packing several smaller files into larger combined chunks to improve transfer efficiency
* using a rolling hash to produce chunks in order to increase likelihood of obtaining same chunks
  even if file contents change slightly or shift
* external secret-less corruption checks that would use checksums over encrypted data
* supporting different backup clients backing up to the same storage
* concealing file sizes

## Idea for avoiding too many small chunks

Transferring many very small files causes a substantial overhead
when transferring them to the storage medium.
It would be nice to avoid that.
Michael Rogers proposed the following idea to address this.

A chunk can either be part of a large file, all of a medium-sized file,
or a (deterministic) zip containing multiple small files.
When creating a backup, we sort the files in the small category by last modification
and pack as many files into each chunk as we can.
Each small file will be stored in the zip chunk under some artificial name
that is unique within the scope of the zip chunk (like a counter).
The path to unique name mapping will be stored in the backup archive (zipRef integer?).
If a small file is inside a zip chunk,
that chunk ID will be listed as the only chunk of the file in the backup metadata
and likewise for any other files inside that chunk.

When creating the next backup, if none of the small files have changed,
we just increase the ref count on the existing chunk.
If some of them have changed, they will be added to a new zip chunk
together with other new/changed small files.

When fetching a chunk for restore, we know in advance whether it is a zip chunk,
because the file we need it for has a counter to path mapping (zipRef),
so we will not confuse it with a medium-sized zip file.
Then we unzip the zip chunk and extract the file by counter/path.

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
