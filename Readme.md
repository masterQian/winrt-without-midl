## WinUI3 Project Independent Version Based On C++

1. Download ***Visual Studio 2022*** and ***Windows APP SDK C++ VS2022 Templates*** extension (In Visual Studio Installer).
2. Create a new project based on ***Blank project, packaged (WinUI3 in desktop version)***.
3. Update all SDKs in the NuGet package of the solution.
4. Delete file ***Package.appxmanifest***.
5. Edit file ***App1.vcxproj***
``` xml
<PropertyGroup Label="Globals">
	<AppxPackage>false</AppxPackage>
	<WindowsPackageType>None</WindowsPackageType>
	<WindowsAppSDKSelfContained>true</WindowsAppSDKSelfContained>
</PropertyGroup>
```
6. Build your project and organize the Release folder (At least some files need to be retained to ensure the independent operation of the WinUI3 project).

File | Role
------------ | -------------
Assets | Resource Folder
App1.exe | Main Executable Program
Microsoft.WindowsAppRuntime.Bootstrap.dll | WinUI3 Startup Program Entry
resources.pri | Compile Resource File
*.dll | WinUI3 Library Dependencies

---

The size of an executable program compiled for an empty project is about 50MB.

In fact, you can use software such as ***procexp64*** to determine which DLLs your WinUI3 project specifically uses, and then delete the DLLs that the project does not depend on to save space.

---

In addition, for better compatibility, it is best to also include ***msvcp140.dll***, ***vcruntime140.dll*** and ***vcruntime140_1.dll*** in the directory.

You can try testing your WinUI3 standalone project in a Windows Sandbox.
