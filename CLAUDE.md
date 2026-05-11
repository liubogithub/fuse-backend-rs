# CLAUDE.md

This file provides guidance for Claude Code when working in this repository.

## Project Overview

`fuse-rs` is a Rust library for implementing FUSE (Filesystem in Userspace) server daemons. It supports both the Linux FUSE device (`/dev/fuse`) and the [virtiofs](https://virtio-fs.gitlab.io/) paravirtualized protocol used for guest-host filesystem sharing in virtualization environments.

## Architecture

The crate is divided into four subsystems:

- **`src/abi/`** — ABI definitions shared between the Linux FUSE kernel driver and userspace daemons (Linux FUSE ABI v7.27+; also virtio-fs extensions).
- **`src/api/`** — The Fuse API layer connecting transport and filesystem drivers:
  - `filesystem.rs` — The `FileSystem` trait that backend implementations must satisfy.
  - `server.rs` — Request dispatch: parses raw FUSE messages, calls `FileSystem` methods, sends replies.
  - `vfs.rs` — A virtual filesystem multiplexer allowing multiple filesystems behind one virtio-fs device.
  - `pseudo_fs.rs` — A minimal pseudo-filesystem used internally by VFS.
- **`src/transport/`** — Transport layer implementations:
  - `fusedev/` — Linux `/dev/fuse` device transport (feature `fusedev`).
  - `virtiofs/` — virtio-fs Virtio queue transport (feature `virtiofs`).
  - `file_traits.rs` — `FileReadWriteVolatile` trait for zero-copy I/O with `vm-memory` buffers.
- **`src/passthrough/`** — A sample passthrough filesystem that forwards all operations to a host directory.

## Build & Feature Flags

```bash
# Default build (no transport enabled)
cargo build

# Enable Linux /dev/fuse transport
cargo build --features fusedev

# Enable virtio-fs transport
cargo build --features virtiofs

# Enable vhost-user-fs (virtiofs over vhost-user socket)
cargo build --features vhost-user-fs
```

Key feature flags in `Cargo.toml`:
| Feature | Enables |
|---|---|
| `fusedev` | Linux `/dev/fuse` device transport |
| `virtiofs` | Virtio-fs transport via `vm-virtio` |
| `vhost-user-fs` | Vhost-user slave support via `vhost-rs` |

## Testing

```bash
cargo test
cargo test --features fusedev
cargo test --features virtiofs
```

## Key Types & Traits

- **`FileSystem`** (`src/api/filesystem.rs`) — The central trait to implement a FUSE backend. All filesystem operations (`lookup`, `read`, `write`, `mkdir`, etc.) have default no-op implementations returning `ENOSYS`.
- **`Server<F>`** (`src/api/server.rs`) — Wraps a `FileSystem` implementation and dispatches raw FUSE protocol messages to it.
- **`Vfs`** (`src/api/vfs.rs`) — Multiplexes multiple `FileSystem` implementations under a single mount point.
- **`PassthroughFs`** (`src/passthrough/fs.rs`) — Reference implementation of `FileSystem` that passes operations through to an underlying host directory.

## Code Conventions

- Dual-licensed Apache-2.0 AND BSD-3-Clause; all new files must carry the appropriate SPDX header.
- The crate enforces `#![deny(missing_docs)]` — all public items require doc comments.
- Error handling uses the crate-local `Error` enum and `Result<T>` alias defined in `src/lib.rs`.
- Transport-specific code must be gated behind the appropriate feature flag.

## External Dependencies of Note

- [`vm-memory`](https://github.com/rust-vmm/vm-memory) — used for safe access to guest physical memory in virtio-fs scenarios.
- [`vm-virtio`](https://github.com/rust-vmm/vm-virtio) — Virtio queue abstractions (optional, `virtiofs` feature).
- [`vhost-rs`](https://github.com/cloud-hypervisor/vhost) — vhost-user slave support (optional, `vhost-user-fs` feature).
