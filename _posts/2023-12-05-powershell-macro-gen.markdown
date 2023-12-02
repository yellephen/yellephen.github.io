---
layout: post
title:  "PowerShell Macro Word Doc Generation"
date:   2023-12-05 08:42:53 +1100
categories: Tip
---

I'm working on a python tool that generates exploits and reverse shells from templates. Has some templates like some C# code that does some AV sandbox evasion and runs some shell code. The tool calls out to msfvenom or sliver for some shell code and compiles the C# into a .dll or .exe.

I'm working on some macro generation for that tool. Doesn't look like I can generate the macro without word installed so it's going to spit out a PowerShell script to be executed elsewhere. Took a bit of fiddling and chatGPTing to get the template of the PowerShell script together so pasting it here.

```
$wordApp = New-Object -ComObject Word.Application
$wordApp.Visible = $true
$doc = $wordApp.Documents.Add()
$paragraph = $doc.Content.Paragraphs.Add()
$paragraph.Range.Text = "Hello"

$project = $doc.VBProject
$component = $project.VBComponents.Add([Microsoft.Vbe.Interop.vbext_ComponentType]::vbext_ct_StdModule)

$code = @"
Sub AutoOpen()
    MsgBox "Hello", vbInformation, "Welcome Message"
End Sub
"@

$component.CodeModule.AddFromString($code)

$filepath = "C:\\users\\user\\desktop\\poshtest.doc"
$doc.SaveAs2($filepath,[Microsoft.Office.Interop.Word.WdSaveFormat]::wdFormatDocument97)
$wordApp.Quit()
[System.Runtime.Interopservices.Marshal]::ReleaseComObject($wordApp) | Out-Null
[System.GC]::Collect()
[System.GC]::WaitForPendingFinalizers()
```