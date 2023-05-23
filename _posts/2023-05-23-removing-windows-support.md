---
layout: post
title:  "Removing Support for Windows"
date: '2023-05-23 00:05:00 -0500'
---

**Let's talk about our decision to remove support for Windows.** 

Nickvision has recently decided to remove support for Windows as a platform for our apps. We do not make this decision lightly, but feel that it is necessary for the growth of our applications.

## Problems with Windows as an OS

Windows 11 is becoming a platform for Microsoft to push ads instead of a platform to get work done. Although having a pretty design, Windows is becoming an OS for Microsoft to push ads and steal and seal users' data and it's become more and more apparent which each update. 

Not to mention the load of design inconsistencies and legacy code baked into the operating system, with no cleanup in sight.

## Problems with Windows as a development platform

The Windows versions of our apps are distributed as MSIX packages on the Microsoft Store. We felt as this was the best way to reach users instead of using traditional EXEs and installer. But, boy, were we wrong!

Being built with the new [Windows App SDK](https://github.com/microsoft/WindowsAppSDK), Microsoft basically forces developers into using MSIX packaging instead of EXEs and like I said, we didn't mind, however, this came with a ton of issues:
- First, MSIX packages cannot be installed freely as EXEs can. They require a digital signature which can only be generated using complicated CLI tools from Microsoft. Although increasing security, there are no options for power-users who want to quickly test their unsigned MSIX packages.
- Secondly, MSIX apps are weirdly sandboxed. Flatpak gets it right, provided limiting file-system access, but besides that you are free to run other commands and utilities as they will only effect the sandboxed app. When it comes to MSIX, not only is the file-system restricted, but you can't run external utilities that edit this sandboxed filesystem without getting permissions denied errors (which makes no sense because if the file system is sandboxed, these utilities should be allowed to run since they wouldn't effect the host system).

Besides MSIX issues, the Microsoft Store is also a mess. 

Microsoft has recently gave us a lot of problems with keeping our apps on the store. They brought up some issues in our apps and upon fixing them, and being 100% sure they were fixed, Microsoft Store certification employees still complained about the same issues and denied updates to our apps. A few weeks later, without being able to update our apps, Microsoft removed our apps from the store completely stating that they didn't meet their "quality expectations". 

Now we could just choose to not use the Windows App SDK and instead use another framework, such as WPF or Avalonia, and distribute our apps traditionally, instead of using MSIX and the Store, however for the low number of Windows users using our apps and with the difficultly of becoming popular on the Windows platform to begin with, we've decided that it would be best to stop supporting Windows as a platform for our apps all together. 

As previously stated, we do not make this decision lightly, but feel that it is in the best interest for the growth of Nickvision apps.

For users who cannot switch to Linux but wish to continue using Nickvision apps, we recommend to setup a linux distro via [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) (we recommend OpenSuse). Through your WSL distro, you can install `flatpak` and install the linux versions of our apps and continue using them through WSL.

## Moving Forward

We will continue to support and develop our apps for Linux (GNOME specificity with libadwaita) and plan to develop mobile companions for our apps in the near future.

<script src="https://utteranc.es/client.js"
        repo="nickvisionapps/nickvisionapps.github.io"
        issue-term="url"
        label="blog-comments"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
We thank all contributors, users, and sponsors for the continued support of our applications :)
