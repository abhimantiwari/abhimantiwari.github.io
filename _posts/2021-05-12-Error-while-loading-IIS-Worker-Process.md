---
title: "Error while loading IIS Worker Processes."
#title: "Welcome to Jekyll!"
date: 2021-05-12
#date: 2019-04-18T15:34:30-04:00
categories:
  - Blog

tags:
  - IIS
  - Worker Process
---
Sometimes we see failure dialog, while loading the Worker Process feature in Internet Information Service (IIS). It fails with different types of similar errors something like - `There was an error while performing this operation. Details : Category Does not exist”.`  

<!--p><a href="https://abhimantiwari.github.io/Content/WorkerProcessError.png"></a> </p -->
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="200" alt="image" src="/Content/WorkerProcessError.png" width="300" border="0">

<br/>
<p>If you look at the Application event log, you may see events from `Perflib` source, something like this - </p>

```ruby
1. The Open Procedure for service "BITS" in DLL "C:\\Windows\System32\bitsperf.dll" failed. Performance data for this service will not be available. The first four bytes (DWORD) of the Data selection contains the error code.

2. The Collect Procedure for the "PerfProc" service in DLL "C:\Windows\system32\perfproc.dll" generated an exception or returned an invalid status. The performance data returned by the counter DLL will not be returned in the Perf Data Block. The first four bytes (DWORD) of the Data section contains the exception code or status code.
```
<br/>
<p>You can verify if below registry location has any key with the name `DisablePerformanceCounters`.</p>
```ruby
Registry path: `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\%servicename%\Performance`
```

<p>Open the command prompt and run this command - `lodctr /q:PerfProc` and check the output if it shows `Performance Counters (Disabled/ Enabled)`. </p>

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


<p>If the above output returns `Performance Counters (Disabled)`. It can be fixed by running the below commands as shown below. </p>
 ```ruby
 lodctr /e:PerfProc
 ```
  
<p>To ensure that Performance Counters has been Enabled, run the below command, and see if the Performance counter is enabled now (as shown below). If so, you can reload the IIS and see if worker process is loading fine now. </p>

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
<p>If Worker Processes still fails to load, even if the `Performance Counters is (Enabled)` or after executing above commands. You will need to rebuild all the performance counters by executing below commands -</p>

```ruby
Open command prompt
cd c:\windows\system32
lodctr /R

cd c:\windows\sysWOW64
lodctr /R

Note - If the above commands fail (*it does sometimes*), just change the order of execution and it should run fine.


Resync the counters with Windows Management Instrumentation (WMI) by running below command -
WINMGMT.EXE /RESYNCPERF
```
<br/>
<br/>
<p>Reference link - </p>
<li><a title="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters" href="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters">https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters</a></li>
