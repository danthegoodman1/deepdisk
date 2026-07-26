# DeepDisk Spec

DeepDisk is a bottomles block device (disk) for Linux that offers local read and write performance (most of the time).

It does this by using local disk as the durability boundary, while asynchronously writing in batches to another storage location (e.g. S3). DeepDisk also keeps track of block heat, and locally caches blocks in disk.

Because it is just a block device, that means the implementation can be simplified, and existing filesystems (ext4, xfs) can be used on top. Because there are no special filesystem rules, almost all Linux software will work out of the box on top, with the same performance a majority of the time.

## Block caching

The OS uses its own normal page cache, while the local disk cache is managed by DeepDisk. When we refere to the cache, we're referring to the local disk cache.

DeepDisk will keep well known blocks always locally cached (e.g. where filesystem metadata is expected to be), and otherwise has three cache categories:
1. Dirty blocks
2. Hot blocks
3. Recently read blocks

For the local cache, a block is marked dirty via a bitmap, where the bitmap indicates the offset index in the block device (e.g. for a 4k block size, block offset 8192 is index 2). A separate system that runs async to the hot path is used to classify hot vs recently read.

The rules follow this:
1. Dirty blocks will always evict recently read blocks, or hit blocks
2. Hot blocks can evict recently read blocks
3. If the local cache is filled with dirty blocks due to local writes exceeding flush rate, the disk is throttled

### Flushing

There are two separate things called a flush, and they never block on each other:

1. A block device flush (`REQ_OP_FLUSH`, `REQ_FUA`) issued by the filesystem above means writing to the local cache. It completes once the data is durable on local disk
2. A remote flush means uploading dirty blocks to the remote, and always happens async in the background

So the most durable a flush can be is local performance. An `fsync()` above DeepDisk returns at local disk speed, and never waits on the remote. This is what makes the local performance claim hold, and it is why local disk is the durability boundary. Everything below concerns the remote flush.

Dirty blocks are packed into segments (4-8MB) and uploaded as a single object, rather than uploaded individually. Segments are assembled at flush time, not write time, so a block overwritten many times between flushes is only uploaded once.

Dirty blocks are flushed for a number of reasons:

1. We've hit a size threshold, i.e. a full segment. This is the normal path
2. Age cap, where a block has been dirty for longer than some duration (default 10s). Age is measured from the clean to dirty transition, not from the last write, so a block that is continuously rewritten still gets uploaded on schedule at whatever value it currently holds. This bounds how much data is lost if the local disk fails, and is the primary RPO knob
3. Idle, no writes for ~250ms, so we seal and upload a partial segment
4. Dirty pressure, see watermarks below
5. Clean shutdown or an explicit checkpoint, where everything is flushed

A block written in the last ~1s is held back from a segment being packed, to absorb repeated overwrites. This is only an optimization to avoid redundant uploads, never a reason to defer one, so the age cap, dirty pressure, idle, shutdown and checkpoint all override it.

When selecting which dirty blocks to pack into a segment, we prefer spatially adjacent blocks. A read fault range gets only the blocks it needs, at the same request cost as fetching the whole segment, so contiguous packing is what makes widening that range into a prefetch worth doing rather than pulling in unrelated blocks.

#### Watermarks

As a fraction of the local cache occupied by dirty blocks:

1. Under 25%, lazy, only reasons 1-3 apply
2. 25-60%, steady background flush
3. 60-85%, maximum upload concurrency, and the overwrite window is ignored
4. Over 85%, incoming writes are throttled
5. Over 95%, writes stall

#### Epochs

Uploading lazily and out of order leaves the remote copy torn across many points in time, so restoring it would produce an image that never existed. Flushing is therefore epoch structured:

1. Segments are uploaded as soon as they seal, in any order
2. A delta, listing block index -> segment + offset for only the blocks that moved this epoch, is uploaded under the epoch number
3. A small root object recording the latest committed epoch is written. This is the atomic commit, and the durability point
4. Only then are those blocks marked clean in the dirty bitmap, making them evictable

Segments are immutable, and an uploaded segment is invisible until a committed root references it. This is what keeps the remote path off the critical path: uploads can be issued the moment a segment seals, need no ordering between them, and can be retried or duplicated freely, because the only step that has to be ordered is the small metadata write at the end. A partially uploaded epoch is harmless, as nothing ever refers to it.

The delta is proportional to the data it describes rather than to the size of the device, so committing stays cheap no matter how large the volume is. It should encode to little: the segment ID is constant across the entries and belongs in a header, the offset is implicit if entries are in segment slot order, which leaves only the block indices, and those are runs of consecutive values whenever packing was contiguous. A full 4MB segment is a few hundred bytes as extents, up to ~4KB for a random scatter.

The index that these deltas apply to is checkpointed periodically, so recovery is the newest checkpoint plus the deltas since it rather than the entire history. See Manifest below.

Recovery always restores the last committed root. Filesystem flush/FUA barriers are used as epoch boundaries where they land, since they mark points the filesystem itself considers consistent, and a barrier only costs a delta and a root write. Otherwise the epoch is cut on the age cap. Root writes are rate limited to ~1/s so a bursty workload cannot commit continuously, and use a conditional put so a writer that was partitioned and then revived cannot clobber a newer root.

#### Snapshots and clones

Immutable segments plus a versioned root make the remote copy-on-write. A write never modifies remote data, it lands in a new segment and the map is repointed, so every committed epoch stays a complete and consistent image for as long as the segments under it survive. Note that this applies only to the remote, the local cache is still overwrite in place over a dirty bitmap.

That gives us, with no extra machinery:

1. Snapshots, which are a pinned root. Nothing is copied and nothing moves, so taking one is O(1)
2. Rollback, which is selecting an older root
3. Clones, which are a copied root. Both volumes share every existing segment and only diverge as they are written to

#### Failure and garbage

A failed upload leaves its blocks dirty and retries with backoff, so a sustained remote outage walks up the watermarks into throttling and then stalling. Whether a stalled device should instead go read-only is open.

Because the remote is copy-on-write, overwriting a block leaves the old value in place in an already uploaded segment, so dead data accumulates and segments have to be compacted. A block is dead only when no retained root references it, not simply when a newer write supersedes it, so a held snapshot pins segments that would otherwise be reclaimable. Space pinned per snapshot should be exposed as a metric, since a forgotten snapshot silently stops reclamation and grows the remote footprint without bound.

## Manifest

The manifest maps block index to segment + slot. There are two separate maps and this is the cold one: the local cache map (block index -> cache slot, dirty, heat) is consulted on every I/O and has to be fast, while the manifest is only consulted on a local cache miss, at which point we are already committed to a 20-100ms network fetch. A lookup taking microseconds is free in that context, so the manifest is optimized for size rather than latency.

It is a write ahead log plus a periodically checkpointed index. It is not an LSM: there are no levels, no bloom filters, and no read path through the deltas. The merged view is always materialized locally, and deltas are read only during mount. Writing index pages every epoch would amplify badly, since one scattered entry dirties a leaf plus its whole path to the root, so instead each epoch appends a small delta and the dirty pages accumulate in memory until a checkpoint makes them durable.

### Structure

Per block entries do not scale. At 6 bytes each, 1TB of 4K blocks is 1.5GB, and 10TB is 15GB. Because we pack spatially adjacent blocks into segments and slot order within a segment is implicit, a contiguous run is instead a single extent.

The index is a radix tree over the block index. Block indices are dense integers, so there are no comparisons and no rebalancing, and unwritten regions are simply absent. That last property matters for a bottomless device, where most of the address space was never touched: a read of a never written block returns zeros with no I/O and costs nothing in the map.

Pages are not fixed size, since object storage does not require it. A page serializes to whatever its encoding needs and its parent records the length.

Interior entries are 11 bytes, `(index u8, pack u32, offset u32, len u16)`, so a fanout of 256 is 2816 bytes. Entries are locations rather than hashes, so an unchanged child is never rewritten and keeps pointing into whatever older pack it already lives in.

A leaf covers 4096 blocks, 16MB of address space at 4K, and picks its encoding:

1. Absent, never written, no parent entry and no bytes
2. Single extent, one contiguous run, ~12 bytes
3. Extent list, a few runs, varint delta encoded, ~8 bytes per extent
4. Dense, badly fragmented, 4096 x 6 bytes = 24KB

1TB written sequentially is ~65k single extent leaves plus 256 interior pages and a root, around 1MB of metadata. The same 1TB written as pure random 4K degenerates to dense leaves and ~1.5GB, which is why the tree lives on local disk with an LRU of pages in memory rather than being required to fit in RAM. A page fault on the manifest costs ~100us in front of a network fetch that costs 50ms.

The tree is copy on write. The flusher builds new pages for the paths it touches and swaps the root with a release store, so readers take the root pointer and traverse an immutable snapshot with no locks on the read path at all. A block device has a single writer, so this needs no further coordination. Old pages are reclaimed once no reader holds them.

Pages are packed rather than written individually. 65k tiny leaf objects would be a terrible object to byte ratio, so a checkpoint concatenates all of its dirty pages into one pack object and uploads it once, making a checkpoint 1-2 puts no matter how many pages changed. Packs accumulate dead pages as those pages are superseded, so packs need liveness tracking and compaction using the same mechanism as data segments.

### Layout

Three object classes, and the root is the only mutable one:

```
data/seg/000000A1F3              4MB immutable segment, self describing
data/seg/000000A1F4

meta/pack/00000012               index pages written by checkpoint @ epoch 41200
meta/pack/00000013               index pages written by checkpoint @ epoch 42200

meta/delta/0000000000042201      ~1KB, one per epoch
meta/delta/0000000000042202
   ...
meta/delta/0000000000042847

meta/root                        { epoch: 42847,
                                   tree: (pack 13, off 8192, len 3400),
                                   checkpoint_epoch: 42200,
                                   delta_from: 42201 }
```

Keys are deterministic, and the root records the current checkpoint plus the live delta range, so we never list a prefix to find objects. Listing is only for garbage collection and repair.

```
                        root page          (pack 13)
                      /      |      \
              L1 page     L1 page     L1 page
             (pack 13)   (pack 9)    (pack 13)     <- unchanged subtree still
             /   |   \                                points into an older pack
        leaf   leaf   leaf
          |      |      |
      SINGLE  EXTENT   DENSE
      EXTENT   LIST    24KB
       ~12B    ~200B
```

### Loops

Every epoch, around 10s, which is two small puts plus the segment and no index pages:

```
PUT data/seg/...            the segment
apply to in-memory tree     copy on write leaves + paths, stays dirty in memory
PUT meta/delta/N            the delta
PUT meta/root               { epoch: N, tree: <unchanged>, delta_from: M+1 }
mark blocks clean
```

Every ~1000 epochs, around 3h:

```
serialize pages dirtied since the last checkpoint -> one pack
PUT meta/pack/K
PUT meta/root               { epoch: N, tree: (K, off, len), delta_from: N+1 }
DELETE meta/delta/M+1 .. N
```

Mount:

```
GET meta/root
GET the root page                      range get into its pack
GET meta/delta/{delta_from..epoch}     in parallel, ~1000 objects, sub-second
replay them into memory
-> serving I/O, with interior and leaf pages faulted in on demand
```

Index pages can be faulted in lazily, but deltas cannot, because we cannot know which of them supersede a page we have not fetched yet. So the delta count between checkpoints is what sets mount latency, and it is the reason to bound it at ~1000 rather than letting the stream grow to the ~260k objects a month of 10s epochs would produce.

### Read path

```
read LBA 9,412,096
  |- local cache map hit -> serve from local disk               (common case)
  \- miss -> walk in-memory radix tree -> (segment A1F3, slot 412)
             range GET data/seg/000000A1F3 bytes 1687552..1691647
```

Once warm there are no metadata requests on the read path, which is the property this layout exists to protect.

### Reverse mapping

Garbage collection needs to know which blocks in a segment are still live, and scanning the forward map to find out is far too expensive. Instead each segment carries a live block counter, decremented at epoch apply time when a block index that pointed at it is repointed, since the old value is already in hand at that moment. Garbage collection then sorts segments by live count. The counter is a cache and can be rebuilt by scanning if lost.

Snapshots complicate this, because the counter only tracks the current root while a block is dead only when no retained root references it. Computing the union of live sets across retained roots during a collection pass is fine for a handful of snapshots, and gets expensive if hundreds are held.
