# deemix-worker

A work-in-progress remote Deemix worker and command-line client. The worker will
run on a home server, accept download jobs from authenticated clients, stream
progress, and transfer completed files back to the requesting device.

## Current packages

- `deezer-sdk`: Deezer API and session integration.
- `deemix`: Downloading, decryption, metadata, and file organization.
- `deemix-cli`: Existing local CLI retained as the scaffold for the remote client.

The worker service and shared client/server protocol have not been implemented
yet.

See [`docs/PROJECT.md`](docs/PROJECT.md) for the agreed architecture, protocol,
security requirements, and implementation sequence.

## Development

This repository requires Node.js 24 and uses pnpm workspaces with Turborepo.

```bash
corepack enable
pnpm install
pnpm run ci
```

Run the current CLI from the repository with:

```bash
pnpm --filter deemix-cli start -- <url>
```

## Upstream

This project is derived from [bambanah/deemix](https://github.com/bambanah/deemix),
which revived the original Deemix project created by RemixDev. The `upstream`
Git remote is retained so fixes to the core packages can be reviewed and
integrated.

## License

Licensed under the GNU General Public License v3.0. See `LICENSE.txt`.
