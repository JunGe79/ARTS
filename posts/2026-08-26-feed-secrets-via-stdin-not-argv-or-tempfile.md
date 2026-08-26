# Feed a secret to a child process over stdin, not argv or a temp file

**Question:** A parent process — a script, a scheduled job, an orchestrator — needs to hand a password or API key to a command it launches. The obvious ways are a command-line flag (`--key SECRET`) or a temp file (`--key-file /tmp/k`). Both leak. What's better?

**Answer:** Pipe the secret to the child's standard input. The parent writes it to the child's stdin, adds a newline, and closes the stream; the child reads it behind a `--key-stdin`-style flag. It never shows up in the process table (unlike argv) and never touches disk (unlike a temp file). It lives only in the in-memory pipe between the two processes and is gone the moment the child exits.
