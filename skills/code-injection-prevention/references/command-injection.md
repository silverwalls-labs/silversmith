# Command injection

An attacker injects shell metacharacters through untrusted input that is
passed to a system command via a shell interpreter. The shell parses the
attacker's input as part of the command — executing arbitrary commands,
piping data, or chaining destructive operations. Passing arguments as a
list bypasses the shell entirely: the OS receives each argument as a
discrete value, and no parsing occurs.

The principle is language-agnostic: use the list/array form of command
execution in every language. The examples below use Python; the same
pattern applies to Node.js (`execFile` over `exec`), Go
(`exec.Command`), Ruby (multi-argument `system`), Java
(`ProcessBuilder`), etc.

## ❌ Before (vulnerable)

```python
import subprocess

# UNSAFE — shell=True passes the entire string to /bin/sh.
# An attacker can supply: filename = "file.txt; rm -rf /"
# and the shell executes both commands.

def count_lines(filename):
    result = subprocess.run(
        f"wc -l {filename}",
        shell=True,
        capture_output=True,
        text=True,
    )
    return result.stdout
```

`shell=True` tells the OS to invoke `/bin/sh -c "..."` with the entire
string — semicolons, pipes, backticks, `$()`, and every other shell
metacharacter are live.

## ✅ After (safe)

```python
import subprocess

# SAFE — argument list. Each element is passed as a separate argv
# entry; the OS never invokes a shell, so metacharacters are inert.

def count_lines(filename):
    result = subprocess.run(
        ["wc", "-l", "--", filename],
        capture_output=True,
        text=True,
    )
    return result.stdout
```

The list form calls the executable directly without a shell — the
filename is a single argument regardless of its content. The `--`
sentinel ends option parsing, so a filename starting with `-` is not
mistaken for a flag (argument injection).

## Key points

- **No `shell=True` with external input.** If a shell feature is
  genuinely required (pipes, globbing, redirection), hardcode the shell
  fragment and pass untrusted values as separate arguments — never as
  part of the command string.
- **`shlex.quote()` is a fallback, not a fix.** Quoting is fragile and
  shell-specific (POSIX vs. Windows `cmd.exe`). The argument-list form
  avoids the problem entirely — prefer it unconditionally.
- **Equivalent dangers in other languages:** Node.js `child_process.exec()`
  invokes a shell; use `execFile()` or `spawn()` with an argument array.
  Ruby's backtick syntax and single-argument `system()` invoke a shell;
  use `system("cmd", arg1, arg2)`. Go's `exec.Command` is safe by
  default — do not wrap it in `sh -c`.
- **End-of-options `--`.** Many commands interpret arguments starting
  with `-` as flags. If untrusted input can start with a dash, place
  `--` before it to stop option parsing — `["rm", "--", filename]`
  ensures `filename = "-rf"` is treated as a literal path.
- **Environment variables count.** If an attacker controls an env var
  that a command reads (e.g., `PATH`, `LD_PRELOAD`), they can hijack
  execution even without shell metacharacters. Sanitize the environment
  when spawning subprocesses in sensitive contexts.
- **Windows bat/cmd caveat (BatBadBut, CVE-2024-24576).** On Windows,
  invoking a `.bat` or `.cmd` file — even via the list form — silently
  invokes `cmd.exe`, which parses metacharacters in the arguments. The
  list form is safe for `.exe` binaries, but batch scripts need explicit
  argument escaping or should be avoided as entry points for untrusted
  input.
