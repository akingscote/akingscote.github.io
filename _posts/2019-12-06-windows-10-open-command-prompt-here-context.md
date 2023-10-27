---
title: "Windows 10 - Open Command Prompt here context"
date: "2019-12-06"
categories: 
  - "miscellaneous"
---

In Windows 10, the useful option to open a terminal window from the right click context menu has been removed. This is really frustrating and even when you make the necessary changes, the registry settings seem to get changed with every big update.

Ive exported my registry values, so you just need to merge these two files into your registry (right click -> merge on the files) and youll have the context menu back.

## CommandPromptHere.reg

```
Windows Registry Editor Version 5.00
 
[HKEY_CLASSES_ROOT\Directory\Background\shell\cmd]
@="@shell32.dll,-8506"
"Extended"=""
"NoWorkingDirectory"=""
"ShowBasedOnVelocityId"=dword:00639bc8
"_HideBasedOnVelocityId"=dword:00639bc8
 
[HKEY_CLASSES_ROOT\Directory\Background\shell\cmd\command]
@="cmd.exe /s /k pushd \"%V\""
```

## CommandPromptHere2.reg

```
Windows Registry Editor Version 5.00
 
[HKEY_CLASSES_ROOT\Directory\shell\cmd]
@="@shell32.dll,-8506"
"Extended"=""
"NoWorkingDirectory"=""
"_HideBasedOnVelocityId"=dword:00639bc8
 
[HKEY_CLASSES_ROOT\Directory\shell\cmd\command]
@="cmd.exe /s /k pushd \"%V\""
```

The changes should take effect instantly. Now hold down shift, then right click on the explorer somewhere (or on the desktop) and you should now see the option to open a command window in addition to the option of opening a powershell command.

![](/images/context-menu.png)
