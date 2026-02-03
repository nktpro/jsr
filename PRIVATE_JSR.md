# Private JSR Registry - Implementation Plan

This document outlines the architecture and implementation plan for a private JSR (JavaScript Registry) that acts as an overlay on top of the public jsr.io registry. Private packages are served locally while public packages are transparently proxied.

## Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Client (Deno/npm)                        │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                   Private JSR Proxy                          │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  1. Parse request (scope, package, version, path)      │  │
│  │  2. Check: Is @scope/package in private registry?      │  │
│  │     ├─ YES → Serve from local storage                  │  │
│  │     └─ NO  → Proxy to public jsr.io/npm.jsr.io         │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────┬──────────────────────────────────┬───────────────┘
            │                                  │
            ▼                                  ▼
┌───────────────────────┐          ┌───────────────────────────┐
│   Local Storage       │          │     Public JSR.io         │
│  /storage/            │          │   (transparent proxy)     │
│   @scope/pkg/meta.json│          │                           │
│   @scope/pkg/1.0.0/   │          │                           │
└───────────────────────┘          └───────────────────────────┘
```

## JSR API Architecture

### Three API Domains

| Domain | Purpose |
|--------|---------|
| `jsr.io` | Registry API - download modules/metadata |
| `npm.jsr.io` | npm compatibility - tarballs/package.json |
| `api.jsr.io` | Management API - publishing, scopes, auth |

### Key URL Patterns

```
# Package metadata
https://jsr.io/@{scope}/{package}/meta.json
https://jsr.io/@{scope}/{package}/{version}_meta.json

# Source files
https://jsr.io/@{scope}/{package}/{version}/{path}

# NPM compatibility
https://npm.jsr.io/@jsr/{scope}__{package}
https://npm.jsr.io/~/11/@jsr/{scope}__{package}/{version}.tgz
```

## Publishing Flow

### How `deno publish` Works

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         deno publish flow                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Read jsr.json/deno.json → extract @scope/package@version            │
│  2. Check auth token (or initiate device OAuth flow)                    │
│  3. Create gzipped tarball of package files                             │
│  4. POST to /api/scopes/{scope}/packages/{package}/versions/{version}   │
│     - Query: ?config=/jsr.json                                          │
│     - Headers: Content-Encoding: gzip, Content-Type: application/x-tar  │
│     - Body: gzipped tar archive                                         │
│  5. Poll GET /api/publishing_tasks/{id} until success/failure           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### API Endpoints Required

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/authorizations` | POST | Start device OAuth flow |
| `/api/authorizations/exchange` | POST | Exchange code for token |
| `/api/authorizations/details/:code` | GET | Check auth status |
| `/api/scopes/:scope/packages/:package/versions/:version` | POST | Upload package |
| `/api/publishing_tasks/:id` | GET | Poll publish status |

## Storage Structure

The storage structure mirrors jsr.io's layout:

```
/storage/
├── publishing_tasks/
│   └── {uuid}.tar.gz              # Temporary upload storage
├── @{scope}/
│   └── {package}/
│       ├── meta.json              # Package metadata
│       ├── {version}_meta.json    # Version metadata
│       └── {version}/
│           ├── mod.ts
│           └── lib/
│               └── utils.ts
└── npm/
    └── @jsr/
        └── {scope}__{package}/
            ├── package.json       # npm manifest
            └── {version}.tgz      # npm tarball
```

### Metadata File Formats

**meta.json** (Package metadata):
```json
{
  "scope": "myorg",
  "name": "private-pkg",
  "latest": "1.0.0",
  "versions": {
    "1.0.0": { "yanked": false }
  }
}
```

**{version}_meta.json** (Version metadata):
```json
{
  "manifest": {
    "/mod.ts": { "size": 234, "checksum": "sha256-..." }
  },
  "exports": { ".": "./mod.ts" },
  "moduleGraph2": { ... }
}
```

## Implementation Components

### 1. Proxy Server Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Private JSR Registry Server                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐   │
│  │   Auth       │   │   Publish    │   │   Registry (Read)        │   │
│  │   Endpoints  │   │   Endpoint   │   │   Endpoints              │   │
│  ├──────────────┤   ├──────────────┤   ├──────────────────────────┤   │
│  │ POST /auth   │   │ POST /api/   │   │ GET /@scope/pkg/meta.json│   │
│  │ POST /exchg  │   │ scopes/../   │   │ GET /@scope/pkg/1.0.0/.. │   │
│  │              │   │ versions/..  │   │ GET /npm/@jsr/...        │   │
│  └──────┬───────┘   └──────┬───────┘   └────────────┬─────────────┘   │
│         │                  │                        │                  │
│         ▼                  ▼                        ▼                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                     Request Router                                │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  Is this a private scope/package?                           │ │ │
│  │  │    YES → Handle locally                                     │ │ │
│  │  │    NO  → Proxy to public jsr.io                             │ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                     Local Storage                                 │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                     SQLite Database                               │ │
│  │  - tokens (auth tokens)                                           │ │
│  │  - publishing_tasks (status, errors)                              │ │
│  │  - packages (scope, name, latest)                                 │ │
│  │  - package_versions (version, manifest)                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

### 2. Core Request Handler

```typescript
// Core routing logic
async function handleRequest(req: Request): Promise<Response> {
  const url = new URL(req.url);
  const path = url.pathname;

  // Handle API routes (publishing, auth)
  if (path.startsWith("/api/")) {
    return handleApiRoute(req, path);
  }

  // Handle npm registry routes
  if (path.startsWith("/npm/") || path.startsWith("/@jsr/")) {
    return handleNpmRoute(req, path);
  }

  // Handle JSR registry routes (/@scope/package/...)
  const parsed = parseJsrPath(path);

  if (parsed && isPrivatePackage(parsed.scope, parsed.package)) {
    return serveFromLocalStorage(parsed);
  }

  // Proxy to public registry
  return proxyToPublic(req, path);
}
```

### 3. Publish Handler

```typescript
async function handlePublish(req: Request): Promise<Response> {
  const { scope, package: pkg, version } = parseParams(req);

  // 1. Verify auth token
  const token = req.headers.get("Authorization")?.replace("Bearer ", "");
  if (!verifyToken(token, scope)) {
    return new Response("Unauthorized", { status: 401 });
  }

  // 2. Get config path from query
  const configPath = new URL(req.url).searchParams.get("config");

  // 3. Stream tarball to temp storage, compute SHA256
  const taskId = crypto.randomUUID();
  const tarballPath = `publishing_tasks/${taskId}.tar.gz`;
  const hash = await streamToStorage(req.body, tarballPath);

  // 4. Create publishing task record
  await db.insert("publishing_tasks", {
    id: taskId,
    scope, package: pkg, version,
    config_file: configPath,
    status: "pending"
  });

  // 5. Queue async processing
  processPublishingTask(taskId);

  // 6. Return task ID for polling
  return Response.json({
    id: taskId,
    status: "pending"
  });
}
```

### 4. Tarball Processing

```typescript
async function processPublishingTask(taskId: string) {
  await db.update("publishing_tasks", taskId, { status: "processing" });

  try {
    const task = await db.get("publishing_tasks", taskId);
    const tarball = await readTarball(`publishing_tasks/${taskId}.tar.gz`);

    // 1. Extract and validate files
    const files = await extractTarball(tarball);
    validateFiles(files); // size limits, no symlinks, etc.

    // 2. Parse config file (jsr.json)
    const config = JSON.parse(files[task.config_file]);
    if (config.name !== `@${task.scope}/${task.package}`) {
      throw new Error("configFileNameMismatch");
    }
    if (config.version !== task.version) {
      throw new Error("configFileVersionMismatch");
    }

    // 3. Build manifest with checksums
    const manifest = {};
    for (const [path, content] of Object.entries(files)) {
      const checksum = await sha256(content);
      manifest[path] = { size: content.length, checksum: `sha256-${checksum}` };

      // 4. Write file to storage
      await writeFile(`@${task.scope}/${task.package}/${task.version}${path}`, content);
    }

    // 5. Create version metadata
    const versionMeta = {
      exports: config.exports,
      manifest,
      moduleGraph2: buildModuleGraph(files, config)
    };
    await writeJson(
      `@${task.scope}/${task.package}/${task.version}_meta.json`,
      versionMeta
    );

    // 6. Update package metadata
    const pkgMeta = await readJson(`@${task.scope}/${task.package}/meta.json`)
      ?? { scope: task.scope, name: task.package, versions: {} };
    pkgMeta.versions[task.version] = { yanked: false };
    pkgMeta.latest = task.version;
    await writeJson(`@${task.scope}/${task.package}/meta.json`, pkgMeta);

    // 7. Generate npm compatibility files (optional)
    await generateNpmTarball(task, files, config);

    // 8. Mark success
    await db.update("publishing_tasks", taskId, { status: "success" });

  } catch (error) {
    await db.update("publishing_tasks", taskId, {
      status: "failure",
      error: { code: error.message, message: error.toString() }
    });
  }
}
```

### 5. Authentication (Simplified for Private Use)

```typescript
// Device auth flow - simplified for private use
const pendingAuths = new Map<string, { verifier?: string, approved: boolean }>();

// POST /api/authorizations - Start auth
app.post("/api/authorizations", async (req) => {
  const { challenge } = await req.json();
  const code = generateCode(); // "ABCD-EFGH"
  const exchangeToken = generateToken();

  pendingAuths.set(exchangeToken, { approved: false });

  return Response.json({
    verification_url: "http://localhost:8080/auth",
    code,
    exchange_token: exchangeToken,
    expires_at: new Date(Date.now() + 600000).toISOString(),
    poll_interval: 2
  });
});

// POST /api/authorizations/exchange - Get token
app.post("/api/authorizations/exchange", async (req) => {
  const { exchange_token, verifier } = await req.json();
  const auth = pendingAuths.get(exchange_token);

  if (!auth?.approved) {
    return new Response("Pending", { status: 400 });
  }

  // Generate permanent token
  const token = `jsrd_${generateToken()}`;
  await db.insert("tokens", { token, created_at: new Date() });

  return Response.json({ token });
});
```

## Client Configuration

### Deno Clients

```bash
# Set registry URL
export JSR_URL=http://your-private-registry:8080

# Publish as normal
deno publish
```

### npm/pnpm/yarn Clients

```ini
# .npmrc
@jsr:registry=http://your-private-registry:8080/npm/
```

### Import Maps (deno.json)

```json
{
  "imports": {
    "@myorg/private-pkg": "http://localhost:8080/@myorg/private-pkg@1.0.0/mod.ts"
  }
}
```

## Minimum Viable Implementation

For the simplest possible private registry, these shortcuts can be taken:

1. **Skip authentication** - Trust the network for internal use
2. **Skip npm compatibility** - If only using Deno
3. **Skip module graph validation** - Trust the publisher
4. **Use filesystem storage** - Instead of a database for metadata

This reduces the implementation to:
- 1 publish endpoint (accept tarball, extract, write files)
- 1 polling endpoint (return success immediately)
- File serving with proxy fallback to public jsr.io

## Reference Files in JSR Codebase

| Purpose | File Path |
|---------|-----------|
| Publish endpoint | `api/src/api/package.rs:753-912` |
| Tarball processing | `api/src/publish.rs` |
| File validation | `api/src/tarball.rs` |
| Metadata structures | `api/src/metadata.rs` |
| Auth flow | `api/src/api/authorization.rs` |
| Storage paths | `api/src/gcs_paths.rs` |
| NPM generation | `api/src/npm/mod.rs` |
| NPM type mapping | `api/src/npm/types.rs` |
| Import rewriting | `api/src/npm/specifiers.rs` |

## Implementation Phases

### Phase 1: Read-Only Proxy
- Implement transparent proxy to jsr.io
- Add local file serving for private packages
- Manual package placement in storage directory

### Phase 2: Publishing Support
- Implement publish endpoint
- Tarball extraction and storage
- Metadata generation

### Phase 3: Authentication
- Device OAuth flow
- Token management
- Scope-based access control

### Phase 4: npm Compatibility
- npm manifest generation
- Tarball creation with transpiled JS
- Import specifier rewriting (`jsr:` → `@jsr/`)

### Phase 5: Advanced Features
- Module graph validation
- Documentation generation
- Search indexing
- Web UI for browsing private packages
