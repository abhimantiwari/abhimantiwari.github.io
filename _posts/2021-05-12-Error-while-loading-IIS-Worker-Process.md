---
title: "Error while loading IIS Worker Processes"
date: 2021-05-12

categories:
  - Blog

tags:
  - IIS
  - Worker Process
---
Sometimes we see failure dialog, while loading the Worker Process feature in Internet Information Service (IIS). It fails with different types of similar errors something like - `There was an error while performing this operation. Details : Category Does not exist.`  

<!--p><a href="https://abhimantiwari.github.io/Content/WorkerProcessError.png"></a> </p -->
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="200" alt="image" src="/Content/WorkerProcessError.png" width="300" border="0">

<br/>
If you look at the Application event log, you may see events from `Perflib` source, something like this -

```ruby
1. The Open Procedure for service "BITS" in DLL "C:\Windows\System32\bitsperf.dll" failed. Performance data for this service will not be available. The first four bytes (DWORD) of the Data selection contains the error code.

2. The Collect Procedure for the "PerfProc" service in DLL "C:\Windows\system32\perfproc.dll" generated an exception or returned an invalid status. The performance data returned by the counter DLL will not be returned in the Perf Data Block. The first four bytes (DWORD) of the Data section contains the exception code or status code.
```
<br/>
You can verify if below registry location has any key with the name `DisablePerformanceCounters`.
<br/>
```ruby
Registry path: `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\%servicename%\Performance`
```
<br/>
Open the command prompt and run this command - `lodctr /q:PerfProc` and check the output if it shows `Performance Counters (Disabled/ Enabled)`.

```ruby
Performance Counter ID Queries [PERFLIB]:
    Base Index: 0x00000737 (1847)
    Last Counter Text ID: 0x00004F00 (20224)
    Last Help Text ID: 0x00004F01 (20225)

[PerfProc] Performance Counters (Disabled)
    DLL Name: %SystemRoot%\System32\perfproc.dll
    Open Procedure: OpenSysProcessObject
    Collect Procedure: CollectSysProcessObjectData
    Close Procedure: CloseSysProcessObject
```

<br/>
If the above output returns `Performance Counters (Disabled)`. It can be enabled/fixed by running the below commands -
 ```ruby
 lodctr /e:PerfProc
 ```
 <br/>
  
<p>To ensure that Performance Counters has been Enabled now, run the below command, and see if the Performance counter is enabled now (as shown below). If so, restart IIS (Internet Information Services) Manager and load Worker Processes to see if it's loading fine now. </p>

```ruby
lodctr /q:PerfProc

Output:
Performance Counter ID Queries [PERFLIB]:
    Base Index: 0x00000737 (1847)
    Last Counter Text ID: 0x00004F00 (20224)
    Last Help Text ID: 0x00004F01 (20225)

[PerfProc] Performance Counters (Enabled)
    DLL Name: %SystemRoot%\System32\perfproc.dll
    Open Procedure: OpenSysProcessObject
    Collect Procedure: CollectSysProcessObjectData
    Close Procedure: CloseSysProcessObject
```

<br/>
If Worker Processes still fails to load, even if the `Performance Counters is (Enabled)` or after executing above commands. You will need to rebuild all the performance counters including extensible and third-party counter by executing below commands -

```ruby
1. Rebuild the counters:

Open elevated command prompt (Run as Administrator)
cd c:\windows\system32
c:\windows\system32>lodctr /R

cd c:\windows\sysWOW64
c:\windows\sysWOW64>lodctr /R

**Note** - If the above commands fail (*it does sometimes*), just change the order of execution and it should run fine.


2.Resync the counters with Windows Management Instrumentation (WMI) by running below command -
c:\windows\sysWOW64>WINMGMT.EXE /RESYNCPERF

3. Restart IIS (Internet Information Services) Manager and load Worker Processes
```
>PS - Sometimes, running `lodctr /R` may not recover all counters. If you notice this happening, verify the file `c:\windows\system32\PerfStringBackup.INI` contains the proper information. You can copy this file from an identical machine to restore the counters. There may be slight differences in this file from machine to machine. But if you notice a drastic difference in size, it may be missing information. Always create a backup copy before replacing. Refer below mentioned reference article for more details.
<br/>
<br/>
<p>Reference link - </p>
<li><a title="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters" href="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters">https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters</a></li>
