---
layout: page
title: Manually adding EasyHook to your C++ project
index: 03
managed: false
---
<h2>Manually adding EasyHook to your C++ project</h2>

Read on if you are using a version of Visual Studio that is not supported by the native NuGet package or for some other reason you need to manually reference the native EasyHook library.

Here we walk through one way of manually adding a reference to EasyHook in your C++ project:
{% include medrectad.html %}
 1. Download the latest EasyHook binary package from [downloads](../downloads.html).
 2. Extract the binaries (in this example it has been extracted to a new folder under the project)
 3. Within your project properties add one of the EasyHook paths to your Include and Library paths, e.g. `.\EasyHook-2.7.5870.0-Binaries\NetFX3.5` Note: it does not matter which EasyHook folder you reference, as the native DLLs are the same.
 4. Copy EasyHook32.dll and/or EasyHook64.dll from the EasyHook binaries to your build directory or add a build task to do it for you.
 5. You can now add the following to your code:
{% highlight c++ %}
#include <easyhook.h>

#if _WIN64
#pragma comment(lib, "EasyHook64.lib")
#else
#pragma comment(lib, "EasyHook32.lib")
#endif

...
{% endhighlight %}

Try using this approach with the "[Using EasyHook with C++](nativehook.html)" tutorial.