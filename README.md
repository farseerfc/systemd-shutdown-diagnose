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

Then try to restart the system.

After restart, there will be a log filename at `/shutdown.log`
that records the sequence of all process shudown in ftrace format.
To make it more readable, parse the log by:
```
analyze-shutdown </shutdown.log
```

