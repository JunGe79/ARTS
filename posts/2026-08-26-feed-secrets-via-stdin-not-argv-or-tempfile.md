# Feed a secret to a child process over stdin, not argv or a temp file

**Question:** A parent process — a script, a scheduled job, an orchestrator — needs to hand a password or API key to a command it launches. The obvious ways are a command-line flag (`--key SECRET`) or a temp file (`--key-file /tmp/k`). Both leak. What's better?

**Answer:** Pipe the secret to the child's standard input. The parent writes it to the child's stdin, adds a newline, and closes the stream; the child reads it behind a `--key-stdin`-style flag. It never shows up in the process table (unlike argv) and never touches disk (unlike a temp file). It lives only in the in-memory pipe between the two processes and is gone the moment the child exits.

## Why not the command line

`--key SECRET` puts the secret in the child's argv, which is readable by anyone on the box while the process runs:

```
$ ps aux | grep mytool        # the full command line, secret included
$ cat /proc/<pid>/cmdline      # same thing, straight from the kernel
```

Argv also tends to end up in shell history, audit / EDR logs, and the parent job's own log. It is not a secret channel.

## Why not a temp file

`--key-file /tmp/k` moves the secret onto disk. Now you own a file that:

- is readable by anyone who can read the path (a permissions slip is easy),
- survives a crash or a kill — so you need cleanup, and a sweeper for the stale files you inevitably leave behind,
- can be picked up by a backup, a snapshot, or a container image layer.

A file beats argv, but it turns "pass a secret" into "manage a secret's lifetime on disk."

## stdin: a pass-through pipe

The parent starts the child, writes the secret to its stdin, and closes it.

Parent (bash):
```bash
printf '%s\n' "$secret" | mytool --key-stdin
```

Parent (Python):
```python
import subprocess
p = subprocess.Popen(["mytool", "--key-stdin"], stdin=subprocess.PIPE, text=True)
p.communicate(secret + "\n")      # secret -> child stdin, then EOF
```

Parent (Java, e.g. from an orchestrator):
```java
Process proc = new ProcessBuilder("mytool", "--key-stdin").start();
java.io.OutputStream os = proc.getOutputStream();
os.write(secret.getBytes("UTF-8"));
os.write('\n');                   // delimiter
os.close();                       // EOF -> the child's read returns
```

Child reads it:
```python
import sys
if args.key_stdin:
    secret = sys.stdin.readline().rstrip("\n")   # or sys.stdin.read()
```

The secret exists only in the anonymous pipe the OS set up between the two processes. It is not in argv, not on disk, and it is gone when the pipe closes.

## Things to get right

- **Close the stream (send EOF).** If the child does `sys.stdin.read()` and the parent never closes stdin, the child blocks forever. Write, then close.
- **Pick a clear precedence if you keep fallbacks.** It is fine to accept stdin *or* a file *or* a raw value, but resolve them in a fixed order and prefer stdin. If you must keep a file fallback, sweep stale key files on the way in.
- **Env vars are a step up from argv, but not as clean as stdin.** A child's environment is readable via `/proc/<pid>/environ` by its owner and is inherited by grandchildren. stdin is scoped to the one child you handed it to.
- **Don't echo it.** Make sure the child never logs the value it just read, and the parent never prints the command with the secret interpolated in.

## Takeaways

- Command-line arguments are visible in `ps` / `/proc/<pid>/cmdline` / logs — never pass secrets there.
- Temp files move the problem to disk: permissions, cleanup, crash-survival, backups.
- stdin passes the secret through an in-memory pipe: not in argv, not on disk, gone at exit.
- Parent: write the secret plus a newline, then **close** stdin (EOF). Child: read one line from stdin.
- If you keep a file fallback, prefer stdin and sweep stale key files.
