## [Datadog를 활용한 Kafka Monitoring 방법]

<br>

### [Usage & Advantage]

<br>

> #### Datadog는 유료이긴 하지만 Elastic Stack과 같이 직접 설치하여 운영할 필요가 없으며, agent 설치 만으로 안정적인 모니터링 가능
- 운영 용이 : Datadog agent 설치만으로 운영 가능하며 설정이 단순함
- 사용량 기반 : pricing 정책에 따라 월 단위 과금 가능
- kafka dashboard 제공 : 사전에 정의된 kafka monitoring용 dashboard 제공


<br><br>

----------------------------------------------------

<br>


<br><br>

> ##  STG.01 Zookeeper, Kafka, Producer, Consumer 기동

<br>

#### 1) broker-01 서버에 접속하여 zookeeper 및 kafka를 기동한다


```
./start_zk.sh
./start_kafka.sh
```

<br>

#### 2) client-01 서버에 접속하여 producer(logstash) & consumer(logstash)를 실행한다

```
## run the producer (GCP VM Instance의 external ip로 변경해준다)
> cd ~
> export LS_JAVA_OPTS='-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
  -Dcom.sun.management.jmxremote.ssl=false 
  -Dcom.sun.management.jmxremote.port=9998 
  -Dcom.sun.management.jmxremote.rmi.port=9998
  -Djava.rmi.server.hostname=34.64.200.217'
> ~/logstash-7.15.0/bin/logstash -w 1 -b 1 -f ~/logstash_conf/producer.conf

## run the consumer (GCP VM Instance의 external ip로 변경해준다)
> export LS_JAVA_OPTS='-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
  -Dcom.sun.management.jmxremote.ssl=false 
  -Dcom.sun.management.jmxremote.port=9997 
  -Dcom.sun.management.jmxremote.rmi.port=9997
  -Djava.rmi.server.hostname=34.64.200.217'
> ~/logstash-7.15.0/bin/logstash -w 1 -b 1 --path.data ~/data/consumer_data -f ~/logstash_conf/consumer.conf
```

<br><br>

> ##  STG.02 Install & Configure Datadog Agent(Centos

<br>

#### 1) broker-01 서버에 접속하여 Agent 설치

- Integrations > Agent > CentOS/Red Hat 선택 > Agent Install 명령어 Copy & broker-01 서버에서 paste

![image](https://user-images.githubusercontent.com/30817824/172829047-6a035bfc-fa1f-4fd8-bea7-320bc595e50d.png)

```
# Example
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=344XXXXXXXXXXXXXXXX DD_SITE="datadoghq.com" bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"
```

<br>
- Agent 정상 설치 및 동작 여부 확인

```
> ps -ef | grep datadog
dd-agent 10384     1  1 09:13 ?        00:00:03 /opt/datadog-agent/bin/agent/agent run -p /opt/datadog-agent/run/agent.pid
dd-agent 10385     1  0 09:13 ?        00:00:00 /opt/datadog-agent/embedded/bin/trace-agent --config /etc/datadog-agent/datadog.yaml --pid /opt/datadog-agent/run/trace-agent.pid
dd-agent 10386     1  0 09:13 ?        00:00:00 /opt/datadog-agent/embedded/bin/process-agent --config=/etc/datadog-agent/datadog.yaml --sysprobe-config=/etc/datadog-agent/system-probe.yaml --pid=/opt/datadog-agent/run/process-agent.pid

> sudo systemctl status datadog-agent
```

<br>

#### 2) Kafka Integration 설정 (broker-01에 설정)

Integrations 설정을 통해 어떤 metrics 정보를 수집할 지 설정할 수 있다 kafka.d/conf.yaml 파일을 수정한다

- Integrations > Integrations > Kafka 검색 및 선택 > kafka.d/conf.yaml 파일 수정

![image](https://user-images.githubusercontent.com/30817824/172830022-eeec8556-c398-41fa-bc9f-870725264a26.png)

<br>

![image](https://user-images.githubusercontent.com/30817824/172830820-f0e08daa-ebf5-423c-a983-6d5798b5c6df.png)

<br>

##### Kafka Broker/Producer/Consumer JMX 설정 추가
- JMX로 모니터링할 서버와 jmx port를 지정한다. 

<br>

> sudo vi /etc/datadog-agent/conf.d/kafka.d/conf.yaml
```
init_config:

    is_jmx: true
    collect_default_metrics: true
    new_gc_metrics: true

instances:

  - host: localhost
    port: 9999
    tags:
      - kafka:broker0

  - host: client-01
    port: 9997
    tags:
      - kafka:consumer0

  - host: client-01
    port: 9998
    tags:
      - kafka:producer0

```

<br>



#### 3) Kafka Consumer Integration 설정 (broker-01에 설정)

- Integrations > Integrations > Kafka Consumer Integration 선택 > kafka_consumer.d/conf.yaml 파일 수정

![image](https://user-images.githubusercontent.com/30817824/172830773-737678a8-4732-414a-8f75-32e67826ec0d.png)

<br>

![image](https://user-images.githubusercontent.com/30817824/172831017-0b7e076b-fbfc-45fd-88dd-40a41af5744f.png)


##### Kafka Consumer 관련(lag, offet 등) JMX 설정 추가
- Consumer 관련 내용만 수집
- https://docs.datadoghq.com/integrations/kafka/?tab=host#kafka-consumer-integration 참고

<br>

> sudo vi /etc/datadog-agent/conf.d/kafka_consumer.d/conf.yaml 
```
init_config:
    # kafka_timeout: 5

instances:

  - kafka_connect_str:
    - localhost:9092
    kafka_consumer_offsets: true
    monitor_unlisted_consumer_groups: true

```


<br><br>

> ##  STG.03 Datadog Agent 재시작 및 정상 동작 확인

<br>

#### 1) Datadog Agent 재시작
```
sudo systemctl restart datadog-agent
```

<br>

#### 2) Datadog Agent 상태 확인

```
> sudo datadog-agent status


# 아래 kafka의 정보가 정상적으로 보이면 됨.
=========
Collector
=========

  Running Checks
  ==============
    kafka_consumer (2.12.1)
    -----------------------
      Instance ID: kafka_consumer:23a55777eda24f89 [OK]
      Configuration Source: file:/etc/datadog-agent/conf.d/kafka_consumer.d/conf.yaml
      Total Runs: 1
      Metric Samples: Last Run: 6, Total: 6
      Events: Last Run: 0, Total: 0
      Service Checks: Last Run: 0, Total: 0
      Average Execution Time : 617ms
      Last Execution Date : 2021-12-09 12:29:07 UTC (1639052947000)
      Last Successful Execution Date : 2021-12-09 12:29:07 UTC (1639052947000)

========
JMXFetch
========

  Information
  ==================
    runtime_version : 1.8.0_312
    version : 0.44.3
  Initialized checks
  Information
  ==================
    runtime_version : 1.8.0_312
    version : 0.44.3
  Initialized checks
  ==================
    kafka
      instance_name : kafka-localhost-9999
      message : <no value>
      metric_count : 69
      service_check_count : 0
      status : OK
      instance_name : kafka-kafka-client-9997
      message : <no value>
      metric_count : 34
      service_check_count : 0
      status : OK
      instance_name : kafka-kafka-client-9998
      message : <no value>
      metric_count : 56
      service_check_count : 0
      status : OK
```

![image](https://user-images.githubusercontent.com/30817824/172836583-85349fc0-8629-4ff1-8a8e-49b4541ac12a.png)

<br>

##### [ERROR 발생 시] /etc/datadog-agent/conf.d/kafka.d/conf.yaml 파일의 host가 VM Instance(producer & consumer)의 Name과 일치하는지 확인

![image](https://user-images.githubusercontent.com/30817824/172836073-c0ea9610-29ae-4a24-b2f0-b8bb110d0027.png)


<br>

#### 3) Datadog Dashboard에서 Kafka Metrics 확인

![image](https://user-images.githubusercontent.com/30817824/172836756-23ab3d33-64c7-4898-abfa-737444febb10.png)

<br>

![image](https://user-images.githubusercontent.com/30817824/172836980-73302037-9873-4629-8e9f-d57316cbabcd.png)


-------------------------------------

##### [references]
- https://www.datadoghq.com/blog/monitor-kafka-with-datadog/
- https://docs.datadoghq.com/integrations/faq/troubleshooting-and-deep-dive-for-kafka/
- https://github.com/freepsw/kafka-metrics-monitoring
