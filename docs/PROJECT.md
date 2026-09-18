# Deemix Worker Project Context

## Goal

Build a private home-server download worker with a small CLI client. A user on
any device on the local network submits a Deezer playlist URL, watches live
progress, and receives the completed audio files on the device running the CLI.

The destination path is always local to the CLI device. The server must not try
to interpret or write to that path.

Example target command:

```bash
deemix-worker download \
  --server http://media-server:6595 \
  --output "D:\Music" \
  --bitrate flac \
  "https://www.deezer.com/playlist/123456"
```

## Current State

This repository was created from
[`bambanah/deemix`](https://github.com/bambanah/deemix). The upstream Git
history and `upstream` remote are retained. Commit `9546ef9` removed the Web UI,
Electron GUI, and their deployment and release configuration.

The remaining packages are:

- `packages/deezer-sdk`: Deezer API, gateway API, session, and account access.
- `packages/deemix`: Link resolution, downloading, decryption, metadata,
  tagging, settings, and Spotify conversion.
- `packages/cli`: The upstream local CLI, retained only as a scaffold.

The existing CLI still creates a downloader on the local machine. It is not yet
a network client. The worker service and shared protocol do not exist yet.

The repository currently builds, lints, type-checks, and passes all 63 tests
with Node.js 24 and pnpm.

## Target Architecture

```text
CLI device                              Home server
----------                              -----------
POST /v1/jobs ------------------------> create isolated job
GET /v1/jobs/:id/events <------------- stream SSE progress
GET /v1/jobs/:id/files/:fileId <------ transfer completed file
write under local --output directory
POST file acknowledgement ------------> mark transfer complete
DELETE /v1/jobs/:id ------------------> remove temporary job data
```

Use the following project boundaries:

```text
apps/
  worker/       Fastify service and job runner
  cli/          Remote command-line client
packages/
  protocol/     Shared Zod schemas and TypeScript types
  deemix/       Upstream-derived download engine
  deezer-sdk/   Upstream-derived Deezer integration
```

Move or replace the existing `packages/cli` when the remote client is
implemented. Keep worker-specific behavior out of `deemix` and `deezer-sdk` so
upstream fixes remain straightforward to review and integrate.

## Technology Decisions

- Node.js 24 with TypeScript and ESM.
- pnpm workspaces and Turborepo.
- Fastify for the worker HTTP API.
- REST for commands and file operations.
- Server-Sent Events for one-way progress streaming and reconnection.
- HTTP file streaming with range support for resumable transfers.
- Zod schemas shared through `packages/protocol`.
- SQLite for durable jobs, event sequence numbers, files, and acknowledgements.
- Commander for the CLI.
- Pino for structured worker logging.
- Vitest for unit and integration tests.
- Docker for eventual home-server deployment, after the worker exists.

SSE is preferred over WebSockets because commands already fit HTTP and progress
is server-to-client. SSE also provides a simpler reconnect model using event
IDs.

## Job Lifecycle

1. The CLI submits a URL and requested bitrate.
2. The worker creates a job ID and an isolated temporary output directory.
3. The worker creates a per-job copy of Deemix settings whose download location
   points at that directory. It must not mutate global settings per request.
4. Deemix listener events are translated into versioned protocol events.
5. As each track finishes, the worker records its safe relative path, size, and
   SHA-256 checksum and emits `fileReady`.
6. The CLI downloads the file to a `.part` path under `--output`.
7. The CLI verifies size and checksum, atomically renames the file, and
   acknowledges it.
8. The worker removes job files after all files are acknowledged or after a
   configured expiration period.

Transfer individual files as they become ready. Do not wait for the complete
playlist or create a ZIP by default; audio is already compressed and per-file
transfer provides earlier results and easier retries.

## Initial API

```text
POST   /v1/jobs
GET    /v1/jobs/:jobId
GET    /v1/jobs/:jobId/events
POST   /v1/jobs/:jobId/cancel
GET    /v1/jobs/:jobId/files
GET    /v1/jobs/:jobId/files/:fileId
POST   /v1/jobs/:jobId/files/:fileId/ack
DELETE /v1/jobs/:jobId
GET    /health
```

`POST /v1/jobs` should accept a source URL and bitrate, but not the CLI's local
destination path.

Important event types include:

- `jobQueued`
- `jobStarted`
- `trackProgress`
- `fileReady`
- `jobFinished`
- `jobFailed`
- `jobCancelled`

Every event needs a monotonically increasing ID so a reconnecting CLI can
resume from its last received event.

## Security Requirements

- Keep the Deezer ARL and all Deezer credentials exclusively on the worker.
- Require a bearer token even when running on a private LAN.
- Never log credentials or bearer tokens.
- Scope every file identifier to its owning job.
- Never expose arbitrary server filesystem paths through the API.
- Reject absolute paths, `..`, and paths escaping the selected destination on
  the CLI before writing files.
- Apply request size limits and basic rate limiting.
- Bind intentionally to a configured LAN interface; do not accidentally expose
  the service to the public internet.
- Use a reverse proxy such as Caddy if TLS is required.

## Operational Defaults

- Start with one active Deemix download job at a time.
- Allow queued jobs.
- Permit a small configurable number of concurrent file transfers.
- Download each job into its own directory under the worker data directory.
- Preserve job state across worker restarts.
- Expire abandoned job files, initially after 24 hours.
- Support HTTP range requests and CLI `.part` files for interrupted transfers.

## Immediate Implementation Order

1. Add `packages/protocol` with job, file, error, and event schemas.
2. Add `apps/worker` with configuration, bearer authentication, `/health`, and
   an in-memory job repository behind an interface.
3. Add job creation and bridge Deemix listener events into the protocol.
4. Add isolated job directories, file manifests, and HTTP file transfer.
5. Convert the existing CLI into an HTTP/SSE client with safe local writes.
6. Add SQLite persistence and resume behavior.
7. Add integration tests covering submission, progress, transfer verification,
   reconnection, cancellation, and cleanup.
8. Add worker Docker packaging and deployment documentation.

## Open Decisions

- Choose the SQLite implementation after checking Node.js 24 support and
  cross-platform packaging requirements.
- Decide whether the CLI package and executable should retain the `deemix-cli`
  and `deemix` names or be renamed to `deemix-worker`.
- Define overwrite behavior when a destination file already exists.
- Define storage limits and behavior when the worker runs out of temporary
  space.
- Decide whether completed files are removed immediately after acknowledgement
  or retained for a short retry window.

## Constraints

- This is intended for private, lawful use with content the user is authorized
  to access and download.
- The project is GPLv3. Preserve `LICENSE.txt` and upstream attribution. GPL
  distribution obligations apply to derivatives that are distributed.
- Do not combine code from the old extracted release directories outside this
  repository. This repository is the canonical working copy.

## Suggested Agent Skills

- Use `lavish` when presenting a substantial architecture or API plan for human
  review.
- Use `handoff` before ending a future session with significant uncommitted
  context that is not already captured in this document or repository history.
