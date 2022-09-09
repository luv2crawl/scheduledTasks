# ProcMon



![procmon filter](/images/procmon_filter.png)

`SCHTASKS /create /tn "On demand demo3" /tr "C:\Windows\notepad.exe" /sc ONCE /sd 01/01/1910 /st 00:00`

Shortly after schtasks.exe parses the command line parameters supplied to it, looks up the CLSID `HKCR\CLSID\{0f87369f-a4e5-4cfc-bd3e-73e6154572dd}` and queries its values. The value at the main sub-key says `TaskScheduler class`

![look up the clsid value in the reg](/images/clsid_lookup.png)

The `InProcServer32` value tells us the name of the file where the server side implementation of this COM object lives. The value is `C:\Windows\System32\taskschd.dll` There are a number of procmon events recorded showing schtasks.exe interacting with this file, which I interpreted as inter-process communication. **TODO: how to validate this with proof**

Creates a registry key at `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree\On demand demo3`

![look up the clsid value in the reg](/images/reg_taskcache_tree.png)

Creates another registry key at `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tasks\{D7F87DE5-01F8-4A7A-969E-1A268FE4E5D8}`

![look up the clsid value in the reg](/images/reg_taskcache_tasks.png)

Creates file at `C:\Windows\System32\Tasks\On demand demo3`

![look up the clsid value in the reg](/images/task_file.png)