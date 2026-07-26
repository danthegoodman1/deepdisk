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

When selecting which dirty blocks to pack into a segment, we prefer spatially adjacent blocks. A read fault fetches a whole segment, so contiguous packing turns that fetch into a useful prefetch.

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

A full manifest is written periodically as a checkpoint, so recovery is the newest checkpoint plus the deltas since it, rather than the entire history.

Recovery always restores the last committed root. Filesystem flush/FUA barriers are used as epoch boundaries where they land, since they mark points the filesystem itself considers consistent, and a barrier only costs a delta and a root write. Otherwise the epoch is cut on the age cap. Root writes are rate limited to ~1/s so a bursty workload cannot commit continuously, and use a conditional put so a writer that was partitioned and then revived cannot clobber a newer root.

#### Snapshots and clones

Immutable segments plus a versioned root make the remote copy-on-write. A write never modifies remote data, it lands in a new segment and the map is repointed, so every committed epoch stays a complete and consistent image for as long as the segments under it survive. Note that this applies only to the remote, the local cache is still overwrite in place over a dirty bitmap.

That gives us, with no extra machinery:

1. Snapshots, which are a pinned root. Nothing is copied and nothing moves, so taking one is O(1)
2. Rollback, which is selecting an older root
3. Clones, which are a copied root. Both volumes share every existing segment and only diverge as they are written to

Versioning here is our own, via epoch named objects. S3 object versioning is per object and gives no coherent point in time across objects, which is the entire property we want.

Two limits to be clear about. A snapshot captures remote state, so one that is meant to include current writes has to close an epoch first, then pin. And granularity is the epoch, so values a block held between two epochs are coalesced away and cannot be recovered. This is not continuous data protection.

#### Failure and garbage

A failed upload leaves its blocks dirty and retries with backoff, so a sustained remote outage walks up the watermarks into throttling and then stalling. Whether a stalled device should instead go read-only is open.

Because the remote is copy-on-write, overwriting a block leaves the old value in place in an already uploaded segment, so dead data accumulates and segments have to be compacted. A block is dead only when no retained root references it, not simply when a newer write supersedes it, so a held snapshot pins segments that would otherwise be reclaimable. Space pinned per snapshot should be exposed as a metric, since a forgotten snapshot silently stops reclamation and grows the remote footprint without bound.
