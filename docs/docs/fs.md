# logagg-fs
[Fuse file system](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)  for`logagg-collector`. Captures logs when it is written to a file and caches them until the `logagg-collector` collects and processes the contents.

## Features
* Guarantees capturing every log line
* Rotation proof
* One time set-up
* Supports file patterns; i.e. `/var/log/syslog*`; rather than fpaths

## Limitations
* No way of getting logs from files before start-up of the program
* Requires a reboot of the machine after set-up is done

## Components/Architecture/Terminology
* **mountpoint**: path to the directory where logs are being written (e.g. /var/log).
* **logcache**: path to the directory where the logagg-fs program stores all of it's data.
* **logcache/mirror**: directory inside logcache path which is mounted to the `mountpoint` directory path. If `logcache` path is '/logcache' and the `mountpoint` is '/var/log', then the directory '/logcache/mirror' is mounted on to '/var/log'.
* **logcache/trackfiles.txt**: file inside logcache directory where file-patterns are mentioned that need to be tracked by logagg-fs (eg. '/var/log/syslog')
* **logcache/logs**: path to directory where log-files that are cached temprorarily until processed and deleted.

## Prerequisites
* Python => 3.6
* Expected restart of server after mounting to non-empty directories like /var/log/

## Installation
### Dependencies
```
$ sudo apt install libfuse-dev python3-dev python3-pip pkg-config build-essential python3-pip
$ pip3 install setuptools
```

### Install logagg-fs
- **NOTE:** Make sure you are a root user.

#### Root user
```
$ pip3 install git+https://github.com/deepcompute/logagg-collector.git
```

#### Check installation
```
$ logagg-fs --version
logagg-fs 0.3.1
logagg-fs 0.3.1
```

### Setting up  logagg-fs for mounting /logcache/mirror to /var/log
#### Make a directory so that logagg-fs can use it as `logcache`
```
# mkdir /logcache/
```

#### Write configuration to mount /logcache/mirror to /var/log/ directory in `fstab`
```
# vim /etc/fstab
# Add the following line to /etc/fstab: "logagg-fs /var/log/ fuse rw,user,auto,exec,nonempty,allow_other,root=/logcache/,loglevel=INFO,logfile=/logcache/fuse.log 0 0"
```
###### Command breakdown:
* `logagg-fs`: the path to logagg-fs program
* `/var/log/`: the mountpoint
* `root=/logcache/`: the logcache directory for logagg-fs
* `logfile=/logcache/fuse.log`: path where logagg-fs is supposed to store own logs

![image](https://user-images.githubusercontent.com/33823698/45282589-fd569880-b4f8-11e8-99e4-0207d2bbbf9f.png)
#### Setting up logrotate for the log file of logagg-fs (Optional)

Create configuration file of logrotate
```
$ vim /etc/logrotate.d/logagg-fs
```
Write the following lines in the file
```
/logcache/fuse.log {
weekly
rotate 3
size 10M
compress
delaycompress
}
```
#### Run & Reboot to load the configuration in /etc/fstab

- **IMPORTANT:** Copy files all inside mountpoint directory to a temprorary location.
```
# mkdir ~/temp_logs && cp -R /var/log/* ~/temp_logs/
```
Mount logagg-fs from fstab configuration
```
# mount /var/log/
```
Copy back files to mountpoint directory
```
# cp -R ~/temp_logs/log/* /var/log/
```
Reboot to make changes to take effect and running programs to use the mountpoint as storage location for logs
```
# reboot
```
## Usage
Check if '/logcache/mirror' is mounted properly to '/var/log'
```
# ls /var/log/
# # The same as:
# ls /logcache/mirror/
```

```
# cat /logcache/mirror/test
# cat: /logcache/mirror/test: No such file or directory
# echo "testing.." > /var/log/test
# cat /logcache/mirror/test
testing..
```

Check caching of log files
```
# ls /logcache/logs/ # No logs yet
# # Now add the files to be tracked in logcache/trackfiles.txt file
# echo "/var/log/syslog" >> /logcache/trackfiles.txt
# # Takes atmost 10sec to update state
# ls /logcache/logs/ # To see the cached log-files
f5fdf6ea0ea92860c6a6b2b354bfcbbc.1536590719.4519932.log
# tail -f /logcache/logs/* # The contents of the file are being written simultaneously to cached files
```
* Just remove the file pattern from `/logcache/trackfiles.txt` to stop caching of logs

* To unmount directory
```
# umount /var/log
```
Or Delete configuration from /etc/fstab
```
# reboot
```

