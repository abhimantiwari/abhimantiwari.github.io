---
title: "Capture dumps using Procdump for each instance of a Process"
date: 2021-11-15
categories:
  - Blog
tags:
  - Dumps
  - Procdump
  - PowerShell
  - IIS
---

PowerShell script to execute [Procdump](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump) on all instances of a given process (by name) to capture memory dumps.

It will automatically download Procdump tool and execute the same for all the running instances of given process e.g. w3wp and Zip the output (*.dmp files) into a folder (if you wish to).

Here are the steps to execute -
- Create a Folder `C:\dumps` (if folder doesn't exist script will create the same)
- Save the below script as `*.ps1` e.g.`BatchProcDump.ps1`
- Run the script from elevated PowerShell Terminal (as shown in screen shot below)


<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="650" alt="image" src="/Content/PSProcdump.png" width="880" border="0"><br/>

> Note: In script below the default command mentioned is `-accepteula -s 20 -n 2 -ma` to capture 2 dumps (-n 2) on each process in interval of 20 seconds (-s 20 //default is 10 seconds). You may also alter the command in script to trigger dump collection on something like High Memory utilization (`-ma -m 4000 -n 3`), High CPU utilization (`-ma -c 70 -s 10 -n 3`), Crash (`–ma –t`) or on some first chance exception (`-ma -e 1 “System.StackOverflowException”`) etc. To know more about these switches/parameters and descriptions, please refer [this](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump).

**Script-**
```ruby
#Script

Param(
    [Parameter(Mandatory=$True)]
    [string]$processName,
    [string]$procdumpArguments="-accepteula -s 20 -n 2 -ma",
    [string]$outputFolder = "c:\dumps",
    [Parameter(Mandatory=$True)]   
    [bool]$zipOutput = [System.Convert]::ToBoolean($zipOutput)
)

$procdumpURL = "https://download.sysinternals.com/files/Procdump.zip"
$procdumpEXE = "procdump"
$originalLocation = Get-Location

#REGEX
$outFolderRegex = "(?<pre>.*\s)(?!\d{1,})(?<outfolder>[^-][^-\s|.]*)\s?(?<post>\-.*)?"
#match 1 (pre) is whatever come before the out folder
#match 2 (outfolder) is the out folder
#match 3 (post) whatever is next


#EXEC
Write-Output ""
if (!$processName) {
    PrintUsage
    return
    }

if ($procdumpArguments -match $outFolderRegex) {
    $outputFolder = $Matches["outfolder"]
    $procdumpArguments = $procdumpArguments -replace $outFolderRegex, '$1$3'
    }
if (!(Test-Path -Path $outputFolder)) {
    New-Item -ErrorAction Ignore -ItemType directory -Path $outputFolder >$null 2>&1
}
Set-Location $outputFolder

#Download procdump if not exists
if (!(Get-Command $procdumpEXE -ErrorAction SilentlyContinue)) {
    $procdumpZip = $outputFolder + "\procdump.zip"
    Invoke-WebRequest $procdumpURL -OutFile $procdumpZip
    #Unzip procdump
    Unblock-File $procdumpZip
    Expand-Archive $procdumpZip -DestinationPath $outputFolder
    Remove-Item -Path $procdumpZip
    $procdumpEXE = ".\procdump.exe"
}

#Get PIDs 
$pids = (get-process -Name $processName).Id
if (!$pids.count) {
    Write-Output "Process not running"
    return
}

Write-Output "There are $($pids.count) running processes with name $($processName)"
Write-Output "Invoking procdump with arguments: $($procdumpArguments)"
Write-Output "Collecting dumps, please wait while dumps are being captured..."

#Call procdump for each PID
$pids |foreach {
        $outputLogFile = $outputFolder + "\procdump_$($_).txt"
        Start-Process $procdumpEXE -ArgumentList "$($procdumpArguments) $($_) $($outputFolder)" -RedirectStandardOutput $outputLogFile -NoNewWindow
}

#wait to return and set original location
Wait-Process -Name procdump
Write-Output "Dumps were written in $($outputFolder)!"

#zip and delete temp folder
if ($zipOutput) {
    if (!(Get-Command Compress-Archive -ErrorAction SilentlyContinue)) {
        Write-Output "Compress-Archive cmdlet not found. Data compression will be skipped."
    }
    else {
        Write-Output "Compressing data..."
        Compress-Archive -Path "$($outputFolder)\*.dmp" -Update -DestinationPath "$($outputFolder)\$($processName)-dumps.zip" -CompressionLevel Fastest
        Get-ChildItem $outputFolder | Where { $_.Name -match ".*\.dmp" } | Remove-Item -ErrorAction Ignore >$null 2>&1
        Write-Output "Dumps were compressed and deleted. The ZIP file is located in $($outputFolder)"
    }
}

#QUIT
Set-Location $originalLocation
Write-Output ""


function Unzip($file, $destination)
{
     $shell = new-object -com shell.application
     $zip = $shell.NameSpace($file)
     $items = $zip.items() | Where-Object {$_.Name -match ".*\.exe"}
     foreach($item in $items)
     {
        $shell.Namespace($destination).copyhere($item)
     }
}

#print
function PrintUsage() {
    Write-Output "Usage: BatchProcDump.ps1 [-processName] (-procdumpArguments) (-outputFolder) (-zipOutput)"
    Write-Output ""
    Write-Output "-processName = target process name"
    Write-Output "-procdumpArguments = procdump command line arguments. Default value is: -accepteula -ma -n 2 -s 20\n"
    Write-Output "-outputFolder = sets the output folder !! If not already specified in the procdump arguments !!"
    Write-Output "-zipOutput = archives the dump files into a zip file and deletes the uncompressed files !! Requires PS v5"

    Write-Output ""
}
```

HTH!