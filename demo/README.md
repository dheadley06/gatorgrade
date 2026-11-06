# Issue 218 Demo

## Setup

```bash
git clone -b issue218-demo https://github.com/dheadley06/gatorgrade.git
cd gatorgrade
uv sync --dev
```

`uv sync` installs this checkout in editable mode, so every command below runs
the code in this folder. In Git Bash on Windows, also run
`export PYTHONIOENCODING=utf-8`.

## Steps

### 1. A timed-out check is a failed check, not a crash

```bash
uv run gatorgrade --config demo/checks.yml --no-progress-bar
```

The issue: `subprocess.TimeoutExpired` was never caught, so a slow check crashed
with a traceback and hid the rest of the report.

Now `Hanging with output` and `Hanging with no output` fail with
`Command timed out after N second(s)`, `Last check` still runs, the summary shows
`Checks: 3/6 (50%)`, and the exit code is the normal failed-check status of 1.

### 2. Partial output is kept

Look at `Hanging with output` in the output from step 1.

The issue: output captured before a timeout should be retained.

Now `starting...`, printed before the limit, follows the timeout message.
`Hanging with no output` shows only the timeout message, so empty output is also
handled.

### 3. Both progress modes behave the same

```bash
uv run gatorgrade --config demo/checks.yml
```

The issue: the fix must work in both progress-bar modes.

Now the run gives the same result as step 1, with the progress bar on.

### 4. Invalid bytes do not crash a check

```bash
uv run gatorgrade --config demo/checks.yml --filter-query "Invalid bytes"
```

The issue: strict decoding raised `UnicodeDecodeError` on output that is not
UTF-8.

Now the check fails normally. Its diagnostic shows `out�put` and `err�or`,
covering invalid bytes in mixed standard output and standard error.

### 5. A timed-out setup command stops setup cleanly

```bash
uv run gatorgrade --config demo/setup_timeout.yml
```

The issue: setup had the same uncaught timeout.

Now setup still stops at the first failure: it shows `first ran` and
`second is stuck`, names the command that `timed out after 2 second(s)`, and
exits with 1 and no traceback. `third ran` never appears and no checks run.

### 6. Invalid bytes do not crash setup

```bash
uv run gatorgrade --config demo/setup_bytes.yml
```

The issue: setup decoded its output with an implicit UTF-8 assumption.

Now setup shows `out�put` and `err�or`, reports that the command failed, and
exits with 1.

### 7. The timeout is configurable

```bash
uv run gatorgrade --config demo/checks.yml --filter-query "Slow build" -t 2
```

```bash
uv run gatorgrade --config demo/checks.yml --filter-query "Slow build" -t 5
```

```bash
uv run gatorgrade --config demo/checks.yml --filter-query "output" -t 30
```

The issue: the timeout should be configurable per check or through a CLI option.

Now the 3-second `Slow build` is stopped by `-t 2` and passes with `-t 5`, so the
CLI option works, and `--filter-query` singles out which checks run. In the third
command, `Hanging with output` still stops after 2 seconds and
`Hanging with no output` after 1 second, because a per-check `timeout` wins over
the CLI option.

### 8. Invalid timeouts are rejected

```bash
uv run gatorgrade --config demo/checks.yml --timeout=0
```

```bash
uv run gatorgrade --config demo/bad_timeout.yml
```

The issue: the timeout must be validated as positive.

Now both give a clear error instead of running any check: a usage error for the
option and a configuration error for the file.

### 9. The effective timeout is in verbose output and reports

```bash
uv run gatorgrade --config demo/checks.yml --filter-query "Quick" -t 7 --verbose
```

```bash
uv run gatorgrade --config demo/checks.yml --report file json demo/report.json
```

```bash
grep -E '"(description|timeout|timed_out)"' demo/report.json
```

The issue: the effective timeout should be in verbose and report metadata.

Now the verbose panel shows `Timeout: 7`, and the report gives each check its
limit and whether it timed out: 2 and 1 for the hanging checks, and 300 for the
rest.

### 10. Existing configurations keep the same default

Look at `Quick check` and `Slow build` in the report from step 9.

The issue: behavior must not change unless the user opts in.

Now checks with no timeout settings still get 300 seconds and pass as before.

### 11. Regression tests and documentation

```bash
uv run pytest tests/output/test_output.py tests/input/test_set_up_shell.py tests/test_shell_runner.py
```

The issue: tests must cover timeouts, empty output, mixed output, invalid bytes,
and both progress modes, and the README must document the policy.

Now these tests cover each of those, and the README sections `Timeout` and
`Timeouts and Output Encoding` document the defaults, configuration, and
encoding.
