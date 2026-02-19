# Operational Runbook: Automated Windows User Profile Cleanup

**Version:** 1.0 (Stable) **Target System:** Windows 10 / 11 / Server 2016+ 
**Execution Method:** PDQ Deploy (PowerShell)

**Repository:** [Prune_Old_User_Profiles.ps1](https://github.com/Arthur-K-99/Scripts/blob/main/Prune_Old_User_Profiles/Prune_Old_User_Profiles.ps1)

---

## 1. Executive Summary

This utility automates the removal of stale user profiles on shared endpoints. Its primary goal is to reclaim disk space and maintain system hygiene without disrupting active users.

Unlike standard cleanup scripts that simply delete folders (which corrupts the Registry and leads to "Temporary Profile" errors), this solution uses the **CIM/WMI interface** to atomically uninstall the profile, removing both the file structure and the Registry keys simultaneously.

## 2. The "Triple-Tap" Safety Logic

Windows is notoriously bad at reporting when a user last logged off due to "Fast Startup" and hybrid shutdown states. Relying solely on Windows' reported `LastUseTime` can lead to the accidental deletion of active users.

To solve this, the script employs a **"Triple-Tap" Validation Strategy**:

1. **The Filter (Safety Net):** It automatically excludes System profiles (`Default`, `Public`), currently loaded profiles, and specific admin/service accounts.
2. **The WMI Check (The Bookkeeping):** It queries the system to see if the profile's `LastUseTime` is older than the configured threshold (Default: **30 Days**).
3. **The Disk Check (The Physics):** If WMI says the profile is old, the script physically checks the timestamps of two files inside the user's folder (`NTUSER.DAT` and `UsrClass.dat`).
    - **The Rule:** If _either_ of these files has been modified recently, the script **ABORTS** deletion. This confirms the user is active, even if Windows failed to update the logoff timestamp.

---

## 3. Configuration & Usage

### A. Script Configuration

At the top of the PowerShell script, you will find the user-configurable variables:

- **`$DaysThreshold`**: (Default: `30`) The number of days a profile must be inactive to be considered for removal.
- **`$DeleteMode`**:
    - `$false` (Default): **Dry Run / Audit Mode**. The script logs what it _would_ delete but takes no action.
    - `$true`: **Destructive Mode**. The script actively deletes profiles.
- **`$ProtectedPatterns`**: A list of usernames to _never_ delete (e.g., `"Administrator"`, `"Scanner_Service"`, `"$env:USERNAME"`).

### B. PDQ Deploy Setup

To deploy this via PDQ, create a new package with the following settings:

1. **Step Type:** PowerShell.
2. **Script:** Paste the code block provided in Section 5.
3. **Options Tab:**
    - **Run As:** Deploy User (Must be a Local Admin). _Do not use "Logged on User"._
    - **Error Mode:** Stop Deployment on Error.
4. **Conditions Tab:**
    - **Architecture:** **64-bit**. (Crucial: WMI calls are most reliable in 64-bit PowerShell).

---

## 4. Troubleshooting & FAQ

**Q: The script ran successfully, but no profiles were deleted.** **A:** Check the `$DeleteMode` variable. It defaults to `$false` for safety. You must change it to `$true` to enable deletion.

**Q: The log says "SKIPPED: WMI says old, but Registry Hive modified recently."** **A:** This is the safety system working. It means Windows _thought_ the user was inactive, but the script found evidence (file timestamps) that the user or a background process accessed that profile recently. The profile was saved to prevent data loss.

**Q: What is an "Orphan Found"?** **A:** This indicates a user folder was deleted manually in the past (via Explorer), but the Registry key was left behind. The script detects this mismatch and cleans up the "ghost" registry entry to prevent future login errors.

**Q: Can I run this while users are logged in?** **A:** Yes. The script checks the `Loaded` status of every profile. If a user is currently logged in, they are automatically skipped.

---

## 5. The Script (Production Ready)

```powershell
# ==========================================
# CONFIGURATION
# ==========================================
$DaysThreshold = 30
# SET THIS TO $true TO ENABLE ACTUAL DELETION
$DeleteMode = $false 
$LogPrefix = "[PROFILE-CLEANUP]"

# Calculates the date before which profiles are considered old
$CutoffDate = (Get-Date).AddDays(-$DaysThreshold)

# ==========================================
# EXCLUSIONS
# ==========================================
# 1. Always protect these specific paths
$ProtectedPaths = @(
    'C:\Users\Public'
    'C:\Users\Default'
    'C:\Users\Default User'
)

# 2. Dynamic Protections (The account running the script + Admin accounts)
$CurrentRunner = $env:USERNAME
$ProtectedPatterns = @(
    $CurrentRunner    # Protect the PDQ Runner/Service Account
    "Administrator"   # Protect the built-in Admin
    "LAPS_Admin"      # Example: Add other specific admin accounts here
)

# ==========================================
# EXECUTION
# ==========================================
Write-Output "$LogPrefix Starting scan. Threshold: $DaysThreshold days. (Cutoff: $($CutoffDate.ToString('yyyy-MM-dd')))"
Write-Output "$LogPrefix Run Mode: $(If ($DeleteMode) {'DESTRUCTIVE'} Else {'REPORT ONLY (Dry Run)'})"
Write-Output "$LogPrefix Script Runner: $CurrentRunner"

# Stats Counters
$Stats = @{ Removed = 0; Skipped = 0; Errors = 0; Orphans = 0 }

# Get Candidates
# Filter: Not Special, Not Currently Loaded, Must be in C:\Users (Avoids System Profiles)
try {
    $Candidates = Get-CimInstance -ClassName Win32_UserProfile -ErrorAction Stop | Where-Object {
        $_.Special -eq $false -and 
        $_.Loaded -eq $false -and 
        $_.LocalPath -like "C:\Users\*"
    }
}
catch {
    Write-Error "$LogPrefix FATAL: Could not query Win32_UserProfile. WMI may be broken on this host."
    exit 1
}

foreach ($UserProfile in $Candidates) {
    
    $Path = $UserProfile.LocalPath
    $SID = $UserProfile.SID
    $Username = Split-Path $Path -Leaf

    # CHECK 1: Exclusions
    if ($Path -in $ProtectedPaths -or ($ProtectedPatterns | Where-Object { $Username -like $_ })) {
        Write-Output "$LogPrefix SKIPPED: [$Path] matches exclusion list."
        $Stats.Skipped++
        continue
    }

    # CHECK 2: Primary WMI Age Check
    if ($Profile.LastUseTime -lt $CutoffDate) {
        
        $IsActuallyStale = $true

        # CHECK 3: Filesystem Validation (The "Triple-Tap")
        # We check both NTUSER.DAT and UsrClass.dat. If EITHER is new, we keep the profile.
        $HivePaths = @(
            (Join-Path $Path "NTUSER.DAT"),
            (Join-Path $Path "AppData\Local\Microsoft\Windows\UsrClass.dat")
        )

        # Check if the folder actually exists
        if (Test-Path $Path) {
            $NewestHiveDate = $null

            foreach ($Hive in $HivePaths) {
                if (Test-Path $Hive) {
                    $HiveDate = (Get-Item $Hive -Force).LastWriteTime
                    if ($null -eq $NewestHiveDate -or $HiveDate -gt $NewestHiveDate) {
                        $NewestHiveDate = $HiveDate
                    }
                }
            }

            if ($NewestHiveDate -and $NewestHiveDate -gt $CutoffDate) {
                Write-Output "$LogPrefix SKIPPED: [$Path]. WMI says old, but Registry Hive modified recently ($NewestHiveDate)."
                $IsActuallyStale = $false
                $Stats.Skipped++
            }
        } 
        elseif (-not (Test-Path $Path)) {
            Write-Output "$LogPrefix ORPHAN FOUND: [$Path] does not exist on disk but exists in WMI."
            $Stats.Orphans++
            # We let $IsActuallyStale remain true so we clean up the dead WMI entry
        }

        # ACTION: Delete
        if ($IsActuallyStale) {
            
            if ($DeleteMode) {
                Write-Output "$LogPrefix REMOVING: [$Path] | SID: $SID | Last Used: $($Profile.LastUseTime)"
                try {
                    # -Confirm:$false is vital for unattended scripts
                    $Profile | Remove-CimInstance -ErrorAction Stop -Confirm:$false
                    Write-Output "$LogPrefix SUCCESS: Removed $Path"
                    $Stats.Removed++
                }
                catch {
                    Write-Output "$LogPrefix ERROR: Failed to remove $Path. Exception: $($_.Exception.Message)"
                    $Stats.Errors++
                }
            }
            else {
                Write-Output "$LogPrefix [DRY RUN] WOULD REMOVE: [$Path] | SID: $SID | Last Used: $($Profile.LastUseTime)"
                $Stats.Removed++ # Count as removed for the report
            }
        }
    }
}

# Summary for PDQ Output
Write-Output "--------------------------------------------------"
Write-Output "$LogPrefix SUMMARY COMPLETE"
Write-Output "Processed: $($Stats.Removed) | Skipped: $($Stats.Skipped) | Orphans Found: $($Stats.Orphans) | Errors: $($Stats.Errors)"
Write-Output "--------------------------------------------------"
```