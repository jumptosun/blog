---
layout: post
title:  "linux 时间设置"
date:   2019-09-30
categories: linux
---

linux 有系统时间和硬件时间之分。
时间设置通常使用两个命令 timedatectl 和 hwclock。

# 1. timedatectl

``` shell
[billy@billy-pc ~]$ timedatectl -h
timedatectl [OPTIONS...] COMMAND ...

Query or change system time and date settings.

  -h --help                Show this help message
     --version             Show package version
     --no-pager            Do not pipe output into a pager
     --no-ask-password     Do not prompt for password
  -H --host=[USER@]HOST    Operate on remote host
  -M --machine=CONTAINER   Operate on local container
     --adjust-system-clock Adjust system clock when changing local RTC mode
     --monitor             Monitor status of systemd-timesyncd
  -p --property=NAME       Show only properties by this name
  -a --all                 Show all properties, including empty ones
     --value               When showing properties, only print the value

Commands:
  status                   Show current time settings
  show                     Show properties of systemd-timedated
  set-time TIME            Set system time
  set-timezone ZONE        Set system time zone
  list-timezones           Show known time zones
  set-local-rtc BOOL       Control whether RTC is in local time
  set-ntp BOOL             Enable or disable network time synchronization

systemd-timesyncd Commands:
  timesync-status          Show status of systemd-timesyncd
  show-timesync            Show properties of systemd-timesyncd

```

# 2. hwclock
``` shell
hwclock [function] [option...]

Time clocks utility.

功能：
 -r, --show           display the RTC time
     --get            display drift corrected RTC time
     --set            set the RTC according to --date
 -s, --hctosys        set the system time from the RTC
 -w, --systohc        set the RTC from the system time
     --systz          send timescale configurations to the kernel
 -a, --adjust         adjust the RTC to account for systematic drift
     --predict        predict the drifted RTC time according to --date

选项：
 -u, --utc            the RTC timescale is UTC
 -l, --localtime      the RTC timescale is Local
 -f, --rtc <file>     use an alternate file to /dev/rtc0
     --directisa      use the ISA bus instead of /dev/rtc0 access
     --date <time>    date/time input for --set and --predict
     --delay <sec>    delay used when set new RTC time
     --update-drift   update the RTC drift factor
     --noadjfile      do not use /etc/adjtime
     --adjfile <file> use an alternate file to /etc/adjtime
     --test           dry run; implies --verbose
 -v, --verbose        display more details

 -h, --help           显示此帮助
 -V, --version        显示版本

```

# 3. 设置
``` shell 

[billy@billy-pc ~]$ timedatectl set-ntp true && hwclock -w
[billy@billy-pc ~]$ timedatectl set-ntp false


```