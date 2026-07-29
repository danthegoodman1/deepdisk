# DeepDisk v1 Specification

DeepDisk is a Linux block device backed by local NVMe and an S3-compatible
object store. Local NVMe is the write durability boundary. S3 is an
asynchronous, space-bounded backing store.

This specification intentionally describes a small system. DeepDisk is not a
filesystem, a distributed database, or a snapshot service. Its job is to make
one local block device larger than its local media while preserving local
performance for the working set.

`format_version` is 1.

## 1. Product contract

DeepDisk has five required properties.

1. Writes complete at local NVMe speed and never wait for S3; `FLUSH` and
   `FUA` make them power-loss durable on that NVMe.
2. A locally cached read is served from NVMe without an S3 request.
3. Local and remote space have explicit quotas.
4. When remote writeback cannot keep up, local writes are slowed and then
   blocked before durable journal space is exhausted.
5. Performance, pressure, recovery, and correctness are observable.

The first version does not provide:

- snapshots, forks, rollback, or historical roots;
- multiple writers or independently attached read-only views;
- transparent failover between local caches;
- block-data compression, encryption, deduplication, or content addressing;
- online grow or shrink;
- atomic multi-block writes beyond normal Linux block semantics;
- durable `DISCARD` or `WRITE_ZEROES`.

A volume has a fixed size, one remote prefix, one current manifest checkpoint,
a bounded sequence of later manifest deltas, and at most one writer.

## 2. Guarantees and failure model

### 2.1 Local durability

DeepDisk advertises a volatile write cache and implements normal Linux block
semantics. An ordinary `WRITE` completes after its journal write completes on
the local NVMe. It does not issue a flush and may be lost in a host power
failure.

`FLUSH` waits for all preceding journal writes and then completes one NVMe
flush. `FUA` makes that write and every ordered predecessor stable before
completion. `PREFLUSH` establishes the same boundary before its write. None of
these operations waits for S3.

After power loss, recovery may expose any completely written ordinary batch,
including one whose completion was not observed. It must preserve every write
ordered before a completed `FLUSH` or carrying completed `FUA`. This is the
same durability contract as a local block device with a volatile write cache.

An operator that requires every completed write to survive power loss must use
a verified power-loss-protected NVMe with its volatile cache disabled, or must
configure the host workload to issue `FUA` or `FLUSH`.

### 2.2 Remote durability

The remote head records a completed journal batch sequence through which its
manifest deltas and data segments are complete. A control-plane `remote-sync`
captures a local batch sequence and waits until the remote head covers it.

Apart from ordinary writes explicitly allowed to be lost by the local volatile
write-cache contract, loss of the cache NVMe can lose every write after the
remote head's covered sequence. An intact cache remains the authority for that
local suffix.

S3 writeback is crash-consistent at a commit boundary. Recovery sees a
contiguous committed delta sequence after one complete checkpoint, never a
partially published commit.

### 2.3 Local recovery without S3

The local journal is self-describing. DeepDisk can recover it and expose the
device without contacting S3.

In offline recovery mode:

- journal hits are readable;
- persistent clean-cache hits are readable when they validate against the
  retained local manifest;
- new writes are accepted until journal pressure blocks them;
- a block absent from both local tiers fails with `EIO`;
- remote writeback and remote maintenance are disabled.

This is a partial local view, not a promise that an arbitrary filesystem can
mount without all of its metadata being cached. The `export-journal` command
can emit every locally recoverable block and its sequence without mounting a
filesystem.

### 2.4 Failure outcomes

| Failure | Result |
|---|---|
| daemon crash | recover local journal; clean cache is validated lazily |
| host power loss | preserve flushed/FUA writes; recover any additional valid batches |
| S3 outage | local hits and writes continue until journal pressure |
| cache NVMe loss | restart from remote head; lose its later local suffix |
| local RocksDB loss | restore the checkpoint named by the remote head |
| corrupt clean block | treat as a cache miss |
| corrupt dirty journal block | fail the affected read and stop writeback |
| corrupt remote object | fail the read; never return unchecked bytes |
| corrupt checkpoint blob | fail restore; never open a partial manifest |
| ambiguous S3 mutation | resolve by reading `meta/head` and its pending operation |

DeepDisk never falls back to an older remote value when the latest acknowledged
local value is known to be corrupt. That would silently acknowledge data loss.

## 3. Architecture

```text
                         asynchronous
                    +--------------------+
                    |                    v
filesystem -> ublk -> local NVMe       S3 prefix
                    |                    |
                    | journal            | immutable data segments
                    | clean cache        | committed manifest deltas
                    | RocksDB manifest   | periodic RocksDB checkpoints
                    +--------------------+
```

The local and remote designs are deliberately different.

- The journal is append-only because host writes require ordered durability.
- The clean cache uses deterministic placement because clean data is
  disposable and does not justify a complete RAM index.
- The logical block map is a conventional RocksDB on local NVMe. It is backed
  up incrementally to S3; RocksDB never runs on object storage.
- Ordinary remote publication appends one immutable redo delta and advances
  one conditional head. Periodic physical RocksDB checkpoints bound replay.

There is no copy-on-write radix tree, checkpoint overlay, per-cache-block RAM
map, global eviction queue, product snapshot history, remote LSM, or
concurrent remote collector.

## 4. Linux device behavior

DeepDisk attaches through `ublk` and uses `io_uring` with registered buffers.
Linux 6.1 or newer is required. A supported release is a tested kernel build,
not merely any kernel whose version compares greater than 6.1.

The cache device must correctly implement flush and FUA. DeepDisk records its
write-cache mode and flush/FUA behavior at format time and verifies them at
attach. Power-loss-protected enterprise NVMe is strongly recommended; a device
whose flush is slow remains correct but may not satisfy the performance goal.
NVMe protection information is optional.

The device exposes 4096-byte logical and physical blocks. Requests must be
aligned to 4096 bytes.

Supported operations are:

- `READ`;
- `WRITE`;
- `FLUSH`;
- `FUA` and `PREFLUSH`;
- `DISCARD` as a non-durable hint.

`DISCARD` may invalidate clean cache entries but does not promise zeroes and
does not reclaim remote space. DeepDisk reports `discard_zeroes_data = 0`.
`WRITE_ZEROES` is not advertised; callers must issue ordinary zero writes.

The presented size is fixed at volume creation. The maximum is `2^48` blocks,
or 1 EiB at 4 KiB, though practical object and manifest limits are normally
reached first.

One worker owns one volume, one raw cache device, and one manifest-quota
directory. The raw device is opened exclusively and the manifest directory is
process-locked. The supervisor is outside the data path and may restart the
worker using ublk user recovery.

Default request resources are:

```text
ublk queues            min(online CPUs, 2)
queue depth            32
maximum request        128 KiB
registered buffers       8 MiB
```

When registered buffers are exhausted, ublk queues wait. They do not allocate
from the general heap on the I/O path.

## 5. Local NVMe layout

DeepDisk uses two explicitly bounded local resources:

1. a raw NVMe partition or logical volume for the journal and clean cache;
2. a quota-limited directory on local NVMe for the RocksDB manifest.

They may be separate partitions of one physical device. `cache_bytes` and
`manifest_disk_bytes` are independent hard limits. The manifest directory
must be on ext4 or XFS with project quota enabled; attach rejects an
unenforceable quota.

The formatted space contains:

1. two alternating superblocks and small control logs;
2. a circular durable write journal;
3. a persistent clean data cache and its tag pages;
4. maintenance scratch and emergency journal reserve.

Defaults are:

```text
region_bytes             64 MiB
journal_bytes            min(max(5% of cache_bytes, 8 GiB), 16 GiB)
maintenance_bytes        min(max(0.5% of cache_bytes, 1 GiB), 4 GiB)
journal_reserve_regions  2
clean_bytes              all remaining formatted space
manifest_disk_bytes      required explicit quota, at least 8 GiB
```

Formatting fails if these partitions and reserves do not fit. Their byte
counts never change implicitly after attach. `clean_bytes` includes both data
slots and both on-disk tag-page copies; the formatter chooses the greatest
whole number of 16-way sets that fits that partition.

The manifest quota includes the writable RocksDB, its retained checkpoint, one
pending periodic checkpoint, and compaction scratch. Before flush, compaction,
or checkpoint creation, DeepDisk reserves a conservative bound for every new
local file while leaving an emergency reserve of
`max(1 GiB, 5% of manifest_disk_bytes)`. If that bound does not fit, host
writes encounter the ordinary journal backpressure path.

The writable database and retained and pending checkpoint directories must be
on the same quota-enabled filesystem and in the same quota project. This is
required for RocksDB checkpoint hard links to remain cheap and for every
linked byte to be charged to the same hard limit.

This is the deliberate cost of not using an object-store-native index: local
manifest disk grows with mapping fragmentation. Sequentially written data is
represented by extents and is cheap. In the worst case of unrelated 4 KiB
writes, operators should budget roughly 64 bytes per mapped block plus 100%
temporary headroom, or about 3.1% of mapped logical data. Actual RocksDB live,
checkpoint, and temporary bytes are reported separately. DeepDisk estimates
remaining worst-case mappings and never silently consumes clean-cache space.
The quota is therefore not derived from clean-cache capacity. Format reports
both the fully fragmented requirement for `device_bytes` and the smaller
operator-selected limit; exhaustion throttles writes rather than expanding it.

The superblock stores:

- format version and volume UUID;
- cache UUID;
- presented size;
- all local partition offsets and lengths;
- manifest-directory UUID, quota, base checkpoint catalog and commit;
- locally applied remote commit and installed head revision;
- journal oldest/newest sequence hints;
- the greatest locally recorded remote-covered batch;
- clean-cache hash seed;
- a checksum and superblock generation.

The two superblock copies alternate. Recovery selects the valid copy with the
highest generation. A superblock update is complete only after its FUA write
completes.

## 6. Local write journal

### 6.1 Ordered local writes

All host writes enter one ordered journal sequencer. The worker copies each
write once from its registered ublk buffer into one of two aligned 4 MiB
journal buffers; it may fill one while the other is submitted. At most two
ordinary batch writes are in flight, and mapping publication and host
completion remain in `batch_seq` order. Under light load there is no batching
delay; under sustained load the sequencer may coalesce already queued requests
for at most:

```text
batch_bytes          4 MiB encoded maximum
busy_batch_delay    50 us
```

When dequeued, `FLUSH` seals the preceding batch. An `FUA` write is appended
and then seals its batch. `PREFLUSH` seals the batch before its write, waits
for that batch and an NVMe flush, and only then starts the write's batch. A
combined `PREFLUSH|FUA` request therefore has one flush before its data and one
after.

For an ordinary batch the worker:

1. reserves journal, footer-index, and batch-buffer capacity;
2. assigns the next `batch_seq`;
3. writes the complete aligned batch to the journal;
4. waits for its data write;
5. publishes its active-region mappings;
6. completes the host requests in submission order.

Resource reservation happens before sequence assignment. A request that waits
for space consumes no sequence number.

`FLUSH` waits for all submitted batches and issues one NVMe flush. An `FUA`
batch is not completed until its data and an NVMe flush are complete. Journal
footers, RocksDB, clean-cache metadata, and S3 are absent from the host
completion path.

`FLUSH`, `PREFLUSH`, and `FUA` are sequencer barriers. No later journal batch
is submitted across one until its required flush completes.

### 6.2 Batch format

A batch is aligned to 4096 bytes and never crosses a journal region.

```text
header
  magic, format version
  volume UUID, cache UUID
  region generation
  batch_seq
  record count and encoded length
  header CRC32C

records
  block_index u64
  data_offset u32
  data_crc32c u32

data
  one or more 4096-byte blocks

trailer
  batch_seq and encoded length
  CRC32C over the header, records, and data
```

A multi-block host request occupies consecutive record ordinals in one batch.
Recovery accepts the request only when the entire batch validates.
Every `data_offset` is relative to its 64 MiB region, and every data block
begins at a 4096-byte-aligned journal offset.

The CRC domains include the volume UUID, cache UUID, batch sequence, logical
block index, and data. A valid block cannot be redirected to another logical
address without detection.

### 6.3 Journal regions

The journal is a circular sequence of 64 MiB regions. Each reserves its final
512 KiB for a disposable lookup footer. A region has two alternating headers
containing its physical number, reuse generation, first batch sequence, last
batch sequence, and checksum.

Batches append sequentially. A batch that does not fit closes the region and
starts in a free region. Region reuse first writes and persists the new
generation header.

A normal region is reusable only when its last batch is covered by the remote
head, local RocksDB has applied the commit that established that coverage, and
the region has no read or writeback pin. DeepDisk intentionally does not
reclaim holes created by overwrites inside a newer, uncommitted region.
Journal capacity therefore measures the physical write rate during an outage,
not unique dirty bytes.

Two reserve regions are unavailable to ordinary admission. They provide a
fixed safety margin for the active partial region, boundary padding, and
administrative recovery. Correctness never depends on admitting new ordinary
host data into them.

### 6.4 Region indexes and Bloom directory

There is no global exact RAM map of dirty blocks. The active region has a small
RAM hash containing the newest record for each block in that region. At region
close, DeepDisk writes a footer containing:

```text
footer header
  volume UUID, cache UUID
  physical region and reuse generation
  first and last batch sequence
  unique block count

page directory
  first block index and offset of each sorted index page

sorted entries
  block_index u64
  data_offset u32
  data_crc32c u32

Bloom filter
  16 bits per possible data record

footer CRC32C
```

Only the newest occurrence of a block within that region appears in the
footer. The page directory and one exact index page are sufficient to resolve
a positive Bloom result. Footer writes are sequential, asynchronous, and not a
durability boundary; recovery rebuilds a missing or invalid footer from the
self-describing batches.

Even if every usable record is unique, 16-byte entries, the Bloom filter, page
directory, and headers occupy less than 320 KiB; the fixed 512 KiB footer
reserve is therefore a checked format invariant.

The active region and at most four closed regions awaiting footer writes keep
their exact hashes in RAM. If that bounded footer pipeline fills, journal
admission waits at the next region boundary.

Closed-region Bloom filters form a bit-sliced RAM directory. A lookup computes
the Bloom positions once and intersects a bit vector of regions for each
position, yielding candidate regions in newest-first order. At 16 bits per
4 KiB record, a full 16 GiB journal needs about 8 MiB for this directory.
The small footer page directories are retained in the same RAM budget. False
positives cost one local footer-page read; false negatives are forbidden.

### 6.5 Runtime reads and staging

A bounded staging table contains writes admitted to the filling or submitted
batches but not yet published in the active-region index. It is ordered by
`batch_seq` and record ordinal. A read checks, in order:

1. staging, newest first;
2. active and footer-pending exact hashes, newest first;
3. the bit-sliced Bloom directory and exact footer pages, newest first.

The first exact match is the newest journal value. Therefore a read cannot
return an older remote or clean value while a newer local write exists.

Staging capacity is bounded by the registered request buffers. It has no
independent growth path.

### 6.6 Recovery

Recovery:

1. selects the newest valid superblock;
2. orders regions by generation and sequence range;
3. scans resident batches in global sequence order;
4. requires a contiguous sequence above locally recorded remote coverage and
   stops at the first missing, invalid, or incomplete expected batch;
5. rebuilds missing footers, active hashes, and the Bloom directory;
6. verifies every journal block that may be newer than local remote coverage;
7. exposes the device.

Journal recovery itself needs no S3 request. It scans at most `journal_bytes`
and makes no fixed-time promise. Normal online attach may subsequently contact
S3 to validate ownership and update the local manifest; offline recovery does
not.

A complete ordinary batch may be retained even if its host completion was lost
or it was never explicitly flushed. The invalid batch and every higher
sequence are ignored. NVMe submission ordering and the `FLUSH`/`FUA` protocol
ensure that no completed durability boundary can exist above an invalid
earlier batch.

## 7. Clean read cache

### 7.1 Deterministic placement

Clean data does not have a global RAM map. The clean partition is divided into
16-way sets:

```text
set = keyed_hash(volume_uuid, cache_hash_seed, block_index) % set_count
slot = set * 16 + way
```

The hash is keyed so a guest cannot deliberately direct chosen logical blocks
to one set without knowing the local seed.

Each on-disk tag is 32 bytes:

```text
block_index       u64
mapping_tag       u128
data_crc32c       u32
state             u32
```

`mapping_tag` is the first 128 bits of SHA-256 over the volume UUID, logical
block, segment ID, and slot in the current manifest. State includes validity,
a small re-reference score, and an entry generation. Tags are kept in fixed
4 KiB pages. Two copies of each tag page alternate; each copy has a page
generation and CRC. The copies are adjacent and may be loaded with one 8 KiB
metadata read.

`data_crc32c` covers the volume UUID, logical block index, and 4096 data bytes.
The mapping tag detects a stale but intact cache entry; the CRC detects
misdirected or corrupt local data.
The tag-page CRC also covers the volume UUID, cache UUID, physical tag-page
number, page generation, and all tags, so a valid page cannot be redirected.

At 32 bytes per 4 KiB data slot, two tag copies consume about 1.56% of clean
capacity. This cost is on disk, not in compulsory RAM.

### 7.2 Crash-safe disposable entries

The current local manifest, not a cache tag, decides which remote version of a
block is current. A tag is accepted only when its `mapping_tag` matches that
manifest mapping and its address-bound data CRC validates.

To fill a way, the cache worker:

1. writes the data slot;
2. waits for the data write to complete;
3. writes a new alternate tag-page copy;
4. publishes the in-memory entry after revalidating the manifest generation.

Tag writes are asynchronous, batched, and do not issue an NVMe flush or FUA.
They are never part of host-write completion, remote publication, or journal
reclamation.

After a crash, an old tag and new data, a new tag and old data, a torn tag
page, or a tag for an old remote mapping all fail validation and become a
miss. A valid surviving pair remains useful. Because the clean cache is never
authoritative, no ordering stronger than completed individual writes is
required.

### 7.3 RAM tag cache

Decoded tag pages use a fixed RAM LRU:

```text
clean_tag_ram_bytes  4 MiB
```

After the manifest lookup, a tag-RAM hit needs one 4 KiB NVMe data read. A
tag-RAM miss reads the adjacent tag-page pair and then the selected data
block. A cold manifest lookup may add RocksDB I/O. These local metadata reads
are the explicit price of bounded RAM and durable cache coherence without
per-write invalidations.

Increasing `clean_bytes` does not increase required RAM. Administrators may
increase the tag-page cache as a performance option.

An asynchronous clean read registers a sparse in-flight pin containing its
manifest generation, set, way, and entry generation, then revalidates the tag
before returning data. Eviction may not overwrite a pinned way. The pin table
is bounded by ublk queue depth, not clean-cache capacity. Journal reads and
remote writeback similarly pin a journal region until their NVMe reads finish;
region reuse waits for zero pins.

### 7.4 Admission and eviction

Replacement is local to a 16-way set. Each tag has a two-bit RRIP-style
re-reference score. There is no global queue and no clean-data relocation.
Hits update that score in the decoded RAM page. The score is persisted only
when that tag page is already rewritten for admission or eviction; a cache hit
never schedules a metadata write merely to preserve replacement state.

A fixed 1 MiB TinyLFU sketch estimates recent frequency and decays
periodically. Admission rules are:

- use an invalid way immediately;
- do not admit a detected large sequential stream on its first pass;
- otherwise admit only when the incoming estimate is at least the selected
  victim's estimate;
- never wait for clean-cache space.

Journal-resident data already provides read-after-write locality. When a
covered journal region is reclaimed, a latest block is promoted to the clean
cache only if it was read recently or admission finds free space. Otherwise
any old cache entry is left in place and will fail its next mapping-tag check.

Clean cache writes, tag updates, and admission are lower priority than host
writes, journal recovery, and remote progress.

### 7.5 Read lookup

The read path is:

1. capture the current manifest generation;
2. check journal staging;
3. search the active and closed journal-region indexes;
4. seek the current local RocksDB extent;
5. return zeroes if no extent covers the block;
6. check the deterministic clean-cache set for the exact mapping tag;
7. range-read the mapped remote segment on a cache miss.

Every returned nonzero block is verified against a checksum bound to its
logical block index. A clean checksum failure invalidates the way and continues
as a miss. A journal checksum failure stops at that tier and returns `EIO`.

Clean fills revalidate the manifest generation and repeat the staging and
journal lookup before publishing. A read retries its manifest lookup if the
generation changed. A remote read racing a local write can never replace the
newer local value. Remote objects selected by an in-flight read remain pinned
until that read drains.

## 8. Backpressure

`journal_used_bytes` counts occupied regions, including padding and overwritten
records. It is the primary pressure signal.

Defaults are:

```text
drain watermark       70% of usable journal
pace watermark        85%
stall watermark       95%
hard limit            journal minus two reserve regions
```

Behavior is:

- below 70%, remote writeback follows its normal age and batch targets;
- at 70%, writeback runs continuously;
- at 85%, the durability sequencer delays admission toward the measured remote
  committed-byte rate;
- at 95%, new writes wait for a reusable region;
- at the footer-pipeline or local allocation-reserve limit, new writes wait
  regardless of byte percentage;
- at the retained-delta limit, remote commits pause for a required checkpoint
  while host writes continue under the ordinary journal watermarks.

The measured rate is an EWMA of data bytes covered by a successfully installed
remote commit, not bytes merely uploaded. If that rate is zero, pacing
converges toward zero and the hard stall eventually applies.

There is no default write timeout. A blocked write remains pending until
progress, detach, or administrator cancellation. Reads that require S3 use a
configurable deadline and return `EIO` after it expires.

RocksDB apply or checkpoint work, delta-budget exhaustion, segment repacking,
quota pressure, expired credentials, throttled bandwidth, S3 errors, and
network loss all feed the same pressure mechanism. No special failure path is
allowed to bypass the journal limit.

The journal sequencer always owns reserved NVMe queue capacity. All other local
I/O passes through one bounded background scheduler. Clean fills and optional
checkpoint work stop in `pace`; commit-critical segment streaming,
manifest-delta application, and required compaction retain a capped but
nonzero share so writeback cannot deadlock. A checkpoint becomes critical only
when the retained-delta limit would otherwise block commits. As pressure
rises, host admission is paced to free that share. Lower OS I/O priority is
used where supported but is not trusted for correctness. A deployment whose
single NVMe cannot preserve the latency gate under background load must place
the manifest directory on a separate local NVMe.

## 9. RAM budget

The default userspace budget is independent of clean-cache capacity.

```text
registered request buffers        8 MiB
journal batch buffers             8 MiB
journal Bloom and active indexes 10 MiB
clean tag-page cache              4 MiB
RocksDB block cache               8 MiB
RocksDB total write buffers       4 MiB
TinyLFU and sequential detector   1 MiB
remote build, hashing, clean fill 8 MiB
control, native, and headroom     21 MiB
                                 -------
default process target           72 MiB
```

The default process target is 72 MiB and its cgroup `memory.max` is 96 MiB.
The 21 MiB target headroom includes native-library allocations, allocator
slack, and thread stacks. Attach prints every arena and refuses to start if
configured bounds exceed the limit. Release testing, not configured cache
totals, is the authority for the RSS bound.

The Bloom directory uses two bytes per possible 4 KiB record. Footer page
directories add about 256 KiB at the maximum journal size. Active and
footer-pending indexes use a preallocated packed open-addressed table capped at
20 bytes per occupied entry; a general-purpose hash map is forbidden here. A
16 GiB journal, one active region, and four completely full footer-pending
regions therefore fit in the 10 MiB default. Increasing journal capacity
reports and reserves the corresponding bytes before attach; clean-cache and
presented-volume sizes do not change required RAM.

RocksDB is configured with partitioned indexes and filters charged to its
8 MiB block cache, no mmap, direct I/O for SST reads, flushes, and compaction,
at most 4 MiB of memtables, and bounded background concurrency.
The engine may not pin all indexes or filters in RAM.

The kernel's ublk request structures and the process's TLS and code pages are
reported separately. DeepDisk exports RSS, locked bytes, and every arena's
committed and peak bytes.

## 10. Remote object store

### 10.1 Required semantics

The backend must provide:

- strongly consistent `GET`, `HEAD`, `PUT`, `DELETE`, and `LIST`;
- byte-range `GET`;
- conditional replacement with exact ETag comparison;
- request and response checksums for uploaded objects;
- atomic visibility of a completed single-object `PUT`.

Bucket versioning, object lock, and retention policies must be disabled for the
volume prefix. No other writer may create objects in the prefix.

DeepDisk uses single-object PUTs only. An implementation may add multipart
upload in a later format after specifying incomplete-part accounting.

### 10.2 Keys

```text
meta/super                   immutable volume configuration
meta/head                    the only mutable remote object
commit/<20-digit-number>     immutable commit descriptor and manifest delta
gc/<id>                      immutable bounded deletion descriptor
manifest/catalog/<id>        immutable RocksDB checkpoint catalog
manifest/blob/<sha256>       immutable checkpoint file content
segment/<segment-id>         immutable data segment
```

Catalog, GC, and segment IDs are random 128-bit values. Commit numbers are
contiguous unsigned 64-bit integers rendered at fixed width. Segment data is
not content-addressed. Checkpoint files use their SHA-256 as an object key so
unchanged hard-linked SSTs upload only once. An existing blob is reused only
after its length and checksum validate.

Every immutable key is created with `If-None-Match: *`. An existing
content-addressed blob may be reused only after validation; an existing random
ID is a collision or stale-writer error and is never overwritten.

`meta/super` fixes the volume UUID, block and device sizes, remote quota,
segment and commit formats, manifest schema, and compatible RocksDB format
set. It is created with `If-None-Match: *` and never replaced.

No old checkpoint is a supported user-visible state. The live recovery chain
is exactly the base catalog named by `meta/head` plus every contiguous commit
after `base_commit` through `last_commit`. Objects outside that chain or a
pending operation are garbage.

### 10.3 Data segments

The remote writer packs journal blocks into immutable segments, 32 MiB of data
by default. A segment contains:

- a checksummed header with volume UUID and segment ID;
- fixed-size slot records, each containing segment ID, slot number, logical
  block index, flags, data CRC, record CRC, and 4096 bytes of data;
- a checksum over the complete object.

An extent value identifies `(segment_id, first_slot, block_count)`. The data
offset is computable from the slot. The slot-record CRC domain includes the
volume UUID, segment ID, slot number, logical block index, flags, data CRC, and
data. A read uses one slot-sized range GET after resolving the extent and
verifies the complete slot record before returning its 4096 data bytes.

A segment may contain unrelated logical blocks. The writer produces at most
one underfilled segment per ordinary commit.

### 10.4 Local RocksDB manifest

The operational manifest is one ordinary RocksDB database on local NVMe.
DeepDisk uses a pinned Rust RocksDB binding and pins one exact RocksDB release
in the build and remote format metadata. It does not implement an LSM
table format, compaction policy, WAL, Bloom filter, or block cache.

The database is writable and records one locally applied remote commit. It is
never ahead of `meta/head`. Between a successful remote commit CAS and local
application it may be one commit behind. For ordinary writeback, every changed
logical block remains in the higher-priority journal. A repacking commit does
not change logical bytes, and its old segments remain available until local
application and read-pin drain.

There are three column families:

```text
extents
  key:   first logical block, big-endian u64
  value: block_count u32, segment_id u128, first_slot u32

segments
  key:   segment_id u128
  value: total_slots, live_slots, object_bytes, creation_commit

control
  key:   singleton names
  value: volume UUID, applied commit, last observed head revision
```

The extent keyspace is canonical and non-overlapping. Consecutive logical
blocks stored in consecutive slots of one segment are one extent. A point
lookup performs `SeekForPrev(block_index)` and checks that the returned extent
covers the requested block. No result means the block has never been written
and reads as zero.

Each committed remote delta contains sorted, coalesced mapping runs. The local
applier edits overlapping extents in one RocksDB `WriteBatch`: an overwritten
extent is deleted and any unaffected left or right remainder is reinserted.
New and surviving neighbors are merged whenever they name consecutive slots
of the same segment. The same batch adjusts exact per-segment live-slot counts
and advances the applied commit and head revision. The batch uses a synced
RocksDB WAL before DeepDisk records local coverage or reclaims journal data.
Only this serialized applier writes the database, so no database transaction
layer is needed.

A process-local manifest generation increments after each applied batch. Reads
that observe a generation change repeat their mapping lookup. RocksDB
compaction changes physical files but not this logical generation.

The engine configuration is fixed by `format_version`:

- leveled compaction and 256 MiB target SST files;
- LZ4 compression for manifest data only;
- an 8 MiB block cache;
- partitioned indexes and Bloom filters charged to that cache;
- maintenance scans and compaction use `fill_cache = false`;
- at most 4 MiB of total write buffers;
- WAL enabled, with each committed-delta batch synced before journal reclaim;
- no mmap and no unbounded pinned index or filter set;
- at most 128 open RocksDB files;
- direct I/O for SST reads, flushes, and compaction, with attach failure rather
  than a buffered-I/O fallback;
- at most one flush and one compaction job;
- at most 1 GiB of input plus output in one compaction.

DeepDisk does not start a flush or compaction whose declared output bound
cannot fit local manifest headroom. If a committed delta cannot be applied
within the quota, its journal data is retained and host writes enter ordinary
pressure. The remote commit remains authoritative and the database can be
restored and replayed after space is added.

Small RocksDB control and WAL files may use buffered filesystem I/O. Their page
cache is charged to the process cgroup and remains subject to the same 96 MiB
limit.

RocksDB is chosen over Fjall for v1 because RocksDB has a supported online
physical-checkpoint API. Fjall may replace it only in a future format after it
provides an equivalent consistent checkpoint/export contract. Supporting two
engines in one format is forbidden.

A binary that does not recognize the recorded RocksDB build and table-format
set refuses read-write attach. Engine upgrades require an explicit offline
migration and restore verification; changing the crate version is not a
transparent format change.

After an unclean shutdown, the database is trusted only when its control
record, RocksDB integrity checks, and local superblock agree. If its applied
commit is behind the remote head, DeepDisk replays the missing contiguous
commit objects. If it is ahead, attach stops as corruption. If it is invalid,
DeepDisk restores the base checkpoint and replays. The journal remains the
authority for later local writes.

### 10.5 Periodic manifest checkpoints

Ordinary remote commits do not create RocksDB checkpoints. The recovery chain
is bounded by:

```text
maximum retained commit objects       4096
maximum retained commit-object bytes  1 GiB
```

Before either limit would be exceeded, ordinary remote commits pause and
DeepDisk publishes a checkpoint. Local host writes continue into the journal.
A checkpoint may also be requested manually.

Checkpoint creation requires a normal head with no pending operation and a local
database applied through `last_commit = N`. DeepDisk force-flushes every column
family and asks RocksDB for a physical checkpoint. RocksDB creates a
consistent directory by hard-linking immutable SSTs and writing the required
metadata files.

Every regular file receives a SHA-256. An SST hard-linked to the retained
checkpoint has the same inode and immutable contents, so DeepDisk reuses the
hash already recorded for that inode. A new inode is streamed and hashed once;
path and length alone are never accepted as identity. Small mutable metadata
files are always rehashed. A checkpoint catalog contains:

```text
catalog format and volume UUID
RocksDB build and format identifiers
base commit N and expected prior base
covered local batch sequence
relative file path, length, and SHA-256 for every file
catalog checksum
```

Paths must be relative, normalized, unique, and contain no symlinks. The
catalog itself is immutable. File contents are uploaded under
`manifest/blob/<sha256>`; existing valid blobs are reused. This is an
incremental backup because unchanged SST hard links retain the same content.
DeepDisk interprets none of the files.

Catalog file records are sorted by relative path. If they exceed 8 MiB, they
are split into content-addressed pages of at most 8 MiB under
`manifest/blob/<sha256>`; the top-level catalog contains only page hashes,
lengths, and checksums. This is a paged file list, not a searchable remote
manifest. The top-level catalog is also limited to 8 MiB. Creation and restore
stream catalog pages through bounded buffers and store them on manifest disk;
the full file list is never retained in RAM. Restore validates all of it before
opening RocksDB.

Checkpoint publication is:

1. Build and hash the local checkpoint and its exact catalog.
2. CAS the head from `mode = normal` with no pending operation to
   `mode = maintenance` with the pending catalog, reserving its catalog and
   missing-blob bytes.
3. Put the catalog with `If-None-Match: *`, then upload every missing
   checkpoint blob.
4. Validate the catalog and every blob by exact key, length, and checksum.
5. CAS the head back to `mode = normal` with no pending operation, install the
   new catalog with `base_commit = N`, leave `last_commit = N`, account new
   bytes, reset retained commit count and bytes to zero, and release unused
   reservation.
6. Retain the new checkpoint locally and make the old base catalog and commit
   objects through N eligible for garbage collection.

The old base checkpoint and all absorbed commit objects remain live until step
5 succeeds. A lost CAS response is resolved by reading the head. An aborted
checkpoint leaves the old recovery chain unchanged and deletes or conservatively
accounts every newly created object before another attempt.

One unopened immutable copy of the base checkpoint named by the remote head is
retained locally. One pending checkpoint may coexist until publication
resolves. Their hard-linked SSTs share physical blocks, and all directories
count against `manifest_disk_bytes`.

On attach, a pending checkpoint is resolved before ordinary commits. If its
catalog and every blob validate, recovery completes the base-install CAS. If
objects are missing and the matching local checkpoint directory survives,
recovery resumes their upload. Otherwise it deletes or conservatively accounts
the pending objects and restores the old normal head. The old base-and-delta
chain remains live throughout.

Recovery onto a new cache downloads the current catalog and all named blobs,
verifies them, recreates the database, and then fetches
`commit/(base_commit + 1)` through `commit/last_commit` by deterministic key.
Every object must validate its predecessor, volume, mappings, and checksum
before its synced `WriteBatch` is applied. Remote reads are unavailable until
this bounded replay finishes. No `LIST` is used for restore.

Restore preflights the catalog's exact file bytes against the local manifest
quota, retained-delta replay bound, and emergency reserve before downloading
anything. If it does not fit, attach fails with the required byte count;
DeepDisk does not partially open or silently borrow clean-cache space.

These physical checkpoints are an internal backup mechanism. They do not
provide user-visible snapshots, rollback, forks, or historical reads.

## 11. Remote reads

After the current RocksDB mapping and clean cache both miss, the worker:

1. range-reads the mapped segment slot;
2. verifies the segment identity, logical block index, slot, and data CRC;
3. returns the block;
4. offers it to clean-cache admission with the captured mapping tag.

Admission copies the block into the bounded background arena before the host
request buffer is released. If that arena or its local-I/O budget is
unavailable, admission is skipped; returning a remote read never waits for
clean-cache persistence.

Concurrent reads for the same missing segment range share one S3 request.
RocksDB lookup I/O is local and bounded by its caches. For a hot working set,
both its metadata block and the clean tag page should be RAM-resident, leaving
one NVMe data read. Cold metadata may add local I/O but never an S3 request.

A retry always repeats validation. S3 returning a successful status with the
wrong bytes is corruption, not a miss.

## 12. Remote writeback

### 12.1 Selecting a cut

The writer selects a whole-batch prefix:

```text
local_remote_covered_seq < cut_seq <= newest_completed_batch_seq
```

It walks journal batches once in sequence order and keeps the last value for
each block in the selected interval. Earlier overwritten values need not be
uploaded.

Ordinary commits target:

```text
commit target data      128 MiB
commit maximum data     256 MiB
commit maximum mappings  65,536
maximum age               10 s
```

A cut never splits a local batch. `remote-sync` may request a smaller final
commit so it does not wait for unrelated later writes.

Every journal region containing a selected record remains pinned until the
commit attempt finishes or aborts.

Ordinary segment objects are not assembled in RAM or copied into maintenance
scratch. The commit descriptor fixes their record order and headers, then
streams the pinned journal records once to compute exact lengths and checksums.
After reservation, the uploader deterministically streams the same records
again. Retries repeat that stream. Two uploads use bounded 1 MiB chunks while
the commit descriptor remains in the 8 MiB remote-work arena; no
segment-sized buffer is allocated.

Both passes verify the address-bound journal CRC before using a block. A
mismatch stops writeback and preserves the journal evidence; DeepDisk never
rechecksums corrupt local bytes into an apparently valid segment.

### 12.2 Head

`meta/head` contains:

```text
volume UUID and format version
head revision
writer token
base checkpoint catalog ID and base commit
last committed commit and retained commit-object bytes
writer cache UUID and covered batch sequence
mode: normal or maintenance
used remote bytes and reserved remote bytes
pending operation: none, commit number, checkpoint catalog, or GC descriptor
mode to install after success or abort
checksum
```

Every mutation uses exact ETag compare-and-swap. There is at most one pending
operation. An ordinary commit returns to normal, a repacking commit and GC
batch retain maintenance, and a checkpoint returns to normal whether it
finishes or aborts.

### 12.3 Commit object and publication

`commit/N` is both the immutable publication descriptor and the ordered redo
delta needed to advance the manifest from commit `N-1` to `N`. It contains:

- volume UUID, writer token, cache UUID, commit N, and expected commit N-1;
- expected head revision, base checkpoint, and cut sequence;
- every new segment key, slot count, exact length, and checksum;
- ordered logical-block runs assigning each new segment slot;
- the exact remote-byte reservation and commit-object checksum.

The object is limited to 4 MiB. If its encoded mapping runs would exceed that
limit, the worker reduces the cut before writing any remote object. Commit
creation uses `If-None-Match: *`; an existing key must validate as the
identical object or the writer stops with a fencing error. At most one
unclaimed next-commit object may consume the control allowance.

Publication is:

1. Verify that local RocksDB is applied through the head's `last_commit` and
   that the retained-delta limits permit another commit; reserve conservative
   local manifest space for applying its maximum delta.
2. Lay out and hash new data segments from pinned journal records.
3. Write `commit/N`, where `N = last_commit + 1`.
4. CAS the normal head from no pending operation to pending commit N, retaining
   `mode = normal` and reserving the commit object and every new segment byte.
5. Upload every segment named by commit N.
6. Verify all planned object lengths and checksums.
7. CAS the head to clear pending commit N while retaining `mode = normal`,
   advance `last_commit` and the covered batch sequence, add the commit and
   segment bytes to `used_bytes`, update retained commit-object bytes, and
   release unused reservation.
8. Apply commit N to local RocksDB with one synced `WriteBatch`.
9. Increment the manifest generation and persist applied commit, installed
   head revision, and covered batch sequence in one local superblock update.
10. Reclaim eligible journal regions and complete waiting `remote-sync` calls.

Step 4 reserves remote state; only step 7 makes the mapping current. Until step
9, every changed block remains in the higher-priority journal. No later remote
commit begins until local RocksDB has applied N.

Every mapping run in commit N references a segment listed by N; unaffected
extents are inherited implicitly. If the final CAS response is lost, the
worker reads the head. A head with `last_commit = N` and no pending operation
means success; a pending operation naming N is resumed; any other state is a
fencing failure.

A committed object remains part of the recovery chain until a later checkpoint
absorbs it. An aborted or unclaimed object is deleted and its absence confirmed
before the same commit number is retried. Ambiguous deletion blocks new
publication.

### 12.4 Commit recovery

On attach, a pending commit is resolved before new remote work:

- if the commit object and every segment validate, finish the final CAS;
- otherwise resume missing segment uploads when the referenced journal records
  validate;
- or delete every object unique to the pending commit, confirm absence, and
  CAS the head back to the preceding commit in the pending operation's retained
  mode.

Aborting never discards local journal data. If the final CAS succeeded but
local RocksDB was not updated, the worker replays the committed object, syncs
the database, updates the superblock, and only then reclaims journal data.

Because no data segment is uploaded before the pending head reserves and names
its commit, normal upload cannot race garbage collection. If the entire local
cache is lost during a pending commit, recovery finishes it when all remote
objects validate and otherwise aborts it to the preceding commit.

## 13. Remote quota

`remote_limit_bytes` is a required hard limit on the sum of object payload
bytes in the volume prefix, subject to the backend semantics required above.
Provider-internal metadata and allocation overhead are outside this portable
contract.

The limit is hard only inside the DeepDisk ownership contract: every mutation
of the prefix must pass through the current fenced writer. External uploads,
retention charges, bucket versioning, or stale credentials that can bypass
the head CAS violate the backend contract and can exceed the limit.

The head tracks:

```text
used_bytes       segments, committed deltas, and manifest objects not deleted
reserved_bytes   maximum bytes of the one pending operation
```

The fixed 32 MiB control allowance covers `meta/head`, `meta/super`, one
not-yet-accounted commit or GC descriptor, and unused fixed protocol headroom;
no other object may charge to it. A checkpoint catalog is part of its pending
reservation. A commit consumes control allowance before its reservation CAS and
`reserved_bytes` afterward; it is never charged to both in the quota equation.
Define:

```text
payload_limit = remote_limit_bytes - control_allowance
```

For a proposed reservation of `new_bytes`, beginning an operation requires:

```text
used_bytes + new_bytes <= payload_limit
```

Normal data writeback also leaves a maintenance reserve:

```text
maintenance_reserve = max(1 GiB, 5% of remote_limit_bytes), capped at 64 GiB
```

An ordinary commit requires:

```text
used_bytes + new_bytes <= payload_limit - maintenance_reserve
```

The reserve is available only to checkpoint publication, segment repacking,
commit recovery, and garbage collection. Volume creation rejects a limit
too small for the control allowance, reserve, and one maximum commit.

Bytes remain counted after becoming unreachable. They are subtracted only
after deletion is confirmed. A crash may therefore overcount space but must
never undercount it.

Reused checkpoint blobs add zero bytes only when already accounted. A
checkpoint reservation may exclude a blob only when the object validates and
its bytes are already included in `used_bytes`. A valid but unaccounted object
is either deleted before publication or explicitly adopted: its full length
is reserved and added to `used_bytes` by the final CAS even though no upload
is needed. An arbitrary pre-existing object has no free accounting credit.

Control bytes are tracked separately and may not exceed their allowance.
Exhausting it stops new publication and requires maintenance.

Committed delta objects remain in `used_bytes` until a checkpoint with
`base_commit >= their commit` is current and their deletion is confirmed.
Reaching either retained-delta limit forces checkpoint publication before
another ordinary commit.

## 14. Garbage collection and compaction

Remote maintenance is intentionally serialized. Normal commits stop while the
head is in `mode = maintenance`; local writes continue into the bounded
journal.

### 14.1 Crash-safe deletion batches

Remote quota is an aggregate counter, so deletion must be replayable. A GC
descriptor is an immutable, checksummed list of at most 4 MiB:

```text
volume UUID, writer token, GC ID, and expected head revision
target object key and exact length
whether each target is currently included in used_bytes
total accounted bytes to subtract
```

A deletion batch is:

1. Build the descriptor from a fixed maintenance view and write `gc/<id>` with
   `If-None-Match: *`.
2. CAS the maintenance head from no pending operation to that pending GC ID,
   checksum, and byte total.
3. Delete every target and confirm its absence by exact key.
4. CAS the head to clear the pending operation and subtract exactly the
   descriptor's accounted byte total.
5. Delete the unaccounted GC descriptor and confirm its absence before starting
   another operation.

An unclaimed descriptor has caused no deletion and is safe to remove. A crash
after step 2 resumes the exact list; a crash after target deletion but before
step 4 repeats the deletes and performs the subtraction once. An ambiguous
delete or CAS leaves the head pending and creates no quota credit. The control
allowance admits at most one unaccounted commit or GC descriptor.

Before any delete, recovery revalidates the descriptor checksum, writer token,
canonical volume-relative keys, exact lengths, and the fixed live view. A
descriptor may never target `meta/*`, itself, another pending operation, or a
currently marked object.

### 14.2 Finding garbage

A collection pass:

1. resolves any pending operation and removes unclaimed commit or GC
   descriptors, which cannot have published data;
2. CASes a normal head with no pending operation into maintenance;
3. verifies that local RocksDB is applied through `last_commit`;
4. marks `meta/super`, `meta/head`, the current base catalog and blobs, the
   deterministic commit range, and every segment with nonzero `live_slots`;
5. emits unmarked objects and zero-live segments through bounded deletion
   batches;
6. reconciles accounting and CASes the head back to normal.

The `segments` column family is the data-object live set. Remote listing, its
ordered records, the base catalog, and the numeric commit range are merged as
streams through `maintenance_bytes`; the full mark or delete set is never held
in RAM. No normal upload can create an object after the mark. A crash leaves
maintenance ownership in the head and recovery resumes or restarts the pass.

A missing `LIST` result can leak an object but cannot delete a live object:
every proposed deletion is checked against the fixed live streams and by exact
key. Listing is authority to discover candidates, never by itself authority to
lower `used_bytes`. Duplicate results are harmless. Upward reconciliation uses
the greater of the remote ledger and listed non-control bytes.

After a zero-live segment is deleted and its deletion batch is accounted, its
local descriptor may be removed in a synced RocksDB batch. This cleanup needs
no remote delta; replay may resurrect a harmless zero-live descriptor, and the
next checkpoint absorbs its removal.

### 14.3 Segment repacking

Repacking selects a low-live segment, verifies every apparently live slot
against the current extent lookup, and copies only exact matches. It publishes
replacement mappings through a pending maintenance commit, advances the same
contiguous commit number, and leaves the covered journal sequence unchanged.
The old segment enters a deletion batch only after the commit is current,
locally applied, and all reads that selected it have drained.

One pass publishes at most one repacking commit and obeys the ordinary 4 MiB
delta and retained-delta limits. If no delta capacity remains, the worker exits
maintenance, checkpoints, and restarts. A sampled audit compares extent
mappings with segment counters; repairing a mismatch requires a full offline
audit.

This stop-the-world remote design is a simplicity trade. A long pass can push
the journal into pacing or stall. Maintenance starts manually, above 75% remote
usage, when known garbage or low-live segments exceed 10% of data bytes, after
a checkpoint leaves obsolete manifest objects, or at least every 30 active
days.

## 15. Ownership and fencing

The head contains a random writer token. A normal attach must either:

- present the token already associated with the same cache UUID; or
- claim an unowned head by CAS.

A different cache requires explicit `takeover`. Takeover CASes a new token and
cache UUID into the head. Every later head mutation checks that token. A stale
worker may create only an unreferenced commit or GC descriptor; it cannot
install a commit or checkpoint.

Takeover is allowed only after a privileged recovery step has resolved every
pending operation from remote objects and returned the head to normal. It then
changes the token and cache UUID in one exact-ETag CAS. If the pending state
cannot be safely finished or aborted without the old journal, takeover refuses
and leaves it unchanged.

Takeover does not preserve dirty data on the old cache. Operators must recover,
export, or intentionally abandon that journal first.

Offline recovery never changes the remote writer token and never uploads. On
reconnection, the worker reads the head:

- if its cache UUID and token still match, it reconciles covered sequence and
  resumes;
- if the remote head advanced under another token, it freezes remote writeback
  and reports an orphaned local journal.

S3 CAS prevents remote corruption from two writers. It cannot prevent two
hosts from separately acknowledging divergent local writes. Operational
single-attachment fencing remains required.

## 16. Control operations

```text
deepdisk create
  Create an empty RocksDB checkpoint at base commit 0, meta/super, and a head
  with last commit 0.

deepdisk format-cache
  Format local cache partitions, initialize the quota-limited manifest
  directory, and assign a new cache UUID.

deepdisk attach
  Verify ownership, validate or restore the base RocksDB checkpoint, replay
  bounded commit deltas, recover the journal, and expose ublk.

deepdisk recover-local
  Recover and expose local data without S3; remote misses return EIO.

deepdisk export-journal
  Emit the newest journal-resident value of each block with sequence and CRC.

deepdisk remote-sync
  Seal the current local batch and wait for remote coverage through it.

deepdisk status
  Report ownership, pressure, coverage, quota, and recovery state.

deepdisk detach
  Stop new I/O, flush the local journal, and preserve any remotely dirty
  journal data.

deepdisk delete
  Require a completed remote sync followed by detach, tombstone the head, then
  delete the prefix.
```

`delete --abandon-local` is destructive and requires the cache UUID plus an
explicit confirmation token. Normal delete refuses while any known local
journal is ahead of the remote head.

## 17. Configuration

```text
fixed at volume creation
  block_bytes                  4096
  device_bytes                 fixed presented size
  remote endpoint, bucket, prefix
  remote_limit_bytes           required hard quota

fixed at cache format
  cache_bytes                  explicit local quota
  manifest directory          ext4/XFS project-quota path on NVMe
  manifest_disk_bytes          required explicit quota, at least 8 GiB
  region_bytes                 64 MiB
  journal_footer_bytes         512 KiB
  journal_bloom_bits_per_record 16
  journal_footer_pending_regions 4
  journal_bytes                min(max(5% cache, 8 GiB), 16 GiB)
  maintenance_bytes            min(max(0.5% cache, 1 GiB), 4 GiB)
  clean associativity          16

tunable at attach
  journal_directory_bytes       10 MiB
  clean_tag_ram_bytes             4 MiB
  rocksdb_block_cache_bytes       8 MiB
  rocksdb_write_buffer_bytes      4 MiB total
  rocksdb_max_open_files         128
  request_buffer_bytes             8 MiB
  journal_batch_buffer_bytes       8 MiB
  remote_work_buffer_bytes          8 MiB
  process_memory_limit            96 MiB
  ublk_queues                  min(online CPUs, 2)
  ublk_queue_depth                32
  max_request_bytes             128 KiB

local batching
  batch_bytes                  4 MiB encoded maximum
  busy_batch_delay             50 us

remote writeback
  commit_target_data_bytes     128 MiB
  commit_max_data_bytes        256 MiB
  commit_max_mappings           65,536
  commit_object_max_bytes        4 MiB
  remote_age_target            10 s
  segment_data_bytes           32 MiB
  upload_concurrency           2
  read_deadline                60 s

pressure
  drain_watermark              70%
  pace_watermark               85%
  stall_watermark              95%
  background_io_concurrency    2
  local_flush_p99_budget       2x measured idle p99

local manifest
  engine                       pinned RocksDB build
  target_sst_file_bytes        256 MiB
  retained_commit_max_count    4096
  retained_commit_max_bytes      1 GiB
  max_compaction_bytes           1 GiB input plus output
  manifest_emergency_bytes     max(1 GiB, 5% manifest quota)
  checkpoint_catalog_page_max_bytes 8 MiB
  segment_repack_live_ratio    50%

remote quota
  maintenance_reserve          max(1 GiB, 5%), capped at 64 GiB
  control_allowance             32 MiB
  gc_descriptor_max_bytes        4 MiB
```

Attach reports the interpreted byte values and refuses inconsistent
combinations. The configured journal directory must fit the derived Bloom
directory, footer page directories, active exact index, and the maximum footer
pipeline at worst-case 4 KiB records. Percentages are converted to bytes once;
pressure calculations do not use a moving denominator.

## 18. Observability

Metric labels use only bounded dimensions: volume, operation, source, state,
status, result, reason, direction, object type, I/O class, arena, and RocksDB
level. Object IDs, commit IDs, block numbers, and error strings are event
fields, never metric labels.

### 18.1 Host I/O

```text
deepdisk_io_requests_total{op,status}
deepdisk_io_bytes_total{op}
deepdisk_io_latency_seconds{op}
deepdisk_read_source_total{source=staging|journal|clean|remote|zero}
deepdisk_read_source_latency_seconds{source}
deepdisk_flush_wait_seconds
deepdisk_fua_writes_total
deepdisk_volatile_write_completions_total
deepdisk_stable_write_completions_total
deepdisk_inflight_requests
deepdisk_request_buffer_bytes
deepdisk_request_buffer_wait_seconds
```

### 18.2 Journal and local durability

```text
deepdisk_journal_used_bytes
deepdisk_journal_free_bytes
deepdisk_journal_regions{state}
deepdisk_journal_bloom_bytes
deepdisk_journal_bloom_lookups_total
deepdisk_journal_bloom_candidate_regions_total
deepdisk_journal_footer_page_reads_total
deepdisk_journal_footer_false_positives_total
deepdisk_journal_footer_pending_regions
deepdisk_journal_footer_build_seconds
deepdisk_journal_active_index_entries
deepdisk_local_batch_bytes
deepdisk_local_batch_requests
deepdisk_local_batch_delay_seconds
deepdisk_local_nvme_write_seconds
deepdisk_local_nvme_flush_seconds
deepdisk_local_batch_sequence
deepdisk_local_stable_batch_sequence
deepdisk_remote_covered_batch_sequence
deepdisk_dirty_batch_lag
deepdisk_oldest_uncovered_seconds
deepdisk_recovered_batches_total
deepdisk_recovery_scan_bytes
deepdisk_recovery_seconds
deepdisk_journal_crc_errors_total
```

### 18.3 Clean cache

```text
deepdisk_clean_hits_total
deepdisk_clean_misses_total
deepdisk_clean_hit_ratio
deepdisk_clean_tag_ram_hits_total
deepdisk_clean_tag_disk_reads_total
deepdisk_clean_tag_write_seconds
deepdisk_clean_mapping_mismatches_total
deepdisk_clean_admissions_total{result}
deepdisk_clean_evictions_total{reason}
deepdisk_clean_sequential_bypass_bytes_total
deepdisk_clean_occupancy_bytes
deepdisk_clean_fill_bytes
deepdisk_clean_crc_errors_total
deepdisk_tinylfu_bytes
```

### 18.4 Throttling

```text
deepdisk_pressure_ratio
deepdisk_pressure_state{state=normal|drain|pace|stall}
deepdisk_throttled_requests_total
deepdisk_throttle_wait_seconds
deepdisk_remote_commit_rate_bytes_per_second
deepdisk_local_admit_rate_bytes_per_second
deepdisk_journal_exhaustion_seconds_estimate
deepdisk_background_local_io_bytes_total{class}
deepdisk_background_local_io_wait_seconds{class}
```

### 18.5 Remote I/O and commits

```text
deepdisk_s3_requests_total{op,status}
deepdisk_s3_request_seconds{op}
deepdisk_s3_bytes_total{direction,type}
deepdisk_s3_retries_total{op}
deepdisk_remote_objects{type}
deepdisk_remote_commit_number
deepdisk_remote_commit_seconds
deepdisk_remote_commit_bytes
deepdisk_remote_commit_object_bytes
deepdisk_remote_retained_commits
deepdisk_remote_retained_commit_bytes
deepdisk_remote_head_mode{state=normal|maintenance}
deepdisk_remote_pending_operation{type=none|commit|checkpoint|gc}
deepdisk_remote_pending_recoveries_total{type,result}
deepdisk_head_cas_total{result}
deepdisk_remote_sync_wait_seconds
deepdisk_remote_read_coalesced_total
deepdisk_remote_corruption_total{type}
```

### 18.6 Manifest and maintenance

```text
deepdisk_manifest_extent_records
deepdisk_manifest_segment_records
deepdisk_manifest_seek_seconds
deepdisk_manifest_disk_bytes{class=live|checkpoint|pending|temporary}
deepdisk_manifest_disk_limit_bytes
deepdisk_manifest_worst_case_blocks_remaining
deepdisk_rocksdb_sst_files{level}
deepdisk_rocksdb_sst_bytes{level}
deepdisk_rocksdb_block_cache_bytes
deepdisk_rocksdb_block_cache_access_total{result}
deepdisk_rocksdb_memtable_bytes
deepdisk_rocksdb_pending_compaction_bytes
deepdisk_rocksdb_compaction_bytes{direction,level}
deepdisk_rocksdb_compaction_seconds
deepdisk_manifest_generation
deepdisk_manifest_applied_commit
deepdisk_manifest_base_commit
deepdisk_checkpoint_create_total{status}
deepdisk_checkpoint_trigger_total{reason}
deepdisk_checkpoint_create_seconds
deepdisk_checkpoint_files
deepdisk_checkpoint_logical_bytes
deepdisk_checkpoint_new_upload_bytes
deepdisk_checkpoint_unpublished_bytes
deepdisk_checkpoint_restore_bytes
deepdisk_checkpoint_restore_seconds
deepdisk_checkpoint_replay_commits_total
deepdisk_checkpoint_replay_bytes_total
deepdisk_checkpoint_replay_seconds
deepdisk_gc_state{state}
deepdisk_gc_mark_bytes
deepdisk_gc_list_objects
deepdisk_gc_deleted_objects
deepdisk_gc_deleted_bytes
deepdisk_gc_segment_live_ratio
deepdisk_gc_seconds
```

### 18.7 Quota, ownership, and resources

```text
deepdisk_remote_used_bytes
deepdisk_remote_reserved_bytes
deepdisk_remote_limit_bytes
deepdisk_remote_control_used_bytes
deepdisk_remote_maintenance_reserve_bytes
deepdisk_local_partition_bytes{class}
deepdisk_manifest_quota_ratio
deepdisk_process_rss_bytes
deepdisk_process_locked_bytes
deepdisk_process_memory_limit_bytes
deepdisk_cgroup_memory_current_bytes
deepdisk_arena_bytes{arena}
deepdisk_writer_token_valid
deepdisk_offline_mode
deepdisk_orphaned_journal
deepdisk_last_successful_remote_operation_timestamp_seconds
```

Structured events are emitted for:

- attach, detach, takeover, and fencing failure;
- entry into and exit from every pressure state;
- offline recovery and journal export;
- local dirty corruption and remote corruption;
- pending commit, checkpoint, and GC-batch recovery;
- credential expiry and remote outage;
- quota refusal and maintenance start, pause, resume, and completion;
- any invariant failure that stops the device.

## 19. Correctness invariants

The implementation must continuously assert:

1. An ordinary `WRITE` completes only after its complete journal batch write
   finishes. A completed `FLUSH` or `FUA` implies that every ordered
   predecessor crossed a successful NVMe flush.
2. A journal region is reused only after the remote head covers its last batch,
   local RocksDB has applied the commit that established that coverage, and
   the region has no read or writeback pin.
3. Every closed-region Bloom filter has no false negatives for its valid
   footer, and every accepted footer entry matches the region generation and
   an address-bound journal CRC.
4. Read lookup always checks staging and journal data before manifest, clean,
   or remote data.
5. A clean block is returned only when its block index, mapping tag, entry
   generation, and data CRC match the captured current manifest generation.
6. The live recovery chain is one validated base checkpoint followed by every
   contiguous commit from `base_commit + 1` through `last_commit`.
7. A remote head covers a journal cut only after every segment named by the
   new commit validates and that commit contains the last selected write to
   each block through the cut.
8. Extents are non-overlapping, and every extent references exact slots of a
   validated immutable segment.
9. No remote data segment is uploaded before a pending head reservation names
   its commit, checksum, and maximum bytes.
10. Remote used bytes decrease only in the final CAS of a pending GC batch
    whose exact targets are confirmed absent; ambiguous accounting may
    overcount but never creates quota credit.
11. Normal remote publication and garbage deletion never run concurrently.
12. Segment `live_slots` equals the number of current extent slots that name
    that segment.
13. Local RocksDB is never ahead of the remote head, and no later commit begins
    while it is behind. An ordinary commit's changed blocks remain in the
    journal; a repacking commit's old segments remain readable and contain the
    same logical bytes.
14. An old base checkpoint or absorbed commit is not deleted until the head
    installs its replacement base; a segment selected by a read is not deleted
    or replaced until that read drains.
15. A writer whose token no longer matches the head cannot publish.

An invariant failure stops new writes and preserves local evidence. It is not
converted into a cache miss or retried indefinitely.

## 20. Verification gates

### 20.1 Local crash tests

Fault injection must crash before and after every:

- journal data write;
- NVMe flush;
- host completion;
- superblock update;
- region-header and footer write;
- footer-index and Bloom-directory publication;
- clean data write;
- clean tag-page publication;
- journal-region reuse.

After each power-loss crash, a reference model permits only a contiguous
prefix of complete journal batches. The prefix must include every write before
a completed `FLUSH` and every completed `FUA` write; it may include later
complete ordinary batches even when their host completion was not observed.
It must never include a torn batch or a batch above the first invalid sequence.
A daemon-crash test, which does not discard device cache contents, requires
every completed write to survive.

The harness independently drops, tears, and reorders clean data-slot and tag
writes. Every resulting entry must either validate against the retained
manifest and return current bytes or behave as a cache miss.

Footer tests corrupt every field, omit the newest footer, and saturate the
maximum footer-pending pipeline. Rebuilt indexes must produce the same newest
journal value as a full sequence scan, and a Bloom lookup must never omit a
region containing the queried block.

### 20.2 Remote commit tests

Crash and ambiguous-response injection covers commit-object creation, pending
head reservation, every segment upload, final head CAS, local RocksDB apply,
and superblock update. It separately covers checkpoint reservation, catalog
and blob uploads, base installation, and every step of a GC deletion batch.

Recovery must finish or abort without:

- publishing a missing object;
- dropping local journal data;
- undercounting quota;
- installing an incomplete or corrupt checkpoint catalog;
- deleting an object reachable from the current base-and-delta chain;
- applying a commit out of order or applying it twice with a different result.

A successful final head CAS immediately followed by local RocksDB quota
exhaustion must retain the journal, block the next commit, and recover by
replay after space is restored.

Takeover tests lose the old cache in every pending state. They must either
finish or abort that state before changing the writer token; a stale worker's
later CAS must fail.

### 20.3 Manifest and GC tests

Random histories of writes, extent splits and merges, RocksDB compactions,
commit aborts, checkpoint creation and restore, repacking, and garbage
collection are compared to a simple in-memory block map. After every operation,
per-segment live counts are recomputed independently.

The collector is tested with:

- crashes during mark, repack, delete, and accounting;
- duplicate and omitted `LIST` results;
- missing, truncated, substituted, and hash-mismatched checkpoint blobs;
- a crash with a pending commit, checkpoint, or GC batch at every publication
  step;
- a final head CAS immediately before local manifest-generation installation;
- old-generation reads that outlive a manifest switch;
- restore from a base followed by 0, 1, and 4096 deltas;
- refusal before a 4097th retained delta or more than 1 GiB of retained delta
  bytes, followed by successful checkpoint publication;
- missing, duplicate, reordered, wrong-predecessor, and wrong-volume commit
  objects during replay;
- manifest project-quota exhaustion during flush and compaction;
- quota exhaustion with only maintenance reserve available;
- a segment becoming completely dead and one remaining barely live.

### 20.4 Offline recovery tests

With every S3 operation forced to fail:

- daemon recovery exposes every completed local write still present in the
  journal;
- power-loss recovery preserves every completed `FLUSH`/`FUA` boundary and
  accepts only additional complete batches;
- persistent clean hits validate against the retained local manifest and
  return;
- absent local blocks return `EIO`;
- writes proceed until pressure stalls them;
- `export-journal` produces a deterministic latest-block stream;
- reconnect resumes only when the remote writer token still matches.

### 20.5 Performance and resource gates

Before release:

- before pressure, ordinary sustained writes achieve at least 90% of a raw
  O_DIRECT local append baseline's throughput and no more than 1.5 times its
  p99 latency with the same NVMe write-cache mode, request mix, batch size,
  queue depth, and no flushes;
- a separate `FLUSH`/`FUA` comparison uses identical durability boundaries and
  reports both throughput and latency against the raw device;
- both write comparisons are repeated while clean fill, RocksDB delta apply
  and compaction, hashing, and remote upload are saturated, with p99 no more
  than twice the idle baseline;
- a manifest-hot, tag-hot clean hit has p99 latency no more than 1.5 times a
  raw 4 KiB NVMe read;
- after its manifest lookup, a tag-cold hit performs no more than one
  tag-page-pair read plus one data read;
- manifest-cold clean-hit latency and RocksDB I/O count are reported
  separately and issue no S3 request;
- changing clean capacity from 1 TiB to 4 TiB increases required userspace RAM
  by less than 4 MiB;
- the maximum 16 GiB journal's Bloom directory, footer directories, active
  index, and four footer-pending indexes fit in the configured 10 MiB arena;
- a 100-million-extent RocksDB remains inside its configured block-cache,
  write-buffer, file-handle, and process limits;
- an unchanged checkpoint reuploads no SST content, and an incremental
  checkpoint uploads only newly created files and its catalog;
- commit encoding, delta application, checkpoint hashing, and manifest-blob
  upload reduce steady-state remote coverage throughput by no more than 10%
  versus uploading the same data segments without metadata work;
- checkpoint restore time and required local bytes are measured at every
  supported manifest scale;
- default attach and every steady-state stress case remain below the 96 MiB
  worker-cgroup limit, including RocksDB native allocations and filesystem
  page cache charged to that cgroup;
- remote outage tests demonstrate pacing and a clean stall before reserve
  exhaustion;
- ext4 and XFS pass power-cut, daemon-crash, and remote-outage workloads.

No performance number is accepted without the matching queue depth, durability
mode, working-set size, cache warmth, and percentile.

## 21. Design summary

DeepDisk v1 has four mechanisms:

1. an append-only local journal that keeps the write path on NVMe and makes
   flushed dirty data locally recoverable without S3;
2. a deterministic set-associative clean cache whose authoritative tags live
   on disk and whose RAM is only a bounded accelerator;
3. one conventional local RocksDB extent map;
4. immutable S3 data segments plus small ordered redo commits, with occasional
   physical RocksDB checkpoints to bound restore.

The ordinary write hot path is one bounded memory copy and one local journal
write. It performs no RocksDB write, clean-cache update, S3 request, or NVMe
flush. Linux `FLUSH`/`FUA` adds exactly the local durability operation the
caller requested. A hot clean read performs one local RocksDB lookup from cache
and one NVMe data read; colder local metadata can add NVMe reads but never an
S3 request.

The design spends complexity only where correctness requires it:

- standard Linux volatile-write-cache semantics;
- one mutable remote head serializing reservation and publication;
- one conventional local metadata database;
- one writer;
- one stop-the-world collector.

There is no database running on object storage. RAM and clean-cache capacity
are independent, and the default worker is hard-limited to 96 MiB. Manifest
disk and disaster-recovery time scale with mapping fragmentation and are
exposed as first-class costs.
