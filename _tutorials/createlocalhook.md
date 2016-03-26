---
layout: page
title: Creating a local hook
---
<h2>Creating a local hook</h2>
A local hook is a hook that targets the same process that your code is running within.

To install a local hook we need to do four things:
{% include medrectad.html %}
 1. Retrieve the address of the native method to be hooked, for this we will use `LocalHook.GetProcAddress`
 2. Define a delegate type that matches the native method calling convention and parameters
 3. Write a hook handler method that we want to run in place of the original native method
 4. Lastly we need to create the hook using `LocalHook.Create`, passing in the original method address and the replacement delegate.

The complete sample is provided [below](#fullexample).
 
<h3>1. Retrieving the native method's address</h3>
Assuming that the method you wish to hook is exported by the target module we can use `LocalHook.GetProcAddress` to retrieve the method address, e.g. to retrieve the address of the MessageBeep method exported by user32.dll we would use the following snippet: 
{% highlight csharp %}
LocalHook.GetProcAddress("user32.dll", "MessageBeep");
{% endhighlight %}

<h3>2. Creating a delegate type that matches the native method</h3>
When creating the delegate type it is important that we match the same calling convention as the original and use the correct parameter types and marshalling. Here is the native WINAPI method declaration for MessageBeep:
{% highlight c++ %}
BOOL WINAPI MessageBeep(
  _In_ UINT uType
);
{% endhighlight %}
The corresponding delegate type would then look like: 
{% highlight csharp %}
[UnmanagedFunctionPointer(CallingConvention.StdCall, SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
delegate bool MessageBeepDelegate(uint uType);
{% endhighlight %}

<h3>3. Write the hook handler</h3>
The hook handler method must be compatible with the delegate that was created.

A hook handler for the MessageBeep method might look like this:

{% highlight csharp %}
[DllImport("user32.dll")]
static extern bool MessageBeep(uint uType);

static bool MessageBeepHook(uint uType)
{
    // Change the message beep to always be "Asterisk" (0x40)
    // see https://msdn.microsoft.com/en-us/library/windows/desktop/ms680356(v=vs.85).aspx
    return MessageBeep(0x40);
}
{% endhighlight %}

If you cannot use a `DllImport` and you still want to call the original method after retrieving the address in some other manner, you can use the original function address with `Marshal.GetDelegateForFunctionPointer`. To call the original method in this manner :

{% highlight csharp %}
static bool MessageBeepHook(uint uType)
{
    // Calling the original method with no changes
    return Marshal.GetDelegateForFunctionPointer<MessageBeepDelegate>(origAddr)(uType);
}
{% endhighlight %}

Note: EasyHook implements the hook trampoline code in such a way that calling the original method directly (or indirectly) while still within the hook handler will bypass the hook handler and call the original method.

<h3>4. Create and enable the local hook</h3>
We now have everything we need to create the hook. This involves two steps:
 
 1. Create the LocalHook instance using `LocalHook.Create`, and
 2. Activate the hook by telling it which threads to include/exclude from the hook

{% highlight csharp %}
// Create the local hook using our MessageBeepDelegate and MessageBeepHook handler
var hook = EasyHook.LocalHook.Create(
        EasyHook.LocalHook.GetProcAddress("user32.dll", "MessageBeep"),
        new MessageBeepDelegate(MessageBeepHook),
        null);
        
// Only hook this thread (threadId == 0 == GetCurrentThreadId)
hook.ThreadACL.SetInclusiveACL(new int[] { 0 });
{% endhighlight %}



<h2><a name="fullexample"></a>Full MessageBeep hook example</h2>
Create a new console application, install the EasyHook NuGet package and then replace the existing `Program.cs` with the following code.

This example hooks the MessageBeep method in order to prevent it from being called.

{% include responsivead.html %}

{% highlight csharp linenos %}
using System;
using System.Runtime.InteropServices;

namespace BeepHook
{
    class Program
    {
        // The matching delegate for MessageBeep
        [UnmanagedFunctionPointer(CallingConvention.StdCall, SetLastError = true)]
        delegate bool MessageBeepDelegate(uint uType);

        // Import the method so we can call it
        [DllImport("user32.dll")]
        static extern bool MessageBeep(uint uType);

        /// <summary>
        /// Our MessageBeep hook handler
        /// </summary>
        static private bool MessageBeepHook(uint uType)
        {
            // We aren't going to call the original at all
            // but we could using: return MessageBeep(uType);
            Console.Write("...intercepted...");
            return false;
        }
        
        /// <summary>
        /// Plays a beep using the native MessageBeep method
        /// </summary>
        static private void PlayMessageBeep()
        {
            Console.Write("    MessageBeep(BeepType.Asterisk) return value: ");
            Console.WriteLine(MessageBeep((uint)BeepType.Asterisk));
        }

        static void Main(string[] args)
        {
            Console.WriteLine("Calling MessageBeep with no hook.");
            PlayMessageBeep();

            Console.Write("\nPress <enter> to call MessageBeep while hooked by MessageBeepHook:");
            Console.ReadLine();

            Console.WriteLine("\nInstalling local hook for user32!MessageBeep");
            // Create the local hook using our MessageBeepDelegate and MessageBeepHook function
            using (var hook = EasyHook.LocalHook.Create(
                    EasyHook.LocalHook.GetProcAddress("user32.dll", "MessageBeep"),
                    new MessageBeepDelegate(MessageBeepHook),
                    null))
            {
                // Only hook this thread (threadId == 0 == GetCurrentThreadId)
                hook.ThreadACL.SetInclusiveACL(new int[] { 0 });

                PlayMessageBeep();

                Console.Write("\nPress <enter> to disable hook for current thread:");
                Console.ReadLine();
                Console.WriteLine("\nDisabling hook for current thread.");
                // Exclude this thread (threadId == 0 == GetCurrentThreadId)
                hook.ThreadACL.SetExclusiveACL(new int[] { 0 });
                PlayMessageBeep();

                Console.Write("\nPress <enter> to uninstall hook and exit.");
                Console.ReadLine();
            } // hook.Dispose() will uninstall the hook for us
        }
        
        public enum BeepType : uint
        {
            /// <summary>
            /// A simple windows beep
            /// </summary>            
            SimpleBeep = 0xFFFFFFFF,
            /// <summary>
            /// A standard windows OK beep
            /// </summary>
            OK = 0x00,
            /// <summary>
            /// A standard windows Question beep
            /// </summary>
            Question = 0x20,
            /// <summary>
            /// A standard windows Exclamation beep
            /// </summary>
            Exclamation = 0x30,
            /// <summary>
            /// A standard windows Asterisk beep
            /// </summary>
            Asterisk = 0x40,
        }
        
    }
}
{% endhighlight %}

Running this code results in the following output:

{% highlight console %}
Calling MessageBeep with no hook.
    MessageBeep(BeepType.Asterisk) return value: True

Press <enter> to call MessageBeep while hooked by MessageBeepHook:

Installing local hook for user32!MessageBeep
    MessageBeep(BeepType.Asterisk) return value: ...intercepted...False

Press <enter> to disable hook for current thread:

Disabling hook for current thread.
    MessageBeep(BeepType.Asterisk) return value: True

Press <enter> to uninstall hook and exit.
{% endhighlight %}
