---
layout: post
title: "windows文件进程占用"
description:
headline:
modified: 2016-09-10 03:06:10 +0800
category: windows
tags: [windows]
imagefeature: covers/programmer_cover.jpg
mathjax:
chart:
featured: true
---

很多情况下，我会遇到在删除windows系统下某个文件或文件夹时，系统提示我该文件或文件夹被其它进程占用，从而无法删除该文件。也许你会尝试安装一些其它辅助软件（比如<span style="color:red">流氓360软件、XX卸载大师</span>），反正我觉得这个实在是太重量级了，况且我根本不想安装这些XX软件。那么到底还有没有其它更加方便的方法可以让我找到是哪一个进程占用了该文件呢？

> 1. 右键 - Windows 7任务栏 - 启动任务管理器 - 性能 - 资源监视器；
> 2. 在控制台中点击“CPU”标签定位到该标签页下；
> 3. 在“关联的句柄”右侧的搜索框中输入“test”，此时系统会自动搜索与test句柄相关联的系统进程；
> 4. 可以看到搜索到的进程为cmd.exe。cmd.exe进程正在调用test文件夹，才造成了对该文件夹删除的失败。右键单击该进程，然后选择“结束进程”命令弹出警告对话框，确认后即可结束cmd.exe进程。最后，删除test文件夹，可以看到该文件夹被成功删除。
