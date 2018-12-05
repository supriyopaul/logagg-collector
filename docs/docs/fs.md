# logagg-fs
[Fuse file system](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)  for [logagg-collector](https://deep-compute.github.io/logagg-collector/collector/). Captures logs when it is written to a file and caches them until the `logagg-collector` collects and processes the contents.

## Features
* Guarantees capturing every log line.
* Rotation proof.
* One time set-up.
* Supports file patterns; i.e. `/var/log/logagg/syslog*`; rather than fpaths.

## Limitations
* No way of getting logs from files before start-up of the program.
* Requires a reboot of the machine after set-up is done.

## Components/Architecture/Terminology
* **mountpoint**: path to the directory where logs are being written/stored (e.g. /var/log/logagg/).
* **logcache**: path to the directory where the logagg-fs program stores all of it's data.
* **logcache/mirror**: directory inside logcache path which is mounted to the `mountpoint` directory path. If `logcache` path is '/logcache' and the `mountpoint` is '/var/log/logagg/', then the directory '/logcache/mirror' is mounted on to '/var/log/logagg/'.
* **logcache/trackfiles.txt**: file inside logcache directory where file-patterns are mentioned that need to be tracked by logagg-fs (eg. '/var/log/logagg/syslog')
* **logcache/logs**: path to directory where log-files that are cached temprorarily until processed and deleted.

## Prerequisites
* Python => 3.6

## Installation
#### Dependencies
* Install all dependencies prior to actual installation.
```
$ sudo apt install libfuse-dev python3-dev python3-pip pkg-config build-essential python3-pip
$ pip3 install setuptools
```

#### Install logagg-fs
* **NOTE:** Make sure you are a root user.
```
$ pip3 install git+https://github.com/deep-compute/logagg-collector.git
```

* Check installation by the following command
```
$ logagg-fs --version
logagg-fs 0.3.1
logagg-fs 0.3.1
```

## Set-up/Run logagg-fs for mounting /logcache/mirror to /var/log/logagg/
#### Make a directory so that logagg-fs can use it as `logcache`
```
# mkdir /logcache/
```

#### Write configuration to mount /logcache/mirror to /var/log/logagg/ directory in [fstab](https://en.wikipedia.org/wiki/Fstab)
```
# vim /etc/fstab
# Add the following line to /etc/fstab: "logagg-fs /var/log/logagg/ fuse rw,user,auto,exec,nonempty,allow_other,root=/logcache/,loglevel=INFO,logfile=/logcache/fuse.log 0 0"
```
**Command breakdown:**
<br/>
* `logagg-fs`:* the path to logagg-fs program
<br/>
* `/var/log/logagg/`:* the mountpoint
<br/>
* `root=/logcache/`:* the data/logcache directory creater for logagg-fs
<br/>
* `logfile=/logcache/fuse.log:`* path where logagg-fs is supposed to store own logs

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

#### Run to load the configuration in /etc/fstab

Mount logagg-fs from fstab configuration
```
# mount /var/log/logagg/
```

## Usage
Check if '/logcache/mirror' is mounted properly to '/var/log/logagg/'
```
# ls /var/log/logagg/
# # The same as:
# ls /logcache/mirror/
```

```
# cat: /logcache/mirror/test: No such file or directory
# echo "testing.." > /var/log/logagg/test
# cat /logcache/mirror/test
testing..
```

Check caching of log files
```
# ls /logcache/logs/ # No logs yet
# # Now add the files to be tracked in logcache/trackfiles.txt file
# echo "/var/log/logagg/some.log" >> /logcache/trackfiles.txt
# # Takes atmost 10sec to update state
# echo 'Something' >> /var/log/logagg/some.log
# ls /logcache/logs/ # To see the cached log-files
f5fdf6ea0ea92860c6a6b2b354bfcbbc.1536590719.4519932.log
# tail -f /logcache/logs/* # The contents of the file are being written simultaneously to cached files
```
* Just remove the file pattern from `/logcache/trackfiles.txt` to stop caching of logs

* To unmount directory
```
# umount /var/log/logagg/
```
Or Delete configuration from /etc/fstab
```
# reboot
```

