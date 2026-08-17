# Kinetic Engine — Indexer Subsystem

## Source
Repository: `Alayra-Systems-Pvt-Limited/KineticIDE`
Branch: `main`
Path: `extensions/kinetic-core/kinetic-engine/src/indexer/`

## Status
**PARTIAL — all currently enumerated indexer files have been inspected in this pass, but the called `WorkspaceIndexer`, `QueryEngine`, `VectorEngine`, graph, hashing, and embedding implementations still require deeper cross-file tracing before the engine/indexer can be marked complete.**

## 1. Module surface

`indexer/mod.rs` defines the public indexer subsystem and pins the wire contract for four modes:

- `local`
- `remote_engine`
- `team`
- `sharded`

Serialization uses snake_case. Tests explicitly reject older names such as `TeamServe`, `ShardWorker`, and `team_serve`, making the current literals a compatibility contract with the TypeScript indexer contract.

The module exports:

- `job_queue`
- `query_cache`
- `retrieve`
- `config`
- `auth`
- `state`
- `http`
- `serve`

The current implementation therefore contains both local retrieval/index orchestration and a Team HTTP serving mode. `RemoteEngine` and `Sharded` are contract-level modes here but their full runtime implementations are not established by this directory alone.

## 2. Single-flight build coordination

`job_queue.rs` prevents duplicate index builds for the same workspace/storage pair.

The stable key is:

```text
workspace_path + "::" + global_storage_path
```

A process-wide `OnceLock<Mutex<JobQueueState>>` stores active builds. Each `InFlightBuild` contains:

- a `Notify` used to wake followers;
- shared build result;
- shared current phase.

`run_coalesced()` elects the first caller as leader. Concurrent callers with the same key become followers and await the leader's notification, then receive the same result. The build entry is removed after completion.

This means concurrent `index_workspace` requests do **not** independently rebuild the same workspace.

The module also exposes best-effort progress state:

- `set_phase`
- `set_phase_for_workspace`
- `is_build_in_progress`
- `current_phase`

Tests verify that two overlapping calls execute the underlying build exactly once.

## 3. Retrieval facade

`retrieve.rs` is the indexer's structured retrieval boundary. It does not implement search algorithms itself; it coordinates `VectorEngine`, `FileHasher`, `QueryEngine`, and `query_cache`.

`RetrieveContextResult` carries:

- retrieved chunks;
- formatted matches;
- graph status;
- dropped graph-neighbor count;
- BM25 health;
- vector-search health;
- indexed file count;
- indexed Git revision;
- current HEAD revision;
- cache-hit state.

It also provides `index_revision_stale()`, which compares indexed and current HEAD commits.

### Retrieval flow

```text
retrieve_context / query_codebase
        ↓
VectorEngine initialization
        ↓
FileHasher → indexed revision
        ↓
Git HEAD → current revision
        ↓
query_cache key
        ├── hit → cached SearchOutcome + current file count
        └── miss
             ↓
        query_engine::search_index
             ↓
        cache SearchOutcome
             ↓
        indexed file count
             ↓
        RetrieveContextResult
```

The cache key includes workspace, global storage, indexed revision, query, top-K, and path-filter key. Therefore cached results are revision-scoped, query-scoped, and filter-scoped.

After index mutations, `invalidate_retrieval_caches()` clears both retrieval result cache and the QueryEngine indexed-file-count cache for that workspace/storage pair.

## 4. Query result cache

`query_cache.rs` implements a process-local in-memory cache with a **60-second TTL**.

The key is composed of:

```text
workspace
storage
indexed revision
query
Top-K
path filter
```

Entries are protected by a `Mutex<HashMap<...>>` and return cloned `SearchOutcome` values.

Workspace invalidation removes every key sharing the workspace/storage prefix. Tests cover both cache hits and invalidation.

Architecturally this is an optimization layer, not durable index state. The index itself lives behind `VectorEngine`/LanceDB.

## 5. Team indexer configuration

`config.rs` defines `ServeConfig` for the Team HTTP server. Version 1 is explicitly **one repository per process**.

Required configuration:

- bind address
- workspace path
- data directory
- repo ID
- API key
- Git polling interval

Optional:

- company ID

Validation fails fast when:

- workspace is empty;
- data directory is empty;
- repo ID is empty;
- API key is empty;
- API key does not start with `knt_idx_`;
- bind address is empty;
- Git polling is below 30 seconds.

The comments indicate configuration can originate from CLI/environment variables. No implicit secret defaults are used.

## 6. Team API authentication

`auth.rs` validates `Authorization: Bearer ...` against the configured API key.

The comparison is implemented byte-wise with an accumulated XOR result rather than ordinary string equality, reducing timing-side-channel exposure for the key comparison.

The accepted header is either `Bearer ` or lowercase `bearer ` followed by the configured token. Missing/empty/wrong tokens fail.

## 7. Team server state

`state.rs` defines process-wide `ServeState`:

- validated `ServeConfig`;
- shared `Arc<VectorEngine>`;
- last successful index timestamp;
- last Git poll timestamp.

It provides the same workspace/storage-derived build key as the local job queue and async methods for updating timestamps.

The Team server is therefore a single-process, single-repository index service with one long-lived VectorEngine instance.

## 8. Team HTTP API

`http.rs` mounts:

```text
/api/v1/indexer/health
/api/v1/indexer/status
/api/v1/indexer/query
/api/v1/indexer/index/trigger
/api/v1/indexer/register
```

All routes are behind the bearer authentication middleware.

### `/health`
Returns a minimal health contract containing:

- `status: ok`
- `index_mode: team`
- vector schema version
- repository ID
- optional company ID

### `/status`
Reports operational index state including:

- ready/building status;
- Team mode;
- repository ID;
- indexed Git revision;
- current HEAD revision;
- indexed file count;
- build progress/phase;
- last index timestamp;
- last Git poll timestamp.

### `/query`
The query body contains:

- `repoId`
- optional `revision`
- `query`
- `topK` defaulting to 5
- optional path list

Validation requires:

- request repo ID matches this server's configured repo;
- non-empty query;
- `topK` between 1 and 50.

If a revision is requested and differs from the indexed revision, the server returns HTTP 409 rather than serving an inconsistent revision.

The query delegates to the same structured retrieval path used by local MCP retrieval, using the server's shared VectorEngine.

The response includes chunks, formatted matches, revision metadata, cache state, graph state, and BM25/vector health.

### `/index/trigger`
Runs the same single-flight index build path and returns `indexed` on success.

### `/register`
Currently returns `501 Not Implemented` and explicitly identifies shard-worker registration as a later IDX-5 capability. This is an important distinction: the route exists as a contract placeholder but is **not implemented**.

## 9. Team serve lifecycle

`serve.rs` implements the long-running Team indexer process.

Startup flow:

```text
ServeConfig
   ↓
create data directory
   ↓
VectorEngine::new
   ↓
ServeState
   ├── boot index build (background)
   ├── Git polling loop (background)
   └── Axum HTTP listener
```

The boot index uses `WorkspaceIndexer::build_index_with_options(..., false)` through the same `job_queue::run_coalesced()` mechanism used by MCP.

After a successful build, retrieval caches are invalidated and `last_index_at` is updated.

### Git freshness loop

The server polls Git at the configured interval (minimum 30 seconds). It obtains the current HEAD commit and compares it with `FileHasher.last_commit`.

If unchanged, no rebuild occurs.

If changed:

```text
Git HEAD changed
      ↓
run_index_build
      ↓
job_queue single-flight
      ↓
WorkspaceIndexer full build
      ↓
cache invalidation
```

The first timer tick is skipped so boot indexing is not immediately duplicated.

## 10. Local and Team paths converge

A significant architectural property is that the Team server does **not** create a separate indexing algorithm. Both local MCP and Team serve mode converge on the same `WorkspaceIndexer` and retrieval machinery.

```text
                         ┌─ MCP index_workspace ─┐
                         │                       │
Host / RPC ──────────────┤                       ├→ WorkspaceIndexer
                         │                       │
                         └─ Team /index/trigger ─┘

WorkspaceIndexer
       ↓
VectorEngine / LanceDB
       ↓
query_engine / graph / BM25 / vector retrieval
```

This reduces behavioral divergence between local and Team indexing.

## 11. Current implementation boundary

Confirmed implemented in this directory:

- four-mode wire contract;
- local retrieval facade;
- 60-second retrieval cache;
- cache invalidation;
- per-workspace single-flight builds;
- Team configuration validation;
- bearer authentication;
- Team state;
- authenticated Team HTTP health/status/query/index routes;
- Team boot indexing;
- Git-based freshness polling;
- explicit placeholder for future shard registration.

Not claimed as implemented from this directory alone:

- full RemoteEngine runtime;
- full Sharded runtime;
- shard-worker registration;
- actual indexing/chunking/embedding implementation;
- BM25/vector ranking internals;
- graph construction/retrieval internals;
- file hashing semantics beyond the calls observed here.

Those require source tracing into the parent engine modules and must be studied before the overall indexer can be declared complete.

## 12. Cross-layer contracts discovered

The indexer subsystem establishes several contracts with the rest of the engine:

1. **`WorkspaceIndexer` is the canonical build implementation.**
2. **`VectorEngine` is the durable local vector/graph database boundary.**
3. **`FileHasher` is the revision/freshness authority.**
4. **`QueryEngine` owns actual hybrid retrieval and indexed-file counting.**
5. **`GraphRetriever` contributes graph health/status to retrieval results.**
6. **MCP and Team HTTP are transport/orchestration layers around the same underlying index/retrieval implementation.**

These relationships must be preserved in the master architecture map.
