# NFS backend (libnfs): what was built and what it learned

Phase 3 of [network-shares-plan.md](./network-shares-plan.md), built early in parallel with the netfs core. Code: `ps5/netfs/backend_nfs.cpp` in `thatboialex/snes9xPS5`, implementing `netfs::Backend` (`netfs_backend.h`). Reimplemented from the design in [swanstation-nfs-reference.md](./swanstation-nfs-reference.md); no SwanStationPS5 code. libnfs 6.0.2 is vendored in `ps5/third_party/libnfs` (changes listed in its `PATCHES.md`).

## Decisions

| Decision | Why |
|---|---|
| Use libnfs's async calls with our own wait loop, not its sync wrappers. | The sync wrappers cannot be interrupted and default to a 60 s RPC timeout. Our loop checks `Abort()` and a 4.5 s deadline (under the 5 s watchdog) every 50 ms step. |
| A call that is given up on (deadline, socket error) ends the connection. | libnfs still points at that call and at the caller's read buffer. No libnfs call is made afterwards except destroying the context; the call objects outlive the context. |
| Mount the whole path first, then one folder higher (at most 8 tries), a fresh context per try. | The export is not known in advance. A failed `nfs_mount` leaves the context unfit for another. A transport failure (refused, timeout, closed) stops the loop at once. |
| After a mount above the configured folder, `stat` the remainder. | Missing folder gives `FolderNotFound`, not a confusing empty shelf. |
| If every try fails, ask mountd for the export list (v3 only). Path inside a listed export: `AccessDenied`. Otherwise `ShareNotFound` with the list. | The server answers "permission denied" for an unexported path as well as for a refused client. The list tells them apart. |
| `Open()` does a `stat` first. | The `Backend` contract wants `EISDIR` for folders; libnfs would open one. One extra round trip per open, a few per launch. |
| `List()` follows symlinks with a `stat` and drops dangling ones; entries sent without attributes get the same. | Contract rule 6. Normal entries cost nothing extra: READDIRPLUS carries the attributes. |
| `Pread()` splits into 128 KiB READs, up to 8 in flight, and returns the bytes that arrived whole from the start. | libnfs 6.0.2 sends one request per call (it clamps to the negotiated size, so a 1 MiB call is not pipelined), and it cannot join an RPC reply a server split into fragments. Small pipelined requests keep both fast and compatible. |

## What real libnfs does

Loopback test (`ps5/tests/nfs_loopback_test.cpp`, no server needed):

- A refused connection and a server that hangs up both end as `Socket failed with POLLHUP`. The wait loop reads `SO_ERROR` (`ECONNREFUSED`, `ETIMEDOUT`, ...) to tell them apart.
- libnfs keeps two error strings: `nfs_get_error` and `rpc_get_error`. Socket and resolver failures only appear in the second.
- An unresolvable host fails when the mount is started (`nfs_mount_async` returns non-zero), not in the callback.
- `Abort()` shuts the socket down, so the loop can see a socket error before the flag; the flag decides the result.

Real NFSv3 server (unfs3 0.11, `make -C ps5 test-nfs-server`, needs root, rpcbind and unfsd):

- Mounts by export and by folder inside an export, lists with symlinks followed, reads a 3.5 MB file byte for byte, reports a missing folder, an unexported path (with the export list) and a refused export, and fails a read in under 5 s when the server stops (SIGSTOP).
- Stock unfs3 splits every reply above a few KB into RPC fragments and libnfs answers `Fragment support not yet working`. Servers on libtirpc do this; Linux knfsd, FreeBSD nfsd and Windows send whole records. The test server is built with a larger send size (see the script header).
- Not run: NFSv4 (unfs3 is v3 only), knfsd, Windows, NAS firmware.

## Needs from the shared PS5 layer (not in this backend)

- A working `getaddrinfo` (libnfs resolves the server with it) and an `fcntl` that makes sockets non-blocking. Both are the planned net shims; the console's own versions are unverified or refused (spec, "Wrapped calls").
- `nfs://host:port/...`: `port` is used as the NFS port when `nfsport` is not given.

## Unverified

Everything on the console: the PS5 patches to libnfs (`__PROSPERO__`), `SO_ERROR` and `SO_NOSIGPIPE` on console sockets, throughput, behaviour on Wi-Fi.

## Known limits

- A server that sends fragmented RPC records does not work with libnfs 6.0.2 (see above); the read ends as a lost connection.
- The export list (`mount_getexports_timeout`, 2 s) blocks and cannot be aborted. It runs only after a mount failed, when the share is unusable anyway.
- A file handle open on a connection that died is leaked with the context (a few dozen bytes). Closing it would need the dead socket.
- NFSv4 mounts by path (no export list); `Enumerate()` returns `ENOTSUP`.
