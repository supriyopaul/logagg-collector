# logagg-collector
**Log collector for logagg**

Track and collects all the logs from given files and parses them to make a common schema for all the logs and sends to NSQ.

## Prerequisites
* Python >= 3.5
* We expect users to follow [Best practices](https://github.com/deep-compute/logagg/issues/85) for logging their application.
* Most importantly, do structured logging. Since, parsing/formatting logs is way easier that way.
* Install, set-up and run `logagg-fs` beforehand.

## Components/Architecture/Terminology

* `files` : Log files which are being tracked by logagg.
* `node` : The server(s) where the log `files` reside.
* `formatters` : The parser function that the `collector` uses to format the log lines to put it the common format.
* `nsq` : The central location where logs are sent by `collector`(s) after formatting as messages.

## Features

* Guaranteed delivery of each log line from files to `nsq`
* Reduced latency between a log being generated an being present in the `nsq`
* Options to add custom `formatters`
* File poll if log file not yet generated
* Works on rotational log files
* Custom `formatters` to support parsing of any log file.
* Output format of processed log lines (dictionary)
    * `id` (str) - A unique id per log with time ordering. Useful to avoid storing duplicates.
    * `timestamp` (str) - ISO Format time. eg:
    * `data` (dict) - Parsed log data
    * `raw` (str) - Raw log line read from the log file
    * `host` (str) - Hostname of the node where this log was generated
    * `formatter` (str) - name of the formatter that processed the raw log line
    * `file` (str) - Full path of the file in the host where this log line came from
    * `type` (str) - One of "log", "metric" (Is there one more type?)
    * `level` (str) - Log level of the log line.
    * `event` (str) - LOG event
    * `error` (bool) - True if collection handler failed during processing
    * `error_tb` (str) - Error traceback

## Installation
> Prerequisites: Python3.5

#### Setup
- Install, setup and run a [logagg-master](https://github.com/deep-compute/logagg-master) which collectors can register to.

- Install, setup and run [logagg-fs](https://deep-compute.github.io/logagg-collector/fs/) on the same machine so that logagg-collector can efficiently track log files.

- [Install](https://docs.docker.com/install/linux/docker-ce/ubuntu/#extra-steps-for-aufs) the Docker package, at both `collector` nodes.
Run the following commands to install :
```
$ sudo apt-get update

$ sudo apt-get install \
apt-transport-https \
ca-certificates \
curl \
software-properties-common

$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

$ sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \

$(lsb_release -cs) \
stable"

$ sudo apt-get update

$ sudo apt-get install docker-ce
```

- Check Docker version >= 17.12.1
```
$ sudo docker -v
Docker version 18.03.1-ce, build 9ee9f40
```

- Run serverstats docker container to store server metrics in /var/log/serverstats/serverstats.log file.
```
$ sudo docker plugin install deepcompute/docker-file-log-driver:1.0 --alias file-log-driver
$ sudo docker run --hostname $HOSTNAME --name serverstats --restart unless-stopped --label formatter=logagg_collector.formatters.basescript --log-driver file-log-driver --log-opt labels=formatter --log-opt fpath=/serverstats/serverstats.log --detach deepcompute/serverstats:latest
```

#### Install the `logagg-collector` package, at where we collect the logs and at where we forward the logs:
- Run the following command to **pip** install `logagg`: 
```
$ pip3 install https://github.com/deep-compute/pygtail/tarball/master/#egg=pygtail-0.6.1
$ pip3 install git+https://github.com/deep-compute/logagg-utils.git
$ pip3 install git+https://github.com/deep-compute/logagg-collector.git
```

## Basic Usage
- Run collector
```
logagg-collector runserver --host <IP/DNS> --port <port_number> --master host=<master_service_host>:port=<master_service_port>:topic_name=<topic_name> --data-dir <logagg_collector_data_dir> --logaggfs-dir <logagg-fs_logcache_dir>
```
This command runs logagg-collector service, registers collector to master and gets NSQ details from master to send logs to the topic mentioned in the command. File path **'/var/log/serverstats/serverstats.log'** is automatically added as default file-path to be tracked (can be removed using API command if needed).

**Command breakdown:**
<br/>
*`logagg-collector:`* The path to logagg-fs program 
<br/>
*`--host <IP/DNS>:`* Localhost IP or DNS so that other components/services can contact logagg-collector.
<br/>
*`--port <port_number>:`* Port number on which logagg-collector service runs on.
<br/>
*`--master host=<master_service_host>:port=<master_service_port>:topic_name=<topic_name>:`* Host and port where master service is running along with the topic name where the logs are supposed to be sent. Mention same details for aggregating logs in same place.
<br/>
*`--data-dir <logagg_collector_data_dir>:`* Path to a directory where logagg-collector stores it's data (files being tracked, NSQ details). Specify different directories for collectors running on same machine.
<br/>
*`--logaggfs-dir <logagg-fs_logcache_dir>:`* Path to the `logagg-fs` 'logcache' directory.

#### APIs

* To see files collector is supposed to collect
	* *API:* `http://<host>:<port>/collector/v1/get_files`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/get_files'`
	* *Sample response:* `{"success": true, "result": [{"fpath": "/var/log/serverstats/serverstats.log", "formatter": "logagg_collector.formatters.docker_file_log_driver"}]}`
	* *Note:* `Returns file paths including the ones that are not being actively collected but only added to the tracking list`

* To add a new file in the collector's tracking list
	* *API:* `http://<host>:<port>/collector/v1/add_file?fpath="<path_to_file_on_collector_machine>"&formatter="<formatter_for_file_path>"`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/add_file?fpath="/var/log/serverstats.log"&formatter="logagg_collector.formatters.docker_file_log_driver"'`
	* *Sample response:* `{"success": true, "result": [{"fpath": "/var/log/serverstats/serverstats.log", "formatter": "logagg_collector.formatters.docker_file_log_driver"}, {"fpath": "/var/log/serverstats.log", "formatter": "logagg_collector.formatters.docker_file_log_driver"}]}`

* To remove a file from collector's tracking list
	* *API:* `http://<host>:<port>/collector/v1/add_file?fpath="path_to_file_on_collector_machine"`
	* *Sample command:* `curl 'http://localhost:6600/collector/v1/add_file?fpath="/var/log/serverstats.log"'`
	* *Sample response:* `Empty response`
	* *Note:* `Stops and ends the logagg-collector service, should be restarted again`

* To see files collector is actively tracking
	* *API:* `http://<host>:<port>/collector/v1/get_active_log_collectors`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/get_active_log_collectors`
	* *Sample response:* `{"success": true, "result": ["/var/log/serverstats/serverstats.log"]}`

* To get the NSQ details logagg-collector is using
	* *API:* `http://<host>:<port>/collector/v1/get_nsq`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/get_nsq`
	* *Sample response:* `{"success": true, "result": {"nsqd_http_address": "localhost:4151", "topic_name": "logagg_logs"}}`

* To set the NSQ details logagg-collector manually
	* *API:* `http://<host>:<port>/collector/v1/set_nsq?nsqd_http_address=<host:port>&topic_name=<topic_name>`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/set_nsq?nsqd_http_address=localhost:4151&topic_name=logagg_logs`
	* *Sample response:* `{"success": true, "result": {"nsqd_http_address": "78.47.113.210:4151", "topic_name": "supriyo_logs"}}`

* To start tracking logfiles added to logagg-collector tracklist (used when logagg-collector service is started with no master and initialized with NSQ details manually via API command)
	* *API:* `http://<host>:<port>/collector/v1/start`
	* *Sample command:* `curl 'http://localhost:6666/collector/v1/start`
	* *Sample response:* `{"success": true, "result": {"fpaths": [{"fpath": "/var/log/serverstats/serverstats.log", "formatter": "logagg_collector.formatters.docker_file_log_driver"}]}}`

* To stop and exit logagg-collector service
	* *API:* `http://<host>:<port>/collector/v1/stop`
	* *Sample command:* `'http://localhost:6666/collector/v1/stop'`
	* *Sample response:* `Empty response`

## Advanced usage

#### Types of handlers we support
| Formatter-name | Comments |
| -------- | -------- |
|logagg_collector.formatters.nginx_access| See Configuration [here](https://github.com/deep-compute/logagg/issues/61)|
|logagg_collector.formatters.mongodb||
|logagg_collector.formatters.basescript||
|logagg_collector.formatters.docker_log_file_driver|See example [here](https://github.com/deep-compute/serverstats/issues/6)|

----

## Run logagg-collector without a `master`
###### [Install](http://nsq.io/deployment/installing.html) the `NSQ` package, at where you need to bring up the NSQ server.
**Run the following commands to install `nsq`:**
</br>
```
$ sudo apt-get install libsnappy-dev
$ wget https://s3.amazonaws.com/bitly-downloads/nsq/nsq-1.0.0-compat.linux-amd64.go1.8.tar.gz
$ tar zxvf nsq-1.0.0-compat.linux-amd64.go1.8.tar.gz
$ sudo cp nsq-1.0.0-compat.linux-amd64.go1.8/bin/* /usr/local/bin
```
###### Bring up the `nsq` instances at the required server with following commands:
**NOTE:** Run each command in a seperate Terminal window
</br>
*nsqlookupd:*
```
$ nsqlookupd
```
</br>
*nsqd:*
```
$ nsqd -lookupd-tcp-address localhost:4160
```
</br>
*nsqadmin:*
```
$ nsqadmin -lookupd-http-address localhost:4161
```

###### Run logagg-collector service and start collecting logs
*Start loagg-collector service*
```
$ logagg-collector runserver --no-master --data-dir <logagg_collector_data_dir> --logaggfs-dir <logagg-fs_logcache_dir>
```
*Add NSQ details to logagg-collector*
```
$ curl http://<host>:<port>/collector/v1/set_nsq?nsqd_http_address=<host:port>&topic_name=<topic_name>
```
*Start tracking files*
```
$ curl http://<host>:<port>/collector/v1/start
```

</br>
You can check message traffic at `nsq` by going through the link: `http://<nsq-server-ip-or-DNS>:4171/` ; for **localhost** see [here](http://localhost:4171/)
</br>
You can see the collected logs in realtime using the following command:
```
$ nsq_tail --topic=logagg --channel=test --lookupd-http-address=<nsq-server-ip-or-DNS>:4161
```

---

## How to create and use custom formatters for log files
###### Step 1: make a directory and append it's path to evironment variable $PYTHONPATH
```
$ echo $PYTHONPATH

$ mkdir customformatters
$ #Now append the path to $PYTHONPATH
$ export PYTHONPATH=$PYTHONPATH:/home/path/to/customformatters/

$ echo $PYTHONPATH
:/home/path/to/customformatters
```

###### Step 2: Create a another directory and put your formatter file(s) inside it.
```
$ cd customformatters/
$ mkdir myformatters
$ cd myformatters/
$ touch formatters.py
$ touch __init__.py
$ echo 'import formatters' >> __init__.py
$ #Now write your formatter functions inside the formatters.py file
```
#### Step 3: Write your formatter functions inside the formatters.py file
**Important:**
</br> 
1. Only **python standard modules** can be imported in formatters.py file.
</br>
2. A formatter function should return a **dict()** datatype.
</br>
3. The 'dict()' should only contain keys which are mentioned in the above `log structure`.
</br>
4. Sample formatter functions:
```
import json 
import re

sample_log_line = '2018-02-07T06:37:00.297610Z [Some_event] [Info] [Hello_there]'

def sample_formatter(log_line):
log = re.sub('[\[+\]]', '',log_line).split(' ')
timestamp = log[0]
event = log[1]
level = log[2]
data = dict({'message': log[3]})

return dict(timestamp = timestamp,
	     event = event,
	     level = level,
	     data = data,
	    )
```
To see more examples, look [here](https://github.com/deep-compute/logagg-collector/blob/master/logagg_collector/formatters.py).
</br>
5. Check if the custom handler works in `python interpreter` like for logagg.
```
>>> import myformatters
>>> sample_log_line = '2018-02-07T06:37:00.297610Z [Some_event] [Info] [Hello_there]'
>>> output = myformatters.formatters.sample_formatter(sample_log_line)
>>> from pprint import pprint
>>> pprint(output)
{'data': {'message': 'Hello_there'},
'event': 'Some_event',
'level': 'Info',
'timestamp': '2018-02-07T06:37:00.297610Z'}
```
6. Pseudo API command:
```
curl 'http://<host>:<port>/collector/v1/add_file?fpath="logfile.log"&formatter="myformatters.formatters.sample_formatter"'
```

## Debugging
If there are multiple files being tracked by multiple collectors on multiple nodes, the collector information can be seen in "Heartbeat" topic of NSQ.
Every running collector sends a hearbeat to this topic (default interval = 30 seconds). The heartbeat format is as follows:
</br>
`timestamp` : Timestamp of the recieved heartbeat.
</br>
`heartbeat_number` : The heartbeat number since the collector started running.
</br>
`host` : Hostname of the node on which the collector is running.
</br>
`nsq_topic` : The nsq topic which the collector is using.
</br>
`files_tracked` : list of files that are being tracked by the collector followed by the fomatter.
</br>
You can run the following command to see the information:
```
$ nsq_tail --topic=<heartbeat_topic> --channel=test --lookupd-http-address=<nsq-server-ip-or-DNS>:4161
```

## Build on logagg

You're more than welcome to hack on this:-)
```
$ git clone https://github.com/deep-compute/logagg-collector
$ cd logagg-collector
$ sudo python setup.py install
$ docker build -t logagg-collector .
```
