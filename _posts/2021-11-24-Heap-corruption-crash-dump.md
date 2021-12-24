---
title: "Heap corruption crash dump collection"
date: 2021-11-24
categories:
  - Blog
tags:
  - Heap corruption
  - Debug diag
  - Crash dump
  - PageHeap
---
A heap corruption is defined as a problem that violates the heap's integrity and produces unexpected behavior in an application. It can happen if dynamic memory allocation and deallocation aren't handled appropriately in application code.

Some of the common causes of heap corruption -
- Buffer overrun (writing beyond the allocated memory) overrun and underrun
- Double free (freeing a pointer twice) 
- Old pointer reuse(reusing a pointer after being freed)
- Dangling Pointers

The process would not crash until the corrupted heap is accessed, but when a thread tries to use that corrupted block of memory in the heap, the application process will terminate, which makes it difficult to troubleshoot. If we capture a normal crash dump in this scenario, the problematic thread we’ll see in the dump, may not be the one that triggered the problem, but rather could be the victim thread.

So, to find the underlying cause and determine the actual issue, we should capture crash dump with pageheap enabled.

To know more about Heap, please refer below articles -
- <a title="What a Heap of ...(Part One)" href="https://techcommunity.microsoft.com/t5/ask-the-performance-team/what-a-heap-of-part-one/ba-p/372424">What a Heap of ...(Part One)</a>
- <a title="What a Heap of ...(Part Tow)" href="https://techcommunity.microsoft.com/t5/ask-the-performance-team/what-a-heap-of-part-two/ba-p/372431">What a Heap of ...(Part Two)</a>

<br/>
Following are the steps to capture heap corruption crash dumps using [Debug Diag](https://www.microsoft.com/en-us/download/details.aspx?id=102635) tool –

- Launch Debug Diag tool (Start -> Programs -> DebugDiag 2 Collection)
- If the Debug diag is already running, Start the Rules Wizard by activating the Rules tab and clicking the `Add Rule` button on the bottom of the tabbed dialog.  If this is the first time you have run DebugDiag.Collection.exe then the rules wizard will start automatically.
- Select the `Crash` option in the Select Rule Type Dialog and click `Next`
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/Crash.png" width="600" border="0"><br/>

- In the Select Target Type Dialog choose `A specific process` and click `Next`
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/SpecificProcess.png" width="600" border="0"><br/>

> Alternatively, we could also select `A specific IIS Web application pool` and then affected `Application Pool`.

- Select the `w3wp.exe` process from the list of processes and click `Next`
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/w3wp.png" width="600" border="0"><br/>

- In the Advanced Configuration dialog box click the `PageHeap Flags...` button
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/PageHeap.png" width="600" border="0"><br/>

- In the PageHeap Flags dialog choose the `Enable Full PageHeap` option and click `OK` and then `YES` for the warning prompt
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/FullPageheap.png" width="600" border="0"><br/>
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/PageheapWarning.png" width="600" border="0"><br/>

- In the Advanced Configuration dialog click `Next`.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/AdNext.png" width="600" border="0"><br/>

- In the `Select Dump Location and Rule Name` dialog, specify a name for this crash rule, and specify a directory where the dump files will be created (you may leave the defaults as well) and click `Next`.                                               
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/DDRuleName.png" width="600" border="0"><br/>

- In the Rule Completed dialog choose `Activate the rule now` and click `Finish` button.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/FinishRule.png" width="600" border="0"><br/>

- Once you click on `Finish` button on previous dialog, it will take you to the final screen where you will see the newly created rule details and make sure status should be `Active`.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/ActiveRule.png" width="800" border="0"><br/>

- At this point in time, w3wp.exe will need to be restarted to enable PageHeap for the process. Once w3wp.exe is restarted, you are done. You will need to wait for the next occurrence to have the dumps generated. The rule is now active and DebugDiag has attached DbgHost.exe to the w3wp.exe process and is debugging the process live. If heap corruption occurs, a full userdump will be generated.

> **Note:** Since pageheap is enabled per process, every instance of w3wp.exe running on the system will have pageheap. There is also a performance impact associated with pageheap that would cause the processing to slow down a bit due to heap verification.

<br/>

> If pageheap dump is not much helpful and you want to retrieve more detailed info about the heap corruption to understand the root cause of the issue and make the code fix easier, you can use Application Verifier in combination with Debug Diagnosis to capture the dumps.

**Other alternative data capturing Options:** <br/>
[Capturing Dumps using Debug diag and Application Verifier. ](https://techcommunity.microsoft.com/t5/iis-support-blog/debugging-heap-corruption-with-application-verifier-and/ba-p/376844).
Application Verifier can be downloaded from [here](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier)

**OR**

Capture Heap corruption crash dumps using GFlag -
<a title="Steps to capture Heap corruption crash dumps using GFlag. " href="https://web.archive.org/web/20190128135459/https:/blogs.msdn.microsoft.com/webdav_101/2010/06/22/detecting-heap-corruption-using-gflags-and-dumps/">Steps to capture Heap corruption crash dumps using GFlag.</a>
GFlag can be downloaded from [here](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/gflags).


Hope it helps! :slightly_smiling_face: