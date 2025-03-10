---
title: Static and Dynamic Verification Tools
description: Static and Dynamic Verification Tools
keywords:
- dynamic verification tools WDK
- static verification tools WDK
ms.date: 07/02/2024
---

# Static and Dynamic Verification Tools

There are two basic types of verification tools:

- **Static verification tools** examine the driver code without running the driver. Because these tools do not rely on tests that exercise the code, they can be extremely thorough. Theoretically, static verification tools can examine all of the driver code, including code paths that are rarely executed in practice. However, because the driver is not actually running, they could generate false-positive results. That is, they might report an error in a code path that might not occur in practice.

- **Dynamic verification tools** examine the driver code while the driver is running, typically by intercepting calls to commonly used driver support routines and substituting calls to their own error-checking versions of the same routines. Because the driver is actually running while the dynamic tools are doing the verification, false-positive results are rare. However, because the dynamic tools detect only the actions that occur while they are monitoring the driver, the tools can miss certain driver defects if the driver test coverage is not adequate. At the same time, by using information available at run time, for example, information that is harder to extract statically from the source code, dynamic verification tools can detect certain classes of driver errors that are harder to detect with static analysis tools.

The best practice is to use a combination of static and dynamic verification tools. Static tools allow you to check code paths that are difficult to exercise in practice, while the dynamic tools find serious errors that are occurring in the driver.

> [!IMPORTANT]
> Windows Hardware Compatibility Program requires CodeQL for Static Tool Logo (STL) Tests on our Client and Server Operating Systems. We will continue to maintain support for SDV and CA on older products.  Partners are highly encouraged to review the CodeQL requirements for the [Static Tool Logo Test](/windows-hardware/test/hlk/testref/6ab6df93-423c-4af6-ad48-8ea1049155ae).
> For more information about using CodeQL, see [CodeQL and the Static Tools Logo Test](static-tools-and-codeql.md).

## Survey of Verification Tools

The following verification tools are described in the WDK and recommended for use by driver developers and testers. They are listed in the order in which they are typically used.

## As soon as the code compiles

- CodeQL, by GitHub, is a powerful semantic code analysis engine, and the combination of an extensive suite of high-value security queries along with a robust platform make it an invaluable tool for securing driver code. For more information, see [CodeQL and the Static Tools Logo Test](static-tools-and-codeql.md).

## Additional static tools

Depending on what version of Windows you are building a driver from, other static tools may be required.

- [Code Analysis for Drivers](code-analysis-for-drivers.md) is a static verification tool that runs at compile time. Code Analysis for Drivers can verify drivers written in C/C++ and managed code. It examines the code in each function of a driver independently, so you can run it as soon as you can build your driver. It runs relatively quickly and uses few resources.

  The basic features of the Code Analysis tool in Visual Studio detect general coding errors, such as not checking return values. The driver-specific features detect more subtle driver coding errors, such as leaving uninitialized fields in a copied IRP and failing to restore a changed IRQL by the end of a routine.

- [Static Driver Verifier](static-driver-verifier.md) (SDV) is a static verification tool that runs at compile time and verifies kernel-mode driver code written in C/C++. It is included in the WDK and can be started from Visual Studio Ultimate 2012 or from a Visual Studio Command prompt window using MSBuild.

  Based on a set of interface rules and a model of the operating system, Static Driver Verifier determines whether the driver properly interacts with the Windows operating system kernel. Static Driver Verifier is extremely thorough -- it explores all reachable paths in the driver source code and executes them symbolically. As such, it finds bugs that are not detected by using any other conventional method of driver testing.

> [!IMPORTANT]
> SDV is no longer supported and SDV is not available in Windows 24H2 WDK or EWDK releases. It is not available in WDKs newer than build 26017, and is not included in the Windows 24H2 RTM WDK.
> SDV can still be used by downloading the Windows 11, version 22H2 EWDK (released October 24, 2023) with Visual Studio build tools 17.1.5 from [Download the Windows Driver Kit (WDK)](../download-the-wdk.md). Only the use of the Enterprise WDK to run SDV is recommended. Using older versions of the standard WDK in conjunction with recent releases of Visual Studio is not recommended, as this will likely result in analysis failures. <br>
> Going forward, CodeQL will be the primary static analysis tool for drivers. CodeQL provides a powerful query language that treats code as a database to be queried, making it simple to write queries for specific behaviors, patterns, and more.
> For more information about using CodeQL, see [CodeQL and the Static Tools Logo Test](static-tools-and-codeql.md).

## When the driver runs

Use the following dynamic verification tools as soon as the driver is built and is running without obvious errors.

- [Driver Verifier](driver-verifier.md) is a dynamic verification tool written especially for Windows drivers. It includes multiple tests that can be run on several drivers simultaneously. Driver Verifier is so effective at finding serious bugs in drivers that experienced driver developers and testers configure Driver Verifier to run whenever their driver runs in a development or test environment. Driver Verifier is included in Windows. When you enable Driver Verifier for a driver, you must also run multiple tests on the driver. Driver Verifier can detect certain driver bugs that are difficult to detect by using static verification tools alone. Examples of these types of bugs include the following:

  - **Kernel pool buffer overruns.** When the verified driver allocates pool memory buffers, Driver Verifier guards them with a non-accessible memory page. If the driver tries to use memory past the end of the buffer, the Driver Verifier will issue a bug check.

  - **Using memory after freeing it.** Special pool memory blocks use their own memory page, and do not share memory pages with other allocations. When the driver is freeing the block of pool memory, the corresponding memory page becomes non-accessible. If the driver attempts to use that memory after freeing it, the driver will crash instantly.

  - **Using pageable memory while running at elevated IRQL.** When a verified driver raises the IRQL at DISPATCH\_LEVEL or higher, Driver Verifier trims all pageable memory from the system working set, simulating a system under memory pressure. The driver crashes if it tries to use one of these pageable virtual addresses.

  - **Low Resources Simulation.** To simulate a system under low resources conditions, Driver Verifier can fail various operating system kernel APIs called by drivers.

  - **Memory leaks.** Driver Verifier tracks memory allocations made by a driver and makes sure the memory is freed before the driver gets unloaded.

  - **I/O operations that take too much time to complete or to be canceled.** The Driver Verifier can test the driver's logic for responding to STATUS\_PENDING return values from [**IoCallDriver**](/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver).

  - **DDI Compliance Checking.** (Available starting with Windows 8) Driver Verifier applies a set of device driver interface (DDI) rules that check for the proper interaction between a driver and the kernel interface of the operating system. These rules correspond to rules that Static Driver Verifier uses in analyzing driver source code. If Driver Verifier finds an error when DDI Compliance Checking is enabled, run [Static Driver Verifier](static-driver-verifier.md) and select the same rule that caused the error. Static Driver Verifier can help you locate the cause of the defect in the driver source code.

- The [Kernel Address Sanitizer](kasan.md) (KASAN) is a bug detection technology supported on Windows drivers that enables you to detect several classes of illegal memory accesses, such as buffer overflows and use-after-free events.

- [Application Verifier](application-verifier.md) is a dynamic verification tool for user-mode applications and drivers written in C/C++. It does not verify managed code.
