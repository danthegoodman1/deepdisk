# DeepDisk Spec

DeepDisk is a bottomles block device (disk) for Linux that offers local read and write performance (most of the time).

It does this by using local disk as the durability boundary, while asynchronously writing in batches to another storage location (e.g. S3). DeepDisk also keeps track of block heat, and locally caches blocks in disk.

Because it is just a block device, that means the implementation can be simpler than a network filesystem, and existing filesystems (ext4, xfs) can be used on top. Because there are no special filesystem rules, almost all Linux software will work out of the box on top, with the same performance except under extreme write pressure or very sparse reads.

## Units

```
block     4KB           the addressable unit, what every map is keyed by
region    32MB          allocation and reclamation unit of the local cache
segment   4-8MB         a remote object, what dirty blocks are packed into
slot      one block     a block's position inside a segment or a region
extent    variable      a contiguous run of block indices, an encoding rather than storage
leaf      4096 blocks   a manifest page, covering 16MB of address space
pack      variable      a remote object holding manifest pages
epoch     ~10s          one commit, the unit of remote durability
```

Regions are local and segments are remote, and neither has anything to do with the address space. Both are logs: a region holds whatever this host wrote or fetched while it was open, a segment holds whatever happened to be dirty when a flush fired, and in both a slot records where the data landed rather than where it belongs. Extents are the odd one out, since they are how a map writes down a run of blocks compactly rather than a unit anything is stored in.

## Device

DeepDisk attaches as a `ublk` device, a userspace block driver that passes requests over `io_uring`, so the policy engine, the object store client and the TLS stack all live in a normal userspace process while the kernel sees a real block device. It is the only device backend, so Linux 6.0 or newer is a hard requirement.

The daemon sits in the writeback path, which makes it a memory reclaim hazard. Reclaim flushes dirty pages to satisfy an allocation; that flush reaches the filesystem, the block layer, `ublk` and then the daemon; and if serving it requires the daemon to allocate, that allocation enters the same reclaim already waiting on it. Nothing times out. The machine hangs with the filesystem mid writeback and the only way out is a reset.

`PR_SET_IO_FLUSHER` is half the answer. It sets `PF_MEMALLOC_NOIO`, so the daemon's own allocations do not recurse into I/O to satisfy themselves, and `PF_LOCAL_THROTTLE`, so a writeback participant is not throttled against its own writeback. It needs `CAP_SYS_RESOURCE`. What it does not do is make an allocation succeed: it converts a deadlock into a stall or a failure, so not allocating remains a requirement in its own right.

Not allocating is a property of every line rather than a rule anyone can hold in their head, and the process contains a TLS stack and an HTTP client that allocate per request by construction. So the daemon reserves one arena at startup, sized from the cache device and the configured concurrency, locks it, and runs a global allocator over it. Everything after initialisation is served out of memory the process already owns and the kernel is not involved, which replaces reclaim deadlock with arena exhaustion: bounded, observable and reportable.

That is also what makes the claim testable. The allocator counts every fall through to the system allocator, and that count is asserted zero after warmup under the full workload. Allocation on the hot path is a correctness bug, so it is a gate rather than an intention.

The remote client is inside the arena rather than beside it, because the paths are not as separate as they look. A read miss is the milder case, since reclaim never needs a read. A write normally lands in the cache and is acked without touching the remote at all, but above the throttle watermark it waits for an upload to complete, and dirty pressure and memory pressure arrive together.

`ublk` also supports user recovery, where the daemon can exit and be restarted without tearing down the device or the filesystem above it. That matters because the daemon holds the cache map and the manifest overlay, so without it a restartable process becomes an outage. It does not make a restart free: the cache map is rebuilt from the device in a second or two, but the overlay is rebuilt by replaying deltas from the remote, so a restart blocks I/O for as long as that takes.

## Processes

One worker process per volume, plus a singleton supervisor that owns the control API and nothing on the data path.

Isolation decides it. A worker holds the cache map and the manifest overlay for its volume, all in memory, so a crash is scoped to one volume and user recovery makes it survivable. Buffers are preallocated per worker, which keeps one volume's writeback from waiting on another to release memory.

Every resource a worker uses is its own. Cache space follows the device, and upload and compaction bandwidth are static per volume settings fixed at attach, so a worker's rate is settled the moment it starts and stays settled.

Workers therefore serve I/O with no dependency on the supervisor being alive. A dead supervisor means no new control operations, while every attached volume keeps serving at exactly the rate it would have anyway. Each worker also exposes the same per volume API on its own socket, so a volume stays drivable while the supervisor is down.

### Reconciliation

The supervisor holds no durable state. It rebuilds its whole view on startup from the kernel and from the workers themselves.

`flock` on `/run/deepdisk/supervisor.lock` keeps it a singleton, and the kernel releases that lock on process death however the death happens. Attached volumes come from the ublk devices the kernel already knows about, each of which reports the pid of the process serving it, and workers bind a socket per volume at `/run/deepdisk/volumes/{uuid}.sock`. The supervisor connects to each, reads `SO_PEERCRED`, and requires the peer pid to match the one the kernel names as that device's server, which settles identity across a pid reuse.

A live worker is adopted as it stands, with its device untouched, and nothing is renegotiated with it. A device whose worker is gone sits in recovery pending, and the supervisor starts a replacement that reattaches through user recovery, leaving the device and the filesystem above it in place.

Workers are spawned detached and outlive the supervisor that started them. The gap this leaves is a worker crashing while the supervisor is down, which holds that one volume in recovery pending until the supervisor returns.

### Control API

HTTP and JSON over a unix socket at `/run/deepdisk/control.sock`, guarded by filesystem permissions, since these calls create and destroy volumes.

Epochs are the currency. Every call that changes durable state returns the epoch it produced and every call that reads it returns the epoch it observed, so a caller can always ask what point in time it is holding, and ask for everything up to now to become durable.

```
POST   /v1/volumes                       create with { remote }, or fork with { fork_of, at }
GET    /v1/volumes                       list, with size, attach state and sibling heads
GET    /v1/volumes/{id}                  epoch, watermark, dirty, sizes, usage as of, remote
DELETE /v1/volumes/{id}                  { limit } -> { deleted, remaining }

POST   /v1/volumes/{id}/attach           bring up the ublk device, takes { remote }
POST   /v1/volumes/{id}/detach           flush, commit, tear down, or { force }
POST   /v1/volumes/{id}/credentials      re-supply on a live attachment, for rotation

POST   /v1/volumes/{id}/flush            -> { epoch, committed, noop }
POST   /v1/volumes/{id}/await            { epoch }, blocks until it is durable
POST   /v1/volumes/{id}/grow             { device_bytes } -> { epoch }

POST   /v1/volumes/{id}/snapshots        { epoch | "now" } -> { snapshot, epoch }
GET    /v1/volumes/{id}/snapshots        each with the bytes it pins
DELETE /v1/volumes/{id}/snapshots/{sid}

POST   /v1/volumes/{id}/compact          -> { job }
GET    /v1/jobs/{id}
```

`flush` is the call to get right. It commits an epoch and returns its number, and if nothing is dirty it returns the current epoch immediately with `noop: true`, so calling it defensively is free. Concurrent calls coalesce onto one commit and all return the same epoch, which makes it safe to call from several places at once. An explicit flush bypasses the ~1/s root write rate limit, since the caller is asking for a durability point directly. `wait=false` returns as soon as the epoch is assigned, to be paired with `await`.

Fork writes one head naming an existing root, so it is O(1) and needs no worker at all, just the supervisor issuing a conditional create. Forking from `"now"` implies a flush first, forking from a snapshot or an explicit epoch does not. The new head shares the prefix and every object under it, diverging only as it is written to, so it needs no credentials beyond the ones the origin already had. It starts with a cold cache and warms by being read, which is the cost of the whole feature being a single small put.

Rolling back is that same operation. Every committed epoch stays addressable, so a fork at epoch N is the volume as it stood at N, and attaching it in place of the original is how a volume goes backwards.

Delete removes the head first, which makes the volume unmountable from the moment the call returns, so an interrupted delete is definitively gone rather than mysteriously broken. Reclaiming what it referenced is then a garbage collection question rather than a prefix walk, since other heads may reach the same objects, and it proceeds a bounded number of objects per call so an operator paces a large reclaim instead of issuing millions of deletes at once. Deleting the last head in a prefix has nothing left to union against, so it degenerates to erasing the prefix. `--all` loops for the impatient.

Detach flushes, commits and tears down, and refuses if it cannot commit, reporting the dirty bytes it would have stranded. `--force` tears down anyway and leaves those blocks in the local cache, where a later attach on the same host resumes uploading them. The danger it protects against is detaching to move a volume elsewhere without noticing that recent writes are still sitting on the machine being left behind, so the amount is always printed rather than inferred.

Recovering to an earlier point is a fork at that epoch, attached in place of the original. The original stays intact until someone deletes it, so going back is non destructive and reversible, and it reaches any epoch the retention window still holds.

### CLI

`deepdisk` is a thin client over that socket, adding no surface of its own.

```
deepdisk create <vol> --size 100T [--block-size 4096] --remote s3://bucket/prefix#head
                      --cache /dev/nvme0n1p2 [--cache-size 100G]
                      [--endpoint <url>] [--region <r>] [--addressing path|vhost]
deepdisk ls                              size and attach state
deepdisk status <vol>                    epoch, watermark, dirty, device/logical, usage as of
deepdisk rm <vol> --yes [--limit N] [--all]

deepdisk attach <vol> [--read-only] [--snapshot <id>] [--cache-size 100G]
                      [--endpoint <url>] [--region <r>] [--addressing path|vhost] [--ca <path>]
deepdisk detach <vol> [--force]
deepdisk credentials <vol>               re-supply on a live attachment

deepdisk flush <vol> [--no-wait]         -> epoch
deepdisk await <vol> <epoch>
deepdisk grow <vol> --size 200T          -> epoch

deepdisk snap <vol> [--at <epoch>]       -> snapshot, epoch
deepdisk snaps <vol>                     each with the bytes it pins
deepdisk snap rm <vol> <snap> --yes

deepdisk fork <src> <dst> [--at <epoch> | --snapshot <id>] [--attach]
deepdisk heads <vol>                     every head over the same prefix

deepdisk compact <vol>                   -> job
deepdisk jobs [<job>]
deepdisk metrics [<vol>]
```

Credentials are the one input that is never an argument. `create`, `attach` and `credentials` read `DEEPDISK_ACCESS_KEY_ID`, `DEEPDISK_SECRET_ACCESS_KEY` and `DEEPDISK_SESSION_TOKEN` from the environment, or `--credentials-file` naming a file the CLI reads itself, and send the values over the socket.

It is built to be driven by scripts and agents as much as by people. Every command takes `--json`, and human output is never the only format. Nothing ever prompts, so destructive operations take `--yes` instead of asking. Commands that produce an epoch print it alone on stdout, so `EPOCH=$(deepdisk flush vol0)` works.

Exit codes separate the cases a caller would act on differently:

```
0   ok
1   usage or arguments
2   no such volume, snapshot or job
3   conflict, the operation is refused in this state, such as compacting while detached
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

Dirty is a flag on the cache map entry rather than a bitmap over the address space. A bitmap indexed by device block would scale with presented capacity, 3.2GB for a 100TB volume at 4K, almost all of it describing blocks that were never written and could not be dirty. Only a cached block can be dirty, so dirty state belongs in the structure that already holds one entry per cached block. The last two categories are the main and small classes below.

The rules follow this:
1. Dirty blocks will always evict recently read blocks, or hit blocks
2. Hot blocks can evict recently read blocks
3. If the local cache is filled with dirty blocks due to local writes exceeding flush rate, the disk is throttled

### Cache layout

The cache is a raw block device or partition, so DeepDisk owns allocation, journaling and flush semantics all the way down to the hardware.

One device belongs to one volume. A shared cache with a shared allocator would let one volume's dirty pressure starve another's read cache, which is the coupling the process model exists to avoid. Subdividing an NVMe across several volumes is a job for partitions or LVM, which already solve it, and DeepDisk sees a block device either way.

`cache_bytes` caps how much of that device is used, defaulting to all of it, and it bounds the footprint on a device rather than dividing one. Setting a 2TB drive to give up 100GB and no more is one number rather than a repartition. Holding back 10 to 20% is worth doing even on a dedicated device, since a log structured cache that fills a drive completely leaves its translation layer nothing to work with.

It is read at attach. Growing means the extra regions are there next time. Shrinking drops the regions above the new mark, and refuses when any of them still holds dirty blocks, since those are acknowledged writes and the fix is to attach at the old size and flush first. `map_unit` holds across a change, because entries are laid out on the device per unit; a cache grown a long way past what its unit was sized for is worth reformatting, which costs a cold cache and nothing else.

Sizing the cache sizes the exposure. Everything not yet uploaded lives here and nowhere else, so the throttle watermark against `cache_bytes` is the ceiling on how much acknowledged data a host failure destroys, in the same way `age_cap` is the ceiling on how old it can be.

The device is a log. It is divided into fixed 32MB regions, and a block written or fetched is appended to whichever region is currently open for its class. Nothing is ever overwritten in place, and a block's address has no bearing on where it sits on the device.

Three things follow from that, and they are the reasons for it.

A rewrite is an append plus a map repoint, so the previous copy stays intact and stays referenced until the new mapping is durable. Overwriting in place has a failure mode no checksum can catch: a write lands in a block the cache already holds clean, the machine dies before the metadata recording it dirty reaches disk, and recovery restores a map that calls the block clean while the device holds content that was never uploaded. That block then differs from the remote forever, and losing the cache silently reverts it. Appending removes the case rather than ordering around it.

Every write to the device is sequential. Appends are batched into ~1MB writes rather than issued per block, which is what makes advertising a volatile write cache worth doing, and the drive sees the one access pattern its translation layer is built for.

Allocation is a pointer bump inside a region a thread already owns, so the write path has no shared allocator and no free list.

The map is keyed by `map_unit`, a run of consecutive blocks cached and evicted together, and a mapped unit is fetched and stored whole. At one block it is exact. Above that it amplifies: a random 4K read into a 64K unit occupies sixteen times the space it asked for, since the fifteen neighbours it drags in were never wanted and may live in fifteen different segments, which turns one miss into up to fifteen extra range gets. Whether that matters is entirely a property of the workload, so `map_unit` is not a constant.

An entry covers a whole unit, so a write smaller than one cannot form an entry by itself. The blocks it does not carry are in the remote, and fetching them to complete the unit would put a remote round trip on the write path, which is what making the local disk the durability boundary exists to prevent. So it does not fetch them. The blocks land in the log and are durable and readable like any other write, and the unit is held in a side table carrying a validity bitmap for as long as it stays incomplete. At commit the unit has either been completed by later writes, and becomes an ordinary clean entry, or it has not, and its blocks drop from the cache, leaving a later read of them an ordinary miss that fetches the whole unit. That side table is bounded by the scattered writes of one flush interval rather than by the size of the cache, tens of MB at the worst rate the watermarks allow.

So a write earns clean residency exactly when it forms a complete unit, which is the read amplification above seen from the other side. It is also why a coarse map belongs to a workload doing large sequential writes: those form units by construction and keep their residency for free, while scattered sub-unit writes stay durable, upload correctly and simply do not stay cached. At `map_unit` equal to `block_bytes` every write forms a complete unit and none of this exists.

Two structures, both sized at startup, both locked, neither ever resized or reallocated:

1. The slot table, one 10 byte entry per mapped unit of the device, `(block index 40b, crc32c 32b, state 8b)`, indexed by slot so it wastes nothing
2. A forward index, open addressed, hashing block index to slot at 4 bytes an entry, ~5.3 bytes at a 75% load factor

An index entry holds a slot and nothing else, so a probe is resolved by comparing the block index in the slot table entry it names, which the lookup was about to read anyway. That comparison catches a hash collision, and the checksum beside it catches what the comparison cannot: a misdirected write lands a block that is internally consistent at the wrong slot, and `(block index || data)` is what distinguishes it.

So ~15 bytes an entry, and the number of entries is `cache_bytes / map_unit`. That product is the entire cost, and it is worth writing out because it does not extrapolate the way a per terabyte figure suggests:

```
cache    map_unit   entries    map RAM
1TB      4K         268M       ~4GB
4TB      4K         1.1G       ~16GB
4TB      16K        268M       ~4GB
100TB    64K        1.7G       ~25GB
100TB    256K       419M       ~6GB
```

Fifteen bytes is close to the floor for an entry that can detect a misdirected write, which is what Integrity exists for, so spending less means fewer entries rather than smaller ones and `map_unit` is the lever.

Which is fine, because the two regimes want different values. A working set cache of a few TB sits in front of a much larger volume and sees the page cache's miss stream, random 4K included, so it maps at one block and pays ~4GB per TB, which is a couple of percent of a host proportioned for it. A 100TB cache is a bulk tier, and a host does not have 100TB of NVMe in order to serve scattered 4K; it has it for throughput on large objects, where locality at a few hundred KB is high and the amplification is close to nothing. dm-cache defaults to 256KB blocks on the same arithmetic.

So `map_unit` defaults to auto: the smallest power of two at or above `block_bytes` whose map fits `map_ram`, a share of host memory defaulting to 3%. That lands one block on a small cache and a few hundred KB on a very large one without anyone choosing, and an explicit value overrides it in either direction. Attach reports the footprint and refuses outright when it does not fit, rather than leaving the OOM killer to discover the problem. One value covers the whole device, so a workload that is bulk in one file and random in another gets a single compromise, which open question 9 takes up.

The slot table's durable form is the device itself. Each region reserves a metadata area holding that region's slice of the table, written as the region fills, and the region header carries a monotonic sequence number so that `(region sequence, slot)` totally orders every copy of a block index and the newest wins. Checksums sit there per block whatever `map_unit` is, because Integrity below needs them at that granularity, and the rest of the entry is per unit, so the area is ~0.25% of the region at one block per entry and a little over 0.1% at 64K. There is no separate intent journal and no periodic checkpoint, because the log is one and the map is materialized from it: mount reads every region's metadata area, which is two thirds of the map's own size, and rebuilds both tables in a second or two. Heat state rides in the same entry, so it comes back with it.

The device carries a superblock holding the volume uuid, `block_bytes`, `region_bytes`, `map_unit` and the epoch the cache is working toward, the last of which is written before each root put and is what Fencing compares against the head to tell a legitimate restart from a fenced one. Everything else on the device is regions.

DeepDisk advertises a volatile write cache, so a write is acknowledged once it is in memory and only has to reach disk on `REQ_OP_FLUSH` or `REQ_FUA`. That is the same contract as any disk with a write cache. A barrier issues the buffered appends and the metadata blocks they touched and waits for one flush. One flush is enough because recovery validates rather than orders: it does not matter which of the two reached the platter first, only that a slot is accepted when the recorded index and the data's checksum agree. A slot whose metadata never landed, or whose checksum does not match, is absent on recovery, which is the correct answer for a write that was never barriered.

### Heat and eviction

Eviction follows S3-FIFO, with admission acting on blocks and reclamation acting on regions. Two of its three queues are carried: a small one everything fetched enters, and a main one that repeated use earns.

The third is not. S3-FIFO's ghost queue holds a fingerprint per unit evicted from small so that a later miss on one can skip straight to main, and at one entry per evicted unit that is a structure roughly the size of the map itself. It answers reuse distances longer than small's residency and shorter than main's, which is a real effect of unmeasured size here, so the eviction structure carries two queues and question 7 is where the third gets decided.

A block's class decides which log it lands in. Each writing thread holds three open regions, small, main and dirty, so a region ends up uniform in heat and can be dropped whole. Everything fetched on demand enters small, which holds ~10% of capacity. A hit increments a two bit counter in the entry, saturating at three, and that is the entire promotion mechanism: nothing is copied at the moment of a hit. Three is also how many reclaim passes a block outlives its last read by, since reclamation decrements as it relocates.

Reclamation frees a region whole, and it is where promotion and eviction both happen. One rule covers both classes: a live block with a counter above zero is relocated into a main region with the counter decremented as it moves, and everything else is dropped, since it is still in the remote. The decrement is what makes main a cache rather than a compactor. Without it a block promoted once is resident forever and main only ever grows, and the write path is the worst offender, because every block committing clean enters main having never been read.

Which class is reclaimed from is decided first and by budget alone: small when it is over `small_queue`, main otherwise, dirty never. That is what holds the admission filter at a fixed size, since choosing globally on heat would find newly fetched blocks the coldest thing on the device and reclaim small down to nothing.

Within that class the region chosen is the one with the fewest blocks that would survive the rule above. It is one number meaning two things at once: cheapest to reclaim, since it counts the blocks that have to be copied, and least valuable, since it counts the blocks anything has asked for. Live slots alone inverts on both, reclaiming a nearly empty region of hot blocks ahead of a full region of cold ones and copying the hot blocks for nothing. Heat is not maintained per region, because the read path bumps a counter and does nothing else, so it is computed by scanning a candidate's slice of the slot table, 80KB at one block per entry, over `reclaim_candidates` regions picked by live count, once per region allocated.

This is what keeps a bulk write from evicting a working set. A restore's regions score zero, so they are always the chosen victims, are freed without copying anything, and never age the counters of the blocks they pass over. Written data that is never read gets residency proportional to how much other cold data is competing rather than a fixed budget, and one read moves it out of that population entirely.

That is the only copying the cache does, and doing it at reclaim rather than at the hit that earned it means the copies are batched, sequential and made against a counter that has had the region's whole lifetime to be right.

Scan resistance is why. A backup, a `grep -r` or an `updatedb` walks the whole volume once, and under recency that walk evicts the working set it passes over. Here it never leaves small.

The read path does nothing but bump that counter in the slot table entry it has already loaded. There is no list to append to, no queue to reorder, no lock and no allocation, which is what the writeback path requires. That is a stronger property than a FIFO append, which would still be a shared write from every thread.

DeepDisk is a second level cache and sees the page cache's miss stream, already stripped of the short term reuse that recency exploits. Frequency based admission is the right shape for that, and it also means published hit rates for this family come from web and CDN traces and want measuring here rather than assuming.

The same reasoning locates sequentiality. The page cache's readahead has already turned a sequential stream into large requests by the time they reach us, so it is a field on the request in hand rather than a pattern to infer across requests. A large request fetches a large range and is admitted at low value, so a scan gets its readahead without occupying main.

Dirty blocks are exempt, since they are pinned until uploaded. A dirty region is not reclaimable, and once every block still live in it has committed clean the region is relabelled main in place with no copying at all, so data that was just written stays cached exactly where it landed. It enters main at a counter of zero, which is the whole of what admission does for a write: residency with no claim on it, ended by the first reclaim that needs a victim unless a read has intervened. Blocks whose units never completed are dropped at that point instead of relabelled, per the completeness rule in Cache layout. Replacement therefore governs only the clean portion of the cache, and that portion shrinks as dirty pressure climbs, so at the 85% watermark the policy is working with 15% of the device and hit rate degrades exactly when the system is already stressed.

Dirty pressure is the fraction of regions that are dirty, a count of a few tens of thousands that needs no scan.

Heat lives in the slot table entry and the slot table is materialized from the device, so a worker restart resumes with its working set intact and its counters where it left them. A host with no cache at all, after a migration or a fork, starts cold and warms by being read.

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
5. Clean shutdown, detach or an explicit `flush`, where everything dirty is uploaded and committed

A block written in the last ~1s is held back from a segment being packed, to absorb repeated overwrites. This is only an optimization to avoid redundant uploads, never a reason to defer one, so the age cap, dirty pressure, idle, shutdown and an explicit flush all override it.

When selecting which dirty blocks to pack into a segment, we prefer spatially adjacent blocks. A read fault range gets only the blocks it needs, at the same request cost as fetching the whole segment, so contiguous packing is what makes widening that range into a prefetch worth doing rather than pulling in unrelated blocks. Widening is a policy here and not a structural requirement: the cache appends exactly the blocks that came back, so a fault is free to fetch one block or a hundred consecutive ones and the difference is bytes on the wire rather than cache space.

Age is free to measure. Dirty regions are logs in arrival order, so the oldest dirty block is the head of the oldest dirty region and `oldest_dirty_age` needs no scan and no per block timestamp. Packing wants address order rather than arrival order, so the flusher gathers the dirty set and sorts it, which is a few milliseconds over a flush interval's worth of blocks.

#### Watermarks

As a fraction of the local cache occupied by dirty blocks:

1. Under 25%, lazy, only reasons 1-3 apply
2. 25-60%, steady background flush
3. 60-85%, maximum upload concurrency, and the overwrite window is ignored
4. Over 85%, incoming writes are throttled
5. Over 95%, writes stall

Dirty and clean share one device, so this ladder is also what reserves space for the read cache: the stall at 95% is the floor that keeps a clean portion at all. Tuning these for flush behaviour tunes read hit rate at the same time.

Throttling is admission control on region allocation. Below the watermark a thread whose open dirty region fills takes a fresh one from the clean pool through the replacement policy. Above it, opening a new dirty region instead waits for an upload to complete and its epoch to commit, which releases a dirty region back to the pool. Writes are then paced to exactly the rate the flusher drains at, and the clean portion is preserved for reads rather than consumed to absorb a burst. Waiters are served in arrival order.

Because releases arrive on epoch commits and commits are floored at ~1/s, the pacing is quantised: a write that has to wait waits for the next commit, so throttling shows up as a sub second tail rather than as smooth backpressure. Sustained throttling forces commits ahead of `commit_rate` for the same reason an explicit flush does, since the caller is being made to wait on durability either way.

An overwrite takes a fresh slot and leaves a dead one behind, so an overwrite heavy workload consumes cache space at its write rate rather than at its unique block rate, and reaches the watermarks sooner than its working set suggests. The consumption is transient: the region holding the dead slots is released as soon as its remaining live blocks commit, so it is bounded by roughly one flush interval of writes.

The stall is this same mechanism with nothing completing to wait on, so it is reached when uploads have stopped rather than merely fallen behind. This is why the ladder is bimodal in practice: a workload that outruns the flusher settles at the throttle watermark and stays there, and only losing the remote walks it to the top.

#### Epochs

Uploading lazily and out of order leaves the remote copy torn across many points in time, so restoring it would produce an image that never existed. Flushing is therefore epoch structured:

1. Segments are uploaded as soon as they seal, in any order
2. A delta, listing block index -> segment + offset for the blocks that moved this epoch and tombstones for the blocks that were unmapped, is uploaded under the epoch number
3. A root object is written for the epoch and the head is swapped to point at it. That swap is the atomic commit, and the durability point
4. Only then are those blocks marked clean in the cache map, making them evictable and releasing the regions holding them

Segments are immutable, and an uploaded segment is invisible until the head names a root that references it. This is what keeps the remote path off the critical path: uploads can be issued the moment a segment seals, need no ordering between them, and can be retried in any order, because the only step that has to be ordered is the small metadata write at the end. A partially uploaded epoch is harmless, as nothing ever refers to it.

The delta is proportional to the data it describes rather than to the size of the device, so committing stays cheap no matter how large the volume is. It is a sequence of typed records. A map record carries the segment ID in a header, since it is constant across the record's entries, and then the block indices in segment slot order, so the offset is implicit and only the indices are written, and those are runs of consecutive values whenever packing was contiguous. An unmap record carries extents of block indices and no segment at all. A full 4MB segment is a few hundred bytes as extents, up to ~4KB for a random scatter, and an `fstrim` releasing a large free region is a handful of extents no matter how many blocks it covers.

The unmap record is what makes discard expressible. Without it a delta can only say where a block moved to, and there is no encoding for a block that moved nowhere, so a discard has nothing to commit as. It also has to be a state rather than an absence: absent from a delta means the older mapping still stands and the lookup falls through to the tree, while unmapped means the lookup stops and the read returns zeros. The overlay and the tree fold carry the same distinction, and folding an unmap into a leaf removes the entry, which is what lets a discard covering a whole subtree collapse to one removal in its parent.

A delta records the final state of each block index it touches, at most one record per index, since the writer coalesces in memory before packing and a block written and then discarded inside one epoch commits only as unmapped. Records are nonetheless applied in file order, so a delta assembled any other way still resolves the same way.

The index that these deltas apply to is checkpointed periodically, so recovery is the newest checkpoint plus the deltas since it rather than the entire history. See Manifest below.

Recovery always restores the last committed root. Filesystem flush/FUA barriers are used as epoch boundaries where they land, since they mark points the filesystem itself considers consistent, and a barrier only costs a delta and a root write. Otherwise the epoch is cut on the age cap. Commits are rate limited to ~1/s so a bursty workload cannot commit continuously, and the head swap is conditional, see Fencing below.

#### Snapshots and forks

Immutable segments plus a versioned root make the remote copy-on-write. A write never modifies remote data, it lands in a new segment and the map is repointed, so every committed epoch stays a complete and consistent image for as long as the segments under it survive. The local cache is copy-on-write too, for a different reason and with no history kept: it appends and repoints so that a crash can never leave a block marked clean while holding content that was never uploaded, and the superseded copy is dead the moment the map moves.

That gives us, with no extra machinery:

1. Snapshots, which mark a root as retained along with the delta range it spans. Nothing is copied and nothing moves, so taking one is O(1)
2. Forks, which are a new head naming an existing root. Both heads reach every segment under that root and diverge only as they are written to
3. Going back, which is a fork at an older epoch, attached in place of the original

A fork sharing objects with its origin is why liveness is a union rather than a walk from one root. A segment is dead when no head reaches it and no retained root spans it, so garbage collection reads every head in the prefix and every retention marker, and takes the union. That is the same computation snapshots already require, applied to one more kind of reference, and it is what keeps a fork's data from being collected by whoever happens to run a pass. Concurrent writers on other heads are safe against it for free, since a writer can only reference segments an earlier root already reached or ones it wrote itself, and a pass ignores objects newer than its own start.

A read only attachment is none of those things. It holds the epoch it mounted and nothing else, so reclamation proceeds underneath it and a reader that outlives the retention window can find content it needs already collected and takes `EIO`. Holding a view for longer than that is exactly what the three above are for: attach `--snapshot`, which retains its root and delta range for as long as the snapshot exists, or fork if the view is going to be written to. The alternative, making every reader a retention root, was rejected because a forgotten read only mount would then stop reclamation the same way a forgotten snapshot does, and unlike a snapshot it has no operator visible object to find and no way to signal that it has gone away.

#### Failure and garbage

A failed upload leaves its blocks dirty and retries with backoff, so a sustained remote outage walks up the watermarks into throttling and then stalling. See Errors below for what happens at the top of that ladder.

Because the remote is copy-on-write, overwriting a block leaves the old value in place in an already uploaded segment, so dead data accumulates and segments have to be compacted. A block is dead only when no retained root references it, not simply when a newer write supersedes it, so a held snapshot pins segments that would otherwise be reclaimable. Space pinned per snapshot should be exposed as a metric, since a forgotten snapshot silently stops reclamation and grows the remote footprint without bound. See Compaction below for who does the reclaiming.

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

Everything is immutable except the heads:

```
data/seg/9F3A1C77B204E815        4MB immutable segment, self describing
data/seg/2B84D0FE61A9C33D

meta/pack/7C1E05B8AA42F961       index pages written by checkpoint @ epoch 41200
meta/pack/D3902A6F58E1740B       index pages written by checkpoint @ epoch 42200

meta/delta/0000000000042801      ~1KB, one per epoch
meta/delta/0000000000042802
   ...
meta/delta/0000000000042847

meta/usage                       what the last collection pass found, no durable state

orphan/main/42847/3f2ab19c/      a fenced writer's last writes, root and segments

meta/super                       { uuid, format_version, block_bytes: 4096 }

meta/root/0000000000042847       { epoch: 42847,
                                   writer_id: "3f2ab19c",
                                   tree: (pack D3902A6F58E1740B, off 8192, len 3400),
                                   tree_bytes: 1046528,
                                   checkpoint_epoch: 42200,
                                   delta_from: 42801,
                                   device_bytes: 109951162777600,
                                   logical_bytes: 8246337208320,
                                   superseded_bytes: 3092376453 }

meta/head/main                   { epoch: 42847 }
meta/head/dev-fork-1             { epoch: 42800 }
```

Roots are immutable and written one per epoch, and the heads are the only mutable objects. Committing is a create of the root followed by a compare and swap of one head, so a head is what fencing contends on and it stays small enough that the check is a few bytes. Every epoch that ever committed is still addressable, which is what snapshots and point in time recovery reach for.

A prefix holds one object store and any number of heads over it, so a volume is a prefix and a head name rather than a prefix alone. Heads are peers: none is the original, each advances on its own epochs, and each has its own CAS, so two writers on different heads never contend while two on the same head fence each other exactly as before. Roots, segments and packs are shared by whichever heads reach them, and nothing is copied to create that sharing.

Segment and pack IDs are random 64 bit values. Heads advance independently, so no counter can be shared between them, and a randomly drawn ID collides at a rate that never matters while a conditional create catches the case anyway. They carry no meaning and nothing sorts by them: garbage collection ranks segments by live count, and every reference to one is explicit in a delta or a page, so nothing has to guess a key.

Keys are deterministic, and the root records the current checkpoint plus the live delta range, so we never list a prefix to find objects. Listing is only for garbage collection and repair.

Because keys are deterministic, no bucket lifecycle rule may be applied to a volume prefix. A segment is written once and then only read, so age says nothing about whether it is live, and a routine expiry policy would delete data a current root still points at.

Creating a volume writes `meta/super`, then `meta/root/0` describing an empty tree, then `meta/head/{name}` at epoch 0. Every step is a conditional create, which makes the whole sequence idempotent: a create interrupted halfway is completed by running it again, and two concurrent creates resolve to one winner, with the loser finding `meta/super` already present and comparing it to decide whether it agrees. A volume is mountable only once its head exists, so a partial create reads as absent rather than as broken.

Forking is that same last step on its own. A new head is a conditional create naming a root that already exists, so it is one small put, it copies nothing, and it is O(1) in the size of the volume rather than merely cheap. Creating a fork and creating a volume differ only in whether the root it names is empty.

`meta/super` is written once at creation and never again. Block size cannot change without invalidating every mapping in the volume, so it lives in an object that nothing is allowed to rewrite, and is validated on mount.

The root carries the sizes because they change with the epoch and have to be atomic with it:

1. `device_bytes`, the capacity presented to Linux. Keeping it here makes an online grow a normal epoch commit rather than separate mutable state that can disagree with the manifest, and it means a volume mounted on another host needs nothing out of band. It must never shrink below the highest mapped block
2. `logical_bytes`, the address space actually written. This is the thin provisioned used figure, and is what a bottomless device reports as consumed rather than `device_bytes`
3. `superseded_bytes`, how much this head has orphaned since the last collection pass it saw. A repointed block already decrements its old segment's live counter, so the byte count comes free at the same moment, and compaction is excluded because relocating a live block orphans nothing

All three are per head and all three are exact. What is not in the root is a figure for the bytes stored under the prefix, and leaving it out is deliberate.

Heads share a prefix and advance independently, so no single writer knows what the prefix holds, and a per head root is the wrong place to claim otherwise. Two heads each recording a prefix total would record different ones and neither would be right. The honest division is that a head reports what it can know exactly, and everything prefix wide comes from the only thing that ever sees the whole prefix, which is a collection pass.

`meta/usage` is what a pass leaves behind: when it ran, live and dead bytes for the prefix, and per head the bytes only that head reaches. That last number is the one an operator actually wants, since it answers what deleting a fork would free, and a pass computes it for nothing because the liveness union already walks every reference. It holds no durable state, so it takes no conditional create and a stale or missing one costs a worse answer and nothing else.

The bill belongs to the provider. Every object store meters stored bytes and issues an invoice from its own count, and a number DeepDisk maintains can at best approximate one that is already computed exactly somewhere else. What DeepDisk owes is the two things the provider cannot tell you: how much of that storage is garbage, and which fork is holding it.

`superseded_bytes` against the live figure from the last pass is therefore the compaction signal, and it needs no scan. In a forked prefix it overestimates, since one head can supersede a block another still reaches, which is the right direction for a trigger: it errs toward running a pass, and the pass is what establishes the truth.

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
CAS meta/head/{name}        { epoch: N } if it still holds N-1
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

Delta and checkpoint are the two tiers, and a mount replays the delta range whole. At ~1000 objects of ~1KB issued in parallel that is a few hundred milliseconds, which is why `checkpoint_interval` bounds mount latency as well as metadata rewriting.

Mount:

```
GET meta/head/{name} then the root     two small sequential gets
GET the root page                      range get into its pack
GET meta/delta/{delta_from..epoch}     in parallel, up to 1000 small objects
replay them into an overlay
-> serving I/O

background:
  L2   100 pointers    -> sorted, coalesced -> a few range gets    ~280KB
  L1   25,600 pointers -> sorted, coalesced -> tens of range gets  ~72MB
  leaves faulted in on demand
```

The overlay is a flat in memory map of block index to segment + slot, holding every epoch committed since the last checkpoint and consulted before descending the tree. It is how the map is maintained at all times, and mount rebuilds it by replaying the deltas the running writer holds in memory. Any lookup checks the overlay, then walks the tree, faulting what is missing. An overlay entry has three states rather than two: mapped, which answers the lookup; unmapped, which also answers it, with zeros; and absent, which is the only one that falls through to the tree. Collapsing the first two would make a discard indistinguishable from never having been asked, and the tree underneath still holds the superseded mapping. It is bounded by the checkpoint interval at ~1M entries, around 10MB, and is dropped once folded in.

Interior pages are pinned rather than faulted. They are few and small, ~724KB for 1TB as a root plus 256 interior pages and ~72MB for 100TB written, so holding all of them costs nothing next to the volume they describe and makes every cold lookup exactly one fetch, the leaf. Note this scales with written data rather than presented capacity, so a bottomless volume showing 100TB with 10TB written is ~7MB. The crossover where this stops being free is closer to 1PB written, at ~720MB.

They are pulled in the background rather than blocking the mount, and a lookup that arrives before its interior page does simply faults it. The load is breadth first and bulk rather than page at a time: once a level is parsed we hold every `(pack, offset, len)` for the level below, so those pointers are sorted by pack and offset and coalesced into large range reads, merging any two separated by less than ~64KB since reading junk bytes is far cheaper than a second request. That is three dependent rounds, since a level's locations are unknown until its parent is parsed, and tens of requests, or ~700ms on 1Gbps. It is also a cold host cost only, since the tree is materialized on local disk and a remount reads the same 72MB from NVMe in ~35ms.

Leaves are always faulted, and a faulting leaf widens its fetch to its neighbours in the same pack, which are the leaves for regions written at the same time, so the extra bytes are nearly free next to the request already being made. This is the same reasoning that makes contiguous packing worthwhile for data segments. `tree_bytes` in the root is what tells an operator how much metadata the volume carries.

Index pages can be faulted in lazily, but deltas cannot, because we cannot know which of them supersede a page we have not fetched yet. Everything committed since the last checkpoint has to be in hand before the first read is served, so `checkpoint_interval` bounds mount latency as well as metadata rewriting.

### Read path

```
read LBA 9,412,096
  |- forward index -> slot -> verify (block index || data) -> serve   (common case)
  \- miss -> overlay, then the radix tree
             |- unmapped or absent -> zeros, no I/O
             \- (segment A1F3, slot 412)
                range GET data/seg/000000A1F3 bytes 1687552..1691647
                append into the open small region, repoint
```

Once warm there are no metadata requests on the read path, which is the property this layout exists to protect.

### Reverse mapping

Garbage collection needs to know which blocks in a segment are still live, and scanning the forward map to find out is far too expensive. Instead each segment carries a live block counter, decremented at epoch apply time when a block index that pointed at it is repointed, since the old value is already in hand at that moment. Garbage collection then sorts segments by live count. The counter is a cache and can be rebuilt by scanning if lost.

Snapshots complicate this, because the counter only tracks the current root while a block is dead only when no retained root references it. Computing the union of live sets across retained roots during a collection pass is fine for a handful of snapshots, and gets expensive if hundreds are held.

## Compaction

Compaction is performed by the volume's writer and by nothing else.

It has to be, because it repoints the map. Reclaiming a segment means reading the blocks in it that are still live, packing them into a new segment and changing what the manifest says about them, which is a durable change to the tree and therefore an epoch and a head swap. The single writer rule is what fencing, the conditional head swap and the whole epoch structure rest on, so the writer is the only thing allowed to do it.

Compaction therefore emits its repointing as ordinary map records in an ordinary epoch delta and rides the normal commit, needing no new object, no second CAS and no coordination beyond what a write already does. It runs inside the worker that holds the attachment, so the writer role and the attachment stay the same thing, and `POST /compact` requires the volume attached.

It yields to flushing rather than competing with it. Compaction draws from the same segment pipeline, the same upload budget and the same commit, at strictly lower priority than dirty blocks, and stops entirely above the `aggressive` watermark. Dirty pressure therefore never loses upload bandwidth to reclamation, and the ladder in Watermarks stays a function of the workload rather than of what garbage collection happened to be doing.

A compacted segment is deletable once the epoch that stopped referencing it has committed and no retained root spans an epoch that still referenced it. Pack compaction is the same operation against the manifest and rides a checkpoint instead of an epoch, grouping live pages by tree region as described in Structure.

What is still open is the trigger. `superseded_bytes` against the live figure `meta/usage` recorded says how much garbage has accumulated without scanning anything, but the ratio at which a pass is worth running, and how that interacts with the retention window that is keeping the garbage alive, are unset.

## Concurrency

The read path has to be lock free and allocation free, and everything else has to stay out of its way.

One thread per ublk queue, and a request is handled to completion on the thread it arrived on. There is no handoff to an owning shard, so no queueing inside the daemon and no cross core wakeup on a hit. A miss issues its device or network read through that thread's `io_uring` and the thread continues, so a request waiting 50ms on the remote occupies no thread.

The forward index is bucketed to a cache line and read under a per bucket seqlock, so a lookup is one line, no atomic write and a retry only when a mutation raced it. The slot table is a flat array, so an entry read is one more line, and the frequency bump the read path performs is a relaxed store into a line it already holds. Mutations take the bucket.

Regions make the write path shared nothing. Each thread owns its own open region per class, so appending is a thread local pointer bump against a buffer that was preallocated at startup. Acquiring a fresh region and reclaiming a spent one are both off the I/O path.

The overlay is immutable and published by a release store, the same way the tree is. The flusher builds the next version by applying the epoch's delta and publishes it at commit, and readers take the pointer and hold it for the lookup. Rebuilding the whole thing every commit costs a few milliseconds against ~1M entries and two copies is ~20MB, which is cheaper in both effort and risk than making a concurrently mutated map correct. Retired overlay and tree versions are reclaimed by epoch: a thread publishes the version it holds in its own slot, and a retired version is freed once every thread has moved past it, so nothing on the read path waits for a grace period.

A write concurrent with a read of the same block is not ordered by the block layer and does not need to be, but it must not tear. Log structure gives that for nothing: the writer appends elsewhere and repoints, so the reader resolves to one slot or the other and reads a whole block either way.

One flusher per volume. Packing, uploading and committing are serial with respect to each other by the single writer rule, uploads are concurrent within an epoch, and the only genuinely serial step in the system is the head swap. Everything else scales with threads.

All of it is preallocated and locked. The slot table, the forward index, the region buffers and the request pool are sized at startup from the cache device and never grow, and everything that does allocate, the remote client included, draws from the process arena described in Device. In flight remote requests are bounded so the arena can be sized against them.

## Discard

`REQ_OP_DISCARD` unmaps rather than writes. The manifest entry is removed, the cache slots holding those blocks are marked dead, and the segment that held the data has its live counter decremented through the same reverse mapping path an overwrite uses. `logical_bytes` drops immediately and `superseded_bytes` rises by the same amount, while the storage itself is only released when those segments are compacted.

A discard of a block that is still dirty and unuploaded drops it from the segment being packed, so it never leaves the host. A discard of one already packed into a sealed segment cannot, so that segment ships with a block nothing will ever reference and the delta records the unmap, which is correct and merely wasteful for as long as it takes compaction to notice.

Because an absent entry already reads as zeros, DeepDisk can honestly report that discarded blocks return zeros deterministically. That is worth advertising, since it lets the filesystem skip zeroing ranges it has just released, and it makes `REQ_OP_WRITE_ZEROES` the same operation as discard.

Discards arrive in floods, since `fstrim` releases everything a filesystem is not using in a single pass, so they have to be range operations. This is where the radix tree pays off a second time: unmapping a large range drops whole leaves and whole interior entries, so the cost is proportional to the tree nodes the range spans and not to the blocks in it, and a discard covering an entire subtree is one entry removed from its parent.

Unmapping commits like a write, as an entry in the epoch delta, so a discard is durable only once its epoch commits and is never visible remotely ahead of the root that records it.

## Remote

The S3 API is the interface and no provider behind it is assumed.

A volume's identity is its bucket, prefix and head name, fixed at creation. A prefix holds one object store and every head over it, so forks of a volume share an identity down to the last component and share credentials with it exactly. How a host reaches that bucket is not part of the identity, so endpoint, region, addressing, TLS trust and credentials are supplied to create and attach and hold for the life of the attachment. Both calls exercise them before returning, writing a `_deepdisk` probe object under the prefix and deleting it, or listing instead when the attach is read only. Credentials are held in memory and never returned by a get, the rest comes back from `GET /v1/volumes/{id}`.

A 403 after a successful attach means revoked or expired rather than unlucky, so it is logged as an event immediately instead of disappearing into upload retries, and uploads then fail into the watermark ladder as any unreachable remote does. A read only attach needs get and list alone, which makes that flag enforceable at the boundary rather than only by us. A fork reads and writes the same prefix its origin does, so one credential covers every head over one object store, and access cannot be granted per head.

Credentials are assumed to be long lived, and that is a limitation rather than an oversight. DeepDisk does not obtain or refresh them: there is no STS assume role, no IMDS, no container identity provider and no refresh timer, so an attachment is only as durable as the credentials it was given, and short lived session credentials will expire underneath a running volume and walk it up the watermark ladder to a stall. An attachment is expected to outlive its credentials only if those credentials do not expire.

Rotation is still a thing operators do, so `POST /v1/volumes/{id}/credentials` re-supplies them on a live attachment. It exercises the new values against the prefix the same way attach does, replaces them in memory only on success, and takes effect on the next request without disturbing I/O, so rotating a key does not mean a detach and attach cycle. Automatic refresh is the obvious thing to build on top of it and is out of scope here.

Encryption is out of scope, and it composes rather than being missing. A deployment that needs client held keys puts `dm-crypt` between the filesystem and `/dev/deepdisk0`, so DeepDisk sees ciphertext blocks and never sees a key; one that needs encryption at rest against the storage provider turns on SSE-S3 or SSE-KMS on the bucket, which also covers the `meta/` objects `dm-crypt` cannot reach. Both are standard, both are someone else's problem to get right, and holding keys is a liability worth not taking on. Object formats carry a version field so this can be revisited, but nothing is reserved for it.

Five operations are required: range get, put with conditional create, put conditional on the current version for the head swap, delete, and list by prefix. Conditional support is the discriminator, and a backend that accepted the header and ignored it would break fencing silently. So create reuses `_deepdisk` to prove it, and the proof has to be concurrent rather than sequential, because the property fencing needs is atomicity under contention and that is not what a sequential check observes. A backend can honour `If-Match` request by request and still resolve a tie by last write wins, or serialise per key only within one gateway node, and every one of those passes a probe that writes twice and waits.

Three rounds, each firing several conditional writes at once against the same base version, asserting exactly one succeeds and the rest are refused. Conditional create is proved the same way, several creates of one key with exactly one winner. Anything else refuses the volume. Paying that once, before there is data on the bucket, is what lets every later write assume it, and it is the difference between discovering a backend's limits at create time and discovering them during a split brain.

## Integrity

Two layers check different failures and neither substitutes for the other.

The object layer belongs to S3. It checksums on upload, verifies at rest and repairs from redundancy, so object bit rot is its problem, and DeepDisk supplies CRC32C in the API's own checksum field rather than wrapping an object in a second scheme of its own. Two things sit outside that reach.

The local cache is the durability boundary, holding acked data before it exists remotely at all and clean data indefinitely. Device ECC catches most of what goes wrong there, but a misdirected write lands a block at the wrong LBA and produces one that is internally consistent and simply wrong.

The larger surface is our own translation. DeepDisk maps addresses, so a delta that encodes an extent off by one, or a leaf that decodes wrong after compaction, returns another block's data: intact, correctly checksummed by S3, and not what was asked for.

The block layer covers both, by checksumming `(block_index || data)` rather than the data alone, which turns a bit rot check into a misdirection check: reading block 500 and receiving block 900 fails even though block 900 is intact. CRC32C is 4 bytes per 4K block, 0.1%, and a hardware instruction.

There is one such checksum per block and it is computed once, when the block value first exists, then carried. A block written locally is checksummed as it enters the cache and the segment that later ships it reuses that value; a block fetched from a segment arrives with its checksum and keeps it. Verification happens many times, computation once.

It is stored wherever a reader needs it, which is three places holding one value:

1. In a segment, at a fixed stride alongside each slot, for a block arriving from the remote
2. In the region metadata area on the cache device, the durable local copy, which is what recovery validates slots against
3. In the slot table entry in RAM, so verifying a cache hit costs no request, no seek and no extra cache line

The third is a cache of the second rather than an independent check, and it is why the map entry carries 4 bytes it could otherwise skip. Reading the device copy on every hit would mean a second 4K read for each one the cache serves, which is the cost the cache exists to remove.

Checksums never enter the manifest, where per block entries would break extent encoding and cost ~1GB per TB.

Verifying every hit this way holds at one block per map entry, which is the working set case. Above it the coverage thins deliberately: per block checksums stay in the region metadata area, the unit is verified end to end when it is filled, and a later hit is covered by the block index comparison and by the drive's own ECC. Verifying a 4K read against a checksum over a 256KB unit would cost a 256KB read, and reading a metadata block per hit would double the I/O the cache exists to avoid, so a coarse map buys its capacity partly with verification depth. Manifest pages carry their own over `(page identity || content)`, since a corrupted leaf misdirects everything beneath it.

A clean block that fails verification is discarded and refetched, so local corruption self heals and the read still completes. A dirty block that fails is real loss, since it was never uploaded, and returns `EIO`.

## Errors

Reads and writes fail differently, because the truth is in different places. A write is durable locally the moment its barrier completes, so an unreachable remote costs us space rather than data, and the response is to accumulate dirty blocks and walk up the watermarks. A read that misses the local cache has no local truth to fall back on, so for it an unreachable remote is a real failure.

A missing read retries with backoff against a deadline, 60s by default, then returns `EIO`, which surfaces an error the layers above can report and act on. The deadline is generous because most outages are shorter than it and an `EIO` is expensive, often a read only remount.

At the top of the watermark ladder the stall is held indefinitely, because the data is already durable and the workload is only being asked to wait, so the outage costs latency and leaves integrity intact. A deadline after which writes take `EIO` is available as policy for workloads that prefer to fail fast.

## Fencing

Running one writer is the application's guarantee to make, and it is the operational requirement DeepDisk places on whoever deploys it. What follows detects a second writer and contains it. It does not make concurrent mounts work, and it does not recover what the second writer costs, so an orchestrator that fails over on unreachability alone will eventually lose data here. Unreachable and dead are different states, and only the deployment can tell them apart.

Fencing lives entirely on the remote and the epoch is the token. Every head swap is conditional on that head still holding the epoch this writer last committed, so two writers advancing the same head from the same epoch resolve into one winner and one conditional failure. Writers on different heads never contend at all, since each swaps its own object. `writer_id` is a fresh 128 bit value per incarnation, so it never repeats and a writer can always recognise its own bytes.

Every object is a conditional create as well, since keys are deterministic and a second writer computes them identically, and under last write wins the winner's committed epoch could otherwise point at the loser's bytes. What a create conflict means depends on how the key was chosen, because fencing belongs where two writers necessarily contend on the same key rather than where they collide by accident. Segments and packs draw from a shared ID space and so use *different* keys from it, and a writer that fenced on those would have writer A take `A1F4` while B takes `A1F5` and each then fence itself on the other's key, leaving the volume with no one serving it.

So the rule splits by how the key was chosen:

1. Segments and packs, keyed by an allocated ID. A conflict is read back. Our own `writer_id` means the retry already landed. Anyone else's means draw the next ID and carry on, since IDs are opaque and burning one costs nothing
2. Deltas and roots, keyed by epoch. Both writers compute the identical key, so there is exactly one contention point per epoch with exactly one winner. Our own `writer_id` is again a retry; anyone else's is a real fence, and it fences on the first object of the epoch rather than at the root put
3. The head, where the CAS is decisive

A head swap whose response is lost is not a fence either. The writer re-reads the head: if it names this writer's root at the epoch being committed, the swap landed and the epoch is committed. Anything else means another writer moved it and this one is fenced. That is the one head read a writer performs after mount, and it exists to resolve an unknown outcome rather than to retry from a newer epoch.

Segments a fenced writer uploaded before losing are left behind under `data/seg/`, referenced by no root. Garbage collection reclaims them on liveness like any other dead object, which is the correct treatment because they genuinely are dead. This is the one case where an unreferenced segment is expected rather than a symptom.

The loser holds an invalid copy. Not a stale one that could be caught up and not a divergent one that could be merged, since its blocks were written against allocation state the winner never saw. So it flushes its dirty blocks to `orphan/{head}/{epoch}/{writer_id}/`, fails outstanding and subsequent I/O with `EIO`, and drops the mount. Once fenced it never reads the root again, which is the one step that would turn a fence back into a race, and this is why the head read above is scoped to an in flight swap whose outcome is unknown rather than being a general retry. That prefix holds a private root and its segments, so an operator can point a head at it and pull files out by hand. It is never applied onto the winner. Garbage collection also never touches it, since its liveness rules would read an unreferenced root as dead, so the prefix stays until an operator removes it and a split brain leaves a trace that does not expire on its own.

The same rule runs at mount, since a writer can be fenced and then crash before acting on it. An ordinary restart keeps its cache, because the dirty blocks there are acknowledged writes that have not been uploaded, so one number separates the two cases: the cache superblock records the epoch it is working toward, written before the root put, which holds it at or ahead of the root for a legitimate writer even when a crash lands between them. A head ahead of the cache means another writer committed, and the whole cache is orphaned and dropped, since entries are keyed by block index and the winner may have rewritten any of them. This depends on the epoch being strictly monotonic for the life of a head, which it is: a fork at an older epoch advances a head of its own, so no head ever moves backwards.

This protects the integrity of the remote. Two writers can never produce a torn or interleaved image, and a zombie serving reads mutates nothing. The flush window is separate: the writes the loser acknowledged and had not yet uploaded are unreachable from the winner, so split brain converts the age cap from an RPO window into a data loss window and one knob bounds both. Surviving a host means uploading, and fencing is orthogonal to it.

The preconditions all of this rests on are proven against the backend when the volume is created, so fencing assumes them at runtime rather than discovering at the moment of a split brain that they were never honoured.

## Observability

Four numbers are what to alert on. Everything after them is for working out what happened once one of them moves.

`oldest_dirty_age` against `age_cap` is the live RPO: what a host failure would cost right now, as a maximum rather than an average. A flusher keeping up holds it near the cap and one falling behind walks it past.

`dirty_fraction` is the share of regions held dirty, so it is position on the watermark ladder and predicts a stall instead of reporting one.

`cache_hit_rate` explains latency. Everything above DeepDisk feels this number and very little else.

`superseded_bytes` is what deferring compaction has cost since the last pass, and it is in the root, so reading it scans nothing. `meta/usage` carries the live figure it is measured against, and how stale that figure is says how much to trust the ratio.

Reads are counted by where they were served as well as by how long they took, because that decomposition says what to do. A local hit, a lookup plus a segment range get, and a cold lookup that also faulted a leaf are three different problems with three different fixes, and a p99 distinguishes none of them.

Latency is a histogram per stage rather than one across all of them, since the stages differ by three orders of magnitude and shared buckets would resolve none of them usefully:

```
read, local hit        ~100us     NVMe
read, remote fetch     20-100ms   the range get, plus a leaf fault when cold
read, device total     -          what ublk returns, the number the filesystem feels
barrier                ~1ms       REQ_OP_FLUSH to durable on local disk
segment upload         -          seal to acknowledged
commit                 -          delta, root and head swap, floored by commit_rate
checkpoint             -          fold and pack write
mount                  -          by phase, against the checkpoint interval that sets it
```

The barrier histogram is the one that checks the design rather than the deployment. An `fsync` above DeepDisk completes at local disk speed and never waits on the remote, so remote latency appearing in this distribution means that separation has broken somewhere.

Saturation is counted beside it: segments awaiting upload, requests in flight against the volume's upload budget, and overlay entries against the checkpoint bound. Time spent throttled or stalled belongs here too, since it is delay the application feels that no device latency accounts for.

The cache has three of its own. Bytes relocated per second is the write amplification log structure costs, all of it survivors copied out at reclaim, and it is the number that says whether the region size is wrong. Forward index occupancy against its capacity is the second, since the table is fixed at startup and a workload that fragments the cached set does not get to grow it. The third is bytes dropped at commit for never completing a unit, which is zero whenever `map_unit` is one block and is the direct measure of a coarse map costing more than it was chosen to save.

Then the slow signals, all of them from the last collection pass: bytes pinned per snapshot, since a forgotten one silently stops reclamation, and bytes reached only by each head, since a forgotten fork does the same thing and is harder to notice.

Some conditions matter more than any gauge and are logged as events as well as exposed as state: a fence, a checksum failure, entering or leaving a stall, an upload that failed after retries, credentials refused after a successful attach, and the existence of an orphan prefix. A fence carries a gauge of its own so it can be alerted on directly, since it is the one state where nothing recovers without an operator.

Prometheus text at `GET /v1/metrics` on the control socket, labelled by volume, which is what `deepdisk metrics` prints.

## Verification

A block device fails by returning the wrong bytes, quietly, to a filesystem that trusts it. Everything in this document that argues an epoch commit is atomic, that a fence resolves to one winner, or that a crash loses only what was never barriered, is prose until something mechanically tries to break it. Two harnesses do that, and they answer different questions.

### Deterministic simulation

The whole daemon runs inside a simulated world driven by one seed. Every source of nondeterminism is injected rather than ambient: the clock, the cache device, the remote, thread interleaving and the completion order of everything in flight. A run is then a pure function of its seed, so a failure found on seed 91,824,663 is reproduced exactly by replaying it, and a fix is verified against the same seed rather than against the absence of a recurrence.

The simulated cache device is an adversary rather than a mock. It reorders completions, tears writes at sector boundaries, loses everything not flushed, corrupts blocks at chosen rates and cuts power between any two operations, including between a data write and the metadata block that describes it. The simulated remote is the same: arbitrary latency, request reordering, 500s and 503s, throttling, conditional create races, and the response to a head swap being lost after the swap has already taken effect.

A reference model runs beside it, a plain map of block index to bytes, along with the set of states a crash could legally leave. The checker asserts after every operation that what the device returns is what the model holds, and after every crash that it is one of the states the model permits. That is what turns "recovery restores the last committed root" from a claim into a property.

What it is aimed at, in order of how much it would cost to find in production: a crash at any point in the commit sequence, two writers racing the same head through conditional creates and a CAS whose outcome is unknown, cache recovery from partially written region metadata, unmap ordering inside an epoch, and the watermark ladder under a remote that has stopped answering.

This constrains the architecture rather than testing it afterwards, in the same way the allocation arena does. Nothing may read a clock, spawn a thread or open a socket except through an interface the simulator can substitute. Retrofitting that is a rewrite, so it is a rule from the first commit.

### Trace replay

Correctness is what simulation settles. Hit rate is what decides whether the product's claim is true at all, and it wants real traces rather than synthetic ones.

The eviction policy runs standalone against captured block traces, with no daemon, no kernel and no remote, reporting hit rate against cache size. Public block traces are captured at the block layer and are therefore already the post page cache miss stream DeepDisk sees, which is the awkward part of measuring a second level cache and is already solved.

It is cheap and it comes first, because it is the only experiment that can report a result before anything is built, and the only one whose answer could say the design is wrong rather than mistuned. It settles `small_queue`, whether the ghost queue earns a structure the size of the map, where the `map_unit` crossover sits and how much relocation a given `region_bytes` costs, which is four of the open questions below from one harness.

## Configuration

```
volume, fixed at creation
  block_bytes           4096      changing it invalidates every mapping
  leaf_blocks           4096      manifest leaf coverage, 16MB of address space
  cache_device          -         raw device or partition
  remote                -         bucket, prefix and head name, the volume's identity

remote access, supplied per attach
  endpoint              -         any host speaking the S3 API
  region                -         signing region, when the backend uses one
  addressing            vhost     path only when stated, never probed for
  ca_bundle             system    trust for a private endpoint

volume, tunable
  device_bytes          -         presented capacity, grows only
  segment_bytes         4MB       upload unit, 4-8MB
  age_cap               10s       oldest dirty block before a flush, the RPO knob
  idle_seal             250ms     quiet period after which a partial segment seals
  overwrite_window      1s        holds back very recent writes, always overridable
  commit_rate           1/s       floor on the interval between epochs
  checkpoint_interval   1000      epochs between tree folds, also the mount knob
  read_deadline         60s       before a missing read returns EIO
  write_deadline        off       before a stalled write returns EIO
  upload_budget         all       static, this volume's share of the host's uplink
  compaction_budget     all       the same

watermarks, as a fraction of the cache held dirty
  steady                25%       background flush begins
  aggressive            60%       maximum upload concurrency, overwrite window ignored
  throttle              85%       allocating writes wait on flush completions
  stall                 95%       nothing completing, writes blocked

cache
  cache_bytes           device    how much of the device to use, read at attach
  region_bytes          32MB      allocation and reclamation unit, fixed at format
  map_unit              auto      bytes per map entry, fixed at format, from map_ram
  small_queue           10%       of capacity, the admission filter
  reclaim_candidates    8         regions scanned for heat before a victim is chosen

manifest
  coalesce_gap          64KB      range reads closer than this are merged

host
  map_ram               3%        of host memory, what auto sizes map_unit against
```

Two of these are the ones worth reaching for. `age_cap` sets how much data a host failure costs, and `checkpoint_interval` sets both how much metadata is rewritten and how long a cold mount takes. The rest have defaults that should hold.

Credentials are missing from this table on purpose. They belong to the attach that supplied them rather than to the volume, so they have no default and no stored value, see Remote above.

The fixed settings are fixed for different reasons. `block_bytes` is recorded in `meta/super` and validated on mount, since every mapping in the volume is expressed in it, and `leaf_blocks` is structural in the remote, so changing it means rebuilding rather than setting a flag. `region_bytes` and `map_unit` are recorded in the cache superblock and are local only, so changing them means reformatting the cache device, which costs a cold cache and nothing else. `cache_bytes` sits in the same group but moves freely, since using fewer or more regions of a device changes no layout.

Nothing in this table sets the daemon's memory footprint directly, because it is derived: `cache_bytes / map_unit` entries at ~15 bytes each, allocated and locked at startup. `map_ram` is the constraint that picks `map_unit` rather than the other way around, which is what keeps the footprint a decision rather than a consequence of buying a bigger cache device.

## Open questions

1. Compaction now has an owner, a priority and a signal, but not a trigger. `superseded_bytes` against the live figure from the last pass gives the garbage ratio without scanning anything, and the ratio at which a pass is worth running is unset, as is how that interacts with the retention window keeping the garbage alive
2. Snapshot liveness is a union across retained roots, fine for a handful and expensive at hundreds. Liveness may need to be tracked per snapshot, or snapshot count may need a bound
3. Leaf size is fixed at 4096 blocks, which makes a dense leaf 24KB, and that one number drives two costs. A checkpoint over a badly fragmented region writes roughly 6x the bytes that changed, and it rules out a key value backend like FoundationDB, which recommends values under ~10KB and would otherwise be attractive for mount time and for the RPO floor that rate limiting root writes imposes. Leaf size should be tunable, and the dense threshold may want to split a leaf rather than densify it
4. Compression is unaddressed. It belongs at the segment boundary, which is also where it collides with range gets, since a read fault has to decompress a slice rather than the whole object. Encryption above rules it out in practice, since ciphertext does not compress, so this is only worth taking up for a deployment that has chosen not to encrypt
5. Roots and their deltas accumulate at one per epoch, so something has to expire them. A retention window bounds it, ~8,600 roots and deltas a day at 10s epochs, a few tens of MB, but the window and whether it is time or count based are unset. It is now load bearing beyond storage cost, since it is also the bound on how long a read only attachment can outlive reclamation before it takes `EIO`
6. Heads over one prefix cannot be separated. Access is granted per prefix, so a fork cannot be handed to someone the origin is withheld from, and moving a fork to another bucket means materialising it by copying everything it reaches. Both are the price of sharing objects rather than copying them, and whether a `promote` that materialises a head into a prefix of its own is worth building is unanswered
7. Hit rate is unmeasured. The eviction structure is chosen on the shape of the workload rather than on numbers from it, and the small queue size is a guess until trace replay has run. Whether the ghost queue earns a structure the size of the map is the first thing it should settle
8. Region size trades reclamation granularity against relocation cost and 32MB is a guess. Too large and a region comes up for reclaim holding survivors worth copying; too small and the region table and the per region metadata overhead grow. Bytes relocated per second is the metric that settles it and there is nothing to measure yet
9. `map_unit` auto sizing picks one granularity for a whole device, which is a single compromise for any workload that is bulk in one file and random in another, and 3% of host memory is a guess at the budget. It costs on both sides of that compromise, amplifying a small read and denying clean residency to a small write, and the second is the sharper edge because the data is already local when it is discarded. Typing regions by granularity as well as by class would resolve it, promoting a proven hot block out of a coarse region into a finely mapped one and giving a sub-unit write somewhere to complete, but the promotion policy, the split between the two pools and whether the win justifies two mapping paths on the read path are all undesigned
10. Cold start after a fork or a migration is unmeasured. A fork warms by being read, and whether that is fast enough decides if a published working set summary and a prefetcher to consume it are worth their own object, encoding and budget
11. The map entry carries a checksum only because the durable copy on the device cannot be read without a second I/O. A drive formatted with per LBA protection information returns data and its guard in one command, which would take the entry to 7 bytes, ~2.5GB per TB, with hardware checking and a reference tag that catches misdirection natively. It is strictly better where it exists and it is a hardware requirement rather than an option, so the question is whether requiring that class of drive is acceptable
