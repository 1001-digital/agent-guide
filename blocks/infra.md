# Infra

Use this guide when an agent needs deployable IPFS infrastructure for a dapp project.

## What To Use This For

- Running a project-owned Kubo IPFS node.
- Serving pinned content through a public gateway.
- Protecting the Kubo admin API behind Caddy basic auth.
- Uploading and pinning build artifacts, metadata, images, or static sites.
- Publishing IPNS records.
- Giving apps a stable gateway for assets the project controls.

## When Not To Use It

- Do not add `ipfs.server` as a frontend dependency.
- Do not deploy an IPFS node when public gateways or a pinning provider are enough.
- Do not expose the Kubo admin API publicly without authentication.
- Do not set a public gateway to fetch arbitrary network content unless the product intentionally needs that behavior.
- Do not store private credentials in the repo.

## Packages/Repos Involved

- Source repo: <https://github.com/1001-digital/ipfs.server>
- `ipfs.server`: Docker/Kamal deployment for Kubo plus Caddy.
- Kubo: IPFS daemon for pinning, gateway, IPNS, and RPC API.
- Caddy: reverse proxy for admin API auth and large uploads.
- Kamal: deployment tool.

## Install Commands

Prerequisites:

```sh
gem install kamal
pnpm install
```

The project itself is deployed from its repo. A dapp normally consumes its gateway URL; it does not install the repo as a package.

## Minimal Setup

1. Copy the production environment template:

```sh
cp .env.production.example .env.production
```

2. Fill the deployment variables:

```txt
DOCKER_REGISTRY_USERNAME=
DEPLOY_HOST=
KAMAL_REGISTRY_PASSWORD=
IPFS_HOST=
IPFS_ADMIN_HOST=
ADMIN_PASSWORD=
ADMIN_PASSWORD_HASH=
```

3. Generate the Caddy bcrypt hash:

```sh
caddy hash-password --plaintext 'your-password'
```

4. Run setup:

```sh
pnpm kamal:setup
```

5. Deploy:

```sh
pnpm kamal:deploy
```

## Core APIs/Components/Contracts/Config

### Architecture

The Docker image bundles:

| Service | Port | Purpose |
| --- | --- | --- |
| Kubo gateway | `8080` | Public read-only gateway for pinned content. |
| Kubo API | `5001` | Admin API, not exposed directly. |
| Caddy admin proxy | `5080` | Basic-auth proxy to Kubo API. |
| Kubo swarm | `4001` | IPFS P2P networking. |

Typical public URLs:

```txt
https://ipfs.example.com/ipfs/<CID>
https://ipfs.example.com/ipns/<name>
https://admin.ipfs.example.com/api/v0/...
```

Default gateway posture:

- `Gateway.NoFetch = true`
- Public gateway serves locally pinned/cached content only.
- Admin API requires basic auth.
- Upload script can pin after adding content.
- IPNS records use a long lifetime by default.
- Data persists via Docker volume or configured bind mount.

### Environment Variables

Required deployment variables:

| Variable | Purpose |
| --- | --- |
| `DOCKER_REGISTRY_USERNAME` | Docker registry username. |
| `DEPLOY_HOST` | Server IP or hostname. |
| `KAMAL_REGISTRY_PASSWORD` | Registry token/password. |
| `IPFS_HOST` | Public gateway domain. |
| `IPFS_ADMIN_HOST` | Admin API domain. |
| `ADMIN_PASSWORD` | Plaintext admin password for upload script. |
| `ADMIN_PASSWORD_HASH` | Bcrypt password hash for Caddy. |

Node tuning variables:

| Variable | Default | Purpose |
| --- | --- | --- |
| `GATEWAY_NO_FETCH` | `true` | Serve only pinned/cached content when true. |
| `GATEWAY_DESERIALIZED_RESPONSES` | `true` | Enable directory listings/deserialized responses. |
| `IPNS_RECORD_LIFETIME` | `336h` | IPNS record lifetime. |
| `STORAGE_MAX` | `20GB` | Maximum datastore size. |
| `ENABLE_GC` | `true` | Enable garbage collection. |
| `STORAGE_GC_WATERMARK` | `90` | Storage percentage that triggers GC. |
| `GC_PERIOD` | `1h` | GC check interval. |
| `CONN_MGR_HIGH_WATER` | `96` | Maximum peer connections. |
| `CONN_MGR_LOW_WATER` | `32` | Trim-to peer connection count. |
| `IPFS_VOLUME` | `ipfs_data` | Docker volume or host path. |
| `CONTAINER_CPUS` | `2` | CPU allocation. |
| `CONTAINER_MEMORY` | `6G` | Container memory limit. |
| `RESOURCE_MGR_MAX_MEMORY` | `4GB` | Libp2p resource manager memory cap. |
| `RESOURCE_MGR_MAX_FILE_DESCRIPTORS` | `4096` | Libp2p file descriptor cap. |

### Uploading Content

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

Required local env for upload:

```txt
IPFS_ADMIN_HOST=
ADMIN_PASSWORD=
IPFS_HOST=
```

The upload script logs timestamp, CID, MFS path, and source path to `uploads.log`.

### Admin API

Pin content:

```sh
curl -u admin:<password> -X POST \
  "https://admin.ipfs.example.com/api/v0/pin/add?arg=<CID>"
```

Add a file:

```sh
curl -u admin:<password> -X POST -F file=@myfile.txt \
  "https://admin.ipfs.example.com/api/v0/add"
```

Publish IPNS:

```sh
curl -u admin:<password> -X POST \
  "https://admin.ipfs.example.com/api/v0/name/publish?arg=<CID>"
```

Shell into deployment:

```sh
pnpm kamal:sh
```

## Common Pairings With Other 1001 Blocks

- Pair with `dweb-fetch` by configuring the app's IPFS gateway to the project-owned gateway.
- Pair with `normalize-dweb-url` by storing `ipfs://` URIs while serving through this gateway.
- Pair with `resolve-metadata` for metadata JSON uploaded to IPFS.
- Pair with `layers.evm` by setting `evm.ipfsGateway` in `app.config.ts`.
- Pair with NFT contracts using `WithIPFSMetaData` or `WithContractMetaData`.

## Practical Implementation Patterns

### Dapp-Owned Gateway

In app config:

```ts
export default defineAppConfig({
  evm: {
    ipfsGateway: 'https://ipfs.example.com/ipfs/',
    arweaveGateway: 'https://arweave.net/',
  },
})
```

In metadata fetcher:

```ts
const dweb = createDwebFetch({
  ipfs: {
    mode: 'gateway',
    gateways: ['https://ipfs.example.com'],
  },
})
```

`layers.evm` app config uses the path gateway form (`https://ipfs.example.com/ipfs/`). Direct `dweb-fetch` config uses the gateway origin/root (`https://ipfs.example.com`) because the client appends `/ipfs/<cid>` or `/ipns/<name>` internally.

Keep stored metadata as `ipfs://<cid>/...`; only resolve to the gateway at fetch/render time.

### Static Dapp Publish

1. Build the static output.
2. Upload the output directory with `pnpm upload ./dist --pin`.
3. Record the returned CID.
4. Optionally publish/update an IPNS name.
5. Use the CID/IPNS path in release notes or app config.

### NFT Metadata Publish

1. Generate metadata JSON and media assets.
2. Put metadata and assets in a deterministic directory structure.
3. Upload and pin the directory.
4. Set contract base URI or contract URI to `ipfs://<cid>/...`.
5. Use the metadata guide in the frontend to normalize and fetch.

### Gateway Policy

Use `GATEWAY_NO_FETCH=true` when:

- The gateway should only serve project-owned pinned content.
- You want predictable bandwidth/storage.
- You do not want to become a general public IPFS gateway.

Set it to `false` only when:

- The product explicitly needs a general fetch gateway.
- You have capacity, monitoring, and abuse controls for public fetching.

## CSS/Config/Runtime/Env Details

- Treat `.env.production` as secret-bearing deployment config.
- Use DNS records for `IPFS_HOST` and `IPFS_ADMIN_HOST` before deploying.
- Keep admin and public domains separate.
- Make sure the server exposes ports 4001, 8080, and 5080 as required by the deployment.
- Configure memory and storage according to expected pin volume.
- If using a host path for `IPFS_VOLUME`, ensure permissions and backups are handled.
- For CI/CD, use secret storage for registry and admin credentials.

## Gotchas And Failure Modes

- Missing `ADMIN_PASSWORD_HASH` can leave Caddy auth misconfigured.
- Mismatch between plaintext `ADMIN_PASSWORD` and hash breaks upload/admin workflows.
- `Gateway.NoFetch=true` means unpinned content will not load through the public gateway.
- GC can remove unpinned content when storage exceeds thresholds.
- IPNS records expire if not republished before their lifetime ends.
- Admin API must never be exposed without auth.
- Large uploads need proxy/body-size settings to remain compatible.
- DNS/TLS issues can look like IPFS failures; verify HTTP routing first.

## Agent Checklist

- Confirm the task really needs owned IPFS infrastructure.
- Configure public and admin domains.
- Generate a bcrypt admin password hash.
- Fill `.env.production` with registry, host, domain, and admin credentials.
- Keep `GATEWAY_NO_FETCH=true` unless a general gateway is required.
- Deploy with Kamal.
- Upload and pin content with `pnpm upload`.
- Configure apps to store protocol URIs and resolve through the owned gateway.
- Monitor storage, GC, peer connections, memory, and admin access.
- Keep admin credentials out of git and client-side code.
