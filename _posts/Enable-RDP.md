---
title: "How to enable RDP on macOS and Windows"
date: "2025-01-11"
author:
  name: Charles
  picture: "/assets/blog/authors/cc.png"
categories: 
  - "instruction"
coverImage: "/assets/blog/hello-world/cover.jpg"
ogImage:
  url: "/assets/blog/preview/cover.jpg"
---

# MacOS

## How to enable screen sharing on MacOS

1. Open System Preferences
2. Click on Security & Privacy
3. Click on Privacy
4. Click on Screen Recording
5. Add the app to the list

## Terminal solution

```
// Enable Screen Sharing via kickstart
"/System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate",
"/System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -configure -allowAccessFor -allUsers",
"/System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -configure -access -on",

// Enable via system preferences
"defaults write /var/db/launchd.db/com.apple.launchd/overrides.plist com.apple.screensharing -dict Disabled -bool false",
"defaults write com.apple.ScreenSharing dontWarnOnDirectConnect -bool true",

// Start the service
"launchctl load -w /System/Library/LaunchDaemons/com.apple.screensharing.plist || true"`
```

## How to enable screen sharing on MacOS via terminal

```
defaults write com.apple.ScreenSharing allowScreenRecording -bool true
```

## How to enable remote management for all users on MacOS via terminal ?

```
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
 -activate -configure -access -on \
 -configure -allowAccessFor -allUsers \
 -configure -restart -agent -privs -all
```

Note: This command will work, but the user can't connect to the screen sharing service unless we manually toggle the screen sharing service in the system preferences.

## How to disable remote management for all users on MacOS via terminal ?

```
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -deactivate -configure -access -off
```

## How to enable remote management for current users on MacOS via terminal ?

```
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart \
-activate -configure -access -on \
-clientopts -setvnclegacy -vnclegacy yes \
-clientopts -setvncpw -vncpw mypasswd \
-restart -agent -privs -all
```

## How to disable remote management for current users on MacOS via terminal ?

```
sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -deactivate -configure -access -off
```

Reference:

- https://apple.stackexchange.com/questions/30238/how-to-enable-os-x-screen-sharing-vnc-through-ssh
- http://ss64.com/osx/kickstart.html

## Windows enable remote desktop

```
:: Enable remote access.
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
:: allow through firewall.
netsh advfirewall firewall set rule group=\"remote desktop\" new enable=Yes

Start-Process -FilePath 'cmd.exe' -ArgumentList '/c reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f' -Verb RunAs
Start-Process -FilePath 'cmd.exe' -ArgumentList '/c netsh advfirewall firewall set rule group="remote desktop" new enable=Yes' -Verb RunAs
```
