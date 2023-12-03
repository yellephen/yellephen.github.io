---
layout: post
title:  "Dorky Website"
date:   2023-12-04 06:53:00 +1100
categories: Tool
---

Some people are awesome at finding technical articles with google. I'm bad at it. I don't know how they do it. Dorking felt clunky so I wrote a tiny bit of JavaScript and host this website in a VM for me to use.
![image of mstsc connecting to localhost](/images/goodsearch2.png?raw=true)
![image of mstsc connecting to localhost](/images/goodsearch1.png?raw=true)

Here's the javascript that does it.
```
<script>
var inputElement = document.getElementById("myInput");

var domains = [
"github.io", 
"medium.com",
"reddit.com",
"microsoft.com",
"twitter.com",
"substack.com"
];

inputElement.addEventListener("keyup", function(event) {
            if (event.keyCode === 13) {
                var inputValue = inputElement.value;
                domains.forEach(function(domain){
                	window.open("https://google.com/search?q=site:" + domain + " " + inputValue,'_blank')
                	
                });
                window.open("https://google.com/search?q=" + inputValue,'_blank')
            }
        });

</script>
```

It's a static site so you can just put the source in a folder in `/var/www/html`. Need to allow popups from your browser. [https://github.com/yellephen/goodsearch](https://github.com/yellephen/goodsearch). I'll see if the domains I'm querying grows/changes over time.