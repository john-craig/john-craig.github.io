---
title: "HP Smart Array S.M.A.R.T. Prometheus Exporter"
---
# Project Spotlight: HP Smart Array S.M.A.R.T. Prometheus Exporter
### Introduction
Like any homelabber, I love having Grafana dashboards displaying metrics about my hardware. Uptime, RAM utilization, CPU load, you name it. These metrics are usually displayed by querying a time-series data model called [Prometheus](https://prometheus.io/docs/introduction/overview/), which itself aggregrates metrics from a variety of different sources. 

One common type of Prometheus source is called an *exporter*, a program which either observes the activity of a system or application and surfaces that activity over an HTTP server in a standardized format easily consumable by Prometheus. For example, the exporter used to collect metrics from a host computer is called the [node_exporter](https://github.com/prometheus/node_exporter).

Some of these metrics are very important to keep an eye on: for example, knowing if a machine is too hot based on its current CPU temperature, or if it's about to run out of storage based on its disk utilization. When it comes to hard drives, it's also very important to know if a drive is close to failure, so you can make preparations to replace it.

### S.M.A.R.T. Tests
[Self-monitoring, Analysis and Reporting Technology System (S.M.A.R.T.)](https://www.smartmontools.org/) is a protocol available on most modern hard drives. It can be used to run tests against disks that will show signs of degradation or impending failure, as well as collecting information about the disk's lifetime usage of bytes read, total operational hours, and so on.

These are a strong, but not guaranteed, early warning sign of drive failure, and so naturally there is a Prometheus exporter which is used to surface S.M.A.R.T. data as metrics: the [smartctl_exporter](https://github.com/prometheus-community/smartctl_exporter), written in Go. This is a very effective tool when targeting disks that are directly accessible to the operating system. Where it falls short, however, is in hardware RAID arrays.

### HP Smart Array
See, now you understand why I used periods in the initialization of S.M.A.R.T. It's so you won't get it confused with Smart Arrays. :)

[Hewlett Package Smart Array Controllers](https://support.hpe.com/connect/s/product?language=en_US&kmpmoid=3883890&tab=manuals) are a type of hardware RAID array, usually found in HP ProLiant servers. I won't go too far into the weeds of RAID arrays here, so suffice to say that when using a RAID array, the individual disks of the array are opaque to the operating system, appearing instead as a single very large physical disk.

These individual disks can be interacted with using a special set of utilities-- in this case, a CLI program called [`hpssacli`](https://support.hpe.com/connect/s/softwaredetails?language=en_US&softwareId=MTX_05d4c11e7ed3433e85c89ea604). `smartctl`, the CLI program used to run S.M.A.R.T. tests manually, also has support for running tests against an array's individual disks, so long as the correct drivers are installed and the command is formulated correctly. 

### The Problem
The existing `smartctl_exporter`, which is the program that collects the output of a test run by `smartctl` and exposes it as Prometheus metrics, does *not* support running tests against individual disks. This is really no fault of the exporter itself. Recall that I mentioned that although `smartctl` has the ability to run such tests, it [requires a special format of command](https://www.reddit.com/r/sysadmin/comments/f7ub4x/checking_smart_status_of_drives_in_raid/). Here's what that looks like:
```
# Against a normal disk
smartctl -d /dev/sda

# Against the first disk in a RAID array:
smartctl -d ciss,0 /dev/sg0
```

Notice the second command takes a path to a device, as well as this `ciss,0` piece. The `0` is the index of the physical disk in the RAID array, while the `/dev/sg0` path is a way of referencing the RAID array logically using the [SCSI Generic driver](https://sg.danny.cz/sg/). To my knowledge, there is not a way of generically determining how many physical disks are located inside of a hardware RAID array controller. Some controllers have eight slots, some have only four, some have sixteen, and so on and so on. The number of slots can usually only be identified using a utility specific to that hardware controller.

This makes the task of supporting S.M.A.R.T. metrics on hardware RAID arrays much more difficult for the `smartctl_exporter`. They would have to check if the bespoke CLI utility for each supported model of RAID array, then go into special-case logic to invoke that utility to determine the number of individual disks in that controller. Hypothetically they could do something like attempting to run the tests against a range of disks sequentially until the command fails, then cache the number for later, but this is still rather inelegant.

What might work better is an entirely separate exporter dedicated to the RAID array controller it is intended to collect the metrics for.

### A Partial Solution
Now admittedly, I am telling the story a bit backwards here. In my search for an easy way to get S.M.A.R.T. metrics from my HP Smart Array, I came across a repository called [smartctl_ssacli_exporter](https://github.com/jakubjastrabik/smartctl_ssacli_exporter). This looked like exactly what I wanted, but upon trying to build it, it failed to compile for want of a couple of small tweaks, so I forked the repository and fixed it up myself.

It was not until digging fairly deep into this repository that I realized it didn't export much in the way of S.M.A.R.T. metrics at all. There was some information being surfaced about the array's physical and logical disks that was collected from the `hpssacli` utility, but little in the way of S.M.A.R.T. data. In fact it was investigating this lack of metrics that lead me down the rabbit hole of fully understanding how `smartctl` interacts with a RAID array and why `smartctl_exporter` is unable to provide metrics for them.

### Let's go. In and out. 20 minute Go package update.
So my little fork of `smartctl_ssacli_exporter` became more or less a full rewrite. I kept some of the metrics surfaced from the `hpssacli` command (although I rewrote the parsing) then added logging, logic for determining how many RAID array controllers were on the system and logic for running S.M.A.R.T. tests against each disk on each array. The actual metrics surfaced about the S.M.A.R.T. tests I mostly tore straight out of the code for `smartctl_exporter`.

Initially I waffled about whether or not to just open a pull request against `smartctl_exporter` and add the functionality there instead, but decided that the dependency upon the `hpssacli` utility that my implementation would introduce meant that it was best left as a dedicated project.

### 