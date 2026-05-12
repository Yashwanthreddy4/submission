# Part 3 - Prompt Preparation

## Selected PR for Prompt Preparation

### beets/pull/3478 (Fetchart Concurrency Fix)

## 3.1.1 Repository Context

`beetbox/beets` is a Python-based command-line music library management system designed to organize and maintain audio collections efficiently. The application scans music files, retrieves metadata from online services such as MusicBrainz and Discogs, and arranges files using customizable directory templates. The project is built around a SQLite-backed library model (`beets.library`) along with a modular plugin system where plugins interact through centralized event hooks managed by `beets.plugins`.

### Intended Users

The software is mainly intended for music collectors, archivists, DJs, and developers who need flexible and automated media management. Its user base ranges from individual users managing personal collections to organizations maintaining very large music archives.

### Problem Domain

The repository focuses on digital media asset management while ensuring data safety and non-destructive operations. Since the system supports concurrent imports and external file operations, it must safely handle simultaneous read/write access without corrupting user data. A key design goal of beets is maintaining reliable and reproducible library states.

### Architecture Overview

The project follows a layered architecture structure:

- **UI Layer:** `beets.ui` manages command-line interactions and user prompts.
- **Application Layer:** `beets.importer` controls the import workflow, including metadata fetching and candidate selection.
- **Domain Layer:** `beets.library` models albums and tracks, while `beets.autotag` communicates with metadata providers.
- **Infrastructure Layer:** `beets.util` contains filesystem utilities, database helpers, and operating system abstractions.
- **Plugin System:** Plugins inherit from `beets.plugins.BeetsPlugin` and subscribe to events such as `album_imported`, `write`, and `import_task_choice`.

The `fetchart` plugin, located in `beetsplug/`, is responsible for downloading album artwork from multiple sources including local filesystems and online services.

This pull request mainly affects both the plugin layer and infrastructure layer because it modifies shared filesystem utilities while improving concurrency handling inside the artwork-fetching workflow.

---

## 3.1.2 Pull Request Description

### Changes Introduced

This pull request improves concurrency safety in the `fetchart` plugin by introducing secure temporary file handling and protected artwork installation. The previous hash-based temporary filename generation mechanism is replaced with `tempfile.mkstemp()`, which creates unique temporary files using the album database ID as part of the filename. 

The PR also introduces a new utility function called `atomic_move_with_lock` inside `beets/util`, which ensures that artwork files are moved safely into their final destination while protected by operating system-level file locks.

### Reason for the Changes

Earlier, temporary artwork filenames were generated using:

```python
hash(str(album)) % 10000
```

This approach created collisions during parallel imports, especially when multiple instances of beets processed the same album simultaneously. These collisions resulted in overwritten artwork files, filesystem exceptions such as `OSError`, and situations where albums received incorrect cover art.

The issue was more noticeable in automated environments running several concurrent import workers.

### Previous vs New Behavior

#### Previous Behavior
- Artwork was downloaded into temporary paths such as `/tmp/1234.jpg`.
- Concurrent imports could generate the same temporary filename.
- Final file movement operations were unprotected, leading to race conditions and occasional corrupted images.

#### New Behavior
- Temporary files are now generated using `mkstemp()` with names similar to:
  ```text
  /tmp/beets-42-xxxxxx.art
  ```
- Final artwork installation is protected using OS-level file locking.
- If a lock cannot be acquired, beets logs a warning and safely retries or falls back to sequential processing instead of crashing.

This update improves concurrency reliability while preserving beets’ lightweight dependency philosophy.

---

## 3.1.3 Acceptance Criteria

### Unique Temporary Paths
- Artwork downloads must use `tempfile.mkstemp()`.
- Temporary filenames should include the album database ID in the prefix or suffix.
- Tests should verify filename uniqueness by mocking `mkstemp()`.

### Atomic Move with Lock
- `atomic_move_with_lock` must acquire an exclusive lock on a companion `.lock` file before moving artwork files.
- The implementation should handle situations where another process already holds the lock.

### Cross-Platform Support
- POSIX systems should use `fcntl.lockf`.
- Windows systems should use `msvcrt`-based locking or a similar standard-library solution.
- CI tests must pass on Linux, macOS, and Windows environments.

### Resource Cleanup
- Temporary file descriptors must always be closed properly.
- Temporary files should be deleted if exceptions occur during file operations.
- Resource cleanup should be validated using simulated failure scenarios.

### Backward Compatibility
- Existing plugin configurations and fetchart settings must remain unchanged.
- Users not using parallel imports should experience no behavioral differences.
- Existing fetchart tests should continue passing without modifications.

---

## 3.1.4 Edge Cases

### Exhausted Temporary Directory
Under heavy concurrency, the temporary directory may run out of available space or inodes. In such cases, the implementation should catch `OSError`, log a meaningful warning, and skip artwork fetching for that album without stopping the full import process.

### Read-Only or Network Filesystems
If artwork is being stored on restricted or network-mounted filesystems, lock creation may fail differently across operating systems. The implementation should catch `PermissionError` and related exceptions, attempt a fallback atomic move, and notify the user that concurrency protection is reduced.

### Unicode and Path Encoding Issues
Album metadata containing non-ASCII characters may create filesystem issues on systems with strict path length limits or incompatible encodings. Temporary filename prefixes should therefore be sanitized into safe ASCII-compatible formats and handle encoding-related exceptions properly.

---

## 3.1.5 Initial Prompt

### Prompt

You are responsible for implementing a concurrency-safety improvement for the `fetchart` plugin in the `beetbox/beets` music library manager.

#### Repository Context

Beets is a Python command-line application used for organizing music collections. The `fetchart` plugin (`beetsplug/fetchart.py`) downloads album artwork from multiple sources and stores it in the user’s music library. Currently, temporary artwork files are created using a weak hash-based naming approach:

```python
hash(str(album)) % 10000
```

This creates race conditions and filename collisions during concurrent imports.

Core filesystem utilities are located in:
- `beets/util/__init__.py`
- `beets/util/fshandler.py`

The import process is managed by `beets/importer.py`.

The project avoids external dependencies wherever possible and mainly relies on Python’s standard library.

#### Tasks

### 1. Refactor Temporary File Creation
- Replace manual temporary filename generation in `beetsplug/fetchart.py`.
- Use `tempfile.mkstemp()` for secure and unique temporary file creation.
- Include the album database ID in the filename prefix:
  ```text
  beets-{album.id}-
  ```
- Ensure file descriptors are properly closed after creation.

### 2. Add `atomic_move_with_lock`
Create a new helper function:

```python
atomic_move_with_lock(src, dst)
```

Requirements:
- Perform atomic file moves while holding an exclusive OS-level lock.
- Use `fcntl.lockf` on POSIX systems.
- Use a Windows-compatible locking strategy with standard-library support.
- Add a timeout mechanism (recommended: 5 seconds).
- If locking fails, log a warning and continue with a fallback move operation.
- Use `try...finally` blocks to guarantee cleanup.

### 3. Update `beets/util/fshandler.py`
- Add internal platform-specific locking helpers.
- Keep helper methods private by prefixing them with `_`.

### 4. Testing
- Add tests in `test/test_fetchart.py` simulating concurrent downloads for the same album.
- Verify temporary filenames remain unique.
- Verify `atomic_move_with_lock()` is invoked correctly.
- Add cross-platform lock validation tests in `test/test_util.py`.

### 5. Documentation
Update `docs/plugins/fetchart.rst` to mention:
- Improved parallel import safety
- Atomic artwork installation behavior

#### Constraints
- Do not introduce new external dependencies.
- Ensure compatibility across Linux, macOS, and Windows.
- Handle unsupported locking scenarios gracefully.
- Maintain backward compatibility with existing plugin APIs and configurations.

#### Acceptance Checklist

- [ ] `tempfile.mkstemp()` is used for artwork temporary files.
- [ ] Temporary filenames include the album database ID.
- [ ] `atomic_move_with_lock()` supports POSIX and Windows.
- [ ] Concurrency tests pass successfully.
- [ ] No new dependencies are added.
- [ ] CI passes on Linux, macOS, and Windows.
