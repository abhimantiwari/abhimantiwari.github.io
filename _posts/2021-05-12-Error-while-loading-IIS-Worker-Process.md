---
title: "Error while loading IIS Worker Processes."
#title: "Welcome to Jekyll!"
date: 2021-05-12
#date: 2019-04-18T15:34:30-04:00
categories:
  - blog

tags:
  - IIS
  - Worker Process
---
Sometimes we see failure while loading the Worker Process feature in Internet Information Service (IIS). It fails with different errors something like - `There was an error while performing this operation. Details : Category Does not exist”.`  

<p><a href="https://abhimantiwari.github.io/Content/WorkerProcessError.png"></a> </p>

<p>If you look at the Application event log may see event logs from `Perflib` source, something like this - </p>

```ruby
1. The Open Procedure for service "BITS" in DLL "C:\\Windows\System32\bitsperf.dll" failed. Performance data for this service will not be available. The fist four bytes (DWORD) of the Data selection contains the error code.

2. The Collect Procedure for the "PerfProc" service in DLL "C:\Windows\system32\perfproc.dll" generated an exception or returned an invalid status. The performance data returned by the counter DLL will not be returned in the Perf Data Block. The first four bytes (DWORD) of the Data section contains the exception code or status code.
```

<p>You can check if below registry location have any key for `DisablePerformanceCounters`.
`HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\%servicename%\Performance`
</p>

<p>Open the command prompt and Run this command - `lodctr /q:PerfProc` and see the output. If the output shows `Performance Counters (Disabled/ Enabled)`. </p>

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

<p>If the above output returns Performance Counters (Disabled). it can be fixed by running the below command. After running the below command run `lodctr /q:PerfProc` again and see if the Performance counter is enabled now (as shown below). If so, you can reload the IIS and see if workder process is loading fine now. </p>

```ruby
lodctr /e:PerfProc

lodctr /q:PerfProc
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

<p>If Worker Process still fails to load even if the `Performance Counters is (Enabled)` or after executing above commands. We will need to rebuld all the perfomance counters by execuing below commands -</p>

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

<p>Reference link - </p>
<li><a title="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters" href="https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters">https://docs.microsoft.com/en-us/troubleshoot/windows-server/performance/manually-rebuild-performance-counters#rebuild-all-performance-counters-including-extensible-and-third-party-counters</a></li>