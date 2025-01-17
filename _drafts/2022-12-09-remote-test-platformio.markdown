---
layout: post
title: Setting up Remote Testing for PlatformIO
date: 2022-12-09 12:00:00 -400
---

# Goal

Lets set up a mac-mini to run as a remote test server for our PlatformIO project.

I love to have continuous testing. Working with embedded systems, though, it's not as simple as hooking up your favorite cloud CI/CD provider.
In most cases, you'll need an actual device hooked up to your build server to actually run your tests on.

## Initial Setup

We will build on a Mac Mini --  we call this the server or remote.

We will run our tests on an ESP32 which is attached to the Mac Mini.

I will submit code and tests from my laptop -- we call this the client or local.

# High Level

PlatformIO has a remote development solution built in.
This handles communication between computers, sync'ing of code, and running of remote commands.

# Steps

### 1. PlatformIO and Account

* You need to have [PlatformIO CLI](https://docs.platformio.org/en/stable/core/index.html) installed on both machines.
* You need an account for PlatformIO to broker communications. I signed up via [command line](https://docs.platformio.org/en/stable/core/userguide/account/index.html).
* [Generate](https://docs.platformio.org/en/stable/core/userguide/account/cmd_token.html) a Personal Authentication Token and use below.
* Have your code building and running on both your local machine, and the server.

(You technically don't have to get the code building on the server if everything goes peachy, but while troubleshooting I found it invaluable to be able
to switch between client and server to diagnose issues.)

### 2. Launch Script

On the server, create a script which will run the remote agent. We're going to run our server as a [LaunchDaemon](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html),
so we need to make sure the `platformio` is available. By default, Launchdaemon's have a pretty limited path, just `/usr/bin:/bin:/usr/sbin:/sbin`.
I installed PlatformIO via homebrew, so my executable is located in `/opt/homebrew/bin` which I had to make sure was on the executable path and
I also provided my auth token from above.

```bash
#! /bin/bash
# /Users/USERNAME/run_pio_server.sh

export PATH=$PATH:/opt/homebrew/bin
PLATFORMIO_AUTH_TOKEN=YOUR_TOKEN /opt/homebrew/bin/pio remote agent start --name YOUR_SERVER_NAME
```

### 3. Set up LaunchD

In order to have this run, I needed to create a launchdaemon. Here is the `.plist` file I went with. Put this in a file called `ag.lumo.platformio.plist`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ag.lumo.platformio</string>

  <key>ProgramArguments</key>
  <array>
  <string>/Users/USERNAME/run_pio_server.sh</string>
  </array>

  <key>UserName</key>
  <string>USERNAME</string>

  <key>KeepAlive</key>
  <true/>

  <key>StandardErrorPath</key>
  <string>/Users/USERNAME/pio-build.err</string>

  <key>StandardOutPath</key>
  <string>/Users/USERNAME/pio-build.out</string>
</dict>
</plist>
```

• The `Label` `ag.lumo.platformio` is used by `launchctl` to label our process in the future. It's a strong convention for this to share the same name as the file.

* The `ProgramArguments` key points to the script from the previous step. So all we are doing is running our script with this daemon. `UserName` indicates to run this process as a given usernam. `
* `KeepAlive` indicates this should run at startup and be restarted if it fails
* `StandardOutPath` and `StandardErrorPath` will direct log output, mostly for troubleshooting.

I moved this file to `/Library/LaunchDaemons/ag.lumo.platformio.plist` and then I had to do the following. OS X security requires narrow permissions on the system daemons.

```bash
mv ag.lumo.platformio.plist /Library/LaunchDaemons/ag.lumo.platformio.plist
sudo chown root /Library/LaunchDaemons/ag.lumo.platformio.plist
sudo chgrp wheel /Library/LaunchDaemons/ag.lumo.platformio.plist
sudo chmod a-w /Library/LaunchDaemons/ag.lumo.platformio.plist
# Then I could register with
launchctl start /Library/LaunchDaemons/ag.lumo.platformio.plist
```

I restarted, and I could see the pio process running using `ps aux | grep pio`

### 4. Run the tests

On the client now, it was much easier. From my environment, I just had to log in and then look for my remote server.

```bash
# log in with 
pio account login
```

I called my server `lumo-build`

```bash
# look for our remote server with
pio remote agent list
lumo-build
```

And then I could run my tests with

```bash
pio remote --egent lumo-build test -e V5
```

Success!

#### Helpful references

* [PIO Core](https://docs.platformio.org/en/stable/core/index.html)
* [PIO Remote Agent](https://docs.platformio.org/en/stable/core/userguide/remote/cmd_agent.html)
* [LaunchDaemons](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
* [LaunchD blog](https://thoughtbot.com/blog/example-writing-a-launch-agent-for-apples-launchd)
* [LuanchD reference](https://www.real-world-systems.com/docs/launchdPlist.1.html)
* [LaunchD example](https://alvinalexander.com/mac-os-x/launchd-examples-launchd-plist-file-examples-mac/)
