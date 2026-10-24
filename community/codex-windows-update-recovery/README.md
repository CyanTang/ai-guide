# Codex Windows Update Recovery: EPERM / `rename_staging` Case Study

**A verified Windows 11 troubleshooting case for Codex desktop:** after a client update, the app repeatedly displayed **"Unable to load organization settings"**. The initial launch was recovered, but the UI still became unresponsive when switching windows. Inspecting the logs revealed repeated failures to relocate bundled runtime executables (`EPERM` on `rename_staging`).

> **中文详细排障教程 / Full Chinese walkthrough:** [guide.zh-CN.md](./guide.zh-CN.md)

## What happened

| Stage | Observed evidence | What restored functionality |
| --- | --- | --- |
| Can't open Codex after updating | `bundled_executable_relocation_failed`, `EPERM`, `source=missing` for `codex.exe` | Copy **four matching EXE files** from the *locally installed official Codex package* to the exact version directory shown in the logs; verify SHA-256 |
| Codex opens but hangs after Alt+Tab / navigating threads | Repeated `node.exe` / `node_repl.exe` relocation errors under `runtimes/cua_node`, `main_thread_jank_snapshot` | Copy the **entire matching CUA Node runtime** directory from the same installed package, check counts/sizes and hashes of essential files |

**Verified case environment:** Windows 11 x64, Microsoft Store / WindowsApps Codex package version `26.1021.3705.0`, October 24, 2026.

**Outcome:** The reporter regained access to the main Codex UI after restoring the core binaries and reported smooth window switching after restoring the CUA Node runtime. A full post-repair log audit was **not** performed, so the underlying cause of `EPERM` and long-term reliability are **not proven**.

## Symptoms and signals

```text
bundled_executable_relocation_failed
errorCode=EPERM
operation=rename_staging
component=codex  executablePath=null  source=missing
component=node   source=missing
main_thread_jank_snapshot  processRole=browser_main
```

The UI error may suggest an organization-policy or network problem, but in this case a *local executable relocation failure* was the concrete blocker. The case also had separate network / TLS warnings; those were **not confirmed** as the cause of the launch/focus hangs.

## Safe diagnostic sequence

1. Stop active Codex tasks and quit the desktop client completely.
2. Record the current package version with `Get-AppxPackage -Name OpenAI.Codex`.
3. Inspect the latest desktop log for `source=override`, `source=missing`, `bundled_executable_relocation_failed`, `EPERM`, `destinationPath`, and `main_thread_jank_snapshot`.
4. If `CODEX_CLI_PATH` overrides the bundled CLI, back it up and assess whether it should be removed. **Removing it alone did not fix this case.**
5. For a matching `source=missing` runtime, verify the **current official installation source**, the **actual destination hash from the log**, and a matching version before restoring the relevant components.
6. Reopen Codex, test recent sessions and Alt+Tab repeatedly. If still failing, obtain *new* logs; do not repeat copying blindly.

### Read-only PowerShell checks

```powershell
Get-AppxPackage -Name OpenAI.Codex |
    Select-Object Name,Version,InstallLocation,PackageFamilyName

foreach ($scope in 'Process','User','Machine') {
    [pscustomobject]@{
        Scope = $scope
        CODEX_CLI_PATH = [Environment]::GetEnvironmentVariable('CODEX_CLI_PATH', $scope)
    }
}
```

Detailed, **version-locked** commands reproducing the successful steps are provided in [the Chinese guide](./guide.zh-CN.md).

## Important safety limitations

- **Never reuse the sample version/hash directory for a newer Codex release.** Read the fresh logs first. The scripts intentionally reject mismatched package versions.
- Don't delete `%USERPROFILE%\.codex` or your project folders; they may contain account/session state or work.
- Don't change ownership of `C:\Program Files\WindowsApps`, disable Windows Defender, or bypass TLS certificate verification.
- Don't assume `EPERM` proves antivirus interference. File locking, permissions or an application deployment bug are all possible and were not isolated in this case.
- **No original screenshots or raw logs are published**: they may contain personal paths, project details and credentials.

## Publishing details

**中文标题：** Codex Windows 更新后无法加载组织设置、切窗卡死：EPERM 故障排查与实测修复  
**Suggested project description:** Windows Codex desktop update recovery: a verified EPERM/rename_staging troubleshooting case with sanitized logs, diagnosis and PowerShell recovery steps.  
**Topics:** `codex` · `windows-11` · `powershell` · `troubleshooting` · `electron` · `eperm`

This is an **independent community case study**, not an official OpenAI repair guide or a universal workaround.

## License

MIT License for the original documentation and scripts in this folder. See [LICENSE](./LICENSE).
