---
author: normesta
Description: Fix issues that prevent your desktop application from running in an MSIX container
Search.Product: eADQiWindows 10XVcnh
title: Fix issues that prevent your desktop application from running in an MSIX container
ms.author: normesta
ms.date: 07/02/2018
ms.topic: article
ms.prod: windows
ms.technology: uwp
keywords: windows 10, uwp
ms.localizationpriority: medium
---

# Apply runtime fixes to an MSIX package by using the Package Support Framework

The Package Support Framework is an open source kit that helps you apply fixes to your existing win32 application when you don't have access to the source code, so that it can run in an MSIX container. The Package Support Framework helps your application follow the best practices of the modern runtime environment.

To create the Package Support Framework, we leveraged the [Detours](https://www.microsoft.com/en-us/research/project/detours) technology which is an open source framework developed by Microsoft Research (MSR) and helps with API redirection and hooking.

This framework is open source, lightweight, and you can use it to address application issues quickly. It also gives you the opportunity to consult with the community around the globe, and to build on top of the investments of others.

## A quick look inside of the Package Support Framework

The Package Support Framework contains an executable, a runtime manager  DLL, and a set of runtime fixes.

![Package Support Framework](images/desktop-to-uwp/package-support-framework.png)

Here's how it works. You'll create a configuration file that specifies the fix(s) that you want to apply to your application. Then, you'll modify your package to point to the shim launcher executable file.

When users start your application, the shim launcher is the first executable that runs. It reads your configuration file, and injects the runtime fix(s) and the runtime manager  DLL into the application process.

![Package Support Framework  DLL Injection](images/desktop-to-uwp/package-support-framework-2.png)

The runtime manager applies the fix when it's needed by the application to run inside of an MSIX container.

This guide will help you to identify application compatibility issues, and to find, apply, and extend runtime fixes that address them.

<a id="identify" />

## Identify packaged application compatibility issues

First, create a package for your application. Then, install it, run it, and observe its behavior. You might receive error messages that can help you identify a compatibility issue. You can also use [Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) to identify issues.  Common issues relate to application assumptions regarding the working directory and program path permissions.

### Using Process Monitor to identify an issue

[Process Monitor](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon) is a powerful utility for observing an app's file and registry operations, and their results.  This can help you to understand application compatibility issues.  After opening Process Monitor, add a filter (Filter > Filter…) to include only events from the application executable.

![ProcMon App Filter](images/desktop-to-uwp/procmon_app_filter.png)

A list of events will appear. For many of these events, the word **SUCCESS** will appear in the **Result** column.

![ProcMon Events](images/desktop-to-uwp/procmon_events.png)

Optionally, you can filter events to only show only failures.

![ProcMon Exclude Success](images/desktop-to-uwp/procmon_exclude_success.png)

If you suspect a filesystem access failure, search for failed events that are under either the System32/SysWOW64 or the package file path. Filters can also help here, too. Start at the bottom of this list and scroll upwards. Failures that appear at the bottom of this list have occurred most recently. Pay most attention to errors that contain strings such as "access denied," and "path/name not found", and ignore things that don't look suspicious. The [PSFSample](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/samples/PSFSample/) has two issues. You can see those issues in the list that appears in the following image.

![ProcMon Config.txt](images/desktop-to-uwp/procmon_config_txt.png)

In the first issue that appears in this image, the application is failing to read from the "Config.txt" file that is located in the "C:\Windows\SysWOW64" path. It's unlikely that the application is trying to reference that path directly. Most likely, it's trying to read from that file by using a relative path, and by default, "System32/SysWOW64" is the application's working directory. This suggests that the application is expecting its current working directory to be set to somewhere in the package. Looking inside of the appx, we can see that the file exists in the same directory as the executable.

![App Config.txt](images/desktop-to-uwp/psfsampleapp_config_txt.png)

The second issue appears in the following image.

![ProcMon Logfile](images/desktop-to-uwp/procmon_logfile.png)

In this issue, the application is failing to write a .log file to its package path. This would suggest that a file redirection shim might help.

<a id="find" />

## Find a runtime fix

The PSF contains runtime fixes that you can use right now, such as the file redirection shim.

### File Redirection Shim

You can use the [File Redirection Shim](https://github.com/Microsoft/MSIX-PackageSupportFramework/tree/develop/FileRedirectionShim) to redirect attempts to write or read data in a directory that isn't accessible from an application that runs in an MSIX container.

For example, if your application writes to a log file that is in the same directory as your applications executable, then you can use the [File Redirection Shim](https://github.com/Microsoft/MSIX-PackageSupportFramework/tree/develop/FileRedirectionShim) to create that log file in another location, such as the local app data store.

### Runtime fixes from the community

Make sure to review the community contributions to our [GitHub](https://github.com/Microsoft/MSIX-PackageSupportFramework/tree/develop) page. It's possible that other developers have resolved an issue similar to yours and have shared a runtime fix.

## Apply a runtime fix

You can apply an existing runtime fix with a few simple tools from the Windows SDK, and by following these steps.

> [!div class="checklist"]
> * Create a package layout folder
> * Get the Package Support Framework files
> * Add them to your package
> * Modify the package manifest
> * Create a configuration file

Let's go through each task.

### Create the package layout folder

If you have a .appx file already, you can unpack its contents into a layout folder that will serve as the staging area for your package.  You can do this from an **x64 Native Tools Command Prompt for VS 2017**, or manually with the SDK bin path in the executable search path.

```
makeappx unpack /p PSFSamplePackage_1.0.60.0_AnyCPU_Debug.appx /d PackageContents

```

This will give you something that looks like the following.

![Package Layout](images/desktop-to-uwp/package_contents.png)

If you don't have a .appx file to start with, you can create the package folder and files from scratch.

### Get the Package Support Framework files

You can get the PSF Nuget package by using Visual Studio. You can also get it by using the standalone Nuget command line tool.

#### Get the package by using Visual Studio

In Visual Studio, right-click your solution or project node and pick one of the Manage Nuget Packages commands.  Search for **Microsoft.PackageSupportFramework** or **PSF** to find the package on Nuget.org. Then, install it.

#### Get the package by using the command line tool

Install the Nuget command line tool from this location: https://www.nuget.org/downloads. Then, from the Nuget command line, run this command:

```
nuget install Microsoft.PackageSupportFramework
```

### Add the Package Support Framework files to your package

Add the required 32-bit and 64-bit PSF  DLLs and executable files to the package directory. Use the following table as a guide. You'll also want to include any runtime fixes that you need. In our example, we need the file redirection runtime fix.

| Application executable is x64 | Application executable is x86 |
|-------------------------------|-----------|
| [ShimLauncher64.exe](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimLauncher/readme.md) |  [ShimLauncher32.exe](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimLauncher/readme.md) |
| [ShimRuntime64.dll](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimRuntime/readme.md) | [ShimRuntime32.dll](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimRuntime/readme.md) |
| [ShimRunDll64.exe](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimRunDll/readme.md) | [ShimRunDll32.exe](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/ShimRunDll/readme.md) |

Your package content should now look something like this.

![Package Binaries](images/desktop-to-uwp/package_binaries.png)

### Modify the package manifest

Open your package manifest in a text editor, and then set the `Executable` attribute of the `Application` element to the name of the shim launcher executable file.  If you know the architecture of your target application, select the appropriate version, ShimLauncher32.exe or ShimLauncher64.exe.  If not, ShimLauncher32.exe will work in all cases.  Here's an example.

```xml
<Package ...>
  ...
  <Applications>
    <Application Id="PSFSample"
                 Executable="ShimLauncher32.exe"
                 EntryPoint="Windows.FullTrustApplication">
      ...
    </Application>
  </Applications>
</Package>
```

### Create a configuration file

Create a file name ``config.json``, and save that file to the root folder of your package. Modify the declared app ID of the config.json file to point to the executable that you just replaced. Using the knowledge that you gained from using Process Monitor, you can also set the working directory as well as use the file redirection shim to redirect reads/writes to .log files under the package-relative "PSFSampleApp" directory.

```json
{
    "applications": [
        {
            "id": "PSFSample",
            "executable": "PSFSampleApp/PSFSample.exe",
            "workingDirectory": "PSFSampleApp/"
        }
    ],
    "processes": [
        {
            "executable": "PSFSample",
            "shims": [
                {
                    "dll": "FileRedirectionShim.dll",
                    "config": {
                        "redirectedPaths": {
                            "packageRelative": [
                                {
                                    "base": "PSFSampleApp/",
                                    "patterns": [
                                        ".*\\.log"
                                    ]
                                }
                            ]
                        }
                    }
                }
            ]
        }
    ]
}
```
Following is a guide for the config.json schema:

| Array | key | Value |
|-------|-----------|-------|
| applications | id |  Use the value of the `Id` attribute of the `Application` element in the package manifest. |
| applications | executable | The package-relative path to the executable that you want to start. In most cases, you can get this value from your package manifest file before you modify it. It's the value of the `Executable` attribute of the `Application` element. |
| applications | workingDirectory | (Optional) A package-relative path to use as the working directory of the application that starts. If you don't set this value, the operating system uses the `System32` directory as the application's working directory. |
| processes | executable | In most cases, this will be the name of the `executable` configured above with the path and file extension removed. |
| shims | dll | Package-relative path to the shim  .appx  to load. |
| shims | config | (Optional) Controls how the shim dl behaves. The exact format of this value varies on a shim-by-shim basis as each shim can interpret this "blob" as it wants. |

The `applications`, `processes`, and `shims` keys are arrays. That means that you can use the config.json file to specify more than one application, process, and shim DLL.


### Package and Test the App

Next, create a package.

```
makeappx pack /d PackageContents /p PSFSamplePackageFixup.appx
```

Then, sign it.

```
signtool sign /a /v /fd sha256 /f ExportedSigningCertificate.pfx PSFSamplePackageFixup.appx
```

For more information, see [how to create a package signing certificate](https://docs.microsoft.com/en-us/windows/desktop/appxpkg/how-to-create-a-package-signing-certificate)
and [how to sign a package using signtool](https://docs.microsoft.com/en-us/windows/desktop/appxpkg/how-to-sign-a-package-using-signtool)

Using PowerShell, install the package.

>[!NOTE]
> Remember to uninstall the package first.

```
powershell Add-AppxPackage .\PSFSamplePackageFixup.appx
```

Run the application and observe the behavior with runtime fix applied.  Repeat the diagnostic and packaging steps as necessary.

### Use the Trace Shim

An alternative technique to diagnosing packaged application compatibility issues is to use the Trace Shim. This DLL is included with the PSF and provides a detailed diagnostic view of the app's behavior, similar to Process Monitor.  It is specially designed to reveal application compatibility issues.  To use the Trace Shim, add the DLL to the package, add the following fragment to your config.json, and then package and install your application.

```json
{
    "dll": "TraceShim.dll",
    "config": {
        "traceLevels": {
            "filesystem": "allFailures"
        }
    }
}
```

By default, the Trace Shim filters out failures that might be considered "expected".  For example, applications might try to unconditionally delete a file without checking to see if it already exists, ignoring the result. This has the unfortunate consequence that some unexpected failures might get filtered out, so in the above example, we opt to receive all failures from filesystem functions. We do this because we know from before that the attempt to read from the Config.txt file fails with the message "file not found". This is a failure that is frequently observed and not generally assumed to be unexpected. In practice it's likely best to start out filtering only to unexpected failures, and then falling back to all failures if there's an issue that still can't be identified.

By default, the output from the Trace Shim gets sent to the attached debugger. For this example, we aren't going to attach a debugger, and will instead use the [DebugView](https://docs.microsoft.com/en-us/sysinternals/downloads/debugview) program from SysInternals to view its output. After running the app, we can see the same failures as before, which would point us towards the same runtime fixes.

![TraceShim File Not Found](images/desktop-to-uwp/traceshim_filenotfound.png)

![TraceShim Access Denied](images/desktop-to-uwp/traceshim_accessdenied.png)

## Debug, extend, or create a runtime fix

You can use Visual Studio to debug a runtime fix, extend a runtime fix, or create one from scratch. You'll need to do these things to be successful.

> [!div class="checklist"]
> * Add a packaging project
> * Add project for the runtime fix
> * Add a project that starts the Shim Launcher executable
> * Configure the packaging project

When you're done, your solution will look something like this.

![Completed solution](images/desktop-to-uwp/runtime-fix-project-structure.png)

Let's look at each project in this example.

| Project | Purpose |
|-------|-----------|
| DesktopApplicationPackage | This project is based on the [Windows Application Packaging project](desktop-to-uwp-packaging-dot-net.md) and it outputs the the MSIX package. |
| Runtimefix | This is a C++ Dynamic-Linked Library project that contains one or more replacement functions that serve as the runtime fix. |
| ShimLauncher | This is C++ Empty Project. This project is a place to collect the runtime distributable files of the Package Support Framework. It outputs an executable file. That executable is the first thing that runs when you start the solution. |
| WinFormsDesktopApplication | This project contains the source code of a desktop application. |

To look at a complete sample that contains all of these types of projects, see [PSFSample](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/samples/PSFSample/).

Let's walk through the steps to create and configure each of these projects in your solution.


### Create a package solution

If you don't already have a solution for your desktop application, create a new **Blank Solution** in Visual Studio.

![Blank solution](images/desktop-to-uwp/blank-solution.png)

You may also want to add any application projects you have.

### Add a packaging project

If you don't already have a **Windows Application Packaging Project**, create one and add it to your solution.

![Package project template](images/desktop-to-uwp/package-project-template.png)

For more information on Windows Application Packaging project, see [Package your application by using Visual Studio](desktop-to-uwp-packaging-dot-net.md).

In **Solution Explorer**, right-click the packaging project, select **Edit**, and then add this to the bottom of the project file:

```
<Target Name="PSFRemoveSourceProject" AfterTargets="ExpandProjectReferences" BeforeTargets="_ConvertItems">
<ItemGroup>
  <FilteredNonWapProjProjectOutput Include="@(_FilteredNonWapProjProjectOutput)">
  <SourceProject Condition="'%(_FilteredNonWapProjProjectOutput.SourceProject)'=='<your runtime fix project name goes here>'" />
  </FilteredNonWapProjProjectOutput>
  <_FilteredNonWapProjProjectOutput Remove="@(_FilteredNonWapProjProjectOutput)" />
  <_FilteredNonWapProjProjectOutput Include="@(FilteredNonWapProjProjectOutput)" />
</ItemGroup>
</Target>
```

### Add project for the runtime fix

Add a C++ **Dynamic-Link Library (DLL)** project to the solution.

![Runtime fix library](images/desktop-to-uwp/runtime-fix-library.png)

Right-click the that project, and then choose **Properties**.

In the property pages, find the **C++ Language Standard** field, and then in the drop-down list next to that field, select the **ISO C++17 Standard (/std:c++17)** option.

![ISO 17 Option](images/desktop-to-uwp/iso-option.png)

Right-click that project, and then in the context menu, choose the **Manage Nuget Packages** option. Ensure that the **Package source** option is set to **All** or **nuget.org**.

Click the settings icon next that field.

Search for the *PSF** Nuget package, and then install it for this project.

![nuget package](images/desktop-to-uwp/psf-package.png)

If you want to debug or extend an existing runtime fix, add the runtime fix files that you obtained by using the guidance described in the [Find a runtime fix](#find) section of this guide.

If you intend to create a brand new fix, don't add anything to this project just yet. We'll help you add the right files to this project later in this guide. For now, we'll continue setting up your solution.

### Add a project that starts the Shim Launcher executable

Add a C++ **Empty Project** project to the solution.

![Empty project](images/desktop-to-uwp/blank-app.png)

Add the **PSF** Nuget package to this project by using the same guidance described in the previous section.

Open the property pages for the project, and in the **General** settings page, set the **Target Name** property to ``ShimLauncher32`` or ``ShimLauncher64`` depending on the architecture of your application.

![shim launcher reference](images/desktop-to-uwp/shim-exe-reference.png)

Add a project reference to the runtime fix project in your solution.

![runtime fix reference](images/desktop-to-uwp/reference-fix.png)

Right-click the reference, and then in the **Properties** window, apply these values.

| Property | Value |
|-------|-----------|-------|
| Copy local | True |
| Copy Local Satellite Assemblies | True |
| Reference Assembly Output | True |
| Link Library Dependencies | False |
| Link Library Dependency Inputs | False |

### Configure the packaging project

In the packaging project, right-click the **Applications** folder, and then choose **Add Reference**.

![Add Project Reference](images/desktop-to-uwp/add-reference-packaging-project.png)

Choose the shim launcher project and your desktop application project, and then choose the **OK** button.

![Desktop project](images/desktop-to-uwp/package-project-references.png)

>[!NOTE]
> If you don't have the source code to your application, just choose the shim launcher project. We'll show you how to reference your executable when you create a configuration file.

In the **Applications** node, right-click the shim launcher application, and then choose **Set as Entry Point**.

![Set entry point](images/desktop-to-uwp/set-startup-project.png)

Add a file named ``config.json`` to your packaging project, then, copy and paste the following json text into the file. Set the **Package Action** property to **Content**.

```json
{
    "applications": [
        {
            "id": "",
            "executable": "",
            "workingDirectory": ""
        }
    ],
    "processes": [
        {
            "executable": "",
            "shims": [
                {
                    "dll": "",
                    "config": {
                    }
                }
            ]
        }
    ]
}
```
Provide a value for each key. Use this table as a guide.

| Array | key | Value |
|-------|-----------|-------|
| applications | id |  Use the value of the `Id` attribute of the `Application` element in the package manifest. |
| applications | executable | The package-relative path to the executable that you want to start. In most cases, you can get this value from your package manifest file before you modify it. It's the value of the `Executable` attribute of the `Application` element. |
| applications | workingDirectory | (Optional) A package-relative path to use as the working directory of the application that starts. If you don't set this value, the operating system uses the `System32` directory as the application's working directory. |
| processes | executable | In most cases, this will be the name of the `executable` configured above with the path and file extension removed. |
| shims | dll | Package-relative path to the shim DLL to load. |
| shims | config | (Optional) Controls how the shim dl behaves. The exact format of this value varies on a shim-by-shim basis as each shim can interpret this "blob" as it wants. |

When you're done, your ``config.json`` file will look something like this.

```json
{
  "applications": [
    {
      "id": "DesktopApplication",
      "executable": "DesktopApplication/WinFormsDesktopApplication.exe",
      "workingDirectory": "WinFormsDesktopApplication"
    }
  ],
  "processes": [
    {
      "executable": ".*App.*",
      "shims": [ { "dll": "RuntimeFix.dll" } ]
    }
  ]
}

```

>[!NOTE]
> The `applications`, `processes`, and `shims` keys are arrays. That means that you can use the config.json file to specify more than one application, process, and shim DLL.

### Debug a runtime fix

In Visual Studio, press F5 to start the debugger.  The first thing that starts is the shim launcher application, which in turn, starts your target desktop application.  To debug the target desktop application, you'll have to manually attach to the desktop application process by choosing **Debug**->**Attach to Process**, and then selecting the application process. To permit the debugging of a .NET application with a native runtime fix DLL, select managed and native code types (mixed mode debugging).  

Once you've set this up, you can set break points next to lines of code in the desktop application code and the runtime fix project. If you don't have the source code to your application, you'll be able to set break points only next to lines of code in your runtime fix project.

>[!NOTE]
> While Visual Studio gives you the simplest development and debugging experience, there are some limitations, so later in this guide, we'll discuss other debugging techniques that you can apply.

## Create a runtime fix

If there isn't yet a runtime fix for the issue that you want to resolve, you can create a new runtime fix by writing replacement functions and including any configuration data that makes sense. Let's look at each part.

### Replacement functions

First, identify which function calls fail when your application runs in an MSIX container. Then, you can create replacement functions that you'd like the runtime manager to call instead. This gives you an opportunity to replace the implementation of a function with behavior that conforms to the rules of the modern runtime environment.

In Visual Studio, open the runtime fix project that you created earlier in this guide.

Declare the ``SHIM_DEFINE_EXPORTS`` macro and then add a include statement for the `shim_framework.h` at the top of each .CPP file where you intend to add the functions of your runtime fix.

```c++
#define SHIM_DEFINE_EXPORTS
#include <shim_framework.h>
```
>[!IMPORTANT]
>Make sure that the `SHIM_DEFINE_EXPORTS` macro appears before the include statement.

Create a function that has the same signature of the function who's behavior you want to modify. Here's an example function that replaces the `MessageBoxW` function.

```c++
auto MessageBoxWImpl = &::MessageBoxW;
int WINAPI MessageBoxWShim(
    _In_opt_ HWND hwnd,
    _In_opt_ LPCWSTR,
    _In_opt_ LPCWSTR caption,
    _In_ UINT type)
{
    return MessageBoxWImpl(hwnd, L"SUCCESS: This worked", caption, type);
}

DECLARE_SHIM(MessageBoxWImpl, MessageBoxWShim);
```

The call to `DECLARE_SHIM` maps the `MessageBoxW` function to your new replacement function. When your application attempts to call the `MessageBoxW` function, it will call the replacement function instead.

#### Protect against recursive calls to functions in runtime fixes

You can optionally apply the `reentrancy_guard` type to your functions that protect against recursive calls to functions in runtime fixes.

For example, you might produce a replacement function for the `CreateFile` function. Your implementation might call the `CopyFile` function, but the implementation of the `CopyFile` function might call the `CreateFile` function. This may lead to an infinite recursive cycle of calls to the `CreateFile` function.

For more information on `reentrancy_guard` see [authoring.md](https://github.com/Microsoft/MSIX-PackageSupportFramework/blob/master/Authoring.md)

### Configuration data

If you want to add configuration data to your runtime fix, consider adding it to the ``config.json``. That way, you can use the `ShimQueryCurrentDllConfig` to easily parse that data. This example parses a boolean and string value from that configuration file.

```c++
if (auto configRoot = ::ShimQueryCurrentDllConfig())
{
    auto& config = configRoot->as_object();

    if (auto enabledValue = config.try_get("enabled"))
    {
        g_enabled = enabledValue->as_boolean().get();
    }

    if (auto logPathValue = config.try_get("logPath"))
    {
        g_logPath = logPathValue->as_string().wstring();
    }
}
```

## Other debugging techniques

While Visual Studio gives you the simplest development and debugging experience, there are some limitations.

First, F5 debugging runs the application by deploying loose files from the package layout folder path, rather than installing from a .appx package.  The layout folder typically does not have the same security restrictions as an installed package folder. As a result, it may not be possible to reproduce package path access denial errors prior to applying a runtime fix.

To address this issue, use .appx package deployment rather than F5 loose file deployment.  To create a .appx package file, use the [MakeAppx](https://docs.microsoft.com/en-us/windows/desktop/appxpkg/make-appx-package--makeappx-exe-) utility from the Windows SDK, as described above. Or, from within Visual Studio, right-click your application project node and select **Store**->**Create App Packages**.

Another issue with Visual Studio is that it does not have built-in support for attaching to any child processes launched by the debugger.   This makes it difficult to debug logic in the startup path of the target application, which must be manually attached by Visual Studio after launch.

To address this issue, use a debugger that supports child process attach.  Note that it is generally not possible to attach a just-in-time (JIT) debugger to the target application.  This is because most JIT techniques involve launching the debugger in place of the target app, via the ImageFileExecutionOptions registry key.  This defeats the detouring mechanism used by ShimLauncher.exe to inject ShimRuntime.dll into the target app.  WinDbg, included in the [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/index), and obtained from the [Windows SDK](https://developer.microsoft.com/en-US/windows/downloads/windows-10-sdk), supports child process attach.  It also now supports directly [launching and debugging a UWP app](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugging-a-uwp-app-using-windbg#span-idlaunchinganddebuggingauwpappspanspan-idlaunchinganddebuggingauwpappspanspan-idlaunchinganddebuggingauwpappspanlaunching-and-debugging-a-uwp-app).

To debug target application startup as a child process, start ``WinDbg``.

```
windbg.exe -plmPackage PSFSampleWithFixup_1.0.59.0_x86__7s220nvg1hg3m -plmApp PSFSample
```

At the ``WinDbg`` prompt, enable child debugging and set appropriate breakpoints.

```
.childdbg 1
g
```
(execute until target application starts and breaks into the debugger)

```
sxe ld fixup.dll
g
```
(execute until the fixup DLL is loaded)

```
bp ...
```

>[!NOTE]
> [PLMDebug](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/plmdebug) can be also used to attach a debugger to an app upon launch, and is also included in the [Debugging Tools for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/index).  However, it is more complex to use than the direct support now provided by WinDbg.

## Support and feedback

**Find answers to your questions**

Have questions? Ask us on Stack Overflow. Our team monitors these [tags](http://stackoverflow.com/questions/tagged/project-centennial+or+desktop-bridge). You can also ask us [here](https://social.msdn.microsoft.com/Forums/en-US/home?filter=alltypes&sort=relevancedesc&searchTerm=%5BDesktop%20Converter%5D).
