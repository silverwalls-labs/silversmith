# Path traversal

An attacker manipulates a file path derived from untrusted input to
escape the intended directory — reading, writing, or deleting files
elsewhere on the filesystem. The classic vector is `../` sequences, but
symlinks, encoded separators (`%2f`, `%5c`), null bytes, and OS-specific
quirks (Windows alternate data streams, case folding) can all bypass
string-level checks. The only reliable defense is to canonicalize the
path and verify that the result is inside the allowed root.

The principle is language-agnostic: resolve to an absolute canonical
path (following symlinks), then check containment. The examples below
use Python; the same pattern applies to Node.js (`fs.realpathSync` +
`startsWith(root + path.sep)`), Go (`filepath.EvalSymlinks` +
`strings.HasPrefix(result, root + string(os.PathSeparator))`), Java
(`Path.toRealPath` + `startsWith`), etc. Note: Node.js `path.resolve`
and Go `filepath.Abs` do **not** resolve symlinks — they only normalize
the string — so they are insufficient on their own.

## Before (vulnerable)

```python
import os

UPLOAD_DIR = "/var/app/uploads"

# UNSAFE — joins the user-supplied filename directly.
# An attacker can supply: filename = "../../etc/passwd"
# and read arbitrary files outside the upload directory.

def read_upload(filename):
    path = os.path.join(UPLOAD_DIR, filename)
    with open(path) as f:
        return f.read()
```

`os.path.join` does not prevent traversal — it happily produces
`/var/app/uploads/../../etc/passwd`, which the OS resolves to
`/etc/passwd`.

## After (safe)

```python
import os

UPLOAD_DIR = "/var/app/uploads"

# SAFE — canonicalize with realpath, then verify containment.
# realpath resolves symlinks, .., and redundant separators.

def read_upload(filename):
    requested = os.path.realpath(os.path.join(UPLOAD_DIR, filename))
    root = os.path.realpath(UPLOAD_DIR)
    if not requested.startswith(root + os.sep) and requested != root:
        raise ValueError(f"Path escapes upload directory: {filename}")
    with open(requested) as f:
        return f.read()
```

`os.path.realpath` resolves the path to its canonical absolute form —
symlinks, `..` segments, and redundant separators are all eliminated.
The `startswith(root + os.sep)` check ensures the result is strictly
inside the root, not merely a prefix match (e.g., `/var/app/uploads_evil`
would falsely match `/var/app/uploads` without the separator).

## Key points

- **Canonicalize first, then check.** Rejecting `..` in the raw input
  is not sufficient — encoded variants, symlinks, and OS normalization
  can bypass string-level filters. Always resolve to the real path
  before comparing.
- **Append the separator.** `"/var/uploads".startswith("/var/uploads")`
  is true, but so is `"/var/uploads_evil".startswith("/var/uploads")`.
  Check against `root + os.sep` (or use `os.path.commonpath` /
  `pathlib.PurePath.is_relative_to`).
- **Absolute paths from user input.** If the user supplies an absolute
  path (`/etc/passwd`), `os.path.join("/root", "/etc/passwd")` returns
  `/etc/passwd` in Python — the root is silently discarded. Reject or
  strip leading separators before joining, and always canonicalize
  afterward.
- **Write and delete operations are higher stakes.** A traversal on a
  read leaks data; a traversal on a write or delete can overwrite
  configuration, plant a web shell, or destroy data. Apply the same
  containment check to every file operation.
- **Temporary files and downloads.** Files received from external
  sources (uploads, downloads, extracted archives) should land in a
  dedicated directory with containment checks — never in a shared or
  sensitive location.
- **TOCTOU between check and use.** In multi-tenant or shared-hosting
  environments, an attacker with local filesystem access can swap a
  path component for a symlink between the `realpath` check and the
  subsequent `open` call. Mitigate by opening the file first (via
  `O_NOFOLLOW` where supported, or `openat` relative to a directory
  fd), then verifying the opened file's real path — or by ensuring the
  allowed root is not writable by untrusted users.
