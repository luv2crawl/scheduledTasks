# schtasks.exe Analysis
ProcMon didn't really help us identify where the COM interaction occurs or which internal function is called when you use schtasks.exe to create a scheduled task. Luckily there is a function named `CreateScheduledTask()` - seems like a good place to start!

## CreateScheduledTask()

In CreateScheduledTask(), the first significant function call is to CreateProcessOptions()
- If this call is successful, we end up calling either `CreateTaskVista()` or `CreateTaskLegacy()`
- mega_struct is of the type `struct __tagCreateSubOps`

### ProcessCreateOptions()

    ProcessCreateOptions(a1, a2, &mega_struct, v21, &v20) == 1 )

- this function appears to populate a shit load of members in the __tagCreateSubOps struct (&mega_struct in the above snippet)
- lots of references to known params for schtasks in the body of this function, seems to be comparing values to determine what to put into mega_struct
- mega_struct is initialized with 7332 bytes of memory just prior to the call to ProcessCreateOptions(), so there isn't any data in that memory buffer until after that function call returns


## CreateTaskVista()

A number of param checking function calls
- date/time params
- system account param parsing

    TaskService = GetTaskService(*(mega_struct + 1), *(mega_struct + 4), *(mega_struct + 5), &v72);

- name suggests getting a handle to some "service" (duh)
- several calls to COM related functions: `CoInitializeEx`, `CoInitializeSecurity`, `CoCreateInstance`

[CoInitializeEx Reference](https://docs.microsoft.com/en-us/windows/win32/learnwin32/initializing-the-com-library)
- Any Windows program that uses COM must initialize the COM library by calling the CoInitializeEx function. Each thread that uses a COM interface must make a separate call to this function.


[CoInitializeSecurity Reference](https://docs.microsoft.com/en-us/windows/win32/com/setting-processwide-security-with-coinitializesecurity)
- The CoInitializeSecurity function allows you to control complex security scenarios by setting security for an application programmatically

[CoCreateInstance Reference](https://docs.microsoft.com/en-us/windows/win32/learnwin32/creating-an-object-in-com)
- After a thread has initialized the COM library, it is safe for the thread to use COM interfaces. To use a COM interface, your program first creates an instance of an object that implements that interface.
- In general, there are two ways to create a COM object: 1) The module that implements the object might provide a function specifically designed to create instances of that object. 2) Alternatively, COM provides a generic creation function named CoCreateInstance.

The call to `CoCreateInstance` seems to have some useful parameters to understand what's going on:

    v8 = CoCreateInstance(&CLSID_TaskScheduler, 0i64, 0x17u, &IID_ITaskService, &ppv);

Two GUID values of note here: 
- `CLSID_TaskScheduler` = 0f87369f-a4e5-4cfc-bd3e-73e6154572dd = the CLSID value for the TaskScheduler class, likely used to look up the InProcServer32 value for this COM class
- `IID_ITaskService` = 2FABA4C7-4DA9-4013-9697-20CC3FD40F85 = the interface id for the [ITaskService interface](https://docs.microsoft.com/en-us/windows/win32/api/taskschd/nn-taskschd-itaskservice)
- confirmed these guid values here: https://github.com/Maximus5/ConEmu/blob/master/src/ConEmu/TaskScheduler.cpp


![](/images/schtasks_com_guids.png)

    HRESULT CoCreateInstance(
        [in]  REFCLSID  rclsid,
        [in]  LPUNKNOWN pUnkOuter,
        [in]  DWORD     dwClsContext,
        [in]  REFIID    riid,
        [out] LPVOID    *ppv
        );

    [in] riid

    A reference to the identifier of the interface to be used to communicate with the object.

    [out] ppv

    Address of pointer variable that receives the interface pointer requested in riid. Upon successful return, *ppv contains the requested interface pointer. Upon failure, *ppv contains NULL.

The param descriptions tell us that the call to CoCreateInstance is going to `return a pointer to the ITaskService interface into ppv`. In other words, we now have a `pointer to an instance of a COM object that implements the ITaskService interface`.

Eventually make a call to ITaskService::NewTask 
- [ITaskService::NewTask Docs](https://docs.microsoft.com/en-us/windows/win32/api/taskschd/nf-taskschd-itaskservice-newtask): Returns an empty ITaskDefinition object to be filled in with settings and properties and then registered using the ITaskFolder::RegisterTaskDefinition method.
- [ITaskDefinition interface](https://docs.microsoft.com/en-us/windows/win32/api/taskschd/nn-taskschd-itaskdefinition) Once we have an empty ITaskDefinition object, we can set the registration info, triggers, and action for our scheduled task (each of these are properties of the ITaskDefinition object).
- [ITaskFolder::RegisterTaskDefinition](https://docs.microsoft.com/en-us/windows/win32/api/taskschd/nf-taskschd-itaskfolder-registertaskdefinition): Registers (creates) a task in a specified location using the ITaskDefinition interface to define a task.


### Example: Create a Scheduled task to run notepad at a specific time

[reference](https://docs.microsoft.com/en-us/windows/win32/taskschd/time-trigger-example--c---)

![](/images/com_taskcreation_steps.png)

# taskschd.dll Analysis

There are a number of COM object methods implemented in taskschd.dll that are called when an application wants to create a scheduled task. While `ITaskService::NewTask` seems like the natural place to start, this method is essentially a setup step. Dynamic analysis showed us that two registry keys and a file are created when we create a task with schtasks.exe, but none of these operations occur in `ITaskService::NewTask`. This aspect of COM can be a bit difficult to reverse because the client (schtasks) is responsible for stitching together calls to `ITaskService::GetFolder`, `ITaskService::NewTask`, functions related to populating ITaskDefinition object properties, and `ITaskFolder::RegisterTaskDefinition`.

This flow essentially boils down to schtasks 1) instantianting and populating an ITaskDefinition object and then 2) passing that populated object to `ITaskFolder::RegisterTaskDefinition`. With this in mind, we will begin our COM server-side analysis with the implementation of RegisterTaskDefinition in taskschd.dll.

1. The server side implementation of the ITaskFolder COM server is located in `C:\Windows\System32\taskschd.dll`
2. We can easily find an internal function named "RegisterTaskFolder", which is just a wrapper that passes several paramaters to another function named `TaskFolderImpl::Register`
3. `TaskFolderImpl::Register` makes a call to `RpcSession::RegisterTask`
4. `RpcSession::RegisterTask` uses NdrClientCall to make an RPC call

### RPC Call

Looking at the paramaters passed to NdrClientCall from `RpcSession::RegisterTask`, we learn the following:
- We are interacting with the [Task Scheduler Service Remoting Protocol (MS-TSCH)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tsch/849c131a-64e4-46ef-b015-9d4c599c5167), which has a interface ID of 86D35949-83C9-4044-B424-DB363231FD0C
- We are invoking procedure number 1, which corresponds to the procedure named [SchRpcRegisterTask](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tsch/849c131a-64e4-46ef-b015-9d4c599c5167)


# Locating the MS-TSCH RPC Server Implementation

## RPC Server Enumeration
Locate the file where the server-side implementation of the MS-TSCH RPC protocol is located using NtObjectManager

*ps1:*

    Import-Module -Name NtObjectManager

    $rpc = ls C:\Windows\System32\* | Get-RpcServer

    $guid = '86D35949-83C9-4044-B424-DB363231FD0C'

    $server = $rpc | Where-Object {$_.InterfaceId -eq $guid}

    $server | Select-Object -Property Name, InterfaceId, ProcedureCount, FilePath, IsServiceRunning, Client

*Output:*

    Name             : schedsvc.dll
    InterfaceId      : 86d35949-83c9-4044-b424-db363231fd0c
    ProcedureCount   : 20
    FilePath         : C:\Windows\System32\schedsvc.dll
    IsServiceRunning : True
    Client           : False

![](/images/ubpm.png)

## Procedures

$scmrServer = Get-RpcServer -DbgHelpPath 'C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\dbghelp.dll' -FullName C:\Windows\system32\services.exe | Where-Object {$_.InterfaceId -eq '86D35949-83C9-4044-B424-DB363231FD0C'}
$Proc0 = $scmrServer.Procedures | Where-Object {$_.ProcNum -eq 1}

# SchRpcRegisterTask Analysis

Function def:

    HRESULT SchRpcRegisterTask(
        [in, string, unique] const wchar_t* path,
        [in, string] const wchar_t* xml,
        [in] DWORD flags,
        [in, string, unique] const wchar_t* sddl,
        [in] DWORD logonType,
        [in] DWORD cCreds,
        [in, size_is(cCreds), unique] const TASK_USER_CRED* pCreds,
        [out, string] wchar_t** pActualPath,
        [out] PTASK_XML_ERROR_INFO* pErrorInfo
    );
    
`SchRpcRegisterTask` calls `RpcServer::RegisterTask`:

    v15 = RpcServer::RegisterTask(
        (RpcServer *)&RpcServer::s_singleton,
        path,
        xml,
        flags,
        sddl,
        logonType,
        cCreds,
        pCreds,
        0i64,
        pActualPath,
        pErrorInfo,
        hkey);

# SchRpcRegisterTask Analysis

IDA seems to be very confused about how to decompile the `RpcServer::RegisterTask` method. This seems to be the right way to rename it on the callee side:

    __int64 __fastcall RpcServer::RegisterTask(
        RpcServer *this,
        const unsigned __int16 *path,
        unsigned __int16 *xml,
        unsigned int flags,
        unsigned __int16 *sddl,
        unsigned int logonType,
        unsigned int cCreds,
        const struct _TASK_USER_CRED *pCreds,
        BYTE *a9,
        unsigned __int16 **pActualPath,
        struct _TASK_XML_ERROR_INFO **pErrorInfo,
        struct TschedSqm::V2Server::ITaskFeatures *hkey)

Summary of boring parts before we hit the meat:
- Lots of WPP related calls depending on specific param values
- Some conditional logic depending on logonType and cred related params --> ex. interact with SecretGuard depending on pCred value

### XML Handling?

`TaskXMLReader::ProcessXML`

JobStore::SaveTaskXML --> JobStore::SaveJobFile

# Miscellaneous Research Questions

- When does the .job file get written? 
- How are .job files in C:\Windows\System32\Tasks\ used?
- When do the two registry keys get written?
- How are the two registry keys used?
- Do I have to write both registry keys AND the .job file?
- How do internal functions access the default .job file path and the reg key paths?
- When are relevant windows event logs written?
    - Auditor::AuditJobOperation --> makes a call to AuthziLogAuditEvent, called by SchRpcEnableTask
    - ETW all over the place --> EventManager::EvtReport

