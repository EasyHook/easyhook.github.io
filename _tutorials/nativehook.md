---
layout: page
title: Using EasyHook with C++
index: 01
managed: false
---
<h2>Using EasyHook with C++</h2>
In this tutorial we will use EasyHook to hook the Win API [Beep](https://msdn.microsoft.com/en-au/library/windows/desktop/ms679277(v=vs.85).aspx) function. Whenever a call to `Beep` is made we will increase the frequency originally passed in by 800hz.

Preparation:

 1. Create a new C++ console app
 2. Install the EasyHook Native Package from NuGet (you will see more than one package if you search for EasyHook, you are after only the "EasyHook **Native** Package"). Alternatively you can download the EasyHook binary package and manually reference the EasyHook library and "easyhook.h" (see [Manually adding EasyHook to your C++ project](nativemanuallyaddref.html)).
 3. Adding `#include <easyhook.h>` will allow us to call the EasyHook functions.

With our sample we will be demonstrating how to do the following:

 1. Retrieve the address of the original function.
 2. Gain an understanding of the original function's parameters and calling convention.
 3. Prepare a replacement function that has the same number and type of parameters, as well as the same calling convention as the original function.
 4. We will use LhInstallHook to create the hook by adding a trampoline to the start of the original method. We will also uninstall the hook once we are done with it.

The complete example can be found at the [end of this post](#fullexample).

{% include responsivead.html %}

<h3>Retrieving the original address</h3>
There are many ways to determine the correct address for a function, for this example we will assume that the function has been exported from another DLL and we will retrieve the address using `GetProcAddress`.

{% highlight c++ %}
#include <Windows.h>

// GetProcAddress can be used 
GetProcAddress(GetModuleHandle(TEXT("kernel32")), "Beep");
{% endhighlight %}

<h3>An understanding of the original parameters / calling convention</h3>
By taking a look at the declaration within the Windows API we can see that the function `Beep` looks like:
{% highlight c++ %}
BOOL WINAPI Beep(
    _In_ DWORD dwFreq,
    _In_ DWORD dwDuration
);
{% endhighlight %}

From this we can see that the function will return a `BOOL`, that the calling convention is set to `WINAPI` (which will be `__stdcall`), and the function receives two `DWORD` parameters as input.

Our replacement function is therefore going to look like:

{% highlight c++ %}
BOOL WINAPI myBeepHook(DWORD dwFreq, DWORD dwDuration);
{% endhighlight %}

<h3>Write the hook handler code</h3>
Our hook handler is going to simply intercept the call to `Beep` and then call the original function while adding 800 to the frequency supplied.

{% highlight c++ %}
BOOL WINAPI myBeepHook(DWORD dwFreq, DWORD dwDuration)
{
    // Call the original, adding 800 to the supplied frequency
    return Beep(dwFreq + 800, dwDuration);
}
{% endhighlight %}

<h3>Installing and uninstalling the hook handler</h3>
{% include medrectad.html %}
Now we can install our hook handler using `LhInstallHook`. After installing the hook we also tell enable the hook handler for the current thread.

We keep a reference to the hook using a `HOOK_TRACE_INFO` structure. This allows us to set-up the access control list (ACL) for which threads will be intercepted and which will continue unchanged - this can also be changed at a later time as needed. The ACLs can be configured as inclusive or exclusive with a call to `LhSetInclusiveACL` and `LhSetExclusiveACL` respectively.

Once we are finished with a hook then we can disable the hook with a call to `LhUninstallHook`. This does not remove the trampoline, however the trampoline code will see that the hook is no longer active and continue back to the original function without using any handler. To restore the function to its original state before the trampoline was installed we can use `LhWaitForPendingRemovals`.

{% highlight c++ %}
HOOK_TRACE_INFO hHook = { NULL }; // keep track of our hook
NTSTATUS result = LhInstallHook(
    GetProcAddress(GetModuleHandle(TEXT("kernel32")), "Beep"), 
    myBeepHook, 
    NULL, 
    &hHook);
if (FAILED(result))
{
    // Hook could not be installed, see RtlGetLastErrorString() for details
    return;
}
// If the threadId in the ACL is set to 0, 
// then internally EasyHook uses GetCurrentThreadId()
ULONG ACLEntries[1] = { 0 };

// Enable the hook for the provided threadIds
LhSetInclusiveACL(ACLEntries, 1, &hHook);

...

// Remove the hook handler
LhUninstallHook(&hHook);

// This will restore all functions that have any 
// uninstalled hooks back to their original state.
LhWaitForPendingRemovals();

{% endhighlight %}

<h3><a name="fullexample"></a>Full Example</h3>
Here is the full example:

{% highlight c++ %}
#include <string>
#include <iostream>
#include <Windows.h>
#include <easyhook.h>

using namespace std;

BOOL WINAPI myBeepHook(DWORD dwFreq, DWORD dwDuration);

BOOL WINAPI myBeepHook(DWORD dwFreq, DWORD dwDuration)
{
    cout << "\n****All your beeps belong to us!\n\n";
    return Beep(dwFreq + 800, dwDuration);
}

int _tmain(int argc, _TCHAR* argv[])
{
    HOOK_TRACE_INFO hHook = { NULL }; // keep track of our hook
    cout << "\n";
    cout << GetProcAddress(GetModuleHandle(TEXT("kernel32")), "Beep");
    
    // Install the hook
    NTSTATUS result = LhInstallHook(
        GetProcAddress(GetModuleHandle(TEXT("kernel32")), "Beep"), 
        myBeepHook, 
        NULL, 
        &hHook);
    if (FAILED(result))
    {
        wstring s(RtlGetLastErrorString());
        wcout << "Failed to install hook: ";
        wcout << s;
        cout << "\n\nPress any key to exit.";
        cin.get();
        return -1;
    }
    
    cout << "Beep after hook installed but not enabled.\n";
    Beep(500, 500);

    cout << "Activating hook for current thread only.\n";
    // If the threadId in the ACL is set to 0, 
    // then internally EasyHook uses GetCurrentThreadId()
    ULONG ACLEntries[1] = { 0 };
    LhSetInclusiveACL(ACLEntries, 1, &hHook);

    cout << "Beep after hook enabled.\n";
    Beep(500, 500);

    cout << "Uninstall hook\n";
    LhUninstallHook(&hHook);

    cout << "Beep after hook uninstalled\n";
    Beep(500, 500);

    cout << "\n\nRestore ALL entry points of pending removals issued by LhUninstallHook()\n";
    LhWaitForPendingRemovals();

    cout << "Press any key to exit.";
    cin.get();

    return 0;
}
{% endhighlight %}