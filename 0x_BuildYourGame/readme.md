# Building Your Game in Unreal Engine 5

## What a Build Actually Is

When you press Play in the editor you are running the game inside a sandboxed environment that has access to all of Unreal's editor tools, uncooked assets, and debug systems. A packaged build is different. It compiles all your assets into an optimised format, strips out everything the editor needs but the game does not, and produces a standalone executable that runs without Unreal installed.

This is what you ship. This is what you send to a playtester. This is what you submit for grading.

---

## Before You Build

Three things to check before you touch the packaging settings.

**1. The game runs cleanly in the editor.**
If it crashes in the editor it will crash in the build. Fix all known issues first. A build takes time and there is no point waiting ten minutes for a broken game.

**2. The default map is set.**
Go to **Edit > Project Settings > Maps and Modes**. Check that **Game Default Map** and **Editor Startup Map** both point to your actual level. If this is not set, the packaged game opens an empty black screen.

**3. You have enough disk space.**
A packaged UE5 game is large. Even a small project can be 2-4 GB after packaging because it includes the engine's base shaders and content. Make sure the target drive has at least 10 GB free.

---

## Setting Up Packaging

Go to **Edit > Project Settings > Packaging**.

The settings that matter for a assignment project:

**Build Configuration:** Set to `Development` for a build you want to test and debug. Set to `Shipping` for a final submission. Development builds include some debug information and are slightly larger. Shipping builds are optimised and have no debug overhead.

**List of Maps to Include:** Expand this and add your level explicitly. If you leave it empty Unreal tries to include all maps it can find, which sometimes picks up test levels or the default template maps you do not want.

**Use Pak File:** Leave this on. It bundles all your assets into a single `.pak` archive which loads faster than thousands of loose files.

---

## Running the Build

Go to **Platforms > Package Project** in the top toolbar (or **File > Package Project** in older versions).

Select:
* Package for, Windows, Mac or other platform
* Build Configuration set to project settings or custom config
* And then **Package Project** to build 

A file picker opens asking where to save the build. Create a folder called `Build` somewhere outside your project directory. Do not save it inside the project folder, Unreal will try to cook it as a content asset.

Click **Select Folder**. The Output Log at the bottom of the editor starts streaming. The first build takes the longest because shaders need to compile. Subsequent builds are faster.

When it finishes you will see `BUILD SUCCESSFUL` in the Output Log unless something went wrong, if so, check the output log to diagnose and find any errors that caused the build to fail!

---

## The build

Inside the folder you chose you will find the **.exe** file along with the engine folder and project content and some manifest text files. 

Double-click `EscapeRoomLab.exe` to run the game.

No Unreal installation is needed on the target machine, but it does need the Visual C++ redistributables which are included in the build.