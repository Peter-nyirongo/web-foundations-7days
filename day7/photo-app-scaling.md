# SnapShare — Scaling Plan

This document is the Day 7 assignment for the SnapShare photo-sharing application.
It contains the assumptions, traffic and storage calculations, workload classification,
storage recommendations, a text-based architecture diagram, component explanations,
the photo-upload flow, and the main trade-offs considered during the design.

---

## 1. Starting Facts and Assumptions

### Given facts (from the assignment)

- Registered users: **10,000,000**
- Daily active users (DAU): **10% of registered users**
- Each DAU uploads **1 photo per day**
- Each DAU views **50 feed pages per day**
- Average original photo size: **2 MB**
- Each photo also has a **50 KB thumbnail**
- Peak feed traffic is **5× the average traffic**

### Additional assumptions used in this plan

- **DAU calculation:** DAU = 10% × 10,000,000 = 1,000,000 users per day.
- **Daily photo uploads:** 1,000,000 DAU × 1 photo/DAU = **1,000,000 photos per day**.
- **Daily feed page views:** 1,000,000 DAU × 50 pages/DAU = **50,000,000 feed pages per day**.
- **Time base:** All per-second calculations use a 24-hour day of **86,400 seconds**.
- **Average traffic:** Traffic is assumed to be spread evenly across the 24 hours for the
  purpose of estimating an *average* per-second rate. Real traffic is bursty, which is why
  the 5× peak multiplier is applied separately.
- **Peak traffic:** Peak feed views per second = average feed views per second × 5.
- **Storage horizon:** Storage estimates assume uploads continue at the same daily rate for
  **365 days** and that every uploaded photo is kept for the full year.
- **Thumbnails included:** Yes — storage figures are calculated for both original photos
  and their 50 KB thumbnails.
- **Excluded from storage figures:** Database metadata, backups, read replicas, temporary
  processing files, CDN edge caches, and other operational copies are **not** included in
  the baseline storage numbers. They are mentioned separately as overhead.

---

## 2. Traffic and Storage Calculations

### A. Daily Active Users

```
DAU = 10% × 10,000,000
DAU = 0.10 × 10,000,000
DAU = 1,000,000
```

**SnapShare has 1,000,000 daily active users.**

### B. Photo Uploads Per Second

```
Daily photo uploads = 1,000,000 DAU × 1 photo/DAU
Daily photo uploads = 1,000,000 photos/day

Average uploads per second = Daily photo uploads ÷ 86,400
Average uploads per second = 1,000,000 ÷ 86,400
Average uploads per second ≈ 11.57 uploads/second
```

This is an **average rate** used for capacity planning. In reality, uploads are not
distributed evenly — they cluster around peak hours — so the system must be able to
absorb bursts well above 11.57 uploads/sec.

### C. Feed Views Per Second

```
Daily feed page views = 1,000,000 DAU × 50 pages/DAU
Daily feed page views = 50,000,000 pages/day

Average feed views per second = Daily feed page views ÷ 86,400
Average feed views per second = 50,000,000 ÷ 86,400
Average feed views per second ≈ 578.70 views/second

Peak feed views per second = Average × 5
Peak feed views per second = 578.70 × 5
Peak feed views per second ≈ 2,893.52 views/second
```

A feed page view generates far more traffic than a photo upload because each feed page
contains many photos (metadata, URLs, and eventually the image bytes themselves). One
upload produces one photo, but one feed page view triggers reads of many photos’ metadata
and, typically, many image files. That is why the read side dominates the workload.

> Note: A *feed page view* is not the same as an *individual photo view*. A single feed
> page view loads many photos at once.

### D. Photo Storage Per Year

All storage figures below use **decimal (SI) units** consistently:
1 KB = 1,000 bytes, 1 MB = 1,000 KB, 1 GB = 1,000 MB, 1 TB = 1,000 GB.

```
Original photo size = 2 MB
Thumbnail size      = 50 KB = 0.05 MB

Original photos per day:
  1,000,000 × 2 MB = 2,000,000 MB = 2,000 GB = 2 TB

Original photos per year:
  2 TB × 365 = 730 TB

Thumbnails per day:
  1,000,000 × 0.05 MB = 50,000 MB = 50 GB

Thumbnails per year:
  50 GB × 365 = 18,250 GB = 18.25 TB

Combined (originals + thumbnails) per year:
  730 TB + 18.25 TB = 748.25 TB
```

| Item                     | Per Day    | Per Year      |
|--------------------------|-----------:|--------------:|
| Original photos          | 2 TB       | 730 TB        |
| Thumbnails               | 50 GB      | 18.25 TB      |
| **Combined (baseline)**  | 2.05 TB    | **748.25 TB** |

These figures are **baseline estimates** for the image files alone. Production storage
requirements will be higher because of:

- Database rows storing photo metadata (user id, timestamps, captions, tags, etc.).
- Backups and point-in-time recovery snapshots.
- Read replicas of the database.
- Multiple copies of each object for durability (e.g., 3-copy replication in object storage).
- Temporary files during thumbnail processing.
- CDN edge caches and other operational overhead.

---

## 3. Is SnapShare Read-Heavy or Write-Heavy?

**SnapShare is a read-heavy system.**

Evidence from the calculations:

- Writes (uploads): **1,000,000 per day**
- Reads (feed page views): **50,000,000 per day**
- Read-to-write ratio: **50 : 1**

Definitions:

- A **write** is an operation that creates or changes data — in SnapShare, this is
  uploading a new photo and recording its metadata in the database.
- A **read** is an operation that retrieves existing data — loading a feed, fetching
  photo metadata, or downloading a photo or thumbnail.

Because every upload is read many times (each photo appears in the feeds of the
uploader’s followers), reads dominate the workload by a large margin.

### Effect on system design

The architecture must **prioritize efficient reads**:

- A **CDN** delivers cached photos and thumbnails close to users, removing most image
  traffic from the application servers.
- A **cache** (e.g., Redis/Memcached) stores hot feed data and metadata so the database
  is not hit on every page view.
- **Database read replicas** absorb the bulk of read queries, leaving the primary
  database free to handle writes.
- Uploads still need to be reliable, but they represent a much smaller fraction of the
  total traffic, so the write path can be simpler and more synchronous than the read path.

---

## 4. Why Photos Should Not Be Stored Inside the Database

Storing large image files directly inside a relational database is a bad fit for
SnapShare for several reasons:

- **Database size explodes.** At ~2 TB of new photos per day, the database would grow
  far too quickly, making routine operations slow and expensive.
- **Storage and backups become very expensive.** Database storage is priced higher than
  object storage, and every backup now has to copy terabytes of image bytes instead of
  just metadata.
- **Extra load on database operations.** Reading and writing large blobs consumes CPU,
  memory, and I/O that should be used for query processing.
- **Backup and recovery become slow and complicated.** Restoring a multi-terabyte
  database takes much longer than restoring a small metadata database.
- **Hard to scale photo delivery independently.** A CDN cannot efficiently cache data
  that lives inside database rows, and the application servers end up proxying huge
  amounts of image traffic.

### Recommendation

- Store **original photos and thumbnails in object storage** (e.g., S3-compatible
  storage). Object storage is cheap, highly durable, and designed for large binary files.
- Store **photo metadata and references** (object keys or signed URLs) in the relational
  database. The database remains small and fast.

Databases *can* store images as BLOBs, but for a photo-sharing app at this scale,
object storage is the clearly better choice.

---

## 5. System Architecture (Text Diagram)

```
                         +------------------+
                         |   Users / Apps   |
                         | (mobile & web)   |
                         +--------+---------+
                                  |
            +---------------------+----------------------+
            |                                            |
            v                                            v
+--------------------+                       +---------------------+
|        CDN         |                       |   Load Balancer     |
| (photos & thumbs)  |                       +----------+----------+
+--------------------+                                  |
            ^                                +----------+----------+
            |                                |                     |
            |                       +--------v--------+   +--------v--------+
            |                       |   App Server 1  |   |   App Server N  |
            |                       +--------+--------+   +--------+--------+
            |                                |                     |
            |                                +----------+----------+
            |                                           |
            |                     +---------------------+---------------------+
            |                     |                     |                     |
            |                     v                     v                     v
            |            +----------------+    +----------------+    +-----------------+
            |            |     Cache      |    |   Primary DB   |    |  Read Replica   |
            |            | (Redis/Memc.)  |    |    (writes)    |    |     (reads)     |
            |            +----------------+    +----------------+    +-----------------+
            |
            |            +-----------------+       +-----------------+
            +------------+ Object Storage  +<------+ Thumbnail       |
                         | (originals &    |       | Worker          |
                         |  thumbnails)    +<------+                 |
                         +-----------------+       +--------+--------+
                                                            ^
                                                            |
                                                   +--------+--------+
                                                   |  Message Queue  |
                                                   | (thumbnail jobs)|
                                                   +-----------------+
```

The diagram shows two main flows:

- **Upload flow:** Users → Load Balancer → App Server → Object Storage (original photo)
  → Primary DB (metadata) → Message Queue → Thumbnail Worker → Object Storage (thumbnail).
- **Feed-reading flow:** Users → CDN (cached images) and Users → Load Balancer → App
  Server → Cache / Read Replica (metadata and feed data).

---

## 6. Architecture Components

Each component below is described in exactly one sentence, specific to SnapShare.

1. **Users / mobile & web clients** — End users who upload photos and scroll through
   their personal feeds on phones or browsers.
2. **CDN** — Caches original photos and thumbnails at edge locations so SnapShare can
   serve billions of image requests without hitting the origin servers.
3. **Load balancer** — Distributes incoming upload and feed requests across multiple
   application servers so no single server becomes a bottleneck.
4. **Application servers** — Run SnapShare’s business logic: authenticating users,
   validating uploads, building feeds, and coordinating reads and writes.
5. **Cache** — Stores hot feed data and photo metadata in memory so repeated feed views
   do not hit the database on every request.
6. **Primary database** — The single source of truth for photo metadata and user data,
   handling all write operations.
7. **Database read replica** — A copy of the primary database that serves read queries
   such as feed lookups, taking read load off the primary.
8. **Object storage** — Holds the original photos and their thumbnails as durable,
   cheaply stored binary files addressed by object keys.
9. **Message queue** — Holds thumbnail-generation jobs so uploads can return quickly
   while thumbnails are created in the background.
10. **Thumbnail worker** — Pulls jobs from the queue, resizes the original photo into a
    50 KB thumbnail, and writes the thumbnail back to object storage.

---

## 7. Photo Upload Flow (Step by Step)

1. The user selects a photo in the SnapShare app and taps **Upload**.
2. The upload request travels over the internet to the **load balancer**, which forwards
   it to one of the **application servers**.
3. The application server **authenticates** the user, **authorizes** the upload, and
   **validates** the file (type, size, content).
4. The application stores the **original photo in object storage**, either by streaming
   it through the app server or by using a secure direct-to-storage upload mechanism
   (e.g., a pre-signed URL).
5. The application writes the photo’s **metadata** (owner, timestamp, object key, etc.)
   to the **primary database**.
6. The application publishes a **thumbnail-generation job** to the **message queue**,
   containing the object key of the original photo.
7. A **thumbnail worker** picks the job up from the queue.
8. The worker downloads the original, **creates a 50 KB thumbnail**, and prepares it
   for storage.
9. The worker **stores the thumbnail in object storage** under a predictable key.
10. Once the thumbnail exists in object storage, the **CDN** can fetch and cache it,
    and both the original and the thumbnail are ready to be served to any user.
11. The application returns a **success response** to the user, and the photo appears
    in the uploader’s profile and in followers’ feeds.

### Asynchronous thumbnail generation

Thumbnail generation happens **asynchronously** through the queue and worker. The user
does not wait for the thumbnail to finish before the upload request completes — the
upload returns as soon as the original photo and metadata are safely stored. The
thumbnail becomes available shortly afterward.

### Handling failures

- If the thumbnail worker fails, the job can be **retried** automatically by the queue.
- After a configurable number of failed attempts, the job is moved to a **dead-letter
  queue** for manual inspection or later reprocessing.
- The system must also handle partial failures — for example, the original photo is
  successfully stored in object storage but the database write fails. In production,
  this is handled with idempotent operations, compensating transactions, or a
  reconciliation job that cleans up orphaned objects.

---

## 8. Trade-Offs

### Trade-Off A: Caching vs. Fresh Data

- **Caching** improves speed and reduces database load, but it can return **stale data**
  (e.g., a new photo may not appear in a follower’s feed immediately).
- **Always reading from the database** guarantees fresh data, but it increases latency
  and puts heavy load on the database and read replicas.

**Recommendation for SnapShare:** Use caching aggressively for the read-heavy feed,
with short TTLs and cache invalidation on new uploads. A few seconds of staleness is
acceptable for a social feed, while the performance and cost benefits are essential at
this scale.

### Trade-Off B: Asynchronous vs. Synchronous Thumbnail Processing

- **Asynchronous** (queue + worker) makes uploads fast and responsive and isolates
  thumbnail processing from the upload path, but it introduces queues, workers, retries,
  and **eventual consistency** for the thumbnail.
- **Synchronous** processing returns the thumbnail together with the upload, giving
  immediate consistency, but it makes uploads slow, couples the systems, and does not
  scale well under load.

**Recommendation for SnapShare:** Use **asynchronous** thumbnail processing. Users
expect the upload itself to be fast; the thumbnail can appear a moment later without
noticeable impact on the experience.

### Trade-Off C: Database Read Replicas vs. a Single Database

- **Read replicas** greatly increase read capacity and isolate reads from writes, but
  they introduce **replication lag**, extra operational cost, and more complexity in
  choosing which database to query.
- A **single database** is simpler to operate, but it becomes a bottleneck as feed
  traffic grows and cannot scale reads independently of writes.

**Recommendation for SnapShare:** Use **read replicas**. With 50 million feed page
views per day, reads must be scaled horizontally, and a small amount of replication lag
is acceptable for a social feed.

### Trade-Off D: CDN Delivery vs. Serving Photos From Application Servers

- A **CDN** offloads image traffic from the application servers, reduces bandwidth
  costs, and delivers photos faster to users around the world, but it adds CDN cost,
  caching configuration, and cache-invalidation complexity.
- **Serving photos directly** from application servers is simpler to set up, but it
  consumes application-server bandwidth, increases latency for distant users, and does
  not scale to SnapShare’s traffic levels.

**Recommendation for SnapShare:** Use a **CDN** for all photos and thumbnails. At
50 million feed page views per day, serving images from application servers would be
prohibitively expensive and slow.

---

## 9. Conclusion

SnapShare serves **1 million daily active users** who together upload **1 million photos
per day** and view **50 million feed pages per day**, producing peak feed traffic of
roughly **2,900 feed views per second**. The system is clearly **read-heavy**, with a
read-to-write ratio of about 50:1, so the architecture prioritizes efficient reads
through a CDN, an in-memory cache, and database read replicas.

Original photos and thumbnails are stored in **object storage**, keeping the relational
database small and focused on metadata. A **message queue** and **thumbnail worker**
allow uploads to complete quickly while thumbnails are generated asynchronously in the
background.

Together, the CDN, cache, read replica, queue, and thumbnail worker let SnapShare scale
to the stated workload while keeping uploads reliable, reads fast, and operational costs
reasonable. The proposed architecture is well suited to the traffic and storage profile
calculated in this plan.