
---
layout: post
title: "Optimizing Rider on Wayland"
date: 2026-07-29 21:00:00 +0200
categories: Dotnet, Linux, Wayland, Ubuntu, Gnome

---

Beeing a .NET developer on Linux is an awesome experiece, the days of .NET beeing a Windows only thing is long gone. For Linux (and Mac) users the go to places for writing .NET code is JetBrains Rider, Visual Studio Code and NeoVim. NeoVim and Visual Studio Code beeing more general and needing some tweaking for going from an texteditor to an actual IDE, the obivious choice for those of us seeking and IDE is therefore Rider. I've been using it on Linux for the last 4 years and it worked like a charm. But that said, there are of course ways in how we can make it even better.

### How do Rider run?

First thing first, Rider is a hybrid consisting of two different procceses where the UI runs on a modfied version on the JVM (Java virtual machine) called the JBR, JetBrains Runtime.
The analytic motor, ReSharper which is the motor that interprent and understands your code runs as a separate background process on .NET.
These two procceses work together when you type something the JVM process receives the input, draws the charatchers on screen and sends an IPC (inter-process communication) to the .NET proccese that does the syntax validation and other compiler related jobs and then sends it back to the JVM proccese to it to act on the result.

If curious check active proccesses with `pgrep -fa rider` and you will see the these two procceses running along with helper processes.

Since the JVM is a virtual machine and works as a sandbox on your maching we can configure the JVM for better perforomance. To do this we need to go to `Help -> Edit Custom VM Options`. 


### Increasing heapsize (give Rider more RAM)

The most appearant thing we can do is increasing the amount of memory Rider can consume. Depending on your machine you can set this to whatever see fit, the default value is 2048 MB. On my machine with 32 GB I have bumped it up to 6144 MB.

### Change the garbage collector

On a modern machine you can swap the garbage collector to G1GC. G1GC is optimized for doing the cleanup in shorter bursts, giving you a bit softer writing and scrolling experience under heavy load. The trade off beeing that it uses a bit more CPU, so again this is depending on the machine you have.

```
-Xmx6144m
```

### Use Wayland

For us running Wayland, make sure the JVM uses Wayland and not X11 or else Rider will run a compability layer called Xwayland which translates X11 calls to Wayland.

```
-Dcom.jetbrains.use.wayland.toolkit=true
```

### Use hardware acceleration for 2D-rendering

Instead of having the CPU drawing every UI element we can send it directly to the graphics card.

```
-Dsun.java2d.opengl=true
```

### Summary

So these simple changes can make Rider even better runing on Linxu with Wayland. My complete config looks like:

```
-Dide.managed.by.toolbox=Path to toolbox
-Dtoolbox.notification.token=Toobox token
-Dtoolbox.notification.portFile= Even more toolbox shit...
-Xmx6144m
-Dcom.jetbrains.use.wayland.toolkit=true
-XX:+UseG1GC
-Dsun.java2d.opengl=true
```

To make sure everything is working as expected you can check your settings in `Help -> About` and close and copy (yes, weird I know...) and paste it into a texteditor and check that the settings are there. If you have any other tips for optimizing Rider on Linux please let me know!  
