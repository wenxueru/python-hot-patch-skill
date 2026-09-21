---
name: python-hot-patch
description: Safely patch a live CPython process without restart using gdb and Py_AddPendingCall when preserving expensive in-memory state is necessary and the change is narrowly scoped.
---

# Python Hot Patch

Use this only when restarting would lose valuable in-memory state and the requested change can be expressed as a small mutation of existing Python objects or methods.

## Safety boundary

- Obtain authorization immediately before attaching to the process.
- Prefer restart, a supported control API, signal handler, or existing debug endpoint when available.
- Do not hot-patch interpreter internals, object layouts, native extension state, model structure, optimizer state, or invariants spanning multiple threads.
- Never call `PyRun_SimpleString` directly from an arbitrary stopped thread. It may not own the GIL and can deadlock or corrupt the interpreter.
- Apply once. Use a marker containing the old/new values so a retry cannot repeat non-idempotent mutations.

## Workflow

1. Identify the exact Python PID and inspect the relevant source and live-state invariant. Confirm CPython version, `ptrace`/seccomp policy, debugger availability, and required exported symbols.
2. Define acceptance checks before attaching. Include process survival, semantic state, expected throughput or concurrency, and persisted configuration.
3. Build a tiny temporary shared library:
   - An exported scheduling function calls `Py_AddPendingCall`.
   - The pending callback calls `PyRun_SimpleString` with the narrowly scoped patch.
   - The Python patch finds the target object defensively, validates uniqueness and old state, mutates it, and writes a marker.
4. With gdb, attach briefly, `dlopen` the library, call the scheduling function, verify it returns zero, then detach immediately. The callback must run later on the Python main thread at an interpreter safe point.
5. Wait passively for the marker or process exit. Do not repeatedly attach or enqueue the same callback while its outcome is unknown.
6. Verify behavior independently from the marker. Check the process, observable application state, errors, and resource usage.
7. Apply the equivalent source or configuration change on disk so restart and recovery preserve the fix. Keep live and persisted state consistent.

## Async semaphore changes

For an `asyncio.Semaphore` concurrency increase, run the mutation on the event-loop thread. If the old limit is `old` and the target is `new`, call `release()` exactly `new - old` times and update the owning limit field. Do not replace the semaphore: existing waiters are attached to the original object.

If the callback does not run promptly because the main thread is blocked in native I/O, wait for a natural Python safe point. Do not force arbitrary Python execution through gdb merely to wake it.

## Handoff

Report the PID, old/new state, attach result, marker, independent verification, persisted change, and any temporary debugger environment left behind. Never claim success from `Py_AddPendingCall` returning zero alone; that only confirms the callback was queued.
