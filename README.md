# Overview
The event service app can receive events from Healthbot, Fluentd and convert to Appformix format and send to Appformix server for events display and alarming. 

# Scenarios
##  1. Application to Route Critical events into AppFormix Application Ingest packaged as a Docker container.
After executing the `playbooks/logging/deploy-event-service.yml` playbook, the container for event service should be up and running. This can be verified by executing the command `docker ps` on the event server.
  
```
  root@event-server:~/app# docker ps
  CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                      NAMES
  9188c11f235f        event-service:latest   "/bin/sh -c 'python3…"   6 seconds ago       Up 5 seconds        0.0.0.0:10000->10000/tcp   event-service
```

The log can be verified by executing the command `docker logs event-service --tail 10 -f`
```
  root@event-server:~/app# docker logs event-service --tail 10 -f
   * Serving Flask app "event_service" (lazy loading)
   * Environment: production
     WARNING: This is a development server. Do not use it in a production deployment.
     Use a production WSGI server instead.
   * Debug mode: off
   * Running on http://0.0.0.0:10000/ (Press CTRL+C to quit)
  
```

To log into the container by executing the command `docker exec -it event-service /bin/sh`
```
root@event-server:~/app# docker exec -it event-service /bin/sh
/application_events #
```
  
##  2. Source of Input messages supported in this Application are Syslog (from Fluentd Log Collector and Log Processor modules) and Healthbot Critical Events.  
The event service app is able to receive parsed syslog and snmp trap events from Fluentd, events from Healthbot and convert the format and send to Appformix event API.  
![flow_chart](/images/flow_chart.png)

**example: server syslog -> Fluentd -> Event Service -> Appformix**  

The Fluentd server needs to send parsed server syslog message to the Event Service server. If you install with fluentd playbook, then the http output plug-in and td-agent.conf should already be configured; otherwise, you can install the http output plug-in manually following https://github.com/fluent-plugins-nursery/fluent-plugin-out-http ; and configure the setting by modifying `/etc/td-agent/td-agent.conf` file. The following configuration file shows an example of fluentd receiving syslog and snmp trap and send the output to event service server.    
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
    endpoint_url http://<<event_service_server_ip>>:10000
    serializer json
  </store>
  <store>
    @type stdout
  </store>
</match>
```

restart the td agent after making configuration changes:
```
[root@linux1 ~]# /etc/init.d/td-agent stop
Stopping td-agent: td-agent                                [  OK  ]
[root@linux1 ~]# /etc/init.d/td-agent start
Startint td-agent: td-agent                                [  OK  ]
```

In order for the server to send syslog messages, the system syslog must be configured to send log message to the fluentd server.  
The following shows a configuration example/step:
```
root@server1:/etc/rsyslog.d# cat /etc/rsyslog.d/10-rsyslog.conf
*.* @10.49.65.48:1514
root@server1:/etc/rsyslog.d# service rsyslog restart
```

Check the Fluentd log to see if syslog messages are received:  
```
tail -f /var/log/td-agent/td-agent.log
2019-10-04 08:38:37.000000000 -0400 system.kern.info: {"host":"healthbot","ident":"kernel","message":"[1536386.184297] device ens3f0 entered promiscuous mode"}
2019-10-04 08:38:38.000000000 -0400 system.kern.info: {"host":"healthbot","ident":"kernel","message":"[1536387.998722] device ens3f0 left promiscuous mode"}
```

Check the event-service container log to see if events are parsed and sent to Appformix correctly:  
```
root@event-server:~/app# docker logs event-service --tail 10 -f
 * Serving Flask app "event_service" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:10000/ (Press CTRL+C to quit)

<appformix_message.AppFormixMessage object at 0x7fe6b0a05fd0>
post succesful
10.49.65.48 - - [04/Oct/2019 08:38:45] "POST / HTTP/1.1" 200 -
{'ident': 'kernel', 'host': 'healthbot', 'message': '[1536386.184297] device ens3f0 entered promiscuous mode'}
<fluentd_message.FluentdMessage object at 0x7fe6b0a05470>
converted data:{
    "EventId": "syslog",
    "Metric": "0",

    "Metadata": {
        "element_name": "kernel",
        "message": "[1536386.184297] device ens3f0 entered promiscuous mode",
        "device_ID": "healthbot"
    },
    "ApplicationId": "fluentd"
}


<appformix_message.AppFormixMessage object at 0x7fe6b0a05e10>
post succesful
10.49.65.48 - - [04/Oct/2019 08:38:45] "POST / HTTP/1.1" 200 -
{'ident': 'kernel', 'host': 'healthbot', 'message': '[1536387.998722] device ens3f0 left promiscuous mode'}
<fluentd_message.FluentdMessage object at 0x7fe6b0a0e2b0>
converted data:{
    "EventId": "syslog",
    "Metric": "0",

    "Metadata": {
        "element_name": "kernel",
        "message": "[1536387.998722] device ens3f0 left promiscuous mode",
        "device_ID": "healthbot"
    },
    "ApplicationId": "fluentd"
}

```

Check the Appformix GUI to see if the event has been received successfully:  
![appformix_event_fluentd_server_syslog](/images/appformix_event_fluentd_server_syslog.png)

Make sure that the event and application is registered( this should be done during the Appformix installation). API can be used to manually register the application events.

To check the already registered application and events. Use GET http://{{server}}:{{port}}/appformix/v1.0/application_registration  
![QQ20191004-130112](/images/postman_api_register.png)

To register application and events.Use POST http://{{server}}:{{port}}/appformix/controller/v2.0/application_registration  
![register1](/images/postman_api_register2.png)
![register2](/images/postman_api_register3.png)



##  3. Application can send critical events as Application Events into AppFormix
**example: Healthbot event -> Event Service -> Appformix**  

In order for Healthbot to send events to the Event Service server, Healthbot needs to be configured to send notification to the Event Service server web hook address. Follow the Healthbot installation and configuration guide for the setup.  
In this example, we will use Healthbot to monitroing juniper device CPU utilization (using OpenConfig) and sends out an event to Event Service when the CPU exceeds 80%. After processing the event, Event Service will send the event to Appformix.  

To enable OpenConfig on the Juniper device. Here is an example configlet:  
```
set system services extension-service request-response grpc clear-text port 32767
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 0.0.0.0/0
```

To configue the Rule and Playbook on Healthbot:  
```
set iceberg topic check-cpu rule check-system-cpu keys element_name
set iceberg topic check-cpu rule check-system-cpu synopsis "Routing engine CPU analyzer"
set iceberg topic check-cpu rule check-system-cpu description "Collects system RE CPU statistics periodically and notifies anomalies when CPU utilization exceeds threshold"
set iceberg topic check-cpu rule check-system-cpu function subtract description "Subtracts cpu-idle% from 100 to get CPU used percent"
set iceberg topic check-cpu rule check-system-cpu function subtract path system-sensors.py
set iceberg topic check-cpu rule check-system-cpu function subtract method subtract
set iceberg topic check-cpu rule check-system-cpu function subtract argument threshold mandatory
set iceberg topic check-cpu rule check-system-cpu sensor components synopsis "system components open-config sensor definition"
set iceberg topic check-cpu rule check-system-cpu sensor components description "/components open-config sensor to collect telemetry data from network device"
set iceberg topic check-cpu rule check-system-cpu sensor components open-config sensor-name /components/
set iceberg topic check-cpu rule check-system-cpu sensor components open-config frequency 60s
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization formula user-defined-function function-name subtract
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization formula user-defined-function argument threshold "$re-cpu-utilization-oc"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization type integer
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization description "Derives used percentage from idle CPU utilization"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-high-threshold constant value "{{re-cpu-high-threshold}}"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-high-threshold type integer
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-high-threshold description "RE CPU utilization high threshold"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-low-threshold constant value "{{re-cpu-low-threshold}}"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-low-threshold type integer
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-low-threshold description "RE CPU utilization low threshold"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-oc sensor components where "/components/component/properties/property/@name == 'cpu-utilization-idle'"
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-oc sensor components path /components/component/properties/property/state/value
set iceberg topic check-cpu rule check-system-cpu field re-cpu-utilization-oc description "RE CPU idle utilization"
set iceberg topic check-cpu rule check-system-cpu field element_name sensor components where "/components/component/@name =~ /^Routing Engine[{{re-slot-no}}]*$/"
set iceberg topic check-cpu rule check-system-cpu field element_name sensor components path "/components/component/@name"
set iceberg topic check-cpu rule check-system-cpu field element_name description "RE name to monitor"
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization synopsis "Routing engine CPU KPI"
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization description "Sets health based on increase in RE CPU utilization"
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization frequency 60s
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-abnormal when greater-than-or-equal-to "$re-cpu-utilization" "$re-cpu-utilization-high-threshold" time-range 5m
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-abnormal when greater-than-or-equal-to "$re-cpu-utilization" "$re-cpu-utilization-high-threshold" all
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-abnormal then status color red
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-abnormal then status message "$element_name CPU utilization($re-cpu-utilization) exceeds high threshold($re-cpu-utilization-high-threshold)"
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-middle when greater-than-or-equal-to "$re-cpu-utilization" "$re-cpu-utilization-low-threshold" time-range 5m
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-middle when greater-than-or-equal-to "$re-cpu-utilization" "$re-cpu-utilization-low-threshold" all
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-middle then status color yellow
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term is-re-cpu-utilization-middle then status message "$element_name CPU utilization($re-cpu-utilization) in medium range"
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term re-cpu-utilization-normal then status color green
set iceberg topic check-cpu rule check-system-cpu trigger re-cpu-utilization term re-cpu-utilization-normal then status message "$element_name CPU utilization($re-cpu-utilization) is normal"
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-high-threshold value 80
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-high-threshold description "RE CPU utilization high threshold: Utilization increase between metrics, before we report anomaly"
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-high-threshold type int
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-low-threshold value 50
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-low-threshold description "RE CPU utilization low threshold: Utilization increase between metrics, before we report anomaly"
set iceberg topic check-cpu rule check-system-cpu variable re-cpu-low-threshold type int
set iceberg topic check-cpu rule check-system-cpu variable re-slot-no value 0-1
set iceberg topic check-cpu rule check-system-cpu variable re-slot-no description "Routing engine numbers to monitor, regular expression, e.g. '0'"
set iceberg topic check-cpu rule check-system-cpu variable re-slot-no type string

set iceberg playbook check-cpu rules check-cpu/check-system-cpu

set iceberg device vmx101 host 10.49.66.3
set iceberg device vmx101 open-config port 32767
set iceberg device vmx101 iAgent port 830
set iceberg device vmx101 authentication password username northstar
set iceberg device vmx101 authentication password password "$9$pWnk0BRLX-bwgRhdbs4Dj/CtpBEylM"
set iceberg device vmx101 vendor juniper operating-system junos

set iceberg device-group group_all devices vmx101
set iceberg device-group group_all devices vmx102
set iceberg device-group group_all playbooks check-cpu
set iceberg device-group group_all notification normal event_service
set iceberg device-group group_all notification minor event_service
set iceberg device-group group_all notification major event_service
set iceberg device-group group_all notification enable
set iceberg device-group group_all variable check-cpu check-cpu check-cpu/check-system-cpu

set iceberg notification event_service http-post url http://10.49.64.178:10000
```

After the device CPU goes above 80%, check the Event Service log to see if the event is received and posted to Appformix:  
```

root@ubuntu:~/app/application_events# python3 event_service.py
 * Serving Flask app "event_service" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:10000/ (Press CTRL+C to quit)

{'severity': 'major', 'group': 'group_all', 'keys': {'_playbook_name': 'check-cpu', '_instance_id': '["check-cpu"]', 'element_name': 'Routing Engine0'}, 'rule': 'check-system-cpu', 'trigger': 're-cpu-utilization', 'device-id': 'vmx101', 'topic': 'check-cpu', 'message': 'Routing Engine0 CPU utilization(100) exceeds high threshold(80)'}
<healthbot_message.HealthbotMessage object at 0x7f95c554b7b8>
converted data:{
    "EventId": "re-cpu-utilization",
    "Metric": "0",

    "Metadata": {
        "element_name": "Routing Engine0",
        "severity": "major",
        "message": "Routing Engine0 CPU utilization(100) exceeds high threshold(80)",
        "device_ID": "vmx101"
    },
    "ApplicationId": "healthbot"
}
<appformix_message.AppFormixMessage object at 0x7f95c554bc88>
post succesful
```

check Appformix GUI to see if the event is received and displayed:  
![appformix-healthbot-event](/images/appformix-healthbot-event.png)


##  4. Application can send critical events as Application Events with Alarms and Subscriptions using AppFormix v2.0 APIs.
**example: router jitter syslog -> Event Service -> Appformix**

The Fluentd server needs to send parsed syslog jitter to the Event Service server. If you install with fluentd playbook, then the http output plug-in and td-agent.conf should already be configured; otherwise, you can install the http output plug-in manually following https://github.com/fluent-plugins-nursery/fluent-plugin-out-http ; and configure the setting by modifying `/etc/td-agent/td-agent.conf` file. The following configuration file shows an example of fluentd receiving syslog and snmp trap and send the output to event service server.    
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
    endpoint_url http://<<event_service_server_ip>>:10000
    serializer json
  </store>
  <store>
    @type stdout
  </store>
</match>
```

restart the td agent after making configuration changes:
```
[root@linux1 ~]# /etc/init.d/td-agent stop
Stopping td-agent: td-agent                                [  OK  ]
[root@linux1 ~]# /etc/init.d/td-agent start
Startint td-agent: td-agent                                [  OK  ]
```

In order for the router to send syslog jitter messages, the rpm and system syslog must be configured on the router.  
The following shows a configuration example. Note that the rpm-log.slax can be obtained from Northstar installation under /opt/northstar/data/logstash/utils/junoscripts/ directory.
```
system {
    syslog {
        host 172.16.18.1 {
            daemon info;
            port 1514;
            match-strings RPM_TEST_RESULTS;
        }
    }
}
services {
    rpm {
        probe northstar-ifl {
            test ge-0/1/1.0 {
                probe-type icmp-ping-timestamp;
                target address 11.101.105.2;
                probe-count 15;
                probe-interval 1;
                test-interval 20;
                source-address 11.101.105.1;
                history-size 512;
                moving-average-size 60;
                traps test-completion;
                destination-interface ge-0/1/1.0;
                one-way-hardware-timestamp;
        }
}
event-options {
    event-script {
        file rpm-log.slax;
    }
}
```

Verify that the rpm probe is working on the router:
```
northstar@vmx101# run show services rpm probe-results
    Owner: northstar-ifl, Test: ge-0/1/1.0
    Target address: 11.101.105.2, Source address: 11.101.105.1, Probe type: icmp-ping-timestamp, Icmp-id: 1147
    Destination interface name: ge-0/1/1.0
    Test size: 15 probes
    Probe results:
      Response received
      Probe sent time: Fri Oct  4 11:47:27 2019
      Probe rcvd/timeout time: Fri Oct  4 11:47:27 2019, No hardware timestamps
      Rtt: 3706 usec, Round trip jitter: 515 usec
      Round trip interarrival jitter: 3825 usec
```

Check the Fluentd log to see if syslog jitter messages are received:  
```
[root@linux1 ~]# tail -f /var/log/td-agent/td-agent.log
2019-10-04 12:35:04.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"cscript","message":"RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/1/1.0 loss=0 min-rtt=2.76 max-rtt=38.48 avgerage-rtt=4.71 jitter=35.723"}
2019-10-04 12:35:06.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"cscript","message":"RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/0/6.0 loss=0 min-rtt=2.65 max-rtt=59.12 avgerage-rtt=7.38 jitter=56.467"}
2019-10-04 12:35:20.000000000 -0400 system.daemon.info: {"host":"vmx101","ident":"cscript","message":"RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/0/5.0 loss=0 min-rtt=2.7 max-rtt=52.37 avgerage-rtt=6.05 jitter=49.671"}
```

Check the event-service container log to see if events are parsed and sent to Appformix correctly:  
```
root@event-server:~/app# docker logs event-service --tail 10 -f
 * Serving Flask app "event_service" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:10000/ (Press CTRL+C to quit)

{'ident': 'cscript', 'host': 'vmx101', 'message': 'RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/1/1.0 loss=0 min-rtt=2.76 max-rtt=38.48 avgerage-rtt=4.71 jitter=35.723'}
<fluentd_message.FluentdMessageJitter object at 0x7fe6b0a076a0>
converted data:{
    "EventId": "syslogJitter",
    "Metric": "35.723",

    "Metadata": {
        "element_name": "ge-0/1/1.0",
        "message": "RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/1/1.0 loss=0 min-rtt=2.76 max-rtt=38.48 avgerage-rtt=4.71 jitter=35.723",
        "device_ID": "vmx101"
    },
    "ApplicationId": "fluentd"
}

<appformix_message.AppFormixMessage object at 0x7fe6b09fd4a8>
post succesful
10.49.65.48 - - [04/Oct/2019 12:35:04] "POST / HTTP/1.1" 200 -
{'ident': 'cscript', 'host': 'vmx101', 'message': 'RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/0/6.0 loss=0 min-rtt=2.65 max-rtt=59.12 avgerage-rtt=7.38 jitter=56.467'}
<fluentd_message.FluentdMessageJitter object at 0x7fe6b09fd5c0>
converted data:{
    "EventId": "syslogJitter",
    "Metric": "56.467",

    "Metadata": {
        "element_name": "ge-0/0/6.0",
        "message": "RPM_TEST_RESULTS: test-owner=northstar-ifl test-name=ge-0/0/6.0 loss=0 min-rtt=2.65 max-rtt=59.12 avgerage-rtt=7.38 jitter=56.467",
        "device_ID": "vmx101"
    },
    "ApplicationId": "fluentd"
}

<appformix_message.AppFormixMessage object at 0x7fe6b09fd710>
post succesful
```

Check the Appformix GUI to see if the event has been received successfully:  
![appformix_jitter](/images/appformix_jitter.png)



##  5. Fluentd Snmptrap Plugin (attached to Log Collector and Log Processor modules) can ingest Snmp Traps from Servers and pass them through this Application

**example: interface down  SNMP trap -> Event Service -> Appformix**  

The Fluentd server needs to send parsed SNMP trap to the Event Service server. If you install with fluentd playbook, then the SNMP input plug-in, http output plug-in and td-agent.conf should already be configured; otherwise, you can install the SNMP input plug-in and http output plug-in manually following https://github.com/Bigel0w/fluent-plugin-snmptrap and https://github.com/fluent-plugins-nursery/fluent-plugin-out-http ; and configure the setting by modifying `/etc/td-agent/td-agent.conf` file. The following configuration file shows an example of fluentd receiving syslog and snmp trap and send the output to event service server.  
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
    endpoint_url http://<<event_service_server_ip>>:10000
    serializer json
  </store>
  <store>
    @type stdout
  </store>
</match>
```

restart the td agent after making configuration changes:
```
[root@linux1 ~]# /etc/init.d/td-agent stop
Stopping td-agent: td-agent                                [  OK  ]
[root@linux1 ~]# /etc/init.d/td-agent start
Startint td-agent: td-agent                                [  OK  ]
```

In order for the juniper device to send snmp trap, the snmp trap must be configured.  
e.g.
```
snmp {
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
}
```

To test this, we can bring down an interface on the router and check if the trap is received.
```
[edit]
northstar@vmx101# set interfaces ge-0/0/5.0 disable

[edit]
northstar@vmx101# commit
commit complete
```

Check the Fluentd log to see if syslog jitter messages are received:  
```
[root@linux1 ~]# tail -f /var/log/td-agent/td-agent.log
2019-10-04 15:56:53.244951713 -0400 snmptrap: {"value":"\"#<SNMP::SNMPv2_Trap:0x00007fa9a3836ea0 @request_id=1478601373, @error_status=0, @error_index=0, @varbind_list=[#<SNMP::VarBind:0x00007fa9a3cc8888 @name=[1.3.6.1.2.1.1.3.0], @value=#<SNMP::TimeTicks:0x00007fa9a3cc8900 @value=154682497>>, #<SNMP::VarBind:0x00007fa9a637f9b8 @name=[1.3.6.1.6.3.1.1.4.1.0], @value=[1.3.6.1.6.3.1.1.5.3]>, #<SNMP::VarBind:0x00007fa9a637d730 @name=[1.3.6.1.2.1.2.2.1.1.552], @value=#<SNMP::Integer:0x00007fa9a637d7d0 @value=552>>, #<SNMP::VarBind:0x00007fa9a637c100 @name=[1.3.6.1.2.1.2.2.1.7.552], @value=#<SNMP::Integer:0x00007fa9a637c150 @value=2>>, #<SNMP::VarBind:0x00007fa9a3837850 @name=[1.3.6.1.2.1.2.2.1.8.552], @value=#<SNMP::Integer:0x00007fa9a38378a0 @value=2>>, #<SNMP::VarBind:0x00007fa9a3836fb8 @name=[1.3.6.1.2.1.31.1.1.1.1.552], @value=\\\"ge-0/0/5.0\\\">], @source_ip=\\\"10.49.66.3\\\">\"","tags":{"type":"alert","host":"10.49.66.3"}}
```

Check the event-service container log to see if events are parsed and sent to Appformix correctly:  
```
root@event-server:~/app# docker logs event-service --tail 10 -f
 * Serving Flask app "event_service" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:10000/ (Press CTRL+C to quit)

10.49.65.48 - - [04/Oct/2019 12:35:20] "POST / HTTP/1.1" 200 -
{'value': '"#<SNMP::SNMPv2_Trap:0x00007fa9a3836ea0 @request_id=1478601373, @error_status=0, @error_index=0, @varbind_list=[#<SNMP::VarBind:0x00007fa9a3cc8888 @name=[1.3.6.1.2.1.1.3.0], @value=#<SNMP::TimeTicks:0x00007fa9a3cc8900 @value=154682497>>, #<SNMP::VarBind:0x00007fa9a637f9b8 @name=[1.3.6.1.6.3.1.1.4.1.0], @value=[1.3.6.1.6.3.1.1.5.3]>, #<SNMP::VarBind:0x00007fa9a637d730 @name=[1.3.6.1.2.1.2.2.1.1.552], @value=#<SNMP::Integer:0x00007fa9a637d7d0 @value=552>>, #<SNMP::VarBind:0x00007fa9a637c100 @name=[1.3.6.1.2.1.2.2.1.7.552], @value=#<SNMP::Integer:0x00007fa9a637c150 @value=2>>, #<SNMP::VarBind:0x00007fa9a3837850 @name=[1.3.6.1.2.1.2.2.1.8.552], @value=#<SNMP::Integer:0x00007fa9a38378a0 @value=2>>, #<SNMP::VarBind:0x00007fa9a3836fb8 @name=[1.3.6.1.2.1.31.1.1.1.1.552], @value=\\"ge-0/0/5.0\\">], @source_ip=\\"10.49.66.3\\">"', 'tags': {'type': 'alert', 'host': '10.49.66.3'}}
<fluentd_message.FluentdMessageSNMPTrap object at 0x7fe6b0a07400>
converted data:{
    "EventId": "SNMPTrap",
    "Metric": "0",

    "Metadata": {
        "message": "#<SNMP::SNMPv2_Trap:0x00007fa9a3836ea0 @request_id=1478601373, @error_status=0, @error_index=0, @varbind_list=[#<SNMP::VarBind:0x00007fa9a3cc8888 @name=[1.3.6.1.2.1.1.3.0], @value=#<SNMP::TimeTicks:0x00007fa9a3cc8900 @value=154682497>>, #<SNMP::VarBind:0x00007fa9a637f9b8 @name=[1.3.6.1.6.3.1.1.4.1.0], @value=[1.3.6.1.6.3.1.1.5.3]>, #<SNMP::VarBind:0x00007fa9a637d730 @name=[1.3.6.1.2.1.2.2.1.1.552], @value=#<SNMP::Integer:0x00007fa9a637d7d0 @value=552>>, #<SNMP::VarBind:0x00007fa9a637c100 @name=[1.3.6.1.2.1.2.2.1.7.552], @value=#<SNMP::Integer:0x00007fa9a637c150 @value=2>>, #<SNMP::VarBind:0x00007fa9a3837850 @name=[1.3.6.1.2.1.2.2.1.8.552], @value=#<SNMP::Integer:0x00007fa9a38378a0 @value=2>>, #<SNMP::VarBind:0x00007fa9a3836fb8 @name=[1.3.6.1.2.1.31.1.1.1.1.552], @value=\"ge-0/0/5.0\">], @source_ip=\"10.49.66.3\">",
        "device_ID": "10.49.66.3"
    },
    "ApplicationId": "fluentd"
}


<appformix_message.AppFormixMessage object at 0x7fe6b1248e80>
post succesful
10.49.65.48 - - [04/Oct/2019 12:56:25] "POST / HTTP/1.1" 200 -
```

Check the Appformix GUI to see if the event has been received successfully:  
![fluentdSNMPtrap](/images/fluentdSNMPtrap.png)
