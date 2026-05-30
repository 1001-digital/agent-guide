# Infra

Use `1001-digital/ipfs.server` when a project needs owned IPFS infrastructure rather than just an app dependency.

## Use This For

- Running a project-owned Kubo IPFS node.
- Serving pinned content through a public gateway.
- Protecting the Kubo admin API behind Caddy basic auth.
- Uploading and pinning build artifacts, metadata, images, or static sites.
- Giving apps a stable gateway for assets the project controls.

## Do Not Use This For

- A frontend package dependency.
- Projects where public gateways or a pinning provider are enough.
- Private content.
- Public, unauthenticated access to the Kubo admin API.

## Repo

- Source repo: <https://github.com/1001-digital/ipfs.server>
- Kubo: IPFS daemon for pinning, gateway, IPNS, and RPC API.
- Caddy: reverse proxy for admin API auth and large uploads.
- Kamal: deployment tool.

## Setup

Clone/deploy from the infra repo itself. A dapp normally consumes the gateway URL; it does not install this repo.

Prerequisites:

```sh
gem install kamal
pnpm install
```

Create production env:

```sh
cp .env.production.example .env.production
```

Fill the key variables:

```txt
DOCKER_REGISTRY_USERNAME=
DEPLOY_HOST=
KAMAL_REGISTRY_PASSWORD=
IPFS_HOST=
IPFS_ADMIN_HOST=
ADMIN_PASSWORD=
ADMIN_PASSWORD_HASH=
```

Generate the Caddy bcrypt hash:

```sh
caddy hash-password --plaintext 'your-password'
```

Put the hash in `.env.production` with single quotes so shell sourcing does not expand `$` separators:

```txt
ADMIN_PASSWORD_HASH='$2a$14$exampleHashFromCaddy'
```

Deploy:

```sh
pnpm kamal:setup
pnpm kamal:deploy
```

## Architecture

Typical public URLs:

```txt
https://ipfs.example.com/ipfs/<CID>
https://ipfs.example.com/ipns/<name>
https://admin.ipfs.example.com/api/v0/...
```

Service shape:

- Kubo gateway on `8080`.
- Kubo API on `5001`, not exposed directly.
- Caddy admin proxy on `5080` with basic auth.
- Kubo swarm on `4001`.

Default gateway posture:

- Public gateway serves pinned/cached content.
- Admin API requires basic auth.
- Data persists via Docker volume or configured bind mount.
- Apps should use the public gateway URL for reads and authenticated admin URL only for operational scripts.

## Upload Basics

Upload a directory:

```sh
pnpm upload ./dist
```

Upload and pin:

```sh
pnpm upload ./dist --pin
```

Upload to a custom MFS path:

```sh
pnpm upload ./dist --mfs-path /my-site
```

Upload env:

```txt
IPFS_ADMIN_HOST=
ADMIN_PASSWORD=
IPFS_HOST=
```

## Pair With

- [`metadata.md`](metadata.md) when apps need a stable gateway for NFT media or metadata.
- [`layers.md`](layers.md) when Nuxt runtime config should point at the owned gateway.
- Static dapp deployment flows when build artifacts should be pinned.

## Agent Checklist

- Treat this as infrastructure, not an app dependency.
- Decide whether owned IPFS is actually needed before deploying.
- Keep admin credentials out of the repo.
- Use authenticated admin API only for uploads/pins/admin tasks.
- Point apps at the public gateway URL for reads.
