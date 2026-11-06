# Crestron CP4 Deployment Gotchas

> Distilled from debugging the `solventum` (Canteen) first deployment on a CP4.
> Each item is a mistake made, diagnosed, and confirmed fixed before being recorded here.

---

## 1. Build with `dotnet build`, not `dotnet publish`

**Symptom:** PROGLOAD unzips the CPZ but prints:
```
Error: Unable to find entry point for Simpl Sharp Pro program.
```

**Cause:** Running `dotnet publish -r linux-arm64 --self-contained` produces a generic
self-contained .NET 8 executable. It has no `CrestronControlSystem` entry point, so the
Crestron runtime rejects it.

**Fix:** Use `dotnet build -c Release`. The `Crestron.SimplSharpPro.SDK.Program` NuGet
package includes MSBuild targets that package the output into a `.cpz` automatically,
placed in `bin/Release/net8.0/<ProjectName>.cpz`. Use that file directly — do not manually
zip the publish output.

```powershell
dotnet build src\MyProject.Host\MyProject.Host.csproj -c Release --nologo --no-incremental
# cpz is at: src\MyProject.Host\bin\Release\net8.0\MyProject.Host.cpz
```

---

## 2. Upload the CPZ to the slot's program directory, not SFTP root

**Symptom:** PROGLOAD with `-F:filename` reports the file does not exist.
Manual `progload -p:1` (no `-F:`) reports "No LPZ/CPZ/ZIP file exists in the program
directory."

**Cause:** The SFTP root on a CP4 is the root of the Crestron virtual filesystem — it is
**not** NVRAM. Uploading to SFTP `/MyProgram.cpz` puts the file at the filesystem root,
not where PROGLOAD looks.

**Fix:** Upload to `/program0{slot}/` (e.g., `/program01/` for slot 1) and run
`PROGLOAD -P:{slot}` without `-F:`. PROGLOAD searches the slot's own directory
automatically.

```
SFTP path:      /program01/MyProject.Host.cpz
PROGLOAD cmd:   PROGLOAD -P:1
Console path:   \simpl\app01\MyProject.Host.cpz
```

---

## 3. The CP4 Linux filesystem is case-sensitive

**Symptom:** Program starts but immediately fails:
```
System.ArgumentException: The directory name '/USER/Solventum/config' does not exist.
```
or a `FileNotFoundException` on a config file that was definitely uploaded.

**Cause:** CP4 runs Linux with a case-sensitive ext4 filesystem. SFTP exposes paths in
their actual on-disk case. `/user/` and `/USER/` are different directories. Historically,
Crestron documentation used `UPPERCASE` paths (`/USER/`, `/NVRAM/`), but the actual SFTP
paths (and therefore what the application must use) are lowercase.

**Fix:** Use lowercase paths consistently in both the deploy script and any hardcoded paths
in the application.

```
SFTP creates:  /user/Solventum/config/
               /user/Solventum/webroot/
App must use:  /user/Solventum/config   ← match exactly
               /user/Solventum/webroot
```

If the application's default config path was written as `/USER/...`, correct it:
```csharp
// Wrong
var configDir = Environment.GetEnvironmentVariable("CONFIG_DIR") ?? "/USER/MyProject/config";
// Correct
var configDir = Environment.GetEnvironmentVariable("CONFIG_DIR") ?? "/user/MyProject/config";
```

---

## 4. Do not pipe commands to SSH — use SSH_ASKPASS or PuTTY instead

**Symptom:** Deploy script runs `PROGLOAD -P:1` via SSH and exits cleanly (no error), but
`progreg` shows no programs registered. The log contains:
```
Pseudo-terminal will not be allocated because stdin is not a terminal.
```
Running `progload -p:1` manually from an interactive SSH session works immediately.

**Cause:** The CP4 SSH daemon requires a pseudo-terminal (PTY) to execute anything —
its SSH server is a Crestron console, not a Unix shell, and it does not run commands in a
non-interactive context. When SSH is invoked with stdin connected to a pipe (e.g.,
`"PROGLOAD -P:1`r`nexit" | & ssh ...`), the OS reports stdin is not a terminal, so SSH
cannot allocate a PTY and prints the warning. Without a PTY the Crestron SSH session opens
and closes silently — nothing executes.

**Fix:** Pass `PROGLOAD` as an SSH argument and handle the password non-interactively so
that stdin remains a terminal (allowing PTY allocation). Two reliable options:

**Option A — PuTTY plink (recommended):**
```powershell
# plink supplies the password via -pw; PTY allocation is handled internally
& plink -batch -pw $pass admin@192.168.1.50 "PROGLOAD -P:1"
```

**Option B — OpenSSH + SSH_ASKPASS (Windows 11 built-in OpenSSH 8.4+):**
```powershell
# Write password to a temp file; askpass script echoes it when SSH prompts
$env:SSH_ASKPASS         = 'C:\Temp\askpass.cmd'   # @echo off / type pass.txt
$env:SSH_ASKPASS_REQUIRE = 'force'
& ssh -o BatchMode=no admin@192.168.1.50 "PROGLOAD -P:1"
```

The deploy script in this repo (`scripts/deploy.ps1`) uses plink for SSH commands and
psftp for file transfers, reading the password from `dev-config/credentials.json`.

> **Note:** `SSH_ASKPASS` is unreliable for SFTP on Windows — use psftp with `-pw` for
> file transfers instead (see gotcha #7). `SSH_ASKPASS` is only used as a fallback when
> PuTTY is not installed.

```powershell
# Wrong — piping to SSH prevents PTY allocation; PROGLOAD never runs
"PROGLOAD -P:$slot`r`nexit" | & ssh $sshTarget

# Correct — pass as argument; password handled out-of-band
Invoke-Ssh "PROGLOAD -P:$slot"   # wrapper uses plink or SSH_ASKPASS
```

---

## 5. webroot must match the path derived from configDir

**Symptom:** Program starts, page loads, but all static assets return 404.

**Cause:** `CompositionRoot` derives the webroot as `Path.Combine(configDir, "..", "webroot")`.
If the deploy script uploads the webapp to a differently-named directory (e.g., `webapp`
instead of `webroot`), the `StaticFileHost` cannot find any files.

**Fix:** Upload the webapp to the directory name that `CompositionRoot` derives. Given
`configDir = /user/MyProject/config`, the webroot is `/user/MyProject/webroot`.

```powershell
# deploy.ps1
$remoteBase   = "/user/MyProject"
$remoteConfig = "$remoteBase/config"
$remoteWebapp = "$remoteBase/webroot"   # must match configDir/../webroot
```

To override the default, pass `webrootDir` explicitly to `CompositionRoot.Build()` and
set it via an environment variable or a second config value.

---

## 6. PowerShell 5.1 reads `.ps1` files as Windows-1252 — avoid non-ASCII characters

**Symptom:** Running a `.ps1` script fails with a `MissingEndCurlyBrace` or
`Unexpected token` parse error. The offending line contains a Unicode character such as
an em-dash (`—`).

**Cause:** PowerShell 5.1 reads `.ps1` files using the system code page (Windows-1252 on
most Windows installs) unless the file has a UTF-8 BOM. The UTF-8 byte sequence for `—`
is `0xE2 0x80 0x94`. In Windows-1252, `0x94` maps to `"` (U+201D RIGHT DOUBLE QUOTATION
MARK), which terminates the string literal mid-line and produces a parse error.

**Fix:** Replace all non-ASCII characters in `.ps1` files with ASCII equivalents. A
quick grep to check before committing:

```powershell
# Find any non-ASCII bytes in your scripts
Select-String -Pattern '[^\x00-\x7F]' scripts\*.ps1
```

Alternatively, save the file with a UTF-8 BOM (but most editors and git configs strip
it, so ASCII is safer).

---

## 7. psftp requires `-batch` when run from a PowerShell script

**Symptom:** SFTP upload fails with no visible error — the script prints `SFTP upload failed`
but nothing between the `Using username "admin".` line and the exception:

```
  put: 6 file(s) ...
Using username "admin".
SFTP upload failed
```

**Cause:** Without `-batch`, psftp exits non-zero silently when invoked as a child process
from PowerShell. The exact trigger varies (banner handling, interactive prompt detection,
or console-mode checks in psftp), but the result is always the same: non-zero exit, no
diagnostic output to stdout or stderr.

A secondary consequence: if the failure causes psftp to retry authentication, the CP4
will lock the account after 3 failures (`The user account (admin) is blocked`).  Unlock
with `USERUNLOCK admin` from a Crestron Toolbox terminal.

**Fix:** Always include `-batch` in psftp options. Redirect stdout to a file and print it
on failure so the actual psftp error is visible:

```powershell
$psftpOpts = @('-batch', '-pw', $pass)

function Invoke-Sftp ([string[]] $commands) {
    $batch = Join-Path $env:TEMP 'deploy-sftp-batch.txt'
    $out   = Join-Path $env:TEMP 'deploy-sftp-out.txt'
    [IO.File]::WriteAllLines($batch, ($commands + 'bye'), [Text.Encoding]::ASCII)
    & psftp @psftpOpts user@host -b $batch > $out
    $ec = $LASTEXITCODE
    if ($ec -ne 0) {
        Write-Host "[psftp batch]:`n$([IO.File]::ReadAllText($batch))"
        if (Test-Path $out) { Write-Host "[psftp output]:`n$(Get-Content $out -Raw)" }
    }
    Remove-Item $batch, $out -ErrorAction SilentlyContinue
    return $ec
}
```

> **Note:** `-batch` suppresses psftp's interactive transfer progress display. This is
> acceptable — the script can print its own `queue:` lines before the session starts.

---

## Quick Reference: CP4 SFTP Path Map

| What | SFTP path | Crestron console path |
|------|-----------|------------------------|
| Program (slot 1) | `/program01/<Name>.cpz` | `\simpl\app01\<Name>.cpz` |
| Program (slot 2) | `/program02/<Name>.cpz` | `\simpl\app02\<Name>.cpz` |
| User files | `/user/<path>` | `/user/<path>` (case-sensitive) |
| SSH authorized keys | `/user/admin/.ssh/authorized_keys` | — |

The SFTP root is the root of the Crestron virtual filesystem. `/user/` is user-writable
storage. There is no direct SFTP path to NVRAM that is useful for program loading — use
the `/program0N/` slot directories instead.
