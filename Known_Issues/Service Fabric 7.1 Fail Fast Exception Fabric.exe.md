# Service Fabric 7.1 0xc0000602 Fail Fast Exception Fabric.exe

[Issue](#Issue)  
[Symptoms](#Symptoms)  
[Cause](#Cause)  
[Impact](#Impact)  
[Mitigation](#Mitigation)  
[Resolution](#Resolution)  

## Issue

Starting in version Service Fabric 7.1, you may experience an Assert exception in Fabric.exe.
This applies to Service Fabric Runtime 7.1 versions.

## Symptoms

Fabric.exe will terminates with exception Fail Fast exception 0xc0000602.  
Example Dump stack:

```text

Value: 7.1.458.9590
ADDITIONAL_XML: 1
OS_BUILD_LAYERS: 1
NTGLOBALFLAG:  0
PROCESS_BAM_CURRENT_THROTTLED: 0
PROCESS_BAM_PREVIOUS_THROTTLED: 0
APPLICATION_VERIFIER_FLAGS:  0
CONTEXT:  (.ecxr)
rax=0000007ff97fdaa0 rbx=0000007ff97fda00 rcx=0000007ff97fdaa0
rdx=0000000000000000 rsi=0000000000000001 rdi=0000007ff97fdaa0
rip=00007ffcb3916c74 rsp=0000007ff97fd9c0 rbp=0000007ff97fe090
 r8=0000000000000000  r9=0000000000000006 r10=0000000000000000
r11=00007ffcb71b6d57 r12=0000007ff97fe378 r13=0000000000000000
r14=0000025cc0a38e00 r15=0000025cc142e5c0
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000244
KERNELBASE!RaiseFailFastException+0x74:
00007ffc`b3916c74 803d9db3220000  cmp     byte ptr [KERNELBASE!BasepIsSecureProcess (00007ffc`b3b42018)],0 ds:00007ffc`b3b42018=00
Resetting default scope
EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff61caa6e5c (Fabric!Common::Assert::DoFailFast+0x00000000000001d8)
   ExceptionCode: c0000602
  ExceptionFlags: 00000001
NumberParameters: 0
PROCESS_NAME:  Fabric.exe
ERROR_CODE: (NTSTATUS) 0xc0000602 - {Fail Fast Exception}  A fail fast exception occurred. Exception handlers will not be invoked and the process will be terminated immediately.
EXCEPTION_CODE_STR:  c0000602
STACK_TEXT:  
0000007f`f97fd9c0 00007ff6`1caa6e5c     : 0000025c`c154e070 0000007f`f97fe0a8 0000007f`f97fe0a8 00000000`00000000 : KERNELBASE!RaiseFailFastException+0x74
0000007f`f97fdf90 00007ff6`1caa6c7b     : 0000007f`f97fe290 0000007f`f97fe290 00000000`ffffffff 0000007f`f97fe2b0 : Fabric!Common::Assert::DoFailFast+0x1d8
0000007f`f97fe1f0 00007ff6`1caa4a5b     : ffffffff`fffffffe 00007ff6`1ca89673 0000007f`f97fe6f0 00000000`00000000 : Fabric!Common::Assert::CodingError+0x5f
0000007f`f97fe270 00007ff6`1caa3976     : 00004b57`e3579d8d 00007ff6`1ca44a55 0000025c`c0a38e00 0000025c`c1236ce0 : Fabric!Common::ErrorCode::TryAssignFrom+0xb7
0000007f`f97fe2e0 00007ff6`1dd65577     : 00007ff6`1dd23100 0000007f`f97fe410 00000000`ffffffff 00007ff6`1dd23120 : Fabric!Common::ErrorCode::operator=+0x22
0000007f`f97fe310 00007ff6`1e5903cc     : 00000000`00000001 0000025c`c14bc3a0 ffffffff`fffffffe 0000025c`d446a7e8 : Fabric!Management::ClusterManager::DeleteApplicationContextAsyncOperation::EndLoadCurrentDnsName+0x74f
0000007f`f97fec40 00007ff6`1dab8539     : 00000000`00000000 00007ff6`1e590340 0000007f`f97ff090 00007ffc`b3b8cc5b : Fabric!Common::AsyncOperationWorkJobItem::EndWork+0x8c
0000007f`f97fecd0 00007ff6`1dcba9e7     : 0000025c`d446a8c0 0000007f`f97feeb0 0000025c`d446a8c0 00000000`00000000 : Fabric!Common::AsyncWorkJobItem::ProcessJob+0x69
0000007f`f97fedb0 00007ff6`1caa7ce7     : 00007ffc`b711f0a0 0000025c`d3218fc8 00007ff6`1dcc2c40 00000000`00000000 : Fabric!Common::AsyncWorkJobQueue<Common::AsyncOperationWorkJobItem>::Process+0x237
0000007f`f97ff360 00007ffc`b711efa1     : 0000025c`d3218f00 00000000`7ffe0386 0000025c`d3218f00 00007ffc`b711cc70 : Fabric!Common::Threadpool::WorkCallback<Common::Threadpool::WorkItem>+0x47
0000007f`f97ff3a0 00007ffc`b711d3fe     : 00000000`00000001 00000000`00000000 00007ffc`b711f0a0 00000000`00000000 : ntdll!TppWorkpExecuteCallback+0x131
0000007f`f97ff3f0 00007ffc`b4fc37e4     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!TppWorkerThread+0x43e
0000007f`f97ff780 00007ffc`b717cb81     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x14
0000007f`f97ff7b0 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x21
FAULTING_SOURCE_LINE:  assertwf.cpp
FAULTING_SOURCE_FILE:  assertwf.cpp
FAULTING_SOURCE_LINE_NUMBER:  193
FAULTING_SOURCE_CODE:  
No source found for 'assertwf.cpp'
SYMBOL_NAME:  Fabric!Common::Assert::DoFailFast+1d8
MODULE_NAME: Fabric
IMAGE_NAME:  Fabric.exe
STACK_COMMAND:  ~5s ; .ecxr ; kb
FAILURE_BUCKET_ID:  FAIL_FAST_EXCEPTION_c0000602_Fabric.exe!Common::Assert::DoFailFast
OS_VERSION:  10.0.16299.637
BUILDLAB_STR:  rs3_release_svc
OSPLATFORM_TYPE:  x64
OSNAME:  Windows 10
IMAGE_VERSION:  7.1.458.9590
FAILURE_ID_HASH:  {088f6433-45df-83fd-0817-22eab3c603c9}
Followup:     MachineOwner
---------
0:005> k
 # Child-SP          RetAddr               Call Site
00 0000007f`f97fd9c0 00007ff6`1caa6e5c     KERNELBASE!RaiseFailFastException+0x74
01 0000007f`f97fdf90 00007ff6`1caa6c7b     Fabric!Common::Assert::DoFailFast+0x1d8
02 0000007f`f97fe1f0 00007ff6`1caa4a5b     Fabric!Common::Assert::CodingError+0x5f
03 0000007f`f97fe270 00007ff6`1caa3976     Fabric!Common::ErrorCode::TryAssignFrom+0xb7
04 0000007f`f97fe2e0 00007ff6`1dd65577     Fabric!Common::ErrorCode::operator=+0x22
05 0000007f`f97fe310 00007ff6`1e5903cc     Fabric!Management::ClusterManager::DeleteApplicationContextAsyncOperation::EndLoadCurrentDnsName+0x74f [deleteapplicationcontextasyncoperation.cpp @ 233] 
06 (Inline Function) --------`--------     Fabric!std::_Func_class<void,std::shared_ptr<Common::AsyncOperation> const &>::operator()+0x21 [asyncoperationworkjobitem.cpp @ 100] 
07 0000007f`f97fec40 00007ff6`1dab8539     Fabric!Common::AsyncOperationWorkJobItem::EndWork+0x8c [asyncoperationworkjobitem.cpp @ 100] 
08 0000007f`f97fecd0 00007ff6`1dcba9e7     Fabric!Common::AsyncWorkJobItem::ProcessJob+0x69 [asyncworkjobitem.cpp @ 141] 
09 (Inline Function) --------`--------     Fabric!Common::AsyncWorkJobQueue<Common::AsyncOperationWorkJobItem>::JobTraits<Common::AsyncOperationWorkJobItem>::ProcessJob+0x13 [asyncworkjobqueue.h @ 505] 
0a 0000007f`f97fedb0 00007ff6`1caa7ce7     Fabric!Common::AsyncWorkJobQueue<Common::AsyncOperationWorkJobItem>::Process+0x237 [asyncworkjobqueue.h @ 219] 
0b (Inline Function) --------`--------     Fabric!std::_Func_class<void>::operator()+0x2e [threadpool.cpp @ 43] 
0c (Inline Function) --------`--------     Fabric!Common::Threadpool::WorkItem::Callback+0x2e [threadpool.cpp @ 42] 
0d 0000007f`f97ff360 00007ffc`b711efa1     Fabric!Common::Threadpool::WorkCallback<Common::Threadpool::WorkItem>+0x47 [threadpool.h @ 79] 
0e 0000007f`f97ff3a0 00007ffc`b711d3fe     ntdll!TppWorkpExecuteCallback+0x131 [minkernel\threadpool\ntdll\work.c @ 671] 
0f 0000007f`f97ff3f0 00007ffc`b4fc37e4     ntdll!TppWorkerThread+0x43e [minkernel\threadpool\ntdll\worker.c @ 1109] 
10 0000007f`f97ff780 00007ffc`b717cb81     kernel32!BaseThreadInitThunk+0x14 [base\win32\client\thread.c @ 64] 
11 0000007f`f97ff7b0 00000000`00000000     ntdll!RtlUserThreadStart+0x21 [minkernel\ntdll\rtlstrt.c @ 997] 
```

## Cause

This is a regression that was introduced in Service Fabric 7.1.  

## Impact

This issue may cause the cluster to become unmanageable.

## Mitigation

TBD

## Resolution

Upgrade the Service Fabric version of the Cluster to a version greater than or equal to 7.2 CU4 as listed here [Service Fabric 7.2 CU4 Release Nodes](https://github.com/microsoft/service-fabric/blob/master/release_notes/Service-Fabric-72CU4-releasenotes.md).
