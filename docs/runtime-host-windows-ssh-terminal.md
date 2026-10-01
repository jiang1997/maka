<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# Windows SSH terminals: `ProgramData` and the exit-time resize race

This note records why adding a Windows computer over SSH still failed after
`#5821` resolved the terminal executable path, what the actual causes were, and
how the fixes were verified.

## Context

`#5821` fixed one Windows-only launch defect: node-pty matches the executable
name literally against `PATH` entries and never applies `PATHEXT`, so a bare
`ssh` always failed with `File not found` even when `ssh.exe` was installed.
The fix resolves `ssh.exe` / `scp.exe` to an absolute `.exe` path before
handing it to node-pty (`runtime-host-ssh-executable.ts`).

Dogfooding that fix on a Windows x64 machine surfaced two further defects that
blocked **Add Runtime Host** over SSH:

1. the interactive SSH process exited immediately, so xterm rendered nothing
   and target detection failed with `exited with code 255`;
2. the main process logged `Cannot resize a pty that has already exited`.

They are independent: (1) is a missing environment variable, (2) is a node-pty
timing difference on Windows.

## Cause 1 — `ssh.exe` exits 255 with no output

### Symptom

The "Connect to remote Runtime Host" dialog opened, the terminal stayed empty,
the connection ended, and the flow reported
`Remote Runtime Host target detection exited with code 255`. The same
destination worked from a normal Windows terminal.

### Evidence

Reproducing the app's exact launch (node-pty + `sshEnvironment()` + the app's
`-tt -o BatchMode=no` arguments) showed `ssh.exe` exiting 255 after emitting
only ConPTY control sequences — 16 bytes, no diagnostics.

Bisecting the environment by adding one dropped variable at a time:

| launch | environment | result |
| --- | --- | --- |
| `cmd /c echo` | filtered | works |
| `ssh -V` | filtered | exit 255, no output |
| `ssh -V` | full `process.env` | prints version |
| `ssh -V` | filtered + `ProgramData` | prints version |
| real connection | filtered + `ProgramData` | reaches the normal OpenSSH prompt |

The single required variable was **`ProgramData`** (typically
`C:\ProgramData`). Windows OpenSSH locates its system configuration and
system known-hosts files through it (`__PROGRAMDATA__\ssh\...`). Without it,
`ssh.exe` fails before it can print anything, which is why the failure looked
like a hang or a blank terminal.

`sshEnvironment()` intentionally forwards only a small allow-list so secrets
never reach the child process; `ProgramData` is a non-secret location variable
and belongs in that list.

### Fix

`sshEnvironment()` keeps `PROGRAMDATA`. The allow-list stays otherwise
unchanged, so credentials and unrelated variables are still dropped.

## Cause 2 — resize after the pty exited

### Symptom

```
Error occurred in handler for 'runtime-host-ssh-terminal:resize':
  Error: Cannot resize a pty that has already exited
    at WindowsPtyAgent.resize (.../node-pty/lib/windowsPtyAgent.js)
```

### Evidence

On Windows, node-pty sets the agent's exit code the moment the process exits,
but only emits `exit` about a second later — it flushes trailing output first
(`FLUSH_DATA_INTERVAL` in `windowsPtyAgent.js`). During that window Maka's
`ActiveTerminal` is still `active` with `phase === 'connecting'` and
`hasExited()` still returns false, so a renderer `ResizeObserver` update can
reach a pty that has already settled and `pty.resize()` throws.

The renderer already ignores the rejection; only the main process logged it.
The exit event remains the authority, so a settled pty makes resize a no-op.

### Fix

The resize handler wraps `pty.resize()` in a `try`/`catch`. Input validation
still throws for genuinely invalid sizes.

## Verification

- `apps/desktop/src/main/__tests__/runtime-host-ssh-terminal.test.ts`:
  - *ignores a resize that arrives after the pty exited but before its exit
    event* — fails with the original `Cannot resize a pty that has already
    exited` error when the `try`/`catch` is removed;
  - *keeps Windows OpenSSH location variables and drops secrets from the SSH
    environment* — fails when `PROGRAMDATA` is removed from the allow-list.
- Full file: 30 tests pass; `tsc -p tsconfig.main.json` and `biome check`
  are clean.
- Manual probe of the exact interactive launch (node-pty, filtered env,
  `-tt -o BatchMode=no`) reaches the ordinary OpenSSH host-key / password
  prompt once `ProgramData` is present.

## Not a code defect

Host-key trust and credentials are environmental. After the environment fix,
`ssh` reaches the normal OpenSSH prompts: an unknown host offers the
"continue connecting?" prompt, an untrusted-but-matching host proceeds, and a
host without key authentication asks for a password. Those are expected and
are handled by the existing terminal UI.
