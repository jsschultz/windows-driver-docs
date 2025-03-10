---
title: Context Monitoring
description: Context monitoring uses a monitored fence object, allowing for flexible synchronization between GPU engines, or across CPU cores and GPU engines.
ms.date: 12/18/2024
---

# Context monitoring

This article provides information about context monitoring, which was introduced in Windows 10 (WDDM 2.0).

Context monitoring allows for flexible synchronization between GPU engines, or across CPU cores and GPU engines. A *monitored fence object* is an advanced form of fence synchronization that allows either a CPU core or a GPU engine to signal or wait on a particular fence object.

## Monitored fence creation

The Direct3D runtime creates a monitored fence object by calling the user-mode driver's (UMD) [*pfnCreateSynchronizationObject2Cb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_createsynchronizationobject2cb) callback with a [**D3DDDICB_CREATESYNCHRONIZATIONOBJECT2**](/windows-hardware/drivers/ddi/d3dumddi/ns-d3dumddi-_d3dddicb_createsynchronizationobject2) structure. This structure's [**Info**](/windows-hardware/drivers/ddi/d3dukmdt/ns-d3dukmdt-_d3dddi_synchronizationobjectinfo2) member is a [**D3DDDI_SYNCHRONIZATIONOBJECTINFO2**](/windows-hardware/drivers/ddi/d3dukmdt/ns-d3dukmdt-_d3dddi_synchronizationobjectinfo2) structure that describes the synchronization object to create. The runtime sets **Info.Type** to [**D3DDDI_MONITORED_FENCE**](/windows-hardware/drivers/ddi/d3dukmdt/ne-d3dukmdt-_d3dddi_synchronizationobject_type) to indicate that the **Info.MonitoredFence** structure is to be used during creation.

The created monitored fence object has the following attributes:

* An initial fence value.
* Flags that specify its waiting and signaling behavior.

Upon creation, a monitored fence object is returned with the following information:

| Item | Description |
| ---- | ----------- |
| **hSyncObject** | Handle to the synchronization object. This handle is used in subsequent calls to *Dxgkrnl*. |
| **FenceValueCPUVirtualAddress** | Read-only mapping of the fence value (64 bits) for the CPU. This address is mapped WB (cacheable) from the point of view of the CPU on platforms supporting I/O coherency, UC (uncached) on other platforms. Allows the CPU to keep track of the fence progress by just reading this memory location. The CPU isn't allowed to write to this memory location. To signal the fence, the CPU is required to call the [*SignalSynchronizationObjectFromCpuCb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_signalsynchronizationobjectfromcpucb). Adapters that support IoMmu should use this address for GPU access. The address is mapped as read-write in this case. |
| **FenceValueGPUVirtualAddress** | Read/write mapping of the fence value (64 bits) for the GPU. This address is mapped as requiring I/O coherency on platforms supporting it. To signal the fence, the GPU is allowed to write directly to this GPU virtual address. IoMmu GPUs shouldn't use this address. |

The fence value is a 64-bit value with their respective virtual addresses aligned on a 64-bit boundary. GPUs should declare whether they're capable of atomically updating 64-bit values as visible by the CPU via the added [**DXGK_VIDSCHCAPS**](/windows-hardware/drivers/ddi/d3dkmddi/ns-d3dkmddi-_dxgk_vidschcaps)::**No64BitAtomics** flag. If a GPU is capable of only updating 32-bit values atomically, the OS handles the fence wraparound case automatically. However it places a restriction that outstanding wait and signal fence values can't be more than **UINT_MAX**/2 away from the last signaled fence value.

## GPU signal

If a GPU engine isn't capable of writing to a monitored fence using its virtual address, the UMD uses the [*SignalSynchronizationObjectFromGpuCb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_signalsynchronizationobjectfromgpucb) callback to queue a software signal packet to the GPU context.

To signal the fence from the GPU, the UMD inserts a fence write command in a context command stream directly without going through kernel mode. The mechanism by which the kernel monitors fence progress varies depending on whether a particular GPU engine supports the basic or advanced implementation of the monitored fence.

When a command buffer completes execution on the GPU, *Dxgkrnl*:

* Goes through the list of fence objects with pending waits that could be signaled for this process.
* Reads their current fence value.
* Determines whether there are any waits that need to be unwaited.

## GPU wait

To wait on a monitored fence on a GPU engine, the UMD first needs to flush its pending command buffer then call [*WaitForSynchronizationObjectFromGpuCb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_waitforsynchronizationobjectfromgpucb) specifying the fence object (**hSyncObject**) and the fence value being waited on. *Dxgkrnl* queues the dependency to its internal database, then returns immediately to the UMD so that it can continue to queue work behind the wait operation. Command buffers submitted after the wait operation aren't scheduled for execution until the wait operation is satisfied.

## CPU signal

The [*SignalSynchronizationObjectFromCpuCb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_signalsynchronizationobjectfromcpucb) callback was added to allow the CPU to signal a monitored fence object. When the CPU signals a monitored fence object, *Dxgkrnl* updates the fence memory location with the signaled value. This value becomes immediately visible to any user-mode reader and immediately unwaits any satisfied waits.

## CPU wait

A [*WaitForSynchronizationObjectFromCpuCb*](/windows-hardware/drivers/ddi/d3dumddi/nc-d3dumddi-pfnd3dddi_waitforsynchronizationobjectfromcpucb) callback was added to allow the CPU to wait on a monitored fence object. Two forms of wait operations are available:

* In the first form, *WaitForSynchronizationObjectFromCpuCb* blocks until the wait is satisfied.
* In the second form, *WaitForSynchronizationObjectFromCpuCb* takes a handle to a CPU event that is signaled once the waiting condition is satisfied.
