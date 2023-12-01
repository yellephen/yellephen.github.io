---
layout: post
title:  "BloodHound moving to community edition"
date:   2023-12-02 08:42:53 +1100
categories: Tip
---

This isn't super recent but I had to start from a fresh testing box recently and had to update myself on the current state of BloodHound. I ran into problems when I used the kali apt BloodHound version and the latest SharpHound - the zip file wasn't importing, just hung, no error.

BloodHound is getting commercialised and they're not updating the original BloodHound repo at https://github.com/BloodHoundAD/BloodHound anymore. They've split it into an enterprise version that I know nothing about and a community edition at https://github.com/SpecterOps/BloodHound.

Chatting to people about the split, some think they've nerfed the original when moving to the CE version (no 'mark as pwned' function), others have said it's just all in progress and the feature backlog for CE looks awesome.

Either way, if you want to use the old BloodHound it's still there, just have to use the right SharpHound for collection.

Bloodhound v4.3.1 is the latest before the CE and is the version currently in the kali apt repository. The latest version of SharpHound that's compatible is v1.1.1 as far as I can tell, availble here [https://github.com/BloodHoundAD/SharpHound/releases](https://github.com/BloodHoundAD/SharpHound/releases).