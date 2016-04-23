# systemd-shutdown-diagnose
Take from https://co-op.space/systemd-guan-ji-chao-shi-wen-ti-ding-wei-fang-fa/ ,
help to diagnose why systemd cannot shutdown properly.

***This may break your system! Use with CAUTION!***

# How to use

Build the package by:
```
makepkg
```

Install the package using `pacman -U`.

Enable the service by:
```
systemctl enable shutdown-diagnose.service
systemctl start shutdown-diagnose.service
```

Then restart your system.

After restart, there will be a log file at `/shutdown.log`
that records the sequence of all process shudown in ftrace format.
To make it more readable, parse the log by:
```
analyze-shutdown </shutdown.log
```

The output will be a list of all process, their name and PID, the time when they shutdown, and the signals they recieved.

# How it works

The details are documented in the original blog post in Chinese:
https://co-op.space/systemd-guan-ji-chao-shi-wen-ti-ding-wei-fang-fa/

The idea is to use ftrace to collect all needed infomation and record it in a log file.
To do this, we need to start ftrace just after `systemd shutdown|reboot` and before every process actually shutdown. So we have a `shutdown-diagnose.service` to be fired at the right timing.

The trick here is to combine `RemainAfterExit` and `ExecStop` on a `idle` service:
```
[Service]
Type=idle
RemainAfterExit=yes
ExecStop=/usr/bin/start-diagnose-shutdown
```

By doing this, the service is considered `Active` after the system boot up entering idle state, and execute `start-diagnose-shutdown` when systemd let it stop.
Because this service is usually the last one to start on a system, it will be stopped firstly by systemd.
Then we can ensure that the `start-diagnose-shutdown` is executated early enough. (Services that stopped before our service is ignorable here as they are not causing problems).
In `start-diagnose-shutdown` we turn on ftrace to collect process kill events and signal events:
```
echo syscalls:sys_enter_exit >set_event
echo syscalls:sys_enter_kill >>set_event
echo syscalls:sys_enter_tkill >>set_event
echo syscalls:sys_enter_tgkill >>set_event
echo signal:signal_deliver >>set_event
echo sched:sched_process_exit >>set_event

echo global >trace_clock
echo 40960 >buffer_size_kb
echo nop >current_tracer
echo 1 >tracing_on
```

Then we need to write out the collected ftrace log into a file right after all service are stoped. `systemd` kindly provided hooks to run just before the system goes shutdown, so we have our `/usr/lib/systemd/system-shutdown/diagnose.shutdown` script:
```
#!/bin/sh
mount -o remount,rw /
cat /sys/kernel/debug/tracing/trace >/shutdown.log
mount -o remount,ro /
```
We remount the root filesystem as read-write, write the ftrace log into it, and remount it read-only.

Lastly, we have an `analyze-shutdown` script to extract the sequence of processes from ftrace log.
