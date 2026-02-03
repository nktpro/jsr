# Private JSR Registry - Minimal Implementation

A minimal private JSR registry using only the filesystem for persistence. Acts as an overlay on public jsr.io - private packages are served locally, everything else proxies transparently.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Client (Deno)                            │
│                  JSR_URL=http://localhost:4873              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  Private JSR Proxy                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Request → Is private scope? → YES → Local filesystem │  │
│  │                              → NO  → Proxy to jsr.io  │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────┬─────────────────────────────────┬────────────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐           ┌─────────────────────────┐
│  ./storage/         │           │  https://jsr.io         │
│  @scope/pkg/...     │           │  (transparent proxy)    │
└─────────────────────┘           └─────────────────────────┘
```

## Filesystem Structure

All state lives in a single `./storage` directory:

```
./storage/
├── .tokens                           # Auth tokens (one per line)
├── .tasks/                           # Publishing tasks
│   └── {uuid}.json                   # Task status file
│
└── @{scope}/
    └── {package}/
        ├── meta.json                 # Package metadata
        ├── {version}_meta.json       # Version metadata
        └── {version}/                # Source files
            ├── mod.ts
            └── lib/utils.ts
```

## API Endpoints

Only 4 endpoints are needed:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/scopes/:scope/packages/:package/versions/:version` | POST | Publish package |
| `/api/publishing_tasks/:id` | GET | Poll publish status |
| `/@:scope/:package/*` | GET | Serve files / proxy to jsr.io |
| `/api/authorizations/*` | POST | Simple token auth |

## Metadata Formats

**meta.json** - Package metadata:
```json
{
  "scope": "myorg",
  "name": "private-pkg",
  "latest": "1.0.0",
  "versions": {
    "1.0.0": {}
  }
}
```

**{version}_meta.json** - Version metadata:
```json
{
  "exports": { ".": "./mod.ts" },
  "manifest": {
    "/mod.ts": { "size": 234, "checksum": "sha256-abc123..." }
  }
}
```

## Implementation

### Main Server

```typescript
// server.ts
const STORAGE_DIR = "./storage";
const PRIVATE_SCOPES = new Set(["myorg", "internal"]); // Configure your private scopes

Deno.serve({ port: 4873 }, async (req) => {
  const url = new URL(req.url);
  const path = url.pathname;

  // Publishing endpoint
  if (req.method === "POST" && path.match(/^\/api\/scopes\/([^/]+)\/packages\/([^/]+)\/versions\/([^/]+)$/)) {
    return handlePublish(req, path, url.searchParams.get("config") ?? "/jsr.json");
  }

  // Poll publishing task
  if (req.method === "GET" && path.startsWith("/api/publishing_tasks/")) {
    return handleTaskStatus(path.split("/").pop()!);
  }

  // Auth endpoints (auto-approve for simplicity)
  if (path.startsWith("/api/authorizations")) {
    return handleAuth(req, path);
  }

  // Serve files - check if private, otherwise proxy
  return handleFileRequest(path);
});
```

### File Request Handler

```typescript
async function handleFileRequest(path: string): Promise<Response> {
  // Parse: /@scope/package/...
  const match = path.match(/^\/@([^/]+)\/([^/]+)/);
  if (!match) {
    return proxyToJsr(path);
  }

  const [, scope] = match;

  // Private scope? Serve from filesystem
  if (PRIVATE_SCOPES.has(scope)) {
    const filePath = `${STORAGE_DIR}/${path}`;
    try {
      const content = await Deno.readFile(filePath);
      return new Response(content, {
        headers: { "content-type": getContentType(filePath) }
      });
    } catch {
      return new Response("Not found", { status: 404 });
    }
  }

  // Public scope - proxy to jsr.io
  return proxyToJsr(path);
}

async function proxyToJsr(path: string): Promise<Response> {
  const resp = await fetch(`https://jsr.io${path}`);
  return new Response(resp.body, {
    status: resp.status,
    headers: resp.headers
  });
}

function getContentType(path: string): string {
  if (path.endsWith(".ts")) return "text/typescript";
  if (path.endsWith(".js")) return "text/javascript";
  if (path.endsWith(".json")) return "application/json";
  return "application/octet-stream";
}
```

### Publish Handler

```typescript
async function handlePublish(req: Request, path: string, configPath: string): Promise<Response> {
  // Parse URL: /api/scopes/{scope}/packages/{package}/versions/{version}
  const match = path.match(/^\/api\/scopes\/([^/]+)\/packages\/([^/]+)\/versions\/([^/]+)$/);
  if (!match) return new Response("Bad request", { status: 400 });

  const [, scope, pkg, version] = match;
  const taskId = crypto.randomUUID();

  // Save task status
  await Deno.mkdir(`${STORAGE_DIR}/.tasks`, { recursive: true });
  await writeJson(`${STORAGE_DIR}/.tasks/${taskId}.json`, {
    id: taskId,
    scope, package: pkg, version,
    configPath,
    status: "processing"
  });

  // Process tarball in background
  processTarball(taskId, req, scope, pkg, version, configPath);

  return Response.json({ id: taskId, status: "processing" });
}

async function processTarball(
  taskId: string,
  req: Request,
  scope: string,
  pkg: string,
  version: string,
  configPath: string
) {
  const taskFile = `${STORAGE_DIR}/.tasks/${taskId}.json`;

  try {
    // Read and decompress tarball
    const gzipped = new Uint8Array(await req.arrayBuffer());
    const tarData = gunzip(gzipped);
    const files = untar(tarData);

    // Parse config
    const configContent = files.get(configPath.replace(/^\//, ""));
    if (!configContent) throw new Error("configFileNotFound");

    const config = JSON.parse(new TextDecoder().decode(configContent));

    // Validate name/version match
    if (config.name !== `@${scope}/${pkg}`) throw new Error("configFileNameMismatch");
    if (config.version !== version) throw new Error("configFileVersionMismatch");

    // Write files and build manifest
    const manifest: Record<string, { size: number; checksum: string }> = {};
    const versionDir = `${STORAGE_DIR}/@${scope}/${pkg}/${version}`;

    for (const [filePath, content] of files) {
      const fullPath = `${versionDir}/${filePath}`;
      await Deno.mkdir(dirname(fullPath), { recursive: true });
      await Deno.writeFile(fullPath, content);

      const hash = await crypto.subtle.digest("SHA-256", content);
      const checksum = "sha256-" + encodeHex(new Uint8Array(hash));
      manifest["/" + filePath] = { size: content.length, checksum };
    }

    // Write version metadata
    await writeJson(`${STORAGE_DIR}/@${scope}/${pkg}/${version}_meta.json`, {
      exports: config.exports ?? { ".": "./mod.ts" },
      manifest
    });

    // Update package metadata
    const metaPath = `${STORAGE_DIR}/@${scope}/${pkg}/meta.json`;
    const meta = await readJsonOr(metaPath, { scope, name: pkg, versions: {} });
    meta.versions[version] = {};
    meta.latest = version;
    await writeJson(metaPath, meta);

    // Mark success
    await writeJson(taskFile, { id: taskId, status: "success" });

  } catch (error) {
    await writeJson(taskFile, {
      id: taskId,
      status: "failure",
      error: { code: error.message, message: String(error) }
    });
  }
}
```

### Task Status Handler

```typescript
async function handleTaskStatus(taskId: string): Promise<Response> {
  try {
    const task = await readJson(`${STORAGE_DIR}/.tasks/${taskId}.json`);
    return Response.json(task);
  } catch {
    return new Response("Task not found", { status: 404 });
  }
}
```

### Simple Auth (Auto-Approve)

```typescript
async function handleAuth(req: Request, path: string): Promise<Response> {
  // POST /api/authorizations - Start auth flow
  if (req.method === "POST" && path === "/api/authorizations") {
    const exchangeToken = crypto.randomUUID();

    // Auto-approve: write token immediately
    await Deno.mkdir(STORAGE_DIR, { recursive: true });
    await Deno.writeTextFile(
      `${STORAGE_DIR}/.tokens`,
      exchangeToken + "\n",
      { append: true }
    );

    return Response.json({
      code: "AUTO",
      exchange_token: exchangeToken,
      expires_at: new Date(Date.now() + 600000).toISOString(),
      poll_interval: 1
    });
  }

  // POST /api/authorizations/exchange - Return token
  if (req.method === "POST" && path === "/api/authorizations/exchange") {
    const { exchange_token } = await req.json();
    return Response.json({ token: `jsrd_${exchange_token}` });
  }

  return new Response("Not found", { status: 404 });
}
```

### Utility Functions

```typescript
import { dirname } from "jsr:@std/path";
import { decodeBase64, encodeHex } from "jsr:@std/encoding";

// Simple gunzip using DecompressionStream
async function gunzip(data: Uint8Array): Promise<Uint8Array> {
  const ds = new DecompressionStream("gzip");
  const writer = ds.writable.getWriter();
  writer.write(data);
  writer.close();

  const chunks: Uint8Array[] = [];
  const reader = ds.readable.getReader();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    chunks.push(value);
  }

  const result = new Uint8Array(chunks.reduce((a, c) => a + c.length, 0));
  let offset = 0;
  for (const chunk of chunks) {
    result.set(chunk, offset);
    offset += chunk.length;
  }
  return result;
}

// Simple tar extraction (handles basic POSIX tar)
function untar(data: Uint8Array): Map<string, Uint8Array> {
  const files = new Map<string, Uint8Array>();
  let offset = 0;

  while (offset < data.length - 512) {
    const header = data.slice(offset, offset + 512);
    if (header.every(b => b === 0)) break;

    const name = new TextDecoder().decode(header.slice(0, 100)).replace(/\0.*/, "");
    const size = parseInt(new TextDecoder().decode(header.slice(124, 136)).trim(), 8);
    const type = header[156];

    offset += 512;

    if (type === 48 || type === 0) { // Regular file
      files.set(name, data.slice(offset, offset + size));
    }

    offset += Math.ceil(size / 512) * 512;
  }

  return files;
}

async function writeJson(path: string, data: unknown): Promise<void> {
  await Deno.mkdir(dirname(path), { recursive: true });
  await Deno.writeTextFile(path, JSON.stringify(data, null, 2));
}

async function readJson(path: string): Promise<unknown> {
  return JSON.parse(await Deno.readTextFile(path));
}

async function readJsonOr<T>(path: string, fallback: T): Promise<T> {
  try {
    return await readJson(path) as T;
  } catch {
    return fallback;
  }
}
```

## Client Configuration

```bash
# Set the registry URL and publish
export JSR_URL=http://localhost:4873
deno publish
```

Or in your shell profile:
```bash
export JSR_URL=http://localhost:4873
```

## Running the Server

```bash
deno run --allow-net --allow-read --allow-write server.ts
```

## Configuration

Edit the `PRIVATE_SCOPES` set in server.ts to define which scopes are private:

```typescript
const PRIVATE_SCOPES = new Set([
  "mycompany",
  "internal",
  "private"
]);
```

Any package under these scopes will be stored locally. All other requests proxy to public jsr.io.

## Limitations

This minimal implementation intentionally omits:

- **npm compatibility** - Only works with Deno
- **Module graph validation** - Trusts the publisher
- **Documentation generation** - No docs site
- **Search** - No package discovery
- **Access control** - All authenticated users can publish to any private scope
- **Version yanking** - Not implemented

For production use, consider adding these features or using the full JSR implementation.

## Reference Files

Key files in the JSR codebase for understanding the full implementation:

| Purpose | Path |
|---------|------|
| Publish endpoint | `api/src/api/package.rs:753-912` |
| Tarball processing | `api/src/publish.rs` |
| Metadata structures | `api/src/metadata.rs` |
| Storage paths | `api/src/gcs_paths.rs` |
