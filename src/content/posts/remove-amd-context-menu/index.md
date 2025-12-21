---
title: "删除右键菜单中的“AMD Software: Adrenalin Edition”"
pinned: false
description: 究竟是谁会去点这个P用没有的右键菜单啊？？？
tags: [日常, 优化]
category: 日常
author: 雪纷飞
draft: false
updated: 2025-12-21
published: 2025-12-21
image: "amd-logo.avif"
---

# 受害者

![无孔不入的右键菜单][amd-victim]

# 解决方案（来自[hosiet][hosiet-sol]）

使用管理员权限执行下面的PowerShell命令即可：

```ps1
Set-Location "Registry::HKEY_CLASSES_ROOT\PackagedCom\ClassIndex\"
$target = Get-ChildItem . -Recurse -Depth 1 | 
    Where-Object { $_.PSChildName -like "*AdvancedMicroDevicesInc*" } | 
    Select-Object -First 1
$uuid = Split-Path $target.PSParentPath -Leaf
Write-Host "找到匹配的 UUID: " $uuid -ForegroundColor Green

Set-Location 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Shell Extensions\'
New-Item Blocked
New-ItemProperty -Path .\Blocked\ -Name $uuid
Write-Host "已阻止AMD的右键菜单 " -ForegroundColor Green
```

在`New-Item`之前，可以先用`Get-ChildItem | Select-Object -Property Name`看一下，应该默认只有`Approved`和`Cached`，没有`Blocked`。当然，如果你有`Blocked`，那么跳过`New-Item`那一行就行了。

<!-- 引用 -->

[amd-victim]: ./amd-context-menu.png "AMD Software: Adrenalin Edition 菜单"
[hosiet-sol]: https://blog.hosiet.me/blog/2023/02/07/hide-amd-software-entry-in-windows-11-context-menu-right-click-menu/ "hosiet给出的解决方案"