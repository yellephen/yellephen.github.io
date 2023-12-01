---
layout: post
title:  "RDP to Training Boxes"
date:   2022-10-12 08:42:53 +1100
categories: Tip
---

I've encountered this a few times where lab boxes have RDP but rdesktop or xfreerdp on kali can't connect. I think it's something to do with credssp and the mutual authentication it can be configured to do - don't really know.
You get this when you try to connect from kali with rdesktop.
![image of rdesktop failing](/images/rdesktop.png?raw=true)

I'm running a windows host with a kali VM and a method I've found that often works is to use the windows mstsc and proxy through my instance of kali which is on the VPN I need to be on. To do this.

First start SSH on the kali box
`sudo systemctl start ssh`

Then connect to the kali VM from the host and forward the local port to the targets RDP port
```powershell
#forwards local port 13389 to 10.129.120.210:3389 through 192.168.43.130
ssh -L 13389:10.129.120.210:3389 kali@192.168.43.130
#ssh -L <local port you will connect to>:<target IP address>:<rdp port> <username>@><kali box IP>
```

Then RDP to the port at localhost on the host machine.
![image of mstsc connecting to localhost](/images/rdp.png?raw=true)

And you should get logon prompt. Go nuts.
![image of credential prompt for mstsc](/images/creds.png?raw=true)