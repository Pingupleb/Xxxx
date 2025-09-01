# Xxxx

# ----------------------------
# Veeam Backup Status Reporter (VBR PowerShell)
# ----------------------------

# Load Veeam PowerShell (both legacy snap-in and modern module paths)
try { Add-PSSnapin VeeamPSSnapIn -ErrorAction SilentlyContinue } catch {}
try { Import-Module Veeam.Backup.PowerShell -ErrorAction SilentlyContinue } catch {}

$now = Get-Date
$reportDir = "C:\VeeamReports"
New-Item -ItemType Directory -Path $reportDir -Force | Out-Null
$csvPath  = Join-Path $reportDir ("Veeam-JobStatus-{0:yyyyMMdd-HHmm}.csv" -f $now)
$htmlPath = Join-Path $reportDir ("Veeam-JobStatus-{0:yyyyMMdd-HHmm}.html" -f $now)

function Get-Prop($obj, $name) { if ($null -eq $obj) { return $null }; $obj | Select-Object -ExpandProperty $name -ErrorAction SilentlyContinue }

$rows = @()

# (1) Hypervisor/Replica/Backup Copy/etc. jobs
$allVbrJobs = Get-VBRJob | Sort-Object Name
foreach ($job in $allVbrJobs) {
    $lastSession = Get-VBRBackupSession -Job $job | Sort-Object EndTime -Descending | Select-Object -First 1

    $jobResult = Get-Prop $lastSession 'Result'
    $jobState  = Get-Prop $lastSession 'State'
    $jobEnd    = Get-Prop $lastSession 'EndTime'
    $jobStart  = Get-Prop $lastSession 'CreationTime'
    $duration  = if ($jobStart -and $jobEnd) { New-TimeSpan -Start $jobStart -End $jobEnd } else { $null }

    $taskFails = 0; $taskWarns = 0; $taskOK = 0
    $taskRows = @()
    if ($lastSession) {
        $tasks = Get-VBRTaskSession -Session $lastSession
        foreach ($t in $tasks) {
            switch ($t.Result) {
                'Failed'  { $taskFails++ }
                'Warning' { $taskWarns++ }
                'Success' { $taskOK++ }
            }
            $taskRows += [PSCustomObject]@{
                JobName    = $job.Name
                JobType    = $job.JobType
                ItemName   = $t.Name
                ItemResult = $t.Result
                Start      = $t.CreationTime
                End        = $t.EndTime
            }
        }
    }

    $rows += [PSCustomObject]@{
        JobName      = $job.Name
        JobType      = $job.JobType
        LastRunStart = $jobStart
        LastRunEnd   = $jobEnd
        Duration     = if ($duration) { "{0:hh\:mm\:ss}" -f $duration } else { $null }
        JobResult    = $jobResult
        State        = $jobState
        ItemsOK      = $taskOK
        ItemsWarning = $taskWarns
        ItemsFailed  = $taskFails
        Source       = 'Get-VBRJob'
    }
}

# (2) Veeam Agent jobs managed by VBR (if present)
$hasComputerJobs   = (Get-Command Get-VBRComputerBackupJob -ErrorAction SilentlyContinue)
$hasAgentSessions  = (Get-Command Get-VBRComputerBackupJobSession -ErrorAction SilentlyContinue)
if ($hasComputerJobs) {
    $compJobs = Get-VBRComputerBackupJob | Sort-Object Name
    foreach ($cjob in $compJobs) {
        $csess = $null
        if ($hasAgentSessions) {
            $csess = Get-VBRComputerBackupJobSession -Job $cjob | Sort-Object EndTimeUTC -Descending | Select-Object -First 1
        }
        $jobResult = Get-Prop $csess 'Result'
        $jobEnd    = Get-Prop $csess 'EndTimeUTC'
        $jobStart  = Get-Prop $csess 'CreationTimeUTC'
        $duration  = if ($jobStart -and $jobEnd) { New-TimeSpan -Start $jobStart -End $jobEnd } else { $null }

        $taskFails = 0; $taskWarns = 0; $taskOK = 0
        if ($csess) {
            $tasks = Get-VBRTaskSession -Session $csess
            foreach ($t in $tasks) {
                switch ($t.Result) {
                    'Failed'  { $taskFails++ }
                    'Warning' { $taskWarns++ }
                    'Success' { $taskOK++ }
                }
            }
        }

        $rows += [PSCustomObject]@{
            JobName      = $cjob.Name
            JobType      = 'ComputerBackup'
            LastRunStart = $jobStart
            LastRunEnd   = $jobEnd
            Duration     = if ($duration) { "{0:hh\:mm\:ss}" -f $duration } else { $null }
            JobResult    = $jobResult
            State        = $null
            ItemsOK      = $taskOK
            ItemsWarning = $taskWarns
            ItemsFailed  = $taskFails
            Source       = 'Get-VBRComputerBackupJob'
        }
    }
}

# Save CSV and HTML
$rows | Sort-Object JobName | Export-Csv -NoTypeInformation -Encoding UTF8 -Path $csvPath

$summary = $rows | Select-Object JobName, JobType, LastRunEnd, JobResult, ItemsFailed, ItemsWarning, ItemsOK | Sort-Object JobName
$failedCount  = ($rows | Where-Object { $_.JobResult -eq 'Failed' -or $_.ItemsFailed -gt 0 }).Count
$warningCount = ($rows | Where-Object { $_.JobResult -eq 'Warning' -or $_.ItemsWarning -gt 0 }).Count

$header = @"
<h2>Veeam Backup Status Report</h2>
<p>Generated: $($now.ToString("yyyy-MM-dd HH:mm"))</p>
<p><strong>Jobs:</strong> $($rows.Count) &nbsp; | &nbsp; <strong>Failures:</strong> $failedCount &nbsp; | &nbsp; <strong>Warnings:</strong> $warningCount</p>
"@
$summary | ConvertTo-Html -Property JobName, JobType, LastRunEnd, JobResult, ItemsFailed, ItemsWarning, ItemsOK -Title "Veeam Backup Status" -PreContent $header |
    Out-File -FilePath $htmlPath -Encoding UTF8

$summary | Format-Table -AutoSize
Write-Host "`nCSV: $csvPath"
Write-Host "HTML: $htmlPath"

if ($failedCount -gt 0) { exit 2 }
elseif ($warningCount -gt 0) { exit 1 }
else { exit 0 }


# ===========================
# (3c) ALL HOSTS INVENTORY
# ===========================
# This block builds a unified "all hosts" list from:
#  - VBR Managed Servers (vCenter, ESXi stand-alone, Hyper-V, Windows/Linux, etc.)
#  - ESXi hosts from each vCenter
#  - Protected items from recent backup sessions (uses $inventory built in (3))
# Then it probes reachability and writes CSV + adds an HTML section.

# --- helper: richer reachability probe (ICMP + common TCP ports) ---
function Test-HostOnlinePlus {
    param(
        [Parameter(Mandatory=$true)][string]$Target,
        [int[]]$Ports = @(443,445,135,22)  # mgmt-ish defaults: HTTPS/SMB/RPC/SSH
    )
    $o = [PSCustomObject]@{
        Target    = $Target
        Online    = $false
        Ping      = $false
        RTTms     = $null
        Method    = $null
        OpenPorts = @()
    }
    # ICMP
    try {
        $r = Test-Connection -ComputerName $Target -Count 1 -ErrorAction Stop
        $o.Ping = $true; $o.Online = $true; $o.RTTms = ($r | Select-Object -First 1).ResponseTime; $o.Method = 'ICMP'
    } catch {}
    # TCP ports
    foreach ($p in $Ports) {
        try {
            $ok = Test-NetConnection -ComputerName $Target -Port $p -InformationLevel Quiet -WarningAction SilentlyContinue
            if ($ok) { $o.OpenPorts += $p }
        } catch {}
    }
    if (-not $o.Online -and $o.OpenPorts.Count -gt 0) { $o.Online = $true; $o.Method = 'TCP' }
    $o.OpenPorts = ($o.OpenPorts | Sort-Object | Select-Object -Unique)
    return $o
}

# --- collect Managed Servers (everything Veeam knows about) ---
$managedServers = @()
try {
    $managedServers = Get-VBRServer | Sort-Object Name  # includes VC, Hv, Esx (standalone), Windows, Linux, etc.
} catch {}

# --- collect ESXi hosts from each vCenter (for host-level visibility) ---
$vmwHosts = @()
try {
    $vCenters = $managedServers | Where-Object { $_.Type -eq 'VC' }
    foreach ($vc in $vCenters) {
        # List ESXi hosts known to this vCenter via Veeam inventory
        $esxi = Find-VBRViEntity -Server $vc -Servers
        foreach ($h in $esxi) {
            $vmwHosts += [PSCustomObject]@{
                Name      = $h.Name
                Category  = 'ESXiHost'
                Subtype   = 'ESXi'
                Source    = $vc.Name
                PortsHint = @(443)   # vCenter/ESXi mgmt port
            }
        }
    }
} catch {}

# --- protected items from recent backups (from (3) inventory you already have) ---
$protected = @()
if ($inventory -and $inventory.Count -gt 0) {
    foreach ($i in $inventory) {
        $protected += [PSCustomObject]@{
            Name       = $i.Device
            Category   = 'ProtectedItem'
            Subtype    = $i.LastResult
            Source     = $i.Jobs
            LastBackup = $i.LastBackup
            PortsHint  = @(445,135,22,443)
        }
    }
}

# --- managed servers normalized ---
$managed = @()
foreach ($ms in $managedServers) {
    # Choose reasonable default port hints by type
    $ports = switch ($ms.Type) {
        'VC'       { @(443) }
        'Esx'      { @(443) }
        'Hv'       { @(5985,5986,445,135) }  # WinRM + SMB/RPC
        'Windows'  { @(445,135,3389) }
        'Linux'    { @(22,443) }
        default    { @(443,445,135,22) }
    }
    $managed += [PSCustomObject]@{
        Name      = $ms.Name
        Category  = 'ManagedServer'
        Subtype   = $ms.Type
        Source    = 'VBR'
        PortsHint = $ports
    }
}

# --- unify (dedupe by Name, keep best info) ---
$hostRows = @()
$hostRows += $managed
$hostRows += $vmwHosts
$hostRows += $protected

# Some names may repeat across categories; keep unioned view
$byName = $hostRows | Group-Object Name
$allHosts = @()
foreach ($g in $byName) {
    $rows = $g.Group
    $categories = ($rows | Select-Object -ExpandProperty Category | Sort-Object -Unique) -join ', '
    $subtypes   = ($rows | Select-Object -ExpandProperty Subtype  | Where-Object { $_ } | Sort-Object -Unique) -join ', '
    $sources    = ($rows | Select-Object -ExpandProperty Source   | Where-Object { $_ } | Sort-Object -Unique) -join ', '
    $lastBackup = ($rows | Where-Object { $_.PSObject.Properties.Name -contains 'LastBackup' -and $_.LastBackup } |
                   Sort-Object LastBackup -Descending | Select-Object -ExpandProperty LastBackup -First 1)

    # Merge port hints
    $ports = @()
    foreach ($r in $rows) { if ($r.PortsHint) { $ports += $r.PortsHint } }
    $ports = ($ports | Sort-Object -Unique)
    if ($ports.Count -eq 0) { $ports = @(443,445,135,22) }

    # Probe
    $probe = Test-HostOnlinePlus -Target $g.Name -Ports $ports

    $allHosts += [PSCustomObject]@{
        Name        = $g.Name
        Categories  = $categories
        Subtypes    = $subtypes
        Sources     = $sources
        LastBackup  = $lastBackup
        Online      = if ($probe.Online) { 'Online' } else { 'Offline' }
        Method      = $probe.Method
        Ping        = $probe.Ping
        RTTms       = $probe.RTTms
        OpenPorts   = ($probe.OpenPorts -join ',')
    }
}

# --- save CSV & update HTML ---
$allCsv = Join-Path $reportDir ("Veeam-AllHosts-{0:yyyyMMdd-HHmm}.csv" -f $now)
$allHosts | Sort-Object Name | Export-Csv -NoTypeInformation -Encoding UTF8 -Path $allCsv

$allOfflineCount = ($allHosts | Where-Object { $_.Online -eq 'Offline' }).Count

# Add an "All Hosts" table to the HTML
$allHostsHtml = "<h3>All Hosts Inventory</h3>" + (
    $allHosts |
    Select-Object Name, Categories, Subtypes, Online, OpenPorts, LastBackup, Sources |
    ConvertTo-Html -Fragment
)

# Rebuild header to include all-hosts count
$header = @"
<h2>Veeam Backup Status Report</h2>
<p>Generated: $($now.ToString("yyyy-MM-dd HH:mm"))</p>
<p>
<strong>Jobs:</strong> $($rows.Count)
&nbsp;|&nbsp; <strong>Failures:</strong> $failedCount
&nbsp;|&nbsp; <strong>Warnings:</strong> $warningCount
&nbsp;|&nbsp; <strong>Offline devices (last $lookbackH h):</strong> $offlineCount
&nbsp;|&nbsp; <strong>All hosts offline:</strong> $allOfflineCount
</p>
<h3>Job Summary</h3>
"@

# Reuse $bodyJobs, $offlineHtml, $footer from earlier
($header + $bodyJobs + $offlineHtml + $allHostsHtml + $footer + "<p>CSV (all hosts): $allCsv</p>") |
    ConvertTo-Html -Title "Veeam Backup Status" |
    Out-File -FilePath $htmlPath -Encoding UTF8

# Treat any offline host as a warning (tweak as you like)
if ($allOfflineCount -gt 0 -and $failedCount -eq 0) { $warningCount = $warningCount + 1 }


