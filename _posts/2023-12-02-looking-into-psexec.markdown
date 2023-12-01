---
layout: post
title:  "Looking into psexec.exe and impacket-psexec"
date:   2023-12-02 08:42:53 +1100
categories: Investigation
---
psexec.exe from the SysInternal suite and impacket-psexec from impacket can be used for remote code execution to windows hosts launched from cmd.exe or python respectively. I've been using them both in the OSEP labs and there's a few differences. 

With psexec.exe, you might run a pass-the-ticket attack so that you have a CIFS ticket to the target server. If the ticket you've stolen is for say `corp\administrator`, when you psexec to the target, the user context you get is `corp\administrator`. 
This might look like this 
```
PS C:\Tools> klist

Current LogonId is 0:0x11e854

Cached Tickets: (1)

#0>     Client: administrator @ CORP.COM
        Server: CIFS/FILES02.corp.com @ CORP.COM
        KerbTicket Encryption Type: AES-256-CTS-HMAC-SHA1-96
        Ticket Flags 0x40a10000 -> forwardable renewable pre_authent name_canonicalize
        Start Time: 11/30/2023 16:03:30 (local)
        End Time:   12/1/2023 2:03:30 (local)
        Renew Time: 12/7/2023 16:03:30 (local)
        Session Key Type: AES-128-CTS-HMAC-SHA1-96
        Cache Flags: 0
        Kdc Called:

PS C:\Tools> psexec.exe \\files02 cmd -accepteula

PsExec v2.2 - Execute processes remotely
Copyright (C) 2001-2016 Mark Russinovich
Sysinternals - www.sysinternals.com


Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
corp\administrator
```

If you use the same ticket with impacket-psexec (first you probably need to convert the kirbi from rubeus to a CCACHE and add a host entry for the target), it looks like this.
```
┌──(kali㉿kali)-[~]
└─$ echo $KRB5CCNAME
/home/kali/administrator.ccache

┌──(kali㉿kali)-[~]
└─$ impacket-psexec corp.com/administrator@files02.corp.com -k --no-pass
Impacket v0.11.0 - Copyright 2023 Fortra

[] Requesting shares on files02.corp.com.....
[] Found writable share ADMIN$
[] Uploading file bczmEmea.exe
[] Opening SVCManager on files02.corp.com.....
[] Creating service IVdT on files02.corp.com.....
[*] Starting service IVdT.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.169]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

So with the impacket version, you get a system shell while with psexec it respects the ticket and gives you a shell as the user. This can be frustrating, maybe I'd rather be the domain admin on files02 than be system. And sometimes with psexec.exe you'll encounter the below, ruling it out of your attack path.
```
PsExec v2.43 - Execute processes remotely Copyright (C) 2001-2023 Mark Russinovich Sysinternals - www.sysinternals.com 

Could not start PSEXESVC service on files02.corp.com: Access is denied.
```
While the same ticket with the impacket version will get you on. So you have an admin CIFS ticket for files02 but you can't operate as admin, annoying.

So I figured impacket-psexec was doing some privilege escalation from the domain user to system so I had a look to see if I could patch the privilege escalation to get my domain admin shell. Impackets pretty big and distributed and I spent some time tracing through the code to figure it out. I'll skip talking about the various hypotheses I had along the way about how it was escalating except to say I thought it was the remcomsvc for a while which is a binary blob in the impacket repo and it's own project that functions as the default reverse shell for impacket-psexec.

The impacket files are fairly distributed around the default install so following imports was tricky until I found this function in python.
```
┌──(kali㉿kali)-[/usr/share/doc/python3-impacket/examples]
└─$ python3                                                                                                         
Python 3.11.6 (main, Oct  8 2023, 05:06:43) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import impacket.dcerpc.v5
>>> print(impacket.dcerpc.v5.__file__)
/usr/lib/python3/dist-packages/impacket/dcerpc/v5/__init__.py
>>> 
```

Before going through the code I had some idea that impacket-psexec created a service with SMB and executed that to get a shell back. That was about right. It authenticates with SMB and uses a protocol called DCERPC talking to the service control manager to create the service then starts it to get the reverse shell.

I eventually found the code that creates the service in scmr.py.
```
def hRCreateServiceW(dce, hSCManager, lpServiceName, lpDisplayName, dwDesiredAccess=SERVICE_ALL_ACCESS, dwServiceType=SERVICE_WIN32_OWN_PROCESS, dwStartType=SERVICE_AUTO_START, dwErrorControl=SERVICE_ERROR_IGNORE, lpBinaryPathName=NULL, lpLoadOrderGroup=NULL, lpdwTagId=NULL, lpDependencies=NULL, dwDependSize=0, lpServiceStartName=NULL, lpPassword=NULL, dwPwSize=0):
    createService = RCreateServiceW()
    createService['hSCManager'] = hSCManager
    createService['lpServiceName'] = checkNullString(lpServiceName)
    createService['lpDisplayName'] = checkNullString(lpDisplayName)
    createService['dwDesiredAccess'] = dwDesiredAccess
    createService['dwServiceType'] = dwServiceType
    createService['dwStartType'] = dwStartType
    createService['dwErrorControl'] = dwErrorControl
    createService['lpBinaryPathName'] = checkNullString(lpBinaryPathName)
    createService['lpLoadOrderGroup'] = checkNullString(lpLoadOrderGroup)
    createService['lpdwTagId'] = lpdwTagId
    # Strings MUST be NULL terminated for lpDependencies
    createService['lpDependencies'] = lpDependencies
    createService['dwDependSize'] = dwDependSize
    createService['lpServiceStartName'] = checkNullString(lpServiceStartName)
    createService['lpPassword'] = lpPassword
    createService['dwPwSize'] = dwPwSize
    return dce.request(createService)
```

I searched for a reference to hRCreateServiceW and found this MS document and the parameters seemed to line up [https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-createservicew](https://learn.microsoft.com/en-us/windows/win32/api/winsvc/nf-winsvc-createservicew). The lpServiceStartName is what I was interested in because that's the user under which the service will run - maybe I could just update this and I'd be good. I got excited and made my patch so that I could pass a username at the command line and passed it all the way down to lpServiceStartName but I got an authentication error.

I looked around a bit more and noticed that what impacket-psexec was doing was passing null to lpServiceStartName and the documentation states.
```
If this parameter is NULL, CreateService uses the LocalSystem account.
```

So that's how impacket-psexec is working. It's using the CreateService API to create a service and just like I've encountered before, when creating a service you need a username and password. Just that it also has this function where passing null creates a system service. There wasn't a simple patch I could apply to prevent the change of context to system.

This still leaves some questions though. I'm interested in the service control manager and CreateService API authentication. You need a high privileged account to connect with impacket-psexec but I don't know exactly what rights need to be granted to allow it or where those rights are granted and what misconfigurations could exist. It also leaves the question of how psexec.exe is functioning, which does get you the user context you authenticated with. I believe it operates in a similar way particularly hinted at by the error message I saw earlier `Could not start PSEXESVC service on files02.corp.com: Access is denied.`.

