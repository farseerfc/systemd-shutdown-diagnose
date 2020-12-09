# systemd-shutdown-diagnose
Take from https://co-op.space/2016/03/10/systemd-guan-ji-chao-shi-wen-ti-ding-wei-fang-fa/ ,
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

After restart, there will be a log file at `/var/log/shutdown.log`
that records the sequence of all process shutdown in ftrace format.
To make it more readable, parse the log by:
```
analyze-shutdown </var/log/shutdown.log
```

The output will be a list of all process, their name and PID, the time when they shutdown, and the signals they recieved.

# Example of output

```
start-diagnose-(1718) exited at          0s, got signals: 
systemd-cgroups(1825) exited at   0.002294s, got signals: 
systemd-cgroups(1826) exited at   0.002307s, got signals: 
QDBusConnection( 776) exited at  83.975056s, got signals: 
        QThread( 780) exited at  83.975110s, got signals: 
kactivitymanage( 764) exited at  83.975163s, got signals: 
        QThread( 779) exited at  83.975575s, got signals: 
        QThread( 778) exited at  83.978465s, got signals: 
systemd-cgroups(1827) exited at  83.983387s, got signals: 
 systemd-logind( 468) exited at  83.984484s, got signals:  9(83.984228)
systemd-user-se(1828) exited at  84.000099s, got signals: 
systemd-cgroups(1829) exited at  84.004714s, got signals: 
         umount(1861) exited at  84.217824s, got signals: 
...
        systemd(1878) exited at  84.240749s, got signals: 
systemd-cgroups(1877) exited at  84.266284s, got signals: 
  systemd-udevd( 241) exited at  84.269505s, got signals:  19(84.267333)
  kworker/u16:7(1879) exited at  84.287657s, got signals: 
systemd-journal( 196) exited at  84.290971s, got signals:  19(84.267336)
  kworker/u16:7(1880) exited at  84.291027s, got signals: 
  kworker/u16:7(1881) exited at  84.292804s, got signals: 
          mount(1885) exited at  84.427401s, got signals: 
```

We can see from the output here, QDBusConnection (might be a thread owned by kactivitymanager) is the one to be blamed.

# How it works

The details are documented in the original blog post in Chinese:
https://co-op.space/2016/03/10/systemd-guan-ji-chao-shi-wen-ti-ding-wei-fang-fa/

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
cat /sys/kernel/debug/tracing/trace >/var/log/shutdown.log
mount -o remount,ro /
```
We remount the root filesystem as read-write, write the ftrace log into it, and remount it read-only.

Lastly, we have an `analyze-shutdown` script to extract the sequence of processes from ftrace log.
