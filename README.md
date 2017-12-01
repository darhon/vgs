# vgs
Test task for VGS

Made on the following environment:
```
# cat /etc/*release|head -n1
CentOS Linux release 7.4.1708 (Core)

# docker version|grep Version|uniq
Version:      17.09.0-ce

# docker-compose version
docker-compose version 1.17.0, build ac53b73
```

Application server is Tomcat (randomly selected).
Log Agent is Fluentd.
Log Forwarder is Fluentd.

To setup the environment just clone repo and run composer file:
```
# git clone https://github.com/darhon/vgs.git
# cd vgs/
# docker-compose up -d
```

There are many possible solutions to this case. Below is a solution based on the conditions of the task.

In this solution, application container ('tomcat_app') uses data container ('vgs_logs') for storing logs. Data container ('vgs_logs') also mounts to Log Agent container ('log_agent') which reads data from there. Log Agent can mount any number of different data container with logs from other applications. So we can have one Log Agent and many application containers on the same instance.

We can simply configure Log Agent for various log sources and configure a variety of filters. After restarting Log Agent container, it will continue to read the logs from the same position, so restarting the container will not result in data loss.

Log Forwarder(s) for demo purposes configured to put incoming logs to stdout, but can be easily configured to store logs to Elasticsearch cluster and/or to S3.

Log Agent configured with HA mode for Log Forwarder. It forward all logs to the first Log Forwarder ('log_forwarder1'), but in case of its inaccessibility, it automatically starts redirecting the logs to the second Log Forwarder ('log_forwarder2'). When the first Log Forwarder becomes available, Log Agent  begins sending data back to it.

This can be easily verified:
* run in terminal:
```
# docker logs -f log_forwarder1
```
* run in another terminal window:
```
# docker logs -f log_forwarder2
```
* open in web browser http://YOUR_HOST_IP:8080
* check first terminal output. It should be something like:
```
2017-12-01 01:49:50 +0000 tomcat.logs.localhost_access_log.2017-12-01.txt: {"message":"192.168.128.176 - - [01/Dec/2017:01:49:49 +0000] \"GET / HTTP/1.1\" 200 11217"}
```
* check another terminal output. It should be empty.
* stop first log forwarder:
```
# docker stop log_forwarder1
```
* refresh several times web page in your browser.
* check second terminal output. There may be a slight delay (<30s) until the Log Agent switches to the second server.
* start first Log Forwarder:
```
# docker start log_forwarder1
```
* refresh several times web page in your browser.
* make sure that the data is sent back to the first Log Forwarder.
