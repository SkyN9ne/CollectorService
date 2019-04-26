# SystemCollector
PoC for Privilege Escalation in Windows 10 Diagnostics Hub Standard Collector Service

## Affected Products

* Windows 10
* Windows Server
* Windows Server 2016
* Visual Studio 2015 Update 3
* Visual Studio 2017

### Summary

The Diagnostics Hub Packaging library, used by Windows Standard Collector Service, can be forced to copy an arbitrary file to an arbitrary location due to lack of client impersonation in `DiagnosticsHub.StandardCollector.Runtime.dll`. 

Here is a detailed write-up on how this vulnerability was found and exploited: [Privilege Escalation Vulnerability in Windows Standard Collector Service](https://www.atredis.com/blog/cve-2018-0952-privilege-escalation-vulnerability-in-windows-standard-collector-service).

### Technical Details

The Standard Collector Service allows for a several values to be defined when configuring a diagnostics session, including the scratch directory and session ID. The session ID can be any GUID and the scratch directory can be any location the user has write permissions too. If the collection session is configured with an ID of `c13851b2-b1e1-438f-bf73-949df897f1bf` and a scratch path of ` C:\Users\Bob\AppData\Local\Temp\Microsoft\F12\perftools\visualprofiler\`, the following events occur when calling the `GetCurrentResult` method of the `ICollectionSession` object:

1. An Event Trace Log (.etl) file is created in the scratch path: `C:\Users\Bob\AppData\Local\Temp\Microsoft\F12\perftools\visualprofiler\c13851b2-b1e1-438f-bf73-949df897f1bf.1.m.etl`
2. A Report folder is also created in the scratch path: `C:\Users\Bob\AppData\Local\Temp\Microsoft\F12\perftools\visualprofiler\Report.c13851b2-b1e1-438f-bf73-949df897f1bf.1`
3. A folder with a random GUID is created in the report folder: `C:\Users\Bob\AppData\Local\Temp\Microsoft\F12\perftools\visualprofiler\Report.c13851b2-b1e1-438f-bf73-949df897f1bf.1\EAD6A227-31D4-4EA2-94A9-5DF276F69E65`

These folders and ETL files are created by the collector service for the .diagsession package that is normally created when a session has ended. Calling the `Stop` method on the `ICollectionSession` object will cause the collector service to commit the diagnostics package by calling `Microsoft::DiagnosticsHub::Packaging::DhPackageDirectory::CommitPackage`. The `CommitPackage` function will copy or move the original `{scratch path}\c13851b2-b1e1-438f-bf73-949df897f1bf.1.m.etl` file to the random GUID folder: `{scratch path}\Report.c13851b2-b1e1-438f-bf73-949df897f1bf.1\EAD6A227-31D4-4EA2-94A9-5DF276F69E65\c13851b2-b1e1-438f-bf73-949df897f1bf.1.m.etl`

The copy/move operation triggered by the `CommitPackagingResult` function in `DiagnosticsHub.StandardCollector.Runtime.dll`, is performed without impersonating the user (unlike the initial file/folder creation), leading to a possible TOCTOU issue if the target folder is replaced with a mount point that redirects the copy to an arbitrary location. To exploit this issue in a useful way, an attacker would need to swap the contents of the ETL file before it is copied. This can be done by beating the race condition with an OpLock after the file handle has been released by the service.

Although we don't fully control the name of the .etl file that is copied, we can use the object directory symlink trick to control it. The mount point+symlink setup would look something like this:

- Mount point: `{scratch path}\Report.c13851b2-b1e1-438f-bf73-949df897f1bf.1\EAD6A227-31D4-4EA2-94A9-5DF276F69E65\` -> `\RPC Control\`
- Symlink: `\RPC Control\c13851b2-b1e1-438f-bf73-949df897f1bf.1.m.etl` -> `C:\Windows\System32\anything.dll`

Having control of the file contents, copy location, and file name gives an attacker numerous DLL loading possibilities. However, the included PoC demonstrates how control of the filename is not needed since the collector service happily load a DLL with any filename, as long as it is in `C:\Windows\System32` or `C:\Windows\System32\DiagSvcs` directory. This is done by starting a new collector session with an agent that has an assembly name matching the name of the copied DLL `c13851b2-b1e1-438f-bf73-949df897f1bf.1.m.etl`.

The included PoC is a VS solution with a C++ DLL project for the notepad.exe popping payload and a C# project to interact with the service and exploit the vulnerability with the NtApiDotNet library.

**Steps to reproduce:**

1. Build Visual Studio Solution
2. Execute SystemCollector.exe as a normal user

**Expected Result:**

The package commit operation impersonates the user and fails when trying to copy the file.

**Observed Result:**

The file is copied to the mount point target folder `C:\Windows\System32`, then loaded as a collector agent, and finally, notepad.exe is spawned as SYSTEM privileges.

### Additional References

* https://www.atredis.com/blog/cve-2018-0952-privilege-escalation-vulnerability-in-windows-standard-collector-service
* https://portal.msrc.microsoft.com/en-US/security-guidance/advisory/CVE-2018-0952
* https://github.com/atredispartners/advisories/blob/master/ATREDIS-2018-0004.md