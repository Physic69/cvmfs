# CVMFS Core Dumper (`cvmfs2 __COREDUMP__`)

The CVMFS core dumper provides a built-in mechanism to capture crash dumps for debugging, designed specifically to avoid system limitations with standard core dumping mechanisms and to skip unnecessary kernel-mapped memory.

## Why a Built-in Dumper?

Standard system core dump mechanisms (`kernel.core_pattern`) can be restrictive, and reading the entire memory space of a process might fail on certain kernel-mapped regions (such as `[vvar]`, `[vvar_vclock]`, or `[vsyscall]`) returning `EIO` on `pread()`. 

To solve this, CVMFS includes its own dumper invoked via the hidden `__COREDUMP__` subcommand of `cvmfs2`. This allows CVMFS to safely extract process memory state to a core file, specifically ignoring regions that cause failures.

## Dump Levels

The dumper supports two dump levels, specified via flags:
- `--standard` (default): Read-only, file-backed regions of shared libraries are skipped. GDB can seamlessly reload these from disk via `NT_FILE`. However, the dynamic linker and the main executable are always preserved since GDB needs access to `_DYNAMIC -> DT_DEBUG -> r_debug -> link_map` to discover and load shared libraries.
- `--full`: Dumps all readable regions, resulting in a significantly larger core file. Useful for deep debugging.

## Heuristic RBP Recovery

On some architectures (like x86_64), `/proc/pid/syscall` provides the Instruction Pointer (`RIP`) and Stack Pointer (`RSP`), but lacks the Base Pointer (`RBP`). Without `RBP`, GDB backtraces often fail and stop at frame 1 on binaries compiled with frame pointers.

To work around this, the dumper implements a heuristic RBP recovery:
1. It scans the stack for values that look like valid frame pointers.
2. It verifies if `*(candidate+8)` falls inside an executable mapping (acting as a valid return address).
3. It ensures that `*(candidate)` successfully chains to another stack address.

The candidate with the longest valid chain is selected as the recovered RBP, allowing GDB to provide complete backtraces.

## Supported Architectures

Currently, the built-in dumper explicitly supports:
- `x86_64` (and `amd64`)
- `aarch64` (ARM64)

Other architectures will report as unsupported when invoked via `cvmfs_config dcoredump`.

## Usage

A helper script wraps the invocation of the dumper:

```bash
# Standard dump (default, smaller size)
cvmfs_config dcoredump <pid> [output.core]

# Full dump (includes shared libraries)
cvmfs_config dcoredump --full <pid> [output.core]
```

If `[output.core]` is omitted, it defaults to `/tmp/cvmfs_dcoredump_<pid>_<timestamp>.core`.
