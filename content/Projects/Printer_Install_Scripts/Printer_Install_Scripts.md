Repo: [Printer_Install_Scripts](https://github.com/Arthur-K-99/Scripts/tree/main/Printer_Install_Scripts)
### 1. `Install-IPPPrinter.ps1`

```powershell
<#
.SYNOPSIS
    Installs a printer using the IPP Protocol (HTTP/631) and creates the necessary port and driver associations.

.DESCRIPTION
    Stages a driver from a specified INF file (local or UNC), registers it, and maps the printer 
    to an IPP URL (http://<IP>:631/ipp/print) using printui.dll.

.PARAMETER PrinterName
    The name of the printer to be created on the system.
.PARAMETER DriverName
    The exact name of the driver model as it appears inside the INF file.
.PARAMETER DriverInfPath
    The full path (UNC or Local) to the driver .INF file.
.PARAMETER PrinterIP
    The IP address of the printer.
.PARAMETER PortName
    Optional. Defaults to the standard IPP path: http://<IP>:631/ipp/print.
#>
[CmdletBinding()]
Param (
    [Parameter(Mandatory = $true)]
    [string]$PrinterName,

    [Parameter(Mandatory = $true)]
    [string]$DriverName,

    [Parameter(Mandatory = $true)]
    [ValidateScript({Test-Path $_ -PathType Leaf})]
    [string]$DriverInfPath,

    [Parameter(Mandatory = $true)]
    [string]$PrinterIP,

    [string]$PortName
)

$ErrorActionPreference = "Stop"

# Construct default IPP URL if PortName is not explicitly provided
if ([string]::IsNullOrWhiteSpace($PortName)) {
    $PortName = "http://$($PrinterIP):631/ipp/print"
}

Write-Verbose "Configuration :: Printer: $PrinterName | IP: $PrinterIP | Driver: $DriverName"

try {
    # --- Step 1: Cleanup Existing Instances ---
    if (Get-Printer -Name $PrinterName -ErrorAction SilentlyContinue) {
        Write-Host "[-] Printer '$PrinterName' exists. Removing to enforce clean configuration..."
        Remove-Printer -Name $PrinterName
    }

    # --- Step 2: Stage Driver via PNPUtil ---
    Write-Host "[*] Staging driver from: $DriverInfPath"
    $pnp = Start-Process pnputil.exe -ArgumentList "/add-driver `"$DriverInfPath`" /install" -Wait -PassThru
    
    if ($pnp.ExitCode -ne 0) {
        Write-Warning "PNPUtil returned exit code $($pnp.ExitCode). Check if driver requires a reboot or is already present."
    }

    # --- Step 3: Register Driver with Spooler ---
    Write-Host "[*] Registering Driver: $DriverName"
    if (-not (Get-PrinterDriver -Name $DriverName -ErrorAction SilentlyContinue)) {
        Add-PrinterDriver -Name $DriverName
    }
    else {
        Write-Host "    -> Driver already registered."
    }

    # --- Step 4: Install Printer (IPP Port) ---
    # Using rundll32 printui.dll because Add-PrinterPort does not natively support IPP creation easily.
    Write-Host "[*] Creating IPP connection to $PortName..."
    
    $PrintUIArgs = "/if /b `"$PrinterName`" /r `"$PortName`" /m `"$DriverName`" /z /u"
    $proc = Start-Process "rundll32.exe" -ArgumentList "printui.dll,PrintUIEntry $PrintUIArgs" -Wait -PassThru

    if ($proc.ExitCode -ne 0) {
        throw "PrintUI execution failed with code $($proc.ExitCode)."
    }

    # --- Step 5: Verification ---
    Start-Sleep -Seconds 3 # Allow spooler to refresh
    if (Get-Printer -Name $PrinterName -ErrorAction SilentlyContinue) {
        Write-Host "[+] Success: Printer '$PrinterName' installed successfully." -ForegroundColor Green
    }
    else {
        throw "Installation verification failed. Printer object not found."
    }
}
catch {
    Write-Error "FATAL: $($_.Exception.Message)"
    exit 1
}
```

### 2. `Set-DefaultPrinter.ps1`

```PowerShell
<#
.SYNOPSIS
    Disables Windows default printer management and forces a specific default printer.
    
.DESCRIPTION
    Sets the 'LegacyDefaultPrinterMode' registry key to prevent Windows from managing the default printer,
    then uses CIM/WMI to set the specified printer as default.
    
.PARAMETER PrinterName
    The exact name of the printer to set as default.
#>
[CmdletBinding()]
Param (
    [Parameter(Mandatory = $true)]
    [string]$PrinterName
)

$ErrorActionPreference = "Stop"

try {
    # --- Step 1: Disable "Let Windows manage my default printer" ---
    Write-Host "[*] Configuring registry to disable Windows printer management..."
    
    $RegPath = "HKCU:\Software\Microsoft\Windows NT\CurrentVersion\Windows"
    
    if (!(Test-Path $RegPath)) {
        New-Item -Path $RegPath -Force | Out-Null
    }

    # 1 = Disable Windows Management
    Set-ItemProperty -Path $RegPath -Name "LegacyDefaultPrinterMode" -Value 1 -Type DWORD -Force

    # --- Step 2: Validate Printer Existence ---
    $TargetPrinter = Get-CimInstance -ClassName Win32_Printer -Filter "Name='$PrinterName'" -ErrorAction SilentlyContinue

    if (-not $TargetPrinter) {
        throw "Printer '$PrinterName' not found. Ensure it is installed before running this script."
    }

    # --- Step 3: Set Default ---
    Write-Host "[*] Setting default printer to: $PrinterName"
    Invoke-CimMethod -InputObject $TargetPrinter -MethodName SetDefaultPrinter | Out-Null
    
    # --- Verification ---
    $CurrentDefault = Get-CimInstance -ClassName Win32_Printer -Filter "Default=$true"
    if ($CurrentDefault.Name -eq $PrinterName) {
        Write-Host "[+] Success: Default printer is now '$PrinterName'." -ForegroundColor Green
    }
    else {
        throw "Verification failed. Current default is still: '$($CurrentDefault.Name)'"
    }
}
catch {
    Write-Error "FATAL: $($_.Exception.Message)"
    exit 1
}
```

---

### Usage Guide

These scripts are "tool-agnostic." You can run them manually, wrap them in a batch file, or deploy them via endpoint management systems (Intune, SCCM, PDQ).

#### 1. Running Manually (PowerShell Console)

Since these are now parameterized, you must pass the arguments when calling the file.

**Installation:**

PowerShell

```
.\Install-IPPPrinter.ps1 `
    -PrinterName "HR_Color_01" `
    -DriverName "Brother MFC-L8930CDW Printer" `
    -DriverInfPath "\\nas\drivers\Brother\BRPRC23A.INF" `
    -PrinterIP "10.10.13.31"
```

**Setting Default:**

PowerShell

```
.\Set-DefaultPrinter.ps1 -PrinterName "HR_Color_01"
```

#### 2. Deployment Scenario (e.g., Intune/MDT)

If you are packaging this as a Win32 App, you would create a wrapper `Install.cmd` to execute the logic in one pass:

Code snippet

```
@echo off
REM --- Install Printer ---
Powershell.exe -ExecutionPolicy Bypass -File ".\Install-IPPPrinter.ps1" -PrinterName "HR_Color_01" -DriverName "Brother MFC-L8930CDW Printer" -DriverInfPath ".\gdi\BRPRC23A.INF" -PrinterIP "10.10.13.31"

REM --- Set Default (Must run in User Context) ---
REM Note: If deploying via Intune System context, the default printer script 
REM needs to be run separately as a user-context script or Remediation.
```

#### Key Technical Notes

- **Driver Store vs. Spooler:** The script separates `pnputil` (adding to the Windows Driver Store) and `Add-PrinterDriver` (registering with the Spooler). This distinction is critical for newer Windows builds (10/11) to avoid "Driver not found" errors during the `printui` call.
    
- **IPP Port Syntax:** The script defaults to `http://<IP>:631/ipp/print`. If you are using a different print server (e.g., CUPS or a specialized print appliance), you can override this using the optional `-PortName` parameter.

### Advanced Usage & Architecture

#### 1. Deployment Contexts

The installation logic is separated into two distinct scripts to accommodate Windows security boundaries in enterprise deployment tools (Intune, MECM/SCCM).

- **`Install-IPPPrinter.ps1` (System Context)**
    
    - **Execution Requirement:** Must be executed with Administrative/SYSTEM privileges.
        
    - **Function:** Performs operations requiring elevated access, including `pnputil.exe` (Driver Store modification) and `Add-PrinterDriver` (Spooler registration).
        
    - **Deployment Configuration:** Configure this script to run in the **System** context.
        
- **`Set-DefaultPrinter.ps1` (User Context)**
    
    - **Execution Requirement:** Must be executed with the credentials of the currently logged-on user.
        
    - **Function:** Modifies the `HKEY_CURRENT_USER` registry hive to disable Windows default printer management and invokes the `SetDefaultPrinter` CIM method for the active user session.
        
    - **Deployment Configuration:** Configure this script to run in the **User** context (e.g., via Intune Remediations or Logon Scripts).
        

#### 2. Network & Protocol Prerequisites

Successful execution requires specific network configurations to support the IPP stream.

- **Port Accessibility:** The client must have TCP reachability to the target printer on port **631**.
    
- **Protocol Verification:**
    
    - Standard IPP uses `http://<IP>:631/ipp/print`.
        
    - Secure IPP (IPPS) may require `https://` or specific URI paths (e.g., `/ipp/printer` or `/ipp`).
        
    - _Validation:_ Verify the correct URI path via the printer's Embedded Web Server (EWS) or by testing connection via a browser.
        

#### 3. Troubleshooting Matrix

Common failure codes returned by `printui.dll` or PowerShell execution.

| **Error Code**     | **Issue Description**                   | **Resolution**                                                                                                                                                   |
| ------------------ | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **0x00000057**     | `ERROR_INVALID_PARAMETER`               | The `$DriverName` argument does not strictly match the Model Name defined in the `.INF` file. Verify the string against the `[Manufacturer]` section of the INF. |
| **0x00000005**     | `ERROR_ACCESS_DENIED`                   | The script lacks Administrative privileges. Ensure `Install-IPPPrinter.ps1` is executed as Administrator or SYSTEM.                                              |
| **Offline/Error**  | Printer object created but unresponsive | The IPP URL is incorrect or the printer is rejecting the connection. Validate the URI path and ensure port 631 is open.                                          |
| **Driver Missing** | `Get-PrinterDriver` fails               | The driver was not successfully staged to the Driver Store. Check `pnputil` exit codes for failure.                                                              |

#### 4. Logging for Headless Deployment

Standard output (`Write-Host`) is often discarded during background deployments. Wrap execution in a transcript to capture failure states for auditing.

**Example Wrapper:**

PowerShell

```shell
$LogPath = "$env:ProgramData\Logs\PrinterInstall_$(Get-Date -Format 'yyyyMMdd').log"
Start-Transcript -Path $LogPath -Append

.\Install-IPPPrinter.ps1 -PrinterName "HR_Color_01" -DriverName "Brother MFC-L8930CDW Printer" -DriverInfPath ".\gdi\BRPRC23A.INF" -PrinterIP "10.10.13.31"

Stop-Transcript
```

#### 5. Uninstallation Procedure

To revert changes, remove the printer object and the associated driver from the spooler.

```powershell
# Removal Logic
Remove-Printer -Name "<PrinterName>" -ErrorAction SilentlyContinue
Remove-PrinterDriver -Name "<DriverName>" -ErrorAction SilentlyContinue
```