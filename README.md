# Event service lab setup: Appformix, Healthbot, Fluentd, Event Service container

install dependencies:
```
yum install openssl-devel libffi libffi-devel
yum install python-pip python-devel gcc gcc-c++ make openssl-devel libffi-devel
```   
   
follow this to setup the plugin for fluentd snmp trap
https://github.com/Bigel0w/fluent-plugin-snmptrap

follow this to setup the plugin for sending events out to http
https://github.com/fluent-plugins-nursery/fluent-plugin-out-http
```
[root@linux1 ~]# git clone https://github.com/fluent-plugins-nursery/fluent-plugin-out-http.git
Initialized empty Git repository in /root/fluent-plugin-out-http/.git/
remote: Enumerating objects: 100, done.
remote: Counting objects: 100% (100/100), done.
remote: Compressing objects: 100% (57/57), done.
remote: Total 632 (delta 36), reused 77 (delta 25), pack-reused 532
Receiving objects: 100% (632/632), 113.41 KiB, done.
Resolving deltas: 100% (250/250), done.
[root@linux1 ~]# ls
anaconda-ks.cfg  centos  fluent-plugin-out-http  install.log  install.log.syslog  libData
[root@linux1 ~]# td-agent-gem install fluent-plugin-out-http
Fetching: fluent-plugin-out-http-1.3.1.gem (100%)
Successfully installed fluent-plugin-out-http-1.3.1
Parsing documentation for fluent-plugin-out-http-1.3.1
Installing ri documentation for fluent-plugin-out-http-1.3.1
Done installing documentation for fluent-plugin-out-http after 0 seconds
1 gem installed
```

configure the router to send syslog to the fluentd server
```
[edit system syslog]
northstar@vmx101# show
host 10.49.65.48 {
    daemon any;
    port 1514;
    inactive: match-strings RPM_TEST_RESULTS;
}
```

example of the fluentd config file:
```
[root@linux1 ~]# cat /etc/td-agent/td-agent.conf
<source>
  @type syslog
  port 1514
  tag system
</source>

<match system.**>
  @type copy
  <store>
    @type http
    endpoint_url http://10.49.64.178:10000
    serializer json
  </store>
  <store>
    @type stdout
  </store>
</match>
```

to start/stop fluentd service"
```
/etc/init.d/td-agent start/stop
```

to check the log:
```
tail -f /var/log/td-agent/td-agent.log
```
To register events in Appformix
![](https://github.com/wouyang628/egreePeerEngineering/blob/master/resources/jcblueprint.png)
