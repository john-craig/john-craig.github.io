---
title: "Tutorial: Multi-Room Bluetooth Jukebox"
feature_image: "https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fi.pinimg.com%2Foriginals%2Fa9%2Fc8%2F7e%2Fa9c87e53a86bd7e70f0afec46f68a8c5.jpg&f=1&nofb=1&ipt=8c96772708e335b26704ceaa9f169db5c48cb6a137d40420a3e17e190b906dfd&ipo=images"
--- 

### Introduction

I am at the point where I almost cannot shower without listening to music or a podcast. However, this has historically presented a challenge for me, because it meant that upon getting out of the shower, I would have to pick up the speaker I was using to play music and carry it around with me as I got dressed, or else lose my precious tunes.

One solution to this would be to purchase multiple bluetooth speakers and place them in different rooms. Unfortunately, the bluetooth signal on my phone isn't quite strong enough to remain connected to a speaker in my kitchen while I'm in the bathroom, and vice-versa. Plus, it's a bit of a hassle to manually turn on all of the speakers I wanted to use, adding yet another step to my pre-shower ritual.

What I really needed with a bluetooth jukebox, a host that would be in a stationary and ideally central location, and maintain a constant connection to all of the speakers that I wanted to use. Then I could connect to the jukebox with my phone and use it to broadcast the music I wanted to play. The range issue could be solved with a sufficiently powerful plug-and-play bluetooth adapter.

### Tutorial

**Prerequisites**

For my bluetooth jukebox, I chose a Beelink Mini PC. I used NixOS for the operating system, and installed `pipewire` and `wireplumber` to use for audio, for their good bluetooth support.

**Finding the Bluetooth Adapter Address**

To get started, I needed to collect some information about the available bluetooth adapters, namely the MAC address of each one.

The simple way to accomplish this is to issue the `bluetoothctl list` command, to list the current bluetooth adapters on the system, then unplug the plug-and-play adapter, and re-issue the command to see which of them sticks around.

If, for some reason, you can't or don't want to unplug the plug-and-play adapter, you can try the following method instead:

1. Issue `lsusb` , which will display a list of the bluetooth devices connect to your system, along with their bus and device numbers. For example,

```
...
Bus 001 Device 003: ID 0bda:a729 Realtek Semiconductor Corp. Bluetooth 5.3 Radio
...
```

2. Issue `ls -lA /sys/class/bluetooth` , which will give you information about the system's HCI devices. The path to which each HCI device is symlinked will correspond to the bus and device number of its USB device. In the following example, I know `hci0` corresponds to the same USB device displayed above because the `.../usb1/1-3/1-3:1.0/...` in its path have the same Bus and Device numbers.

```
...
hci0 -> ../../devices/pci0000:00/0000:00:15.0/usb1/1-3/1-3:1.0/bluetooth/hci0
...
```

3. Issue `hciconfig -a` to display information about each of the HCI devices. In these results, the bluetooth MAC address are displayed as `BD Address`, for example:

```
hci0:   Type: Primary  Bus: USB
        BD Address: 8C:88:4B:45:CC:11  ACL MTU: 1021:6  SCO MTU: 255:12
        UP RUNNING PSCAN
        RX bytes:244898176 acl:14952 sco:3194773 events:639409 errors:0
        TX bytes:640454708 acl:638153 sco:3191892 commands:580 errors:0
        Features: 0xff 0xff 0xff 0xfe 0xdb 0xfd 0x7b 0x87
        ...
```

Going forwards, we will assume that the MAC address of your plug-and-play adapter is `FF:FF:FF:FF:FF:FF`, and the MAC address of your device's built-in adapter is `AA:AA:AA:AA:AA:AA`.

**Connecting Bluetooth Devices**

Next, we will connect the devices to which you wish to play audio from your jukebox. [This](https://superuser.com/questions/931188/how-many-devices-can-be-hooked-up-to-one-pc-bluetooth-adapter-at-a-time#935508) StackOverflow question gives a good explanation of the limits of the number of devices which can be connected to a single bluetooth adapter.

In order to do this, we want to perform the following steps:

```bash
bluetoothctl
select FF:FF:FF:FF:FF:FF
scan on 
```

You should begin seeing nearby bluetooth devices pop up, along with their associated MAC addresses. Let's suppose you have a device with a MAC address of `11:11:11:11:11:11` which you have found and you want to connect to.

```
connect 11:11:11:11:11:11
pair 11:11:11:11:11:11
trust 11:11:11:11:11:11
```

Depending upon the exact device, you may need to perform some special steps in order to pair it, such as entering a passcode or accepting the pair request on the device itself.

Once you have all your speakers connected, your next step is to connect your phone. When connecting your phone, you will want to connect using the internal bluetooth adapter.\* Supposing your phone's MAC address is `99:99:99:99:99:99`, you would perform the following steps:

```
bluetoothctl
select AA:AA:AA:AA:AA:AA
scan on
...
connect 99:99:99:99:99:99
pair 99:99:99:99:99:99
trust 99:99:99:99:99:99
```

\*The reason for this is because, in my experimenting, I found that when connecting devices that will function as both sources and sinks for audio, the quality drops significantly for whichever type of devices was connected second. I suspect this is because the adapter prioritizes the profile of the first device which is connected.

It's possible that this could be fixed by setting `MultiProfile = multiple` in `/etc/bluetooth/main.conf`, but I haven't tried that myself.

**Configuring Pipewire and Wireplumber**

To set up multi-room playback on our jukebox, we first want to created a [combined sink](https://docs.pipewire.org/page_module_combine_stream.html) in Pipewire. This is a sink that will stream audio to all other available sinks. In `/etc/pipewire/pipewire.conf.d`, create a file named `50-combined-sink.conf` with the following contents:

```
context.modules = [
        {
          name = libpipewire-module-combine-stream
          args = {
            combine.mode = sink
            node.name = "broadcast-sink"
            node.description = "broadcast-sink"
            -- combine.latency-compensate = true   # if true, match latencies by adding delays
            combine.props = {
              audio.position = [ MONO ]
            }
            stream.props = {
            }
            stream.rules = [
              {
                matches = [ { media.class = "Audio/Sink" } ]
                actions = { create-stream = { } }
              }
            ]
          }
        } 
      ]
```

After restarting Pipewire with `systemctl --user restart pipewire.service`, you should see a new sink created when you issue `wpctl status` :

```
Audio
 ├─ Devices:
 │      ...
 │
 ├─ Sinks:
 │      40. broadcast-sink                      [vol: 1.00]
 │      ...
 │
 ├─ Sink endpoints:
 │
 ├─ Sources:
 │      ...
 │
 ├─ Source endpoints:
 │
 └─ Streams:
        ...
```

Now we need to make sure that your phone connects to this sink as its source when you connect it to your jukebox. We can accomplish this by adding a new [BlueZ monitor rule](https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/bluetooth.html#rules) in Wireplumber.

To do this, go to the directory `/etc/wireplumber/bluetooth.lua.d/` (creating it if it does not exist), and then add a file named `51-bluez-config.lua` with following contents:

```
      bluez_monitor.enabled = true

      bluez_monitor.properties = {
        ["bluez5.enable-sbc-xq"] = true,
        ["bluez5.enable-msbc"] = true,
        ["bluez5.enable-hw-volume"] = true,
        ["bluez5.codecs"] = "[sbc sbc_xq]",
        ["bluez5.headset-roles"] = "[ hsp_hs hsp_ag hfp_hf hfp_ag ]"
      }

      bluez_monitor.rules = {
        {
          matches = {
            {
              -- This matches all cards.
              { "device.name", "matches", "bluez_card.*" },
            },
          },
          apply_properties = {
            ["bluez5.auto-connect"]  = "[ a2dp_sink a2dp_source hfp_hf hsp_hs ]",
          }
        },
        {
          matches = {
            {
              -- Pixel 4a 5G
              { "node.name", "matches", "bluez_input.99_99_99_99_99_99.2" },
            },
          },
          apply_properties = {
            ["target.object"] = "broadcast-sink",
          }
        },
      }
```

This configuration has three pieces:

* `bluez_monitor.properties` is a section that sets some useful default settings for all bluetooth sources and sinks, such and higher-quality profiles and codecs
* the first rule under `bluez_monitor.rules` applies to all bluetooth devices, and provides a set of profiles which Wireplumber will attempt to automatically connect. This will make it so your jukebox will reconnect to your bluetooth speakers if they're turned on/off, and will attempt to reconnect to them if it's rebooted
* the second rule under `bluez_monitor.rules` applies to only your phone, and makes it so that it connects to the broadcast sink we added in the previous step

Once this is complete, you should be able to restart Wireplumber with `systemctl --user restart wireplumber`, and then play audio from your phone and hear it in high quality from all your connected devices.