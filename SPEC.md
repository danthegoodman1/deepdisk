# DeepDisk Spec

DeepDisk is a bottomles block device (disk) for Linux that offers local read and write performance (most of the time).

It does this by using local disk as the durability boundary, while asynchronously writing in batches to another storage location (e.g. S3). DeepDisk also keeps track of block heat, and locally caches blocks in disk.

Because it is just a block device, that means the implementation can be simplified, and existing filesystems (ext4, xfs) can be used on top. Because there are no special filesystem rules, almost all Linux software will work out of the box on top, with the same performance a majority of the time.

## Units

```
block     4KB           the addressable unit, what every map is keyed by
granule   64KB          allocation and eviction unit of the local cache, 16 blocks
segment   4-8MB         a remote object, what dirty blocks are packed into
slot      one block     a block's position inside a segment
extent    variable      a contiguous run of block indices, an encoding rather than storage
leaf      4096 blocks   a manifest page, covering 16MB of address space
pack      variable      a remote object holding manifest pages
epoch     ~10s          one commit, the unit of remote durability
```

Granules are local and segments are remote, and the two never align: a granule holds whatever the cache decided to keep, a segment holds whatever happened to be dirty when a flush fired. Extents are the odd one out, since they are how a map writes down a run of blocks compactly rather than a unit anything is stored in.

## Device

DeepDisk attaches as a `ublk` device, a userspace block driver that passes requests over `io_uring`, so the policy engine, the object store client and the TLS stack all live in a normal userspace process while the kernel sees a real block device. `nbd` is the fallback for kernels older than 6.0.

The daemon sits in the writeback path, which makes it a memory reclaim hazard. The kernel can flush dirty pages to satisfy an allocation, and if serving that flush requires the daemon to allocate, the two deadlock. So the daemon sets `PR_SET_IO_FLUSHER`, preallocates its request and segment buffers at startup, and locks itself in memory. Allocation on the hot path is a correctness bug.

`ublk` also supports user recovery, where the daemon can exit and be restarted without tearing down the device or the filesystem above it. That matters because the daemon holds the dirty bitmap and the manifest overlay, so without it a restartable process becomes an outage.

## Processes

One worker process per volume, plus a singleton supervisor that owns the control API and nothing on the data path.

Isolation decides it. A worker holds the dirty bitmap, the granule table and the manifest overlay for its volume, all in memory, so a crash is scoped to one volume and user recovery makes it survivable. Buffers are preallocated per worker, which keeps one volume's writeback from waiting on another to release memory.

The host is what needs coordinating. Upload bandwidth, cache device capacity and compaction budget are shared across volumes, so the supervisor hands them out as leases, and a worker that loses contact keeps its last lease and decays to a fixed share.

Workers serve I/O with no dependency on the supervisor being alive. A dead supervisor means no new control operations and no rebalancing, while every attached volume keeps serving. Each worker also exposes the same per volume API on its own socket, so a volume stays drivable while the supervisor is down.

### Reconciliation

The supervisor holds no durable state. It rebuilds its whole view on startup from the kernel and from the workers themselves.

`flock` on `/run/deepdisk/supervisor.lock` keeps it a singleton, and the kernel releases that lock on process death however the death happens. Attached volumes come from the ublk devices the kernel already knows about, each of which reports the pid of the process serving it, and workers bind a socket per volume at `/run/deepdisk/volumes/{uuid}.sock`. The supervisor connects to each, reads `SO_PEERCRED`, and requires the peer pid to match the one the kernel names as that device's server, which settles identity across a pid reuse.

A live worker is adopted as it stands, with its device untouched. A device whose worker is gone sits in recovery pending, and the supervisor starts a replacement that reattaches through user recovery, leaving the device and the filesystem above it in place. Leases are then reissued from the aggregate: each worker reports what it currently holds, the supervisor sums them and redistributes. Leases carry an expiry, so a supervisor absent for longer than one finds every worker already decayed to its fixed share and can allocate freely.

Workers are spawned detached and outlive the supervisor that started them. The gap this leaves is a worker crashing while the supervisor is down, which holds that one volume in recovery pending until the supervisor returns.

### Control API

HTTP and JSON over a unix socket at `/run/deepdisk/control.sock`, guarded by filesystem permissions, since these calls create and destroy volumes.

Epochs are the currency. Every call that changes durable state returns the epoch it produced and every call that reads it returns the epoch it observed, so a caller can always ask what point in time it is holding, and ask for everything up to now to become durable.

```
POST   /v1/volumes                       create, or clone with { clone_of }
GET    /v1/volumes                       list
GET    /v1/volumes/{id}                  epoch, watermark, dirty bytes, logical/physical
DELETE /v1/volumes/{id}

POST   /v1/volumes/{id}/attach           bring up the ublk device
POST   /v1/volumes/{id}/detach           flush, commit, tear down

POST   /v1/volumes/{id}/flush            -> { epoch, committed, noop }
POST   /v1/volumes/{id}/checkpoint       fold the overlay, write a pack -> { epoch }
POST   /v1/volumes/{id}/await            { epoch }, blocks until it is durable
POST   /v1/volumes/{id}/grow             { device_bytes } -> { epoch }

POST   /v1/volumes/{id}/snapshots        { epoch | "now" } -> { snapshot, epoch }
GET    /v1/volumes/{id}/snapshots        each with the bytes it pins
DELETE /v1/volumes/{id}/snapshots/{sid}
POST   /v1/volumes/{id}/rollback         { epoch }, requires detached

POST   /v1/volumes/{id}/compact          -> { job }
GET    /v1/jobs/{id}
```

`flush` is the call to get right. It commits an epoch and returns its number, and if nothing is dirty it returns the current epoch immediately with `noop: true`, so calling it defensively is free. Concurrent calls coalesce onto one commit and all return the same epoch, which makes it safe to call from several places at once. An explicit flush bypasses the ~1/s root write rate limit, since the caller is asking for a durability point directly. `wait=false` returns as soon as the epoch is assigned, to be paired with `await`.

Clone is a copied root, so it is O(1) and needs no worker at all, just the supervisor copying that object and `meta/heat` under a new prefix, the second so the clone starts warm instead of relearning what the source already knows. Cloning from `"now"` implies a flush first, cloning from a snapshot or an explicit epoch does not. A fork is a clone that is attached.

Rollback requires the volume to be detached. Repointing the tree underneath a live device leaves the page cache and the filesystem above holding state from a tree that no longer describes the volume, so the API refuses it.

### CLI

`deepdisk` is a thin client over that socket, adding no surface of its own.

```
deepdisk create <vol> --size 100T [--block-size 4096] --cache /dev/nvme0n1p2 --remote s3://bucket/prefix
deepdisk ls
deepdisk status <vol>                    epoch, watermark, dirty, logical/physical
deepdisk rm <vol> --yes

deepdisk attach <vol> [--read-only] [--snapshot <id>]
deepdisk detach <vol>

deepdisk flush <vol> [--no-wait]         -> epoch
deepdisk checkpoint <vol>                -> epoch
deepdisk await <vol> <epoch>
deepdisk grow <vol> --size 200T          -> epoch

deepdisk snap <vol> [--at <epoch>]       -> snapshot, epoch
deepdisk snaps <vol>                     each with the bytes it pins
deepdisk snap rm <vol> <snap> --yes
deepdisk rollback <vol> --to <epoch> --yes

deepdisk clone <src> <dst> [--at <epoch> | --snapshot <id>]
deepdisk fork <src> <dst>                clone and attach

deepdisk compact <vol>                   -> job
deepdisk jobs [<job>]
deepdisk metrics [<vol>]
```

It is built to be driven by scripts and agents as much as by people. Every command takes `--json`, and human output is never the only format. Nothing ever prompts, so destructive operations take `--yes` instead of asking. Commands that produce an epoch print it alone on stdout, so `EPOCH=$(deepdisk flush vol0)` works.

Exit codes separate the cases a caller would act on differently:

```
0   ok
1   usage or arguments
2   no such volume, snapshot or job
3   conflict, the operation is refused in this state, such as rollback while attached
4   degraded, stalled or the remote is unreachable, retrying may succeed
5   fenced, the volume needs an operator and retrying will not help
```

Socket discovery follows the same rule the daemons do. It tries `/run/deepdisk/control.sock`, and for a command naming one volume it falls back to that volume's own socket, so a supervisor that is down never blocks operating a volume that is up.

## Block caching

The OS uses its own normal page cache, while the local disk cache is managed by DeepDisk. When we refere to the cache, we're referring to the local disk cache.

DeepDisk will keep well known blocks always locally cached (e.g. where filesystem metadata is expected to be), and otherwise has three cache categories:
1. Dirty blocks
2. Hot blocks
3. Recently read blocks

For the local cache, a block is marked dirty via a bitmap, where the bitmap indicates the offset index in the block device (e.g. for a 4k block size, block offset 8192 is index 2). Those last two are the main and small queues below.

The rules follow this:
1. Dirty blocks will always evict recently read blocks, or hit blocks
2. Hot blocks can evict recently read blocks
3. If the local cache is filled with dirty blocks due to local writes exceeding flush rate, the disk is throttled

### Cache layout

The cache is a raw block device or partition, so DeepDisk owns allocation, journaling and flush semantics all the way down to the hardware.

Space is allocated in 64KB granules, 16 blocks at 4K, which keeps the map for 1TB of cache at 16M entries and ~160MB of RAM. Dirty state is still tracked per block inside the granule as a 16 bit mask, so coarse allocation costs no write amplification to the remote, and only the read that fills a granule is coarse.

DeepDisk advertises a volatile write cache, so a write is acknowledged once it is in memory and only has to reach disk on `REQ_OP_FLUSH` or `REQ_FUA`. That is the same contract as any disk with a write cache, and it means the metadata cost below is paid once per barrier rather than once per write.

Durability of the cache has the same shape as the manifest, a log plus a checkpoint, for the same reason. A barrier is not complete until both the data and enough metadata to find it again are on disk, so each one appends an intent record, `(granule, block mask, block index)` per granule touched, to a ring on the same device. The granule table is checkpointed periodically, and recovery is the last table plus the ring tail, which bounds recovery time by the ring size rather than by the size of the cache.

### Heat and eviction

Eviction is the three FIFO queue structure S3-FIFO describes, operating on granules. A small queue holds ~10% of capacity and everything enters there. A granule touched again while in small is promoted to main on its way out, and one that was never touched again leaves for a ghost queue holding its fingerprint and none of its data. A later hit on a ghost entry admits straight to main. Frequency is two bits.

Scan resistance is why. A backup, a `grep -r` or an `updatedb` walks the whole volume once, and under recency that walk evicts the working set it passes over. Here it never leaves the small queue. FIFO also means the read path appends and never reorders a list, so it takes no lock and allocates nothing, which is what the writeback path requires.

DeepDisk is a second level cache and sees the page cache's miss stream, already stripped of the short term reuse that recency exploits. Frequency based admission is the right shape for that, and it also means published hit rates for this family come from web and CDN traces and want measuring here rather than assuming.

Sequential streams are detected separately, since the queues have no notion of them. A detected stream prefetches ahead and is admitted at low value, so a large read gets readahead without occupying main.

State is three bits per granule, a two bit counter and a queue id, so it folds into the granule table. The ghost queue is the part to size deliberately: one fingerprint per evicted granule is 16M entries for 1TB of cache, and even 8 bytes each is ~128MB on top of the ~160MB table.

Dirty granules are exempt, since they are pinned until uploaded. Replacement therefore governs only the clean portion of the cache, and that portion shrinks as dirty pressure climbs, so at the 85% watermark the policy is working with 15% of the device and hit rate degrades exactly when the system is already stressed. A write allocates unconditionally and joins the queues as a hit once its epoch commits.

### Warm start

Heat is checkpointed with the granule table, so a worker restart resumes with its working set intact. The cached data survives a restart on its own, and this is what keeps it findable as valuable, since a cache that comes back uniformly cold loses its working set to the first scan that follows.

A host with no local cache at all, after a migration, a clone or cache loss, gets a summary instead. The worker walks its main queue on its own timer and overwrites `meta/heat`, reading nothing, since the queues are already in memory. It sits outside the commit path entirely: no root references it, it takes no conditional create, and a second writer clobbering it costs a worse prefetch list and nothing more. A fixed key means a cold mount fetches it without knowing anything about epochs.

What the summary holds is block ranges. A granule is 16 consecutive blocks, so what identifies it on another host is the address range it covers, and address ranges stay meaningful across hosts, clones and compaction in a way cache slots and segment IDs do not. Entries are extent encoded, bucketed into a few heat tiers, and sorted by address within each tier, so a prefetcher walks the tiers in order for priority and gets coalesced range reads inside each one. The object is capped at ~1MB, filled with the hottest extents that fit.

Prefetch resolves each range through the manifest before it can fetch anything, so it follows the interior load rather than racing it, and it inherits the same locality: blocks that were hot together were usually written together, so they sit in the same segments and their range gets coalesce.

Losing it costs a cold cache and nothing else, so it carries no retention rules and nothing has to collect it.

Prefetch never gates I/O. Serving begins as soon as the manifest answers lookups, and a demand read for a range prefetch has not reached yet just fetches it. Prefetched granules are admitted at low value, the same as a detected sequential stream, so speculative data enters the small queue and reaches main only if something actually reads it, which keeps a wrong guess from evicting a working set that was loaded on demand. It draws on a bounded request budget that demand traffic preempts, and on the bandwidth the supervisor leases, so warming a new host never competes with a volume already serving. Past 60% dirty it stops entirely, since by then both cache space and upload bandwidth are worth more to the flusher.

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

Dirty and clean share one device, so this ladder is also what reserves space for the read cache: the stall at 95% is the floor that keeps a clean portion at all. Tuning these for flush behaviour tunes read hit rate at the same time.

#### Epochs

Uploading lazily and out of order leaves the remote copy torn across many points in time, so restoring it would produce an image that never existed. Flushing is therefore epoch structured:

1. Segments are uploaded as soon as they seal, in any order
2. A delta, listing block index -> segment + offset for only the blocks that moved this epoch, is uploaded under the epoch number
3. A root object is written for the epoch and the head is swapped to point at it. That swap is the atomic commit, and the durability point
4. Only then are those blocks marked clean in the dirty bitmap, making them evictable

Segments are immutable, and an uploaded segment is invisible until the head names a root that references it. This is what keeps the remote path off the critical path: uploads can be issued the moment a segment seals, need no ordering between them, and can be retried in any order, because the only step that has to be ordered is the small metadata write at the end. A partially uploaded epoch is harmless, as nothing ever refers to it.

The delta is proportional to the data it describes rather than to the size of the device, so committing stays cheap no matter how large the volume is. It should encode to little: the segment ID is constant across the entries and belongs in a header, the offset is implicit if entries are in segment slot order, which leaves only the block indices, and those are runs of consecutive values whenever packing was contiguous. A full 4MB segment is a few hundred bytes as extents, up to ~4KB for a random scatter.

The index that these deltas apply to is checkpointed periodically, so recovery is the newest checkpoint plus the deltas since it rather than the entire history. See Manifest below.

Recovery always restores the last committed root. Filesystem flush/FUA barriers are used as epoch boundaries where they land, since they mark points the filesystem itself considers consistent, and a barrier only costs a delta and a root write. Otherwise the epoch is cut on the age cap. Commits are rate limited to ~1/s so a bursty workload cannot commit continuously, and the head swap is conditional, see Fencing below.

#### Snapshots and clones

Immutable segments plus a versioned root make the remote copy-on-write. A write never modifies remote data, it lands in a new segment and the map is repointed, so every committed epoch stays a complete and consistent image for as long as the segments under it survive. Note that this applies only to the remote, the local cache is still overwrite in place over a dirty bitmap.

That gives us, with no extra machinery:

1. Snapshots, which mark a root as retained along with the delta range it spans. Nothing is copied and nothing moves, so taking one is O(1)
2. Rollback, which commits a new epoch pointing at an older root, reaching any epoch still retained
3. Clones, which are a copied root. Both volumes share every existing segment and only diverge as they are written to

#### Failure and garbage

A failed upload leaves its blocks dirty and retries with backoff, so a sustained remote outage walks up the watermarks into throttling and then stalling. See Errors below for what happens at the top of that ladder.

Because the remote is copy-on-write, overwriting a block leaves the old value in place in an already uploaded segment, so dead data accumulates and segments have to be compacted. A block is dead only when no retained root references it, not simply when a newer write supersedes it, so a held snapshot pins segments that would otherwise be reclaimable. Space pinned per snapshot should be exposed as a metric, since a forgotten snapshot silently stops reclamation and grows the remote footprint without bound.

## Manifest

The manifest maps block index to segment + slot. There are two separate maps and this is the cold one: the local cache map (block index -> cache slot, dirty, heat) is consulted on every I/O and has to be fast, while the manifest is only consulted on a local cache miss, at which point we are already committed to a 20-100ms network fetch. A lookup taking microseconds is free in that context, so the manifest is optimized for size rather than latency.

It is a write ahead log plus a periodically checkpointed index. The merged view is always materialized locally, and the log is read only during mount. Writing index pages every epoch would amplify badly, since one scattered entry dirties a leaf plus its whole path to the root, so instead each epoch appends a small delta and the dirty pages accumulate in memory until a checkpoint makes them durable.

### Structure

Per block entries do not scale. At 6 bytes each, 1TB of 4K blocks is 1.5GB, and 10TB is 15GB. Because we pack spatially adjacent blocks into segments and slot order within a segment is implicit, a contiguous run is instead a single extent.

The index is a radix tree over the block index. Block indices are dense integers, so there are no comparisons and no rebalancing, and unwritten regions are simply absent. That last property matters for a bottomless device, where most of the address space was never touched: a read of a never written block returns zeros with no I/O and costs nothing in the map.

Pages are variable size. A page serializes to whatever its encoding needs and its parent records the length.

Interior entries are 11 bytes, `(index u8, pack u32, offset u32, len u16)`, so a fanout of 256 is 2816 bytes. Entries are locations rather than hashes, so an unchanged child is never rewritten and keeps pointing into whatever older pack it already lives in.

A leaf covers 4096 blocks, 16MB of address space at 4K, and picks its encoding:

1. Absent, never written, no parent entry and no bytes
2. Single extent, one contiguous run, ~12 bytes
3. Extent list, a few runs, varint delta encoded, ~8 bytes per extent
4. Dense, badly fragmented, 4096 x 6 bytes = 24KB

1TB written sequentially is ~65k single extent leaves plus 256 interior pages and a root, around 1MB of metadata. The same 1TB written as pure random 4K degenerates to dense leaves and ~1.5GB, which is why the tree lives on local disk with an LRU of pages in memory rather than being required to fit in RAM. A page fault on the manifest costs ~100us in front of a network fetch that costs 50ms.

The tree is copy on write. The flusher builds new pages for the paths it touches and swaps the root with a release store, so readers take the root pointer and traverse an immutable snapshot with no locks on the read path at all. A block device has a single writer, so this needs no further coordination. Old pages are reclaimed once no reader holds them.

Pages are packed. A checkpoint concatenates all of its dirty pages into one pack object and uploads it once, making a checkpoint 1-2 puts no matter how many pages changed, and keeping the object to byte ratio sane across 65k leaves.

Within a pack, interior pages are serialized first and in tree order, ahead of the leaves. Nothing else constrains their placement, so without this the quality of a bulk interior load is accidental. Writing them contiguously and in level order turns loading the whole interior into a handful of near sequential range reads, and it compounds with locality aware compaction below.

Because a child pointer is a location rather than a hash, a root to leaf path can span several packs, and a cold walk is then one range get per level into a different object. Two things keep that bounded. The tree is shallow, three levels at 1TB and four at 100TB, so it is at most three or four fetches rather than an unbounded chain. And copy on write dirties the whole path at once, so a freshly written root, interior page and leaf all land in the same pack. Jumping only happens descending from a hot ancestor into a subtree that has not changed in a long time, which is also the subtree least likely to be read.

Packs accumulate dead pages as those pages are superseded, so packs need liveness tracking and compaction using the same mechanism as data segments. Compaction is locality aware: when live pages are rewritten out of old packs they are grouped by tree region, so a path lands back in a single pack. Reclaiming space is the lesser reason to run it, keeping walks from fanning out across a long tail of mostly dead packs is the greater one.

### Layout

Everything is immutable except `meta/head`, and `meta/heat` which holds no durable state:

```
data/seg/000000A1F3              4MB immutable segment, self describing
data/seg/000000A1F4

meta/pack/00000012               index pages written by checkpoint @ epoch 41200
meta/pack/00000013               index pages written by checkpoint @ epoch 42200

meta/delta/0000000000042801      ~1KB, one per epoch
meta/delta/0000000000042802
   ...
meta/delta/0000000000042847

meta/merged/0000000000042800     the overlay serialized, one per ~100 epochs

meta/heat                        hottest block ranges, overwritten freely, no durable state

orphan/42847/3f2ab19c/           a fenced writer's last writes, root and segments

meta/super                       { uuid, format_version, block_bytes: 4096 }

meta/root/0000000000042847       { epoch: 42847,
                                   writer_id: "3f2ab19c",
                                   tree: (pack 13, off 8192, len 3400),
                                   tree_bytes: 1046528,
                                   checkpoint_epoch: 42200,
                                   merged_from: 42800,
                                   delta_from: 42801,
                                   device_bytes: 109951162777600,
                                   logical_bytes: 8246337208320,
                                   physical_bytes: 11338713397657 }

meta/head                        { epoch: 42847 }
```

Roots are immutable and written one per epoch, and `meta/head` is the only mutable object in the volume. Committing is a create of the root followed by a compare and swap of the head, so the head is what fencing contends on and it stays small enough that the check is a few bytes. Every epoch that ever committed is still addressable, which is what rollback and point in time recovery reach for.

Keys are deterministic, and the root records the current checkpoint plus the live delta range, so we never list a prefix to find objects. Listing is only for garbage collection and repair.

`meta/super` is written once at creation and never again. Block size cannot change without invalidating every mapping in the volume, so it lives in an object that nothing is allowed to rewrite, and is validated on mount.

The root carries the sizes because they change with the epoch and have to be atomic with it:

1. `device_bytes`, the capacity presented to Linux. Keeping it here makes an online grow a normal epoch commit rather than separate mutable state that can disagree with the manifest, and it means a volume mounted on another host needs nothing out of band. It must never shrink below the highest mapped block
2. `logical_bytes`, the address space actually written. This is the thin provisioned used figure, and is what a bottomless device reports as consumed rather than `device_bytes`
3. `physical_bytes`, total bytes stored remotely, garbage and metadata objects included. This is the bill, so it counts dead regions inside segments right up until compaction reclaims them, and it counts packs, deltas and merges

The gap between the last two is what deferring compaction costs, so a pass can be scheduled from two numbers in the root without scanning segments or walking the tree. All three are maintained incrementally, since recomputing them means a full scan.

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

One root is written per epoch and one head swap commits it. The two loops below are extra work that rides on an epoch commit, never a second commit of their own.

Every epoch, around 10s, which is three small puts plus the segment and no index pages:

```
PUT data/seg/...            the segment
apply to the overlay        in memory, no leaves touched
PUT meta/delta/N            the delta
PUT meta/root/N             { epoch: N, tree: <unchanged>, delta_from: M+1 }
CAS meta/head               { epoch: N } if it still holds N-1
mark blocks clean
```

An epoch never touches the tree. Applying to leaves instead would require them to be resident, so a write to a cold region would fault its leaf from the remote purely to overwrite entries in it, and that fetch would sit between the commit and marking blocks clean. Remote metadata latency would then feed straight into local dirty pressure and could throttle writes over metadata rather than data. The delta is the durable record and the tree is derived, so the tree is only updated when it is convenient to do so.

Every ~1000 epochs, around 3h, that commit does more before its delta, and its root names the new tree:

```
fold the overlay into leaves    faults only the leaves it touches, batched
serialize pages dirtied since the last checkpoint -> one pack
PUT meta/pack/K
   ...root for this epoch carries tree: (K, off, len), delta_from: N+1
DELETE meta/delta/M+1 .. N      except where a retained root still needs them
drop the overlay
```

Folding at the checkpoint is where leaves are loaded, and doing it in one batch coalesces those loads across the whole interval.

A root between checkpoints carries a tree from the last checkpoint plus a delta range, so it is only readable while those deltas exist. Retaining a root therefore retains its delta range, and the checkpoint skips any delta a retained root still spans. That is what makes a snapshot at an arbitrary epoch a real image rather than a pointer that stops resolving at the next checkpoint, and it is cheap, since a retained interval is ~1000 objects at ~1KB.

Every ~100 epochs, around 15 minutes, that commit carries one extra put:

```
PUT meta/merged/N           the overlay serialized as it stands
   ...root for this epoch carries merged_from: N, delta_from: N+1
```

The overlay already holds the merged view of every delta since the last checkpoint, so writing it out costs one put and reads nothing. A merge exists to keep the delta stream short for whoever mounts next, which is a different concern from folding the tree and wants a different frequency: the tree checkpoint is paced by how much leaf rewriting is worth doing, a merge by how long a cold mount is allowed to take. It uses the same extent encoding as a delta, so it is ~1 byte per entry and reaches ~1MB by the end of a checkpoint interval, against the ~10MB the same map costs in memory. Each one rewrites content the previous already wrote, ~5.5MB across the interval, which is small enough to ignore.

Mount:

```
GET meta/head then meta/root/{epoch}   two small sequential gets
GET the root page                      range get into its pack
GET meta/merged/{merged_from}          one object, up to ~1MB
GET meta/delta/{delta_from..epoch}     in parallel, up to 100 objects
replay them into an overlay
-> serving I/O

background:
  L2   100 pointers    -> sorted, coalesced -> a few range gets    ~280KB
  L1   25,600 pointers -> sorted, coalesced -> tens of range gets  ~72MB
  leaves faulted in on demand
```

The overlay is a flat in memory map of block index to segment + slot, holding every epoch committed since the last checkpoint and consulted before descending the tree. It is how the map is maintained at all times, and mount rebuilds it by replaying the deltas the running writer holds in memory. Any lookup checks the overlay, then walks the tree, faulting what is missing. It is bounded by the checkpoint interval at ~1M entries, around 10MB, and is dropped once folded in. `meta/merged/N` is this same map serialized, and exists only so a cold mount can rebuild it without replaying every delta.

Interior pages are pinned rather than faulted. They are few and small, ~724KB for 1TB as a root plus 256 interior pages and ~72MB for 100TB written, so holding all of them costs nothing next to the volume they describe and makes every cold lookup exactly one fetch, the leaf. Note this scales with written data rather than presented capacity, so a bottomless volume showing 100TB with 10TB written is ~7MB. The crossover where this stops being free is closer to 1PB written, at ~720MB.

They are pulled in the background rather than blocking the mount, and a lookup that arrives before its interior page does simply faults it. The load is breadth first and bulk rather than page at a time: once a level is parsed we hold every `(pack, offset, len)` for the level below, so those pointers are sorted by pack and offset and coalesced into large range reads, merging any two separated by less than ~64KB since reading junk bytes is far cheaper than a second request. That is three dependent rounds, since a level's locations are unknown until its parent is parsed, and tens of requests, or ~700ms on 1Gbps. It is also a cold host cost only, since the tree is materialized on local disk and a remount reads the same 72MB from NVMe in ~35ms.

Loading only the interior is the policy for a large tree. The checkpoint records the total serialized size of the tree in the root as `tree_bytes`, and below a threshold, say 256MB, we load all of it including leaves; a sequentially written 1TB volume is around 1MB in total, and pulling that in one range read means no metadata request ever reaches the read path. Above the threshold a faulting leaf widens its fetch to its neighbors in the same pack, which are the leaves for regions written at the same time, so the extra bytes are nearly free next to the request already being made. This is the same reasoning that makes contiguous packing worthwhile for data segments.

Index pages can be faulted in lazily, but deltas cannot, because we cannot know which of them supersede a page we have not fetched yet. Everything committed since the last checkpoint has to be in hand before the first read is served, so the merge interval is the mount latency knob, and it is set independently of how often the tree is folded.

### Read path

```
read LBA 9,412,096
  |- local cache map hit -> serve from local disk               (common case)
  \- miss -> overlay, then the radix tree -> (segment A1F3, slot 412)
             range GET data/seg/000000A1F3 bytes 1687552..1691647
```

Once warm there are no metadata requests on the read path, which is the property this layout exists to protect.

### Reverse mapping

Garbage collection needs to know which blocks in a segment are still live, and scanning the forward map to find out is far too expensive. Instead each segment carries a live block counter, decremented at epoch apply time when a block index that pointed at it is repointed, since the old value is already in hand at that moment. Garbage collection then sorts segments by live count. The counter is a cache and can be rebuilt by scanning if lost.

Snapshots complicate this, because the counter only tracks the current root while a block is dead only when no retained root references it. Computing the union of live sets across retained roots during a collection pass is fine for a handful of snapshots, and gets expensive if hundreds are held.

## Discard

`REQ_OP_DISCARD` unmaps rather than writes. The manifest entry is removed, the local cache granule is freed, and the segment that held the data has its live counter decremented through the same reverse mapping path an overwrite uses. `logical_bytes` drops immediately, `physical_bytes` only when those segments are compacted.

Because an absent entry already reads as zeros, DeepDisk can honestly report that discarded blocks return zeros deterministically. That is worth advertising, since it lets the filesystem skip zeroing ranges it has just released, and it makes `REQ_OP_WRITE_ZEROES` the same operation as discard.

Discards arrive in floods, since `fstrim` releases everything a filesystem is not using in a single pass, so they have to be range operations. This is where the radix tree pays off a second time: unmapping a large range drops whole leaves and whole interior entries, so the cost is proportional to the tree nodes the range spans and not to the blocks in it, and a discard covering an entire subtree is one entry removed from its parent.

Unmapping commits like a write, as an entry in the epoch delta, so a discard is durable only once its epoch commits and is never visible remotely ahead of the root that records it.

## Errors

Reads and writes fail differently, because the truth is in different places. A write is durable locally the moment its barrier completes, so an unreachable remote costs us space rather than data, and the response is to accumulate dirty blocks and walk up the watermarks. A read that misses the local cache has no local truth to fall back on, so for it an unreachable remote is a real failure.

A missing read retries with backoff against a deadline, 60s by default, then returns `EIO`, which surfaces an error the layers above can report and act on. The deadline is generous because most outages are shorter than it and an `EIO` is expensive, often a read only remount.

At the top of the watermark ladder the stall is held indefinitely, because the data is already durable and the workload is only being asked to wait, so the outage costs latency and leaves integrity intact. A deadline after which writes take `EIO` is available as policy for workloads that prefer to fail fast.

## Fencing

Running one writer is the application's guarantee to make, and it is the operational requirement DeepDisk places on whoever deploys it. What follows detects a second writer and contains it. It does not make concurrent mounts work, and it does not recover what the second writer costs, so an orchestrator that fails over on unreachability alone will eventually lose data here. Unreachable and dead are different states, and only the deployment can tell them apart.

Fencing lives entirely on the remote and the epoch is the token. Every head swap is conditional on the head still holding the epoch this writer last committed, so two writers advancing from the same epoch resolve into one winner and one conditional failure. Every object the head reaches is a conditional create as well, since those keys are deterministic and a second writer computes them identically: both pack segment `A1F4` and both write `meta/delta/106`, so under last write wins the winner's committed epoch could point at the loser's bytes. An existing key means another writer has claimed it, which fences on the first object of an epoch rather than at the root put. A writer retrying its own upload reads the header back and finds its own `writer_id`.

The loser holds an invalid copy. Not a stale one that could be caught up and not a divergent one that could be merged, since its blocks were written against allocation state the winner never saw. So it flushes its dirty blocks to `orphan/{epoch}/{writer_id}/`, fails outstanding and subsequent I/O with `EIO`, and drops the mount, and it never re-reads the root to retry from the newer epoch, which is the one step that would turn a fence back into a race. That prefix holds a private root and its segments, so an operator can clone it into a separate volume and pull files out by hand. It is never applied onto the winner. Garbage collection also never touches it, since its liveness rules would read an unreferenced root as dead, so the prefix stays until an operator removes it and a split brain leaves a trace that does not expire on its own.

The same rule runs at mount, since a writer can be fenced and then crash before acting on it. An ordinary restart keeps its cache, because the dirty blocks there are acknowledged writes that have not been uploaded, so one number separates the two cases: the cache superblock records the epoch it is working toward, written before the root put, which holds it at or ahead of the root for a legitimate writer even when a crash lands between them. A head ahead of the cache means another writer committed, and the whole cache is orphaned and dropped, since entries are keyed by block index and the winner may have rewritten any of them. This depends on a strictly monotonic epoch, so rollback commits forward, writing a new epoch whose tree points at older content.

This protects the integrity of the remote. Two writers can never produce a torn or interleaved image, and a zombie serving reads mutates nothing. The flush window is separate: the writes the loser acknowledged and had not yet uploaded are unreachable from the winner, so split brain converts the age cap from an RPO window into a data loss window and one knob bounds both. Surviving a host means uploading, and fencing is orthogonal to it.

Conditional put is a hard backend requirement. S3, GCS, Azure and R2 all expose it as ETag or object version preconditions, and a transactional store has it inherently.

## Configuration

```
volume, fixed at creation
  block_bytes           4096      changing it invalidates every mapping
  granule_bytes         64KB      local cache allocation unit
  leaf_blocks           4096      manifest leaf coverage, 16MB of address space
  cache_device          -         raw device or partition
  remote                -         bucket and prefix

volume, tunable
  device_bytes          -         presented capacity, grows only
  segment_bytes         4MB       upload unit, 4-8MB
  age_cap               10s       oldest dirty block before a flush, the RPO knob
  idle_seal             250ms     quiet period after which a partial segment seals
  overwrite_window      1s        holds back very recent writes, always overridable
  commit_rate           1/s       floor on the interval between epochs
  merge_interval        100       epochs between meta/merged writes, the mount knob
  checkpoint_interval   1000      epochs between tree folds
  read_deadline         60s       before a missing read returns EIO
  write_deadline        off       before a stalled write returns EIO

watermarks, as a fraction of the cache held dirty
  steady                25%       background flush begins
  aggressive            60%       maximum upload concurrency, overwrite window ignored
  throttle              85%       incoming writes slowed
  stall                 95%       incoming writes blocked

cache
  small_queue           10%       of capacity, the admission filter
  ghost_entries         1x main   fingerprints of evicted granules
  prefetch_cutoff       60%       dirty fraction at which prefetch stops
  heat_bytes            1MB       cap on the published working set summary

manifest
  full_load_threshold   256MB     tree_bytes under which leaves are loaded too
  coalesce_gap          64KB      range reads closer than this are merged

host
  upload_budget         all       leased across volumes by the supervisor
  compaction_budget     all       the same
  lease_ttl             30s       after which a worker decays to a fixed share
```

Three of these are the ones worth reaching for. `age_cap` sets how much data a host failure costs. `merge_interval` sets cold mount latency. `checkpoint_interval` sets how much metadata is rewritten. The rest have defaults that should hold.

The fixed group is fixed for different reasons. `block_bytes` is recorded in `meta/super` and validated on mount, since every mapping in the volume is expressed in it. `granule_bytes` and `leaf_blocks` are structural rather than durable, so changing them is possible by rebuilding, not by setting a flag.

## Open questions

1. Garbage collection is sketched rather than designed. The supervisor leases compaction budget across volumes, but there is no policy for the garbage ratio that should trigger a pass, and no rule for how compaction yields to flushing inside one volume as dirty pressure climbs
2. Snapshot liveness is a union across retained roots, fine for a handful and expensive at hundreds. Liveness may need to be tracked per snapshot, or snapshot count may need a bound
3. Leaf size is fixed at 4096 blocks, which makes a dense leaf 24KB, and that one number drives two costs. A checkpoint over a badly fragmented region writes roughly 6x the bytes that changed, and it rules out a key value backend like FoundationDB, which recommends values under ~10KB and would otherwise be attractive for mount time and for the RPO floor that rate limiting root writes imposes. Leaf size should be tunable, and the dense threshold may want to split a leaf rather than densify it
4. Compression and encryption are unaddressed. Both belong at the segment boundary, which is also where they collide with range gets, since a read fault has to decrypt and decompress a slice rather than the whole object
5. Roots and their deltas accumulate at one per epoch, so something has to expire them. A retention window bounds it, ~8,600 roots and deltas a day at 10s epochs, a few tens of MB, but the window and whether it is time or count based are unset
6. Clones share segments across volume prefixes and nothing refcounts them. A volume's garbage collection has to account for roots in other volumes that reference its objects, and deleting a source volume would currently destroy every clone taken from it
7. Hit rate is unmeasured. The eviction structure is chosen on the shape of the workload rather than on numbers from it, and the queue sizes, the ghost queue length and the sequential detector's thresholds are all guesses until there are traces to tune against
