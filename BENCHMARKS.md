# Performance Benchmarks & Methodology

This document outlines how px0 measures performance, documents its scores across real-world repositories, and breaks down comparative resource consumption against VS Code.

## 1. Requirements

- Go 1.24+: To build the target binary.
- Git: Required only for `--clone`.
- System Utilities: `curl`, `awk`, `find`, `du` (standard on Linux and macOS).
- Disk Space: ~3 GB for the standard multi-repository corpus.
- Memory Measurements: Read via `/proc`, supported natively on Linux (other metrics function cross-platform).

## 2. Running Benchmarks
### 1. Build the binary

```bash
go build -o px0 .
```

### 2. Fetch the standard corpus

Clones shallow copies (`--depth 1`) of seven diverse open-source repositories:

```bash
./benchmark.sh --clone
```

### 3. Execute benchmark suite

Spawns an isolated px0 server process per repository, records metrics, and terminates the instance:

```bash
./benchmark.sh
```

## 3. Benchmark Corpus

The seven repositories were chosen to span two orders of magnitude in size and represent diverse language ecosystems:

| Repository                                            | Primary Language | Character / Role in Benchmark                                  |
| ----------------------------------------------------- | ---------------- | -------------------------------------------------------------- |
| [flask](https://github.com/pallets/flask)             | Python           | Compact library: tests instant sub-millisecond path.           |
| [redis](https://github.com/redis/redis)               | C                | Medium-sized C codebase with large monolithic source files.    |
| [react](https://github.com/facebook/react)             | JavaScript       | Deep directory nesting and many nested `.gitignore` files.     |
| [django](https://github.com/django/django)           | Python           | Large framework with thousands of modules and test suites.     |
| [TypeScript](https://github.com/microsoft/TypeScript) | TypeScript       | Very large source files with a massive generated baseline tree.|
| [kubernetes](https://github.com/kubernetes/kubernetes)| Go               | Large Go monorepo with extensive vendored code.                |
| [linux](https://github.com/torvalds/linux)            | C                | The extreme case: ~95,000 files and 1.8 GB of source text.     |

## 4. Benchmark Results

Measured on Linux x86_64 with language servers disabled (`-no-lsp`):

| Repo       | Source Size | Files  | Index  | Fuzzy   | Full Scan | Open Big | Reopen | Base Mem | Peak Mem |
| ---------- | ----------- | ------ | ------ | ------- | --------- | -------- | ------ | -------- | -------- |
| django     | 74 MB       | 7,014  | 39 ms  | 1.3 ms  | 26.8 ms   | 166.8 ms | 1.0 ms | 20 MB    | 29 MB    |
| flask      | 3 MB        | 235    | 1 ms   | 0.8 ms  | 2.3 ms    | n/a      | n/a    | 16 MB    | 18 MB    |
| kubernetes | 370 MB      | 25,926 | 150 ms | 13.5 ms | 84.6 ms   | 199.0 ms | 0.9 ms | 30 MB    | 44 MB    |
| linux      | 1,809 MB    | 95,710 | 370 ms | 6.0 ms  | 451.8 ms  | 26.7 ms  | 0.6 ms | 55 MB    | 73 MB    |
| react      | 63 MB       | 7,178  | 52 ms  | 2.7 ms  | 32.2 ms   | 57.7 ms  | 0.7 ms | 21 MB    | 28 MB    |
| redis      | 26 MB       | 1,855  | 13 ms  | 1.0 ms  | 18.2 ms   | 80.8 ms  | 1.2 ms | 17 MB    | 27 MB    |
| typescript | 414 MB      | 66,533 | 566 ms | 6.2 ms  | 150.3 ms  | 40.9 ms  | 6.5 ms | 69 MB    | 105 MB   |

### Metric Descriptions

- `Source`: Total working tree size (excluding `.git`).
- `Files`: Number of indexed files after applying `.gitignore` and built-in rules.
- `Index`: Cold startup directory walk and ignore set construction time.
- `Fuzzy`: Time to fuzzy-match a query across every indexed path.
- `Full Scan`: Literal search for a term that matches nothing (worst-case full codebase scan reading every byte).
- `Open Big`: Cold open of the largest source file: read, tokenize viewport window, return HTML.
- `Reopen`: Opening the same file once cached in memory.
- `Base Mem`: Resident memory (RSS) after indexing.
- `Peak Mem`: Peak memory during aggressive search and navigation prior to idle scavenging.

## 5. px0 vs. VS Code Process Comparison

Side-by-side comparison on identical Linux hardware:

| Metric / Parameter          | px0                    | VS Code (Remote / Server)     | Difference                   |
| --------------------------- | ---------------------- | ----------------------------- | ---------------------------- |
| Base RSS Memory             | ~15 - 18 MB            | ~1,160 - 1,440 MB             | ~75x - 90x lighter           |
| Idle Background Memory      | ~16 MB                 | ~1,440 MB                     | Stable & scavenged           |
| Active Startup CPU Spike    | < 1%                   | ~39% - 47%                    | Negligible CPU churn         |
| Index Time                  | < 1 ms                 | ~4 - 10 s                     | Sub-millisecond boot         |
| Process Count               | 1 single Go binary     | 15+ processes                 | Zero-dependency process tree |
| Runtime Model               | Native Go binary       | Electron / Node.js V8 runtime | No V8 heap overhead          |

### Measured VS Code Process Tree Breakdown

```text
PID     Role / Component                 RSS (MB)   CPU %
388357  Extension Host                   345.5 MB   4.0%
388724  Language Server (Pyrefly)        288.4 MB   0.0%
388075  VS Code Server Main              146.1 MB   0.0%
388113  File Watcher                     67.9 MB    0.0%
388089  IPC / Socket Proxy               64.7 MB    0.0%
388715  LSP: JSON Language Server        63.0 MB    0.0%
388697  PTY Host (Terminal)              62.8 MB    0.0%
388098  IPC / Socket Proxy               52.9 MB    0.0%
388427  Remote Containers Extension      51.6 MB    0.0%
388708  Integrated Terminal (bash)       8.9 MB     0.0%
```

In contrast, px0 embeds indexing, fuzzy search, syntax highlighting, and server endpoints inside a single native process.

## 6. Memory Scavenging Verification

To observe resident memory scavenging in real time, run:

```bash
./benchmark.sh --memory bench-repos/linux
```

```text
### linux
  after indexing                     59 MB
  after a fuzzy find                 59 MB
  after 5 full-tree searches         85 MB
  after opening the largest file     88 MB
  after scrolling through it         94 MB
  8 seconds idle                     94 MB
  30 seconds idle                    57 MB
```

After 15 seconds of inactivity, px0 triggers `debug.FreeOSMemory()`, returning unused heap pages back to the Linux kernel and settling back to baseline.
