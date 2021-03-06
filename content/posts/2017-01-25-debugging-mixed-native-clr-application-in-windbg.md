---
title: "Debugging mixed native-CLR application in WinDBG"
created_at: 2017-01-25 10:26:13 +0000
kind: article
published: true
---

While working on [CLR interop for Bee Smalltalk][4], things go wrong every now and
again. This is tricky to debug, at least for me, as there is Smalltalk code,
then some native C++ code (CLR's CCW and all this stuff) and finally managed
CLR code. 

Obviously, Smalltalk debugger is kind of useless here as one would not see
native nor managed stack there. Managed debugger is can be useful to see what's
happening in managed code, but quite often it got confused because we're calling
CLR from native application so more often than not all it tells you is that
something went wrong - in case you don't know that yet. 

<!-- more -->

I resorted to use WinDBG. One cannot see a Bee Smalltalk stack, but one can see
a native stack and CLR stack. One can stop on CLR exception being thrown and
examine stack at throw-point. One does not have source code, so actually all
one can get are function names and eventually inspect CLR objects and their
contents. Not much, but better than nothing. 

I used (and still using) GDB a lot while working on Smalltalk/X VM and other
low-level stuff so I know it by
heart. But I'm still fairly new to this WinDBG stuff and keep forgetting stuff.
This post is merely a note to myself (and perhaps to anybody would have to 
mess up with CLR in otherwise native application). 

## The Problem 

The other day I wanted to debug some rough edges on exception support in Bee's CLR interop
package. One problem is that it somehow missed the very first exception, i.e.,
the first exception is lost and not propagated to Bee Smalltalk do #on:do: block
fails to catch it. 

So, let's debug it. Start a Bee Smalltalk, load in CLR runtime: 


    runtimeInfo := ICLRMetaHostClient newInstance defaultRuntime.
    runtimeInfo isLoaded ifFalse:[
        clrRuntimeHost := runtimeInfo getCLRHost.
        clrRuntimeHost Start
    ].
    corRuntimeHost := runtimeInfo getCorHost.
    appDomain := corRuntimeHost DefaultDomain.

Now attach a WinDBG to Bee Smalltalk process.

## Setting up Symbol Server

First thing one should do is to set up a symbol server so one actually see function
names - at least. To do so, pick "File" -> "Symbol File Path" and enter (or add):

    srv*https://msdl.microsoft.com/download/symbols

then tick "Reload" checkbox at the bottom and confirm. 

## Install SOSEX - a Handy WinDBG Extension to Debug CLR

[SOSEX extension][1] is essential - this is the key to make sense of managed objects,
dump managed stack and so on. A must-have, really. [Download it][2] and copy
both `sosex.dll` and `sosex.pdb` to WinDBG directory. In my case it's:

    C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x86

Then load it into running WinDBG - in command prompt issue:

    .load sosex

You may print list of commands provided by SOSEX:

    !sosex.help

## Setting Breakpoints in Managed Code

One of the problems we'd like to chase down is that the very first exception
is somehow lost. So let's see whether our exception tracking method is called
at all. So let's set a breakpoint in `ExceptionHandler.FirstChanceHandler()` method:

    !mbp ExceptionHandler.cs 37

Once a breakpoint is set, let Bee run again (use `g` command or F5 shortcut)

## Examine CLR Exception when Thrown

Now execute a very first test that causes an exception to be thrown (but not
caught) in managed code: 

    ClrObjectTest debug: #test_exception_01

As soon as you do-it the above code, Bee Smalltalk freezes and in WinDBG you may
see something like: 

    (9a4.d20): CLR exception - code e0434352 (first chance)
    First chance exceptions are reported before any exception handling.
    This exception may be expected and handled.
    *** ERROR: Symbol file could not be found.  Defaulted to export symbols for C:\Windows\syswow64\KERNELBASE.dll - 
    eax=003eca54 ebx=00000005 ecx=00000005 edx=00000000 esi=003ecb0c edi=00000001
    eip=7656c52f esp=003eca54 ebp=003ecaa4 iopl=0         nv up ei pl nz ac po nc
    cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00200212
    KERNELBASE!RaiseException+0x58:
    7656c52f c9              leave
    *** WARNING: Unable to verify checksum for C:\Windows\assembly\NativeImages_v4.0.30319_32\mscorlib\225759bb87c854c0fff27b1d84858c21\mscorlib.ni.dll

This means a CLR exception (a opposed to, say, a C++ exception) has been thrown.
At this point, no catch blocks were executed (not even searched for). To see what
exception has actually be thrown, use `!PrintException` command as: 

    0:000> !PrintException
    Exception object: 07b9ece4
    Exception type:   Bee.CLRInterop.Tests.Mocks.CLRExceptionA
    Message:          <none>
    InnerException:   <none>
    StackTrace (generated):
    <none>
    StackTraceString: <none>
    HResult: 80131500

Good, so this is our exception that should be thrown by managed counterpart
of the test. This exception is expected. We may want to check the stack, though
using `!mk` command. This SOSEX command has the advantage over `!CLRStack` that 
it shows both managed and unmanaged frames. Very handy: 

    0:000> !mk 10
    Thread 0:
            SP       IP
    00:U 003eca54 7656c52f KERNELBASE!RaiseException+0x58
    01:U 003ecaac 6f2d06f2 clr!RaiseTheExceptionInternalOnly+0x27c
    02:U 003ecb48 6f2d1144 clr!IL_Throw+0x138
    03:M 003ecc10 04d508fa *** WARNING: Unable to verify checksum for Bee.CLRInterop.Tests.Mocks.dll
    Bee.CLRInterop.Tests.Mocks.CLRObjectTestsMock.ThrowCLRExceptionA()(+0x0 IL,+0x3a Native) [h:\Projects\Bee\sources3\lib\CLRHosting-Tests\Bee.CLRInterop.Tests.Mocks\CLRObjectTestsMock.cs @ 91,3]
    04:U 003ecc20 6f231396 clr!CallDescrWorkerInternal+0x34
    05:U 003ecc2c 6f23291f clr!CallDescrWorkerWithHandler+0x6b
    06:U 003ecc80 6f2405bd clr!CallDescrWorkerReflectionWrapper+0x55
    07:U 003eccc0 6f24069e clr!RuntimeMethodHandle::InvokeMethod+0x7eb
    08:M 003ecfb4 6e481281 System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(System.Object, System.Object[], System.Object[])(+0x0 IL,+0xc1 Native)
    09:M 003ecfd8 6e480da6 System.Reflection.RuntimeMethodInfo.Invoke(System.Object, System.Reflection.BindingFlags, System.Reflection.Binder, System.Object[], System.Globalization.CultureInfo)(+0x66 Native)
    0a:M 003ed004 6e48e5a6 System.Reflection.MethodBase.Invoke(System.Object, System.Object[])(+0x0 IL,+0x16 Native)
    0b:U 003ed01c 6f231396 clr!CallDescrWorkerInternal+0x34
    0c:U 003ed02c 6f23291f clr!CallDescrWorkerWithHandler+0x6b
    0d:U 003ed080 6f2405bd clr!CallDescrWorkerReflectionWrapper+0x55
    0e:U 003ed0c0 6f24069e clr!RuntimeMethodHandle::InvokeMethod+0x7eb
    0f:M 003ed3b0 6e48121d System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(System.Object, System.Object[], System.Object[])(+0x16 IL,+0x5d Native)


Anyway, we do expect `Bee.CLRInterop.Tests.Mocks.CLRExceptionA` being thrown,
that's what our test does, so let's proceed (use `g` command or F5 shortcut).
Now, another CLR exception is thrown: 

    (9a4.d20): CLR exception - code e0434352 (first chance)
    First chance exceptions are reported before any exception handling.
    This exception may be expected and handled.
    eax=003dcd40 ebx=00000005 ecx=00000005 edx=00000000 esi=003dcdf8 edi=00000001
    eip=7718c52f esp=003dcd40 ebp=003dcd90 iopl=0         nv up ei pl nz ac po nc
    cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000212
    KERNELBASE!RaiseException+0x58:
    7718c52f c9              leave

Again, let's see what's the exception (using familiar `!PrintException` command):

    Exception object: 047689f0
    Exception type:   System.Reflection.TargetInvocationException
    Message:          Exception has been thrown by the target of an invocation.
    InnerException:   Bee.CLRInterop.Tests.Mocks.CLRExceptionA, Use !PrintException 047672cc to see more.
    StackTrace (generated):
    <none>
    StackTraceString: <none>
    HResult: 80131604

OK, so this is the `TargetInvocationException` which wraps any exception thrown 
when a method is executed via reflection. This is also to be expected. 
But wait! Shouldn't we get into our exception handler before that? I mean, before
this, `ExceptionHandler.FirstChanceHandler()` should have been called. But the
breakpoint did not hit. Fishy, but that's what we suspected, didn't we? 

OK, let's continue (`g` or F5), let the test finally fail and run it again. As in
previous case, we get first an expected `Bee.CLRInterop.Tests.Mocks.CLRExceptionA`
but the, breakpoint hits: 

    Breakpoint: JIT notification received for method Bee.CLRInterop.Support.ExceptionHandler.FirstChanceHandler(System.Object, System.Runtime.ExceptionServices.FirstChanceExceptionEventArgs) in AppDomain 004c1ca0.
    Breakpoint set at Bee.CLRInterop.Support.ExceptionHandler.FirstChanceHandler(System.Object, System.Runtime.ExceptionServices.FirstChanceExceptionEventArgs) in AppDomain 004c1ca0.
    Breakpoint 0 hit
    eax=00000000 ebx=003dbd10 ecx=047112f8 edx=00000000 esi=047112f8 edi=003dbc88
    eip=04440a41 esp=003dbc60 ebp=003dbc98 iopl=0         nv up ei pl zr na pe nc
    cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
    04440a41 8b4de8          mov     ecx,dword ptr [ebp-18h] ss:002b:003dbc80=04767360

Let's check the stack again (`!mk`):

    0:000> !mk 20
    Thread 0:
            SP       IP
    00:M 003dbd10 04440a41 Bee.CLRInterop.Support.ExceptionHandler.FirstChanceHandler(System.Object, System.Runtime.ExceptionServices.FirstChanceExceptionEventArgs)(+0x1 IL,+0x39 Native) [h:\Projects\Bee\sources3\lib\CLRHosting-Tests\Bee.CLRInterop.Support\ExceptionHandler.cs @ 37,4]
    01:U 003dbd50 71421396 clr!CallDescrWorkerInternal+0x34
    02:U 003dbd5c 7142291f clr!CallDescrWorkerWithHandler+0x6b
    03:U 003dbdb0 7143dfe1 clr!DispatchCallSimple+0x7d
    04:U 003dbdf0 716d4bc3 clr!ExceptionNotifications::DeliverExceptionNotification+0x39
    05:U 003dbe10 716d5138 clr!ExceptionNotifications::DeliverNotificationInternal+0x113
    06:U 003dbe74 716d4fa3 clr!ExceptionNotifications::DeliverNotification+0x29
    07:U 003dbec4 716d5199 clr!ExceptionNotifications::DeliverFirstChanceNotification+0x7d
    08:U 003dbf18 714c0a65 clr!COMPlusThrowCallback+0x31b
    09:U 003dbf88 714315ae clr!Thread::StackWalkFramesEx+0x89
    0a:U 003dc254 714316b0 clr!Thread::StackWalkFrames+0x9d
    0b:U 003dc588 714c000f clr!CPFH_RealFirstPassHandler+0x65f
    0c:U 003dc738 714c024a clr!CPFH_FirstPassHandler+0x119
    0d:U 003dc780 714c030b clr!COMPlusFrameHandler+0x15d
    0e:U 003dc7a8 77e3b81d ntdll!ExecuteHandler2+0x26
    0f:U 003dc7cc 77e3b7ef ntdll!ExecuteHandler+0x24
    10:U 003dc7f0 77e3b790 ntdll!RtlDispatchException+0x127
    11:U 003dc87c 77df0163 ntdll!KiUserExceptionDispatcher+0xf
    12:U 003dcd40 7718c52f KERNELBASE!RaiseException+0x58
    13:U 003dcd98 714c06f2 clr!RaiseTheExceptionInternalOnly+0x27c
    14:U 003dce34 715b9fc3 clr!RaiseTheException+0x86
    15:U 003dce4c 715ba02a clr!RealCOMPlusThrowWorker+0x72
    16:U 003dce74 715ba05a clr!RealCOMPlusThrow+0x2f
    17:U 003dcea8 7183078a clr!ThrowInvokeMethodException+0xac
    18:U 003dcf00 71830c73 clr!RuntimeMethodHandle::InvokeMethod+0x9f8
    19:M 003dd1f4 69c81281 System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(System.Object, System.Object[], System.Object[])(+0x0 IL,+0xc1 Native)
    1a:M 003dd218 69c80da6 System.Reflection.RuntimeMethodInfo.Invoke(System.Object, System.Reflection.BindingFlags, System.Reflection.Binder, System.Object[], System.Globalization.CultureInfo)(+0x66 Native)
    1b:M 003dd244 69c8e5a6 System.Reflection.MethodBase.Invoke(System.Object, System.Object[])(+0x0 IL,+0x16 Native)
    1c:U 003dd25c 71421396 clr!CallDescrWorkerInternal+0x34
    1d:U 003dd26c 7142291f clr!CallDescrWorkerWithHandler+0x6b
    1e:U 003dd2c0 714305bd clr!CallDescrWorkerReflectionWrapper+0x55
    1f:U 003dd300 7143069e clr!RuntimeMethodHandle::InvokeMethod+0x7eb

Looks good. Now if we continue, test passes. So the problem is, as one may suspect, that 
our exception handler is not called for the very first time. Looks weird - maybe 
something's wrong with our handler. The code reads: 

    ...
    private static void FirstChanceHandler(object source, FirstChanceExceptionEventArgs args)
    {
        var ex = args.Exception as TargetInvocationException;
        if (ex != null) 
        {
            if (Debug) {
                Console.WriteLine("Caught TargetInvocationException in thread {0}, appdomain {1}", Thread.CurrentThread.ToString(), AppDomain.CurrentDomain.FriendlyName);
            }
            LastException = ex;
            LastThread = Thread.CurrentThread;
        }
    }
    
    static ExceptionHandler() 
    {
        AppDomain.CurrentDomain.FirstChanceException += FirstChanceHandler;
        Debug = false;
    }
    ...

Thinking hard...harder...of course! We register our handler in a static 
constructor, but did it run? Likely not. So, let's double check when CLR runs
static initializer of a class. That's what spec says:

> The static constructor for a class executes at most once in a given application 
> domain. The execution of a static constructor is triggered by the first of the 
> following events to occur within an application domain:
> 
>   * An instance of the class is created.
>   * Any of the static members of the class are referenced.

This explains everything, doesn't it? :-)

## Further reading

  * [MANAGED DEBUGGING with WINDBG. Introduction and Index][3]


[1]: http://www.stevestechspot.com/
[2]: http://www.stevestechspot.com/downloads/sosex_32.zip
[3]: https://blogs.msdn.microsoft.com/alejacma/2009/07/07/managed-debugging-with-windbg-introduction-and-index/
[4]: /2016/10/a-taste-of-dotNET-in-Bee-Smalltalk.html