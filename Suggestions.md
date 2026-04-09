Based on your thorough analysis in the README, here are some additional viable options worth considering for sharing GBs of data between MATLAB sessions on Ubuntu 20.04:

## Alternative Approaches

**1. Memory-Mapped Files via HDF5**
- MATLAB has native `h5read`/`h5write` support
- Can memory-map HDF5 datasets using `h5create` with appropriate chunk sizes
- Better than standard `memmapfile` since HDF5 is optimized for large scientific data
- Advantage: Built-in, stable, supports complex data structures
- Tradeoff: Still involves I/O, but HDF5's efficiency may be acceptable

**2. ZMQ (ZeroMQ) Message Queue**
- MATLAB has community bindings available
- Zero-copy messaging between processes
- Very stable and widely used
- Advantage: No need for shared memory, designed for high-throughput IPC
- Tradeoff: Requires serialization/deserialization, but can be very fast

**3. Named Pipes (FIFOs)**
- Standard Linux mechanism, completely stable
- MATLAB can interact via system calls or MEX wrappers
- Simpler than shared memory approaches
- Tradeoff: Sequential access pattern, not ideal for random access

**4. Shared Memory via `mmap()` on Anonymous Memory**
- Write a simple MEX wrapper around `mmap(MAP_ANONYMOUS)` + `fork()` pattern
- Avoids the instability of existing complex toolboxes
- Full control and minimal overhead
- Advantage: Lightweight, stable, direct control
- Tradeoff: Requires writing C/MEX code, but could be minimalist

**5. tmpfs/ramfs Filesystem**
- Write data to `/dev/shm` as binary files (already available on Ubuntu)
- Use `memmapfile` to access directly from multiple sessions
- Advantage: No I/O bottleneck (data never leaves RAM), very stable
- Tradeoff: Still technically file-based but with RAM speed

**6. Docker/Container Shared Volumes**
- Run MATLAB instances as containers with mounted shared volumes
- Uses kernel's shared memory or tmpfs
- Advantage: Containerized isolation, modern approach

## Recommendation for Your Use Case

Given your constraints (GBs of data, high throughput, write-once/read-once pattern), I'd suggest:

**Primary**: **tmpfs with `memmapfile`** - leverages MATLAB's native, stable API while keeping data in RAM. The file I/O overhead is minimal since data never touches disk.

**Secondary**: **Custom MEX wrapper around Linux shared memory** - if you need maximum performance. This gives you stability by avoiding the complex existing toolboxes while keeping full control.

**Tertiary**: **ZMQ bindings** - if you prefer message-based semantics over shared memory.

The key insight from your analysis is that the existing toolboxes are unstable due to MATLAB's undocumented memory management. A minimal, purpose-built solution (either tmpfs or custom MEX) might serve you better than trying to make complex toolboxes work.
