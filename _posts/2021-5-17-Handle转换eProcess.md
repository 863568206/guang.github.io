---
layout:     post
title:      Handle转换eProcess
subtitle:   c++
date:       2021-5-17-Handle转换eProcess
author:     YiMiTuMi
header-img: 
catalog: true
tags:
    - Driver
---

# Handle转换eProcess

我们通过[ObReferenceObjectByHandle](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-obreferenceobjectbyhandle),将句柄转化成所对应的类型可获取到相关数据。

包括： *ExEventObjectType, *ExSemaphoreObjectType, *IoFileObjectType, *PsProcessType, *PsThreadType, *SeTokenObjectType, *TmEnlistmentObjectType, *TmResourceManagerObjectType, *TmTransactionManagerObjectType, or *TmTransactionObjectType。


PS： [参考](https://blog.csdn.net/weixin_30693183/article/details/99975154)

调用方传递给驱动程序的句柄不会经过 I/O 管理器，因此 I/O 管理器不对这类句柄执行任何验证检查。决不要假设一个句柄有效；始终确保句柄拥有正确的对象类型、对于所需任务的合适的访问权、正确的访问模式，并且访问模式与请求的访问兼容。

　　驱动程序应该谨慎使用句柄，特别是那些从用户模式应用程序接收到的句柄。

　　第一，这种句柄特定于进程上下文，因此它们仅在打开句柄的进程中有效。当从不同的进程上下文或工作线程使用时，句柄可以引用不同的对象或者只是变得无效。

　　第二，在驱动程序使用句柄期间，攻击者可以关闭和重新打开句柄来改变其引用的内容。

　　第三，攻击者可以传入这样一个句柄来引诱驱动程序执行对于应用程序非法的操作，例如调用 ZwXxx 函数。对于这些函数的内核模式调用方，访问检查被跳过，因此攻击者可以使用这种机制绕过验证。

　　驱动程序还应该确保用户模式应用程序不能误用驱动程序创建的句柄。为一个句柄设置 OBJ_KERNEL_HANDLE 属性使其成为内核句柄，内核句柄可以在任何进程上下文中使用，但是只能从内核模式进行访问（对于传递给ZwXxx 例程的句柄，这特别重要）。用户模式的进程不能访问、关闭或替换内核句柄。


这里我们将其转化为 *PsProcessType 获取进程相关的信息：

	//将句柄转换为进程名
	PEPROCESS pProcessObj = NULL;
	NTSTATUS status = ObReferenceObjectByHandle(ProcessHandle, NULL, *PsProcessType, KernelMode, (PVOID)&pProcessObj, NULL);
	
	//函数名在 PEPROCESS 的 0x16c 处 ：  char ImageFileName[15];  
	char* ProcessName = (char* )((ULONG)pProcessObj + 0x16c);
	DbgPrint("ProcessName = %s\n", ProcessName);


PEPROCESS结构体用WinDbg显示：

	kd> dt _EPROCESS
	ntdll!_EPROCESS
	   +0x000 Pcb              : _KPROCESS
	   +0x098 ProcessLock      : _EX_PUSH_LOCK
	   +0x0a0 CreateTime       : _LARGE_INTEGER
	   +0x0a8 ExitTime         : _LARGE_INTEGER
	   +0x0b0 RundownProtect   : _EX_RUNDOWN_REF
	   +0x0b4 UniqueProcessId  : Ptr32 Void
	   +0x0b8 ActiveProcessLinks : _LIST_ENTRY
	   +0x0c0 ProcessQuotaUsage : [2] Uint4B
	   +0x0c8 ProcessQuotaPeak : [2] Uint4B
	   +0x0d0 CommitCharge     : Uint4B
	   +0x0d4 QuotaBlock       : Ptr32 _EPROCESS_QUOTA_BLOCK
	   +0x0d8 CpuQuotaBlock    : Ptr32 _PS_CPU_QUOTA_BLOCK
	   +0x0dc PeakVirtualSize  : Uint4B
	   +0x0e0 VirtualSize      : Uint4B
	   +0x0e4 SessionProcessLinks : _LIST_ENTRY
	   +0x0ec DebugPort        : Ptr32 Void
	   +0x0f0 ExceptionPortData : Ptr32 Void
	   +0x0f0 ExceptionPortValue : Uint4B
	   +0x0f0 ExceptionPortState : Pos 0, 3 Bits
	   +0x0f4 ObjectTable      : Ptr32 _HANDLE_TABLE
	   +0x0f8 Token            : _EX_FAST_REF
	   +0x0fc WorkingSetPage   : Uint4B
	   +0x100 AddressCreationLock : _EX_PUSH_LOCK
	   +0x104 RotateInProgress : Ptr32 _ETHREAD
	   +0x108 ForkInProgress   : Ptr32 _ETHREAD
	   +0x10c HardwareTrigger  : Uint4B
	   +0x110 PhysicalVadRoot  : Ptr32 _MM_AVL_TABLE
	   +0x114 CloneRoot        : Ptr32 Void
	   +0x118 NumberOfPrivatePages : Uint4B
	   +0x11c NumberOfLockedPages : Uint4B
	   +0x120 Win32Process     : Ptr32 Void
	   +0x124 Job              : Ptr32 _EJOB
	   +0x128 SectionObject    : Ptr32 Void
	   +0x12c SectionBaseAddress : Ptr32 Void
	   +0x130 Cookie           : Uint4B
	   +0x134 Spare8           : Uint4B
	   +0x138 WorkingSetWatch  : Ptr32 _PAGEFAULT_HISTORY
	   +0x13c Win32WindowStation : Ptr32 Void
	   +0x140 InheritedFromUniqueProcessId : Ptr32 Void
	   +0x144 LdtInformation   : Ptr32 Void
	   +0x148 VdmObjects       : Ptr32 Void
	   +0x14c ConsoleHostProcess : Uint4B
	   +0x150 DeviceMap        : Ptr32 Void
	   +0x154 EtwDataSource    : Ptr32 Void
	   +0x158 FreeTebHint      : Ptr32 Void
	   +0x160 PageDirectoryPte : _HARDWARE_PTE_X86
	   +0x160 Filler           : Uint8B
	   +0x168 Session          : Ptr32 Void
	   +0x16c ImageFileName    : [15] UChar
	   +0x17b PriorityClass    : UChar
	   +0x17c JobLinks         : _LIST_ENTRY
	   +0x184 LockedPagesList  : Ptr32 Void
	   +0x188 ThreadListHead   : _LIST_ENTRY
	   +0x190 SecurityPort     : Ptr32 Void
	   +0x194 PaeTop           : Ptr32 Void
	   +0x198 ActiveThreads    : Uint4B
	   +0x19c ImagePathHash    : Uint4B
	   +0x1a0 DefaultHardErrorProcessing : Uint4B
	   +0x1a4 LastThreadExitStatus : Int4B
	   +0x1a8 Peb              : Ptr32 _PEB
	   +0x1ac PrefetchTrace    : _EX_FAST_REF
	   +0x1b0 ReadOperationCount : _LARGE_INTEGER
	   +0x1b8 WriteOperationCount : _LARGE_INTEGER
	   +0x1c0 OtherOperationCount : _LARGE_INTEGER
	   +0x1c8 ReadTransferCount : _LARGE_INTEGER
	   +0x1d0 WriteTransferCount : _LARGE_INTEGER
	   +0x1d8 OtherTransferCount : _LARGE_INTEGER
	   +0x1e0 CommitChargeLimit : Uint4B
	   +0x1e4 CommitChargePeak : Uint4B
	   +0x1e8 AweInfo          : Ptr32 Void
	   +0x1ec SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
	   +0x1f0 Vm               : _MMSUPPORT
	   +0x25c MmProcessLinks   : _LIST_ENTRY
	   +0x264 HighestUserAddress : Ptr32 Void
	   +0x268 ModifiedPageCount : Uint4B
	   +0x26c Flags2           : Uint4B
	   +0x26c JobNotReallyActive : Pos 0, 1 Bit
	   +0x26c AccountingFolded : Pos 1, 1 Bit
	   +0x26c NewProcessReported : Pos 2, 1 Bit
	   +0x26c ExitProcessReported : Pos 3, 1 Bit
	   +0x26c ReportCommitChanges : Pos 4, 1 Bit
	   +0x26c LastReportMemory : Pos 5, 1 Bit
	   +0x26c ReportPhysicalPageChanges : Pos 6, 1 Bit
	   +0x26c HandleTableRundown : Pos 7, 1 Bit
	   +0x26c NeedsHandleRundown : Pos 8, 1 Bit
	   +0x26c RefTraceEnabled  : Pos 9, 1 Bit
	   +0x26c NumaAware        : Pos 10, 1 Bit
	   +0x26c ProtectedProcess : Pos 11, 1 Bit
	   +0x26c DefaultPagePriority : Pos 12, 3 Bits
	   +0x26c PrimaryTokenFrozen : Pos 15, 1 Bit
	   +0x26c ProcessVerifierTarget : Pos 16, 1 Bit
	   +0x26c StackRandomizationDisabled : Pos 17, 1 Bit
	   +0x26c AffinityPermanent : Pos 18, 1 Bit
	   +0x26c AffinityUpdateEnable : Pos 19, 1 Bit
	   +0x26c PropagateNode    : Pos 20, 1 Bit
	   +0x26c ExplicitAffinity : Pos 21, 1 Bit
	   +0x270 Flags            : Uint4B
	   +0x270 CreateReported   : Pos 0, 1 Bit
	   +0x270 NoDebugInherit   : Pos 1, 1 Bit
	   +0x270 ProcessExiting   : Pos 2, 1 Bit
	   +0x270 ProcessDelete    : Pos 3, 1 Bit
	   +0x270 Wow64SplitPages  : Pos 4, 1 Bit
	   +0x270 VmDeleted        : Pos 5, 1 Bit
	   +0x270 OutswapEnabled   : Pos 6, 1 Bit
	   +0x270 Outswapped       : Pos 7, 1 Bit
	   +0x270 ForkFailed       : Pos 8, 1 Bit
	   +0x270 Wow64VaSpace4Gb  : Pos 9, 1 Bit
	   +0x270 AddressSpaceInitialized : Pos 10, 2 Bits
	   +0x270 SetTimerResolution : Pos 12, 1 Bit
	   +0x270 BreakOnTermination : Pos 13, 1 Bit
	   +0x270 DeprioritizeViews : Pos 14, 1 Bit
	   +0x270 WriteWatch       : Pos 15, 1 Bit
	   +0x270 ProcessInSession : Pos 16, 1 Bit
	   +0x270 OverrideAddressSpace : Pos 17, 1 Bit
	   +0x270 HasAddressSpace  : Pos 18, 1 Bit
	   +0x270 LaunchPrefetched : Pos 19, 1 Bit
	   +0x270 InjectInpageErrors : Pos 20, 1 Bit
	   +0x270 VmTopDown        : Pos 21, 1 Bit
	   +0x270 ImageNotifyDone  : Pos 22, 1 Bit
	   +0x270 PdeUpdateNeeded  : Pos 23, 1 Bit
	   +0x270 VdmAllowed       : Pos 24, 1 Bit
	   +0x270 CrossSessionCreate : Pos 25, 1 Bit
	   +0x270 ProcessInserted  : Pos 26, 1 Bit
	   +0x270 DefaultIoPriority : Pos 27, 3 Bits
	   +0x270 ProcessSelfDelete : Pos 30, 1 Bit
	   +0x270 SetTimerResolutionLink : Pos 31, 1 Bit
	   +0x274 ExitStatus       : Int4B
	   +0x278 VadRoot          : _MM_AVL_TABLE
	   +0x298 AlpcContext      : _ALPC_PROCESS_CONTEXT
	   +0x2a8 TimerResolutionLink : _LIST_ENTRY
	   +0x2b0 RequestedTimerResolution : Uint4B
	   +0x2b4 ActiveThreadsHighWatermark : Uint4B
	   +0x2b8 SmallestTimerResolution : Uint4B
	   +0x2bc TimerResolutionStackRecord : Ptr32 _PO_DIAG_STACK_RECORD

## 蓝色妖姬 -- 相守
