---
layout: page
title: "Foscam Recording during Away Mode Only using Pan/Tilt/Zoom Control and Motion Detection"
description: "Example of how to set Foscam to only have Motion Detection Recording while no one is home. When users are home the Foscam will indicate it is not recording by pointing down and away from users"
date: 2016-03-10 13:05
sidebar: true
comments: false
sharing: true
footer: true
ha_category: Automation Examples
---

This requires a [Foscam IP Camera](/components/camera.foscam/) camera with PTZ (Pan, Tilt, Zoom) and CGI functionality ([Source](http://www.ipcamcontrol.net/files/Foscam%20IPCamera%20CGI%20User%20Guide-V1.0.4.pdf)) 

Foscam Cameras can be controlled by Home Assistant through a number of CGI commands. 
The following outlines examples of the switch, services, and scripts required to move between 2 preset destinations while controlling motion detection, but many other options of movement are provided in the Foscam CGI User Guide linked above.

The `switch.foscam_motion` will control whether the motion detection is on or off. This switch supports `statecmd`, which checks the current state of motion detection.

```yaml
# Replace admin and password with an "Admin" priviledged Foscam user
# Replace ipaddress with the local IP address of your Foscam
switch:
 platform: command_line
 switches:
   #Switch for Foscam Motion Detection
   foscam_motion:
     oncmd: 'curl -k "https://ipaddress:443/cgi-bin/CGIProxy.fcgi?cmd=setMotionDetectConfig&isEnable=1&usr=admin&pwd=password"'
     offcmd: 'curl -k "https://ipaddress:443/cgi-bin/CGIProxy.fcgi?cmd=setMotionDetectConfig&isEnable=0&usr=admin&pwd=password"'
     statecmd: 'curl -k --silent "https://ipaddress:443/cgi-bin/CGIProxy.fcgi?cmd=getMotionDetectConfig&usr=admin&pwd=password" | grep -oP "(?<=isEnable>).*?(?=</isEnable>)"'
     value_template: '{{ value == "1" }}'
```
 
The service `shell_command.foscam_turn_off` sets the camera to point down and away to indicate it is not recording, and `shell_command.foscam_turn_on` sets the camera to point where I'd like to record. h of these services require preset points to be added to your camera. See source above for additional information.

```yaml
shell_command:
  #Created a preset point in Foscam Web Interface named Off which essentially points the camera down and away
  foscam_turn_off: 'curl -k "https://ipaddress:443/cgi-bin/CGIProxy.fcgi?cmd=ptzGotoPresetPoint&name=Off&usr=admin&pwd=password"'
  #Created a preset point in Foscam Web Interface named Main which points in the direction I would like to record
  foscam_turn_on: 'curl -k "https://ipaddress:443/cgi-bin/CGIProxy.fcgi?cmd=ptzGotoPresetPoint&name=Main&usr=admin&pwd=password"'
```

The `script.foscam_off` and `script.foscam_on` can be used to set the motion detection appropriately, and then move the camera. These scripts can be called as part of an automation with `device_tracker` triggers to set `home` and `not_home` modes for your Foscam and disable motion detection recording while `home`.

```yaml
script:
 foscam_off:
   sequence:
   - service: switch.turn_off
     data:
       entity_id: switch.foscam_motion
   - service: shell_command.foscam_turn_off
 foscam_on:
   sequence:
   - service: switch.turn_off
     data:
       entity_id: switch.foscam_motion
   - service: shell_command.foscam_turn_on
   - service: switch.turn_on
     data:
       entity_id: switch.foscam_motion
```

To automate Foscam being set to "on" (facing the correct way with motion sensor on), I used the following simple automation:

```yaml
automation:
  - alias: Set Foscam to Away Mode when I leave home
    trigger:
      platform: state
      entity_id: group.family
      from: 'home'
    action:
      service: script.foscam_on
  - alias: Set Foscam to Home Mode when I arrive Home
    trigger:
      platform: state
      entity_id: group.family
      to: 'home'
    action:
      service: script.foscam_off
```

Newer Foscam cameras (Ones that use the Ambarella chipset) use differnt CGI commands to enable/disable motion detection and recording functions. Main difference is that the SetMotionDetectConfig now has a trsiling 1 in the statement so it is now setMotionDetectConfig1.  Sample statement below.
```
foscam_record_on: 'curl -k "https://<IP address>:443/cgi-bin/CGIProxy.fcgi?cmd=setMotionDetectConfig1&isEnable=1&linkage=8&snapInterval=1&triggerInterval=0&isMovAlarmEnable=1&isPirAlarmEnable=0&schedule0=281474976710655&schedule1=281474976710655&schedule2=281474976710655&schedule3=281474976710655&schedule4=281474976710655&schedule5=281474976710655&schedule6=281474976710655&x1=0&y1=0&width1=10000&height1=10000&threshold1=4&sensitivity1=2&valid1=0&x2=0&y2=0&width2=10000&height2=10000&threshold2=6&sensitivity2=4&valid2=0&x3=0&y3=0&width3=10000&height3=10000&threshold3=4&sensitivity3=0&valid3=1&usr=admin&pwd=<password>"'
```
For automation I have the camera pan to the set point and wait for 60 seconds before enabling the recording function.  If enabled at the on set the pan is detected as motion and recording is produced.  The below script references a lot of the code referenced above.
```
 foscam_on:
   sequence:
   - execute_service: switch.turn_off
     service_data:
       entity_id: switch.foscam_motion
   - service: shell_command.foscam_turn_on
   - delay:
       seconds: 60
   - execute_service: switch.turn_on
     service_data:
       entity_id: switch.foscam_motion
   - service: shell_command.foscam_record_on
   ```
   
