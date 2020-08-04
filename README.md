# securepapermc
This is a simple guide for setting up a PaperMC Minecraft domain to proxy through Google Cloud and TCPShield.

## Intro

This is a very simple, plugin agnostic, guide for automating PaperMC updates and restarts on Windows 10. The goal is to create a sequence that, upon startup, achieves the following:

- Automatic upate & start of the server on boot
- Automatic save & stop of the server
- Timed restart of Windows 10

This guide is set up for Minecraft version 1.16.1, but can be easily altered for past or future versions.

The configuration below calls for automatic login of the Windows 10 account. Depending on your use case, this may not be desirable and alternative settings for Task Scheduler can be set to run without sign in.

This guide assumes you already have a PaperMC server up and running. [Here](https://www.youtube.com/watch?v=st8F2MPyHKk) is an easy to follow video on how to get one going for 1.16.1.

**NOTE: Make sure that every batch/script file we create here is located in the same folder as paperclip.jar.**

___

## Batch & VBScript Commands

### Server Startup

In order to start the server, I create `paperclip.bat` with the following arguments:

```
TITLE papermc
java -Xms10G -Xmx10G -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:G1NewSizePercent=30 -XX:G1MaxNewSizePercent=40 -XX:G1HeapRegionSize=8M -XX:G1ReservePercent=20 -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 -XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 -XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem -XX:MaxTenuringThreshold=1 -Dusing.aikars.flags=https://mcflags.emc.gs -Daikars.new.flags=true -jar paperclip.jar nogui
```
`TITLE papermc` simply renames the command prompt window to `papermc`. This will be useful later.

The arguments come from [Aikar's JVM Tuning Guide](https://aikar.co/2018/07/02/tuning-the-jvm-g1gc-garbage-collector-flags-for-minecraft/). Make sure to check to see that you are using the most up to date JVM arguments from here, and ***DO NOT*** just copy and paste this in without reading the linked guide.

### Server Stop

Normally to stop the server you would have to manually type in `stop` for the server to save and close safely. Simply allowing the PC to restart could cause loss of data. Create the file `autoStop.vbs` and use the following code.

```
Set WshShell = WScript.CreateObject("WScript.shell")
WshShell.AppActivate "papermc"
WshShell.SendKeys "ATTN: THE SERVER WILL BEGIN RESTART IN 1 MINUTE"
WshShell.SendKeys "{ENTER}"
WScript.Sleep 60000 'Sleeps for 60 seconds
WshShell.SendKeys "stop"
WshShell.SendKeys "{ENTER}"
```
This command very simply selects the open server command window (that we named `papermc`) and issues the `stop` command that we usually would have to manually type in. I have it set to issue a warning that the server will restart after 60 seconds. This can be adjusted to your preference, or removed altogether.

Now in order for this code to activate nicely in Task Manager, we will run it from a very simple batch file you can call `autoStop.bat`. Here is that command:

```
@echo off
autoStop.vbs
```

**NOTE: MAKE SURE YOU USE THIS BATCH FILE, NOT THE VBS FILE, IN THE TASK SCHEDULER SECTION BELOW**

### Server Update

We will be using `wget` to pull the most recent server version of PaperMC. For use in Windows, `wget` must be downloaded from [here](https://eternallybored.org/misc/wget/). Ensure that `wget` is added to your Windows path.

Create this batch file for updating. I named mine `mc_server_dl.bat` but feel free to name it whatever you'd like.

```
@echo off
cd <PATH-TO-SERVER-FOLDER>
del paperclip.jar
wget https://papermc.io/api/v1/paper/1.16.1/latest/download/ -O paperclip.jar
```
This command will remove the current server version and download the most recent version using the PaperMC download API. It then renames the file `paperclip.jar` so that the above `paperclip.bat` file will work. Obviously we are using 1.16.1 here, but the API link can be updated for future versions.

___

## Windows 10 Task Scheduler Setup

Now that we have all of the server related commands ready to go, we can automate the starting and stopping of these tasks through Task Scheduler. This guide will configure the server to stop daily at 5:59 AM, restart the computer at 6:01 AM, and have the server update then start back up. These times are specific to my preferences and the sleep command in the Server Stop script. So feel free to set the times/days up however you'd like.

**NOTE: These scripts will only work if you have the user set to login automatically after startup** This is to simplify the commands and is up to preference. You can absolutely run these whether or not the user is logged in. The Task Scheduler configuration can be easily modified in the General tab.

So now open up Task Manager, and select "Create Task..." in the Actions panel on the right side. Below are lists detailing what to enter in each tab when creating the task.

___

### Server Download

#### General

- Name: whatever-you-want
- Description: whatever-you-want
- Security Options: Select "Run only when user is logged on"
- Configure for: Windows 10

#### Triggers

- Click "New..."
- Begin the Task: "At log on"
- Settings: Any user
- Advanced Settings: Enabled

#### Actions

- Action: Start a program
- Settings: In Program/script enter the absolute path to `mc_server_dl.bat` in quotations. Then enter the path to the folder `mc_server_dl.bat` is in **WITHOUT QUOTES**

#### Conditions

- Leave everything here as default.

#### Settings

- If the task is already running, set: "Do not start a new instance"

___

### Server Startup

- Give this task a unique name, and then configure everything the exact same way as Server Download, except in Actions where you define where the `paperclip.bat` server startup file is located.

___

### Stop Server

- Give this task a unique name, define `autoStop.bat` as the startup file in Actions, and then configure everything the exact same way as Server Download except for these options:

#### Triggers

- Click "New..."
- Begin the Task: "On a schedule"
- Settings: Daily, Start on whatever date/time you want, Recur every: 1 days (or however often you want to update the server and restart the PC)
- Advanced Settings: Enabled

#### Settings

- If the task is already running, Stop the exisiting instance.

___

### Restart PC

- Give this task a unique name, and then configure everything the exact same way as Server Download except for these options:

#### Triggers

- Click "New..."
- Begin the Task: "On a schedule"
- Settings: Daily, Start on whatever date/time you want AFTER the stop server command. I have mine set to 2 minutes after so the sleep and stop command have plenty of time to run. Recur every: 1 days (or however often you want to update the server and restart the PC)
- Advanced Settings: Enabled

#### Actions

- Action: Start a program
- Settings: In "Program/script:" type in `shutdown`. Then below in "Add arguments (optional):" type in `/r /f /t 0`. The arugments tell the PC to restart, force any applications to close, and time=0 so it will restart immediately. 

#### Settings

- If the task is already running, Stop the exisiting instance.

___

### Conclusion

And that's it! At this point you should have a server that achieves the tasks we identified in the intro. If you have any questions/issues with the guide please post them in [my github](https://github.com/megacorvega/autopapermc/issues). If common issues arise, I will add an FAQ with those addressed. Enjoy!
