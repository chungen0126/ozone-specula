# Client concurrent positioned reads

Apache Ozone `chungen0126/ozone` branch `ozone-pread-specula` at `a2d5d2415c6cca72b039cc6aab149589099ff485`:
`apache/ozone` master `c4343421fe` plus eight open client read PRs merged together (see Known Incidents and References).

## Required

### Goal

- Decide whether the open client positioned-read PRs are safe to land together. Specifically, check that concurrent
  positioned reads on one open key stream (`read(long, ...)`, `readFully(long, ...)`, `readVectored`) return the
  block's bytes, leave the sequential read state alone, and give back every client reference, streaming-read permit,
  idle reader and buffer they take. This must hold while sequential reads, `seek`, `unbuffer`, `close`, failover and
  block info refresh run on the same stream.

### Scope

- In scope:
  - The positioned-read path from `OzoneFSInputStream` / `CapableOzoneFSInputStream` through
    `MultipartInputStream.readPositioned` down to the part streams:
    - `BlockInputStream`, with the `ChunkInputStream` range reader and `LocalChunkInputStream` for short-circuit.
    - `StreamBlockInputStream`, with `StreamingReader` and the idle positioned-read reader pool.
    - `ECBlockInputStreamProxy`, `ECBlockInputStream` and the EC reconstruction streams.
  - Client-side shared state these reads touch:
    - `XceiverClientGrpc` streaming-read permits (`initStreamRead`, `completeStreamRead`, `resumeStreamRead`).
    - Client references from `XceiverClientFactory.acquireClientForReadData` / `releaseClientForReadData`.
    - Concurrent use of one `XceiverClientShortCircuit` by several block streams.
  - Both settings of `ozone.client.stream.read.pread.reuse.enabled`, and both `BlockInputStream` and
    `StreamBlockInputStream` (`ozone.client.stream.readblock.enable`).
- Boundaries/exclusions:
  - Datanode-side read handling, gRPC/Netty internals, checksum algorithms, EC codec math, and OM key lookup.
    Read these only to set the contract of `refreshFunction`, the response stream and the client cache.
  - Write paths and performance are excluded.

### Priority Questions

1. When several threads issue positioned reads on the same stream concurrently with each other and with sequential
   `read` / `seek` / `unbuffer`, can a call return wrong data, return a count that does not match the bytes written
   into `dst`, or change the sequential cursor or its buffered data? "Wrong data" means bytes that are not the block's
   bytes at the requested offsets. The interleavings to cover include:
   - An idle streaming reader reused for a later request while responses from an earlier request are still queued
     or still arriving.
   - A response that starts before or after the requested offset.
   - A stream that completes early.

   Expected behavior: each positioned read either fills `dst` with exactly the bytes at `[pos, pos + n)` and returns
   `n` (or EOF), or throws with `dst`'s position restored. `getPos()` and later sequential reads are unaffected.
2. Can a streaming-read permit, client reference or idle reader be released twice, never be released, or stay open
   after the stream is closed? Also, can the idle reader count drift from the deque contents or exceed
   `ozone.client.stream.read.pread.max-idle-readers`? Check every path:
   - success, failure, and the single retry on a new stream after an idle reader fails;
   - interrupt, read timeout, and an `idlePreadGeneration` change from `unbuffer()` or a block info refresh;
   - the idle trimmer;
   - `close()` racing with positioned reads in flight;
   - gRPC `onNext` / `onError` / `onCompleted` arriving while a reader is idle, between lease and `resumePermit`, or
     after return.

   Expected behavior: every acquire has exactly one release. Once `close()` has returned and the positioned reads in
   flight have returned, no positioned-read stream stays open and every permit taken by this stream is back.
   For `readVectored`, every range future completes exactly once, and every allocated buffer is either delivered
   through its future or released exactly once.
3. When `close()`, `unbuffer()` or a block info refresh races with lock-free positioned reads, can a read do any of
   the following?
   - Use a pipeline from one refresh and a token from another.
   - Reuse a reader opened on the old pipeline or in an older generation.
   - Read through a released client or a closed short-circuit `FileChannel` and return wrong data instead of failing.
   - Hang past the configured read timeout.

   Also, can `close()` throw or hang? The refresh is pipeline or token changes after a datanode failure, started by a
   sequential read or by another positioned read. Since HDDS-17399, positioned reads in `BlockInputStream` and
   `StreamBlockInputStream` do not take the stream lock after initialization and leave the open check to
   `MultipartInputStream`.

   Expected behavior: such a read either completes with correct data or fails with an `IOException`. `close()`
   completes. Readers from an older generation or pipeline are never reused.
4. Can a failure in one read spoil other concurrent or later reads through shared state? The shared state includes:
   - `failedStreamingDatanodes` and the retry counters;
   - the EC proxy's shared `failedLocations` and its switch to reconstruction;
   - the `ChunkInputStream` position rewound after a failed read (HDDS-17000).

   Ways it could spoil them:
   - Every datanode stays excluded.
   - A stale failure is rethrown after retries are exhausted (HDDS-17765).
   - A healthy EC read is forced into reconstruction with the wrong failed set.
   - Bytes copied by a failed attempt are returned twice or skipped.

   Expected behavior: a failure costs only the failing call's retry budget, plus the datanode exclusion that is
   intended to be shared. After a terminal failure, the next read starts with a fresh retry budget and a full set of
   datanodes.
5. When an EC read that falls back to reconstruction is interrupted, finds its executor saturated, rejecting or shut
   down, or fails partway through a stripe, can the call return in either of these ways? (See HDDS-17001 and
   HDDS-17003.)
   - While a cell read is still writing into a stripe buffer that `close()` or the buffer pool hands to someone else.
   - With a stripe buffer that was not fully filled.

   This covers both a positioned read through a per-request `ECBlockInputStreamProxy` and a sequential read.

   Expected behavior: no call returns until every cell read writing into its buffers has finished. A failed stripe
   read exposes no partial data, and the next read seeks back to the current position first.

### Must-cover Interactions

- `MultipartInputStream.readPositioned` ↔ `BlockInputStream.readPositioned` / `readChunkAt` ↔
  `ChunkInputStream.readPositioned` (`ChunkReader`, client reference owned by the request): these run against the
  sequential `readWithStrategy`, `seek`, `unbuffer`, `close` and `handleReadError` on the same block. They share the
  `BlockChunks` snapshot published by `initialize()`, `pipelineRef` and `tokenRef`.
- `StreamBlockInputStream.readPositioned` ↔ the idle reader pool (`leaseIdlePreadReader`, `returnIdlePreadReader`,
  `trimIdlePreadReaders`, `closeIdlePreadReaders`, `idlePreadGeneration`, `closed`) ↔ `StreamingReader`
  (`responseQueue`, `future`, `semaphoreReleased`, `isReusable`, `releasePermitWhileIdle`, `resumePermit`) ↔
  `XceiverClientGrpc` permits.
- The `StreamBlockInputStream` sequential path (`readFully`, `dataAvailableToRead`, `handleExceptions`,
  `resetAfterTerminalFailure`, `updateBlockLength` for an under-construction block) ↔ concurrent positioned reads
  sharing `pipelineRef`, `tokenRef`, `failedStreamingDatanodes`, `needUpdateBlockLength` and `blockLength`.
- `ECBlockInputStreamProxy.readPositioned` ↔ `ECBlockInputStream.readPositioned` (parallel cell reads under
  `streamLock`, racing `close()`) ↔ the per-request reconstruction proxy ↔
  `ECBlockReconstructedStripeInputStream.loadDataBuffersFromStream` (executor, interrupt, buffer pool) ↔
  `ECBlockReconstructedInputStream.readStripe` failure handling.
- `CapableOzoneFSInputStream.readVectored` workers on the shared executor ↔ `close()` and the caller's
  `allocate` / `release` functions. The workers keep running after `readVectored` returns.
- `BlockInputStream` short-circuit reads ↔ the `XceiverClientShortCircuit` shared through the client cache:
  - Concurrent `connect`, `sendCommand` (GetBlock) and `close`.
  - The receiver thread, `sentRequests` and `blockStreamCache`.
  - Request timeouts.
  - Positional `FileChannel` reads in `LocalChunkInputStream` (HDDS-16335).
- Consider:
  - A thread interrupt at any blocking point.
  - gRPC callbacks at any time, including while a reader is idle.
  - The single shared trimmer thread.
  - A datanode failing mid-stream.
  - The executor rejecting tasks or shutting down.
  - The client cache evicting a client that an idle reader still references.

### Assumptions

- Committed blocks are immutable. Only the last block of an hsync-ed key, which is under construction, can grow, and
  its length comes from the datanode (`needUpdateBlockLength`).
- Sequential `read`, `seek` and `unbuffer` are synchronized on the stream. Positioned reads may run from any number of
  threads at once, concurrently with them. `close()` may run on any thread while positioned reads are in flight.
- A positioned read that starts before `close()` and then succeeds is not a bug by itself, because HDDS-17399 removed
  that check on purpose. Leaking a resource, returning wrong data or hanging is a bug.
- On one gRPC stream, the responses to one request arrive in order and start at a checksum boundary at or before the
  requested offset.
- Treat the merged code on this branch as ground truth. Use the PR descriptions only for their stated intent.

## Optional

### Known Incidents and References

- PRs merged on this branch, in merge order. Each is open on `apache/ozone` at the time of writing.
  - [apache/ozone#11999](https://github.com/apache/ozone/pull/11999) HDDS-17765: `StreamBlockInputStream` keeps
    rethrowing a stale failure after retries are exhausted.
  - [apache/ozone#11836](https://github.com/apache/ozone/pull/11836) HDDS-17000: Rewind `ChunkInputStream` when a
    read fails after copying buffered bytes.
  - [apache/ozone#11940](https://github.com/apache/ozone/pull/11940) HDDS-17001: An EC reconstruction read returns
    unfilled stripe buffers after a failed read or seek.
  - [apache/ozone#11958](https://github.com/apache/ozone/pull/11958) HDDS-17003: An interrupted EC reconstruction read
    returns while its cell reads still write into pooled stripe buffers.
  - [apache/ozone#11724](https://github.com/apache/ozone/pull/11724) HDDS-17399: Positioned reads are lock-free after
    initialization in `BlockInputStream` and `StreamBlockInputStream`.
  - [apache/ozone#11990](https://github.com/apache/ozone/pull/11990) HDDS-17752: Reuse streaming readers for
    positioned reads.
  - [apache/ozone#11608](https://github.com/apache/ozone/pull/11608) HDDS-13660: Bounded asynchronous vectored reads
    in OzoneFS.
  - [apache/ozone#11225](https://github.com/apache/ozone/pull/11225) HDDS-16335: `XceiverClientShortCircuit` supports
    concurrent access.
- Cross-PR interaction found while merging: a test in HDDS-17752 expected `readPositioned` to fail after `close()`,
  which HDDS-17399's design does not do. The branch checks `checkOpen()` instead. Evaluate the combined design, not
  either PR alone.
- Earlier Specula targets on related code:
  - [HDDS-16545](https://issues.apache.org/jira/browse/HDDS-16545) `client-read-retry`: sequential retry and EC
    reconstruction.
  - [HDDS-17428](https://issues.apache.org/jira/browse/HDDS-17428) `dn-short-circuit-read`.

  Do not rediscover what they already cover. Focus on the concurrency that these PRs add or change.
- [HDDS-16983](https://issues.apache.org/jira/browse/HDDS-16983) (merged) added the per-chunk range reader that
  `BlockInputStream` positioned reads use.

### Suggested Starting Points

- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/storage/StreamBlockInputStream.java`:
  `readPositioned`, `preadOnce`, `leaseIdlePreadReader`, `returnIdlePreadReader`, `trimIdlePreadReaders`,
  `handlePreadException`, `refreshBlockInfo`, `close`, `StreamingReader`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/storage/BlockInputStream.java`: `readPositioned`,
  `readChunkAt`, `initialize`, `BlockChunks`, `close`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/storage/ChunkInputStream.java`: `readPositioned`,
  `readChunkPositioned`, `unread`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/storage/MultipartInputStream.java`: `readPositioned`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/XceiverClientGrpc.java`: `initStreamRead`,
  `completeStreamRead`, `resumeStreamRead`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/hdds/scm/XceiverClientShortCircuit.java`: `connect`,
  `sendCommandWithTraceID`, `sendRequest`, `ReceiveResponseTask`, `close`.
- `hadoop-hdds/client/src/main/java/org/apache/hadoop/ozone/client/io/`: `ECBlockInputStreamProxy.readPositioned`,
  `ECBlockInputStream.readPositioned`, `ECBlockReconstructedStripeInputStream.loadDataBuffersFromStream`,
  `ECBlockReconstructedInputStream.readStripe`.
- `hadoop-ozone/ozonefs-common/src/main/java/org/apache/hadoop/fs/ozone/CapableOzoneFSInputStream.java`:
  `readVectored`, `readRange`.
- Harness anchors to extend:
  - Unit tests in `hadoop-hdds/client/src/test/`: `TestStreamBlockInputStream` (idle reader pool and positioned-read
    concurrency), `TestBlockInputStream#testPositionedReadDoesNotWaitForSequentialRead`,
    `TestMultipartInputStream#testPositionedReadAfterCloseFails` and `TestECBlockReconstructedInputStream`.
  - `hadoop-ozone/ozonefs-common/src/test/.../TestOzoneFSInputStream` (vectored reads).
  - Mini-cluster tests in `hadoop-ozone/integration-test/`: `TestXceiverClientManagerSC` (short-circuit concurrency)
    and `TestOzoneFSInputStream`.

### Additional Exploration

- Positioned reads on the under-construction last block of an hsync-ed key while it grows, against
  `updateBlockLength` and `MultipartInputStream.initialize`.
- Whether one stream's slow `closeStream` on the shared idle-trimmer thread delays the trimming of other streams.
- Positioned reads through `OzoneCryptoInputStream` on encrypted keys over the same part streams.
