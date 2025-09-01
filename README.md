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
