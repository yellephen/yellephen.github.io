---
layout: post
title:  "Is NT Authority\\System Anonymous on the Network"
date:   2022-07-22 08:42:53 +1100
categories: Investigation
---

Was recently working on a lab where I had a session as `nt authority\system`. It was an AD lab and I tried `net users /domain` but got an error.

![image of net user /domain failing](/images/netuserfail.png?raw=true)

It's not a domain account so fair enough I thought but on the other hand I thought I'd seen this work before. It made me wonder about how `nt authority\system` presents itself over the network so I started an SMB server to capture an authentication request while running as system.

![](/images/anonsmb.png?raw=true)

This didn't look like much, maybe anonymous authentication. It would make sense that `net users /domain` wouldn't work when I'm authenticating like that.

I chatted to a few people, one who was sure that `nt authority\system` would act as the domain computer account of the computer it owns on the network. So I tried it out in another lab and got some different results.

![](/images/credsmb.png?raw=true)

So here it was authenticating as the domain computer account and `net user /domain` was also working. So why different in the other lab? Throwing around some wild conjecture with a colleague he suggested it could be something in local security policy. So I had a browse and found this.

![](/images/netsecuritygp.png?raw=true)

Looked promising. So I looked up the page for the setting
[https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/network-security-allow-local-system-to-use-computer-identity-for-ntlm](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/network-security-allow-local-system-to-use-computer-identity-for-ntlm).

Turns out that at Vista/2008 the default for this setting changed. Before then, `nt authority\system` would authenticate over the network as anonymous and including and after, it defaults to authenticating as the domain computer account.