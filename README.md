# PHP Process Killer

**Fast, non-blocking, and cross-platform process tree termination for PHP.**

Standard PHP tools like `proc_terminate()` or `posix_kill()` are often insufficient for complex process management. On Windows, native termination can block the executing thread; on Unix-like systems, terminating a parent process frequently leaves orphan grandchildren running in the background.

`rcalicdan/process-killer` resolves these issues by implementing platform-specific, asynchronous, and recursive termination strategies to safely and immediately tear down target processes and all of their descendants.

[![Latest Release](https://img.shields.io/github/release/rcalicdan/process-killer.svg?style=flat-square)](https://github.com/rcalicdan/process-killer/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](./LICENSE)

---

## Features

- **True Non-Blocking Termination on Windows:** Uses a fire-and-forget background command to initiate `taskkill`, preventing your main PHP thread from blocking during process cleanup.
- **Recursive Tree Termination:** Automatically discovers and targets all descendants (children, grandchildren, etc.) of a process to prevent orphaned resource leaks.
- **Platform-Specific Optimization:**
  - **Linux:** Scans `/proc` in a single pass to map relationships, utilizing process group (`pgid`) signaling.
  - **macOS / BSD:** Utilizes optimized `ps` scans to map hierarchies.
  - **Fallback:** Progressively falls back to recursive `pgrep` discovery on systems lacking standard filesystems.
- **Zero Runtime Dependencies:** Built strictly on PHP 8.3+ core capabilities.

---

## Installation

Ensure your environment meets the minimum requirement of PHP 8.3+.

```bash
composer require rcalicdan/process-killer
```

---

## Usage

The library exposes a single, static entry point designed to cleanly accept one or more process IDs (PIDs).

### Basic Example

```php
<?php

declare(strict_types=1);

require __DIR__ . '/vendor/autoload.php';

use Rcalicdan\ProcessKiller\ProcessKiller;

// Spawning a background process
$process = proc_open('php long_running_task.php', [], $pipes);
$status = proc_get_status($process);
$pid = $status['pid'];

// ... some asynchronous execution ...

// Forcefully terminate the process and all of its descendants
ProcessKiller::killTreesAsync([$pid]);
```

### Why Async Termination Matters (The Windows Problem)

When executing `@proc_terminate($resource)` in PHP on Windows, the call often blocks the active PHP thread while waiting for the OS kernel to acknowledge the operation. Under high throughput or inside an async event loop, this stalls execution.

`ProcessKiller` solves this by spawning a detached, shell-bypassed background process on Windows:

```php
// Under the hood on Windows:
$cmd = "cmd /c start /B taskkill /F /T /PID {$pid}";
```

This commands exits in microseconds, handing the actual termination work over to the Windows OS in the background while your event loop continues to execute.

---

## Under the Hood: Strategies

`ProcessKiller` detects the host operating system family at runtime and dynamically chooses the most efficient strategy:

### 1. Windows (`taskkill`)
Issues a `taskkill /F /T` command to forcefully (`/F`) terminate the entire process tree (`/T`) of the given PID. This is executed as a fire-and-forget command bypassing the command shell.

### 2. Linux `/proc` Mapping
Rather than calling shell utilities iteratively, the library reads directly from the Linux `/proc` directory. It parses `/proc/$pid/stat` in a robust single-pass scan (safely handling process names with spaces or parentheses) to build a relational parent-child process map, then delivers a `SIGKILL` to the targets.

### 3. macOS / BSD `ps` Mapping
Constructs parent-child process maps in a single, high-performance execution of `ps -eo pid,ppid,pgid`, grouping processes and terminating them via negative-PID signals (`SIGKILL` to process groups) where applicable.

### 4. Fallback (Iterative `pgrep`)
If the system lacks direct access to `/proc` or standard `ps` configurations, the library falls back to progressive discovery using `pgrep -P` to discover child PIDs recursively.

---

## Development & Testing

Clone the repository and run the test and analysis suites:

```bash
# Install dependencies
composer install

# Run static analysis (PHPStan)
composer run analyze

# Run coding standard formatter (Pint)
composer run format

# Run test suite (Pest)
composer run test
```

---

## License

This library is open-source software licensed under the [MIT License](./LICENSE).