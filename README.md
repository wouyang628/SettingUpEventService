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

configure the server to send syslog to the fluentd server
```
root@healthbot:/etc/rsyslog.d# cat 10-rsyslog.conf
*.* @10.49.65.48:1514
root@healthbot:/etc/rsyslog.d# service rsyslog restart
```

configure the router to send syslog to the fluentd server
```
example:
[edit system syslog]
northstar@vmx101# show
host 10.49.65.48 {
    daemon any;
    port 1514;
    inactive: match-strings RPM_TEST_RESULTS;
}
```

configure the router to send snmp  trap to the fluentd server
```
example:
northstar@vmx101# show snmp
community public {
    authorization read-only;
}
trap-group fluentd {
    version v2;
    destination-port 1062;
    categories {
        link;
    }
    targets {
        10.49.65.48;
    }
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

<source>
  @type snmptrap
  tag snmptrap
</source>

<match {system.**,snmptrap}>
  @type copy
  <store>
    @type http
    endpoint_url http://<<ip_address>>:10000
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
2019-09-23 12:24:33.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"chassisd","pid":"14148","message":"JAM:PL: Registered attributes for c04 "}
2019-09-23 12:24:33.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"chassisd","pid":"14148","message":"JAM:PL: Registered attributes for c05 "}
2019-09-23 15:24:44.154827541 -0400 snmptrap: {"value":"\"#<SNMP::SNMPv2_Trap:0x00007f1e653b5b70 @request_id=1478493738, @error_status=0, @error_index=0, @varbind_list=[#<SNMP::VarBind:0x00007f1e6095a760 @name=[1.3.6.1.2.1.1.3.0], @value=#<SNMP::TimeTicks:0x00007f1e6095a7d8 @value=59451327>>, #<SNMP::VarBind:0x00007f1e60959c20 @name=[1.3.6.1.6.3.1.1.4.1.0], @value=[1.3.6.1.6.3.1.1.5.3]>, #<SNMP::VarBind:0x00007f1e60959388 @name=[1.3.6.1.2.1.2.2.1.1.552], @value=#<SNMP::Integer:0x00007f1e609593d8 @value=552>>, #<SNMP::VarBind:0x00007f1e60958b40 @name=[1.3.6.1.2.1.2.2.1.7.552], @value=#<SNMP::Integer:0x00007f1e60958b90 @value=2>>, #<SNMP::VarBind:0x00007f1e609582f8 @name=[1.3.6.1.2.1.2.2.1.8.552], @value=#<SNMP::Integer:0x00007f1e60958348 @value=2>>, #<SNMP::VarBind:0x00007f1e653b6188 @name=[1.3.6.1.2.1.31.1.1.1.1.552], @value=\\\"ge-0/0/5.0\\\">], @source_ip=\\\"10.49.66.3\\\">\"","tags":{"type":"alert","host":"10.49.66.3"}}
2019-09-23 12:24:33.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"chassisd","pid":"14148","message":"JAM:PL: Registered attributes for c06 "}

```
To register events in Appformix
![](https://github.com/wouyang628/event_service_lab_setup/blob/master/images/headers.png)
![](https://github.com/wouyang628/event_service_lab_setup/blob/master/images/event_register.png)


# Notes
1. Error "405 Method Not Allowed" when posting events to Appformix.  
restart the docker restart appformix-controller and try. due to the license being applied after the controller container has started. The routes are conditional on the license.

2. when using "td-agent-gem install fluent-plugin-snmptrap", it does not work. The code in /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluent-plugin-snmptrap-0.0.1/lib/fluent/plugin/in_snmptrap.rb looks different from the code in the orignal repo/plugin.  I have to manually copy fluent-plugin-snmptrap/lib/fluent/plugin/in_snmptrap.rb to /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluent-plugin-snmptrap-0.0.1/lib/fluent/plugin/in_snmptrap.rb
