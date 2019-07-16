# Geode/GemFire JMX Monitor

The datatx-geode-jmx-monitor is an application which is used to monitor a Geode/GemFire cluster and generate alert messages based on JMX notifications sent from Geode/GemFire locator(s) to the monitor. JMX notifications are error messages written to Geode/GemFire logs with a severity of "warning" or higher. Additional notifications that are also sent from the Geode/GemFire locators include the departure, crash or joining of a member to the cluster.

Another feature of the monitor application supports the definition Geode/GemFire metrics which can be monitored and have alerts generated in the event a metric threshold is exceeded.

This project implements the abstract project datatx-geode-monitor which provides the majority of the monitoring with the exception of the abstract method sendAlert.

***public abstract void sendAlert(LogMessage logMessage);***

## LogMessage Format:

    public class LogMessage {
       private LogHeader header;
       private Notification event;
       private String body;
       private int count = 1;
    }
    
    public LogHeader(String severity, String date, String time, String zone,
       String member, String event, String tid) {
       this.severity = severity;
       this.date = date;
       this.zone = zone;
       this.member = member;
       this.event = event;
       this.time = time.replace(".", ":");
       this.tid = tid;
    }
    
## Monitor Configuration Files

### monitor.propertiers
The monitor properties are used to define the monitor configuration and behaviuor.

1. managers - This property is a comma seperated list of locators that have a JMX manager defined=localhost
2. port - This property defines the port number assiigned to the JMX manager
3. command-port - This property defines the incoming TCP/IP port the monitor listens on for operator commands
4. message-duration  - The time in minutes the monitor will wait before sending a duplicate message
5. maximum-duplicates - The maximum number of duplicate messages
6. reconnect-wait-time - The time in minutes the monitor will wait before trying to reconnect to a locator  
7. reconnect-retry-attempts - The number of retry attempts before the monitor will try to use the next locator JMX maqnaqger defined in the managers property. 
8. log-file-size - The maximum size of a monitor log file
9. log-file-backups - The number of log backupsto maintain before rolling off the oldest log file

### log4j.properties
The log4j properties are used to define the monitor logging behavior.

log4j.rootLogger=TRACE, stdout
log4j.rootLogger=OFF
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.applicationLog=org.apache.log4j.RollingFileAppender
log4j.appender.applicationLog.File=/usr/bin/monitor/logs/Alert_Health_Monitor.log
log4j.appender.applicationLog.layout=org.apache.log4j.PatternLayout
log4j.appender.applicationLog.MaxFileSize=2000KB
log4j.appender.applicationLog.MaxBackupIndex=5
log4j.appender.exceptionLog=org.apache.log4j.RollingFileAppender
log4j.appender.exceptionLog.File=/usr/bin/monitor/logs/Alert_Health_Monitor_Exceptions
log4j.appender.exceptionLog.layout=org.apache.log4j.PatternLayout
log4j.appender.exceptionLog.MaxFileSize=2000KB
log4j.appender.exceptionLog.MaxBackupIndex=5
log4j.category.applicationLog=TRACE, applicationLog
log4j.additivity.applicationLog=false
log4j.category.exceptionLog=DEBUG, exceptionLog
log4j.additivity.exceptionLog=false

### gemfireThreads.xml
The gemfireThreads XML file contain the messages generated by Geode/GemFire threads that will be supprersed by the monitor.

    <gemfireThreads>
        <gemfireThread thread="Event Processor for GatewaySender_" />
        <gemfireThread thread="WAN Locator Discovery Thread" />
    </gemfireThreads>

### excludedMessages.xml
The excludedMessages XML file contain the messages or message snippets generated by Geode/GemFire that will be suppressed by the monitor.

    <excludedMessages>
        <excludedMessage>
            <criteria>DISTRIBUTED_NO_ACK but enable-network-partition-detection</criteria>
        </excludedMessage>
        <excludedMessage>
            <criteria>DLockRequestProcessor</criteria>
            <criteria>have elapsed while waiting for replies</criteria>
        </excludedMessage>
    </excludedMessages>

### mxbeans.xml
The mxbeans XML file contain the JMX mBeans objects and associated object properties used to monitor Geode/GemFire metrics

    <mxBeans sampleTime="5000">
         <mxBean mxBeanName="DistributedSystemMXBean">
             <fields>
                 <field beanProperty="" fieldName="UsedHeapSize" fieldSize="ACTUAL" count="0" percentage=".75"
                    percentageField="TotalHeapSize" percentageFieldSize="ACTUAL" />
                 <field beanProperty="" fieldName="JVMPauses" fieldSize="ACTUAL" count="2" percentage="0" 
                    percentageField="" percentageFieldSize="ACTUAL" />
             </fields>
         </mxBean>
    </mxBeans>

### mxBean Properties

1. mxBeanName - The Geode/GemFire mBean object name
2. Fields
3. Field
   * beanProperty - 
   * fieldName - The mBean field name to monitor
   * fieldSize - Enum ACTUAL/KILOBYTES/MEGABYTES
   * count - Threashold as a count 
   * percentage - Threshold as a percentage 
   * percentageField - The mBean field that will be used to validate if a threshold for a metric is exceeded.
   * percentageFieldSize - Enum ACTUAL/KILOBYTES/MEGABYTES
   
## Dockerfile

The Geode/GemFire monitor can be run in a docker container. 

    FROM centos:centos7
    ENV container=docker
    RUN yum -y update
    RUN yum makecache fast
    RUN yum install -y nc
    RUN yum install -y net-tools
    RUN yum install -y tcpdump
    RUN yum install -y java-1.8.0-openjdk 
    RUN yum install -y java-1.8.0-openjdk-devel
    RUN yum install -y ant
    RUN yum install -y zip
    RUN yum install -y unzip
    RUN yum install -y mtr

    ADD target/datatx-geode-jmx-monitor-1.0.0-archive.zip /usr/bin
    RUN unzip /usr/bin/datatx-geode-jmx-monitor-1.0.0-archive.zip -d /usr/bin/
    RUN chmod +x /usr/bin/monitor/start_monitor.sh
    RUN rm -f /usr/bin/datatx-geode-jmx-monitor-1.0.0-archive.zip

    EXPOSE 6780

    CMD  bash "/usr/bin/monitor/start_monitor.sh"

### Docker Build

docker build -t geode-monitor .

### Docker Run
docker run -d -it geode-monitor

## Start Monitor Script (start_monitor.sh)

#!/bin/bash
java -cp /usr/bin/monitor/conf:/usr/bin/monitor/lib/* util.geode.monitor.jmx.StartMonitor

# Geode/GemFire JMX Monitor Command Client

The Geode/GemFire JMX Monitor Command Client application is used to send commands to the Geode/GemFire JMX Monitor.

## Client Commands

1. Reload - This command will reload the excluded message file to pick up new message exclusions. The execludedMessages.xml file in the Geode/GemFire JMX monitor will need to be modified in the contqainer (if Docker container is used) 
2. Status - Provides the status of the monitor and if the monitor is connected to Geode/GemFire locator(s).
3. Shutdown - Shutdown the Geode/GemFire JMX monitor

### Client Command Example
java -cp datatx-geode-jmx-monitor-1.0.0.jar util.geode.monitor.client.MonitorCommand -h localhost -p 1099 -c status

-h = hostname or IP address where the Geode?GemFire monitor is running
-p = port number the Geode?GemFire JMX Monitor listens for incoming connections.
-c = Command to run [Reload,Status,Shutdown]





