# logagg-fs
**Fuse file system for logagg**

A fuse file-system wich has to be mounted in the file path like '/var/log' so that logagg-collector can collect logs from the files in the file-system.
This guarentees no logs are missed from the files even when the log-files are rotated.

# logagg-collector
**Log collector for logagg**

Collects all the logs from the server and parses it for making a common schema for all the logs and sends to NSQ.

-------------------------------------------------------------------

